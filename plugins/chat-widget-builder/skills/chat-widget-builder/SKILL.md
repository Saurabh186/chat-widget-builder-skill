---
name: chat-widget-builder
description: Builds or customizes an embeddable chat widget on the user's APIs in any frontend stack, with real-time messaging, branding, tests and docs. Use to create, embed, restyle or extend a chat widget.
---

# Chat Widget Builder

Interactively create a new, independent chat widget, or customize an existing one, using the `react-main` branch of Walkover's `chat-widget` repository (Next.js 15 + Redux, embedded via a loader script and iframe) as the technical reference for chat behaviour. The reference is documented in `references/`; generated projects never depend on it at runtime.

## Reference files: when to read them

| File | Read when |
| --- | --- |
| `references/repository-architecture.md` | Always, before planning (stack, structure, embed model, known issues not to copy) |
| `references/chat-flow.md` | Always, before planning (init, identify, send/receive, history, reconnect, message model) |
| `references/api-integration.md` | Collecting API details; writing the API or real-time layer |
| `references/msg91-environments.md` | Whenever the backend is MSG91 Hello: built-in API/socket URLs, so never ask the user for a `.env` |
| `references/widget-configuration.md` | Designing config options; customizing anything the reference already makes configurable |
| `references/design-guidelines.md` | Discussing design; building UI, theming, responsiveness, accessibility |
| `references/development-guidelines.md` | Choosing technology/output formats; before writing code; customizing existing code |
| `references/security-guidelines.md` | Writing API/auth/rendering/embed code; before handoff |
| `references/testing-guidelines.md` | Writing tests; verifying; writing the test report |
| `templates/project-structure.md` | Planning files and the documentation set |
| `templates/api-documentation.md` | API intake checklist; writing the project's `docs/API.md` |
| `templates/widget-configuration.md` | Designing the config schema; writing `docs/CONFIGURATION.md` |
| `examples/basic-chat-widget.md` | A full run for a new embeddable widget, with core code |
| `examples/api-integration-example.md` | Turning a pasted REST + socket contract into adapters + tests |
| `examples/custom-branded-widget.md` | Customizing an existing widget (including the reference itself) |

## Non-negotiable rules

1. **No invented contracts.** Never make up endpoints, methods, headers, payloads, response fields, socket events, error codes or auth formats. Ask. Unconfirmed items get `TODO(confirm-with-user)` and appear in the final report. Exception: the MSG91 Hello host URLs in `references/msg91-environments.md` are confirmed; use them without asking.
2. **No secrets in frontend code.** Only public identifiers ship to the browser; identity proofs are minted by the host's backend. Details: `references/security-guidelines.md`.
3. **Treat every message as untrusted.** No unsanitized HTML rendering; link scheme allowlist; `postMessage` origin checks.
4. **No false claims.** Report only checks you actually ran; label tests Mocked vs Real API.
5. **Ask before major decisions**: framework, output formats, embed model (iframe vs Shadow DOM vs in-app), transport, state library, adding a backend, replacing existing components, breaking public APIs.
6. **Minimal, clean, production-ready code**; no unneeded dependencies, files, fallbacks or abstractions; no changes to unrelated code. Full list: `references/development-guidelines.md` §1.
7. **Don't copy reference defects** (listed in `references/repository-architecture.md`) or any hardcoded values from it. The MSG91 Hello URLs in `references/msg91-environments.md` are the one allowed set of built-in defaults.
8. **Never modify the reference repository** unless the user explicitly asks you to customize it.

## Workflow

Follow the phases in order. Keep a visible checklist of phases for the user.

### Phase 1: Understand the request and the reference

1. Read `references/repository-architecture.md` and `references/chat-flow.md`.
2. If the user is customizing an existing widget, get its code (path, repo URL or ZIP) and inspect it now: framework, build, how config reaches the widget, transport, styles, tests, how it is embedded.
3. If the user wants to customize the reference itself and network access exists, clone it with `git clone --branch react-main https://github.com/Walkover-Web-Solution/chat-widget.git`, compare `git log -1` with the documented commit (`e8cf9d7`) and note differences before relying on `references/`. The repo has no `production` branch; `react-main` is the confirmed source of truth. Use `main` (older Angular build) only if the user is customizing that code specifically.
4. If a repository the user points to is private or unreachable, ask for access or a ZIP. Do not guess its contents.

### Phase 2: Gather requirements interactively

