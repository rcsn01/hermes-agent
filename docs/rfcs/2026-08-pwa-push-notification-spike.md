# Research spike: notification-only PWA for push (no Apple Developer account)

**Status:** research only — no code, no design commitment

**Question:** Can we ship a PWA whose *only* job is receiving and displaying push
notifications, while all real functionality stays in Hermes Mobile (the Capacitor
iOS client in `apps/mobile/`)? Motivation: remote push to a native iOS app requires
the APNs push entitlement, which requires a paid Apple Developer Program
membership; the user doesn't have one.

**Method.** Primary sources only: WebKit blog posts, Apple developer documentation,
the WWDC22 "Meet Web Push" session transcript, MDN, caniuse, Capacitor docs,
Tailscale docs, and the ntfy project's web-push documentation (as a shipped
precedent). Every claim below carries its source. DDG search was unavailable
during the session; sources were fetched directly from known primary URLs.

---

## Headline verdict

**Yes — this works, and it is exactly the designed use of Web Push on iOS.**
Since iOS/iPadOS 16.4, a web app **added to the Home Screen** can receive real
system push notifications (Lock Screen, Notification Center, Apple Watch, Focus)
through the standard Web Push stack (Push API + Notifications API + Service
Worker), and Apple states explicitly: *"You do not need to be a member of the
Apple Developer Program to use it."*
([WebKit, "Web Push for Web Apps on iOS and iPadOS"](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/);
[Apple, "Sending web push notifications in web apps and browsers"](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers))

The PWA can be minimal: one static page (manifest + one "Enable notifications"
button) + a service worker with `push` and `notificationclick` handlers. All
subscribing, sending, and content-producing logic stays with the gateway and the
Capacitor app. The PWA is a per-device notification *sink*, nothing more.

The hard constraints that shape it:

