# ADR 0010 — Deep linking

**Status:** Accepted (Phase 5)
**Date:** 2026-05-29

## Context

Notifications, shared links, and password-reset emails open the app on a specific screen. The current RN setup uses `expo-linking` for both Universal Links (iOS) and App Links (Android). The native rewrite uses each platform's first-party API.

## Decision

| | iOS | Android |
|---|---|---|
| Mechanism | Universal Links via `applinks:` association file | App Links via Digital Asset Links + intent filters |
| Domain | `republike.io` (apex + `www.`, `staging.`, `dev.`) | same |
| Custom scheme fallback | `republike://` for in-app composition only — not for public links | `republike://` ditto |
| Handler | Coordinator's `handle(deepLink:)` (ADR-0001) | NavigationDispatcher's `handle(deepLink:)` |

### URL pattern

Single namespace, mobile and web share it 1:1:

```
https://www.republike.io/p/<postId>                 → PostDetail
https://www.republike.io/u/<username>               → Profile
https://www.republike.io/p/<postId>?moderate=true   → Moderation (if user is moderator)
https://www.republike.io/auth/verify?token=...      → Email verification
https://www.republike.io/auth/reset?token=...       → Password reset
https://www.republike.io/api/open-app/?type=...     → Generic open-app router
```

The native handler parses the URL and translates to a coordinator/dispatcher intent. Web fallback: if the native app isn't installed, the URL renders the web equivalent in the user's browser.

### Association files

| | iOS | Android |
|---|---|---|
| Path | `https://www.republike.io/.well-known/apple-app-site-association` | `https://www.republike.io/.well-known/assetlinks.json` |
| Content (iOS) | `applinks` with `appID = TEAMID.app.republike` and `paths` matching the patterns above | n/a |
| Content (Android) | n/a | `relation: ["delegate_permission/common.handle_all_urls"]`, `target.package_name: io.republike.app`, sha256 fingerprint of the release signing cert |
| Hosting | Served by Caddy at the apex domain | same |

Both files land in `republike-webapp/public/.well-known/` so they ship with the webapp. The webapp's existing routing must NOT 404 on `.well-known/*` paths.

### Verifying

- iOS: `aasa-validator` CLI confirms the file parses.
- Android: `adb shell pm get-app-links io.republike.app` confirms the system verified the App Links.
- Both: open one of the URLs in a browser on the device, confirm the app launches into the right screen.

### Custom scheme rules

`republike://` is reserved for:

- OAuth callbacks if any (none today)
- IAP receipt resume after platform interrupts the flow

It is NOT used for shared content URLs. Anything users see in an email, push, or message is `https://`.

### Open-app router on the webapp

The existing `/api/open-app/?type=moderation&id=…` endpoint stays. The native handler decodes the `type` + `id` and routes accordingly. Adding a new `type` requires:

1. Web route to handle the non-app fallback
2. Native handler to add the type case
3. Per-platform issue in `republike-tech/native/i18n/` for any new strings

## Consequences

- Domain verification depends on the webapp's `.well-known/*` files being correct — they're now critical infrastructure.
- The release signing cert fingerprint must be added to `assetlinks.json` BEFORE the first Android release; otherwise App Links never activate. This is a Phase 5 prerequisite.
- The native handler is one well-tested function per platform; deep link bugs become regressions in that function rather than scattered across feature code.

## Maps to RN source

- `republike-mobile/app.config.js:60-78` — current associated domains config
- `republike-mobile/src/Navigation/AppNavigation.tsx` — current deep-link reception (RN uses `expo-linking` config)
- `republike-webapp/public/.well-known/*` — files served today

## Alternatives considered

- **Branch.io / AppsFlyer OneLink.** Adds deferred deep linking (install-from-link). Rejected — explicit per [`../blog/2026-05-24.md`](../../blog/2026-05-24.md): no third-party attribution SDKs.
- **Custom URL scheme as primary.** Rejected — modern stores penalize custom-scheme-only flows and they don't work cleanly from emails.
