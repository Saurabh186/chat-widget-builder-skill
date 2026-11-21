# Reference Repository Architecture

- Repository: `https://github.com/Walkover-Web-Solution/chat-widget`
- Branch: **`react-main`**, confirmed by the team as the production source of truth. The repo has **no `production` branch**. The default branch `main` holds an older Angular build (last commit April 2025) and is not the reference.
- Documented commit: `e8cf9d7` (2026-10-30, "Merge pull request #498 … feat/main/cobrowse-update").
- If you clone a newer commit, diff the files you rely on against this document before trusting it.

## Contents

1. What it is
2. Tech stack
3. Folder structure
4. Runtime architecture
5. Products and modes
6. Key modules
7. Build, environments, deploy
8. Conventions
9. Reusable ideas worth carrying over
10. Known issues: do not copy

## 1. What it is

The repo is a frontend-only, embeddable chat widget. It is served from its own origin and loaded into customer sites by a small loader script, which renders the chat UI in an iframe.

End users can talk to an AI bot, a human agent, or a team. All routing, ticketing, AI and persistence live in an external backend (MSG91 Hello and GTWY). The repo's own `docs/ARCHITECTURE.md` and `docs/AI_INSTRUCTIONS.md` state this boundary: the frontend must not reimplement backend logic, and backend responses are authoritative.

## 2. Tech stack

| Concern | Choice |
| --- | --- |
| Framework | Next.js 15.1.6 (App Router), React 19, TypeScript (strict, `allowJs`; several `.js` files remain) |
| Styling | Tailwind CSS 3.4 + daisyUI 4 + `@tailwindcss/typography`; MUI 6 + Emotion; per-component CSS files; inline styles for brand colour |
| State | Redux Toolkit 2, Redux Saga, redux-persist, redux-state-sync |
| HTTP | Axios (two instances with interceptors) |
| Real-time | `socket.io-client` 4 (human chat + notifications); `rtlayer-client` (AI replies) |
| Rendering | `react-markdown` + `remark-gfm`, `react-syntax-highlighter`, `linkify-html`, `frimousse` (emoji), `react-infinite-scroll-component`, `dayjs` |
| Calling | `msg91-webrtc-call` (voice), CobrowseIO (co-browsing, loaded on the host page) |
| Deploy | Cloudflare Pages via `@cloudflare/next-on-pages`; chatbot routes use the edge runtime |
| Tests | **None.** No test runner, no test files |

## 3. Folder structure

```
app/                     routes: /chatbot (redirector + layout), /chatbot/[chatbotId], /chatbot/hello,
                         /chatbotpreview (static demo), /rag (KB manager), error.tsx
components/
  AppWrapper.tsx         Redux Provider, PersistGate, MUI theme, ThemeContext, CSS vars
  Chatbot-Wrapper/       mounts hooks + ReplyProvider around Chatbot; posts interfaceLoaded
  Chatbot/Chatbot.tsx    top-level layout (sidebar, header, empty/active views)
  Chatbot/hooks/         useHelloIntegration, useHelloEffects, useChatActions, useChatEffects,
                         useRtlayerEventManager, useColor, useScreenSize, HelloVoiceService, …
  Interface-Chatbot/     Header, HeaderTab, Drawer, TextField (composer), QuickActionsMenu,
                         CallButton, EmojiSelector, ReplyPreview, NotificationPage, Markdown/, Messages/
  Hello/                 interactive, Apple, attachment, feedback, video-call renderers; call UI
  Interface-*/, Grid/    generative-UI components rendered from AI JSON
  FormComponent.tsx      pre-chat form
config/                  api.ts (GTWY), helloApi.ts (HELLO), ragApi.ts
hooks/                   CORE/ (shared postMessage dispatcher), HELLO/, GTWY/ (embed event handlers),
                         socketManager.js, socket.js, socketEventHandler.ts, socketEventEmitter.ts,
                         notifications/, coBrowser/
store/                   chat/, hello/, interface/, appInfo/, draftData/, index.js, combineReducer.js
utils/                   utilities.js (storage, ids, linkify, dates), safeStorage.ts, interceptors,
                         emitEventsToParent/, dataConvertWrappers/ (message normalization), enums.ts
hoc/                     theme.js (MUI theme), addUrlDataHoc.tsx (inject session ids)
public/                  host-side loaders: chat-widget(-dev|-local).js, chatbot(-dev|-local).js,
                         rag-*.js, urlMonitor.js, chat-widget-style.css, chatbot-style.css
docs/                    ARCHITECTURE, AI_INSTRUCTIONS, MODULE_* feature docs
```

