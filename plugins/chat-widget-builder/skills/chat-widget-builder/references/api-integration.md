# API Integration

Part A records how the reference (`react-main` @ `e8cf9d7`) talks to its backends, verified from source, so you can reuse proven patterns and recognize the contract when a user's backend is the same MSG91 Hello / GTWY platform. Part B is how to integrate **the user's** APIs in a generated widget. Never copy Part A endpoints into a new project unless the user confirms that is their backend.

## Contents

- A1. Two products, two stacks
- A2. HELLO (human support) HTTP endpoints
- A3. HELLO auth and identity
- A4. HELLO real-time (socket.io)
- A5. GTWY (AI chatbot) HTTP + RTLayer
- A6. Error handling in the reference
- B1. Integrating the user's API: process
- B2. API client pattern
- B3. Real-time pattern
- B4. Mapping backend messages to the widget model

## A1. Two products, two stacks

The mode is `state.draftData.isHelloUser`.

| Mode | Set by | HTTP | Real-time |
| --- | --- | --- | --- |
| HELLO: human/helpdesk chat (MSG91) | Embed script sets `isHelloUser: true` (`hooks/HELLO/eventHandlers/embeddingScript/embeddingScriptEventHandler.ts`) | `config/helloApi.ts` via `utils/helloInterceptor.ts` | socket.io: `hooks/socketManager.js` (chat) + `hooks/notifications/notificationSocketManager.ts` (notifications) |
| GTWY: AI chatbot ("interface") | `app/chatbot/page.tsx` stores `interfaceToken`, `isHelloUser: false` | `config/api.ts`, `config/ragApi.ts` via `utils/interceptor.ts` | `rtlayer-client` (`components/Chatbot/hooks/useRtlayerEventManager.tsx`) |

Env vars (names only): `NEXT_PUBLIC_MSG91_HOST_URL` (HELLO API), `NEXT_PUBLIC_PUSH_NOTIFICATION_URL`, `NEXT_PUBLIC_API_BASE_URL` (GTWY), `NEXT_PUBLIC_PYTHON_API_BASE_URL` (GTWY AI), `NEXT_PUBLIC_SOCKET_URL` (chat socket), `NEXT_PUBLIC_NOTIFICATOIN_SOCKET_URL` (sic). An iframe query `?env=stage` swaps three base URLs to hardcoded stage hosts.

## A2. HELLO HTTP endpoints

All POST bodies carry `user_data` = stored `userData` minus `name` (`unique_id`, `mail`, `number`, `user_jwt_token` when the host supplied them). `authorization` header = `` `${WidgetId}:${k_clientId || a_clientId}` `` (no `Bearer`), or `WidgetId` alone where noted. Optional `{flow_origin, integration}` merge params come from sessionStorage `helloConfig`.

| Purpose | Method + path | Key request | Response used |
| --- | --- | --- | --- |
| Widget settings | POST `/widget-info/` (auth: `WidgetId`) | `{user_data}` | `hide_launcher`, `launch_widget`, `bot_type`, `voice_call_widget`, `primary_color`, `teams`, `company_id`, `bot_id`, `event_channels`, `inbox_id`, `demo_widget`, `show_widget_form`. Falsy → parent gets `initializeHelloChat_failed` |
| Greeting | GET `/chat-gpt/greeting/?company_id&bot_id`, or POST `/chat-bot/welcome/get-welcome/` when `bot_type === 'lex'` | ids | `data.greeting.{text, options}` (welcome text + quick-reply options) |
| Agents/teams | POST `/agent-team/` | `{user_data}` | `{agents[], teams[]}` → id→name maps |
| Anonymous client | POST `/anonymous-client-details/` (auth: `WidgetId`) | `{}` | `data.data.uuid` → `a_clientId`. Only when no client id and no `unique_id`/`mail`/`number` |
| Update client profile | PUT `/v2/client/{clientId}` | host params (Name, Phonenumber, Email, custom) + `user_data` | Missing Name/Phone/Email → pre-chat form shown |
| Socket JWT | GET `/jwt-token/` | — | `data.data.jwt_token` → `JwtTokenForSocket` |
| List conversations | POST `/pubnub-channels/list/` | `{name, mail, number, unique_id, user_data, anon_client_uuid?}` | `uuid`, `pseudo_name`, `customer_*`, `channels[]` (`channel`, `id`, `widget_unread_count`, `is_closed`, `assigned_type`, `is_peer_channel`, `is_blocked`) |
| Send message (also creates a channel) | POST `/v2/send/` | `type`, `widget_msg_id` (client 24-char id), `message_type` (`text`/`attachment`/`text-attachment`/`voice_call`), `content:{text, attachment[]}`, `chat_id`, `replied_on?`, `channelDetail` (new chat only; includes client-proposed `channel_hex` = `ch-comp-{companyId}.{uuid}`), `sessionVariables` | `uuid`, `channel`, `id`, `presence_channel`, `call_jwt_token` |
| History | POST `/get-history/` | `{channel, origin:"chat", page_size:30, start_from: skip+1}` | `data.data[]` of `{id, timetoken, message}`; more pages while length ≥ 30 |
| Full text of truncated message | POST `/get-history/` + `message_id` | | used by "Read more" |
| Mark read | DELETE `/read-receipt/{channelId}` | — | debounced 1 s; only when open, visible, channel has unread |
| Upload | POST `/v2/upload/?type=chat&inbox_id=` multipart field `attachment` | | `data[0]` = URL |
| Feedback (CSAT) | POST `/receive-feedback/` | `{feedback_msg, rating, token, type:"post-feedback", id}` | — |
| Domain tracking | PUT `/add-domain/` | `{dom, event_data}` | errors swallowed |
| Call client token | GET `/web-rtc/get-client-token/` | — | `jwt_token` → `HelloClientToken` (only if `voice_call_widget`) |
| Push (mobile SDK) | POST `{PUSH_URL}/add-user-fcm-token/` | `{...pushConfig, user_channel}` | auth = socket JWT |

