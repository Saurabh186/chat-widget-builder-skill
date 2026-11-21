# chat-widget-builder: a Claude skill

A Claude skill that builds or customizes an embeddable **chat widget** on your own backend APIs, in the frontend stack of your choice: vanilla TypeScript Web Component, React, Vue, Angular, or a loader script with an iframe.

You describe what you need. Claude asks one question at a time, shows you a plan, and waits for your approval. Then it builds the widget, writes tests, runs them, and documents the result.

## What it does

- **Two modes.** Create a new, independent widget, or customize an existing one with the smallest possible change.
- **Your APIs only.** Claude collects your real request/response and socket examples. It never invents endpoints or fields. Anything not confirmed is marked `TODO(confirm-with-user)`.
- **Real-time support.** Uses whatever your backend speaks: socket.io, WebSocket, SSE or polling. Includes reconnect, history catch-up and deduplication.
- **Pick your features.** Choose from a menu: history, typing indicator, unread badge, attachments, emoji, quick replies, markdown, bot → human handoff, and more.
- **Design.** Reproduce the reference design, follow your screenshots or Figma, or let Claude generate one. Includes theme tokens, responsive layout and accessibility.
- **Security built in.** No secrets in frontend code, sanitized message rendering, `postMessage` origin checks, and upload limits.
- **Honest testing.** A 24-scenario test matrix, with every result labelled Mocked or Real API.
- **Documentation for each widget.** README, `.env.example`, API, configuration, integration, customization, architecture, testing and deployment docs.

The skill's chat-behaviour reference is Walkover's open-source [chat-widget](https://github.com/Walkover-Web-Solution/chat-widget) (branch `react-main`, commit `e8cf9d7`). Generated widgets do not depend on it at runtime.

## Install

### Claude Code: plugin (recommended)

In Claude Code:

```
/plugin marketplace add Saurabh186/chat-widget-builder-skill
/plugin install chat-widget-builder@chat-widget-builder-skill
```

Then ask, for example, *"Build a chat widget for our support API"*, or run `/chat-widget-builder:chat-widget-builder`.

### Claude Code: plain skill folder

Copy `plugins/chat-widget-builder/skills/chat-widget-builder/` into one of these locations:

- `~/.claude/skills/`, for all your projects;
- `.claude/skills/` inside a repository, for everyone who clones it.

Then invoke it with `/chat-widget-builder`.

### Claude.ai / Claude Desktop

1. Download `chat-widget-builder.zip` from the [latest release](https://github.com/Saurabh186/chat-widget-builder-skill/releases/latest).
2. Turn on code execution: **Settings → Capabilities**.
3. Upload the ZIP: **Customize → Skills → "+" → Create skill → Upload a skill**. Upload it as is; the ZIP already contains the skill folder at its root.

Where you can, use Claude Code. Claude.ai's sandbox may block network access, package installs or browsers, so Claude may not be able to run every test there. Either way, the skill reports exactly which checks ran.

## Repository layout

```
.claude-plugin/marketplace.json            marketplace listing
plugins/chat-widget-builder/
├── .claude-plugin/plugin.json             plugin manifest
└── skills/chat-widget-builder/
    ├── SKILL.md                           workflow and rules
    ├── references/                        reference architecture, chat flow, APIs, config, design, dev, security, testing
    ├── templates/                         project structure, API doc, config schema
    └── examples/                          new widget, API mapping (verified code), customization
```

The example code (message store, API client, socket.io transport) passes `tsc --strict` and 18 Vitest tests. The transport tests run against a real local socket.io server.

## Updating

```
/plugin marketplace update chat-widget-builder-skill
```

## Contributing

Issues and pull requests are welcome.

If the reference repository changes, re-check the files in `references/` against the new code. Each file states the commit it describes.
