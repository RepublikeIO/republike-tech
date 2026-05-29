# ADR 0003 — Design-token codegen pipeline

**Status:** Accepted (Phase 1)
**Date:** 2026-05-29

## Context

[`tokens.json`](../tokens.json) is the single source of truth for colors, typography, spacing, radii, elevations, and durations across both platforms. Tokens must land in `Tokens.swift` (iOS) and `Tokens.kt` (Android) byte-identically to whatever `tokens.json` declares — otherwise the platforms drift.

## Decision

A single Node script — `scripts/generate-tokens.mjs` — lives in each platform repo, reads `tokens.json` (either from a local `republike-tech` checkout or fetched from the master branch by URL), and emits one generated source file per platform.

### Pipeline shape

```
                  ┌────────────────────────────────────┐
                  │  republike-tech/native/tokens.json │
                  └──────────────────┬─────────────────┘
                                     │
                ┌────────────────────┼────────────────────┐
                ▼                                         ▼
   republike-ios/scripts/generate-tokens.mjs   republike-android/scripts/generate-tokens.mjs
                │                                         │
                ▼                                         ▼
   Sources/Core/Theme/Generated/                core/theme/.../generated/
       Tokens.swift                                 Tokens.kt
```

### File header

Every generated file starts with:

```
// DO NOT EDIT — generated from native/tokens.json
// Source commit: <republike-tech commit SHA at generation time>
// Run `npm run tokens` (or platform equivalent) to regenerate.
```

### iOS output

`Sources/Core/Theme/Generated/Tokens.swift` exports:

```swift
import SwiftUI

public enum Tokens {
    public enum Color {
        public static let primary = SwiftUI.Color(hex: 0x7B61FF)
        // ...
    }
    public enum Typography {
        public enum FontSize {
            public static let body: CGFloat = 15
            // ...
        }
        public enum FontWeight { ... }
        public enum LineHeight { ... }
    }
    public enum Spacing {
        public static let s5: CGFloat = 16
        // ...
    }
    public enum Radius { ... }
    public enum Elevation { ... }
    public enum Duration { ... }
}
```

A `Color(hex:)` extension in the same file converts the integer constants to `SwiftUI.Color`.

### Android output

`core/theme/src/main/kotlin/io/republike/core/theme/generated/Tokens.kt`:

```kotlin
package io.republike.core.theme.generated

import androidx.compose.ui.graphics.Color
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp

object Tokens {
    object Colors {
        val primary = Color(0xFF7B61FF)
        // ...
    }
    object Typography {
        object FontSize {
            val body = 15.sp
            // ...
        }
        object FontWeight { ... }
        object LineHeight { ... }
    }
    object Spacing {
        val s5 = 16.dp
        // ...
    }
    object Radius { ... }
    object Elevation { ... }
    object Duration { ... }
}
```

### Token-name mapping rules

| `tokens.json` path | iOS identifier | Android identifier |
|---|---|---|
| `color.primary` | `Tokens.Color.primary` | `Tokens.Colors.primary` |
| `color.text.primary` | `Tokens.Color.Text.primary` | `Tokens.Colors.Text.primary` |
| `typography.fontSize.body` | `Tokens.Typography.FontSize.body` | `Tokens.Typography.FontSize.body` |
| `spacing.5` | `Tokens.Spacing.s5` | `Tokens.Spacing.s5` (numeric keys prefixed `s`) |
| `radius.2xl` | `Tokens.Radius.xxl` (dashed/numeric → camel) | `Tokens.Radius.xxl` |

The mapping logic is in the script. Lowercase + camelCase rules: dashes → camel; leading digits → prefix with `s` (spacing) or `r` (radius).

## CI enforcement

Both repos run a CI step:

1. `npm run tokens` (re-generate)
2. `git diff --exit-code` — fail if the generated file is out of sync with what's committed

Engineers can't merge a token change without regenerating.

## When `tokens.json` changes

1. PR against `republike-tech` updating `tokens.json`
2. After merge, ONE PR per platform repo running the regen and committing the diff
3. Both regen PRs reference the `republike-tech` commit SHA in their description
4. Reviewer confirms the SHA in the generated file header matches

## Consequences

- Design system changes are a 3-PR ritual (one tech, one ios, one android). Acceptable for the safety it gives.
- No third-party token-system dependency (Style Dictionary, Theo, etc.). Stays vendor-free; the script is ~150 lines and we own it.
- The script can later add Figma-pull support, but that's deferred — hand-edits to `tokens.json` are fine in v1.

## Alternatives considered

- **Style Dictionary.** Considered. Powerful but heavy: another tool to install, JSON config files, theming layers we don't need yet.
- **Hand-write tokens per platform.** Considered. Rejected because that's exactly the drift mode we're trying to prevent.
- **Codegen at build time (Tuist plugin / Gradle task).** Considered for ergonomics but having the generated file committed makes diffs reviewable and CI gating simple.