No ticket endpoint (ticket view is a UI mode opening an existing channel). No client handoff endpoint (handoff is server-driven, see A4 `assign`).

## A3. HELLO auth and identity

- Tenant: `widgetToken` from the embed → stored as `WidgetId`.
- All `getLocalStorage`/`setLocalStorage` keys are prefixed `${widgetToken}_` (and the session token becomes `widgetToken_uniqueId` for identified users), so storage is namespaced per widget and per user (`utils/utilities.js`).
- `a_clientId` = anonymous client uuid; `k_clientId` = known client uuid. Writing either posts `setDataInLocal` to the host page.
- Anonymous → identified merge: on first login the list call sends the anonymous id as `anon_client_uuid`, so the anonymous conversation is merged into the user; on logout the last known id is sent. Bookkeeping keys: `${widgetId}_last_unique_id`, `${widgetId}_k_clientId`, `${widgetId}_a_clientId`.
- Host identity proof: optional `user_jwt_token`, sent inside `user_data`. When it is required is a backend setting; confirm it with the user.
- No token refresh anywhere; tokens are fetched once per session init.

## A4. HELLO real-time (socket.io)

- One singleton: `io(NEXT_PUBLIC_SOCKET_URL, { auth: { token: jwt }, transports: ['websocket','polling'], reconnection: true, timeout: 20000 })`. `connect()` is a no-op if connected/connecting.
- Subscribe by emitting `subscribe` `{channel: [...]}` (with ack) to: personal channel `ch-comp-{company_id}-{clientId}`, every listed conversation `channel`, every `widgetInfo.event_channels` entry, and after a send the `presence_channel` + new `channel`. `unsubscribe` likewise. Subscriptions are deduplicated; emits while disconnected are queued.
- Before the **first** send of a new chat the client pre-subscribes to its proposed `channel_hex` (2 s timeout) so the bot's first reply is not missed; the next history fetch is skipped once so it does not overwrite that reply.
- Outbound `typing` / `not-typing` `{channel: <typing event channel>, data: {action, channel}}`, sent on state change; `not-typing` after 2 s idle and on send.
- Inbound `NewPublish` `{response: {message, timetoken, id}}`, branched on `message.type`:
  - `chat` → add message (sound, clear typing) or, for own echo while visible, overwrite by id; reopen channel; voice-call status messages; unread + move channel to top for others' messages.
  - `assign` → channel `assigned_type`/`assignee_id` (bot → agent/team handoff).
  - `feedback` → add CSAT message, mark channel closed.
  - `block` → peer channel blocked state.
  - `new-channel` → add + subscribe + refetch list.
  - `update` → client id changed: swap personal-channel subscription.
- Inbound `Typing` `{channel, action}` → per-channel typing flag.
- Reconnect: on `connect` re-subscribe all tracked channels and flush queue; on `reconnect` refetch current channel history.
- No inbound presence or read-receipt events are handled.
- A second socket (notifications) subscribes to `ch-comp-{company_id}.{uuid}` (dot separator) and turns `type:'notification'` publishes into host events (`PUSH_NOTIFICATION`, `LAUNCHER_MESSAGE_PREVIEW`); it acks each message; it does **not** re-subscribe on reconnect.

## A5. GTWY (AI chatbot) HTTP + RTLayer

- Interceptor adds `Authorization: <sessionStorage interfaceToken>` (or `proxy_auth_token: <ragToken>` for URLs containing `rag`).
- Send: POST `{PYTHON_URL}/chatbot/{chatBotId}/sendMessage` with `message, images, files, userId, interfaceContextData, threadId, subThreadId, slugName, thread_flag, chatBotId, version_id, configuration?, service?`. The HTTP response is only an ack; **the reply arrives over RTLayer**.
- History: GET `{URL}/api/v1/config/gethistory-chatbot/{threadId}/{bridgeName}?sub_thread_id&pageNo&limit=40` (previous request aborted with `AbortController`), returns messages + `starterQuestion`.
- Feedback: PUT `{URL}/api/v1/config/status/{status}` `{message_id}`. Threads: GET `{URL}/thread/{threadId}?slugName=`. Image upload: POST `{PYTHON_URL}/image/processing/` field `image` → `image_url`.
- Hello handoff config: POST `{URL}/hello/subscribe` returns `mode`, `widgetInfo`, `ChannelList`, `Jwt`, `supportedServices`; when `mode` includes `human` the header shows AI/Human tabs (the tab click does not actually switch transports in this code).
- RTLayer channel = `chatSessionId + (threadId||userId) + (subThreadId||userId)` (spaces → `_`). Inbound JSON branches: `function_call` (interim "function calling" placeholder), `error`, `role:'reset'`, `suggestions` (quick replies), `display_name` (thread renamed), otherwise replace the assistant placeholder with the final message. No reconnect/refetch logic.

