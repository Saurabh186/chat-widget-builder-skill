# MSG91 Hello environments (built-in defaults)

When the widget talks to MSG91 Hello (the reference's backend), use the host URLs below. They are confirmed by the skill owner and are **public** values (API and socket hosts, no keys), so they may ship in frontend code.

**Do not ask the user for these URLs or for a `.env` file.** Use production by default. Ask only for the **widget token**, and switch to test only if the user says the widget lives on the test environment.

## Contents

- 1. Host URLs
- 2. Env var names per codebase
- 3. How to use them in a generated widget
- 4. Widget tokens are per environment
- 5. Checking a token

## 1. Host URLs

| Purpose | Production (default) | Test |
| --- | --- | --- |
| Hello API (`widget-info`, `send`, history, …) | `https://api.phone91.com` | `https://testapi.phone91.com` |
| Chat socket (socket.io) | `https://chat.phone91.com` | `https://testchat.phone91.com` |
| Notification socket (socket.io) | `https://notifications.phone91.com` | `https://testnotifications.phone91.com` |
| Push (mobile SDK FCM token) | `https://push.phone91.com` | `https://testpush.phone91.com` |
| Static assets (intl-tel-input phone picker, Angular `main` build only) | `https://control.msg91.com/` | `https://control.msg91.com/` |

No trailing slash on the API and socket hosts; the asset host keeps one because the Angular build appends `app/assets/...` to it.

## 2. Env var names per codebase

| Purpose | `react-main` (Next.js) | `main` (Angular, `webpack.config.js` → `process.env`) | New widget (Vite example) |
| --- | --- | --- | --- |
| Hello API | `NEXT_PUBLIC_MSG91_HOST_URL` | `CHAT_WIDGET_API_URL` | `VITE_HELLO_API_URL` |
| Chat socket | `NEXT_PUBLIC_SOCKET_URL` | `CHAT_WIDGET_SOCKET_URL` | `VITE_HELLO_SOCKET_URL` |
| Notification socket | `NEXT_PUBLIC_NOTIFICATOIN_SOCKET_URL` (sic) | `CHAT_WIDGET_PUSH_NOTIFICATION_SOCKET_URL` | `VITE_HELLO_NOTIFICATION_SOCKET_URL` |
| Push | `NEXT_PUBLIC_PUSH_NOTIFICATION_URL` | not used | `VITE_HELLO_PUSH_URL` (only if push is in scope) |
| Static assets | not used | `CHAT_WIDGET_BASE_URL` | not used |

Notes:

- The Angular `main` build also needs `CHAT_WIDGET_APP_URL` (where the widget's own `assets/` are served from, e.g. `/` when they sit next to `chat-widget.js`). It is deployment-specific; set it from where the bundle is hosted, do not ask for it.
- The Angular build reads the notification socket from `CHAT_WIDGET_PUSH_NOTIFICATION_SOCKET_URL`; despite the name, give it the **notifications** host, not the push host.
- Exact env var names for a new widget follow its framework's convention; the names above are only an example.

## 3. How to use them in a generated widget

- Put the production URLs in the config layer as **defaults**, so the widget works with no `.env` at all. Env vars (or init options) only override them:

```ts
// config/hello.ts
export const HELLO_ENVIRONMENTS = {
  production: {
    apiUrl: 'https://api.phone91.com',
    socketUrl: 'https://chat.phone91.com',
    notificationSocketUrl: 'https://notifications.phone91.com',
    pushUrl: 'https://push.phone91.com',
  },
  test: {
    apiUrl: 'https://testapi.phone91.com',
    socketUrl: 'https://testchat.phone91.com',
    notificationSocketUrl: 'https://testnotifications.phone91.com',
    pushUrl: 'https://testpush.phone91.com',
  },
} as const;

export type HelloEnvironment = keyof typeof HELLO_ENVIRONMENTS;

export function resolveHelloUrls(env: HelloEnvironment = 'production') {
  const base = HELLO_ENVIRONMENTS[env];
  return {
    apiUrl: import.meta.env.VITE_HELLO_API_URL || base.apiUrl,
    socketUrl: import.meta.env.VITE_HELLO_SOCKET_URL || base.socketUrl,
    notificationSocketUrl: import.meta.env.VITE_HELLO_NOTIFICATION_SOCKET_URL || base.notificationSocketUrl,
    pushUrl: import.meta.env.VITE_HELLO_PUSH_URL || base.pushUrl,
  };
}
```

- Expose the environment as an init option (e.g. `environment: 'production' | 'test'`, default `production`) only if the user needs the test environment.
- `.env.example` lists the same variables, pre-filled with the production values and commented as optional overrides.
- When building or customizing `react-main` or `main`, write these values into the build's env (`.env.local`, or the build script's exports for the Angular build) instead of asking the user for them. Never commit a real `.env` file.

## 4. Widget tokens are per environment

A widget token exists in only one environment. A token created on test returns `401 {"message":"Invalid widget"}` from the production API, and the widget never opens. If the user gives a token, confirm which environment it belongs to (section 5) before building or deploying against it.

## 5. Checking a token

`POST {apiUrl}/widget-info/` with header `Authorization: <widgetToken>` and body `{}`:

- `200` with widget settings (`name`, `teams`, …): the token belongs to that environment.
- `401 {"message":"Invalid widget"}`: wrong environment, or the token does not exist. Try the other environment, then ask the user.

This only reads public widget settings; it sends no message and creates no data.
