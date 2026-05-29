# Phase 4 — Design system

The atomic-design layer. Lands BEFORE any feature screen. Every component referenced here is shipped on both platforms before Phase 5 starts.

## Atoms

Lowest-level building blocks. Each atom takes its style only from `Tokens.*` and exposes a small, prop-driven surface.

### `Button`

| Variant | Use |
|---|---|
| `primary` | The single most important CTA on a screen |
| `secondary` | Alternative actions next to primary |
| `tertiary` | Subtle text actions ("Forgot password?") |
| `destructive` | Delete, sign out, leave |
| `ghost` | Toolbar / nav-bar buttons with no fill |

| Size | Use |
|---|---|
| `small` | Inline / list-row trailing actions |
| `medium` | Default |
| `large` | Auth screens / paywall primary CTAs |

State: `disabled`, `loading` (replaces label with a spinner).

iOS API sketch:

```swift
public struct RButton: View {
    public init(_ title: String, variant: Variant = .primary, size: Size = .medium, isLoading: Bool = false, action: @escaping () -> Void)
    public enum Variant { case primary, secondary, tertiary, destructive, ghost }
    public enum Size { case small, medium, large }
}
```

Android API sketch:

```kotlin
@Composable
fun RButton(
    label: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    variant: RButtonVariant = RButtonVariant.Primary,
    size: RButtonSize = RButtonSize.Medium,
    isLoading: Boolean = false,
    enabled: Boolean = true,
)
```

### `Label`

Wraps `Text` and forces typography choices to come from tokens.

| Style | Token map |
|---|---|
| `display` | `fontSize.display`, `fontWeight.bold` |
| `title` | `fontSize.title`, `fontWeight.bold` |
| `titleL` | `fontSize.titleL`, `fontWeight.bold` |
| `headline` | `fontSize.headline`, `fontWeight.semibold` |
| `body` | `fontSize.body`, `fontWeight.regular` |
| `bodyL` | `fontSize.bodyL`, `fontWeight.regular` |
| `caption` | `fontSize.caption`, `fontWeight.regular` |
| `footnote` | `fontSize.footnote`, `fontWeight.regular` |

### `Icon`

Single entry point for system icons (SF Symbols / Material Symbols). Brand icons get their own `RBrandIcon` atom.

```swift
RIcon("arrow.left", color: .textPrimary, size: .medium)
```

```kotlin
RIcon(name = "arrow.left", color = TextPrimary, size = RIconSize.Medium)
```

### `TextField`

| Variant | Use |
|---|---|
| `text` | Plain text |
| `email` | Email keyboard, autocorrect off, autocapitalize off |
| `password` | Secure entry, show / hide toggle |
| `code` | One-time code, numeric, no autofill on iOS (uses iOS one-time-code suggestion) |
| `multiline` | Bio fields, post content (deferred-composer placeholder) |

Slots: `leadingIcon`, `trailingIcon`, `helperText`, `errorText`.

### `Avatar`

| Size | px |
|---|---|
| `xs` | 24 |
| `sm` | 32 |
| `md` | 40 |
| `lg` | 56 |
| `xl` | 80 |
| `xxl` | 120 |

Falls back to initials on a tinted background when no image is provided.

### `Badge`

| Variant | Use |
|---|---|
| `count` | Notification count chips |
| `status` | "Founder" / "Consul" pill |
| `dot` | Tiny presence dot |

### `Divider`

Horizontal or vertical, default thickness 1px (1pt on iOS, 1dp on Android — both render as a hairline).

### `Spinner`

Default `Tokens.Color.primary`, size variants matching `Avatar`.

## Molecules

Compositions of 1-2 atoms with bound behavior.

### `FormField`

```
┌──────────────────────────────┐
│ Label (Tokens.Typography...) │
│ ┌──────────────────────────┐ │
│ │   TextField              │ │
│ └──────────────────────────┘ │
│ helperText or errorText      │
└──────────────────────────────┘
```

Props: `label`, `textField`, `helperText`, `errorText`, `isRequired`.

### `ListItemRow`

```
┌───────────────────────────────────────┐
│ [leadingSlot] Title          [trailingSlot] >
│                subtitle                 │
└───────────────────────────────────────┘
```

