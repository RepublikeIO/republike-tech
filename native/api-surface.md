# API surface

The backend procedures the mobile app calls, grouped by router. The native API clients implement these as typed methods one-for-one. The **source of truth is the webapp tRPC routers** (`republike-webapp/src/server/api/routers/*.ts`) — the same backend the currently-published RN app hits. This doc was re-derived directly from those routers (2026-06-08) after an audit found the previous version described endpoints that do not exist on the server.

> **History:** the prior revision of this file described auth as NextAuth REST (`/api/auth/*`), `user.updateProfile`, `topic.list`, `subscription.verify*`, `subscription.getMyCurrentPlan`, `comment.getThread/create`, `user.getByUsername/followUser/getFollowers`, etc. **None of those names exist on the backend.** The real contract is below.

## Transport

- **All app data is tRPC**, POST/GET to `{baseUrl}/api/trpc/<router>.<procedure>`. Base URL: `https://www.republike.io` (prod) / `https://staging.republike.io` (staging) / `https://dev.republike.io` (dev).
- **superjson transformer, both directions** (mandatory). Inputs are wrapped `{"json": <input>, "meta": ...}`; outputs are `{"result":{"data":{"json": <output>, "meta": ...}}}`. Plain-JSON bodies break. (Queries pass input via `?input=<superjson>`; mutations via the body.) The web client uses `httpBatchLink`; native may call single (non-batched) `/api/trpc/<router>.<proc>`.
- **Auth = Bearer JWT.** `Authorization: Bearer <token>`. The token is the `data` field returned by `user.login`. JWT payload `{ userId, email }`, signed with `JWT_SECRET`. **The token does NOT expire and there is NO refresh endpoint** (the `expiresIn` is commented out server-side). The native client persists the token long-term and re-logs-in on `INVALID_AUTH_TOKEN`. (The NextAuth cookie session in `[...nextauth].ts` is web-only; native does not use it.)
- **Auth tiers:** `public` (no token) · `protected` (any status) · `active` (status must be `ACTIVE`) · `activeOrMissingProfile` (ACTIVE or an in-onboarding status) · `capability(cap)` (active + role capability).

## Onboarding is server-status-driven

The app does NOT hard-sequence onboarding on the client. With a token, it calls `user.getMyUserProfile` and routes on `status` (`UserStatus`). The status machine:

`signupUser` → **EMAIL_VERIFICATION** → `verifyAccount` → **MISSING_PROFILE / CREATE_PROFILE** → `completeProfile` (sets) → **FOLLOW_HASHTAGS** → `topic.followTopics` (sets) → **MISSING_OATH** → `swearOath` (sets) → **PAY_WALL** → `activateFreeTier` _or_ `payment.verify*` (sets) → **ACTIVE**.

`UserStatus` enum: `EMAIL_VERIFICATION, MISSING_PROFILE, CREATE_PROFILE, FOLLOW_HASHTAGS, FOLLOW_TOPICS, MISSING_OATH, PAY_WALL, SUBSCRIPTION_EXPIRED, SUBSCRIPTION_PAYMENT_FAILED, PAYMENT_EXPIRED, ACTIVE`.

Client route map (mirrors RN `mappedRoutes`): `EMAIL_VERIFICATION`→verify · `MISSING_PROFILE`/`CREATE_PROFILE`→profile completion · `FOLLOW_HASHTAGS`/`FOLLOW_TOPICS`→interests · `MISSING_OATH`→principles/oath · `PAY_WALL`→paywall · `SUBSCRIPTION_EXPIRED`/`SUBSCRIPTION_PAYMENT_FAILED`/`PAYMENT_EXPIRED`→subscription-ended · `ACTIVE`→main. There is **no client-side "founder" flag** — skipping onboarding is purely a consequence of the server returning `ACTIVE`.

`Plan` enum: `FREE, TRIAL_PERIOD, BASIC, FOUNDING_CITIZEN, FOUNDING_FATHER, FOUNDING_CONSUL`.

## Auth / account — `user` router

