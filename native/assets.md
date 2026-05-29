# Assets

Conventions for icons, images, fonts. Both platforms share the same naming where possible.

## Icons

System-first. We use the OS-provided icon set in 95% of cases:

| | iOS | Android |
|---|---|---|
| Source | SF Symbols 5 | Material Symbols (rounded) |
| Render | `Image(systemName: "arrow.left")` | `Icon(painter = painterResource(...))` from Material Symbols Compose |
| Color | Always from `Tokens.Color` — `.foregroundStyle()` / `.tint` |

A small set of brand icons (the Republike logo glyphs, the reaction icons, the consul badge) is **vendor**:

```
republike-ios/Resources/Icons/
├── republike-logo.svg
├── reaction-agree.svg
├── reaction-smart.svg
├── ...

republike-android/app/src/main/res/drawable/
├── ic_republike_logo.xml      (vector drawable)
├── ic_reaction_agree.xml
├── ...
```

SVG → vector drawable conversion uses Android Studio's import. Both platforms keep the source SVG in `republike-tech/native/assets/icons/` for parity reference.

## Reaction icons

The 8 reactions (agree, smart, useful, inspiring, disagree, aggressive, deceptive, unverifiable) have fixed glyphs from the RN app. Source: `republike-mobile/assets/icons/reactions/*.png`.

These are the only place we accept raster assets, and only because the originals are PNG with hand-tuned aliasing. They live in:

```
republike-ios/Resources/Icons/Reactions/
├── agree@1x.png agree@2x.png agree@3x.png
├── ...

republike-android/app/src/main/res/drawable-{mdpi,hdpi,xhdpi,xxhdpi}/
├── reaction_agree.png
├── ...
```

When you ship a new reaction-icon variant, update `republike-tech/native/assets/reactions/` first, then run the per-platform import script.

## Images

In-app images (not user-content) live in:

| | iOS | Android |
|---|---|---|
| Location | `Resources/Images/` | `app/src/main/res/drawable*/` |
| Source | PNG + @2x/@3x via Asset Catalog | WEBP preferred, PNG fallback, density buckets |

User-content images come from the API as URLs (no bundling).

## Fonts

Brand font is **TT Firs Neue** (per current RN setup).

| | iOS | Android |
|---|---|---|
| Files | `Resources/Fonts/TTFirsNeue-{Regular,Medium,Semibold,Bold}.otf` | `app/src/main/res/font/tt_firs_neue_{regular,medium,semibold,bold}.otf` |
| Register | `Info.plist` `UIAppFonts` | Auto-detected from `res/font/` |
| Access | `Tokens.Typography.fontFamily.main` | Material 3 `Typography` configured with the registered family |

Mono font: SF Mono (iOS — system), Roboto Mono (Android — system).

Source files for the brand font live in the team's design vault (NOT in git for licensing). Engineers grab them at setup time per README instructions.

## App icons

Configurable per environment (debug / staging / production) to make it obvious which build a tester is running:

| Environment | iOS icon | Android icon |
|---|---|---|
| Debug | red dot overlay on the main logo | red dot overlay |
| Staging | amber dot overlay | amber dot overlay |
| Production | clean logo | clean logo |

Source PSDs in `republike-tech/native/assets/app-icons/`. Exported assets land in the platform repos via App Icon Sets (iOS) / mipmap density buckets (Android).

## Splash / launch screen

| | iOS | Android |
|---|---|---|
| Mechanism | `LaunchScreen.storyboard` referenced by Tuist | Splash Screen API (Android 12+) |
| Content | Republike logo centered on `Tokens.Color.surface.background` | same |

No animation — system splash transitions straight into the app's first screen.

## Dark mode

Every committed asset must have a dark-mode variant if its appearance is brand-tinted. For neutral assets (e.g. reaction icons), the same source is used and tinted via `Tokens.Color`. For brand assets (the logo on a light background vs the logo on a dark background), two files are committed:

- iOS: use Asset Catalog Appearance variants.
- Android: `drawable-night/` folder for dark equivalents.

## Naming rules

| Asset type | Convention |
|---|---|
| iOS asset catalog | kebab-case: `republike-logo`, `reaction-agree` |
| Android resources | snake_case (required by aapt): `republike_logo`, `reaction_agree` |
| Source SVGs in `republike-tech/` | kebab-case: `republike-logo.svg` |

The codegen / import scripts handle the cross-platform name translation.
