# Native rewrite — architecture spec

Living document. Last updated: Phase 0 bootstrap.

## 1. Why

The React Native version of Republike has a structural quality ceiling that the manifesto's "you are the owner" positioning cannot afford: lists scroll a beat off the OS, text inputs have keyboard-coordination quirks, custom gestures stutter under JS-thread load, platform conventions don't match. The cost of polishing RN to near-native is higher than rewriting the high-traffic surfaces natively.

## 2. Scope of first delivery

Three surfaces, in priority order:

1. **Auth** — login, signup, email verification, onboarding (interests, principles, profile completion), paywall
2. **Feed** — home feed (Agora chronological + For You ranked), post detail, reactions, media (image, video, polls)
3. **Profile** — own profile, other user profile, edit profile, settings

Deferred to a later delivery: composer, search, notifications, moderation flow, content reporting, DMs, rich-text editor, tip flow, subscription management screens.

Until the deferred surfaces ship natively, **they do not exist in the native app** — this is a full rewrite, not a hybrid. RN's republike-mobile remains shipped in parallel until native reaches feature parity. See `screens.md` for the per-screen scope label.

## 3. Stack

Identical decisions on both platforms where possible; platform-native where not.

| Concern | iOS | Android |
|---|---|---|
| Language | Swift 6 | Kotlin 2.1+ |
| UI | SwiftUI | Jetpack Compose |
| Min OS | iOS 17 | API 33 (Android 13) |
| Navigation | Coordinator pattern over `NavigationStack` | Navigation Compose with type-safe routes |
| HTTP | URLSession + async/await | Ktor + OkHttp |
| Serialization | Codable | kotlinx.serialization |
| Image | Nuke (with `NukeUI.LazyImage`) | Coil 3 (`AsyncImage`) |
| Video | AVKit `VideoPlayer` | Media3 / ExoPlayer |
| Auth storage | Keychain + `Codable` wrapper | DataStore (Preferences) + EncryptedSharedPreferences |
| DI | Environment values + factory protocols | Hilt |
| State | `@Observable` + `@MainActor` | `StateFlow` + `viewModelScope` |
| Crash / errors | Sentry self-hosted | Sentry self-hosted |
| Analytics | Server-side aggregate funnel counters (no SDK) — see [analytics blog post](../blog/2026-05-24.md) | Same |
| Localization | `.xcstrings` catalogs, EN/FR/IT/ES | `strings.xml` per locale, EN/FR/IT/ES |
| Lint | SwiftLint + SwiftFormat | detekt + ktlint |
| CI | Xcode Cloud (or GH Actions w/ macOS runner) | GH Actions, Linux runner |

## 4. Module layout

Identical conceptual structure on both platforms.

```
App                       # entry point, DI graph, root coordinator
Core/
  Models                  # Codable / @Serializable structs ported from prisma schema
  Network                 # API client, request DSL, error types
  Session                 # current user, token storage, sign-in/out
  Theme                   # generated tokens, typography, dark/light
  Localization            # generated locale wrappers
  Utilities               # date, currency, format helpers
Features/
  Auth                    # login, signup, verification, paywall, onboarding
  Feed                    # feed list, post detail, reactions, media playback
  Profile                 # own, other, edit
  TabBar                  # root nav + placeholders for deferred surfaces
UI/
  Atoms                   # Button, Label, Icon, TextField, Avatar, Badge
  Molecules               # FormField, ListItemRow, CardHeader, ReactionPill, ...
  Organisms               # PostCard, ProfileHeader, NavigationBar, ComposerBar
  Templates               # ScrollableLayout, AuthLayout, SettingsLayout
```

**Rule.** No `UI/Screens/` directory holds raw colors, fonts, or padding — everything routes through `UI/Atoms` and the Theme tokens. PR reviewers reject violations.

## 5. Atomic design discipline

The order in which UI components ship is non-negotiable: atoms → molecules → organisms → templates → screens. Until all atoms a screen needs are landed, the screen is blocked.

### Atoms (locked list)

- `Button` (variants: primary, secondary, tertiary, destructive, ghost)
- `Label` (variants by typography token: title, headline, body, caption, footnote)
- `Icon` (SF Symbols on iOS, Material Symbols on Android, wrapped behind a single `Icon(name:)`)
- `TextField` (variants: text, email, password, code; with leading/trailing slots)
- `Avatar` (sizes: 24, 32, 40, 56, 80, 120)
- `Badge` (status pills, count badges)
- `Divider`
- `Spinner`

### Molecules (locked list)

- `FormField` (label + TextField + error message + helper text)
- `ListItemRow` (leading slot + title + subtitle + trailing slot + chevron)
- `CardHeader` (avatar + name + meta + actions menu)
- `ReactionPill`
- `EmptyState` (icon + title + body + primary action)
- `TabButton`
- `LoadingSkeleton` (post, profile row, comment)
- `AlertDialog`

### Organisms (locked list)

- `PostCard` (header + content + media + reactions + footer actions)
- `ProfileHeader` (avatar + name + bio + stats + follow/edit button)
- `NavigationBar` (back, title, trailing actions)
- `TabBar` (5 tabs: Feed, Search, Composer, Notifications, Profile)
- `PaywallCard`

### Templates

- `ScrollableScreen` (NavigationBar + scrollable content)
- `AuthScreen` (logo + content + footer CTAs)
- `EmptyTabScreen` (placeholder used by deferred tabs)

## 6. Networking

Existing tRPC procedures stay; the native clients are hand-written, typed wrappers over plain HTTP. We're not generating from tRPC schema — codegen for Swift / Kotlin from tRPC is too rough; ~30 procedures the mobile app actually calls is bounded enough to maintain manually.

Pattern (iOS):

```swift
public struct PostAPI {
    public func getBatch(input: GetBatchInput) async throws -> GetBatchOutput { ... }
    public func reactToPost(input: ReactInput) async throws -> Post { ... }
    // ...
}
```

Pattern (Android):

```kotlin
class PostApi(private val http: HttpClient) {
    suspend fun getBatch(input: GetBatchInput): GetBatchOutput { ... }
    suspend fun reactToPost(input: ReactInput): Post { ... }
}
```

The list of every procedure mobile uses is in [`api-surface.md`](./api-surface.md).

## 7. Phase plan (compressed view)

| Phase | Output | Both platforms (parallel, 30 % cap) |
|---|---|---|
| 0 | Spec, ADRs, tokens, screen inventory, dependency map | ~2 weeks |
| 1 | Project skeletons + CI + token codegen + mock layer | ~2 weeks |
| 2 | Network + Session | ~3 weeks |
| 3 | Tab bar + nav skeleton | ~1 week |
| 4 | Atoms, molecules, organisms | ~3 weeks |
| 5 | Auth surface | ~3 weeks |
| 6 | Feed surface | ~5 weeks |
| 7 | Profile surface | ~2 weeks |

Total ~22 sprints at 30 %. Compressible to ~10 weeks at full focus.

## 8. Open questions

Tracked as ADRs in [`adrs/`](./adrs/):

- ADR-0001 — Navigation pattern (Coordinator on iOS, type-safe routes on Android)
- ADR-0002 — Rich text editor for composer (deferred; recommended option C — plain text in v1)
- ADR-0003 — Token codegen pipeline (placeholder)
- ADR-0004 — Error / crash reporting (Sentry self-hosted) (placeholder)
