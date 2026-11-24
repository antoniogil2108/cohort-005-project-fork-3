# Hosted in-app notification vendors: Knock, Novu Cloud, Courier, MagicBell

Research for issue #2 (part of map #1, "in-app Notifications for Cadence").
**Sources checked: 2026-11-24.** Every claim links to the vendor's own docs or pricing page. Pricing and SDK APIs change often, so re-check them before deciding. Anything marked **(unverified)** could not be confirmed from a primary source during this pass.

This file gives facts and a comparison only. The vendor decision belongs to a later grilling ticket.

Cadence context: React Router 8 (framework mode, SSR), React 19, Tailwind/shadcn, Node, SQLite/Drizzle. In dev, a DevUI panel switches the current user. All four roles get one combined feed/bell per person, the vendor owns read/unread state, and the scope is in-app only for now, though email/push may be added later.

## Summary table

| | Knock | Novu Cloud | Courier | MagicBell |
|---|---|---|---|---|
| Free tier | 10,000 messages/mo, unlimited environments | 10,000 workflow runs/mo, 2 envs, 5 workflows, 3 team members, Novu branding | 10,000 sends/mo | 1,000 deliveries/mo, 1 project |
| First paid tier | Starter, $250/mo (50k messages) | Pro, $30/mo (30k runs) | Business, $0.005/send, and every send is billed, including the first 10k | Startup, $249/mo (50k deliveries) |
| React feed + bell | `@knocklabs/react`: `NotificationIconButton`, `NotificationFeedPopover`, `NotificationFeed` | `@novu/react`: `<Inbox/>` (Bell, Notifications, Preferences); React 19 listed as supported | `@trycourier/courier-react` v9 (React 18+): `CourierInbox`, popup menu, toasts | `@magicbell/react`: `Inbox`, `FloatingInbox`, bell button |
| Styling vs Tailwind/shadcn | `--rnf-*` CSS vars, `rnf-` classes, `renderItem` | `appearance.elements` accepts **Tailwind class strings**; variables; dark theme | `CourierInboxTheme` objects (light/dark); slot replacement on web | CSS custom properties (`--magicbell-*`), importable stylesheets |
| Headless | Yes: `useNotifications` + `useNotificationStore`, feed client | Yes: `useNotifications` hook; `@novu/js` | Yes, per docs ("build the whole UI yourself"); API names not confirmed | `magicbell-js` user client + `Realtime` |
| Client auth | Public key + user id; **enhanced security** = RS256 JWT (`signUserToken`), recommended for prod | `applicationIdentifier` + `subscriberId`; optional **HMAC-SHA256** `subscriberHash` | **JWT** minted by backend (`auth.issueToken`, scoped) | **HS256 User JWT** signed with the project secret |
| Node trigger | Trigger workflow `key` + `recipients` (ids or inline) + `data`/`actor`/`tenant` | `@novu/api` `trigger({ workflowId, to, payload })` | `@trycourier/courier` `send.message({ message: { to, template \| content, data, routing } })` | `POST /v2/broadcasts` with `recipients` (Project JWT) |
| Idempotency | `Idempotency-Key`, 24h | `Idempotency-Key`, but **needs enabling by Novu support**; `transactionId` | `Idempotency-Key`, 25h default, up to 1 year | Not documented |
| Vendor-owned state | seen / read / interacted / archived (+ unread/unarchive) | read / unread / seen / archive / snooze / delete | read / opened / archived | read / unread / archive (+ seen via API) |
| Real-time | WebSocket; cross-tab option | WebSocket (on by default) | Real-time connection authed by the JWT | WebSocket `Realtime` client |
| Dev/prod separation | Environments with separate keys/data, commit + promote | Dev + Prod (custom envs on paid plans), publish | Test + Production environments, separate keys | Free = 1 project; no environment concept documented |
| Email/push later | Email, SMS, push, chat, webhook | Email, SMS, chat, push | Email, SMS, push, chat, with routing/failover | Email, mobile push, web push, SMS, Slack |

## 1. Free tier and what's paid

- **Knock.** Developer plan is $0 with "up to 10,000 messages sent per month", unlimited environments, workflows, channels, and team members, and built-in multi-tenancy. Starter is $250/mo with 50,000 messages included, then $0.005 each. A message is "a single message delivered to a single user on a single channel", and bounces don't count. https://knock.app/pricing
- **Novu Cloud.** Free plan is $0: 10,000 workflow runs/mo, in-app/email/SMS/chat/push, 2 environments (Dev, Prod), 24h activity-feed retention, up to 5 workflows, and up to 3 team members. Pro is $30/mo (30k runs, $1.20 per extra 1K) and removes Novu branding. Team is $250/mo. https://novu.co/pricing ; on removing branding: https://docs.novu.co/platform/inbox/prepare-for-production
  - Note: **5 workflows** on Free could become the real limit if each notification type is its own workflow.
