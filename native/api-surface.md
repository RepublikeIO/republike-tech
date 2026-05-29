# API surface

Every tRPC procedure the mobile app calls today, grouped by service. The native API clients implement these as typed methods one-for-one. Procedures the mobile app does NOT call are excluded — the native apps don't ship dead surface.

The base URL stays `https://www.republike.io` (prod) / `https://staging.republike.io` (staging) / `https://dev.republike.io` (dev). All endpoints are tRPC-style POST to `/api/trpc/<procedure-name>`.

Headers required on every authenticated call:

- `Authorization: Bearer <session-token>` (NextAuth session JWT)
- `Content-Type: application/json`
- `x-superjson: true` (mobile sends/receives superjson-wrapped payloads — see Network section in `spec.md`)

## Procedure list

### `user`

| Procedure | Used in | Notes |
|---|---|---|
| `user.getMyUserProfile` | every screen on mount | returns user + access context |
| `user.activateFreeTier` | paywall flow | upgrades from anonymous → FREE_CITIZEN |
| `user.updateProfile` | edit profile | partial updates |
| `user.getByUsername` | other-user profile | |
| `user.followUser` | profile / feed actions | |
| `user.unfollowUser` | profile / feed actions | |
| `user.getFollowers` | profile follower list | paginated |
| `user.getFollowing` | profile following list | paginated |
| `user.searchUsers` | deferred (search) | |

### `post`

| Procedure | Used in | Notes |
|---|---|---|
| `post.getBatch` | home feed | `fetchMode: 'latest' \| 'home'` + cursor |
| `post.getById` | post detail | |
| `post.getActivationStatus` | feed header (in RN) — **drop in native** | algorithm activation surface is hidden, no need to call this |
| `post.reactToPost` | feed + post detail | |
| `post.getTipHistoryForPost` | post footer | self-posts only |
| `post.getPreviewByUrl` | composer (deferred) | |

### `comment`

| Procedure | Used in | Notes |
|---|---|---|
| `comment.getThread` | post detail | nested replies |
| `comment.create` | composer (deferred) | |

### `auth` / `session`

| Procedure | Used in | Notes |
|---|---|---|
| NextAuth credentials login | login screen | uses `/api/auth/callback/credentials` (REST), not tRPC |
| NextAuth signup flow | register screen | uses `/api/auth/signup` (REST) |
| Email verification | verify screen | REST: `/api/auth/verify` |
| Password reset request | forgot password | REST: `/api/auth/forgot-password` |
| Password reset confirm | reset password | REST: `/api/auth/reset-password` |

The auth flows are the only non-tRPC endpoints the native clients call. They're handled by a separate `AuthAPI` namespace in each native client.

### `subscription`

| Procedure | Used in | Notes |
|---|---|---|
| `subscription.verifyAndroidPurchase` | post-StoreKit/Play-Billing | server-side receipt validation |
| `subscription.verifyApplePurchase` | post-StoreKit | |
| `subscription.getMyCurrentPlan` | paywall + profile | |

### `topic`

| Procedure | Used in | Notes |
|---|---|---|
| `topic.list` | interest selection | |
| `topic.followTopic` | interest selection | |

### `hashtag`

| Procedure | Used in | Notes |
|---|---|---|
| `hashtag.getTrending` | feed + composer | |

### `tip`

| Procedure | Used in | Notes |
|---|---|---|
| `tip.send` | post footer | gated by `tips:give` capability |

## Error contract

All tRPC errors come back with `code` + `message`. The native client maps the codes to a typed enum:

```swift
public enum APIError: Error {
    case unauthorized
    case forbidden
    case notFound
    case rateLimited
    case userInactive          // triggers SubscriptionEndedScreen
    case validation(String)
    case server(String)
    case network(URLError)
}
```

Same enum on Android. The `userInactive` case maps to RN's existing `USER_INACTIVE` middleware response — see `republike-mobile/src/Services/ApiProvider.tsx:25-80` for current handling. Native intercepts this in the API client middleware and the root coordinator routes to the paywall.
