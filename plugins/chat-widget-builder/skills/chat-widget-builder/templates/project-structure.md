# Project Structure Templates

Starting points, not mandates. Remove anything the requirements do not need (no empty folders, no unused files). Rename `chat-widget` / `cw-` to the user's chosen name/prefix.

## Contents

1. Embeddable widget: loader + iframe app (React + Vite)
2. Embeddable widget: single Web Component (vanilla TS)
3. React component / npm package
4. Vue 3 and Angular variants
5. Documentation set (all projects)
6. README skeleton

## 1. Embeddable widget: loader + iframe app (React + Vite)

Mirrors the reference's split (host-side loader script + chat UI in an iframe) without its Next.js/Redux weight.

```
chat-widget/
├── package.json             scripts: dev, build, preview, test, typecheck, lint, test:e2e
├── tsconfig.json            strict: true
├── vite.config.ts           two inputs: app (index.html) + loader (lib, IIFE)
├── .env.example             VITE_API_BASE_URL, VITE_SOCKET_URL (public values only)
├── index.html               iframe app entry
├── demo/index.html          host-page demo loading dist/chat-widget.js (?mock=1 supported)
├── src/
│   ├── loader/
│   │   ├── index.ts         window.ChatWidget = { init, open, close, toggle, identify, logout, destroy }
│   │   ├── dom.ts           container, launcher button, iframe, unread badge, responsive sizing
│   │   └── bridge.ts        postMessage host side: typed messages, origin checks
│   ├── app/
│   │   ├── main.tsx         iframe entry; receives config via bridge
│   │   ├── bridge.ts        postMessage iframe side
│   │   └── App.tsx
│   ├── config.ts            WidgetConfig type, defaults, validateConfig()
│   ├── api/
│   │   ├── client.ts        fetch + timeout + abort + auth header + error normalization
│   │   └── endpoints.ts     endpoint map + adapters (from the user's confirmed contract)
│   ├── realtime/
│   │   └── transport.ts     single connection manager (socket.io / WebSocket / SSE / polling)
│   ├── state/
│   │   ├── conversation.ts  reducer: messages by id, order, pending/failed, unread, typing
│   │   └── session.ts       session ids/tokens, namespaced safe storage
│   ├── ui/
│   │   ├── ChatWindow.tsx   header, MessageList, Composer
│   │   ├── Header.tsx
│   │   ├── MessageList.tsx  history pagination, date groups, auto-scroll, aria-live
│   │   ├── MessageBubble.tsx  per-type renderers (text/markdown/image/file/buttons)
│   │   ├── Composer.tsx     textarea, send, attachments, emoji (only if in scope)
│   │   ├── states.tsx       loading / empty / error / offline
│   │   └── styles.css       design tokens + components
│   └── mocks/               MSW handlers + fake transport for tests and ?mock=1 (not in prod bundle)
├── tests/
│   ├── unit/                conversation.test.ts, client.test.ts, config.test.ts, transport.test.ts
│   ├── integration/         chat-window.test.tsx (Testing Library + MSW + fake transport)
│   └── e2e/                 embed.spec.ts, chat-flow.spec.ts (Playwright)
└── docs/                    see section 5
```

## 2. Embeddable widget: single Web Component (vanilla TS)

For small feature sets where an iframe is overkill.

```
chat-widget/
├── vite.config.ts           library mode: formats ['es', 'iife'], name 'ChatWidget'
├── index.html               demo
├── src/
│   ├── index.ts             init(), <chat-widget> registration, types
│   ├── config.ts
│   ├── api/client.ts, api/endpoints.ts
│   ├── realtime/transport.ts
│   ├── state/conversation.ts, state/session.ts
│   ├── ui/
│   │   ├── chat-widget.ts   custom element, Shadow DOM, render(state)
│   │   ├── templates.ts     DOM builders (textContent only; sanitized markdown)
│   │   └── styles.css       imported with ?inline into the shadow root
│   └── mocks/
├── tests/
└── docs/
```

## 3. React component / npm package

```
chat-widget-react/
├── package.json             react/react-dom as peerDependencies; "exports", "types", "files": ["dist"]
├── vite.config.ts           library mode + vite-plugin-dts (or tsc --emitDeclarationOnly)
├── src/
│   ├── index.ts             export { ChatWidget, useChat } and types
│   ├── config.ts
│   ├── api/, realtime/, state/   framework-free core
│   ├── hooks/useChat.ts     binds core to React, cleanup on unmount
│   ├── components/          ChatWidget, Launcher, ChatWindow, Header, MessageList, MessageBubble, Composer
│   ├── styles/chat-widget.css   tokens prefixed --cw-*
│   └── demo/main.tsx
├── tests/
└── docs/
```

## 4. Vue 3 and Angular variants

Vue 3: same core (`api/`, `realtime/`, `state/`), `composables/useChat.ts`, SFCs `ChatWidget.vue`, `ChatWindow.vue`, `MessageList.vue`, `MessageBubble.vue`, `Composer.vue`.

Angular 17+: `chat-widget.component.ts` (standalone, OnPush, signals), `chat-api.service.ts` (HttpClient), `chat-realtime.service.ts`, `chat-state.service.ts` (signals), `chat-widget.config.ts` (InjectionToken). For a script embed, wrap with `@angular/elements`; tell the user the bundle-size cost.

## 5. Documentation set (all projects)

Only describe behaviour that exists. Combine files for small projects, but cover every topic.

| File | Must contain |
| --- | --- |
| `README.md` | What it is, implemented features, quick start, scripts, links to docs |
| `.env.example` | Every env var with placeholder + comment; mark public vs server-only |
| `docs/API.md` | Filled from `templates/api-documentation.md` |
| `docs/CONFIGURATION.md` | Filled from `templates/widget-configuration.md` |
| `docs/INTEGRATION.md` | Script tag, npm, framework usage (only built formats); public methods; events/callbacks; identity-proof step for the host backend |
| `docs/CUSTOMIZATION.md` | Theme tokens table, logo/name, texts/i18n, position, custom CSS, examples |
| `docs/ARCHITECTURE.md` | Layer diagram, message lifecycle (pending → sent → delivered/read/failed), transport + reconnect, file map, decisions |
| `docs/TESTING.md` | How to run tests + actual results (`testing-guidelines.md` §7) |
| `docs/DEPLOYMENT.md` | Build commands, outputs, hosting the loader + app (CDN, cache headers, versioned filenames, SRI command), CORS and allowed origins |
| Demo | `demo/index.html` or `index.html`, reachable from `npm run dev` and the production preview; `?mock=1` if mocks exist |

## 6. README skeleton

~~~markdown
# <Widget name>

<One sentence: what it does and for whom.>

## Features
- <Only implemented: real-time transport, history, attachments, markdown, theming, formats>

## Quick start
```bash
npm install
cp .env.example .env   # fill in public values
npm run dev            # demo at http://localhost:5173 (add ?mock=1 for mock backend)
```

## Usage
### Script tag
### npm
### <Framework>

## Configuration
Most used options → docs/CONFIGURATION.md

## Security
- No secrets in this package.
- Identified users need a signature/JWT from your backend: docs/INTEGRATION.md#identity

## Scripts
| Script | Purpose |

## Docs
API · Configuration · Integration · Customization · Architecture · Testing · Deployment
~~~
