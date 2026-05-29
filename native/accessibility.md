# Accessibility (a11y)

Minimum bar for every shippable screen. Violations are reviewable PR comments.

## What we commit to

1. **Every interactive element has a label.** No mystery icons, no unlabeled buttons.
2. **Hit targets are ≥44pt × 44pt** (iOS HIG) / **≥48dp × 48dp** (Android Material).
3. **Text scales with the system's Dynamic Type** (iOS) / **font scale** (Android). No locked-size body text.
4. **Color is not the only signal** — reactions, alerts, validation states all carry a label or icon in addition to color.
5. **Focus order makes sense** — Tab key navigation on iPad / external keyboard, Switch Access on Android.
6. **Status changes announce** — toasts, errors, and "X saved" confirmations post to the screen reader.

## What we DON'T commit to in v1

- WCAG AAA contrast (we hit AA across all locked tokens, see below).
- High-contrast mode tuning beyond what comes free from the system.
- Voice control profile customisation.

These are roadmap items, not v1 blockers.

## Per-platform mechanisms

### iOS — VoiceOver

| Component | Required attribute |
|---|---|
| Button | `.accessibilityLabel("Sign in")` or label text already present |
| Icon-only button | `.accessibilityLabel("Close")` mandatory |
| Image | `.accessibilityLabel("..."` or `.accessibilityHidden(true)` for decoration |
| Decorative content | `.accessibilityHidden(true)` to skip |
| Custom view | conforms to `AccessibilityRotor` if it represents a list |
| Toggles | `.accessibilityValue("On" / "Off")` |
| Loading | `.accessibilityLabel("Loading")` on spinners |

Dynamic Type: every `Text` uses a token from `Tokens.Typography.fontSize.*` — those map to `@ScaledMetric`, NOT raw `CGFloat`. Generated tokens already do this.

### Android — TalkBack

| Component | Required attribute |
|---|---|
| Button | `Modifier.semantics { contentDescription = "Sign in" }` or text content |
| Icon-only button | `IconButton(onClick = …) { Icon(..., contentDescription = "Close") }` |
| Image | `contentDescription` set or `null` (decorative) |
| Toggles | `Modifier.toggleable(role = Role.Switch)` |
| Loading | `Modifier.semantics { liveRegion = LiveRegionMode.Polite }` |

Font scale: every `Text` uses `Tokens.Typography.fontSize.body.sp` etc — these are scalable units by default in Compose.

## Color contrast

Token color choices verified at design time (see `tokens.json` review). The lint catch: never compose two colors against each other for foreground/background outside the documented pairs:

| Background | Foreground (allowed) |
|---|---|
| `surface.card` (#FFF) | `text.primary` (#131721) ≥ AA |
| `surface.background` (#F8F8FF) | `text.primary` ≥ AA |
| `primary` (#7B61FF) | `text.onPrimary` (#FFF) ≥ AA Large; NOT for small body text |
| `feedback.error` (#DC2626) | `text.onPrimary` (#FFF) ≥ AA |

Buttons that use `primary` as background must use the `Label.headline` or larger size for the on-primary text. Atom's `Button` primary variant enforces this.

## Tests

### iOS

- `XCUITest` checks (Phase 5+): every screen passes VoiceOver "swipe-through" smoke (all elements have labels, no decorative-only elements break the focus order).
- Snapshot tests render at largest Dynamic Type size — catches truncation.

### Android

- `paparazzi` snapshot tests render at `fontScale = 1.5` — catches truncation.
- `compose-rules` lint includes `compose-accessibility-rules` checks.

## Reviewer checks

Reviewer sub-agent confirms on UI PRs:

```
- [ ] Every Button / IconButton has an accessibility label (or visible text)
- [ ] No hardcoded font sizes — typography tokens used
- [ ] Hit targets ≥ 44pt / 48dp
- [ ] At least one snapshot rendered at the largest font scale
- [ ] No color-only status (icons + label accompany color signals)
```

A failing check → request changes.
