# Widget Configuration (reference: react-main)

How the reference is embedded and configured, verified from source. Use it to (a) customize the reference or a widget derived from it without breaking its public contract, and (b) borrow proven config ideas for new widgets. For designing a new widget's config, use `templates/widget-configuration.md`.

## Contents

1. Embed scripts and routes
2. HELLO loader: script tag and window API
3. HELLO configuration keys
4. GTWY loader: attributes and window API
5. Backend-managed settings (widget-info)
6. postMessage protocol
7. Storage keys
8. Environments and build
9. Things that look configurable but are broken

## 1. Embed scripts and routes

| Product | Host script | Iframe route | Iframe handler |
| --- | --- | --- | --- |
| HELLO (human/helpdesk chat) | `public/chat-widget.js` (+ `-dev`, `-local`) | `/chatbot?interfaceDetails={"isHelloUser":true}` → `/chatbot/hello` | `hooks/HELLO/eventHandlers/embeddingScript/` |
| GTWY (AI chatbot) | `public/chatbot.js` (+ `-dev`, `-local`) | `/chatbot?interfaceDetails=<login data>` → `/chatbot/<chatbot_id>` | `hooks/GTWY/eventHandlers/embeddingScript/` |
| RAG (knowledge-base manager, not chat) | `public/rag-prod.js` (+ `-dev`, `-local`) | `/rag` | `app/rag/page.tsx` |

`hooks/CORE/.../embeddingScriptEventHandler.ts` is one shared `message` listener that registers both GTWY and HELLO handlers and routes by `event.data.type`. `/chatbotpreview` is a static demo UI that only accepts a `themeChange` message.

## 2. HELLO loader: script tag and window API

```html
<script src="https://<widget-host>/chat-widget.js"></script>
<script>
  window.initChatWidget({
    widgetToken: '<public widget token>',
    unique_id: 'user-123', name: 'Ada', mail: 'ada@example.com', number: '+1…', // optional identity
    user_jwt_token: '<jwt from host backend>',   // optional identity proof
    hide_launcher: false, launch_widget: false, icon_position: 'right', icon_bottom_margin: 24,
    sdkConfig: { customTheme: '#3b5bdb', theme: 'system' },
    variables: { plan: 'pro' },
  }, 0 /* delay ms */);
</script>
```

- Script located via `document.currentScript` (fallback `script[src*="chat-widget"]`). `scriptOrigin` = origin it was served from; the iframe, stylesheet and `urlMonitor.js` load from there.
- Namespace: first hostname label (or port on localhost); globals live on `window.chatWidget[<subdomain>]` (legacy subdomains also copy methods flat onto `window.chatWidget`). Element ids are prefixed `<subdomain>-hello-`.

Window API (`chat-widget.js` ~1623–1695):

| Method | Effect |
| --- | --- |
| `initChatWidget(data, delay=0)` | Initialize / re-initialize (also how identity changes) |
| `SendDataToBot(data)` | Runtime update (`themeColor`, `tagline`, `config`, `variables`, `starter_question`, …); `{variables}` alone posts `SET_VARIABLES_FOR_BOT` |
| `addCustomData` / `modifyCustomData(data)` | Update client profile fields |
| `addUserEvent(data)` | Track a user event |
| `open(id?)` / `close()` / `hide()` / `show()` / `toggleWidget()` | Visibility; `open(ticketId)` opens a ticket |
| `openWithMessage(message, autoSend=true, alwaysNewChat=true)` | Open and prefill/send |
| `showTicket(id)` / `getTicketUnreadCount(id)` | Ticket view; promise of unread count (5 s timeout → 0) |
| `handlePushNotification(data)` | Mobile SDK push |
| `shutdown()` | Logout-like reset of client ids (DOM stays) |
| `initUrlTracker({urls})` | From `urlMonitor.js`: open matching host links in a side panel |

## 3. HELLO configuration keys

Read by the loader (host page): `hide_launcher` (launcher shown only when explicitly `false`), `launch_widget` (auto-open), `parentId` (render inside a host element), `icon_position` (`left`/`right`), `icon_bottom_margin` (number → px, or CSS string), `isMobileSDK`, `previewLinks` (enables URL monitor), `variables`.

Read by the iframe (`helloData`, stored in sessionStorage `helloConfig`):

