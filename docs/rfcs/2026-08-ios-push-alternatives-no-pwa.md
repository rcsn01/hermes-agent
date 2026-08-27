# Research spike: iOS push notifications WITHOUT a PWA (no Apple Developer account)

**Status:** research only — no code, no design commitment

**Question:** What are ALL the ways to receive push notifications on the iPhone
(from a self-hosted Hermes gateway) that do **not** require a paid Apple
Developer account and do **not** use the notification-PWA approach from
[the previous spike](2026-08-pwa-push-notification-spike.md)? Shortcuts
automations and any other mechanism included.

**Method.** Primary sources: project documentation and GitHub repos (ntfy, Bark,
Pushcut, Pushover, Home Assistant companion, PushDeer), Apple's support docs.
DDG search was unavailable during the session; sources were fetched directly
from known primary URLs. Cross-reference: the PWA path is covered in
[the Web-Push spike](2026-08-pwa-push-notification-spike.md).

---

## Headline: the mental model

On iOS, **all background delivery goes through APNs**, and APNs credentials
require a paid Apple Developer Program membership. So without an account there
are exactly three families of options:

1. **Borrow someone else's APNs entitlement** — a relay app already on the App
   Store (its publisher holds the entitlement) that accepts your events over
   plain HTTP and re-delivers them as native push. Either hosted-only
   (Pushover, Pushcut) or a **self-hosted server + their free iOS app** (Bark,
   ntfy). This is the entire category — and it's where the good options live.
2. **Ride a messaging platform's push** — Telegram/Discord/Slack/etc. bots
   produce native iOS push for free. Hermes *already ships adapters* for ~20
   platforms (Telegram, Discord, Slack, Signal, WhatsApp, …), so this option
   exists in the product today with zero new code (repo `AGENTS.md`, platform
   list).
3. **Device-local automation (Shortcuts)** — automations react to device-local
   events (time, location, Wi-Fi, focus, NFC, app open/close…). There is **no**
   trigger for remote/server events, so Shortcuts alone cannot receive push —
   it can only poll (and even polling needs a trigger). Pushcut exists
   precisely to bridge this gap.

Everything below verified against primary sources; dead ends included.

---

## Option A: self-hosted relay + their iOS app (the sweet spot)

### 1. Bark — self-hosted server, free iOS app ⭐ strongest fit

