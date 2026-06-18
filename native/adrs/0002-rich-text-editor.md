# ADR 0002 — Rich text editor for composer

**Status:** Accepted (Phase 0) — interim decision pending composer reintroduction. The plain-text composer (Option C) stands; the **content data model** is now defined by **[ADR 0012](0012-post-content-entities.md)** (plain text + entities), which supersedes this ADR's assumption of a client-side HTML feed reader.
**Date:** 2026-05-29

## Context

The current RN composer uses `@10play/tentap-editor`, a WebView-based wrapper around the TipTap editor (ProseMirror). It supports inline images, video embeds, polls, mentions, hashtags, link previews, and basic formatting (bold / italic / lists).

The composer is **deferred** in the first native delivery — auth + feed + profile do not need to compose new posts. But the architectural decision still has to be locked now so the team knows what dependencies they're committing to when composer's turn comes.

Three options:

### Option A — WebView-wrapped TipTap

iOS: `WKWebView` hosting the same TipTap setup the RN app uses. Android: `WebView` doing the same. Native↔web bridge via `WKScriptMessageHandler` / `WebView.addJavascriptInterface`.

**Pros:** lowest delta vs current behavior, fastest to ship, design and feature parity is "free."
**Cons:** it's a webview inside a native app — exactly the "not quite right" feel the rewrite is meant to escape. Defeats the rewrite's premise on the surface where users spend the second-most time after the feed.

### Option B — Native rich text per platform

iOS: TextKit 2 + `NSAttributedString` + custom attachment-renderer for inline images / videos / polls. Android: Compose `BasicTextField` with a custom `TextFieldValue` extension that supports inline content via Compose 1.7+'s `TextRange` work.

**Pros:** native feel, OS-level text selection, accessibility, IME integration. Best long-term.
**Cons:** writing a rich text editor with inline media is a 3–4 week project per platform. Maintenance is ongoing — every iOS / Android release brings text-stack changes.

### Option C — Plain text in v1 (recommended)

The native composer (when it ships) supports plain text + line breaks, mentions, hashtags, link previews, and **attached** media (image / video shown below the text, not inline). No bold / italic / inline formatting in v1.

**Pros:** ships quickly, uses native `TextField` directly, no bridge, full platform IME behavior. The "attached media" model is what most modern social apps use (X, Threads, Bluesky, Mastodon all do this).
**Cons:** loses inline-media in posts. Users who currently embed images mid-paragraph in RN composer can't reproduce that exact layout. Existing posts that contain inline media still render correctly in the feed reader (which has its own HTML→AttributedString path); only newly-composed posts in v1 are plain.

## Decision

**Option C** for the v1 composer rollout. Revisit options A/B after composer ships and we have data on user pain.

This means:

- The feed's post **reader** must still render the existing inline-rich content correctly (HTML → `AttributedString` on iOS, `AnnotatedString` on Android). That work lands in Phase 6 (feed).
- The native **composer** (Phase 8+, after first delivery) is plain text + below-text media.
- If user demand for inline composer is real, the post-v1 work is option B (native), not A. We do not introduce a webview for a feature the rewrite was supposed to eliminate.

## Consequences

- A small set of posts created during the transition will use the RN composer (still shipped); those will look exactly as they do today.
- Reader-side HTML rendering becomes a non-trivial component on both platforms — ~1 week of work each in Phase 6. This is the cost of supporting legacy content.
- We retire `@tiptap/*`, `tlds`, `linkify-it`, `react-native-pell-rich-editor`, `react-native-render-html`, `@10play/tentap-editor` from the dependency map (see `dependencies.md`).

## Alternatives considered

- **WebView only on first composer ship, native v2.** Same as A. Rejected — once shipped, the WebView lives forever because rewriting it is a months-long project users don't see.
- **Hybrid (text natively, but inline-media inserted via a sheet that returns HTML).** Considered but the bridge between native textfield + HTML-producing modal recreates much of the WebView problem in fewer lines.

## Maps to RN source

- `republike-mobile/src/Screens/Main/ContentCreation/ContentCreationScreenWithTenTap.tsx`
- `republike-mobile/src/Screens/Main/ContentCreation/ContentCreationScreenPollSurveyTenTap.tsx`
- `republike-mobile/src/Screens/Main/ContentCreation/RepostContentCreationScreenWithTenTap.tsx`
- `republike-mobile/editor-web/` — the TipTap configuration vendored into the RN app
