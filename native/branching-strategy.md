# Branching strategy

Identical rules in `republike-ios` and `republike-android`. Designed to keep `main` always shippable and to make the phase tracking issues mean something.

## Branches

| Branch | Purpose | Push allowed | Source of truth for |
|---|---|---|---|
| `main` | Latest reviewed-and-merged work. CI green. Always shippable. | NO (PR merge only) | TestFlight / Play Internal builds |
| `release/<version>` | Release prep — bug fixes only, no new features | PR only | App Store / Play Store builds |
| `<type>/<issue-num>-<slug>` | Working branches | YES (force push allowed) | the in-flight issue |

`<type>` is one of: `scaffold`, `feature`, `fix`, `chore`, `ci`, `docs`, `adr`.

## Workflow

```
issue #N opened
   ↓
implementer creates  feature/N-slug   from main
   ↓
work, push, push, push
   ↓
PR opened — base: main — title carries the issue + scope
   ↓
CI green
   ↓
reviewer approves
   ↓
PR merged  (--merge, NOT squash)
   ↓
issue closes via "Closes #N" in PR description
   ↓
tracking issue (#7 / #13 / etc.) gets checkbox ticked
```

### Why merge commits, not squash

- Each task is small and atomic; the per-commit history within a feature branch is useful in `git log` later.
- Squash loses the implementer's intent breakdown (e.g. "1. scaffold targets, 2. wire CI, 3. fix lint").
- Merge commits also mean `tracking issue#N` shows up cleanly in `git log --first-parent`.

If a feature branch has 30 noisy commits, the implementer rebases / squashes BEFORE opening the PR — the merge is then clean.

## Release branches

Created when preparing a TestFlight / Play submission:

```
git checkout main
git checkout -b release/0.5.0
# Bump version in Tuist manifests / Gradle, freeze the branch
```

From this point:

- Only `fix/*` branches land on `release/X.Y.Z` (via PR).
- Every fix lands BOTH on the release branch and on `main` — keep them in sync via cherry-pick or "merge back."
- When the release is approved by the store, tag `v0.5.0` on the release branch, then merge release back to main.

## Protected branches

| Branch | Protection |
|---|---|
| `main` | required PR, required CI, required reviewer, no force push, no deletion |
| `release/*` | same |

GitHub Settings → Branches → add rules. The orchestrator does NOT change these — branch protection adjustments need human approval.

## Naming examples

Good:

```
scaffold/1-tuist-skeleton
scaffold/4-token-codegen
network/8-http-foundation
session/11-keychain-storage
fix/auth-password-too-short-toast
docs/branching-strategy
adr/0012-something
```

Bad:

```
fix-stuff
phase4
xcode
feature-feed
```

The issue number in the branch name is the contract: a future reviewer sees the branch and knows exactly which issue it implements.

## Commits

Conventional commit prefixes, lowercase scope:

```
scaffold: bootstrap Tuist with Core/Features/UI targets
ci: add SwiftLint pre-build phase
fix(feed): guard against null user in PostCard header
docs: link Tuist install steps in README
```

The implementer's commits live inside the feature branch. The merge commit subject is the PR title.

## Tracking issues update

Per phase, the tracking issue body has a checklist of children. When a child PR merges:

- The implementer (or orchestrator on behalf of) edits the tracking issue and checks the box.
- When all boxes are ticked, the tracking issue closes.
- Phase summary is added as a final comment on the tracking issue: "Closed: <date>. Notes: ..."

## Hot fix to prod

If a critical bug ships:

1. Branch `fix/<short>` from the active `release/X.Y.Z`.
2. Land the fix on that release branch via PR.
3. Cherry-pick the same commit to `main`.
4. Re-tag and re-submit.

Do not branch hotfixes from `main` — `main` may have features that are not in the current store release.