| Key | Effect |
| --- | --- |
| `widgetToken` (required) | Tenant; session key becomes `${widgetToken}_${unique_id}` |
| `unique_id`, `mail`, `number`, `name` | Identity / profile |
| `user_jwt_token` | Identity proof, sent inside `user_data` |
| `sdkConfig.customTheme` | Brand colour (overrides backend `primary_color`) |
| `theme` / `sdkConfig.theme` | `light` / `dark` / `system` |
| `variables` | Context variables sent with each message (`sessionVariables`) |
| `show_close_button`, `show_send_button`, `show_widget_form`, `fullScreen` | UI toggles |
| `hideFullScreenButton` | Intended to hide full-screen (see §9) |
| `prefillMessage` | Composer prefill |
| `viewMode: 'chatbot' \| 'ticket'` | Ticket mode (3-row composer, opens a channel by id) |
| `pushConfig` | Mobile push registration |
| `flow_origin`, `integration` | Merge params for channel list |
| any other key | Sent to `PUT /v2/client/{id}` as client fields |

Runtime (`SendDataToBot` → `helloRunTimeData`): `themeColor`, `tagline`, `show_widget_form`, `variables`, `fullScreen`, `viewMode`; loader-side `hideCloseButton`, `hide_launcher`, `launch_widget`, `parentId`, `starter_question` + `starter_question_text`, `config: {type, height, heightUnit, width, widthUnit, buttonName, iconUrl, title}`.

Window `config.type`: `popup` (default 440 px × `min(724px, 100% - 40px)`), `pop_over`, `all_available_space`, `left_slider`, `right_slider`.

## 4. GTWY loader: attributes and window API

```html
<script id="chatbot-main-script" src="https://<host>/chatbot.js"
  embedToken="<token from host backend>" threadId="t1" bridgeName="support"
  config='{"type":"popup","width":"480","widthUnit":"px"}' defaultOpen="false"></script>
```

Attributes (`chatbot.js` 73–79): `interfaceId`, `embedToken`, `threadId`, `bridgeName`, `variables`, `onOpen`, `onClose`, `iconColor`, `className`, `style`, `environment`, `fullScreen`, `hideCloseButton`, `hideIcon`, `parentId`, `config`, `headerButtons`, `eventsToSubscribe`, `modalConfig`, `allowModalSwitch`, `chatTitle`, `chatIcon`, `hideFullScreenButton`, `defaultOpen` (JSON-parsed: `config`, `headerButtons`, `eventsToSubscribe`, `modalConfig`). The loader logs in with `embedToken` (or anonymously) and passes the login response to the iframe in the URL.

Window API: `SendDataToChatbot(data)`, `openChatbot()`, `closeChatbot()`, `Chatbot.{open, close, show, hide, reloadChats, askAi, sendData}`, `reloadChats()`, `askAi(...)`.

Iframe keys (`interfaceData`): `threadId`, `helloId`, `version_id`, `bridgeName`, `variables`, `modelChanged`, `serviceChanged`, `headerButtons[]` (`setting`, `history`, `verticalThreeDots`, `sectionDropdown`), `eventsToSubscribe[]` (`MESSAGE_CLICK`, `USER_TYPING`), `modalConfig`, `allowModalSwitch`, `chatTitle`, `chatSubTitle`, `chatIcon`, `allowBridgeSwitch`, `hideCloseButton`, `hideFullScreenButton`. Login config: `themeColor`, `chatbotTitle`, `chatbotSubtitle`, `width`, `widthUnit`, `allowBridgeSwitch`, `bridges`.

## 5. Backend-managed settings (widget-info)

HELLO fetches `POST /widget-info/` at init; these are dashboard settings, so change them in the backend, not in code: `primary_color`, `hide_launcher`, `launch_widget`, `bot_type`, `voice_call_widget`, `teams`, `show_widget_form`, `tagline`, `show_msg91` ("Powered by" footer), `demo_widget`, `event_channels`, `inbox_id`. Precedence for colour: host `sdkConfig.customTheme` > backend `primary_color` > default.

## 6. postMessage protocol

Host loader → HELLO iframe: `helloData`, `helloRunTimeData`, `SET_VARIABLES_FOR_BOT`, `CHATBOT_OPEN {id}`, `CHATBOT_CLOSE`, `open`, `askAi`, `UPDATE_USER_DATA_SEGMENTO`, `ADD_USER_EVENT_SEGMENTO`, `SEND_INITIAL_MESSAGE {message, autoSend, alwaysNewChat}`, `SHOW_TICKET {id}`, `GET_TICKET_UNREAD_COUNT {id}`, `SHUTDOWN_CHATBOT`, `parent-route-changed {websiteUrl}`, `STARTER_QUESTION_OPTION_CLICKED {option}`, `ADD_COBROWSE_SCRIPT {origin}`, `OPEN_WITH_NOTIFICATION`.

GTWY loader → iframe: `interfaceData`, `askAi`, `refresh`, `open`.

Iframe → host (type union in `utils/emitEventsToParent/emitEventsToParent.tsx`):