- **Courier.** Developer is $0 with up to 10k sends/mo. One template sent to one recipient is one send, and in-app counts like any other channel. Business is $0.005 per send, and "every send is charged, including the first 10,000". https://www.courier.com/pricing
- **MagicBell.** Builder is $0 with **1,000 deliveries/mo**, 1 project, and unlimited users and team members. Startup is $249/mo with 50,000 deliveries, then $0.0025 each, and 5 projects. https://www.magicbell.com/pricing
  - The page does not say whether Builder has branding or inbox restrictions **(unverified)**.

## 2. Prebuilt React feed/bell and styling vs Tailwind + shadcn

- **Knock.** `@knocklabs/react` provides `KnockProvider` (apiKey, user), `KnockFeedProvider` (feedId, `colorMode`), `NotificationIconButton` (`badgeCountType`), `NotificationFeedPopover`, and `NotificationFeed` (`renderItem`, `renderHeader`). Styling works through `--rnf-*` CSS variables. You can also replace Knock's CSS entirely, since all classes are prefixed `rnf-`, or swap cells via `renderItem`. https://docs.knock.app/in-app-ui/react/feed
  - Headless: `useKnockClient`, `useNotifications(client, feedId)` and `useNotificationStore(feed)` give you `items`, `unread_count`, and `unseen_count`. https://docs.knock.app/in-app-ui/react/headless/feed
  - Not confirmed: whether React 19 is officially supported **(unverified)**.
- **Novu.** `@novu/react` provides `<Inbox/>`, made of Bell, Notifications, InboxContent, and Preferences. It requires React 18+, and the docs say React 19 is supported. https://docs.novu.co/platform/inbox/overview
  - Styling: the `appearance` prop takes `baseTheme`, `variables`, `elements`, `icons`, and `animations`. `elements` values can be style objects or **CSS class strings, with Tailwind classes shown in the docs**. A dark theme is built in. This is the closest match to shadcn tokens. https://docs.novu.co/platform/inbox/configuration/styling
  - Headless: the `useNotifications` hook returns notifications, pagination, and `readAll`/`seenAll`/`archiveAll`, and each notification has `read`/`unread`/`seen`/`archive`/`unarchive`/`snooze`/`delete` methods. https://docs.novu.co/platform/sdks/react/hooks/use-notifications
  - The `useCounts` doc page returned HTTP 500 during this check **(unverified)**.
- **Courier.** `@trycourier/courier-react` v9 targets React 18+, and v9 of `-react-17` covers React 17. It provides `CourierInbox`, popup-menu components, and toasts. https://www.courier.com/docs/platform/inbox/inbox-overview
  - Theming: `lightTheme`/`darkTheme` take `CourierInboxTheme` objects (popup, header, list, empty, error), with `mode: light | dark | system`. The web SDKs support slot replacement. Tailwind and CSS classes are not mentioned. https://www.courier.com/docs/in-app/customize-the-inbox
  - The docs say a fully custom UI is possible by reading the datastore and its change events, but hook names were not confirmed **(unverified)**.
  - React 19 is not explicitly stated **(unverified)**.
- **MagicBell.** `@magicbell/react` provides a ContextProvider (`token`, `toasts`), `Inbox`, and `FloatingInbox` (`ButtonComponent`, default `InboxButton` bell). Styling uses importable stylesheets and CSS custom properties such as `--magicbell-bg-default`. https://www.magicbell.com/docs/libraries/magicbell-react
  - Headless: `magicbell-js/user-client` offers list, read/unread, archive/unarchive, and mark-all calls, and `magicbell-js/realtime` offers a WebSocket listener. https://www.magicbell.com/docs/libraries/magicbell-js
  - React hooks are not documented on the React page **(unverified)**.

## 3. Recipient identity and client feed auth

- **Knock.** The recipient is your own user id string. The client uses the public API key plus the user id. In **Enhanced security mode**, requests without a signed user token are rejected: an RS256 JWT with `sub` = user id, signed with a private signing key, generated locally by `signUserToken` from `@knocklabs/node` (default lifetime 3600s), and passed as `userToken` to `KnockProvider`. The docs say production "should" enable it, because otherwise "a request with your public key can act as any user". https://docs.knock.app/in-app-ui/security-and-authentication
- **Novu.** The recipient is a `subscriberId`, your own id. The client uses the public `applicationIdentifier` plus `subscriberId`. Optional **HMAC** mode uses `subscriberHash` = HMAC-SHA256(secretKey, subscriberId), hex encoded and computed server-side with Node `crypto`. Once enabled in the In-App integration, the Inbox "will not load" without it. https://docs.novu.co/platform/inbox/prepare-for-production
- **Courier.** The recipient is a `user_id`. The backend mints a JWT with `client.auth.issueToken({ scope: "user_id:<id> inbox:read:messages inbox:write:events", expires_in })`, and the client calls `signIn({ userId, jwt })`. Omitting `expires_in` means the token never expires. No unauthenticated or HMAC mode is documented. https://www.courier.com/docs/in-app/authenticate-users
- **MagicBell.** The recipient is an `external_id` or email, and the docs recommend external ids. The User JWT carries `user_external_id` and/or `user_email` plus `api_key`, HS256-signed with the project Secret Key using `jsonwebtoken`. https://www.magicbell.com/docs/api/authentication/user ; https://www.magicbell.com/docs/primitive/user

