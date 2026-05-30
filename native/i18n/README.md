# Native i18n source of truth

This directory holds the **canonical localization strings** for the native apps,
per [ADR 0007 — Localization workflow](../adrs/0007-localization-workflow.md).
The CSVs here are the single source of truth (SoT); the platform repos generate
their native catalogs from them. Do not edit the platform-native files by hand.

## CSV-per-surface

One CSV per UI surface, so PR diffs stay small and unrelated surfaces don't churn:

| File | Surface |
| --- | --- |
| `common.csv` | Shared UI (buttons, generic errors) |
| `auth.csv` | Sign-in, sign-up, verification, password reset |

`feed.csv` and `profile.csv` will be added in later phases as those surfaces land.

## Schema

Every CSV has the same header row, exactly:

```csv
key,en,fr,it,es,context
```

- `key` — dot-namespaced, lowercase, identical across platforms (e.g. `auth.login`).
- `en` — source language; canonical. Required.
- `fr`, `it`, `es` — required; an empty cell fails the CI schema check.
- `context` — short translator hint; required for every key.

Quote any cell that contains a comma.

## Authoring

- **Engineers:** pick the right surface CSV, add a row with EN + FR + IT + ES +
  context, and open a PR against `republike-tech`. After merge, regen PRs land in
  each platform repo.
- **Translators:** open the CSV in any spreadsheet tool, edit cells, send it back;
  we open the PR. No one needs to touch iOS/Android specifics — the CSV is the contract.

## Consumers — `sync-strings.mjs`

Each platform ships `scripts/sync-strings.mjs`, which reads every CSV in this
directory and emits the platform-native catalog:

- **iOS** → `Resources/Localizable.xcstrings` (Xcode String Catalog)
- **Android** → `app/src/main/res/values{,-fr,-it,-es}/strings.xml`

It also writes a `localization-version.txt` recording the source `republike-tech`
commit SHA the catalog was generated from.

## CI checks (per platform)

1. **Schema check** — every row has all four required cells (`en`, `fr`, `it`, `es`).
2. **Regen check** — run `sync-strings.mjs`; fail if the generated native files
   differ from what's committed.
3. **Usage check** — every translation key referenced in source exists in the
   catalog (`swift-format-strings-lint` on iOS, `lint:translation` Gradle task on Android).

## The `mt:` convention

`it` and `es` have **no native-speaker review yet** — they are machine-translated.
Most `fr` cells are sourced from the reviewed RN file
(`republike-mobile/assets/locales/fr/common.json`); the few `fr` cells that had no
matching reviewed string were machine-translated.

Any `fr`/`it`/`es` value that is machine-translated is flagged by prefixing its
`context` cell with `mt:`. These rows are scheduled for human review before the next
public release.

## Native-speaker review before v1

Every locale variant gets a one-time native-speaker pass before each platform's v1
release. After that, new additions land machine-then-reviewed.
