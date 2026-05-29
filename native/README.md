# Native rewrite — planning hub

Spec docs, design tokens, ADRs, and the dependency map for the native iOS + Android rewrite of the Republike mobile app.

## Contents

### Architecture + reference

| File | Purpose |
|---|---|
| [`spec.md`](./spec.md) | High-level architecture, module layout, atomic-design rules, phase plan |
| [`data-model.md`](./data-model.md) | Prisma → Swift + Kotlin type map |
| [`api-surface.md`](./api-surface.md) | tRPC procedures the mobile app calls |
| [`screens.md`](./screens.md) | Inventory of every RN screen, scope label, and porting phase |
| [`dependencies.md`](./dependencies.md) | Dependency replacement matrix (RN → native) |
| [`tokens.json`](./tokens.json) | Design tokens — single source for both platforms |

### Phase plans

| File | Purpose |
|---|---|
| [`phase-3-nav-skeleton.md`](./phase-3-nav-skeleton.md) | Tab bar + coordinator / dispatcher skeleton, placeholders for all 5 tabs |
| [`phase-4-design-system.md`](./phase-4-design-system.md) | Atomic-design components, dependency order, per-component APIs |
| [`phase-5-auth.md`](./phase-5-auth.md) | 17 auth + onboarding + paywall screens with flow graph + IAP integration |
| [`phase-6-feed.md`](./phase-6-feed.md) | Feed shell, post card, reactions, post detail, comments, media playback |
| [`phase-7-profile.md`](./phase-7-profile.md) | Own + other profile, edit profile, settings, follower / following lists |

### Operational docs

| File | Purpose |
|---|---|
| [`branching-strategy.md`](./branching-strategy.md) | Branch types, naming, merge rules, release / hotfix flow |
| [`acceptance.md`](./acceptance.md) | PR definition of done, reviewer checklist, smoke-test bar |
| [`assets.md`](./assets.md) | Icons, fonts, images, app icons — conventions per platform |
| [`accessibility.md`](./accessibility.md) | Minimum a11y bar, per-platform mechanisms, reviewer checks |

### ADRs

| ADR | Subject |
|---|---|
| [`adrs/0001-nav-pattern.md`](./adrs/0001-nav-pattern.md) | Navigation: Coordinator (iOS), type-safe Compose Nav (Android) |
| [`adrs/0002-rich-text-editor.md`](./adrs/0002-rich-text-editor.md) | Composer: plain text in v1, defer inline-rich |
| [`adrs/0003-token-codegen.md`](./adrs/0003-token-codegen.md) | `tokens.json` → `Tokens.swift` / `Tokens.kt` codegen pipeline |
| [`adrs/0004-crash-error-reporting.md`](./adrs/0004-crash-error-reporting.md) | Sentry self-hosted on rp-sentry, no SaaS |
| [`adrs/0005-api-client-style.md`](./adrs/0005-api-client-style.md) | Hand-written typed wrappers, no tRPC codegen |
| [`adrs/0006-session-storage.md`](./adrs/0006-session-storage.md) | Keychain (iOS) + EncryptedSharedPreferences (Android), SessionManager |
| [`adrs/0007-localization-workflow.md`](./adrs/0007-localization-workflow.md) | CSV-driven, single source in `republike-tech/native/i18n/` |
| [`adrs/0008-in-app-purchases.md`](./adrs/0008-in-app-purchases.md) | StoreKit 2 + Play Billing 6, server-side receipt verification |
| [`adrs/0009-media-stack.md`](./adrs/0009-media-stack.md) | Nuke + AVKit (iOS), Coil 3 + Media3 (Android) |
| [`adrs/0010-deep-linking.md`](./adrs/0010-deep-linking.md) | Universal Links + App Links, `https://` only public URLs |
| [`adrs/0011-push-notifications.md`](./adrs/0011-push-notifications.md) | APNs (iOS) + FCM (Android), no Firebase Analytics |

## How this is used

- **Both native repos** (`republike-ios`, `republike-android`) link back to specific anchors here from every issue and PR.
- **Updates** go through a PR. ADRs are append-only — once accepted, an ADR is not edited; superseded ADRs link forward.
- **Engineers** pull tokens via the codegen script in each platform repo (see ADR-0003).
- **Reviewers** verify PRs against the linked spec sections + the rules in [`acceptance.md`](./acceptance.md).

## Related repos

- [`republike-ios`](https://github.com/RepublikeIO/republike-ios) — Swift / SwiftUI implementation
- [`republike-android`](https://github.com/RepublikeIO/republike-android) — Kotlin / Compose implementation
- [`republike-mobile`](https://github.com/RepublikeIO/republike-mobile) — current React Native app; **source of truth for behavior** during the port
- [`republike-webapp`](https://github.com/RepublikeIO/republike-webapp) — backend (tRPC); unchanged by the rewrite
