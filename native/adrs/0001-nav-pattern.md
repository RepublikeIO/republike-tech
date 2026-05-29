# ADR 0001 — Navigation pattern

**Status:** Accepted (Phase 0)
**Date:** 2026-05-29

## Context

Both native apps need a navigation layer that:

1. Survives auth state changes (logged-out routes vs logged-in tab bar).
2. Handles deep links (`republike://`, Universal Links, App Links) into specific posts / profiles.
3. Can intercept `USER_INACTIVE` API errors and route the user to the paywall from anywhere.
4. Is testable — viewmodels and screens can be unit-tested with a stubbed navigator.

## Decision

### iOS — Coordinator pattern over `NavigationStack`

A root `AppCoordinator` owns the navigation state and exposes intent-based methods (`coordinator.openPost(id:)`, `coordinator.goToPaywall()`). Each major surface (auth, feed, profile) has its own subcoordinator. SwiftUI screens receive their coordinator via `@Environment(\.coordinator)`.

Why coordinator and not raw SwiftUI navigation:

- Screens are not coupled to navigation destinations — they call `coordinator.openPost(...)` instead of pushing a destination directly. The screen knows nothing about what comes next.
- Unit tests inject a `MockCoordinator` and assert intent calls.
- Deep-link routing has one entry point (`coordinator.handle(deepLink:)`) rather than scattered destination matching.
- `USER_INACTIVE` interceptor in the API client calls `rootCoordinator.goToPaywall()` from any context.

Sketch:

```swift
protocol AppCoordinator: AnyObject {
    func openPost(id: PostId)
    func openProfile(username: String)
    func goToPaywall()
    func signOut()
    func handle(deepLink: URL)
}

@Observable final class DefaultAppCoordinator: AppCoordinator { ... }
```

### Android — Navigation Compose with type-safe routes

Compose Navigation 2.8+ supports `@Serializable` route classes. Routes are kotlinx-serializable data classes; navigation is `navController.navigate(PostDetailRoute(id))`.

Why type-safe routes and not a manual coordinator:

- Compose's navigation model is already declarative and graph-based; adding a coordinator layer fights the framework.
- `@Serializable` routes give compile-time safety equivalent to what the iOS coordinator provides via methods.
- Testability is preserved by injecting a `NavigationDispatcher` interface (thin wrapper over `NavController`) — screens call `dispatcher.openPost(id)`, tests inject a fake.

Sketch:

```kotlin
sealed interface Route {
    @Serializable data object Feed : Route
    @Serializable data class PostDetail(val id: String) : Route
    @Serializable data class Profile(val username: String) : Route
    @Serializable data object Paywall : Route
}

interface NavigationDispatcher {
    fun openPost(id: PostId)
    fun openProfile(username: String)
    fun goToPaywall()
    fun signOut()
    fun handle(deepLink: Uri)
}
```

## Consequences

- Screens on both platforms never see a `NavigationController` / `NavController` directly.
- Both platforms have a single "USER_INACTIVE → paywall" interceptor location.
- Adding a new destination requires touching: the coordinator/dispatcher interface, its default impl, the route enum (Android) or push call (iOS), and the screen factory. Acceptable friction for the safety.

## Alternatives considered

- **Raw SwiftUI `NavigationStack` + `@Binding` paths.** Rejected because deep linking and out-of-screen navigation (paywall from API interceptor) require global state — the coordinator wraps that cleanly.
- **Single Compose-style nav on iOS via a sealed enum.** Considered but `NavigationStack` works fine when fed by a coordinator; the enum-only approach would still need a coordinator-like indirection for testability.
- **Coordinator on Android too.** Considered for symmetry. Rejected — Compose Navigation already provides type-safe destinations; a coordinator on top is duplication.

## Maps to RN source

- `republike-mobile/src/Navigation/AppNavigation.tsx` — current RN root navigator
- `republike-mobile/src/Navigation/MainTab.tsx` — bottom tab bar definition
- `republike-mobile/src/Services/ApiProvider.tsx:25-100` — current `USER_INACTIVE` interceptor that routes to SubscriptionEndedScreen
