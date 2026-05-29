# Data model

The native apps use a flat set of value types ported from `republike-webapp/prisma/schema.prisma`. Only the fields the mobile API surface returns are kept — internal-only columns (Stripe customer ids, soft-delete flags, audit fields not surfaced) are omitted on the client side.

This file is the contract. Issues that touch model types must update both this file and the generated Swift / Kotlin files in the same PR.

## Conventions

| | iOS | Android |
|---|---|---|
| Struct shape | `struct Foo: Codable, Sendable` | `@Serializable data class Foo(...)` |
| Optionals | `Bar?` | `Bar?` |
| Dates | `Date` (ISO 8601 via decoder strategy) | `Instant` (kotlinx-datetime) |
| Decimals | `Decimal` | `BigDecimal` |
| Ids | typed `struct PostId: RawRepresentable<String>` | `value class PostId(val raw: String)` |

## Core types

### User

```ts
// req. by feed + profile + auth
{
  id: string
  firstName: string
  lastName: string
  username: string
  email: string                    // self only
  bio: string | null
  website: string | null
  avatar: string | null            // S3 URL
  background: string | null
  status: UserStatus
  type: UserType
  currentPlan: SubscriptionPlan
  language: 'EN' | 'FR' | 'IT' | 'ES'
  createdAt: Date
  // counts (denormalised)
  followerCount: number
  followingCount: number
  postCount: number
}
```

### Post

```ts
{
  id: string
  userId: string
  user: UserMinimal               // embedded for feed
  title: string | null
  content: string                  // markdown
  contentHtml: string | null       // pre-rendered
  category: string
  hashtags: string[]
  media: Media[]
  poll: Poll | null
  // engagement counters
  likeCount: number
  dislikeCount: number
  commentCount: number
  repostCount: number
  // per-reaction counts (8 buckets)
  agreeCount: number
  smartCount: number
  usefulCount: number
  inspiringCount: number
  disagreeCount: number
  aggressiveCount: number
  deceptiveCount: number
  unverifiableCount: number
  // viewer-specific
  myReaction: PostReaction | null
  isBookmarked: boolean
  // moderation
  moderationStatus: 'OK' | 'FLAGGED' | 'MODERATING' | 'REMOVED'
  parentPostId: string | null      // for replies / reposts
  rootPostId: string | null
  createdAt: Date
  updatedAt: Date | null
}
```

### Media

```ts
{
  id: string
  type: 'IMAGE' | 'VIDEO' | 'GIF'
  url: string
  thumbnailUrl: string | null
  width: number
  height: number
  durationSeconds: number | null   // video only
}
```

### Reactions

```ts
type ReactionType =
  | 'AGREE' | 'SMART' | 'USEFUL' | 'INSPIRING'
  | 'DISAGREE' | 'AGGRESSIVE' | 'DECEPTIVE' | 'UNVERIFIABLE'

type PostReaction = {
  reaction: ReactionType
  createdAt: Date
}
```

### Subscription plan + access

```ts
type SubscriptionPlan = 'FREE_CITIZEN' | 'CITIZEN' | 'FOUNDING_CITIZEN' | 'FOUNDING_FATHER' | 'FOUNDING_CONSUL'

// returned by user.getMyUserProfile / activateFreeTier
type AccessContext = {
  role: SubscriptionPlan
  capabilities: string[]           // e.g. ['tips:give', 'governance:vote']
  postRateLimit: number            // posts/day allowed
  postsToday: number
  postsLeftToday: number | null    // null = unlimited (paid roles)
  feedVisibilityBoostHours: number // 0 for free, 2/4/8 for founders
  votePower: number
}
```

### Hashtag

```ts
{
  id: string
  name: string
  postCount: number
}
```

### Topic

```ts
{
  id: string
  name: string                     // e.g. "Politics"
  iconName: string
}
```

### Comment

Same shape as `Post` (single table on the backend), but limited fields when nested under a parent in feed responses (lazy-loaded full thread on tap).

## Type ownership

| Type | Lives in | Generated? | Owner |
|---|---|---|---|
| All `Models` | `Core/Models/*.swift` and `Core/Models/*.kt` | manual | both repos |
| `ReactionType`, `SubscriptionPlan`, `UserStatus`, `MediaType`, `ModerationStatus` enums | same | manual | both repos |
| Request / response wrappers per API | `Core/Network/Requests/*.swift` and `.kt` | manual | both repos |

We do **not** codegen models from Prisma. The webapp's Prisma schema is the source of truth for shape, but the native models stay hand-written for clarity and to drop fields the client doesn't need.

## When the schema changes

1. Update [`prisma/schema.prisma`](../../republike-webapp/prisma/schema.prisma) in the webapp.
2. Update this file with the new field.
3. Open a PR against `republike-ios` updating the Swift model.
4. Open a PR against `republike-android` updating the Kotlin model.
5. Both PRs link to the same issue in `republike-tech`.

No mobile model lands without all three PRs merged in the same week.