## A6. Error handling in the reference

- HELLO: every API function catches, toasts, returns `null`/`[]`; no retries, no 401 handling. Widget-info failure stops init and notifies the host.
- GTWY: 401 → clears all localStorage and redirects with `?error=session expired`; ≥500 → toast. Send failure removes the last two messages (user + placeholder).
- A 10 s loading timeout prevents endless "typing" spinners in both modes.

## B1. Integrating the user's API: process

1. Run the intake checklist in `templates/api-documentation.md` for the selected features only.
2. Get **real** request/response examples (and socket event payloads). Paste them into `docs/API.md` and the mock handlers verbatim.
3. Identify the identity model and flag risk if the backend trusts client-supplied identity.
4. Confirm CORS allows the widget origin (the iframe/app origin, not the host page, for iframe builds).
5. Anything unconfirmed: `TODO(confirm-with-user)` in code and in the final report. Do not ship guessed fields.

## B2. API client pattern

```ts
// api/client.ts
export class ApiError extends Error {
  constructor(public code: string, message: string, public status?: number) { super(message); }
}

export function createApiClient(opts: { baseUrl: string; getAuthHeaders: () => Record<string, string>; timeoutMs?: number }) {
  return async function request<T>(path: string, init: RequestInit & { json?: unknown } = {}): Promise<T> {
    const ctrl = new AbortController();
    const timer = setTimeout(() => ctrl.abort(), opts.timeoutMs ?? 15000);
    init.signal?.addEventListener('abort', () => ctrl.abort());
    try {
      const res = await fetch(opts.baseUrl + path, {
        ...init,
        signal: ctrl.signal,
        headers: { ...(init.json !== undefined && { 'Content-Type': 'application/json' }), ...opts.getAuthHeaders(), ...init.headers },
        body: init.json !== undefined ? JSON.stringify(init.json) : init.body,
      });
      const data = await res.json().catch(() => null);
      if (!res.ok) throw new ApiError(mapErrorCode(res.status, data), mapErrorMessage(data), res.status);
      return data as T;
    } catch (e) {
      if (e instanceof ApiError) throw e;
      throw new ApiError(ctrl.signal.aborted ? 'TIMEOUT' : 'NETWORK_ERROR', 'Connection problem. Please try again.');
    } finally {
      clearTimeout(timer);
    }
  };
}
```

`mapErrorCode` / `mapErrorMessage` come from the user's confirmed error format. Endpoint functions live in `api/endpoints.ts` and return **widget-model** objects (B4), so UI and state never see raw backend shapes. Unlike the reference, errors propagate to the state layer, which decides UI (failed bubble, retry, toast); do not toast from the API layer.

## B3. Real-time pattern

```ts
// realtime/transport.ts — one instance per widget
export interface Transport {
  connect(auth: { token: string }): void;
  subscribe(channels: string[]): void;   // idempotent; remembered for re-subscribe
  on(event: 'message' | 'typing' | 'status', cb: (payload: unknown) => void): () => void;
  emit(event: string, payload: unknown): void; // queued while disconnected if the backend expects it
  disconnect(): void;
}
```

Implement it over what the backend speaks (socket.io / WebSocket / SSE / polling; `development-guidelines.md` §5). Carry over the reference's good behaviours: idempotent subscriptions, re-subscribe on connect, history refetch + merge on reconnect, pre-subscribe before the first send when the server replies on a channel created by that send. Fix its gaps: dedupe by id, re-subscribe every socket, unsubscribe/cleanup on destroy, validate payload shape.

## B4. Mapping backend messages to the widget model

Write one adapter per inbound shape (history item, live event, send ack) into a single model:

```ts
export interface ChatMessage {
  id: string;              // server id; before ack, the client id
  clientId?: string;       // client-generated id sent with the request (cf. reference widget_msg_id) for ack/echo matching
  conversationId: string;
  sender: 'user' | 'bot' | 'agent' | 'system';
  senderName?: string;
  type: 'text' | 'image' | 'file' | 'buttons' | 'feedback' | 'system'; // only types in scope
  text?: string;
  attachments?: { url: string; name?: string; mime?: string; size?: number }[];
  options?: { label: string; value: string }[];
  replyToId?: string;
  createdAt: string;       // ISO, from the server when available
  status: 'pending' | 'sent' | 'delivered' | 'read' | 'failed';
}
```

Sender classification must follow fields the backend actually sends (the reference uses `sender_id` of `bot`/`workflow` or `is_auto_response` → bot; own-client checks → user; else agent). Confirm the equivalent with the user.