## 4. Runtime architecture

```
Host page                                   Widget origin (Next.js app in iframe)
─────────                                   ──────────────────────────────────────
<script chat-widget.js>                     /chatbot/hello
  ├─ launcher button + badge                  ├─ CORE postMessage dispatcher → HELLO/GTWY handlers
  ├─ container + <iframe sandbox allow=mic>   ├─ Redux store (persist + cross-tab sync)
  ├─ window.chatWidget API                    ├─ axios → HELLO REST / GTWY REST
  └─ postMessage bridge  ◀──────────────────▶ ├─ socket.io (chat) + socket.io (notifications)
     (origin + source checked on host side)   ├─ rtlayer (AI replies)
                                              └─ React UI (header, drawer, list, composer)
```

The first message goes over HTTP; replies and live updates arrive over sockets.

## 5. Products and modes

- **HELLO** (`isHelloUser: true`): human or helpdesk chat. Supports multiple conversations ("channels"), agents and teams, bot-first handoff (server-driven `assign` events), CSAT, attachments, interactive messages, voice calls, push notifications and tickets.
- **GTWY** (`isHelloUser: false`): AI chatbot with threads and sub-threads, streaming-like updates over RTLayer, function-call progress, suggestions, generative UI, and model/bridge switching.
- **RAG**: an embeddable knowledge-base document manager. It is not chat.
- **Demo widget** (HELLO with `demo_widget`): a trial bot. It sends the last 5 messages as context.

## 6. Key modules

Each module, the file it lives in, and the reference doc that covers it.

| Module | File | Covered in |
| --- | --- | --- |
| Loader + public API | `public/chat-widget.js` | `widget-configuration.md` §2–3 |
| postMessage protocol | the loader + `hooks/*/eventHandlers`, `utils/emitEventsToParent` | `widget-configuration.md` §6 |
| HELLO API | `config/helloApi.ts` | `api-integration.md` A2–A3 |
| Socket manager | `hooks/socketManager.js` | `api-integration.md` A4 |
| AI transport | `useRtlayerEventManager.tsx` | `api-integration.md` A5 |
| Send/receive orchestration | `useHelloIntegration.ts`, `useHelloEffects.ts`, `useChatActions.ts` | `chat-flow.md` A4–A5 |
| Message normalization | `utils/dataConvertWrappers/makeGenericDataFormatUtility.tsx` | `chat-flow.md` A3 |
| Theming | `AppWrapper.tsx`, `hoc/theme.js`, `utils/themeUtility.js`, `useColor.ts` | `design-guidelines.md` §2 |
| Safe storage | `utils/safeStorage.ts`, `utils/utilities.js` | `widget-configuration.md` §7 |

## 7. Build, environments, deploy

- Scripts:
  - `dev`: `next dev --turbopack`
  - `build`
  - `lint`: `next lint`
  - `build:pages`: Cloudflare
  - `preview` / `deploy`: both call the missing `pages:build` script.
- `next.config.ts`:
  - `reactStrictMode: false`
  - **ESLint and TypeScript errors are ignored during builds**
  - image `remotePatterns` allows any host.
