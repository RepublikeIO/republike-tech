# ADR 0011 — Push notifications

**Status:** Accepted (Phase 6, deferred behind feature flag)
**Date:** 2026-05-29

## Context

Push notifications are deferred in the first delivery — the Notifications tab is a placeholder. But the **plumbing** for receiving and routing pushes must land with the rest of Phase 5/6 because:

1. Auth flow opens permission asks at the right moments (Phase 5).
2. Deep links (ADR-0010) need to handle pushes that carry URLs (Phase 5+6).
3. The server-side push infrastructure ALREADY sends notifications to the RN app; the native apps must not break those subscriptions.

## Decision

| | iOS | Android |
|---|---|---|
| Transport | APNs (Apple Push Notification service) | FCM (Firebase Cloud Messaging) |
| SDK | Native `UNUserNotificationCenter` — no third-party | `firebase-messaging-ktx` (Firebase Android SDK) |
| Token registration | `UIApplication.registerForRemoteNotifications` | `FirebaseMessaging.getInstance().token` |
| Token send-up | Existing webapp endpoint `user.registerPushToken` (TBD if exists; ADR creates it) | same |
| Tap → action | Deep-link payload routed through coordinator/dispatcher | same |

### Why Firebase on Android and not Native

Android does not have a first-party push API — all production apps use FCM. There's no escape. The FCM SDK does not phone home with PII as long as `firebase-analytics` is NOT added; we add only `firebase-messaging-ktx`. The Firebase project carries no analytics integration.

### iOS — UNUserNotificationCenter only

We do not adopt the Firebase iOS SDK. Direct APNs registration:

```swift
UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .badge, .sound])
UIApplication.shared.registerForRemoteNotifications()
```

Apple ID + APNs cert / key live in the existing `republike-webapp` push-sending pipeline.

### Payload shape

All payloads carry a `deepLink` field that's a fully-qualified `https://www.republike.io/...` URL. Native handlers route via the existing deep-link handler (ADR-0010).

```json
{
  "aps": { "alert": { "title": "...", "body": "..." }, "sound": "default" },
  "deepLink": "https://www.republike.io/p/cm4..."
}
```

Android:

```json
{
  "notification": { "title": "...", "body": "..." },
  "data": { "deepLink": "https://www.republike.io/p/cm4..." }
}
```

### Permission asks

- **Never** ask for notification permission on first launch — the request fires only after a user signs up or signs in successfully.
- iOS first-runtime ask: after the Profile completion screen.
- Android: API 33+ requires runtime permission for `POST_NOTIFICATIONS` — ask at the same moment.

Permission state is exposed via a `NotificationPermission` service in `Core/Notifications/` so screens can show "enable notifications" cards later without asking again.

### Server token lifecycle

| Event | Action |
|---|---|
| Sign in | Register the platform token via `user.registerPushToken` |
| Sign out | Deregister the token (`user.deregisterPushToken`) — server stops sending |
| Token rotates (FCM / APNs reissues) | Re-register on next app launch |
| User revokes notifications in OS settings | App detects on resume, deregisters server-side |

## Deferring the Notifications tab

The first delivery ships:

- ✅ Permission asks (Phase 5)
- ✅ Token registration on sign-in (Phase 5)
- ✅ Deep-link routing from a tapped notification (Phase 5)
- ⛔ In-app Notifications tab — placeholder template "Notifications coming soon" until the deferred surface ships

This means a user gets push notifications and they open the right post / profile. They just don't have an in-app history view yet.

## Consequences

- Firebase is added to the Android project for FCM only. `google-services.json` lives in `republike-android/app/`. CI must inject it from a GH secret on PR builds (or use a stub for PRs that don't need real FCM).
- APNs auth key + Team ID live in the existing webapp push setup; no new secrets on the native side.
- The push-sending logic on the webapp must adopt the `deepLink` field. Backfill issue tracked outside the rewrite.

## Maps to RN source

- `republike-mobile/src/Services/notifications/*` — RN push handlers
- `expo-notifications` in `package.json` — replaced

## Alternatives considered

- **OneSignal / Notifee.** Add a third-party push aggregator. Rejected — extra vendor, extra data path.
- **WebSockets-only (no push).** Considered for the manifesto angle. Rejected — users expect push and the absence is a real product gap, not a feature.
