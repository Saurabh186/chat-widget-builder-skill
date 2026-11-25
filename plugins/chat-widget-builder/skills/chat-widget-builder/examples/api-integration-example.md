# Example: Mapping a user's API contract into the widget

The user pasted REST and socket docs for their backend ("Acme Support API"). Below is how Claude turned the contract into an HTTP client, endpoint adapters, a real-time transport and tests.

Code marked **verified** was type-checked with `tsc --strict` and passed Vitest when this skill was written:
- 6 adapter tests, run against a stubbed `fetch`;
- 2 transport tests, run against a real local `socket.io` server.

## Contents

- 1. What the user provided
- 2. HTTP client (verified)
- 3. Endpoint adapters (verified)
- 4. Real-time transport (verified)
- 5. Tests (verified)
- 6. Questions Claude asked about gaps
- 7. Docs produced
- 8. Variant: the backend is MSG91 Hello (the reference's backend)

## 1. What the user provided

```
Base URL: https://api.acme.test      Socket: wss://rt.acme.test (socket.io v4)
Auth: POST /v1/widget/session returns sessionToken; send it as "Authorization: Bearer <token>"
      and as socket.io auth { token }.

POST /v1/widget/session
  { "widgetId": "wid_123", "user": { "id": "u_42", "name": "Ada", "email": "ada@x.test", "signature": "<hmac from host backend>" } }
  → 200 { "sessionToken": "…", "visitorId": "v_1", "conversationId": "conv_1" }      (user omitted → anonymous)

GET /v1/conversations/{id}/messages?before=<messageId>&limit=30
  → 200 { "messages": [ { "id": "msg_2", "clientMessageId": null, "conversationId": "conv_1",
                          "author": { "type": "agent", "name": "Riya" }, "text": "Hello!",
                          "createdAt": "2026-11-20T09:15:02Z" } ], "hasMore": true }
  (messages oldest → newest within a page)

POST /v1/conversations/{id}/messages   { "clientMessageId": "c-1", "text": "Hi" }
  → 201 { "message": { …same shape, author.type "visitor", clientMessageId "c-1" } }

Errors: { "error": { "code": "CONVERSATION_CLOSED", "message": "…" } }  (401 SESSION_EXPIRED, 409, 429)

Socket: client emits "conversation:join" { conversationId }
        server emits "message:new" { message }  (includes the visitor's own messages, with clientMessageId)
                     "typing" { conversationId, active, author: { type } }
```

## 2. HTTP client (verified)

`src/api/client.ts`

```ts
export class ApiError extends Error {
  constructor(public readonly code: string, message: string, public readonly status?: number) {
    super(message);
    this.name = 'ApiError';
  }
}

export interface ApiClientOptions {
  baseUrl: string;
  getAuthHeaders: () => Record<string, string>;
  timeoutMs?: number;
  fetchImpl?: typeof fetch;
}

export interface RequestOptions {
  method?: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  json?: unknown;
  body?: BodyInit;
  query?: Record<string, string | number | undefined>;
  signal?: AbortSignal;
}

export type ApiRequest = <T>(path: string, options?: RequestOptions) => Promise<T>;

export function createApiClient({ baseUrl, getAuthHeaders, timeoutMs = 15000, fetchImpl = fetch }: ApiClientOptions): ApiRequest {
  return async function request<T>(path: string, options: RequestOptions = {}): Promise<T> {
    const url = new URL(path, baseUrl);
    for (const [key, value] of Object.entries(options.query ?? {})) {
      if (value !== undefined) url.searchParams.set(key, String(value));
    }

    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), timeoutMs);
    options.signal?.addEventListener('abort', () => controller.abort());

    let response: Response;
    try {
      response = await fetchImpl(url, {
        method: options.method ?? 'GET',
        signal: controller.signal,
        headers: {
          Accept: 'application/json',
          ...(options.json !== undefined && { 'Content-Type': 'application/json' }),
          ...getAuthHeaders(),
        },
        body: options.json !== undefined ? JSON.stringify(options.json) : options.body,
      });
    } catch {
      throw controller.signal.aborted && !options.signal?.aborted
        ? new ApiError('TIMEOUT', 'The server took too long to respond.')
        : new ApiError('NETWORK_ERROR', 'Connection problem. Please try again.');
    } finally {
      clearTimeout(timer);
    }

    const data: unknown = await response.json().catch(() => null);
    if (!response.ok) {
      const error = (data as { error?: { code?: string; message?: string } } | null)?.error;
      throw new ApiError(error?.code ?? `HTTP_${response.status}`, error?.message ?? 'Something went wrong.', response.status);
    }
    return data as T;
  };
}
```

## 3. Endpoint adapters (verified)

`src/api/endpoints.ts`. The UI and the store only ever see `ChatMessage` (defined in `basic-chat-widget.md` §3). The backend's shapes stay inside this file.

```ts
// Adapters for the Acme Support API contract the user pasted (examples/api-integration-example.md §1).
import type { ApiRequest } from './client';
import type { ChatMessage, Sender } from '../state/types';

interface AcmeMessage {
  id: string;
  clientMessageId?: string | null;
  conversationId: string;
  author: { type: 'visitor' | 'bot' | 'agent'; name?: string };
  text: string;
  createdAt: string;
}

export interface AcmeSession {
  sessionToken: string;
  visitorId: string;
  conversationId: string;
}

const SENDER: Record<AcmeMessage['author']['type'], Sender> = { visitor: 'user', bot: 'bot', agent: 'agent' };

export function toChatMessage(m: AcmeMessage): ChatMessage {
  return {
    id: m.id,
    clientId: m.clientMessageId ?? undefined,
    conversationId: m.conversationId,
    sender: SENDER[m.author.type],
    senderName: m.author.name,
    text: m.text,
    createdAt: m.createdAt,
    status: 'sent',
  };
}

export function createChatApi(request: ApiRequest) {
  return {
    startSession(widgetId: string, user?: { id: string; name?: string; email?: string; signature: string }) {
      return request<AcmeSession>('/v1/widget/session', { method: 'POST', json: { widgetId, user } });
    },

    async fetchMessages(conversationId: string, before?: string, signal?: AbortSignal) {
      const page = await request<{ messages: AcmeMessage[]; hasMore: boolean }>(
        `/v1/conversations/${encodeURIComponent(conversationId)}/messages`,
        { query: { before, limit: 30 }, signal },
      );
      return { messages: page.messages.map(toChatMessage), hasMore: page.hasMore };
    },

    async sendMessage(conversationId: string, clientMessageId: string, text: string) {
      const { message } = await request<{ message: AcmeMessage }>(
        `/v1/conversations/${encodeURIComponent(conversationId)}/messages`,
        { method: 'POST', json: { clientMessageId, text } },
      );
      return toChatMessage(message);
    },
  };
}

export type ChatApi = ReturnType<typeof createChatApi>;
```

## 4. Real-time transport (verified)

`src/realtime/transport.ts`

```ts
import { io, type Socket } from 'socket.io-client';
import { toChatMessage } from '../api/endpoints';
import type { ChatMessage } from '../state/types';

export interface TransportHandlers {
  onMessage(message: ChatMessage): void;
  onTyping(active: boolean): void;
  onConnectionChange(connected: boolean): void;
  onReconnected(): void; // caller refetches history newer than the last known message and merges
}

/** One socket per widget instance. Event names and payloads follow the Acme contract. */
export function connectRealtime(url: string, token: string, conversationId: string, handlers: TransportHandlers) {
  const socket: Socket = io(url, { auth: { token }, transports: ['websocket'], reconnectionDelayMax: 10000 });
  let hasConnected = false;

  socket.on('connect', () => {
    socket.emit('conversation:join', { conversationId });
    handlers.onConnectionChange(true);
    if (hasConnected) handlers.onReconnected();
    hasConnected = true;
  });
  socket.on('disconnect', () => handlers.onConnectionChange(false));
  socket.on('message:new', (payload: { message?: Parameters<typeof toChatMessage>[0] }) => {
    if (payload?.message?.conversationId === conversationId) handlers.onMessage(toChatMessage(payload.message));
  });
  socket.on('typing', (payload: { conversationId?: string; active?: boolean; author?: { type?: string } }) => {
    if (payload?.conversationId === conversationId && payload.author?.type !== 'visitor') handlers.onTyping(Boolean(payload.active));
  });

  return {
    disconnect() {
      socket.removeAllListeners();
      socket.disconnect();
    },
  };
}
```

What this transport does:
- **Re-join on reconnect.** On every `connect` it emits the join event again, the same idea as the reference's re-subscribe on connect.
- **Catch up after reconnect.** `onReconnected` lets the controller refetch history and merge it. The store's dedupe makes that merge safe.
- **Stay scoped.** It ignores events for other conversations.
- **Clean teardown.** It removes all its listeners when the widget is destroyed.

## 5. Tests (verified)

`tests/endpoints.test.ts`

```ts
import { describe, expect, it, vi } from 'vitest';
import { ApiError, createApiClient } from '../src/api/client';
import { createChatApi } from '../src/api/endpoints';

const json = (body: unknown, status = 200) =>
  new Response(JSON.stringify(body), { status, headers: { 'Content-Type': 'application/json' } });

function setup(fetchImpl: typeof fetch) {
  const request = createApiClient({
    baseUrl: 'https://api.acme.test',
    getAuthHeaders: () => ({ Authorization: 'Bearer session-token' }),
    timeoutMs: 50,
    fetchImpl,
  });
  return createChatApi(request);
}

const agentMessage = {
  id: 'msg_2', clientMessageId: null, conversationId: 'conv_1',
  author: { type: 'agent', name: 'Riya' }, text: 'Hello!', createdAt: '2026-11-20T09:15:02Z',
};

describe('chat API adapters', () => {
  it('sends a message with the clientMessageId and maps the response', async () => {
    const fetchImpl = vi.fn<typeof fetch>().mockResolvedValue(json({ message: {
      id: 'msg_9', clientMessageId: 'c-1', conversationId: 'conv_1', author: { type: 'visitor' }, text: 'Hi', createdAt: '2026-11-20T09:15:00Z',
    } }, 201));
    const api = setup(fetchImpl);

    const message = await api.sendMessage('conv_1', 'c-1', 'Hi');

    const [url, init] = fetchImpl.mock.calls[0]!;
    expect(String(url)).toBe('https://api.acme.test/v1/conversations/conv_1/messages');
    expect(init?.method).toBe('POST');
    expect(init?.headers).toMatchObject({ Authorization: 'Bearer session-token', 'Content-Type': 'application/json' });
    expect(JSON.parse(String(init?.body))).toEqual({ clientMessageId: 'c-1', text: 'Hi' });
    expect(message).toMatchObject({ id: 'msg_9', clientId: 'c-1', sender: 'user', status: 'sent' });
  });

  it('fetches a history page with the cursor and maps senders', async () => {
    const fetchImpl = vi.fn<typeof fetch>().mockResolvedValue(json({ messages: [agentMessage], hasMore: true }));
    const page = await setup(fetchImpl).fetchMessages('conv_1', 'msg_5');

    expect(String(fetchImpl.mock.calls[0]![0])).toBe('https://api.acme.test/v1/conversations/conv_1/messages?before=msg_5&limit=30');
    expect(page.hasMore).toBe(true);
    expect(page.messages[0]).toMatchObject({ id: 'msg_2', sender: 'agent', senderName: 'Riya', clientId: undefined });
  });

  it('maps the error envelope to ApiError', async () => {
    const fetchImpl = vi.fn<typeof fetch>().mockResolvedValue(json({ error: { code: 'CONVERSATION_CLOSED', message: 'This conversation is closed.' } }, 409));
    await expect(setup(fetchImpl).sendMessage('conv_1', 'c-1', 'Hi')).rejects.toMatchObject({ code: 'CONVERSATION_CLOSED', status: 409 });
  });

  it('maps a non-JSON 5xx to a generic error', async () => {
    const fetchImpl = vi.fn<typeof fetch>().mockResolvedValue(new Response('<html>Bad gateway</html>', { status: 502 }));
    await expect(setup(fetchImpl).fetchMessages('conv_1')).rejects.toMatchObject({ code: 'HTTP_502' });
  });

  it('maps network failure to NETWORK_ERROR', async () => {
    const fetchImpl = vi.fn<typeof fetch>().mockRejectedValue(new TypeError('Failed to fetch'));
    await expect(setup(fetchImpl).fetchMessages('conv_1')).rejects.toMatchObject({ code: 'NETWORK_ERROR' });
  });

  it('times out slow requests', async () => {
    const fetchImpl = vi.fn<typeof fetch>((_url, init) => new Promise((_resolve, reject) => {
      init?.signal?.addEventListener('abort', () => reject(new DOMException('aborted', 'AbortError')));
    }));
    const error = await setup(fetchImpl).fetchMessages('conv_1').catch((e: unknown) => e);
    expect(error).toBeInstanceOf(ApiError);
    expect(error).toMatchObject({ code: 'TIMEOUT' });
  });
});
```

`tests/transport.test.ts` runs against a real `socket.io` server on a random port. It needs `socket.io` and `@types/node` as dev dependencies.

```ts
import { createServer } from 'node:http';
import type { AddressInfo } from 'node:net';
import { Server } from 'socket.io';
import { afterEach, describe, expect, it, vi } from 'vitest';
import { connectRealtime } from '../src/realtime/transport';

let cleanup: (() => void)[] = [];
afterEach(() => { cleanup.forEach((fn) => fn()); cleanup = []; });

async function startServer() {
  const http = createServer();
  const io = new Server(http);
  await new Promise<void>((resolve) => http.listen(0, resolve));
  cleanup.push(() => { io.close(); http.close(); });
  return { io, url: `http://localhost:${(http.address() as AddressInfo).port}` };
}