| Type | Payload / meaning |
| --- | --- |
| `interfaceLoaded` | Iframe ready; loader then sends config |
| `CLOSE_CHATBOT`, `MINIMIZE_CHATBOT`, `OPEN_CHATBOT` | Visibility |
| `ENTER_FULL_SCREEN_CHATBOT` / `EXIT_FULL_SCREEN_CHATBOT` | Resize container |
| `initializeHelloChat_failed` | Init failed; loader cleans up |
| `hide_widget`, `launch_widget` | Backend settings applied to launcher |
| `SET_BADGE_COUNT {badgeCount, channelId}` | Unread badge (`'99+'` cap) |
| `SHOW_STARTER_QUESTION {message, options}` / `HIDE_STARTER_QUESTION` | Greeting bubble over launcher |
| `setDataInLocal {key, payload}` | Mirror ids into host localStorage |
| `uuid {uuid}` | Client id (co-browse) |
| `PUSH_NOTIFICATION`, `LAUNCHER_MESSAGE_PREVIEW` | Proactive messages on host page |
| `TICKET_UNREAD_COUNT {id, count}` | Answer to the ticket query |
| `ENABLE_DOMAIN_TRACKING`, `RELOAD_PARENT`, `downloadAttachment {url}` | Host actions |
| `FRONT_END_ACTION`, `HEADER_BUTTON_PRESS`, `BRIDGE_SWITCH`, `MESSAGE_CLICK` | GTWY host-integration events |

Origin checks: `chat-widget.js` accepts only `scriptOrigin`/host origin **and** `event.source === its iframe`. That is a good pattern to copy. Do not assume the other loaders or the iframe side validate messages the same way. Generated widgets must check both directions, with explicit target origins (`security-guidelines.md` §4).

## 7. Storage keys

- sessionStorage: `widgetToken` (backed up to `__session_backup_widgetToken`), `helloConfig`, `tab_session_id`, `interfaceToken`, `interfaceUserId`, `ragToken`, `persist:appInfo`.
- localStorage, prefixed `${widgetToken}_`: `k_clientId`, `a_clientId`, `WidgetId`, `userData`, `is_anon`, `HelloClientToken`, `HelloCallToken`, `HelloAgentAuth`, `JwtTokenForSocket`, `client`, `default_client_created`, `persist:Chat` (notifications).
- localStorage, unprefixed: `${WidgetId}_last_unique_id`, `${WidgetId}_k_clientId`, `${WidgetId}_a_clientId`, `${WidgetId}_${uid}_k_clientId`; redux-persist `persist:root` (Interface + Hello slices).
- Host page localStorage (via `setDataInLocal`): `widgetInfo`, `hello-widget-uuid`, `hello-widget-anonymoud-uuid`.
- `utils/safeStorage.ts` mirrors writes into memory so the widget works where storage is blocked.
- `redux-state-sync` (`crossTabChannel`) syncs Interface/Hello actions across tabs; appInfo is keyed per `${chatSessionId}_${tabSessionId}`.

## 8. Environments and build

- Env vars: `NEXT_PUBLIC_MSG91_HOST_URL`, `NEXT_PUBLIC_PUSH_NOTIFICATION_URL`, `NEXT_PUBLIC_API_BASE_URL`, `NEXT_PUBLIC_PYTHON_API_BASE_URL`, `NEXT_PUBLIC_SOCKET_URL`, `NEXT_PUBLIC_NOTIFICATOIN_SOCKET_URL` (misspelled in code; keep the spelling when customizing), `REACT_APP_API_ENVIRONMENT`. `.env*` is gitignored; no `.env.example` exists.
- Script variants per environment are separate files (`chat-widget.js`, `-dev.js`, `-local.js`) with different hosts/defaults; `-dev` forwards `?env=stage` to the iframe, which switches HELLO API and sockets to stage hosts.
- Build: `next build`; Cloudflare Pages via `npm run build:pages` (`@cloudflare/next-on-pages`); chatbot routes use `runtime = "edge"`. `preview`/`deploy` scripts call a non-existent `pages:build` script, and there is no `wrangler.toml`. `next.config.ts` ignores ESLint and TypeScript errors during builds.

## 9. Things that look configurable but are broken

Do not rely on these when customizing; fix them only if the user asks:

- `iconColor` in `SendDataToBot` references undefined globals and a missing element (would throw).
- `hideCloseButton` via the HELLO loader targets an element that is never created.
- HELLO `hideFullScreenButton` is written to the Interface slice but the header reads appInfo.
- `chatSubTitle` alone never renders (subtitle shows only when `chatbotSubtitle` is set).
- `askAi` sent through `chat-widget.js` is ignored (only handled when not a Hello user).
- Minimize sets width `'280'` without a unit.
- `data-widget-id` attribute is mentioned in a comment but never read.
