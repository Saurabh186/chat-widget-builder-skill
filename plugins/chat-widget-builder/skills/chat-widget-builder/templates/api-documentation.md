# Template: docs/API.md

Copy into the generated project as `docs/API.md` and fill it **only from the user's confirmed contract**. Delete sections for endpoints the widget does not use. Anything not confirmed is marked `TODO(confirm-with-user)` and listed in the final report.

## Contents

- Template: docs/API.md
- Intake checklist (ask only what the requested features need)

~~~markdown
# API Integration

Source of truth: <link or file the user provided>, confirmed on <date>.

## Environments

| Env var | Purpose | Example (public) |
| --- | --- | --- |
| VITE_API_BASE_URL | REST base URL | https://api.example.com |
| VITE_SOCKET_URL | Real-time endpoint | wss://rt.example.com |

## Authentication

- Tenant identification: <e.g. header `x-widget-token: <public widget id>`>
- Anonymous session: <how the anon id/token is obtained and sent>
- Identified user: <JWT/HMAC signature minted by the host backend; header/field name>
- Expiry / refresh: <behaviour on 401>

## Endpoints

### <Purpose, e.g. Initialize widget>

`<METHOD> <path>`

Headers:
| Name | Value | Required |
| --- | --- | --- |

Request:
```json
<exact example from the user>
```

Response (200):
```json
<exact example from the user>
```

Fields used by the widget:
| Field | Used for |
| --- | --- |

Errors:
| Status / code | Meaning | Widget behaviour |
| --- | --- | --- |

(Repeat for: identify user, create/get conversation, list conversations, send message, fetch history (pagination params + ordering), mark read, upload file, feedback, handoff, … only those in scope.)

## Real-time

- Transport: <socket.io | WebSocket | SSE | polling>
- Connect: <URL, auth payload/header, query params>
- Subscribe: <room/channel naming, join event>

Inbound events:
| Event | Payload example | Widget behaviour |
| --- | --- | --- |

Outbound events:
| Event | Payload example | When |
| --- | --- | --- |

Reconnect: <backoff; history refetch window; dedupe key>

## Pagination and ordering

<cursor/page/before-id; page size; sort field; how duplicates are detected>

## File uploads

<endpoint or pre-signed flow; allowed types; max size; response shape; how the message references the file>

## Error format

```json
<exact example>
```

## Open items

- TODO(confirm-with-user): …
~~~

## Intake checklist (ask only what the requested features need)

1. Base URLs per environment; CORS allowed origins for the widget's origin.
2. Tenant identifier and where it goes (header/query/body).
3. Anonymous vs identified users; who mints the identity proof; token lifetime/refresh.
4. Init / widget-settings endpoint (does the backend own branding and behaviour settings?).
5. Conversation model: one per user or many; create vs reuse; close/reopen.
6. Send message: request/response; is the ack the stored message (with server id/timestamp)?
7. History: endpoint, pagination parameters, page size, ordering.
8. Real-time: transport, URL, auth, rooms/channels, every event name and payload example, reconnection expectations.
9. Message types the backend can send (text, markdown, html?, image, file, buttons/quick replies, cards, forms, system events, feedback requests).
10. Typing indicators, delivery/read receipts, presence: provided or not.
11. File upload: endpoint, limits, types.
12. Bot ↔ human: how the payload distinguishes them; handoff trigger.
13. Error response format and notable codes (401, 403, 409, 413, 429).
14. Rate limits.
15. Test credentials or a staging environment for real-API tests.