| Constraint | Consequence | Source |
|---|---|---|
| iOS 16.4+ only, Home-Screen-installed web apps only | Users on ≤16.3 or who don't install get nothing; a plain browser tab cannot receive push | [WebKit 16.4](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/), [caniuse push-api](https://caniuse.com/push-api) (Safari on iOS = "partial support") |
| Permission + subscription require a **user gesture inside the installed web app** | The PWA needs one real screen with an "Enable notifications" button; it cannot be fully zero-UI | [WebKit 13878](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/), [Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers), [MDN subscribe()](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe) |
| `userVisibleOnly` is mandatory; **no silent/data-only push**; a push that doesn't produce a visible notification gets the permission **revoked** | Every push must render a notification — fine for a notification sink, but push cannot be used as an invisible data-sync channel | [WebKit, "Meet Web Push"](https://webkit.org/blog/12945/meet-web-push/), [Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers) |
| Encrypted payload ≤ **4 KB** (HTTP 413 `PayloadTooLarge` beyond) | Send compact JSON (title/body/deep-link/ids), not content blobs | [Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers) |
| Subscription is **per-origin, per-device, per-browser** | The PWA must be served by (or same-origin with) the thing that stores subscriptions — i.e., the gateway | [MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API), [ntfy](https://docs.ntfy.sh/subscribe/web/) |
| Service workers require a **secure context** (HTTPS) | The gateway origin needs a real TLS cert — `tailscale serve` provides automatically provisioned certs | [Tailscale serve](https://tailscale.com/kb/1242/tailscale-serve) |
| The Capacitor app **cannot** host Web Push itself | Web push surfaces are enumerated exhaustively by Apple as "Home Screen web apps in iOS 16.4 or later and Webpages in Safari 16 for macOS 13 or later" — a WKWebView inside a Capacitor app is neither; it has no `webpushd` integration | [Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers), [WebKit 13878](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/), [caniuse](https://caniuse.com/push-api) |

---

## Findings

### 1. iOS requirements and the minimal viable "notification sink"

- Web Push arrived in iOS/iPadOS **16.4**, only for web apps added to the Home
  Screen via Share → Add to Home Screen. A site with a manifest whose `display`
  is `standalone` or `fullscreen` becomes a Home Screen *web app*; without it,
  the site is saved as a mere bookmark, which gets no push.
  ([WebKit 13878](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/))
- Nothing about iOS installability requires meaningful UI or a service-worker
  `fetch` handler — any website can be added to the Home Screen; the manifest
  (plus `apple-touch-icon` or manifest icons) determines web-app vs bookmark.
  The service worker only needs `push` and `notificationclick` handlers; Apple's
  own canonical walkthrough ("Browser Pets") registers a SW and calls
  `pushManager.subscribe({ userVisibleOnly: true, applicationServerKey })` from a
  button's click handler, with nothing else.
  ([WebKit 13878](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/),
  [WWDC22 session](https://developer.apple.com/videos/play/wwdc2022/10098/))
- The permission flow is: user taps a button in the installed web app →
  `subscribe()` must be called *immediately* in the gesture's event handler →
  iOS shows the system notification prompt → on grant, the JS receives a
  `PushSubscription` (endpoint URL + ECDH keys) to send to the server.
  ([WebKit 13878](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/),
  [Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers))
- Under the hood iOS routes web pushes through the same APNs that native apps
  use (a system daemon, `webpushd`, holds the APNs subscription; the app server
  just posts RFC 8030 requests to `https://*.push.apple.com` endpoints). Server
  egress allowlists must permit `*.push.apple.com`.
  ([WebKit 12945](https://webkit.org/blog/12945/meet-web-push/),
  [Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers))
- Delivery is fully background: the system wakes the service worker on push even
  when the web app is closed (and, on macOS, even when Safari isn't running);
  ntfy's shipped table confirms iOS delivery works "Browser Running ✅ / Not
  Running ✅". Notifications integrate with Focus, per-app notification settings,
  and the Badging API (`setAppBadge`/`clearAppBadge`).
  ([WebKit 13878](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/),
  [WWDC22](https://developer.apple.com/videos/play/wwdc2022/10098/),
  [ntfy](https://docs.ntfy.sh/subscribe/web/))
- **Missing on iOS web push:** notification action buttons (reply/archive/etc.).
  ntfy's browser-support notes: "Only Chrome, Edge, and Opera support displaying
  view and http actions in notifications." Treat actions as unavailable on iOS.
  ([ntfy](https://docs.ntfy.sh/subscribe/web/))

### 2. Why the PWA (and not something inside the Capacitor app)

- Apple's web-push doc enumerates the supported surfaces exhaustively: *"Add web
  push to Home Screen web apps in iOS 16.4 or later and Webpages in Safari 16
  for macOS 13 or later."* A Capacitor app's WKWebView is neither. WebKit's
  `webpushd` integration is per-platform/per-application work for Safari and
  Home Screen web apps, not embedded WebViews.
  ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers),
  [WebKit 12945](https://webkit.org/blog/12945/meet-web-push/))
- The native alternative (APNs) needs the `aps-environment` push entitlement on
  an explicit App ID; free (personal-team) provisioning can't grant it — this is
  the well-known wall the user has hit. *Verify live before relying on the exact
  free-account mechanics; the paid-membership contrast is Apple's own framing:
  web push explicitly requires no membership.*
- A free-provisioned native app *can* show **local** notifications (e.g. while
  the app is open or a background task runs), but cannot receive **remote**
  pushes — so it can't replace a notification channel for gateway events.

### 3. Tap-through: notification → the native app

- The service worker's `notificationclick` handler can focus or open a page via
  `clients.openWindow(url)` / `client.focus()`. MDN: *"Generally this value must
  be a URL from the same origin as the calling script"* — cross-origin URLs
  resolve to `null`. A custom scheme (`hermes://…`) is not a supported
  `openWindow` target.
  ([MDN Clients.openWindow](https://developer.mozilla.org/en-US/docs/Web/API/Clients/openWindow),
  [WWDC22](https://developer.apple.com/videos/play/wwdc2022/10098/))
- So tap-through into Hermes Mobile is a **two-hop** design: notification click →
  open a same-origin PWA page → that page navigates to the app's custom URL
  scheme. The app side needs only (a) a `CFBundleURLTypes` entry in Info.plist
  (no Apple account required) and (b) an `appUrlOpen` listener via the
  `@capacitor/app` plugin. **Today the mobile app registers no URL scheme and no
  `appUrlOpen` handler** (checked `apps/mobile/client/ios/App/App/Info.plist`
  and `src/`).
  ([Capacitor App plugin](https://capacitorjs.com/docs/apis/app))
- The robust second hop should be a visible button ("Open in Hermes") rather
  than an automatic redirect: script-initiated navigation to custom schemes
  without user activation is unreliable in Safari. *Verify live on device.*
- Universal Links (https links that open the app directly) would be the clean
  tap-through, but they require the Associated Domains capability and an
  explicit App ID — paid account territory. Skip without a membership.
  (Same flag as above.)

### 4. Server side — the gateway becomes the push server

The server-side flow is small and library-supported:

1. Generate one VAPID keypair per gateway (ECDSA P-256; `openssl ecparam -name
   prime256v1 -genkey` or `py_vapid`). The public key is embedded in the PWA;
   the private key lives in `HERMES_HOME` (secret — `.env` territory).
   ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers),
   [pywebpush](https://github.com/web-push-libs/pywebpush))
2. The PWA POSTs each `PushSubscription` (endpoint + `keys.p256dh` + `keys.auth`
   — `subscription.toJSON()` gives exactly this) to a gateway endpoint, which
   stores it keyed to the user/profile.
   ([pywebpush](https://github.com/web-push-libs/pywebpush),
   [Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers))
3. On events (message received, background process finished, cron job done…),
   the gateway calls `pywebpush.webpush(subscription_info, data,
   vapid_private_key=…, vapid_claims={"sub": "mailto:…"})`, which encrypts
   (aes128gcm) and POSTs to the endpoint. Honor `TTL` and `Urgency` headers.
   ([pywebpush](https://github.com/web-push-libs/pywebpush),
   [Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers))
4. Handle responses: `201` ok, `404`/`410` subscription dead → delete it and let
   the client resubscribe next time the PWA opens; `413` payload > 4 KB; `429`
   rate limit. Apple's error table is explicit (`410 = The device token has
   expired`).
   ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers))
- pywebpush is functional but maintained by one person (PyPI "Critical Project"
  designation, per the repo's own README). Pin it with an upper bound per repo
  policy. The cryptography is just RFC 8291 + RFC 8292 — vendorable if it ever
  rots.
  ([pywebpush](https://github.com/web-push-libs/pywebpush),
  repo AGENTS.md dependency-pinning policy)

### 5. Subscription lifecycle (the honest warts)

- A subscription endpoint is a secret **capability URL** — knowledge of it is
  sufficient to push; store it per-user and never expose it.
  ([MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API))
- Push permission can be revoked at any time (user, or system for
  `userVisibleOnly` violations). The standard resilience pattern is
  `pushsubscriptionchange` handling plus re-subscribing on every PWA open — the
  subscription registrar page should silently refresh the subscription each
  launch and PUT the (possibly new) subscription to the gateway.
  ([MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API),
  [Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers))
- ntfy (shipped, at scale) pauses background notifications if the web app isn't
  opened for over a week — evidence that "open the PWA occasionally" is part of
  the real-world maintenance story, and a reason the sink PWA should refresh its
  subscription on open rather than relying on indefinite persistence.
  ([ntfy](https://docs.ntfy.sh/subscribe/web/))
- Deleting the Home Screen web app kills its subscription; the gateway learns
  via 404/410 on the next send. (Consistent with Apple's 410 semantics; exact
  edge behavior worth one live test.)

### 6. Hosting and serving

- Service workers require a **secure context** (HTTPS, or localhost for dev).
  Since Hermes gateways are typically reached over Tailscale, `tailscale serve`
  fronts the gateway with an automatically provisioned TLS certificate for the
  tailnet hostname — that satisfies the secure-context requirement with zero
  cost and no Apple account.
  ([Tailscale serve](https://tailscale.com/kb/1242/tailscale-serve))
- The PWA origin must be stable across its life (subscriptions are per-origin,
  and the service worker's scope defines what it controls) — the gateway should
  serve it from a fixed path on the gateway's canonical origin.
  ([MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API),
  [ntfy](https://docs.ntfy.sh/subscribe/web/))

### 7. Precedent

- **ntfy** ships this exact architecture — self-hosted server + web app/PWA
  that subscribes via Web Push and displays notifications; its docs document
  iOS behavior (home-screen required, works with browser closed, per-server
  origin binding, the 1-week pause heuristic). ntfy is the closest shipped
  analog to a "notification-centric PWA" and it works on iOS 16.4+ today.
  ([ntfy](https://docs.ntfy.sh/subscribe/web/))
- Apple's own canonical minimal example (BrowserPets in the WWDC22 session) is
  ~40 lines of service-worker + subscribe code — a sink PWA is strictly less
  ([WWDC22](https://developer.apple.com/videos/play/wwdc2022/10098/)).
- macOS Safari 16.1+ and desktop Chrome/Firefox receive web push without any
  install step, so the same PWA + gateway relay covers desktop notification
  surfaces too (Chrome Android even delivers with the browser closed; desktop
  browsers require the browser running — Safari macOS is the exception).
  ([ntfy table](https://docs.ntfy.sh/subscribe/web/),
  [caniuse](https://caniuse.com/push-api))

### 8. Fit with Hermes (sketch, not a design)

- The PWA is a **gateway web surface**: static manifest + page + service worker
  served by the gateway, plus one subscribe endpoint and a send-on-event relay.
  No new model tools, no toolset changes, no system-prompt surface — prompt
  caching and the narrow-waist rules are untouched. The relay is
  gateway/edge capability, matching the "capability lives at the edges" rubric.
- Config surface (if pursued): a `gateway.web_push` section (enable, VAPID key
  paths) rather than new env vars; VAPID private key is a secret → `HERMES_HOME`
  `.env` file, per the secrets-only policy.
- The Capacitor app's only change (optional, for tap-through) is a URL-scheme
  registration + `appUrlOpen` handler — additive client wiring, no core edits.

---

## Risks & open questions (verify live on device)

1. **Subscription longevity on iOS** — refresh-on-open is the safe pattern;
   measure actual expiration in a week-long test.
2. **Custom-scheme hop without user activation** — whether the PWA page can
   auto-redirect to `hermes://` in Safari/home-screen context, or needs a
   explicit button tap (design for the button; test the auto-redirect).
3. **Notification action buttons** — expected unsupported on iOS web push;
   confirm on current iOS before designing around them.
4. **Free-provisioning entitlement details** for the native alternative
   (`aps-environment` + explicit App ID) — cited from practice, re-verify against
   Apple docs when deciding between routes.
5. **pywebpush bus-factor** — single maintainer; acceptable with a version pin,
   fallback plan is ~100 lines of RFC 8030/8291/8291 code.

## Sources

- [WebKit: Web Push for Web Apps on iOS and iPadOS](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/) — 16.4 release, Home Screen requirement, manifest, gesture, Badging, manifest `id`
- [WebKit: Meet Web Push](https://webkit.org/blog/12945/meet-web-push/) — `webpushd`, userVisibleOnly contract, revocation, no silent push
- [Apple: Sending web push notifications in web apps and browsers](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers) — server requirements, VAPID/TTL/Urgency, 4 KB limit, 410 semantics, revocation rule
- [Apple/WWDC22: Meet Web Push for Safari (transcript)](https://developer.apple.com/videos/play/wwdc2022/10098/) — end-to-end flow, gesture requirement, no-Apple-account statement
- [MDN: Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API), [PushManager.subscribe()](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe), [Clients.openWindow()](https://developer.mozilla.org/en-US/docs/Web/API/Clients/openWindow) — capability URLs, same-origin window rule
- [caniuse: Push API](https://caniuse.com/push-api) — per-engine support, iOS "partial"
- [Capacitor: App plugin API](https://capacitorjs.com/docs/apis/app) — custom URL scheme + `appUrlOpen`
- [ntfy docs: Subscribe from the Web app](https://docs.ntfy.sh/subscribe/web/) — shipped precedent, browser/platform delivery table, action-button support
- [pywebpush](https://github.com/web-push-libs/pywebpush) — Python sender library, VAPID + aes128gcm, maintenance status
- [Tailscale serve](https://tailscale.com/kb/1242/tailscale-serve) — auto-provisioned HTTPS for tailnet services