const until = (check: () => boolean) => vi.waitFor(() => { if (!check()) throw new Error('not yet'); }, { timeout: 3000 });

describe('connectRealtime (real socket.io server)', () => {
  it('authenticates, joins, receives messages and typing, ignores other conversations', async () => {
    const { io, url } = await startServer();
    const joined: unknown[] = [];
    let token: unknown;
    io.on('connection', (s) => {
      token = s.handshake.auth.token;
      s.on('conversation:join', (p) => {
        joined.push(p);
        s.emit('typing', { conversationId: 'conv_1', active: true, author: { type: 'agent' } });
        s.emit('message:new', { message: { id: 'x', conversationId: 'other', author: { type: 'agent' }, text: 'no', createdAt: '2026-11-20T09:00:00Z' } });
        s.emit('message:new', { message: { id: 'm1', conversationId: 'conv_1', author: { type: 'bot' }, text: 'yes', createdAt: '2026-11-20T09:00:01Z' } });
      });
    });

    const onMessage = vi.fn();
    const onTyping = vi.fn();
    const rt = connectRealtime(url, 'session-token', 'conv_1', { onMessage, onTyping, onConnectionChange: vi.fn(), onReconnected: vi.fn() });
    cleanup.push(rt.disconnect);

    await until(() => onMessage.mock.calls.length === 1);
    expect(token).toBe('session-token');
    expect(joined).toEqual([{ conversationId: 'conv_1' }]);
    expect(onTyping).toHaveBeenCalledWith(true);
    expect(onMessage).toHaveBeenCalledWith(expect.objectContaining({ id: 'm1', sender: 'bot', text: 'yes' }));
  });

  it('reports disconnect and calls onReconnected (and re-joins) after reconnect', async () => {
    const { io, url } = await startServer();
    let joins = 0;
    io.on('connection', (s) => s.on('conversation:join', () => { joins += 1; }));

    const onConnectionChange = vi.fn();
    const onReconnected = vi.fn();
    const rt = connectRealtime(url, 't', 'conv_1', { onMessage: vi.fn(), onTyping: vi.fn(), onConnectionChange, onReconnected });
    cleanup.push(rt.disconnect);

    await until(() => joins === 1);
    io.of('/').sockets.forEach((s) => s.conn.close()); // simulate network drop (client auto-reconnects)
    await until(() => joins === 2);
    expect(onConnectionChange).toHaveBeenCalledWith(false);
    expect(onReconnected).toHaveBeenCalledTimes(1);
  });
});
```

## 6. Questions Claude asked about gaps

Each gap became one question, and the answer was recorded. Nothing was guessed:

1. "Does `message:new` include the visitor's own messages, and with `clientMessageId`?" Yes. So the store dedupes the echo by `clientId`.
2. "How do I get messages missed during a disconnect: `after=<id>` or the latest page?" Only `before` exists. So reconnect refetches the latest page and merges it.
3. "What should happen on 401 `SESSION_EXPIRED`?" Start a new session with the same identity once, then show an error. This is implemented in the controller, not in the client.
4. "Is the `typing` author field always present?" Not confirmed, so it was marked `TODO(confirm-with-user)` and listed in the final report.
5. "Which origins does CORS allow?" The marketing and app domains, plus `localhost:5173` for the demo.

## 7. Docs produced

- `docs/API.md`, from `templates/api-documentation.md`, containing the contract above verbatim.
- `.env.example`:

```bash
# Public values only. No secrets belong in this file.
VITE_ACME_API_BASE_URL=https://api.acme.test
VITE_ACME_SOCKET_URL=wss://rt.acme.test
```

- `docs/INTEGRATION.md#identity`. It explains that the host backend computes `signature = HMAC_SHA256(user.id, <secret kept on the host server>)` and passes it to `identify()`. The secret never reaches the browser.

