# ADR 0009 — Image + video stack

**Status:** Accepted (Phase 6)
**Date:** 2026-05-29

## Context

The feed renders images, GIFs, and videos at scale. Performance and feel here drive perceived app quality more than anywhere else. RN's `expo-image` + `react-native-video` are competent but bridge-bound; native equivalents win.

## Decision

### iOS

| Concern | Choice |
|---|---|
| Image loading + caching | **Nuke** (latest stable) — used via `NukeUI.LazyImage` |
| GIF rendering | Nuke's `Image` view supports animated GIFs natively |
| Video playback | **AVKit** — `VideoPlayer(player: AVPlayer)` in SwiftUI |
| Pre-fetch on scroll | `ImagePrefetcher` from Nuke, hooked into `LazyVStack`'s `onAppear` |
| Memory cap | Default Nuke cache (256 MB disk, 128 MB memory) — tune after profiling |

### Android

| Concern | Choice |
|---|---|
| Image loading + caching | **Coil 3** — used via `AsyncImage` |
| GIF rendering | `coil-gif` decoder |
| Video playback | **Media3 / ExoPlayer** — wrapped in a Compose `AndroidView` |
| Pre-fetch on scroll | Coil's `ImageLoader.execute()` triggered by `LazyColumn`'s prefetch hook |
| Memory cap | Coil's default sizing — tune after profiling |

## Why these and not alternatives

- **Nuke** vs Kingfisher (iOS). Nuke has better SwiftUI integration via `NukeUI.LazyImage`, better prefetch APIs, and ships with a tiny dependency surface. Kingfisher is fine but Nuke is the modern choice.
- **Coil 3** vs Glide (Android). Coil 3 is Compose-first; Glide's Compose integration is a wrapper. Coil ships less code and has better Kotlin coroutine integration.
- **AVKit** vs MUX SDK (iOS). MUX is in use on the webapp for streaming but mobile playback works fine with AVKit + HLS URLs that MUX serves. No need for the heavier MUX iOS SDK.
- **Media3** vs older ExoPlayer artifact (Android). Media3 is Google's current packaging; ExoPlayer is now under `androidx.media3.exoplayer`.

## Video specifics

Videos in the feed:

1. Auto-play **on becoming fully visible** (≥75% of view in viewport), muted by default.
2. Pause when scrolled out of view.
3. Tap → toggle sound.
4. Tap-and-hold → enter fullscreen.
5. Single instance plays at a time (only the most-visible video plays).

| | iOS | Android |
|---|---|---|
| Visibility observer | SwiftUI `.onGeometryChange` (iOS 17+) or custom `GeometryReader` | Compose `Modifier.onGloballyPositioned` |
| Single-instance manager | `@Observable VideoPlaybackCoordinator` injected via environment | `VideoPlaybackCoordinator` provided via Hilt |
| Fullscreen | `AVPlayerViewController` modal sheet | `androidx.media3.ui.PlayerView` in a separate Activity / fullscreen Compose route |

The `VideoPlaybackCoordinator` keeps a weak reference to the currently-playing video; when a new one requests playback, it pauses the old.

## Image specifics

| Use | iOS sizing strategy | Android sizing strategy |
|---|---|---|
| Feed thumbnails | Request `width=720` from S3 / Mux | Request `width=720` |
| Avatars | Request the exact size in points × `UIScreen.scale` | Request the exact size × density |
| Hero post detail | Request `width=1080` | same |
| Inline post image | Match container width × scale | same |

URLs include `?width=N&format=webp&quality=80` query params interpreted by the image-CDN side (existing S3 + transform pipeline).

## Caching strategy

- **Memory cache**: aggressive (Nuke / Coil defaults).
- **Disk cache**: 256 MB cap.
- **Cache key**: full URL including transform params (so `width=720` and `width=1080` don't collide).
- **TTL**: respect HTTP cache-control headers from origin.

## Consequences

- The "wrong-feel" feedback users gave about lists also covers video and image playback in the current RN app. AVKit / Media3 + system scroll fix this structurally.
- Two video instances on screen simultaneously could happen during scroll — the coordinator pauses the loser.
- Pre-fetching is the single biggest perceived-quality lever; budget time for tuning the prefetch window in Phase 6.

## Maps to RN source

- `republike-mobile/src/shared/components/composed/FeedCard/FeedListItem/FeedListItem.tsx`
- `republike-mobile/src/shared/components/composed/FeedCard/FeedListItem/hooks/usePreviewLinks.ts`
- `republike-mobile/src/shared/components/composed/MediaCarousel*` — current media display
- `expo-image`, `expo-video`, `react-native-video` in `package.json` — replaced