- ESLint:
  - `.eslintrc.json` disables `rules-of-hooks`, `exhaustive-deps` and `jsx-key`.
  - `.eslintignore` ignores `app/` and `components/`.
  - Flat config extends `next/core-web-vitals` and `next/typescript`.
- Environments:
  - Env vars are `NEXT_PUBLIC_*` (listed in `widget-configuration.md` §8).
  - There is a separate loader file for each environment.
  - `?env=stage` switches the HELLO API and sockets to stage hosts.
  - No `.env.example`.

## 8. Conventions

- Component folders are PascalCase with prefixes (`Interface-Chatbot`, `Chatbot-Wrapper`), with some inconsistencies. Hooks are `useX.ts`. The `@/*` path alias is used.
- Components are wrapped as `React.memo(addUrlDataHoc(Comp, [...]))`, which injects session and tab ids. Selectors go through `useCustomSelector` (deep compare).
- Host communication goes only through `emitEventToParent` / `window.parent.postMessage`.
- Storage is accessed via `getLocalStorage`/`setLocalStorage`, which are auto-prefixed with `${widgetToken}_`, and via `safeStorage`.
- `docs/AI_INSTRUCTIONS.md` rules for this codebase:
  - Reuse existing dependencies, Redux slices and sagas, and the single socket manager.
  - No new state libraries and no extra socket connections.
  - No hardcoded URLs or tokens.
  - Keep changes minimal and incremental.
  - Use descriptive names.
  - Update docs together with behaviour changes.

## 9. Reusable ideas worth carrying over

- **Loader + iframe split**, with launcher, badge and resizing on the host side and the full app isolated in the iframe.
- **Host-side `postMessage` filter:** accept a message only when the origin is in the allowlist **and** `event.source === ourIframe.contentWindow`.
- **Namespaced storage** per widget token and per user, plus an in-memory fallback when storage is blocked.
- **Anonymous → identified history merge** at first login.
- **Client-generated message id** sent with the request (`widget_msg_id`) for echo matching.
- **Pre-subscribe before the first send**, so the first bot reply is not missed on a brand-new conversation.
- **Socket manager design:** idempotent subscriptions, re-subscribe on connect, an emit queue while disconnected, and a history refetch on reconnect.
- **Read-receipt conditions:** open, visible, not minimized, debounced.
- **Badge and greeting bubble** driven from the iframe through typed events.
- **Contrast-aware text colour** on the brand colour, plus derived tints.
- **Markdown without raw HTML** (no `rehype-raw`), and code blocks with a copy button.
- **Mobile behaviour:** full screen below 600 px; drawer overlay below 1024 px.

## 10. Known issues: do not copy

**Security**
- Parts of the reference fall below the bar in `security-guidelines.md`: credential handling, HTML rendering of message content, `postMessage` validation, and how tokens are transported. Do not copy its credential values, message-rendering code, or message-bridge handlers. Build these fresh from `security-guidelines.md` (§1, §3, §4, §7).
- The HELLO loader patches host-page globals (`window.open`, `history.*`) for its URL monitor and domain tracking. Generated widgets must not touch host globals.

**Reliability**
- No message status, no retry, no deduplication, and no offline UI. A HELLO send failure leaves a ghost message; an AI send failure deletes the user's text.
- No token refresh. The GTWY 401 handler wipes **all** localStorage and redirects. The HELLO calls have no 401 handling.
- Error toasts come from inside the API layer, so callers cannot decide the UI.

**Code health**
- Three styling systems, plus undefined utility classes.
- Lint rules disabled, build errors ignored, no tests.
- Dead code: the local `chatReducer.ts`, duplicate `ChatbotPreview` components, uncalled API functions.
- Misspellings that are part of the contract (`NEXT_PUBLIC_NOTIFICATOIN_SOCKET_URL`, `hello-widget-anonymoud-uuid`, `assests/`). Keep them only when customizing the reference itself.

**Broken config:** see `widget-configuration.md` §9.
