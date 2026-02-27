<p align="center">
  <img src="assets/logo.png" alt="Republike" width="120" />
</p>

<h1 align="center">Republike — Technical Overview</h1>

<p align="center">
  A community-driven social platform where citizens shape the rules, moderate content, and earn reputation through meaningful engagement.
</p>

<p align="center">
  <a href="https://www.republike.io">Website</a> · <a href="#weekly-devlog">Devlog</a> · <a href="https://t.me/republike">Telegram</a>
</p>

---

## What is Republike?

Republike is a social platform built on the idea that users — not algorithms — should govern their online community. Content moderation is handled by community voting, not opaque corporate policies. A reputation system tracks meaningful contributions across five pillars, and weekly credit rewards go to the most engaged citizens.

The platform runs across **web**, **mobile** (iOS & Android), and an **admin panel** for operational management. All three clients share a single backend powered by tRPC for end-to-end type safety.

---

## Repositories

The codebase is split into three active repositories. Each serves a distinct role in the platform.

### 1. `republike-webapp`

> The core of the platform — web frontend and full backend.

| | |
|---|---|
| **Stack** | Next.js 13.4 · React 18 · TypeScript · Tailwind CSS |
| **Backend** | tRPC · Prisma (PostgreSQL) · Inngest (background jobs) |
| **Auth** | NextAuth.js (sessions) + JWT (mobile fallback) |
| **Payments** | Stripe · Apple IAP · Google Play Billing |
| **Search** | Algolia |
| **Media** | AWS S3 (images) · Mux (video) |
| **Email** | SendGrid |

#### What it does

The webapp is both the **web client** and the **API server** for the entire platform. Every tRPC procedure that the mobile app and admin panel call lives here.

**Web client features:**
- Public landing pages (bilingual FR/EN) with plans, manifesto, FAQ, team
- Full social feed with multiple modes (latest, best, followed, news)
- Rich content editor (Tiptap) — posts, comments, polls, images, video, embeds
- Community moderation interface — threshold-based visibility voting
- User profiles with reputation scores, followers, post history
- Search (Algolia-powered) across posts, users, hashtags
- Governance proposals and community voting
- Subscription management (Stripe checkout)
- Referral system with invite codes
- Notification center

**Backend features:**
- ~150 tRPC procedures across 20 domain routers
- 63 service files handling business logic
- 21 Inngest background jobs (weekly epoch cycle, notifications, moderation, search indexing)
- 35 REST admin endpoints consumed by the admin panel
- Webhook handlers for Stripe, App Store, Google Play, and Mux
- Reputation calculation engine (5 pillars, weekly epochs)
- Credit distribution system (2,000 credits/week to top 1,000 users)

**Auth model — 6 levels:**

| Procedure | Access level |
|---|---|
| `publicProcedure` | Anyone |
| `protectedProcedure` | Authenticated user (any status) |
| `activeOrMissingProfileProcedure` | Authenticated + onboarding allowed |
| `activeProcedure` | Authenticated + fully active |
| `apiKeyProcedure` | Valid API key |
| `apiKeyAdminProcedure` | Valid API key + admin flag |

---

### 2. `republike-mobile`

> The native mobile app — iOS and Android.

| | |
|---|---|
| **Stack** | React Native 0.76 · Expo 52 · TypeScript · NativeWind (Tailwind) |
| **Navigation** | React Navigation 7 (stack + bottom tabs + material top tabs) |
| **State** | Zustand (SQLite persistence) · React Query |
| **API** | tRPC client → webapp backend |
| **Editor** | TenTap (custom Tiptap fork for React Native) |
| **Payments** | react-native-iap (StoreKit + Google Play Billing) |
| **Push** | Firebase Cloud Messaging |
| **Analytics** | Firebase Analytics · Crashlytics |
| **Search** | Algolia (react-instantsearch) |
| **Build** | EAS Build (development / preview / production channels) |

#### What it does

The mobile app is a full-featured native client that mirrors the web experience with platform-specific optimizations.

**Core flows:**
- Onboarding: landing → register → verify email → create profile → pick topics → swear oath → paywall
- 5-tab navigation: Home Feed (follow/latest/hot) · Search · Create Post · Notifications · Profile
- Rich content creation with mentions, hashtags, images, video, polls
- Community moderation voting
- In-app purchases (founding citizen subscription, founding father lifetime)
- Deep linking (`republike://p/:postId`, `republike://u/:username`, etc.)
- OTA updates via EAS Update (deploy without app store review)
- Push notifications via FCM
- Offline-capable with SQLite-backed Zustand stores

