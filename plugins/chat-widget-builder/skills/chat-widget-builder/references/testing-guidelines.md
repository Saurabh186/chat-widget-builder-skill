# Testing Guidelines

The reference repository (`react-main`) has **no test suite** and no test runner configured. Every generated or customized widget must ship with tests, and the final report must state exactly what was run.

## Contents

1. Honesty rules
2. Tooling per stack
3. Required scenario matrix
4. Mocking strategy (HTTP and real-time)
5. Responsive and integration checks
6. Verification commands
7. Test report format

## 1. Honesty rules

- Never say tests passed unless you ran them in this session and saw the output.
- Label every result **Mocked** (fake API/socket) or **Real API** (user's backend). Without credentials or backend access only mocked tests exist; say so.
- If the environment cannot install packages or run a browser, say which checks were skipped and give the exact commands for the user to run.
- Report failures with their output; do not delete or skip failing tests to get green.

## 2. Tooling per stack

| Stack | Unit / integration | E2E |
| --- | --- | --- |
| Vanilla TS / Web Component (Vite) | Vitest + `happy-dom`/`jsdom` + `@testing-library/dom` | Playwright |
| React (Vite) | Vitest + `@testing-library/react` + `@testing-library/user-event` | Playwright |
| Next.js | Vitest (or Jest via `next/jest`) + Testing Library | Playwright |
| Vue 3 | Vitest + `@vue/test-utils` or `@testing-library/vue` | Playwright |
| Angular 17+ | The project's configured runner + `TestBed`, `HttpTestingController` | Playwright |
| Existing project | Its existing runner; do not add a second one | Its existing E2E tool, else Playwright if E2E is required |

HTTP mocking: MSW (`msw`) for integration tests and the demo's mock mode. Real-time mocking: a fake transport injected into the store (preferred), `mock-socket` for native WebSocket, or a local `socket.io` server in integration tests. Timers: fake timers + an injectable clock.

## 3. Required scenario matrix

Cover each in-scope row at the lowest level that proves it. Skip rows for features not built and say so in the report.

| # | Scenario | Assert |
| --- | --- | --- |
| 1 | Widget initialization | Valid config mounts launcher; missing required config throws a clear error; `onReady` fires once |
| 2 | Launcher | Click/Enter/Space toggles window; `aria-expanded` updates; unread badge shows count and clears on open |
| 3 | Open / close | Window opens/closes via launcher, header close and public API; focus moves into window and back to launcher; Esc closes (if specified) |
| 4 | Send message | Correct request (URL, method, headers, body from the confirmed contract); optimistic bubble appears with pending state; becomes sent on ack |
| 5 | Send failure | API error / network error → bubble marked failed, retry works, text not lost; `onError` called |
| 6 | Receive message (real-time) | Inbound event appends to the right conversation in order; unread increments when closed |
| 7 | Conversation history | Initial load renders newest page; scroll-up loads older page; scroll position preserved; ordering by server timestamp/id |
| 8 | Session handling | Reload resumes the same conversation; `identify()` switches user; `logout()` clears storage and state |
| 9 | Duplicate handling | Same message from HTTP ack and socket → one bubble; replayed events after reconnect → no duplicates |
| 10 | Ordering | Out-of-order arrivals sorted correctly; date separators correct across days/time zones |
| 11 | Disconnect / reconnect | Offline indicator shown; reconnect with backoff; history refetched and merged; queued sends handled per spec |
| 12 | Loading states | Skeleton/spinner on init and history; send button disabled while empty; no double send on rapid Enter |
| 13 | Empty state | No messages → welcome message / starter questions render |
| 14 | Error states | Init failure and history failure show retryable error; 401 clears session |
| 15 | Typing / presence | Typing indicator appears and times out; agent online/offline reflected (only if backend provides it) |
| 16 | Attachments | Allowed type/size uploads with progress; disallowed type/oversize rejected with message; failed upload recoverable |
| 17 | Input validation | Trims whitespace; blocks empty; enforces max length; Shift+Enter newline vs Enter send |
| 18 | Rich content safety | `<script>`/`<img onerror>` in message renders as text; `javascript:` links not clickable; markdown renders allowed syntax only |
| 19 | Message types in scope | Each supported type (text, markdown, image, file, buttons/quick replies, feedback, system) renders and acts correctly |
| 20 | Auth | Correct auth header/proof sent; token refresh or re-init per contract |
| 21 | Theming / config | Brand color, logo, name, position, texts applied; contrast of text on primary color acceptable |
| 22 | Accessibility basics | Launcher and controls have accessible names; message list is a live region (`aria-live="polite"`); keyboard-only flow works |
| 23 | Integration | Script: built loader on a plain HTML page mounts and `destroy()` cleans up DOM/listeners/sockets. npm: imports and types resolve. In-app: renders in the host app's test setup |
| 24 | `postMessage` bridge (iframe builds) | Messages from wrong origin ignored; known types handled; unknown types ignored |

## 4. Mocking strategy (HTTP and real-time)

- Build mock handlers and socket events from the **user's confirmed examples**. Do not invent fields.
- One handler set per scenario (success, empty history, paginated history, 401, 500, network failure, slow response).
- A fake transport exposes `emit(event, payload)`, `disconnect()`, `reconnect()` so tests drive real-time paths deterministically.
- The demo page may offer `?mock=1` using the same handlers and a scripted fake transport (e.g. echoes a bot reply after a delay) so stakeholders can click through every state without a backend. Mock code must be excluded from production bundles.

## 5. Responsive and integration checks

- Viewports 360×740, 768×1024, 1280×800: launcher not overlapping critical host UI corners; window fits; on mobile the window goes full-screen (or as specified); composer stays visible with the on-screen keyboard (use `dvh` units / `visualViewport`); no horizontal scroll; touch targets ≥44 px.
- Screenshot open/closed/empty/error states at mobile and desktop when visual review matters; list paths in the report.
- Embed check: a plain HTML page served over HTTP loading the built loader; verify host CSS does not leak in (test with an aggressive host stylesheet like `* { font-size: 30px !important }`).

## 6. Verification commands

Run whichever exist (adapt to the package manager):

```bash
npm run lint
npx tsc --noEmit
npm test
npm run build
npx playwright test
```

Then serve the demo and exercise send/receive at least once in mock mode, and against the real API when credentials exist.

## 7. Test report format

Write `docs/TESTING.md` in the generated project and summarize it in the final message:

```markdown
## Test results: <date>

Environment: Node <version>, <OS>, <browser(s)>

| Check | Command | Result | Mode |
| --- | --- | --- | --- |
| Lint | npm run lint | ✅ pass | n/a |
| Types | npx tsc --noEmit | ✅ pass | n/a |
| Unit/integration | npm test | ✅ 52 passed | Mocked |
| E2E | npx playwright test | ⚠️ not run: browsers not installed | Mocked |
| Build | npm run build | ✅ pass (loader 6.1 kB gzip) | n/a |
| Real API | manual | ❌ not run: no test credentials | Real API |

Scenario coverage: matrix rows → test file names. Rows out of scope: …
Known gaps: …
```