| Procedure | Type / tier | Input | Output | Notes |
|---|---|---|---|---|
| `user.login` | mutation · public | `{ email, password (min6) }` | `{ success: true, data: <token> }` or `{ success: false }` | `data` is the bearer JWT. Never throws on bad creds. **No user object, no refresh, no expiry** — follow with `getMyUserProfile`. |
| `user.signupUser` | mutation · public | `{ email, password (min8), referralCode?, userLanguage?, isOfLegalAge: true }` | `{ success: true }` or `{ success: false, message }` | |
| `user.verifyAccount` | mutation · public | `{ token, email }` | void | also REST `POST /api/user/verify-account`. Email links deep-link here. |
| `user.sendAccountValidationEmail` | mutation · protected | — | `true` | resend; needs token. |
| `user.forgotPassword` | mutation · public | `{ email }` | `{ success: true }` | |
| `user.resetPassword` | mutation · public | `{ token, email, password (min8) }` | void | |
| `user.completeProfile` | mutation · protected | `{ firstName (ALPHA_REGEX), lastName (ALPHA_REGEX), username (USERNAME_REGEX), bio?, avatar?: string\|null, banner?: string\|null }` | full User row | sets status → `FOLLOW_HASHTAGS`. Throws `Username not available`. This is the onboarding profile step (sets the **username**). |
| `user.swearOath` | mutation · protected | — | full User row | sets status → `PAY_WALL`. **Required onboarding step.** |
| `user.activateFreeTier` | mutation · protected | — | MyUserProfile | valid only from `PAY_WALL`; sets `ACTIVE` + `currentPlan=FREE`. "Continue for free". |
| `user.getMyUserProfile` | query · protected | — | MyUserProfile (see below) | canonical "me"; drives status routing + plan/activation (no separate plan endpoint). |
| `user.getUserProfile` | query · protected | `{ username }` | user + `{ nbPosts, nbFollowers, nbFollowings, isFollowing, isMuted, hasReported }` | other-user profile. |
| `user.editProfile` | mutation · protected | `{ firstName, lastName, username, bio?, website?, avatar?, banner? }` | `{ id, firstName, lastName, username, email(masked), bio, website }` | settings edit-profile. |
| `user.updateAvatar` / `user.updateBanner` | mutation · activeOrMissingProfile | `{ avatar }` / `{ banner }` | user subset | post-upload URL set. |
| `user.getPresignedUrlForAvatar` / `...ForBanner` | **query** · activeOrMissingProfile | — | presigned S3 PUT URL (string) | see Uploads. |
| `user.checkUsername` | mutation · protected | `{ username (USERNAME_REGEX) }` | `boolean` — **`true` = TAKEN** | ⚠️ inverse of the REST endpoint. |
| `user.updateLanguage` | mutation · protected | `{ language: 'en'\|'fr' }` | void | settings. |
| `user.deleteAccount` | mutation · protected | — | void | enqueues deletion job. |
| `user.getSettings` / `user.updateEmailPreferences` | query / mutation · protected | — / 8 booleans | settings | |
| `user.getUserFollowers` / `user.getUserFollowings` | query · protected | `{ userId }` | followers / followings array | (non-paginated arrays.) |

**Username availability** also has REST `GET /api/user/is-username-available?username=<x>` → raw boolean, **`true` = AVAILABLE** (opposite polarity to `user.checkUsername`).

**MyUserProfile shape:** `{ id, username, firstName, lastName, bio, website, background, avatar, banner, type, status (UserStatus), language, defaultFeed, currentPlan (Plan|null), founderStatus, creditBalance, badges: [{type, createdAt}], nbPosts, access: { effectiveRole, capabilities: string[], governanceVotePower, postRateLimit, postsToday, expiresAt: ISO|null, source } }`.

## Topics — `topic` router

| Procedure | Type / tier | Input | Output | Notes |
|---|---|---|---|---|
| `topic.getAllTopics` | query · public | — | topics array | interest selection list. |
| `topic.followTopics` | mutation · protected | `{ topics: string[] (min 3) }` | null | **plural, array.** Sets status → `MISSING_OATH`. |
| `topic.followTopic` / `topic.unfollowTopic` | mutation · active | `{ topicId }` | result | single follow (post-onboarding). |
| `topic.getMyTopics` / `topic.getSuggestedTopics` | query | — | topics | |

## Payment — `payment` router

| Procedure | Type / tier | Input | Output | Notes |
|---|---|---|---|---|
| `payment.verifyAppleReceipt` | mutation · protected | `{ receiptData }` | `{ success, purchaseType, purchaseInfo, receiptData }` | iOS receipt validation; updates plan server-side. Throws `BAD_REQUEST` on failure. |
| `payment.verifyGooglePurchaseToken` | mutation · protected | `{ packageName, productId, purchaseToken }` | `{ success, purchaseType, purchaseInfo, receiptData }` | Android. |

**There is no `getMyCurrentPlan` / plan-status procedure.** After a purchase, poll `user.getMyUserProfile().status === ACTIVE` (and read `currentPlan`). Stripe procs (`payment.handleSubscription`, `createPaymentIntent`, `createSetupIntent`) are web-only.

## Feed & content — `post` router

