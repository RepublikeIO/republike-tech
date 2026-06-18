# ADR 0012 — Post content model: plain text + entities (replacing HTML)

**Status:** Accepted — completes/realizes ADR 0002; supersedes the *HTML feed-reader* assumption in ADR 0002
**Date:** 2026-06-18

## Context

Posts today are authored and stored as **TipTap/ProseMirror HTML**, served as HTML, and parsed on each client into styled text (an `HTML → AttributedString` reader). This is the Mastodon-style minority approach; **X, Bluesky, Threads use plain text + structured entities ("facets")** instead. The HTML path is a recurring source of client rendering bugs (inline-`<img>` placeholder artifacts, paragraph spacing, general tag-soup fragility) and forces every client to carry an HTML parser.

ADR 0002 already chose a **plain-text composer** (its Option C). This ADR finishes that decision by making the **content data model itself** plain text + entities end-to-end, so clients never parse HTML — for *any* post.

**Hard requirement (non-negotiable): legacy posts must remain fully supported at all times.** The existing HTML post corpus, and the live `republike-webapp` + RN apps that still produce HTML, must keep working throughout and after the migration.

## Decision

1. **Canonical post content = `{ text: string, entities: Entity[] }`.** No HTML in the client-facing contract.
2. **Entity** = a typed annotation over a substring range:
   ```jsonc
   { "type": "mention" | "hashtag" | "link",
     "start": <int>, "end": <int>,        // see indexing rule
     // payload by type:
     "userId": "...", "handle": "...",     // mention
     "tag": "...",                          // hashtag
     "url": "..." }                         // link
   ```
   **Media is NOT a text entity** — images/video remain a separate `attachments`/`videoUrl` field (the "attached media" model from ADR 0002 Option C, as X/Threads do). Entities annotate text only.
3. **Ranges are UTF‑8 byte offsets, half-open `[start, end)`.** This is an invariant every client + the server must honor. Rationale: unambiguous and emoji/surrogate-safe across JS, Swift, and Kotlin (Bluesky's proven choice). Clients convert at the edge — Swift `String.utf8`, Kotlin `toByteArray(UTF_8)`, JS `TextEncoder`/`Buffer`. (UTF‑16 code units were rejected: no conversion needed but surrogate pairs are a foot-gun.)
4. **Mentions** are bound to a `userId` at authoring time: the composer's `@` autocomplete (`search.searchUsers`) records the chosen user's id on the entity, so resolution never depends on a fragile late `@handle` lookup. The server validates the userIds, writes the post↔mention relations, and fires notifications.
5. **Hashtags + links** are derivable from `text` by the server (regex) and may also be supplied by the client; the server remains the source of truth for hashtag relations/search.
6. **Composer** (per ADR 0002 Option C): a native plain-text field — `@` autocomplete, `#`/URL auto-detection, attached media, no rich-text/bold/italic, no HTML. It emits `{ text, entities }`.
7. **Rendering:** clients render `text` verbatim and overlay styling on entity ranges (mention/hashtag/link → primary color, tappable). The `HTML → AttributedString` reader is retired once all clients consume entities (kept only behind the backend's legacy adapter — see below).

## Legacy support — permanent and first-class

The migration must never drop legacy posts, so legacy support is built as a **permanent normalization in the serving layer**, not a throwaway bridge:

- **The API always returns `{ text, entities }` for every post.** New posts store it natively; legacy HTML posts are converted by a **read-time adapter that is a permanent part of the read path.** Clients *never* see HTML — HTML becomes an internal storage detail the read layer normalizes away.
- The adapter already has everything it needs (no data is missing):
  | New field | Source on a legacy post |
  |---|---|
  | `text` | the **already-stored `plainTextContent`** (computed via `htmlToText`, already returned in the current feed) |
  | mention entities | existing **post↔mention relations** + the HTML `<span data-type="mention" data-id="…">` (userId) → ranges located in `text` |
  | hashtag entities | existing **hashtag relations** + `#tag` positions in `text` |
  | link entities | `<a href>` parsed once → ranges in `text` |
- **Strategy:** (a) on-read conversion is always available — this is the invariant that guarantees "legacy supported all the time"; (b) an optional one-time **backfill + cache** is purely a performance optimization. Even after a full backfill, the on-read adapter is **retained indefinitely**, so any residual or newly-written HTML row (e.g. from web/RN during transition) is always serveable as entities.
- **Invariant:** *any* post — old HTML, or new native, or future web/RN — resolves to one uniform `{ text, entities }` shape for clients.

## Migration (phased, additive, protects the live web + RN)

1. **Backend read path** — serve `{ text, entities }` for all posts (legacy via the permanent adapter). Additive: web/RN keep reading their `content`/HTML untouched.
2. **Native clients** — consume `{ text, entities }` (drop the HTML wire-mapper + the HTML renderer from the active path), and build the plain-text composer that *writes* `{ text, entities }`.
3. **Backend write path** — accept `{ text, entities }` from native; continue accepting HTML from web/RN.
4. **Later** — web/RN migrate to entities at their own pace; optional backfill. **The legacy read adapter stays forever.**

No step breaks a shipped app.

## Consequences

- **+** Simpler clients (no HTML parser), the whole class of HTML-parse rendering bugs disappears, X/Bluesky parity, smaller payloads.
- **+** Legacy is fully and *permanently* supported; clients are uniform.
- **−** Real backend work in the **shared `republike-webapp`** + cross-repo coordination; must not break the live web + RN apps (mitigated by the additive, phased plan).
- **−** The `{ text, entities }` JSON schema (and the **UTF‑8 byte-offset invariant**) must be documented in `api-surface.md` and honored identically by every client.

## Alternatives considered

- **Keep HTML end-to-end** — rejected: the parse-bug class, minority approach, heavier payloads, every client carries a parser.
- **UTF‑16 code-unit offsets** — rejected: no conversion needed, but emoji/surrogate splitting is a foot-gun.
- **Dual client rendering (HTML for legacy, entities for new)** — rejected: keeps the HTML complexity we're removing on every client. Instead the **backend** normalizes legacy → entities so clients stay uniform.

## Relationships

- **ADR 0002 (rich-text editor):** its plain-text composer decision (Option C) stands and is realized here. This ADR supersedes 0002's assumption that the feed keeps a client-side HTML reader — that path is replaced by the entity model + backend legacy adapter.
- **`api-surface.md`:** must add the `{ text, entities }` post contract + the `Entity` schema + the byte-offset rule; `post.getBatch` / `getUserPosts` / `getPostWithRepliesForMobile` / `createPost` / `editPost` adopt it.
- **ADR 0009 (media stack):** unchanged — media stays attached (`videoUrl`/attachments), not a text entity.
