# ADR 0004 — Crash + error reporting

**Status:** Accepted (Phase 1)
**Date:** 2026-05-29

## Context

The current RN app has no crash reporting and no error telemetry. We've shipped with that gap and have no way to know what's breaking in production. The native rewrite must close this from day one.

The manifesto-aligned analytics post (see [`../blog/2026-05-24.md`](../../blog/2026-05-24.md)) commits us to no third-party SaaS for analytics. The same logic extends to crash reporting: we don't want raw stack traces from real users leaving our infra.

## Decision

**Self-hosted Sentry** on republike-infra. Used for:

- Native crash reports (NSException / Mach exceptions on iOS, JVM crashes and ANRs on Android)
- Uncaught errors from the API client and viewmodels
- Performance traces (cold-launch time, slow API calls — selectively, sampled)

**Not** used for:

- Funnel events — those are server-side aggregate counters, no client SDK
- Per-user behavior tracking — Sentry isn't an analytics tool here
- Any PII — strict PII scrubbing on, IPs and emails redacted before send

## Hosting plan

Sentry self-hosted Docker stack on a new VPS in OVH alongside rp-redis / rp-main / rp-dev:

```
rp-sentry  (new VPS — 4 vCPU / 8 GB / 100 GB)
  ├ Postgres (Sentry's own)
  ├ Redis
  ├ Sentry web + worker + cron
  └ Caddy: sentry.republike.io → :9000
```

Provisioning is tracked outside the native-rewrite issues — it's an infra ticket. Until rp-sentry is up, both native apps initialise Sentry with an empty DSN (no-op, no network calls). Once it's up, the DSN lands in the per-env xcconfig / BuildConfig.

## SDK choices

| | iOS | Android |
|---|---|---|
| Package | `sentry-cocoa` (SPM) | `io.sentry:sentry-android` (Gradle) |
| Min version | latest stable at time of integration | latest stable at time of integration |
| Init | `SentrySDK.start { options in … }` in App init | `SentryAndroid.init(context) { options.dsn = … }` in `Application.onCreate` |
| Symbol upload | dSYM via fastlane / GH Action | Mapping file via Sentry Gradle plugin |
| PII scrubbing | `options.sendDefaultPii = false`; manual scrub for any custom context | same |
| Sample rate | `tracesSampleRate = 0.1` | same |

## Configuration

Per environment:

| Variable | Debug | Staging | Production |
|---|---|---|---|
| `SENTRY_DSN` | empty (no-op SDK) | self-hosted DSN, staging project | self-hosted DSN, prod project |
| `SENTRY_ENVIRONMENT` | "debug" | "staging" | "production" |
| `SENTRY_RELEASE` | bundle version + commit SHA | same | same |

Two Sentry projects on rp-sentry: `republike-ios-staging` + `republike-ios-prod` (and the Android pair). Staging errors do not pollute prod alerts.

## Manual scrubbing rules

Before any event is sent:

- `request.cookies` → removed
- `extra.email`, `extra.username` → removed
- `breadcrumb.message` strings matching `email`, `password`, `token` → redacted
- IP address → not sent (`options.sendDefaultPii = false` covers this, but the rule is documented)

A helper `SentryScrub.swift` / `SentryScrub.kt` lives in `Core/Telemetry/`. All `Sentry.capture*` calls route through that helper.

## Verification

Each platform's `#6` issue closes with a manual verification step:

1. Force an uncaught error (`fatalError` / `error("explosion")`).
2. Confirm the report arrives on rp-sentry within 60 seconds.
3. Confirm no email / token / IP shows up in the captured event.

## Consequences

- One more piece of OVH infra to maintain (rp-sentry). Routine but real.
- Sentry self-hosted upgrades land on us. We pick a quarterly rhythm.
- Engineers get crash reports in a dashboard that's ours — no data leaves the platform.

## Alternatives considered

- **Sentry SaaS.** Cheapest in engineering effort but violates the data-residency posture. Rejected.
- **Crashlytics.** Free, ubiquitous, but ties us to Firebase / Google. Rejected for the same reason.
- **Bugsnag self-hosted (BugSnag Enterprise).** Pricier license, smaller community than Sentry. Rejected.
- **Sentry SaaS with PII-scrubbing on.** Considered as a transitional step. Rejected — the moment scrubbing fails, real user data is in someone else's database.
