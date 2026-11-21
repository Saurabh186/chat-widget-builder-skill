# Security Guidelines

Apply to every generated or customized widget. A chat widget runs on third-party pages, renders text written by strangers, bots and LLMs, and holds conversation history, so it is an XSS and data-leak surface by default.

## Contents

1. Secrets and credentials
2. User identity and authorization
3. Rendering untrusted content (XSS)
4. Embedding and `postMessage`
5. File uploads
6. Storage and privacy
7. Transport
8. Logging and errors
9. Pre-handoff checklist

## 1. Secrets and credentials

- Frontend code is public. Only **public** identifiers belong in it or in `NEXT_PUBLIC_*` / `VITE_*` env vars: widget/tenant ids, public API base URLs, socket URLs.
- Never put API secrets, admin tokens, signing keys, LLM provider keys, or user passwords in frontend code, config defaults, demo pages, tests or git history. If the user's backend only offers a secret-key API, stop and explain that a small server-side proxy (owned by the user) is required; ask before building one.
- `.env.example` holds placeholders only and marks each var as public or server-only.
- Do not copy hardcoded values from the reference repository (see `repository-architecture.md` → known issues).

## 2. User identity and authorization

- Identified users: the host page passes identity **plus a proof minted by the host backend** (signed JWT, HMAC signature of the user id, or a session token) as the backend specifies. A bare client-supplied `email`/`unique_id` lets anyone read anyone's conversations if the backend trusts it; flag this to the user when their API works that way.
- Anonymous users: use the backend-issued anonymous id/token; never let the client choose another user's id.
- Store auth tokens with the shortest lifetime the backend allows; clear them on `logout()`.
- On 401/403: clear the session and re-initialize (or surface an error); do not loop retries.
- Never derive authorization (which conversations a user can see, agent vs bot routing) on the client; the backend decides.

## 3. Rendering untrusted content (XSS)

Every message body, sender name, agent avatar URL, link preview, button label, form label and widget setting from the backend is untrusted.

- Render text as text (`textContent`, framework interpolation). No `innerHTML` / `dangerouslySetInnerHTML` / `v-html` / `[innerHTML]` with unsanitized input.
- Markdown: render with a library that does **not** pass raw HTML through (`react-markdown` without `rehype-raw`, or `marked` + `DOMPurify`). If HTML in messages is a requirement, sanitize with DOMPurify using an allowlist.
- Links: allow only `http:`, `https:`, `mailto:`, `tel:` schemes; add `target="_blank" rel="noopener noreferrer"`; block `javascript:` and `data:` URLs.
- Images/media: only `https:` (or the user's CDN) URLs; set `referrerpolicy="no-referrer"` where privacy matters; size-constrain to avoid layout abuse.
- Code blocks: render as text inside `<pre><code>`; syntax highlighting libraries must not eval.
- Interactive payloads (buttons, carousels, forms): treat labels as text; validate action types against an allowlist before acting (e.g. open URL, send postback).
- Host-supplied custom CSS: inject as a `<style>` text node into the widget's own root only; never inject host-supplied scripts.

## 4. Embedding and `postMessage`

- Loader → iframe: post with an explicit `targetOrigin` (the widget app's origin), never `'*'` when the payload contains identity or tokens.
- Iframe → host: post to the host origin known at init (passed in the iframe URL or handshake), not `'*'`, when the payload contains user or conversation data.
- Every `message` listener checks `event.origin` against the expected origin **and** `event.source` against the expected window, and validates `event.data.type` against a known list. Ignore everything else.
- Give the iframe only the permissions it needs (`allow="microphone; camera"` only if calling is in scope; `clipboard-write` only if copy is used).
- Recommend a CSP for the widget app (`script-src 'self'`, `frame-ancestors` per the user's allowed domains if they restrict embedding).
- The loader must not read or modify the host page beyond creating its container.

## 5. File uploads

- Enforce an allowlist of MIME types **and** extensions, and a max size, on the client for UX; the backend must enforce the same (say so in docs).
- Never trust the filename for display without escaping; never render uploaded SVG/HTML inline.
- Use the backend's upload endpoint or pre-signed URL flow; no storage credentials in the client.
- Show progress, allow cancel (abort the request), and handle failure without leaving a phantom message.

## 6. Storage and privacy

- Store only what is needed to resume a session (session/anon id, conversation id, minimal UI prefs). Avoid persisting full message history to `localStorage` unless the user asks; if persisted, clear on logout.
- Namespace storage keys with the widget prefix and the tenant id so two widgets on one page do not collide.
- Storage can be blocked (third-party iframe, Safari ITP, private mode): wrap access in try/catch and degrade to memory (the reference has a `safeStorage` wrapper for this reason).
- Do not send conversation content to analytics. Do not put PII in URLs or query strings.
- Cross-tab sync (BroadcastChannel / storage events) must carry only non-sensitive state or be same-origin only.

## 7. Transport

- HTTPS/WSS only in production.
- Socket auth via the mechanism the backend specifies (auth payload/header); avoid tokens in socket URLs when the backend supports alternatives, since URLs end up in logs.
- Timeouts on every HTTP request; abort on unmount.
- Reconnect with capped backoff; never hammer the server.
- Validate inbound socket payload shape before using it (ignore unknown event types).

## 8. Logging and errors

- No `console.log` of tokens, user identity, or message content in production builds. Strip or guard debug logging.
- Error messages shown to users are generic and actionable; raw server errors stay out of the UI.
- Do not include message content or tokens in error reports sent to monitoring.

## 9. Pre-handoff checklist

- [ ] `git grep` for keys/tokens/secrets: no hits in source, config, tests, demo
- [ ] No unsanitized HTML sinks (`grep -rn "innerHTML\|dangerouslySetInnerHTML\|v-html"` reviewed)
- [ ] Link scheme allowlist + `rel="noopener noreferrer"`
- [ ] `postMessage` origin checks both directions; no `'*'` with sensitive payloads
- [ ] Upload type/size limits enforced client-side and documented as required server-side
- [ ] Identity proof (JWT/HMAC) flow documented for the host backend, or the risk flagged to the user
- [ ] Storage keys namespaced; cleared on logout; safe when storage is blocked
- [ ] No sensitive logging in production build
