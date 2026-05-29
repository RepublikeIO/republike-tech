# ADR 0007 — Localization workflow

**Status:** Accepted (Phase 2)
**Date:** 2026-05-29

## Context

The native apps ship from day one in **English, French, Italian, Spanish**. Each platform has its own localization format (`.xcstrings` for iOS, `strings.xml` for Android). The webapp uses `i18next` JSON files. We need a workflow that:

1. Keeps translations consistent across platforms (a string key means the same thing everywhere).
2. Makes it easy for the team — or a translator — to add a locale's translation.
3. Catches missing translations in CI before they hit production.
4. Survives the Phase-1-7 development pace without becoming the bottleneck.

## Decision

### Authoring format — one CSV per surface, in `republike-tech/native/i18n/`

```
republike-tech/native/i18n/
├── auth.csv
├── feed.csv
├── profile.csv
├── common.csv
└── README.md
```

Each CSV has the same shape:

```csv
key,en,fr,it,es,context
auth.signIn,Sign in,Se connecter,Accedi,Iniciar sesión,Primary CTA on login screen
auth.signUp,Sign up,S'inscrire,Registrati,Registrarse,Primary CTA on landing
auth.email,Email,E-mail,Email,Correo,Form field label
```

- `key` — dot-namespaced, lowercase, identical across platforms
- `en` — source language; required
- `fr`, `it`, `es` — required; empty cell flagged by lint
- `context` — translator hint; required for new keys

### Import scripts (per platform)

Each platform has `scripts/sync-strings.mjs`. The script:

1. Reads all CSVs in `republike-tech/native/i18n/` (local checkout or fetched).
2. Emits the platform-native file:
   - iOS → `Resources/Localizable.xcstrings` (Xcode String Catalog)
   - Android → `app/src/main/res/values{,-fr,-it,-es}/strings.xml`
3. Writes a `localization-version.txt` with the source `republike-tech` commit SHA.

### CI enforcement (per platform)

CI runs three checks:

1. **Schema check** — every CSV row has all four required cells (`en`, `fr`, `it`, `es`).
2. **Regen check** — run `sync-strings.mjs`; fail if the generated platform files differ from what's committed.
3. **Usage check** — `swift-format-strings-lint` (iOS) / `lint:translation` Gradle task (Android) confirms every translation key referenced in source exists in the catalog.

### How translators work

If we hire / borrow translators:

1. They open the CSV in any spreadsheet tool.
2. They edit cells.
3. They submit the CSV back; we open a PR against `republike-tech`.
4. After merge, two regen PRs land (one per platform).

No translator needs to touch iOS / Android specifics. The CSV is the contract.

### How engineers add a new string

1. Pick the right CSV file (`auth.csv` for auth screens, `feed.csv` for feed, …).
2. Add the row with EN + FR + IT + ES + context.
3. PR against `republike-tech`.
4. After merge, regen PRs in both platform repos.

For "quick" strings during platform development:

- Engineers can add the key to the platform-native file first to unblock UI work.
- BUT the PR opening must include a TODO to backfill the CSV.
- CI eventually catches it via the regen check.

## Italian + Spanish quality bar

Initial translations are sourced from EN by either:

- A native-speaker engineer on the team
- A community contributor (vetted)
- Last resort: machine translation marked `mt:` in the context column, scheduled for human review before the next public release

Every locale variant gets a one-time native-speaker pass before each platform's v1 release. Subsequent additions land machine-then-reviewed.

## Locale picker for testing

Debug builds expose a developer screen with a Locale picker that swaps the runtime locale without restarting. Used to verify long-string truncation, RTL fallbacks, and missing-key behavior.

## Consequences

- One CSV per surface keeps PR diffs small. Adding `feed.csv` doesn't touch `auth.csv`.
- A spreadsheet edit is the canonical workflow for translators. No Lokalise / Crowdin dependency.
- Engineers pay a small tax: every UI string lands as a key, never as an inline literal.
- CI is strict; engineers catch missing translations before review.

## Maps to RN source

- `republike-mobile/assets/locales/en/common.json` etc — current i18next JSON, source for the initial seed
- `republike-mobile/src/types/i18next/resources.d.ts` — generated type definitions; our CSV approach replaces this with platform-native catalogs

## Alternatives considered

- **Lokalise / Crowdin / Phrase.** SaaS for translation management. Rejected — another vendor, another data-residency consideration, and overkill for ~4 locales at our team size.
- **JSON files mirroring i18next.** Considered but each platform needs to derive its own catalog anyway; CSV is friendlier to non-engineers.
- **Reuse the webapp's i18next files directly.** Tempting but the keying conventions and pluralization rules don't map cleanly to `.xcstrings` / `strings.xml`. Treat the CSVs as the SoT and let i18next consume them too in a future refactor.