---

### 3. `republike-admin`

> Internal admin panel for platform operations.

| | |
|---|---|
| **Stack** | Next.js 13.5 · React 18 · TypeScript · Tailwind CSS · shadcn/ui |
| **State** | Zustand (localStorage persistence) · React Query |
| **Auth** | API key → webapp admin endpoints |
| **Export** | XLSX (CSV/Excel export) |

#### What it does

A dashboard for the team to manage the platform without touching code or the database directly. Authenticates via API key against the webapp's admin REST endpoints.

**Capabilities:**
- **Dashboard** — live stats (posts, comments, reactions, votes) with cache controls
- **User management** — founder status, ambassador badges, certification levels, account deletion
- **Content management** — browse/search posts, edit topics, change visibility, set featured posts
- **Governance** — create voting proposals
- **Topic management** — create, edit, remove topics
- **Founder codes** — generate and track claim codes with status filters and CSV export
- **News** — publish announcements
- **Translations** — manage bilingual content (FR/EN)
- **AURE distribution** — view token distribution history by type and period, CSV export
- **Epoch rewards** — search rewards by epoch, view eligibility, export data
- **Settings** — bilingual pinned messages (Markdown)
- **Environment switcher** — switch between dev / staging / production at runtime

---

## System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Republike Platform                     │
├──────────────┬──────────────────┬────────────────────────┤
│  Web Client  │   Mobile App     │    Admin Panel          │
│  (Next.js)   │   (React Native) │    (Next.js)            │
│  SSR + SPA   │   iOS + Android  │    SPA on port 3001     │
├──────────────┴──────────────────┴────────────────────────┤
│                     tRPC (type-safe)      REST /api/admin │
├──────────────────────────────────────────────────────────┤
│                      Service Layer                        │
│            63 services · business logic                   │
├──────────────────────────────────────────────────────────┤
│  Prisma ORM          │ Inngest (21 jobs)                  │
│  PostgreSQL           │ Cron + event-driven                │
├───────────────────────┴──────────────────────────────────┤
│  Algolia  │  Stripe  │  AWS S3  │  Mux  │  SendGrid     │
│  search   │  payments│  media   │  video │  email        │
└───────────┴──────────┴──────────┴───────┴────────────────┘
```

---

## Reputation System

Citizens earn reputation across **five pillars**, scored weekly during each **epoch** (1-week cycle).

| Pillar | Description | Status |
|---|---|---|
| **Quality** | Score from reactions received on your posts. Positive reactions (Smart +3, Useful +3, Inspiring +1) boost it; negative ones (Deceptive -5, Aggressive -5) lower it. Normalized via `tanh(raw/30) × 100`. | Live |
| **Wellbeing** | Activity diversity during the epoch. Measures 5 activity types (reactions given, moderation votes, posts, comments, poll votes). 3+ types = 100, 2 = 80, 1 = 50. | Live |
| **Altruism** | Generosity through tips given. 3+ tips = 100, 2 = 80, 1 = 50. | Live |
| **Respect** | Respectful behavior towards the community. | In development |
| **Competence** | Expertise and contribution value. | In development |

**Global score** = average of all 5 pillars over the last 12 epochs, displayed on a 0–100 scale.

### Weekly Credit Distribution

Every epoch, **2,000 credits** are distributed to the **top 1,000 users** with a positive reputation score:

1. Each eligible user gets a guaranteed **1 credit**
2. Remaining credits distributed proportionally using **√(score)** weighting
3. Residual credits go to highest scorers

---

## Weekly Devlog

Every Friday we publish what shipped, what's in progress, and what's next.

| Date | Title |
|---|---|
| *Coming soon* | [First edition →](blog/) |

Browse all entries in the [`blog/`](blog/) directory.

---

## Links

- [Republike](https://www.republike.io) — the platform
- [Telegram](https://t.me/republike) — community chat
- [App Store](https://apps.apple.com/app/republike/id6499094498) — iOS app
- [Google Play](https://play.google.com/store/apps/details?id=com.republike.app) — Android app