## 8. Variant: the backend is MSG91 Hello (the reference's backend)

If the user's backend **is** the reference's Hello platform, reuse the reference's contract from `references/api-integration.md` A2–A4 instead of asking for docs. Take the base URLs from `references/msg91-environments.md` (production by default); do not ask for them. Confirm two things with the user: the widget token (and check its environment, §5 there) and whether `user_jwt_token` is required.

Adapter mapping:

| Concern | Hello contract |
| --- | --- |
| Session | `POST /anonymous-client-details/`, or `POST /pubnub-channels/list/` with identity. `authorization: <widgetToken>:<clientId>`, no Bearer |
| History | `POST /get-history/` with `{channel, origin:"chat", page_size:30, start_from}`. Offset paging, not cursors |
| Send | `POST /v2/send/` with `widget_msg_id` (that is the `clientId`). A new conversation needs `channelDetail`, and the reference pre-subscribes to the proposed `channel_hex` first |
| Real-time | socket.io with `auth: { token: <jwt from /jwt-token/> }`. Emit `subscribe` with `{channel:[...]}`. Inbound `NewPublish` with `{response:{message,timetoken,id}}` (branch on `message.type`) and `Typing` |
| Sender | `sender_id` of `bot` or `workflow`, or `is_auto_response`, means bot. An own client id means user. Anything else is an agent |

Keep the reference's quirks behind the adapter. The UI and the store stay the same.
