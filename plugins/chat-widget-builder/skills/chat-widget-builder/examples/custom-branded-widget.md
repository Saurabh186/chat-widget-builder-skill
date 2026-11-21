# Example: Customizing an existing chat widget

Two condensed runs of the customize workflow. The pattern is the same in both: inspect the code, record a baseline, map each change to files, make the smallest backward-compatible diff, verify, and update docs. Verification numbers illustrate the report format; in a real run, report only what was executed.

## Contents

- Scenario A: Rebrand the reference itself (`react-main`, Next.js)
- Scenario B: A generated React widget → dark mode + image attachments

## Scenario A: Rebrand the reference itself (`react-main`, Next.js)

**Request:**
- Color the launcher with the brand color.
- Let the composer placeholder be configurable.
- Harden how agent messages are rendered (a security review asked for it).

The user owns a fork and authorized changes to it.

### Inspection findings Claude reported before planning

- Branch `react-main` at `e8cf9d7` matched the documented commit, so `references/` applied as written.
- The brand color is resolved **inside the iframe**:
  - from `sdkConfig.customTheme` (`hooks/HELLO/eventHandlers/embeddingScript/embeddingScriptEventHandler.ts`),
  - otherwise from backend `widget-info.primary_color` (`components/Chatbot/hooks/useHelloEffects.ts`).
  - The launcher lives on the **host page** (`public/chat-widget.js`) and uses a fixed blue SVG fill, so it never learns the brand color.
- The placeholder is hardcoded as "Message AI Assistant..." (`components/Interface-Chatbot/ChatbotTextField.tsx`).
- Claude listed every component that renders message content as HTML, and the helper they share (`linkify` in `utils/utilities.js`). The list stayed in the private plan, not in public docs.
- **Baseline:**
  - `npx tsc --noEmit`: recorded the existing errors (the build ignores TS errors).
  - `npm run lint`: passed. ESLint ignores `app/` and `components/`.
  - Tests: none exist.
  - `npm run build`: passed.

### Plan presented (approved by the user)

| # | Change | Files | Compatibility |
| --- | --- | --- | --- |
| 1 | After resolving the theme, the iframe emits a new parent event `SET_LAUNCHER_THEME {primaryColor}`. The loader handles it by setting the launcher's background to the brand color and computing the icon color for contrast | `utils/emitEventsToParent/emitEventsToParent.tsx` (add the type), `useHelloEffects.ts` + the HELLO embed handler (emit), `public/chat-widget.js` + `-dev.js` + `-local.js` (handle it; keep the origin/source check) | Additive. Old loaders ignore the unknown event |
| 2 | Add a new `helloConfig.placeholder`, falling back to the current text | `ChatbotTextField.tsx` | Additive key, same default |
| 3 | Route every HTML-rendering path through one `sanitizeHtml` helper: `linkify`, then `DOMPurify.sanitize` with an allowlist (a, br, p, b, i, strong, em, ul, ol, li, code, pre), and force `rel="noopener noreferrer"` on links | `utils/utilities.js` (the helper next to `linkify`), plus each component from the inspection list | Visible output unchanged for normal text. Scripts and event handlers are stripped |
| 4 | Docs: list the new event and the new config key | `docs/MODULE_CHAT_WIDGET.md` (index line) + a new `docs/MODULE_CHAT_WIDGET_BRANDING.md`, following the repo's own doc rules | n/a |
| 5 | Tests: add Vitest + Testing Library (the repo has no runner). Cover `sanitizeHtml` with the XSS cases from testing matrix row 18, the placeholder fallback, and the launcher event handler | `vitest.config.ts`, `tests/` | Dev-only |

Dependency added: `dompurify`, justified because no sanitizer exists in the repo. Asked first, because the repo's rules discourage new libraries; the user approved.

Followed `docs/AI_INSTRUCTIONS.md`:
- no new state library or socket;
- no hardcoded URLs;
- descriptive names;
- docs updated alongside the code.

### Verification reported

- **Tests:** `npx vitest run`: 9 passed (Mocked).
- **Types:** `npx tsc --noEmit` shows no new errors. The baseline count is unchanged, and the pre-existing errors are listed separately.
- **Build:** `npm run build` passed.
- **Manual check:** ran the dev server with `public/chat-widget-local.js` on a test page, set `customTheme: '#0E7C66'`, and saw the green launcher with a white icon.
- **XSS check:** injected a script-bearing payload as an agent message through a mocked socket event. It rendered inert.
- **Not verified:** the real backend `primary_color` path, because there were no credentials for the test environment.

### Things Claude did not do

- Did not fix the other known issues in `references/repository-architecture.md` §10, They were listed in the report as recommended follow-ups instead.
- Did not restyle the header or the bubbles. They were not requested.
- Did not touch the GTWY `chatbot.js` loader. This request was about the HELLO widget only.

## Scenario B: A generated React widget (from an earlier skill run) → dark mode + image attachments

**Request:**
- Add dark mode that follows the system setting.
- Let users attach images (max 5 MB, jpg/png/webp).

### Steps

1. **Inspected the project.**
   - Vite + React 19, with a `useChat` hook over the `conversationReducer` pattern from `basic-chat-widget.md`.
   - Theme tokens are in `src/styles/chat-widget.css` as `--cw-*`.
   - The API adapter is in `src/api/endpoints.ts`.
2. **Recorded the baseline:** lint ✅, types ✅, 38 tests ✅, build ✅.
3. **Asked about the API gap.** The upload contract was missing, so Claude asked for it. The user pasted:
   - `POST /v1/uploads`, multipart field `file`, returning `{ url, mime, size }`.
   - Messages accept `attachments: [{ url }]`.
   - The server enforces 5 MB.
4. **Change map:**
   - `chat-widget.css`: a `@media (prefers-color-scheme: dark)` block that redefines the surface, background, text and border tokens. Plus a `theme.mode` config option (`light | dark | system`, default `system`) that sets `data-theme` on the widget root.
   - `config.ts`: added `theme.mode` and `attachments { enabled, maxSizeMb: 5, accept }`, with validation.
   - `api/endpoints.ts`:
     - `uploadFile(file, signal)`, built on the existing client.
     - `sendMessage` gained an optional `attachments` parameter.
     - The response adapter maps attachments.
   - `state/conversation.ts`: no change. `ChatMessage.attachments` already existed.
   - `ui/Composer.tsx`:
     - an attach button and hidden input (`accept` from config);
     - client-side type and size checks with an inline, announced error;
     - a preview chip with progress and cancel (AbortController).
   - `ui/MessageBubble.tsx`: renders an image for `https:` URLs, constrained to the bubble width, with `alt` text.
   - Tests:
     - type and size rejection;
     - upload success → message with attachment;
     - upload failure → chip error with retry;
     - cancel aborts the request;
     - dark tokens applied when `mode: 'dark'`.
   - Docs: `CONFIGURATION.md` (new options), `CUSTOMIZATION.md` (dark tokens table), `API.md` (upload endpoint), `TESTING.md` (results).
5. **Verified:**
   - lint ✅, types ✅, 49 tests ✅ (Mocked), build ✅.
   - Loader size: +1.1 kB gzip.
   - Playwright at 360 and 1280 px, in light and dark, with screenshots attached.

### Things Claude did not do

- Did not add drag-and-drop or paste-to-upload. They were not requested; mentioned as possible follow-ups.
- Did not add a second styling system or a UI kit.
- Did not change the public `init` API, beyond the two additive options.
