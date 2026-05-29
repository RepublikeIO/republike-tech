# Phase 6 — Feed surface

The biggest phase. The screen users spend the most time on. The one whose "off feel" the rewrite is most justified by. Lands after Phase 5 ships, runs ~5 sprints at full focus.

## Scope

| Surface | Status |
|---|---|
| Home feed shell (Agora + For You tabs, header, composer trigger) | new native |
| Feed list (chronological / personalized ranking — same API as RN) | new native |
| Pull-to-refresh + infinite scroll | new native |
| Post card (header, content, media, reactions, footer) | new native |
| Reactions: 8-bucket palette + reaction-stats sheet | new native |
| Tip flow trigger (gated by `tips:give` capability — sheet UI deferred) | trigger only |
| Post detail screen | new native |
| Comments thread under post detail | new native |
| Media playback (image / GIF / video — per ADR-0009) | new native |
| HTML-content rendering (legacy posts have inline-rich HTML) | new native |

## Sub-surfaces (in implementation order)

### 6.1 Feed shell + tabs (1 sprint)

- `FeedScreen` template using `ScrollableScreen` + tab switcher (`Agora` / `For you`)
- `FeedTopBar` organism (logo + profile menu)
- `PostComposer` placeholder (composer is deferred, but the trigger card sits at the top of the feed)
- Pull-to-refresh + infinite scroll plumbed against `post.getBatch`

Per platform:
- iOS: SwiftUI `List` + `.refreshable {}`; visibility-based prefetch via `.onAppear` on the last 5 items.
- Android: Compose `LazyColumn` + `PullToRefresh` from Compose-Material3-Pull-to-Refresh; prefetch via Coil `ImageLoader.execute()`.

### 6.2 Post card (1 sprint)

Uses the `PostCard` organism from Phase 4 plus:

- `MediaCarousel` molecule — added in Phase 6; not in Phase 4 because it's feed-specific
- `PollDisplay` organism — added in Phase 6; carries the existing RN poll/survey behavior
- Mention + hashtag tap targets via `Label` parsing

iOS:
- HTML → `AttributedString` via a `RichTextRenderer` utility (in `Core/Utilities/`). Handles `<a>`, `<strong>`, `<em>`, mentions (`<span data-type="mention">`), hashtags, inline images (legacy posts only — for new posts, composer is plain-text, so this is read-only legacy support).
- `MediaCarousel` is a `TabView(.page)`.

Android:
- HTML → `AnnotatedString` via the same renderer ported to Kotlin.
- `MediaCarousel` is a `HorizontalPager`.

### 6.3 Reactions (1 sprint)

The 8 reactions + stats sheet. Most-touched UX in the feed.

- Reaction tooltip on long-press (`Tooltip` molecule from Phase 4; if not present, ship in 6.3)
- Optimistic UI: tap a reaction → instant icon change + count update, server call fires in background
- Vote-once semantics: once a user reacts, the icon is locked (matches RN behavior — see `republike-mobile/src/shared/components/composed/FeedCard/FeedListItem/FeedCardFooter.tsx`)
- Reaction stats sheet: organism (Phase 4) showing per-reaction counts + the list of users who reacted

API: `post.reactToPost(_:reaction:)`.

### 6.4 Tip trigger (0.5 sprint)

Just the trigger UI in the footer:
- Tap → if `can('tips:give')` is true, show "Tips coming soon" snackbar (the actual tip flow is a deferred surface)
- If `can('tips:give')` is false, show upgrade toast routing to Paywall

This matches the current RN behavior in `republike-mobile/src/shared/components/composed/FeedCard/FeedListItem/FeedCardFooter.tsx:395-425`.

### 6.5 Post detail (1.5 sprints)

A scrollable screen showing the post + its comment thread.

- Top: same `PostCard` organism, expanded (no "see more" truncation)
- Below: comments thread

API: `post.getById(_:)`, `comment.getThread(postId:)`.

Comments are recursive (replies under replies up to nestingLevel 2 — see RN behavior). The thread UI is a flat list with indentation indicating nesting (matches RN).

Components used:
- `PostCard` organism
- `CommentItem` organism (new in 6.5) — same shape as PostCard but compact
- `ReactionPill` molecule
- `EmptyState` molecule for "Be the first to comment"

Composer is deferred — Phase 6 does NOT include "Reply to comment" UI; placeholder is "Replies coming soon" snackbar.

### 6.6 Media playback (1 sprint, runs in parallel with 6.5)

Per ADR-0009.

- Video auto-play on becoming visible (≥75% of view in viewport)
- Single-instance manager (`VideoPlaybackCoordinator`)
- Tap toggle sound; long-press fullscreen
- GIF: handled by Nuke (iOS) / Coil-gif (Android) — no special component needed
- Image zoom: pinch-to-zoom + pan in fullscreen viewer (iOS: custom `MagnificationGesture`; Android: `Modifier.transformable`)

## Issues (per platform — opened after Phase 5 closes, ~5 sprints scope)

Grouped roughly by sub-surface:

- 1 issue: FeedScreen shell + tabs
- 1 issue: post.getBatch wiring + pull-to-refresh + infinite scroll
- 1 issue: PostCard wiring (uses Phase 4 organism, adds feed-specific bindings)
- 1 issue: MediaCarousel molecule
- 1 issue: PollDisplay organism
- 1 issue: HTML → AttributedString / AnnotatedString renderer
- 1 issue: Reactions tap + optimistic update
- 1 issue: Reaction stats sheet
- 1 issue: Tip trigger (gated, toast variants)
- 1 issue: PostDetail screen
- 1 issue: CommentItem organism + thread layout
- 1 issue: Video playback (AVKit / Media3) + VideoPlaybackCoordinator
- 1 issue: Image zoom viewer
- 1 issue: Deep link → PostDetail wiring
- 1 issue: Analytics funnel events for feed (server-side aggregate counters)
- 1 tracking issue

Total: ~16 issues per platform → 32 across both.

## Acceptance for Phase 6

- The feed scrolls smoothly at 60fps on iPhone SE 3 (iOS) / Pixel 6 (Android) with 100 posts loaded.
- Switching Agora ↔ For You preserves scroll position per tab.
- Pull-to-refresh works and reloads correctly.
- Reactions feel instant (optimistic).
- Post detail loads from a deep link in ≤2s on 4G.
- Videos auto-play on visibility, pause on scroll-out, single-instance at a time.
- Images zoom smoothly without dropped frames.
- Legacy HTML-content posts render correctly (no `<` characters visible, all formatting preserved).
- USER_INACTIVE in the middle of a feed scroll → coordinator routes to paywall without losing scroll position when the user returns.

## Maps to RN source

- `republike-mobile/src/Screens/Main/HomeFeed/NewHomeFeedScreen.tsx`
- `republike-mobile/src/Screens/Main/Post/PostDetailScreen.tsx`
- `republike-mobile/src/shared/components/composed/FeedCard/*` — current post card + footer
- `republike-mobile/src/shared/components/composed/ReactionStatsBottomSheet.tsx`
- `republike-mobile/src/shared/components/composed/FeedCard/FeedListItem/FeedCardFooter.tsx:395-425` — tip-gating logic
- `republike-mobile/src/server/services/cache.service.ts` — feed-ranking reference
- `republike-mobile/src/server/api/routers/post.ts` — webapp side
