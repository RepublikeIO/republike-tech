# ADR 0005 — API client style

**Status:** Accepted (Phase 2)
**Date:** 2026-05-29

## Context

The webapp exposes a tRPC API (`republike-webapp/src/server/api/routers/*`). The mobile app calls a known, bounded subset of those procedures (~20 — see [`../api-surface.md`](../api-surface.md)). We need a native client style that:

1. Stays in sync with the webapp's procedure signatures.
2. Surfaces type-safe inputs and outputs in Swift / Kotlin.
3. Doesn't drag in third-party tRPC codegen tooling that is rough in both languages.
4. Is reviewable — adding a procedure should be one short, mechanical PR per platform.

## Decision

**Hand-written typed wrappers per service.** No codegen from tRPC schema. No client-side schema validation (server enforces). The contract for each procedure is documented in `api-surface.md`; the native client mirrors it.

### Why not codegen

- `trpc-codegen-swift` and `trpc-codegen-kotlin` exist but are community-maintained, lag tRPC versions, and emit boilerplate-heavy code.
- Auto-generated names lose the readability of curated method names (`api.user.followUser(_:)` reads better than the generated `api.userFollowUser(input:)`).
- The mobile surface is small (~20 procedures). Cost of hand-writing is bounded; cost of maintaining a codegen pipeline is open-ended.

### Why service-grouped, not flat

Each tRPC router (`user`, `post`, `comment`, `subscription`, `topic`, `hashtag`, `tip`) is a Swift / Kotlin service struct. `protocol API` (Swift) / `interface Api` (Kotlin) exposes each service as a property:

```swift
public protocol API {
    var user: UserAPI { get }
    var post: PostAPI { get }
    var comment: CommentAPI { get }
    var subscription: SubscriptionAPI { get }
    var topic: TopicAPI { get }
    var hashtag: HashtagAPI { get }
    var tip: TipAPI { get }
}
```

```kotlin
interface Api {
    val user: UserApi
    val post: PostApi
    val comment: CommentApi
    val subscription: SubscriptionApi
    val topic: TopicApi
    val hashtag: HashtagApi
    val tip: TipApi
}
```

Mirroring router names gives reviewers an obvious mapping when comparing native calls to webapp source.

### Transport

Each procedure is a POST to `https://<base>/api/trpc/<router>.<procedure>` with a superjson-wrapped JSON body:

```
POST /api/trpc/user.followUser
Content-Type: application/json
Authorization: Bearer <session-jwt>

{ "json": { "userId": "abc123" }, "meta": { "values": {} } }
```

Response shape:

```
{ "result": { "data": { "json": <response>, "meta": ... } } }
```

The HTTPClient layer (per ADR-0003 — wait, this is documented in spec.md §6 and the HTTP issue) handles superjson wrapping / unwrapping so service methods deal in plain Swift / Kotlin types.

### Type identity rules

- All request inputs are dedicated `Codable` / `@Serializable` structs colocated with the service file:
  - `UserAPI.FollowUserInput`
  - `UserApi.FollowUserInput`
- All responses use the models in [`../data-model.md`](../data-model.md) where applicable; new response shapes get their own dedicated structs.
- No anonymous dictionaries, no `[String: Any]`, no `JsonObject` in service signatures.

## Consequences

- Adding a tRPC procedure is a 3-PR change: webapp (new procedure), iOS (new method on service), Android (same). All three reviewed against each other.
- We can't auto-detect server-side schema changes — relies on `api-surface.md` being current. Discipline cost.
- No third-party tRPC client dependency to maintain or version-pin.
- Easier to evolve the native client toward a non-tRPC backend later (REST, GraphQL, gRPC) since there's no codegen lock-in.

## Maps to RN source

The current RN client uses `@trpc/react-query`, which IS the codegen path (TypeScript types flow end-to-end). We give that up in exchange for the bullets above. The RN approach stays the source of truth for the procedure list during the rewrite; once parity ships, the RN app is retired and the native clients become the canonical mobile callers.

## Alternatives considered

- **Generated Swift / Kotlin client from tRPC schema** — rejected for the reasons above.
- **REST adapter on the webapp.** Add a `/api/mobile/*` REST namespace and stop calling tRPC from native. Cleaner long-term but adds a server-side surface to maintain in parallel. Punted to a later ADR if the mobile/web procedure lists diverge enough to justify.
- **GraphQL** (Apollo iOS / Apollo Android) — first-class codegen exists. Rejected because adopting GraphQL is a webapp-side rewrite not on the table.
