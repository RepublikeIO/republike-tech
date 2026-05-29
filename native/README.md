# Native rewrite — planning hub

Spec docs, design tokens, ADRs and the dependency map for the native iOS + Android rewrite of the Republike mobile app.

## Contents

| File | Purpose |
|---|---|
| [`spec.md`](./spec.md) | High-level architecture, module layout, phase plan |
| [`data-model.md`](./data-model.md) | Prisma → Swift + Kotlin type map |
| [`api-surface.md`](./api-surface.md) | tRPC procedures the mobile app calls |
| [`screens.md`](./screens.md) | Inventory of every RN screen, scope label, and porting status |
| [`dependencies.md`](./dependencies.md) | Dependency replacement matrix (RN → native) |
| [`tokens.json`](./tokens.json) | Design tokens — single source for both platforms |
| [`adrs/`](./adrs/) | Architecture decision records, numbered, immutable |

## How this is used

- **Both native repos** (`republike-ios`, `republike-android`) link back to specific anchors here from every issue and PR.
- **Updates** to anything here go through a PR. ADRs are append-only — once accepted, an ADR is not edited; superseded ADRs link forward.
- **Engineers** pull tokens via the codegen script in this folder (committed Phase 1) into their platform repos.

## Related repos

- [`republike-ios`](https://github.com/RepublikeIO/republike-ios) — Swift / SwiftUI implementation
- [`republike-android`](https://github.com/RepublikeIO/republike-android) — Kotlin / Compose implementation
- [`republike-mobile`](https://github.com/RepublikeIO/republike-mobile) — current React Native app; **source of truth for behavior** during the port
- [`republike-webapp`](https://github.com/RepublikeIO/republike-webapp) — backend (tRPC); unchanged by the rewrite
