# Design Guidelines

## Contents

1. Choosing a design approach
2. The reference design (react-main), verified
3. Design tokens for generated widgets
4. Layout anatomy and states
5. Responsive rules
6. Accessibility
7. Design QA checklist

## 1. Choosing a design approach

Ask the user which one (Phase 2, question 9):

| Approach | What to collect | How to build |
| --- | --- | --- |
| A. Reference design | Brand colour, logo, title/subtitle, light/dark | Reproduce section 2's anatomy and proportions in the chosen stack; apply section 3 tokens; fix the reference's inconsistencies listed in section 2.7 |
| B. User-provided | Screenshots, Figma link (ask for exported frames or specs if Figma is not accessible), spec | Extract tokens (colours, radii, spacing, font sizes) into section 3 variables first, then build components; ask about states the design does not show (loading, error, offline, empty, long messages, attachments) |
| C. Generated | Brand colour, logo, tone (friendly/corporate/minimal), audience | Start from section 3 defaults; propose one direction with a short description (or a static mock in the demo) before building everything |

Never trade functionality for visuals: every state in section 4 must exist whatever the design.

## 2. The reference design (react-main), verified

### 2.1 Theming mechanics

- One brand colour drives everything. Sources, in priority: Hello `sdkConfig.customTheme` → backend `widget-info.primary_color` → default `#000000`/`#333333`; GTWY `interfaceDetails.config.themeColor`; runtime `themeColor` postMessage.
- `AppWrapper.tsx` writes `--primary-color` and `--primary-rgb` (6-digit hex only) on the root and builds an MUI theme (`hoc/theme.js`) with `primary.main` = brand.
- Derived colours (`useColor` + `utils/themeUtility.js`): text on brand = black if relative luminance > 0.35 else white (white always in dark mode); brand tint at 12–18 % alpha for avatars/hover; a subtle 150° three-stop brand gradient for the drawer header and active row; MUI `lighten`/`darken` for loaders and the scroll-to-bottom button.
- Modes: `light | dark | system` (system follows `prefers-color-scheme` live). Resolved mode sets `data-theme` and `.dark` on `<html>`; Tailwind `darkMode` keys off it. Only the Hello path can set the mode.
- Global CSS vars (light/dark): `--background` #fff/#121212, `--foreground` #171717/#ededed, `--message-card-background` #f0f0f0/#1f1f1f, `--icon-color` #555/#fff, `--drawer-color` #fff/#1f1f1f, `--error-color` #ef4444; links #0000EE / #90CAF9.

### 2.2 Typography, shape, motion

- System font stack (`system-ui, "Segoe UI", Roboto, …`); body 16 px/1.5 at ≥768 px, 14 px/1.4 below. Title `text-base font-semibold`, subtitle `text-sm` 75 % opacity, messages `text-sm md:text-base`, markdown via Tailwind Typography `prose prose-sm md:prose-base`.
- Radius: user bubble `10px 10px 1px 10px` (tail at bottom-right), agent/bot card 8 px, composer 12 px, round icon buttons, host popup 16 px.
- Shadows: soft popup shadow `0 5px 40px rgba(15,15,15,.08)`; composer `shadow-sm`.
- Icons: lucide-react (22 px in header). Motion: 0.28 s slide-up for sheets, 0.2 s fades, 0.3 s host-side resize; shimmer bars while the bot is "thinking". Scrollbars hidden.

### 2.3 Launcher (host page, `public/chat-widget.js` + `chat-widget-style.css`)

- 48×48 white circle, light grey border, layered shadow, paper-plane icon in a fixed blue (brand colour **not** applied), fixed 18 px from bottom/right; `icon_position` left/right and `icon_bottom_margin` adjustable; `iconUrl` swaps the image; `buttonName` turns it into a dark text pill.
- Red unread badge (min 18 px) with a masked notch; optional starter-question bubbles and a 320 px last-message preview card above the launcher.

### 2.4 Window

- Default popup 440 px wide × `min(724px, 100% - 40px)`; other types `pop_over`, `all_available_space`, `left_slider`, `right_slider`; or inline in a host element (`parentId`). Full-screen below 600 px viewport.
- Inside: optional left sidebar (≤286 px, overlay drawer under 1024 px) + main column: header → thin brand-tinted progress bar → pre-chat form → call bar → AI/Human tabs → content (notifications, empty view, or conversation).
- Empty view: small animated image, "What can I help with?", composer, starter questions.
- Header: **neutral, not brand-coloured** (transparent on desktop, background + bottom border on small screens). Left: drawer toggle with unread dot, new-thread (AI). Centre: optional 24 px round icon + title (+ subtitle). Right: optional bridge/model switchers, host-defined header buttons, overflow menu (minimize, full screen, new conversation), close. Minimized mode is a compact bar with last-message preview and unread badge.
- Drawer: brand-gradient greeting header ("Hello, {first name} 👋" + tagline), conversation list (initial/bot avatars on brand tint, unread badge, "Closed" pill, 3 rows then "See all"), teams list, "Need specialized help?" banner (Call / Message), and a "Powered by" footer (conditional).

### 2.5 Messages and composer

- User: right-aligned, brand fill, computed text colour, max 80–90 % width, 10 px padding; hover reply action; tap shows relative timestamp.
- AI assistant: no bubble, plain markdown, actions row (copy, 👍, 👎), "N functions executed" badge, JSON can render as generative UI grid.
- Agent / Hello bot: left grey card with border; under it `name • time` (shown on the last message by default). Avatars exist in code but are commented out.
- Day dividers between messages; reversed infinite scroll; scroll-to-bottom button.
- Composer: up to 3 quick-reply chips above; image previews (max 4); rounded box with brand focus outline; 1–6-row textarea; bottom row emoji, attach, call, send (round brand button, grey when disabled). Blocked conversation replaces the composer with a notice.
- Toasts: bottom snackbar, filled alert, 3 s.

