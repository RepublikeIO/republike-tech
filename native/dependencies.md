# Dependency replacement matrix

Every `republike-mobile` runtime dependency mapped to its native iOS + Android equivalent. Build / dev dependencies are excluded.

## Network + state

| RN | iOS | Android | Notes |
|---|---|---|---|
| `@trpc/client` + `@trpc/react-query` | hand-written Swift API client over URLSession | hand-written Kotlin API client over Ktor | tRPC codegen for native is too rough; ~30 procedures are bounded enough to hand-write. See [`api-surface.md`](./api-surface.md). |
| `@tanstack/react-query` | per-screen `@Observable` viewmodel with cache | `StateFlow` + `viewModelScope` + Coil/storage cache | No third-party query-cache framework needed |
| `axios` | n/a | n/a | We don't need a second HTTP lib |
| `superjson` | custom JSON encoders/decoders for non-JSON types | custom kotlinx serializers | Only used for `Date`, `Decimal`, `BigInt` |
| `zod` | n/a (server keeps validation) | n/a | Client-side schema validation goes away |
| `class-variance-authority`, `clsx`, `tailwind-merge` | n/a | n/a | Replaced by design-token system |

## UI + interaction

| RN | iOS | Android | Notes |
|---|---|---|---|
| `@gorhom/bottom-sheet` | SwiftUI `.sheet` / custom `BottomSheet` view modifier | Material 3 `ModalBottomSheet` | first-class on both |
| `@react-navigation/*` | NavigationStack + Coordinator pattern (see ADR-0001) | Navigation Compose with type-safe routes | |
| `@rn-primitives/*` (radix-style) | SwiftUI built-ins | Compose Material 3 | |
| `react-native-collapsible-tab-view` | custom (SwiftUI `TabView` + sticky header) | Compose `LazyColumn` with `Modifier.nestedScroll` | |
| `react-native-gesture-handler` | SwiftUI native gestures | Compose Modifier gestures | |
| `react-native-reanimated` | SwiftUI animation + `withAnimation` | Compose animation primitives | |
| `react-native-pager-view` | `TabView(.page)` style | `HorizontalPager` (Accompanist or Compose 1.7+) | |
| `react-native-walkthrough-tooltip` | custom popover via `.popover()` | custom Compose tooltip | |
| `react-native-tab-view` | TabView + custom indicator | Compose Material 3 `TabRow` | |
| `react-native-toast-notifications` | custom Snackbar component | Material 3 Snackbar | |
| `react-native-confirmation-code-field` | custom (focused state across digit fields) | custom Compose component | |
| `react-native-element-dropdown` | SwiftUI Picker / Menu | Compose `ExposedDropdownMenuBox` | |
| `react-native-modal` | `.sheet` / `.fullScreenCover` | `Dialog` / `ModalBottomSheet` | |
| `react-native-shadow-2` | SwiftUI `.shadow` | Compose `Modifier.shadow` | |
| `react-native-svg` | SF Symbols + `Image(systemName:)` for icons; SVG only via SVGKit if vendor assets | Material Symbols + Compose `Icon`; SVGs via `coil-svg` | We prefer system icons across the board |
| `nativewind`, `tailwindcss-animate` | n/a | n/a | Replaced by design tokens |
| `react-instantsearch*` + `algoliasearch` | Algolia Swift Client | Algolia Kotlin Client | Search is deferred, but the client choice is locked |

## Media

| RN | iOS | Android | Notes |
|---|---|---|---|
| `expo-image` | Nuke + `LazyImage` | Coil 3 `AsyncImage` | both first-class, with disk cache |
| `expo-image-picker` | PhotosUI `PHPickerViewController` | Photo Picker API (Android 13+) | both deferred until composer |
| `react-native-image-viewing` | SwiftUI custom zoom + pan view | Compose `Modifier.transformable` | |
| `react-native-video` | AVKit `VideoPlayer` | Media3 / ExoPlayer | |
| `expo-video` | AVKit | Media3 | |
| `expo-blur` | `.background(.ultraThinMaterial)` | `Modifier.blur` (Compose 1.5+) | |
| `expo-linear-gradient` | `LinearGradient` | `Brush.linearGradient` | |
| `expo-haptics` | UIImpactFeedbackGenerator | `HapticFeedbackType` via Compose `LocalHapticFeedback` | |
| `@10play/tentap-editor` (rich text + media) | **ADR-0002** — recommended option C (plain text v1) | **ADR-0002** — same | Biggest dependency risk. Composer is deferred — first delivery doesn't need this. |
| `react-native-pell-rich-editor`, `react-native-render-html`, `@tiptap/*` | n/a (composer deferred; HTML rendered via `AttributedString`) | n/a (composer deferred; HTML rendered via Compose `AnnotatedString`) | |

## Auth + identity

| RN | iOS | Android | Notes |
|---|---|---|---|
| `@react-native-async-storage/async-storage` | UserDefaults + Codable | DataStore (Preferences) | for non-secret prefs |
| Keychain (custom usage) | Keychain wrapper | EncryptedSharedPreferences | for tokens, refresh tokens |
| `expo-contacts` | Contacts framework | ContactsContract | deferred (used only on invite flow) |
| `expo-clipboard` | UIPasteboard | ClipboardManager via `LocalClipboardManager` | |

## Notifications + deep links

| RN | iOS | Android | Notes |
|---|---|---|---|
| `expo-notifications` | UNUserNotificationCenter + APNs | Firebase Messaging | server registration unchanged |
| `expo-linking` | Universal Links | App Links | DNS already configured (`applinks:` association files) |

## Payments

| RN | iOS | Android | Notes |
|---|---|---|---|
| `react-native-iap` (subscriptions) | StoreKit 2 | Google Play Billing 6.x | platform-native, no shared abstraction |
| Stripe (web-paywall path) | Stripe iOS SDK | Stripe Android SDK | only needed for non-IAP flows |

## I18n

| RN | iOS | Android | Notes |
|---|---|---|---|
| `i18next` + `react-i18next` + `i18next-resources-for-ts` | String Catalogs (`.xcstrings`) | `strings.xml` per locale | Single CSV upstream → exports to both (script lands Phase 1) |

## Telemetry

| RN | iOS | Android | Notes |
|---|---|---|---|
| _none_ | Sentry self-hosted (crash + error) | Sentry self-hosted | aggregate-funnel events are server-side — see [analytics blog post](../blog/2026-05-24.md) |

## Dev tooling (not runtime, but listed for completeness)

| RN | iOS | Android |
|---|---|---|
| ESLint + Prettier | SwiftLint + SwiftFormat | detekt + ktlint |
| Jest | Swift Testing (Swift 6 default) + XCTest where needed | JUnit 5 + Kotest |
| TypeScript | Swift (native) | Kotlin (native) |
| Patch-package | n/a | n/a |
| Babel | n/a | n/a |

## Categorical drops

These RN deps disappear entirely with no native equivalent needed:

- All `@tiptap/*`, `tlds`, `linkify-it` — composer is deferred; when it returns, native components serve the same role
- `lodash` — Swift / Kotlin standard libraries cover the same surface
- `dayjs` — replaced by `Foundation.Date` formatters / `kotlinx-datetime`
- `react-hook-form`, `@hookform/resolvers` — replaced by `@FocusState` + custom validator structs on iOS; `rememberSaveable` + state hoisting on Android
- `react-native-uuid`, `cuid`, `bplist-creator`, `add`, `class`, `immer`, `axios`, `superjson`, `zod`, `zustand`, `react`, `react-dom`, `react-native`, all `react-native-*` — RN runtime drops entirely
