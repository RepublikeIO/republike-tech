# Acceptance and review

Definition of done for a PR, screenshot review process, and the minimum smoke-test bar.

## Definition of done

A PR is mergeable when ALL of the following hold:

1. **Linked to an issue.** Description says `Closes #N`.
2. **Acceptance criteria green.** Every checkbox on the linked issue is ticked.
3. **CI green.** Lint, format, tests, build — all passing.
4. **Tokens used.** No inline colors / fonts / spacing values introduced. Reviewer greps the diff for `Color(`, `Color.rgb`, `0xFF`, `\.font(.system`, `fontSize:`, hardcoded `dp` / `sp` / `pt` literals.
5. **Localized.** Any user-visible string lands in EN + FR + IT + ES via the CSV pipeline (ADR-0007).
6. **No webview.** Phase 4–7 PRs must not introduce a `WKWebView` / `WebView` for content the rewrite is meant to replace.
7. **RN reference cited.** The `Maps to RN source` PR field is filled (or marked "N/A — scaffolding").
8. **Screenshot if UI.** Light + dark mode, smallest supported device.
9. **One reviewer approval.** From a peer or the orchestrator's reviewer sub-agent.

## Screenshot review

For any PR that adds or modifies UI:

| Required | What |
|---|---|
| Yes | Screenshot of the new / changed surface, light mode |
| Yes | Screenshot in dark mode |
| Yes | Screenshot in the smallest supported device (iPhone SE 3rd gen iOS 17 / Pixel 6 Android 13) |
| Yes if RTL | Screenshot in Arabic mock locale (catches layout-only bugs) |
| Optional | Screenshot in French, Italian, Spanish — only if the string lengths differ significantly from EN |

Screenshots go directly in the PR description. Reviewers compare against the RN source's behavior (open the RN screen on a device or simulator and visually diff).

## Smoke-test bar before merge

For Phase 5+ PRs (anything user-facing), the implementer manually runs:

1. Cold-launch the app.
2. Sign in.
3. Navigate to the changed surface.
4. Perform the primary user action.
5. Background the app, foreground it — confirm state is preserved.
6. Force-quit + relaunch — confirm session persists, surface still works.

Result is one paragraph in the PR description ("Smoke: cold-launch + sign-in + opened FeedScreen + reacted to a post + backgrounded + force-quit + relaunch — feed restored at top of list").

## Test coverage expectations

| Layer | Required tests |
|---|---|
| Models / Codable / Serializable | Round-trip test per model |
| Network services | One per service: assert correct method, URL, body, response decode |
| Session manager | Sign-in, sign-out, refresh single-flight, expiry |
| Localization | Linting (every key in every locale) — automated, no manual tests |
| UI atoms | Snapshot tests per variant + state combination |
| UI molecules | Snapshot tests for the common variants |
| UI organisms | Snapshot tests for the canonical state |
| Feature viewmodels | Behavior tests (sign-in success → currentUser set; failure → error surfaced) |
| Coordinators / Dispatchers | Intent tests (open-post-via-deep-link → coordinator pushed PostDetail) |
| End-to-end flows | Manual smoke per the bar above; automated E2E deferred |

Snapshot testing tools:

- iOS: `swift-snapshot-testing` from PointFree (SPM)
- Android: `paparazzi` (Square)

## Review checklist (reviewer's own pass)

The reviewer sub-agent runs this before approving:

```
- [ ] All boxes on the linked issue are ticked
- [ ] PR description has `Closes #N`
- [ ] CI is green (no skipped jobs)
- [ ] Tokens used everywhere (no inline color / font / dimension literals)
- [ ] Screenshots present for UI changes (light + dark, smallest device)
- [ ] No webview for in-scope surfaces
- [ ] RN reference cited or "N/A"
- [ ] If a new string was added: it lives in the right CSV with EN/FR/IT/ES
- [ ] If a new model was added: it's in `data-model.md` and the file is regenerated
- [ ] If a new procedure was added: it's in `api-surface.md`
- [ ] Smoke-test paragraph in PR description (for Phase 5+ UI changes)
- [ ] No `// TODO` without a linked follow-up issue
```

A failing checkbox → request changes with a line comment naming the rule.

## Definition of done at the phase level

A phase is done when:

- All child issues closed
- Tracking issue closed
- CI on `main` green
- Brief summary commented on the tracking issue: what shipped, deviations, follow-ups
- For Phases 5–7 (user-facing): a build distributed to TestFlight Internal / Play Internal Testing for one round of human QA on real devices

The phase summary is the artifact a future engineer reads to understand what they're inheriting.