| Procedure | Type / tier | Input | Output | Notes |
|---|---|---|---|---|
| `post.getBatch` | query · active | `{ fetchMode?: 'follows'\|'latest'\|'discover'\|'hot'\|'topics'\|'bookmarks'\|'news'\|'best'\|'home', selector?: { hashtag?, topic? }, cursor?, limit (1–100, def 10) }` | cursor page of posts | `fetchMode` optional; `'home'` = unified timeline. **The `profile-*` modes the native apps invented do NOT exist** — use `getUserPosts` for profile feeds. |
| `post.getUserPosts` | query · active | `{ userId, fetchUserPostsType?: FetchUserPostsTypeEnum (def POSTS), cursor?, limit }` | cursor page | profile Posts/Replies/Reposts via `fetchUserPostsType`. |
| `post.getPostWithRepliesForMobile` | query · protected | `{ postId }` | `{ ...post, rootPostId }` | **post detail for native** (thread included). |
| `post.replyPost` | mutation · active | `{ postId, content, videoUrl?: string\|null }` | created reply | this is comment-create (max 4 nesting levels). |
| `post.reactToPost` | mutation · active | `{ postId, reaction: PostReactionEnum }` | userReaction | LIKE/AGREE/SMART/USEFUL/INSPIRING… Throws on duplicate. |
| `post.createPost` | mutation · capability `post:create:limited` | `{ content, videoUrl?, repostPostId?, visibility?, poll?: { options: string[], duration } }` | post | composer. |
| `post.editPost` / `post.deletePost` | mutation · active | `{ postId, ... }` / `{ postId }` | result | |
| `post.votePoll` / `post.getPollResults` | mutation / query · active | `{ postId, option }` / `{ postId }` | result | |
| `post.createTip` | mutation · capability `tips:give` | `{ postId, amount (1–1000) }` | tip | |
| `post.getTotalTipsForPost` / `post.getTipHistoryForPost` | query | `{ postId }` | totals / history | |
| `post.bookmarkPost` / `removeBookmarkPost` / `viewPost` / `markPostAsRead` / `reportPost` | mutation · active | `{ postId, ... }` | result | |
| `post.getPreviewByUrl` / `post.searchGifs` / `post.getPostTranslation` | query/mutation · active | `{ url }` / `{ query }` / `{ postId }` | … | composer/detail helpers. |

> `post.getActivationStatus` exists (`{ current, target, state }`) but the native apps hide the feed-activation surface — do not call it. There is **no separate `comment` router**: comments are posts (`getPostWithRepliesForMobile` to read, `replyPost` to create).

## Other routers (for later surfaces)

- `notification.*` — `getAll { fetchMode (NotificationsTabsEnum), cursor?, limit }`, `getMyUnreadNotificationsCount`, `markAsRead`, `markAllAsRead`, `delete`.
- `search.*` — `searchUsers` / `searchPosts` / `searchHashtag`, each query · active, input `{ searchTerm, page (def 0) }`.
- `hashtag.*` — `followMany { hashtags: string[] (min 5) }`, `searchHashtags` (mutation), `getMyHashtagsNames`, `getSuggestedHashtags`.
- `contact.subscribeToNewsletter` (public).
- `subscription.cancel` / `getSubscriptionByStripeCustomerId` (Stripe-customer only).

## Uploads (avatar / banner / post media)

Two-step presigned S3 PUT: (1) call `user.getPresignedUrlForAvatar` / `...ForBanner` (queries) or `user.getPresignedUrlForPost` (mutation) → presigned URL (expires 15 min, `ContentType: image/jpeg`, public-read); (2) HTTP **PUT** the raw JPEG bytes to that URL; (3) persist the resulting object URL via `updateAvatar`/`updateBanner` or inside `completeProfile`/`editProfile`. Keys: `uploads/avatars/<userId>/<uuid>`, `uploads/banners/<userId>/<uuid>`, `uploads/users/<userId>/<uuid>`.

## Error contract

tRPC errors return `code` + `message`. Auth failures use code `UNAUTHORIZED` with message strings `NO_AUTH_TOKEN` / `INVALID_AUTH_TOKEN` / `USER_INACTIVE`. Capability failures use `FORBIDDEN` with message `INSUFFICIENT_ROLE` and `cause: { capability, currentRole, requiredRole }`. The native client maps these to a typed enum:

```swift
public enum APIError: Error {
    case unauthorized           // NO_AUTH_TOKEN / INVALID_AUTH_TOKEN → re-login
    case forbidden              // INSUFFICIENT_ROLE → upgrade prompt
    case notFound
    case rateLimited
    case userInactive           // USER_INACTIVE → subscription-ended / paywall
    case validation(String)
    case server(String)
    case network(URLError)
}
```

Same enum on Android. `userInactive` is intercepted in the client middleware and the root coordinator routes to the paywall/subscription-ended surface.
