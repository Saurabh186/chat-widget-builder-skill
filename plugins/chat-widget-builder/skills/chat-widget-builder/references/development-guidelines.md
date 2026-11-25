# Development Guidelines

## Contents

1. Mandatory rules
2. Technology selection
3. Output format selection
4. Code organization
5. Real-time transport layer
6. Customizing an existing widget
7. Dependency policy
8. Quality gates

## 1. Mandatory rules

1. Read `repository-architecture.md` and `chat-flow.md` before writing code. Re-inspect the reference repo only if the user says it changed or you need detail not covered.
2. Understand the existing chat flow (and, when customizing, the user's existing code) before changing anything.
3. Write minimal, clean, production-ready code. No placeholder logic left in shipped paths.
4. Reuse existing logic in the target project; do not duplicate helpers that already exist.
5. Add no dependency without a clear reason (section 7).
6. Create no file or component without a reason; no empty files.
7. No speculative fallbacks, feature flags or abstraction layers the requirements do not need.
8. Follow the chosen framework's current conventions (React function components + hooks; Angular standalone components + signals for Angular 17+; Vue 3 Composition API with `<script setup>`).
9. Separate UI, conversation state, API/transport and configuration (section 4).
10. Never expose secrets (`security-guidelines.md`).
11. Do not modify unrelated files or behaviour.
12. Fix every lint, type, build and test error your change introduces.
13. Ask before major architectural decisions: framework, state library, output formats, transport (socket vs SSE vs polling), adding a backend, replacing an existing component, breaking a public API.
14. Verify the final output (build, tests, demo) before saying it is done; report what was not verified.

Work like a senior engineer: the simplest design that meets the requirements, readable names that say what a thing does, small functions, comments only where intent is not obvious.

## 2. Technology selection

Decide from the integration environment and feature scope, then confirm with the user:

| Situation | Recommended stack | Reason |
| --- | --- | --- |
| Embed in any website via `<script>`, small/medium feature set | Vanilla TypeScript UI in an iframe **or** a Web Component with Shadow DOM, bundled with Vite (IIFE loader + app) | Small bundle, no framework clash with the host, style isolation |
| Embed in any website, large feature set (many message types, calls, tabs) | Loader script + iframe app in React (Vite or Next.js static export) | Mirrors the reference's proven split; host page stays untouched |
| Host app is React / Next.js | React + TypeScript component (`<ChatWidget/>`) + `useChat` hook | Native integration |
| Host app is Angular | Standalone component + `ChatService` (signals) | Matches host DI |
| Host app is Vue / Nuxt | Vue 3 SFC + `useChat` composable | Native integration |
| Needed in several frameworks | Framework-free TS core (API client, transport, conversation store) + thin adapters, or one Web Component | One chat implementation |
| Customizing the reference (`react-main`) itself | Keep Next.js 15 + Redux Toolkit/Saga + its folder layout | Minimal-change rule, the repo's own `docs/AI_INSTRUCTIONS.md` forbids replacing core infra |

Notes:

- Do not force the reference's stack on new projects. The reference ships Next.js + MUI + Tailwind + daisyUI + Redux Toolkit + Saga + redux-persist + redux-state-sync; that is heavy for a simple widget. For a new widget prefer one styling system and the smallest state approach that works (a reducer or a small store).
- Iframe vs Shadow DOM: an iframe (the reference's choice) gives full CSS/JS isolation and its own origin, at the cost of a `postMessage` bridge. Shadow DOM is lighter and simpler for small widgets but shares the host's JS realm. Recommend iframe when the widget is large, needs its own storage origin, or must survive hostile host CSS; Shadow DOM otherwise. Ask.
- Markdown: `react-markdown` + `remark-gfm` (as in the reference) in React; `marked` + `DOMPurify` elsewhere. Never render model or agent output as raw HTML without sanitizing.
- Real-time: use whatever the user's backend offers (section 5). Never add a socket library the backend does not speak.

## 3. Output format selection

Generate only what the user needs:

| Format | Build output | When |
| --- | --- | --- |
| Standalone web app | App bundle + `index.html` (full-page chat) | Hosted support page, or a demo |
| Embeddable script | `chat-widget.js` loader exposing `window.<Name>.init(config)` + iframe app (or one IIFE Web Component bundle) | Third-party sites, CMS, no build step |
| npm package | ES module + `.d.ts`, `exports` map, framework as `peerDependencies` | Teams installing via npm |
| In-app integration | Component/files added to the user's repo | User has an existing frontend |

A working demo page is always delivered.

Script-embed contract (adapt names to the user's brand; only include what is implemented):

```js
const chat = MyChat.init({
  widgetToken: 'public-widget-id',     // public tenant identifier only, never a secret
  user: { id, name, email, signature }, // optional; signature/JWT minted by the host backend
  theme: { primaryColor: '#3b5bdb' },
  position: 'right',
  onReady() {}, onOpen() {}, onClose() {},
  onMessage(msg) {}, onUnreadChange(count) {}, onError(err) {},
});
chat.open(); chat.close(); chat.toggle();
chat.identify(user); chat.logout(); chat.destroy();
```

Fail fast on missing required config with a clear `Error`.

## 4. Code organization

Responsibilities, whatever the framework:

| Layer | Contains | Must not contain |
| --- | --- | --- |
| `config` | Typed config schema, defaults, validation | DOM, HTTP, sockets |
| `api` | Endpoint map, request builder, `fetch`/axios + timeout + abort, response/error adapters | UI strings, state |
| `realtime` | One transport manager (connect, auth, subscribe, reconnect, event → callback) | UI, business routing decisions |
| `state` | Conversation store: messages keyed by id, ordering, pending/failed, unread, session | `fetch`, DOM |
| `ui` | Launcher, window, header, message list, bubbles, composer, styles, a11y | Endpoint paths, storage keys |
| `embed` | Loader script, iframe/Shadow root creation, `postMessage` bridge, public API | Chat logic |

Write `state` and `realtime` so they are testable without a browser (inject the API client, transport and clock).

## 5. Real-time transport layer

Implement exactly the mechanism the backend provides:

| Backend offers | Implement |
| --- | --- |
| socket.io server | `socket.io-client`, one shared connection, auth in `auth`/query as the backend specifies |
| Plain WebSocket | Native `WebSocket` + reconnect with capped exponential backoff + heartbeat if the server expects one |
| SSE | `EventSource` (or `fetch` stream if auth headers are needed) |
| Only HTTP | Polling with backoff while the window is open; slower or paused when closed/hidden |
| Vendor SDK (e.g. RTLayer) | That SDK, wrapped behind the same manager interface |

Rules (from the reference's own `AI_INSTRUCTIONS.md`, and good practice):

- Exactly one connection per transport, owned by one manager; UI never opens sockets.
- On reconnect, refetch recent history and merge by message id, so nothing is lost or duplicated (the reference does this; see `chat-flow.md`).
- Tear down listeners and connections on `destroy()` / unmount / logout.
- Treat socket payloads as backend-authoritative; do not infer routing (bot vs agent) on the client beyond what the payload says.

## 6. Customizing an existing widget

1. Get the code: path on disk, repo URL or ZIP. Do not guess.
2. Inspect: framework, build, how config reaches the widget (script attributes, `init` object, `postMessage`, backend widget settings), where styles live, transport, tests, lint setup, how the widget is consumed.
3. Run the existing build/lint/tests first and record the baseline, including pre-existing failures.
4. Map each requested change to exact files. Present that map in the plan.
5. Make the smallest change that satisfies each requirement. Keep public APIs (init keys, window methods, `postMessage` types, emitted events) backward compatible unless the user approves a break; if breaking, document migration.
6. Re-run build/lint/tests; compare with the baseline; fix only what your change broke and report pre-existing issues separately.
7. Update only the docs that describe changed behaviour.

When customizing the reference (`react-main`) specifically:

- Follow its `docs/AI_INSTRUCTIONS.md`: reuse Redux slices/sagas, the central socket manager, axios interceptors and env vars; no new state libraries, no extra socket connections, no hardcoded URLs. Fill the env vars from `msg91-environments.md` rather than asking the user for them.
- Much appearance and behaviour comes from the backend's widget settings (fetched at init), not from code; check `widget-configuration.md` before changing code for something that is already a setting.
- Its ESLint ignores `app/` and `components/` and disables `rules-of-hooks`; run `npx tsc --noEmit` as the real check and do not add new hook-rule violations.
- It has no test suite: add tests for what you change (see `testing-guidelines.md`).

## 7. Dependency policy

Before adding a package, confirm: (a) the platform or existing deps cannot do it in a few lines, (b) it is maintained, (c) bundle-size impact is acceptable for an embed. Typically justified: the framework, a markdown renderer + sanitizer, the transport client the backend requires, an emoji picker if emoji is in scope, a test runner, Testing Library, Playwright, MSW. Usually unjustified in a new widget: a component library inside an embed, two styling systems, a global state library for one widget, lodash, moment.

Use versions that exist; check with the package manager instead of guessing version numbers.

## 8. Quality gates

Run what the project defines; add only what is missing and needed:

- Type check (`tsc --noEmit` or framework equivalent), strict mode on for new projects.
- Lint with the framework's recommended config.
- Unit + integration tests (`testing-guidelines.md`).
- Production build succeeds; check the embed loader size and that it works from a plain `<script>` on a page served over HTTP.
- Demo page runs; full send/receive flow works against the mock (and the real API when credentials exist).
- `.env.example` complete; no secrets in the repo (`git grep -nE "(api[_-]?key|secret|token|authorization)"` and review hits before handoff).