For Cadence, all four fit an SSR loader that reads the session user and returns `{ userId, token }` to the client. The signing is either local crypto (Knock JWT, Novu HMAC, MagicBell JWT) or **a network call** (Courier `issueToken`).

## 4. Triggering from Node

- **Knock.** You trigger by workflow `key` with `recipients` as ids or inline recipient objects, which Knock guarantees to identify before the run. Optional fields are `data` (JSON up to 10MB), `actor`, `tenant`, and `cancellation_key`. `Idempotency-Key` makes retries within 24h return the original response, and a key reused with different params returns an error. The call returns `workflow_run_id`. https://docs.knock.app/send-notifications/triggering-workflows/api
  - The exact `@knocklabs/node` method signature was not visible on the page **(unverified)**.
- **Novu.** `@novu/api`: `new Novu({ secretKey }).trigger({ workflowId, to, payload, transactionId? })`. `to` takes up to 100 recipients, and subscriber objects (subscriberId, email, name) are accepted. `Idempotency-Key` is supported on `POST /v1/events/trigger` (a 409 on concurrent duplicates), but it "is not enabled for all organizations" and has to be enabled by Novu support. `transactionId` is checked during processing and "is not a substitute for idempotency". https://docs.novu.co/api-reference/events/trigger-event
- **Courier.** `@trycourier/courier`: `client.send.message({ message: { to: { user_id }, template | content, data, routing } })` returns `202` with a `requestId`. `Idempotency-Key` is scoped to `/send`, with a 25h default lifetime that `x-idempotency-expiration` can extend up to 1 year. https://www.courier.com/docs/reference/send/message
- **MagicBell.** `POST https://api.magicbell.com/v2/broadcasts` with a Project JWT. The body has `title` (required), `content`, `action_url`, `category`, `topic`, and `custom_attributes`, plus 1–1000 `recipients` (external_id/email) and per-channel `overrides`. The project client lives at `magicbell-js/project-client`. **No idempotency mechanism is documented.** https://www.magicbell.com/docs/primitive/broadcast ; https://www.magicbell.com/docs/api/reference ; https://www.magicbell.com/docs/libraries/magicbell-js

## 5. Read / unread / seen / archive owned by the vendor

- **Knock.** Statuses are `seen_at`, `read_at`, `interacted_at`, and `archived_at`. In-app seen/interacted/archived are managed implicitly. The feed client has `markAsSeen`, `markAsRead`/`markAsUnread`, `markAsArchived`/`markAsUnarchived`, `markAllAsRead`, and `markAllAsArchived`. The popover marks items seen on open by default. https://docs.knock.app/send-notifications/message-statuses ; https://docs.knock.app/in-app-ui/javascript/sdk/feed-client
- **Novu.** read/unread, seen, archive/unarchive, snooze/unsnooze, and delete, plus bulk `readAll`, `seenAll`, `archiveAll`, and `archiveAllRead`. https://docs.novu.co/platform/sdks/react/hooks/use-notifications
- **Courier.** Read, opened, and archived states sync across devices and persist across sessions. A "seen" state is not documented **(unverified)**. https://www.courier.com/docs/in-app/how-the-inbox-behaves
- **MagicBell.** read/unread, archive/unarchive, and mark-all. The API reference lists "Mark a notification seen" and "Mark all notifications seen", but the JS user client page shows no `markAsSeen` **(partially verified)**. https://www.magicbell.com/docs/libraries/magicbell-js ; https://www.magicbell.com/docs/api/reference

## 6. Real-time delivery to an open page

