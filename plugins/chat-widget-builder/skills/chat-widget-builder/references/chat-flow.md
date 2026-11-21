# Chat Flow (reference: react-main)

Behaviour verified from source. Part A describes the reference; Part B is the target flow every generated widget implements (it keeps the reference's good ideas and closes its gaps).

## Contents

- A1. Lifecycle
- A2. State architecture
- A3. Message model
- A4. Send flow (AI and HELLO)
- A5. Receive, history, unread
- A6. Message types and rendering
- A7. Composer
- A8. UI states
- A9. Gaps in the reference
- B1. Target lifecycle
- B2. Target message state machine
- B3. Conversation store rules

## A1. Lifecycle (HELLO)

1. **Load**: `chat-widget.js` creates launcher (hidden), container and iframe → `/chatbot?interfaceDetails={"isHelloUser":true}` → redirect to `/chatbot/hello`. Iframe posts `interfaceLoaded`.
2. **Init**: host calls `initChatWidget(data, delay)`; once both delay elapsed and `interfaceLoaded` received, loader posts `helloData`.
3. **Identify**: iframe stores `widgetToken` (session key `${widgetToken}_${unique_id}`), `userData`, `WidgetId`. Then: anonymous → `POST /anonymous-client-details/`; otherwise `POST /pubnub-channels/list/` (returns client uuid + conversations; merges anonymous history on first login).
4. **Services**: `POST /widget-info/` (settings; failure → `initializeHelloChat_failed` and stop) → `GET /jwt-token/` → greeting → call token (if enabled) → `PUT /v2/client/{id}` with extra host fields → `ENABLE_DOMAIN_TRACKING`.
5. **Real-time**: socket.io connects with the JWT; subscribes to the personal channel, every conversation channel and `event_channels`.
6. **Open/close**: loader posts `CHATBOT_OPEN`/`CHATBOT_CLOSE`; iframe tracks `isChatbotOpen` (drives read receipts and unread).
7. **Logout**: `shutdown()` → `SHUTDOWN_CHATBOT` clears client-id keys and resets app state; host re-inits without `unique_id`.

AI (GTWY) lifecycle: `chatbot.js` logs in with `embedToken` (or anonymously), passes the login response to the iframe in the URL, stores `interfaceToken` in sessionStorage, fetches history for `threadId`/`subThreadId`, and listens on an RTLayer channel for replies.

## A2. State architecture

- Redux Toolkit store with slices `Chat`, `Hello`, `Interface`, `appInfo`, `draftData`; one saga (`getHelloDetails`). A custom middleware adds `action.urlData` (`chatSessionId`, `tabSessionId`) so `Hello`/`Interface` are keyed per widget session and `appInfo` per tab.
- `Chat` (not keyed by session): `messageIds[subThreadId]` newest-first, `msgIdAndDataMap[subThreadId][id]`, `rawHelloMsgList`, `isTyping`, loading flags, pagination (`currentPage`, `hasMoreMessages`, `skip`), `options` (suggestions), `images` (staged uploads), `notifications`.
- `Hello[chatSessionId]`: `widgetInfo`, `channelListData.channels` (closed last), `greeting`, `clientInfo`, agent/team maps, `socketJwt`, `mode`.
- Persistence: `Interface` + `Hello` → localStorage (`persist:root`); `appInfo` → sessionStorage; only `Chat.notifications` persisted (widget-scoped). **Messages are never persisted; they are refetched.**
- Cross-tab: `redux-state-sync` syncs only `Hello/*` and `Interface/*` (not messages).
- `components/Chatbot/hooks/chatReducer.ts` is dead code (an old `useReducer` mirror).

## A3. Message model

No formal type; normalized in `utils/dataConvertWrappers/makeGenericDataFormatUtility.tsx`.

- AI: `{id, content, role: 'user'|'assistant'|'tools_call'|'reset', createdAt, message_id?, user_feedback?, image_urls?, tools_data?, wait?, timeOut?, error?, urls?}`.
- HELLO: `{id (id || message.id || timetoken), role: 'user'|'Bot'|'Human'|'voice_call', from_name, sender_id, is_auto_response, content, urls[{path,name}], message_type: text|attachment|text-attachment|interactive|video_call|voice_call|feedback|pushNotification, messageJson, time (timetoken), show_more, replied_msg_* }`.
- Role rules (HELLO): own message (matching client id / `sender_id==='user'`) → user; `sender_id` `bot`/`workflow` or `is_auto_response` → Bot; else Human (agents and peers).
- **No send-status field** (pending/sent/failed) in either flow.

## A4. Send flow

Composer: Enter sends (Shift+Enter newline) when not disabled; disabled while uploading or empty; AI also while `loading`; HELLO only while loading **and** bot-assigned (you can keep messaging a human agent).

AI:
1. Validate (text or non-PDF image required); clear input; `loading=true`; 240 s timeout.
2. Optimistic: user message + assistant placeholder ("Talking with AI", `wait:true`).
3. `POST /chatbot/{id}/sendMessage` — only an ack.
4. Reply arrives over RTLayer: function-call interim updates, suggestions, thread rename, then final message replaces the placeholder; loading off.
5. Failure: toast + remove the two optimistic messages (text is lost). No retry button.

HELLO:
1. Read text (composer ref first, then argument), build optimistic message with a client ObjectId, prepend it.
2. New conversation: client proposes channel id `ch-comp-{companyId}.{uuid}`, pre-subscribes to it (2 s race) so the first bot reply is not missed.
3. `POST /v2/send/` with `widget_msg_id` = optimistic id, `channelDetail` for new chats, `replied_on`, attachments, `sessionVariables`.
4. Success on new chat: store channel/chat ids, re-key messages if the server chose another channel, refresh conversation list, subscribe to `channel` + `presence_channel`, skip next history fetch once.
5. Failure: toast; **optimistic message stays with no failed state**.
6. Own-message echo over socket: visible tab → merge by server id; hidden tab → prepend (possible duplicate).
7. Bot-assigned: "Thinking…" until a reply or 10 s timeout.

## A5. Receive, history, unread

- History: HELLO `POST /get-history/` (30/page, `start_from` offset) on channel change, socket reconnect, and call completion; AI `GET gethistory-chatbot` (40/page) with abort of the previous request.
- List: `column-reverse` + inverse infinite scroll (threshold 400 px); scroll to bottom on channel change and after each send; jump-to-bottom button after 500 px.
- Date separators: "Today", "Yesterday", `DD MMM, YYYY`. Per-message relative time (refreshed each minute, exact time tooltip), revealed on tap; shown by default on the newest agent/bot message.
- Live events: see `api-integration.md` A4 (`chat`, `assign`, `feedback`, `block`, `new-channel`, `update`, `Typing`).
- Unread (HELLO): incremented client-side for others' messages; reset with `DELETE /read-receipt/{channel}` (1 s debounce) when the window is open, not minimized, tab visible, drawer closed on mobile. Total (messages + notifications) posted to the host as `SET_BADGE_COUNT` (capped `99+`).
- Sound on incoming HELLO messages (`/notification-sound.mp3`, volume 0.2).

## A6. Message types and rendering

| Type | Rendering |
| --- | --- |
| User text | Escaped `<p>` with `whitespace-pre-wrap`; attachments below |
| AI assistant | react-markdown (code blocks with Prism + copy; links `target=_blank rel=noreferrer`); JSON `components` → generative UI grid; copy + 👍/👎 when `message_id` |
| HELLO bot | Markdown (`InterfaceMarkdown`, Tailwind Typography) |
| HELLO agent / peer / auto-response | Linkified text rendered as HTML (see `security-guidelines.md` before reusing) |
| Attachments | Type by extension or `HEAD` request; image (open in new tab), video/audio controls, PDF icon, generic file; download via host `downloadAttachment` |
| Interactive (WhatsApp-style) | `button`/`quick_reply`, `cta_url`, `list`, `product(_list)`, `location_request`, `carousel`; taps send the option title as a reply |
| Apple Messages for Business | Quick reply, list/time picker, rich link (unfurled via microlink), form, etc. |
| Feedback (CSAT) | 5 emoji ratings + comment → `POST /receive-feedback/`; header text comes from widget settings |
| Voice call / video call | Status pill with duration / "join" button |
| Tool calls (AI) | Collapsible "N functions executed" |
| Greeting | Synthetic bot message with quick-reply options at the top |
| Starter questions (AI) | Grid in empty view; click sends |
| Suggestions | Up to 3 chips above composer |
| Pre-chat form | "Enter your details" bar/modal/bottom sheet; name ≤26 chars, email regex, 10-digit number (+91 default); does not block sending |
| Push notifications | List in sandboxed iframes; in-thread they render the notification's HTML (see `security-guidelines.md` before reusing) |

## A7. Composer

MUI multiline (1–6 rows; 3 min in ticket mode), autofocus, placeholder "Message AI Assistant..." (not configurable), no character limit, no draft persistence (only `prefillMessage`/open-with-message). Attachments: `image/*,video/*,audio/*,application/pdf`, max 4 staged, no size check. Emoji: frimousse picker inserted at cursor. Reply: context provider, preview strip, sent as `replied_on` (HELLO only). Typing emits (HELLO) on input / 2 s idle / send. Blocked conversation replaces the composer with a notice.

## A8. UI states

Loading: top progress bar, pagination spinner, "Uploading…". Thinking: AI placeholder with shimmer; HELLO "Thinking…" only for bot-assigned. Empty: GIF + "What can I help with?" + starter questions. Errors: snackbar toasts (deduplicated), inline AI error/timeout. **Not present**: offline indicator, agent typing indicator (stored but never rendered), failed-message state, retry.

## A9. Gaps in the reference (do not copy)

- No message status or retry; HELLO failures leave a ghost message, AI failures delete the user's text.
- No dedupe on prepend; echo matching depends on the server id.
- Several message renderers do not meet `security-guidelines.md` §3 (HTML rendering, link handling). Build renderers from that section instead of porting the reference's.
- `remark-gfm` loaded with `next/dynamic` in `AssistantMessage.tsx` (not a valid remark plugin there).
- Interactive button taps send the composer draft instead of the button title when a draft exists.
- Typing indicator received but not shown; no offline UI; notification socket does not re-subscribe on reconnect.
- Reply UI shown in AI mode but ignored.

## B1. Target lifecycle (generated widgets)

`init(config)` → validate config → mount launcher (and iframe/shadow root) → restore session from namespaced storage → `ready` → on first open (or eagerly if unread counts are needed): fetch settings (if backend-managed) → identify (anonymous or with host-signed proof) → load conversation(s) + latest history page → connect transport and subscribe → `onReady`. `identify(user)` and `logout()` tear down transport, clear namespaced storage and state, then re-run identification. `destroy()` removes DOM, listeners, timers, and connections.

## B2. Target message state machine

```
draft ──send──▶ pending ──ack (server id)──▶ sent ──receipt──▶ delivered ──receipt──▶ read
                   │                                   (only if backend provides receipts)
                   └──error/timeout──▶ failed ──retry──▶ pending
```

- Insert optimistically with a `clientId`; send it to the backend if the API accepts one (like the reference's `widget_msg_id`).
- On ack, replace by `clientId` with the server message (id, timestamp).
- On socket echo, match by server id **or** `clientId`; never add a second bubble.
- On failure keep the text, show retry and allow delete.

## B3. Conversation store rules

- Messages stored as `Map<id, ChatMessage>` + ordered id list per conversation; every insert path goes through one `upsert` that dedupes by id/clientId.
- Order by server timestamp (fallback: server sequence, then client time for pending); stable for equal times.
- History pages merge with `upsert` (never replace the list while live events may have arrived; this is the race the reference works around with "skip next fetch").
- On reconnect, fetch messages newer than the last known server id/time and merge.
- Unread = incoming messages since last read while window closed or tab hidden; reset on view; sync with the backend's read API if one exists.
- Typing indicators auto-expire (e.g. 5–10 s) if no stop event arrives.
- Bot "thinking" state times out with a visible message, not silently.
