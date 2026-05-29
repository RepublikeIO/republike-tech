# Phase 7 — Profile surface

Final surface in the first delivery. Reuses ~80% of Phase 6 components (PostCard, MediaCarousel, ListItemRow, etc.) plus the new `ProfileHeader` organism from Phase 4.

## Scope

| Surface | Components |
|---|---|
| Own profile | `ProfileHeader` (with Edit button), tab strip (Posts / Replies / Reposts), feed reusing `PostCard` |
| Other user profile | `ProfileHeader` (with Follow / Unfollow), same tab strip + feed |
| Edit profile | Form (avatar + fields + bio) using `FormField` molecules |
| Settings | Stack of `ListItemRow` rows |

## Sub-surfaces (in implementation order)

### 7.1 Profile header + tab strip + posts list (1.5 sprints)

The bulk of the work. Both "own" and "other" reuse the same `ProfileHeader` organism with a variant flag:

```swift
ProfileHeader(user: user, variant: .own)        // shows Edit button
ProfileHeader(user: user, variant: .other)      // shows Follow / Unfollow
```

```kotlin
ProfileHeader(user, variant = ProfileHeaderVariant.Own)
ProfileHeader(user, variant = ProfileHeaderVariant.Other)
```

Header contents (matches RN):
- Avatar (xl)
- Display name + username
- Founder badge / consul badge if applicable
- Bio
- Stats row: post count + follower count + following count (each tap → list screen)
- Primary CTA (Edit / Follow / Unfollow)

Tab strip below header:
- **Posts** (default)
- **Replies** (posts where the user is a commenter on someone else's post — same Post shape)
- **Reposts** (posts the user repost-quoted)

Each tab renders a feed using `PostCard` from Phase 4 + behavior from Phase 6. Lazy-load on tab focus.

API:
- `user.getByUsername(_:)` for other profiles
- `user.getMyUserProfile()` for own (already on mount via SessionManager)
- `post.getBatch(fetchMode:cursor:)` with profile-scoped fetch mode (`profile-posts`, `profile-replies`, `profile-reposts`) — confirm these modes exist server-side; if not, this is an `api-surface.md` extension

### 7.2 Edit profile (0.5 sprint)

A form screen reached from Own Profile → Edit.

Fields (per RN):
- Avatar (tap → photo picker → upload)
- First name
- Last name
- Username (read-only or editable per server policy)
- Bio (multiline `TextField`, max 280 chars, character counter)
- Language preference (picker EN / FR / IT / ES)

API: `user.updateProfile(input:)`.

Save behavior:
- Optimistic: apply changes locally, fire the API call, revert + toast on failure.
- "Save" disabled until at least one field changed.

### 7.3 Settings (1 sprint)

Stack of `ListItemRow` molecules grouped by section.

| Section | Rows |
|---|---|
| Account | Edit profile, Email, Language, Sign out |
| Subscription | Current plan, Manage subscription (deep link to Apple ID / Play subscription page) |
| Notifications | Push permission state, per-category toggles (placeholder until in-app notifications ship) |
| Privacy | Data export request, Privacy policy (webview), Terms of Use (webview) |
| Help | FAQ link (webview), Contact support |
| About | App version + build number, libraries / acknowledgments |
| Danger zone | Delete account |

Most rows tap-route or open a sheet. The Danger zone "Delete account" calls a confirmation `AlertDialog` then `user.deleteAccount()` (confirm exists server-side; if not, infra ticket).

### 7.4 Follower / Following lists (0.5 sprint)

Two list screens reached by tapping the stats row.

Components: `ListItemRow` per follower (avatar + name + Follow / Unfollow inline button).
API: `user.getFollowers(cursor:)`, `user.getFollowing(cursor:)`, `user.followUser(_:)`, `user.unfollowUser(_:)`.

## Issues (per platform — opened after Phase 6 closes, ~2 sprints scope)

- 1 issue: ProfileHeader wiring (Own + Other variants)
- 1 issue: Profile tab strip + posts list
- 1 issue: Replies tab + Reposts tab
- 1 issue: Edit profile screen
- 1 issue: Settings screen
- 1 issue: Follower / Following list screens
- 1 issue: Follow / Unfollow optimistic updates from the feed too (PostCard header gets a Follow tap target)
- 1 issue: Account deletion flow + confirmation dialog
- 1 issue: Subscription management deep link (Apple ID / Play subscriptions URL)
- 1 issue: Deep link → Profile wiring (`/u/<username>`)
- 1 tracking issue

Total: ~11 issues per platform.

## Acceptance for Phase 7

- Tapping a username in the feed lands on that user's profile in ≤1s.
- Own profile shows the correct logged-in user with all stats updated post-action.
- Following / unfollowing a user is instant (optimistic) and propagates to the feed.
- Edit profile saves correctly; avatar upload works.
- Settings rows route to the expected destination.
- Account deletion goes through a confirmation dialog + 7-day grace period UI (matches webapp behavior).
- A user with no posts sees an `EmptyState` molecule, not an empty list.

## Maps to RN source

- `republike-mobile/src/Screens/Main/Profile/MyProfileScreen.tsx`
- `republike-mobile/src/Screens/Main/Profile/OtherProfileScreen.tsx`
- `republike-mobile/src/Screens/Main/Profile/EditProfileScreen.tsx`
- `republike-mobile/src/Screens/Main/Profile/SettingsScreen.tsx`
- `republike-mobile/src/Screens/Main/Profile/FollowersScreen.tsx` / `FollowingScreen.tsx`
- `republike-webapp/src/server/api/routers/user.ts` — webapp side