- Free, open-source iOS app purpose-built for "push custom notifications to
  your iPhone"; uses APNs; supports notification grouping, custom icons,
  sounds, **time-sensitive** (breaks through Focus) and **critical alerts**
  (ignore silent/DND), and **encrypted push**.
  ([Finb/Bark README](https://github.com/Finb/Bark))
- Self-hostable Go server (Docker image, single binary; stores device tokens,
  MySQL or bbolt) or use their hosted `api.day.app`.
  ([Finb/bark-server](https://github.com/Finb/bark-server))
- API is a one-liner: `GET/POST /:key/:title/:body`, with params for `url`
  (tap → open URL), `group`, `icon`, `sound`, `call` (repeat sound 30s),
  `level=timeSensitive|critical|active`, `ciphertext` (encrypted payload).
  ([Bark README usage](https://github.com/Finb/Bark))
- Tap-through: the `url` param opens a URL on tap — if Hermes Mobile registers
  a custom scheme (`hermes://…`, no Apple account needed), a Bark tap should
  deep-link straight into the app. *Verify on device (custom-scheme taps from a
  third-party app are normally fine, but verify).*
- Risks: single-maintainer project; docs are bilingual with the community
  centered in China; the App Store app owns the APNs cert (self-hosting the
  server does **not** remove the app dependency). Server must be reachable from
  the phone when Bark fetches, or use hosted endpoint.

### 2. ntfy (self-hosted) + official iOS app ⭐ best FOSS ecosystem

- The official iOS app (App Store, MIT-licensed) subscribes to topics on your
  server; publishing is a plain HTTP POST with headers (title, priority, tags,
  **Click URL**, **action buttons**, email forwarding).
  ([ntfy publish docs](https://docs.ntfy.sh/publish/),
  [iOS app](https://apps.apple.com/us/app/ntfy/id1625396347))
- **iOS reality check:** iOS has no foreground services and background app
  refresh is unreliable ("executed only once in the day" in ntfy's own tests).
  For **self-hosted servers**, instant delivery therefore works by setting
  `upstream-base-url: https://ntfy.sh` on your server: your server relays a
  poll signal through ntfy.sh, which holds the APNs credentials and wakes the
  app; the app then fetches the actual message from *your* server.
  ([ntfy-ios TECHNICAL_LIMITATIONS](https://github.com/binwiederhier/ntfy-ios/blob/main/docs/TECHNICAL_LIMITATIONS.md),
  [ntfy config docs — iOS compose example with `NTFY_UPSTREAM_BASE_URL`](https://docs.ntfy.sh/config/))
- Consequence: instant iOS push on a self-hosted server **depends on ntfy.sh
  being reachable** (as a poll-signal relay), plus your server for content.
  Topic name is the credential ("the topic is essentially a password").
  ([ntfy publish docs](https://docs.ntfy.sh/publish/))
- Bonus: ntfy also ships the Web-Push PWA path from the previous spike
  (`NTFY_WEB_PUSH_*`), so one server covers PWA + native-app + desktop clients.

### 3. Home Assistant companion app (if HA is already in the stack)

- `notify.mobile_app_<device>` sends real push; supports **actionable
  notifications** (buttons) and tap-through to a URL (custom or lovelace view).
  ([companion docs: basic notifications](https://companion.home-assistant.io/docs/notifications/notifications-basic/))
- Push is relayed through Home Assistant's hosted push service — no Apple dev
  account, no HA Cloud subscription needed. *Delivery-chain detail flagged as
  standard-setup knowledge; verify current proxy docs if this route is chosen.*
- Requires running Home Assistant. Hermes already has a `homeassistant`
  platform and toolset, so this composes naturally if the user runs HA.

### 4. Gotify — self-hosted, but iOS is the weak leg

- Gotify's iOS client is historically WebSocket/foreground-based without APNs
  instant delivery; given iOS's documented hostility to background polling and
  sockets ([ntfy-ios TECHNICAL_LIMITATIONS](https://github.com/binwiederhier/ntfy-ios/blob/main/docs/TECHNICAL_LIMITATIONS.md)),
  expect unreliable background delivery. Android-first; verify current iOS app
  state before choosing.

### 5. PushDeer — **avoid** (dead)

- Officially unmaintained: "本项目已不在维护" (project no longer maintained);
  hosted Android app broken; self-hosted requires **annual push-certificate
  renewal**; recommends migrating to a paid successor. A cautionary example of
  the failure mode for small APNs-entitled apps.
  ([PushDeer README warning](https://github.com/easychen/PushDeer))

---

## Option B: hosted relay apps (no server to run)

### 6. Pushover — the boring, reliable one ($5 once, no subscription)

- *"no subscription and just a simple one-time in-app purchase on each platform
  where you need it, after a 30-day free trial"* — iOS client + dead-simple
  HTTP API (`POST https://api.pushover.net/1/messages.json` with
  `token`/`user`/`message`). ([Pushover](https://pushover.net/))
- Battle-tested for a decade; supports URL tap-through, priorities, emergency
  repeat. Closed-source client; notifications transit their cloud; device
  tokens belong to their app (fine — that's the borrowed entitlement).

### 7. Pushcut — Shortcuts-native smart notifications

- iOS app that delivers notifications triggered by **webhook/Web API**; every
  notification can carry **actions**: run a Shortcut, open a URL or **custom
  scheme** (their own examples: `omnifocus:///add?…` — so `hermes://…` fits),
  call a web request, or run it on their **Automation Server** in background.
  ([Pushcut notifications docs](https://pushcut.io/support/notifications))
- **Free tier is limited to pre-defined notifications without dynamic
  content**; dynamic title/body via API needs Pushcut Pro ($1.99/mo, $17.99/yr,
  $39.99 lifetime).
  ([Pushcut pricing](https://pushcut.io/),
  [notifications docs](https://pushcut.io/support/notifications))
- This is the closest thing to the "Shortcuts automation" idea: a device-local
  Shortcuts/HomeKit automation can call "Get contents of URL" → Pushcut webhook
  ([Pushcut docs](https://pushcut.io/support/notifications)), and Pushcut's
  Automation Server runs Shortcuts 100% in the background on a webhook — but
  the **server→phone** direction is Pushcut's own hosted relay.

---

## Option C: messaging platforms (zero new code — already in Hermes)

- **Telegram bots**: free, instant native push, inline keyboards + deep links;
  the gateway can also receive button callbacks as commands. Hermes ships a
  Telegram adapter today (repo `AGENTS.md` platform list; `gateway/platforms/`).
- Same story for **Discord, Slack, Signal, WhatsApp, SMS, email, …** — all
  already integrated. iOS delivers these as normal push; per-conversation Focus
  filtering works; no Apple account needed.
- Tradeoffs: notifications arrive inside the chat app (no per-app Hermes icon
  granularity beyond the platform app), content transits the platform's cloud,
  and Telegram bot messages from the agent share the same thread as chat.

---

## Option D: Shortcuts automations — what the phone can do alone

- Personal automations fire on **device-local events**: time of day, alarms,
  sleep, arrive/leave, Wi-Fi/Bluetooth, focus, NFC, app open/close, battery,
  sound recognition, CarPlay, etc. There is **no remote/server event trigger
  and no "on notification" trigger**; a server cannot wake a shortcut.
  (Trigger catalog per Apple's Shortcuts/iPhone user guides,
  [Shortcuts user guide](https://support.apple.com/guide/shortcuts/intro-to-personal-automations-iphone-apdc33a9e3cc/ios)
  — catalog as of current iOS; verify in-app.)
- With no push trigger and no timer shorter than time-of-day, Shortcuts alone
  cannot implement server-event push. Closest hack: an automation on a frequent
  device event that runs "Get contents of URL" to poll the gateway — coarse,
  battery-hostile, and gated by iOS scheduling. **Shortcuts + Pushcut (or +
  Bark's share-sheet) is the practical shortcut integration, not a standalone
  channel.**
- Exotic unverified idea (not recommended): an iMessage-sending bridge (needs a
  always-on Mac) + a Shortcuts "Message received" automation. Fragile; noted
  for completeness only.

---

## Comparison

| Option | Cost | Self-hosted relay | iOS background push | Dynamic content | Tap into Hermes app | Setup |
|---|---|---|---|---|---|---|
| **Bark** (self-hosted server + app) | Free (app + server, OSS) | ✅ server; APNs via their app | ✅ instant, incl. time-sensitive/critical | ✅ full | `url` param → `hermes://` (verify) | Run tiny Go server; one GET |
| **ntfy** (self-hosted + iOS app) | Free, OSS | ✅ (upstream relay via ntfy.sh for iOS instant) | ✅ instant via upstream | ✅ full (headers) | Click URL (custom scheme — verify) | Run ntfy server + set `upstream-base-url` |
| **Pushover** | $5 one-time/platform | ❌ hosted | ✅ instant, very reliable | ✅ full | URL param | Install app, POST from gateway |
| **Pushcut** | Free (predefined) / Pro $39.99 lifetime | ❌ hosted | ✅ instant | ⚠️ Pro only | ✅ custom-scheme actions documented | Install app, define notification, webhook |
| **Telegram** (existing gateway) | Free | ❌ (Telegram cloud) | ✅ instant | ✅ full | Inline buttons / deep links | **Zero — already shipped** |
| **HA companion** | Free | ✅ HA server (push proxied by HA) | ✅ instant + actionable buttons | ✅ full | URL / lovelace view | Only if HA already runs |
| **Web-push PWA** (prev. spike) | Free | ✅ gateway | ✅ iOS 16.4+, home-screen install | ✅ (4 KB) | 2-hop via page | Serve PWA + pywebpush |
| **Gotify** | Free, OSS | ✅ | ⚠️ weak (no APNs path) | ✅ | basic | Run server; iOS caveat |
| **PushDeer** | — | — | ⚠️ unmaintained | — | — | **avoid** |
| **Shortcuts alone** | Free | — | ❌ no remote trigger | — | — | not viable alone |

## Verdict

For "self-hosted, no Apple account, native-feeling push, tap opens Hermes":

1. **Bark** is the strongest single choice: free app, self-hostable one-binary
   server, instant APNs delivery, time-sensitive/critical levels, one-line API,
   tap-through URL. The tap-to-`hermes://` behavior is the one thing to verify
   on device.
2. **ntfy** if the FOSS/multi-client story matters (one server → iOS app,
   Android, desktop clients, and the Web-Push PWA from the previous spike) —
   accepting that iOS instant delivery relays a poll signal via ntfy.sh.
3. **Telegram (or any existing Hermes platform)** is the zero-effort option
   that works today, at the cost of riding a messaging cloud.
4. **Pushover** if a one-time $5 for maximum reliability/boringness is fine.
5. **Pushcut** if the user wants Shortcuts actions attached to notifications
   (and accepts Pro for dynamic content).

All gateway-side integrations are trivial HTTP POSTs — the same shape as the
web-push relay in the previous spike; per repo policy, a notifier for a
third-party product belongs in a **standalone plugin repo** (`~/.hermes/plugins/`),
with keys in `.env` and behavior config in `config.yaml`.

## Open questions / verify live

1. Bark tap-through to a custom URL scheme (`url=hermes://…`) — expected to
   work; verify on device.
2. ntfy upstream relay privacy semantics (what exactly ntfy.sh sees for
   self-hosted topics — poll signals vs content) before trusting it with
   sensitive payloads.
3. Pushcut free-tier limits in practice (predefined notifications only).
4. HA push proxy details (free hosted APNs relay, no Cloud needed) — confirm
   against current docs if that route is chosen.
5. Whether Bark/ntfy's App Store apps can open custom schemes reliably from
   notification taps on current iOS.

## Sources

- [Bark (iOS app) — GitHub](https://github.com/Finb/Bark) · [bark-server](https://github.com/Finb/bark-server) — free app, self-hosted server, API params (url/group/icon/sound/call/timeSensitive/critical/ciphertext)
- [ntfy: subscribe from your phone](https://docs.ntfy.sh/subscribe/phone/) · [publish](https://docs.ntfy.sh/publish/) · [config (upstream-base-url / iOS docker example)](https://docs.ntfy.sh/config/) · [ntfy-ios TECHNICAL_LIMITATIONS](https://github.com/binwiederhier/ntfy-ios/blob/main/docs/TECHNICAL_LIMITATIONS.md)
- [Pushcut](https://pushcut.io/) · [notifications/webhook docs](https://pushcut.io/support/notifications) — actions incl. custom schemes, free-tier limits, pricing
- [Pushover](https://pushover.net/) — one-time purchase model, HTTP API
- [Home Assistant companion: notifications](https://companion.home-assistant.io/docs/notifications/notifications-basic/) — `notify.mobile_app`, actionable notifications, URL tap-through
- [PushDeer README](https://github.com/easychen/PushDeer) — maintenance-status warning (unmaintained; annual cert renewal)
- [Apple Shortcuts user guide](https://support.apple.com/guide/shortcuts/intro-to-personal-automations-iphone-apdc33a9e3cc/ios) — automations context (trigger catalog flagged)
- Hermes repo `AGENTS.md` / `gateway/platforms/` — existing Telegram/Discord/Slack/Signal/WhatsApp/email/SMS adapters
- Cross-reference: [PWA Web-Push spike](2026-08-pwa-push-notification-spike.md)