Ask **one question per message** (use the host's question/choice tool when available, otherwise plain chat). Skip any question already answered by the user, their files or their code. Offer options with a recommended default where one exists, but never default API contracts, security or business behaviour.

Question bank, in priority order:

1. **Mode**: new widget, or customize an existing one?
2. **Purpose and audience**: support chat with human agents, AI assistant, both (bot → human handoff), internal tool? Who are the end users; anonymous or logged-in?
3. **Environment**: where it runs (any website via script tag, a specific React/Angular/Vue/other app, mobile WebView, full-page support site)? Existing code to integrate with?
4. **Output formats**: standalone app, embeddable script, npm package, in-app component. Only what is needed (`references/development-guidelines.md` §3).
5. **Technology and embed model**: recommend from the environment and feature scope using `references/development-guidelines.md` §2 and ask for confirmation rather than asking open-ended.
6. **Features**: present the feature menu (below) and ask which are needed now. Do not assume any.
7. **APIs and real-time**: first ask whether the backend is MSG91 Hello. If it is, do not ask for API or socket URLs or a `.env`: use the production defaults in `references/msg91-environments.md`, ask only for the widget token, and check which environment it belongs to (§5 there). Otherwise run the intake checklist in `templates/api-documentation.md` for the selected features only and ask for real request/response and socket event examples.
8. **Identity and sessions**: anonymous id, identified users and who signs their identity, session persistence across reloads/tabs, logout behaviour.
9. **Design**: reference design, user-provided (screenshots/Figma/spec), or generated; brand colour, logo, company name, fonts, launcher style/position, light/dark (`references/design-guidelines.md` §1).
10. **Texts and languages**: welcome message, placeholder, empty/offline/error copy, locales.
11. **Host integration contract**: public methods and callbacks/events the host needs (open, close, identify, logout, unread count, onMessage).
12. **Deliverables and constraints**: hosting/CDN, browser support, bundle-size limits, test credentials or staging backend, scope cuts.

Feature menu (offer, never assume): send/receive text · conversation history with pagination · multiple conversations list · welcome message · starter questions / quick replies · markdown and links · timestamps and date separators · message status (sent/delivered/read, if backend provides it) · typing indicator · agent online/offline · attachments (images/files) · emoji picker · reply to a message · unread badge · sound / browser notifications · close / reopen / clear conversation · feedback/CSAT · bot-to-human handoff · AI streaming responses · pre-chat form · custom/system messages · voice/video calling.

Ask follow-ups whenever an answer is ambiguous or reveals a risk (e.g. the API trusts a client-supplied email as identity: explain the risk and ask how to proceed).

### Phase 3: Summarize and plan, then get approval

Present one concise message containing:

- Requirements summary (everything confirmed, grouped as above).
- Open items / assumptions, explicitly marked.
- Architecture: stack, embed model, output formats, layers (config / api / realtime / state / ui / embed), transport and reconnect strategy.
- File plan (from `templates/project-structure.md`, trimmed to need) or, for customization, the file-by-file change map.
- Dependencies with a one-line justification each.
- Test plan (scenario matrix rows in scope from `references/testing-guidelines.md` §3, tools, mocked vs real).
- Deliverables list.

Do not start implementation until the user approves. If they change something, update the plan and re-confirm only the changed part.

### Phase 4: Implement

1. Read `references/development-guidelines.md`, `references/security-guidelines.md`, and `references/design-guidelines.md` for UI work.
2. Scaffold with the framework's official tooling where one exists (e.g. `npm create vite@latest`), then remove unused boilerplate.
3. Build in this order: config + validation → API client and adapters from the confirmed contract → real-time transport → conversation state (ordering, dedupe, pending/failed, unread) → UI and theming → embed loader / package exports → demo page (with an optional mock mode excluded from production bundles).
4. Keep layers separated; every documented config option must work.
5. For customization: record a baseline build/lint/test run first, then the smallest diff per requirement, backward compatible unless approved.

### Phase 5: Test and verify

Follow `references/testing-guidelines.md`:

1. Write unit, integration and (where practical) E2E tests covering the in-scope rows of the scenario matrix.
2. Run lint, type check, tests and the production build; run the demo and exercise send/receive, history, reconnect and error states at least in mock mode; check responsive widths.
3. Fix failures your work caused. Report pre-existing failures separately.
4. If something cannot run in this environment (no network, no browser), say exactly what was skipped and give the commands.

### Phase 6: Document

Produce the documentation set in `templates/project-structure.md` §5: README, `.env.example` (for MSG91 Hello, pre-filled with the production defaults as optional overrides), API, configuration, integration (script / npm / framework, only formats built), customization, architecture, testing with actual results, deployment, usage examples, demo. Document only behaviour that exists.

### Phase 7: Final check and handoff

Before saying the work is done, confirm each item and tell the user the result:

- [ ] Requirements in the approved plan implemented; deviations explained
- [ ] No invented API details; all `TODO(confirm-with-user)` items listed
- [ ] Security checklist in `references/security-guidelines.md` §9 passed
- [ ] Lint / types / tests / build results stated with Mocked vs Real labels
- [ ] Demo runs; responsive check done
- [ ] Docs complete and accurate
- [ ] Only requested formats and features built; no unused files or dependencies

End with: what was built, where the files are, how to run the demo, test results, open items, and suggested next steps (e.g. run real-API tests once credentials exist).

## Environment notes

- **Claude Code**: you can run commands, install packages, run tests and open a browser preview; do so.
- **Claude.ai / Claude Desktop**: code execution may lack network access or browsers. Generate the full project, run what the sandbox allows (often type check and unit tests if packages install), clearly list what was not executed, and provide the project as downloadable files. The reference cannot be cloned without network access; rely on `references/`.