Used by: Settings, Profile menu, Topics list, follower list. Tap routes via slot or whole-row callback.

### `CardHeader`

Avatar + name + meta + actions menu. Used on every PostCard.

### `ReactionPill`

Small pill showing reaction count + icon. Tinted by reaction type (uses the per-reaction color tokens, not the brand primary).

### `EmptyState`

```
┌──────────────────────────┐
│       [icon]             │
│       Title              │
│       Body               │
│       [Primary action]   │
└──────────────────────────┘
```

Standard component for any empty list — feed empty, profile empty, search no-results.

### `TabButton`

For the bottom tab bar's tab cells. Selected / unselected states.

### `LoadingSkeleton`

| Variant |
|---|
| `postCard` |
| `profileRow` |
| `comment` |

Shimmer animation runs only when `isLoading == true`.

### `AlertDialog`

Title + body + primary action + optional secondary action. Used sparingly.

## Organisms

Compositions that represent a meaningful UI section.

### `PostCard`

```
┌────────────────────────────────────────┐
│ CardHeader (avatar + name + meta + …)  │
│                                        │
│ post content (Label.body, multiline)   │
│                                        │
│ optional media carousel                │
│                                        │
│ optional poll display                  │
│                                        │
│ Divider                                │
│                                        │
│ [reaction] [tip] [comment] [repost]    │
└────────────────────────────────────────┘
```

Tap → PostDetail. Long-press on reaction → reaction-stats sheet.

### `ProfileHeader`

Avatar (xl) + name (Label.title) + bio (Label.body) + stats row + primary action (Follow / Edit). Used on both own + other profile screens.

### `NavigationBar`

| Slot | |
|---|---|
| `leading` | usually Back icon |
| `title` | Label.headline, optional |
| `trailing` | up to 3 actions (icons) |

Adheres to native conventions: large title on top-of-stack screens (iOS), Material top app bar (Android).

### `TabBar` (bottom)

5 cells per `screens.md` plan: Feed / Search / Composer (placeholder) / Notifications (placeholder) / Profile.

### `PaywallCard`

Per-plan card showing price, perks, CTA. Used on PlanScreen.

## Templates

### `ScrollableScreen`

```
NavigationBar (sticky)
  ↓
ScrollView
  content
```

### `AuthScreen`

```
Logo
  ↓
ScrollView (only if content > screen)
  ↓
Sticky bottom footer
  CTAs
```

### `EmptyTabScreen`

Used by the three placeholder tabs (Search, Composer, Notifications) until deferred surfaces ship. Single `EmptyState` molecule.

## Dependencies between atoms / molecules / organisms

```
PostCard         depends on  CardHeader, Avatar, Label, Divider, ReactionPill, Icon
CardHeader       depends on  Avatar, Label, Icon
ReactionPill     depends on  Icon, Label
ProfileHeader    depends on  Avatar, Label, Button
TabBar           depends on  TabButton
TabButton        depends on  Icon, Label
NavigationBar    depends on  Icon, Label
PaywallCard      depends on  Label, Button, Divider
FormField        depends on  Label, TextField
ListItemRow      depends on  Avatar, Label, Icon
EmptyState       depends on  Icon, Label, Button
LoadingSkeleton  depends on  (no atoms — animation primitives directly)
AlertDialog      depends on  Label, Button
```

Implementation order in Phase 4:

1. Atoms first (parallel): `Button`, `Label`, `Icon`, `TextField`, `Avatar`, `Badge`, `Divider`, `Spinner`
2. Molecules next (parallel): `FormField`, `ListItemRow`, `CardHeader`, `ReactionPill`, `EmptyState`, `TabButton`, `LoadingSkeleton`, `AlertDialog`
3. Organisms last (parallel): `PostCard`, `ProfileHeader`, `NavigationBar`, `TabBar`, `PaywallCard`

Each component is one issue per platform → ~22 issues per platform → 44 total. Phase 4 is the biggest batch.

## Documentation per component

Every atom / molecule / organism PR includes:

1. Source file with doc comments
2. SwiftUI Preview (iOS) / Compose `@Preview` (Android) showing every variant
3. Snapshot test (one per variant)
4. Entry in `republike-tech/native/design-system.md` (Phase 4.5 — generated from previews)