### 2.6 Responsive

Breakpoints: 1024 px (sidebar becomes overlay; header/footer get solid background; pre-chat form becomes a bottom sheet), 768 px (font size), 600 px (host forces full-screen, minimize closes), 480 px (launcher bubbles).

### 2.7 Inconsistencies to fix, not copy

- Brand colour not applied to the launcher; daisyUI `primary` not mapped to brand, so `bg-primary` shows a stock colour.
- Three styling systems (Tailwind + daisyUI + MUI/Emotion) plus per-component CSS files and inline styles.
- `--primary-rgb` missing for shorthand hex / `rgb()` inputs.
- Undefined utility classes (`animate-fade-in`, `drop-shadow-pink`, `from-primary-500`) silently do nothing.
- Subtitle only renders when `chatbotSubtitle` is set (`chatSubTitle` alone never shows).
- Contrast threshold (luminance > 0.35) is a heuristic; check WCAG ratio instead.

## 3. Design tokens for generated widgets

Expose tokens as CSS custom properties on the widget root (prefix with the widget's own prefix, e.g. `--cw-`). Compute derived values in code from `primaryColor`, not by asking the user for each.

| Token | Default (light) | Dark | Notes |
| --- | --- | --- | --- |
| `--cw-color-primary` | brand | brand | Launcher, user bubble, send button, focus ring |
| `--cw-color-on-primary` | computed | computed | Black or white, whichever has the higher WCAG contrast with primary |
| `--cw-color-primary-tint` | primary @ 12 % | primary @ 18 % | Avatars, hover, chips |
| `--cw-color-bg` | #ffffff | #121212 | Window background |
| `--cw-color-surface` | #f2f3f5 | #1f1f1f | Incoming bubbles, cards |
| `--cw-color-text` | #171717 | #ededed | |
| `--cw-color-text-muted` | #6b7280 | #a1a1aa | Timestamps, meta |
| `--cw-color-border` | #e5e7eb | #2e2e2e | |
| `--cw-color-danger` | #dc2626 | #f87171 | Failed message, errors |
| `--cw-font-family` | system-ui stack | | |
| `--cw-font-size` | 15px (14 on mobile) | | |
| `--cw-radius-window` | 16px | | |
| `--cw-radius-bubble` | 14px | | Tail corner 4px |
| `--cw-shadow-window` | 0 5px 40px rgb(0 0 0 / .12) | | |
| `--cw-window-width` | 400px | | Reference uses 440 |
| `--cw-window-height` | min(680px, 100dvh - 40px) | | |
| `--cw-launcher-size` | 56px | | ≥44 px touch target |
| `--cw-z-index` | 2147483000 | | Host-page stacking |

Contrast: verify `on-primary` vs `primary` ≥ 4.5:1 for message text; if neither black nor white reaches it, warn the user and darken the bubble shade.

## 4. Layout anatomy and states

Required components (only those in scope): launcher (with unread badge), window, header (logo/avatar, title, subtitle or status, close, optional minimize/full-screen/menu), message list, bubbles per sender and type, date separators, composer, optional conversation list, optional pre-chat form, optional "Powered by".

Every widget must have these states designed and implemented:

| State | Minimum |
| --- | --- |
| Initializing | Skeleton or spinner inside the window; launcher usable |
| Empty conversation | Welcome message and/or starter questions; composer focused |
| Loading older history | Small spinner at top; scroll position kept |
| Sending | Bubble at reduced opacity or clock icon |
| Failed send | Red indicator + "Retry"; text preserved |
| Bot/agent typing | Three-dot indicator or "thinking" shimmer, auto-clears |
| Offline / reconnecting | Thin banner in the window; composer behaviour per spec |
| Error (init/history) | Message + retry button |
| Conversation closed | Notice + "Start new conversation" (if supported) |
| Long content | Wraps long words/URLs (`overflow-wrap: anywhere`), images constrained, code scrolls horizontally inside the bubble |

## 5. Responsive rules

- Mobile (<600 px): window full-screen (`100dvh`), header with clear close button, launcher hidden while open, safe-area insets (`env(safe-area-inset-*)`), composer above the on-screen keyboard.
- Tablet/desktop: floating window anchored to the launcher corner; never taller than the viewport minus margins.
- Inline mode (`mount` target): fills the container; no launcher.
- Respect `prefers-reduced-motion` (disable slide/shimmer animations).

## 6. Accessibility

- Launcher: `<button>` with `aria-label`, `aria-expanded`, `aria-controls`; badge count in the label ("Open chat, 2 unread").
- Window: `role="dialog"` with `aria-label`, focus moved in on open and back to the launcher on close; Esc closes if the user wants it.
- Message list: `role="log"` / `aria-live="polite"`; each message conveys sender to screen readers.
- Composer: labelled textarea; Enter sends, Shift+Enter newline; buttons have labels; attachment errors announced.
- Colour is never the only status signal (failed = icon + text).
- Contrast ≥ 4.5:1 for text, ≥ 3:1 for UI controls; visible focus ring.

## 7. Design QA checklist

- [ ] Tokens applied; changing `primaryColor` re-themes launcher, user bubbles, send button and focus ring with readable text
- [ ] Light and dark (if in scope) checked
- [ ] All section 4 states visible in the demo (mock mode can force them)
- [ ] 360 / 768 / 1280 px checked; no horizontal scroll; keyboard does not hide the composer on mobile
- [ ] Host CSS does not leak in (aggressive host stylesheet test) and widget CSS does not leak out
- [ ] Keyboard-only flow works; screen-reader labels present
