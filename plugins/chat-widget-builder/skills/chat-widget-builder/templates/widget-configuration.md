# Template: Widget Configuration Schema and docs/CONFIGURATION.md

Use this to design the config surface of a generated widget and to write its `docs/CONFIGURATION.md`. Keep only options for features that are actually built. Every documented option must work and be covered by a test (scenario 21).

## Contents

1. Design rules
2. Schema skeleton (TypeScript)
3. docs/CONFIGURATION.md skeleton

## 1. Design rules

- One typed config object (`WidgetConfig`), validated at `init()`; unknown keys warn in dev, required keys missing throw.
- Defaults live in one place (`config.ts`).
- Decide with the user which settings are **host-side** (passed to `init`) vs **backend-side** (fetched from a widget-settings endpoint). The reference does both: script-tag/init values plus backend widget settings (see `references/widget-configuration.md`). Do not duplicate a setting in both places without a precedence rule; document the rule.
- Runtime updates: support `update(partial)` only if the user needs it.
- Keep names consistent and boring: `primaryColor`, not `brandHue`.

## 2. Schema skeleton (TypeScript)

```ts
export interface WidgetConfig {
  // Required
  widgetToken: string;                 // public tenant id

  // Identity (optional; proof minted by host backend)
  user?: { id: string; name?: string; email?: string; phone?: string; signature?: string };

  // Placement
  position?: 'right' | 'left';         // default 'right'
  offset?: { x?: number; y?: number }; // px from the edge, default 20/20
  mount?: string;                      // CSS selector to render inline instead of floating
  hideLauncher?: boolean;              // host opens via API
  fullScreen?: boolean;                // open as full-viewport panel

  // Branding
  theme?: {
    primaryColor?: string;             // brand colour; text colour on it is computed for contrast
    mode?: 'light' | 'dark' | 'system';
    fontFamily?: string;
    radius?: number;
  };
  title?: string;
  subtitle?: string;
  logoUrl?: string;
  launcherIconUrl?: string;
  customCss?: string;                  // injected into the widget root only

  // Content
  welcomeMessage?: string;
  starterQuestions?: string[];
  placeholder?: string;
  texts?: Partial<Record<TextKey, string>>; // all UI strings, for i18n
  locale?: string;

  // Features (only those built)
  attachments?: { enabled: boolean; maxSizeMb: number; accept: string[] };
  emoji?: boolean;
  sound?: boolean;

  // Callbacks
  onReady?(): void;
  onOpen?(): void;
  onClose?(): void;
  onMessage?(message: ChatMessage): void;
  onUnreadChange?(count: number): void;
  onError?(error: { code: string; message: string }): void;
}
```

## 3. docs/CONFIGURATION.md skeleton

~~~markdown
# Configuration

## Required
| Option | Type | Description |
| --- | --- | --- |
| widgetToken | string | Public widget id from <where> |

## Optional
| Option | Type | Default | Description |
| --- | --- | --- | --- |

## Backend-managed settings
<Settings that come from the backend's widget-settings response, and precedence vs init options.>

## Theme tokens
| CSS variable | Default | Controls |
| --- | --- | --- |
| --cw-color-primary | #3b5bdb | Launcher, header, user bubbles, send button |

## Texts
| Key | Default |
| --- | --- |

## Public methods
| Method | Description |
| --- | --- |
| open() | Opens the window |

## Events / callbacks
| Callback | Payload | When |
| --- | --- | --- |

## Examples
<script-tag example, npm example>
~~~
