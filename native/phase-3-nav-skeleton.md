# Phase 3 — Tab bar + navigation skeleton

Lands before Phase 4 (design system) because the design system needs a shell to be mounted in. The navigation skeleton has no atom dependencies — it uses bare native components and inline strings (the only phase where this is allowed; cleaned up in Phase 4 + Phase 5).

## Scope

| | iOS | Android |
|---|---|---|
| Root | `App.swift` mounting a `RootCoordinator` | `MainActivity.kt` mounting a `Navigation` graph |
| Auth state branch | Logged-out → `AuthCoordinator` route; logged-in → `MainCoordinator` route | Routes: `Auth.Landing` vs `Main.TabBar` |
| Tab bar | 5 tabs (Feed, Search, Composer, Notifications, Profile) | same |
| Tab implementations | Empty placeholders showing tab name + "coming soon" for Search / Composer / Notifications; Feed + Profile receive a "Hello world" placeholder | same |
| Deep link plumbing | `coordinator.handle(deepLink:)` stub that logs to console | `dispatcher.handle(deepLink:)` stub that logs |
| USER_INACTIVE → Paywall | API client interceptor calls `coordinator.goToPaywall()` (Paywall is a placeholder route in Phase 3) | same |

## Tab bar layout

5 cells, fixed left-to-right order:

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│                  active screen content                   │
│                                                          │
├──────────────────────────────────────────────────────────┤
│  Feed    Search    [+]Compose    Notif    Profile        │
└──────────────────────────────────────────────────────────┘
```

| Tab | Icon (system) | Visible in Phase 3 |
|---|---|---|
| Feed | `house` / `home` | placeholder "Feed (Phase 6)" |
| Search | `magnifyingglass` / `search` | placeholder "Search coming soon" |
| Composer | `plus.circle.fill` / `add_circle` — center cell, larger | placeholder "Composer coming soon" |
| Notifications | `bell` / `notifications` | placeholder "Notifications coming soon" |
| Profile | `person.circle` / `person` | placeholder "Profile (Phase 7)" |

The center Composer tab is visually larger per the RN app's existing pattern (`republike-mobile/src/Navigation/MainTab.tsx`).

## Coordinator / Dispatcher contracts

Per ADR-0001.

iOS `AppCoordinator` (intent-based):

```swift
protocol AppCoordinator: AnyObject {
    func openPost(id: PostId)
    func openProfile(username: String)
    func openFeed()
    func openSettings()
    func goToPaywall()
    func goToLanding()
    func signOut()
    func handle(deepLink: URL)
}

@Observable final class DefaultAppCoordinator: AppCoordinator {
    var route: AppRoute = .splash
    enum AppRoute { case splash, auth, main, paywall }
}
```

Android `NavigationDispatcher` (typed routes):

```kotlin
sealed interface Route {
    @Serializable data object Splash : Route
    @Serializable data object Landing : Route
    @Serializable data object Main : Route
    @Serializable data class PostDetail(val id: String) : Route
    @Serializable data class Profile(val username: String) : Route
    @Serializable data object Paywall : Route
}

interface NavigationDispatcher {
    fun openPost(id: PostId)
    fun openProfile(username: String)
    fun openFeed()
    fun openSettings()
    fun goToPaywall()
    fun goToLanding()
    fun signOut()
    fun handle(deepLink: Uri)
}
```

In Phase 3 most intents log + navigate to a placeholder. Phase 5+ replaces placeholders with real screens.

## Issues (per platform — opened after Phase 2 closes)

1. **scaffold: tab bar shell with 5 placeholder tabs** — minimal navigation host, 5 tabs, no logic; the entry point fulfilled.
2. **scaffold: coordinator / dispatcher** — protocol + default impl; intent stubs log to console; unit tests assert intent → route mapping.
3. **scaffold: deep link parser** — `parseDeepLink(URL) → CoordinatorIntent` pure function, exhaustive tests over every URL pattern from ADR-0010.
4. **scaffold: USER_INACTIVE → Paywall route** — wire the API client interceptor (from Phase 2 #1) to call `coordinator.goToPaywall()`. Manual test: pass an expired session, confirm the paywall placeholder appears.
5. **scaffold: SignOut + Landing route** — calling `signOut()` clears Keychain / EncryptedSharedPreferences and routes to landing placeholder.
6. **Phase 3 tracking issue** — closes when 1-5 land.

## Acceptance for Phase 3

- App cold-launches into either the auth landing placeholder (signed out) or the tab bar (signed in).
- Tapping every tab routes to its placeholder.
- Pasting a `republike.io/p/<id>` URL into Safari/Chrome opens the app (or attempts to) and the coordinator/dispatcher logs the parsed intent.
- Force a USER_INACTIVE API response → app routes to the paywall placeholder.
- Force a sign-out → Keychain / EncryptedSharedPreferences cleared, app routes to landing placeholder.
- No design tokens needed (placeholders use raw system styles); Phase 4 replaces every visual element.

## Maps to RN source

- `republike-mobile/src/Navigation/AppNavigation.tsx` — root navigator
- `republike-mobile/src/Navigation/MainTab.tsx` — tab bar + 5 tabs
- `republike-mobile/src/Services/ApiProvider.tsx:25-100` — USER_INACTIVE handling