- **Knock.** WebSocket, and you may need to allow Knock's socket hosts in your CSP. Cross-tab sync is opt-in via `__experimentalCrossBrowserUpdates` (BroadcastChannel). Sockets in inactive tabs disconnect automatically. `listenForUpdates` and the `items.received.realtime` event are available for headless use. https://docs.knock.app/in-app-ui/react/feed ; https://docs.knock.app/in-app-ui/javascript/sdk/feed-client
- **Novu.** WebSocket subscription is on by default (`realtime` on NovuProvider/hooks), and there is a `notifications.notification_received` event. https://docs.novu.co/platform/sdks/react/hooks/use-notifications
- **Courier.** A "realtime connection" with no separate auth beyond the JWT. The transport is not named on the page **(unverified)**, and tab-level behavior isn't documented. https://www.courier.com/docs/in-app/how-the-inbox-behaves
- **MagicBell.** "Persistent connection" for in-app, and `magicbell-js/realtime` is described as WebSocket. https://www.magicbell.com/docs/channels/in_app ; https://www.magicbell.com/docs/libraries/magicbell-js

## 7. Local dev with many switchable users (DevUI panel)

None of the four offers an offline emulator. Local dev talks to the vendor's cloud dev environment/project. Some points are inferred from the auth model rather than documented:

- **Switching users works with any of them** as long as the client provider re-initializes when the DevUI changes the session user. In practice: key the provider on `userId` and return a fresh token from the loader. Because every vendor identifies recipients by your own id string, seed users map 1:1 with no extra lookup table. (Inference.)
- **Knock.** Separate development/production environments with separate keys and data. Test users live in development. Content is committed and promoted. You can leave Enhanced security off in development so the dev loop needs only the user id, but production should have it on. https://docs.knock.app/concepts/environments ; https://docs.knock.app/in-app-ui/security-and-authentication
  - Inline recipients on trigger mean seeded users don't need a separate identify step. https://docs.knock.app/send-notifications/triggering-workflows/api
- **Novu.** Development/Production keep subscriber data separate, and workflows are published from Dev to Prod. https://docs.novu.co/platform/concepts/environments
  - HMAC can be left off in dev, since it is enabled per environment's In-App integration.
  - The optional code-first **Novu Framework** adds `npx novu@latest dev`, a Local environment in the dashboard, and a tunnel to a Bridge endpoint (default `localhost:4000/api/novu`). Dashboard-built workflows apparently don't need it, though this wasn't confirmed **(unverified)**. https://docs.novu.co/framework/studio
  - Novu is open source and self-hostable, but self-hosting wasn't researched here **(unverified)**.
- **Courier.** Test and Production environments with separate keys. https://www.courier.com/docs
  - Every user switch needs a server round-trip to `issueToken`, which you can cache per user in dev.
- **MagicBell.** Free tier = **1 project**, and there's no documented dev/prod environment split on Free. Dev and prod would share one project unless you pay ($99 per extra project on Startup). https://www.magicbell.com/pricing

## 8. Adding email/push later without re-platforming

All four are multi-channel, so adding channels later is a workflow/config change, not a migration.

- **Knock.** Email (SendGrid, Postmark), SMS, push (APNs, FCM), chat (Slack, Teams, Discord), in-app feed, and webhook. https://docs.knock.app/integrations/overview
- **Novu.** Free tier includes in-app, email, SMS, chat, and push. https://novu.co/pricing
- **Courier.** Email, SMS, push, inbox, and chat (Slack, Teams, WhatsApp), with routing and failover from one send. https://www.courier.com/docs
- **MagicBell.** Broadcast `overrides` for in_app, email, mobile_push, and sms, plus web_push/APNs/FCM/Expo providers and Slack. https://www.magicbell.com/docs/primitive/broadcast ; https://www.magicbell.com/docs/channels/in_app
  - Every channel counts against **1k deliveries** on Free.

## Non-binding leaning

These points are for the decision ticket, not a recommendation:

- **Novu Cloud and Knock** look like the strongest fits.
- **Novu** advantages:
  - Its Inbox accepts Tailwind class strings, which suits shadcn.
  - HMAC auth is a few lines of Node `crypto`.
  - The step to paid ($30) is cheap.
- **Novu** watch-outs:
  - 5 workflows and Novu branding on Free.
  - Idempotency must be enabled by support.
- **Knock** advantages:
  - The most complete documented state model (seen/read/interacted/archived) and headless API.
  - First-class idempotency.
  - Unlimited environments on Free.
- **Knock** watch-out: the next tier jumps to $250/mo.
- **Courier**: workable, but the token mint is a network call and the free tier turns into pay-per-send from send one.
- **MagicBell**: weakest on free tier (1k deliveries, 1 project) and undocumented idempotency.

## Not verified / gaps

- Knock `@knocklabs/node` trigger method signature, and official React 19 support for Knock, Courier, and MagicBell.
- Novu `useCounts` page (returned HTTP 500), whether dashboard-built workflows need a Bridge, and the self-hosting option.
- Courier "seen" state, realtime transport, multi-tab behavior, headless hook names, and Tailwind compatibility.
- MagicBell Free-plan branding, a JS-level mark-seen call, idempotency, and dev/prod separation.
