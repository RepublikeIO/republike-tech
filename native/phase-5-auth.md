# Phase 5 — Auth surface

The first user-facing native delivery. Replaces the placeholder landing / sign-in / sign-up / onboarding / paywall with real screens. Builds entirely on Phase 4 atoms / molecules / organisms — no inline styling, no inline strings.

## Scope

Every screen labelled "first-delivery, phase 5" in [`screens.md`](./screens.md):

1. Splash
2. Landing (logo + "Sign in" / "Sign up")
3. Login
4. Register
5. Email verification (code entry)
6. Verifying (post-verification loading + success animation)
7. Forgot password
8. Reset password
9. Interest selection (multi-select topics)
10. Pre-principle intro
11. Pre-principle 1, Pre-principle 2 (manifesto excerpts the user acknowledges)
12. Moderation explanation
13. Profile completion (avatar, first name, last name, bio)
14. Add-friend prompt (post-onboarding)
15. Paywall — plan list
16. Paywall — verify payment
17. Paywall — subscription ended

## Flow graph

```
Splash
  ├─→ (auth session valid) → Tab bar Feed
  └─→ Landing
         ├─→ Login → Tab bar
         │     └─(forgot password)→ Forgot password → Reset password → Login
         └─→ Register
                ├─→ Email verification
                │      └─→ Verifying
                │             ├─→ (founder code) → Tab bar
                │             └─→ Onboarding
                │                    ├─→ Interest selection
                │                    ├─→ Pre-principle intro
                │                    ├─→ Pre-principle 1
                │                    ├─→ Pre-principle 2
                │                    ├─→ Moderation explanation
                │                    ├─→ Profile completion
                │                    ├─→ Add-friend prompt
                │                    └─→ Paywall (plan list)
                │                           ├─→ (Free) → activateFreeTier → Tab bar
                │                           └─→ (Paid) → StoreKit/Play Billing → Verify → Tab bar
                └─→ Subscription ended (entry point: USER_INACTIVE interceptor)
                       └─→ Paywall (plan list) → (same as above)
```

## Per-screen breakdown

For each screen below: components used + business-logic ref in RN + key acceptance.

### Splash

Components: `RBrandIcon` (logo), `Spinner`, template `AuthScreen` with no CTAs.
RN ref: `republike-mobile/src/Screens/Auth/SplashScreen.tsx`.
Acceptance: visible for ≤1s under good network, transitions to either tab bar or Landing based on session state.

### Landing

Components: `RBrandIcon` (logo), `Label.display`, `Label.body`, two `RButton` (primary "Sign in", secondary "Sign up").
RN ref: `republike-mobile/src/Screens/Auth/LandingScreen.tsx`.
Acceptance: keyboard-shortcut Enter does nothing (Landing has no field).

### Login

Components: `FormField` × 2 (email, password), `RButton` primary, `Label.body` "Forgot password?" tap-target.
RN ref: `republike-mobile/src/Screens/Auth/LoginScreen.tsx`.
Calls: `auth.signIn(email:password:)` (Phase 2 #9).
Acceptance:
- Email validated on blur (format).
- Password validated on submit (server returns error → mapped to FR/IT/ES error string).
- Successful sign-in routes to tab bar via coordinator.
- "Forgot password?" → Forgot password screen.

### Register

Components: `FormField` × 4 (firstName, lastName, email, password), `Label.body` (link to ToU + Privacy), `RButton` primary "Continue".
RN ref: `republike-mobile/src/Screens/Auth/RegisterScreen.tsx` (note the progressive-validation pattern in the RN file — keep that behavior).
Calls: `auth.signUp(input:)`.
Acceptance:
- Progressive validation: each field validates on blur, the button enables when all 4 are valid.
- Password rules visible (8+ chars, 1 uppercase, 1 number, 1 symbol) and update in real-time.
- ToU + Privacy links open in a `WKWebView` / Chrome Custom Tab (these external policy docs are the only webview exception in the app).

### Email verification

Components: `Label.headline` "Check your email", `Label.body` instruction, a 6-cell code-entry field (custom atom: `RCodeField` — adds during Phase 5 if not present from Phase 4), `RButton` primary "Verify", `Label.body` "Resend code" tap-target.
RN ref: `republike-mobile/src/Screens/Auth/EmailVerificationScreen.tsx`.
Calls: `auth.verifyEmail(code:)`, `auth.resendVerificationCode()`.
Acceptance:
- Code auto-fills from SMS / email OTP suggestion (iOS one-time-code, Android autofill).
- 6 digits typed → auto-submit.
- Resend disabled for 30s after first send.

### Verifying

Animated transition screen: spinner → checkmark.
RN ref: `republike-mobile/src/Screens/Auth/VerifyingScreen.tsx`.
Acceptance: animation runs at 60fps on the smallest supported device.

### Forgot password

Single `FormField` (email), `RButton` primary "Send reset email".
RN ref: `republike-mobile/src/Screens/Auth/ForgotPasswordScreen.tsx`.
Calls: `auth.requestPasswordReset(email:)`.
Acceptance: regardless of whether the email exists, show the same success message ("If that email exists, we've sent a reset link") — server contract.

### Reset password

Entered via deep link from email — URL contains a token. `FormField` × 2 (new password + confirm), `RButton` primary "Reset".
RN ref: `republike-mobile/src/Screens/Auth/ResetPasswordScreen.tsx`.
Calls: `auth.confirmPasswordReset(token:password:)`.
Acceptance: token expiry surfaces a clear "Link expired" state with a "Request a new link" button.

### Interest selection

Multi-select grid of topic chips. User picks ≥3, then "Continue".
RN ref: `republike-mobile/src/Screens/Auth/InterestScreen.tsx`.
Calls: `topic.list()`, `topic.followTopic(id:)` per selection.
Components: `Label.title`, custom `TopicChip` atom (added in Phase 5 if not in Phase 4), `RButton` primary, secondary "Skip".

### Pre-principle screens

Three screens showing manifesto excerpts the user acknowledges by tapping "I understand".
RN ref: `republike-mobile/src/Screens/Auth/PrePrincipleScreen.tsx`, `PrePrinciple1Screen.tsx`, `PrePrinciple2Screen.tsx`.
Components: `Label.title`, `Label.bodyL`, `RButton` primary.

### Moderation explanation

Same shape as pre-principle. Explains the moderation system + vote responsibility.
RN ref: `republike-mobile/src/Screens/Auth/ExplanationModerationScreen.tsx`.

### Profile completion

Components: `Avatar` (xl, tap to pick image via PhotosUI / Photo Picker), `FormField` × 3 (firstName, lastName, bio), `RButton` primary.
RN ref: `republike-mobile/src/Screens/Auth/ProfileCompletionScreen.tsx`.
Calls: `user.updateProfile(input:)`.
Acceptance:
- Avatar tap opens system photo picker → uploads via API (Phase 5 also lands `media.uploadImage(file:)` — verify with infra ticket).
- Bio character counter visible (max 280).

### Add-friend prompt

Suggests following a curated set of accounts.
RN ref: `republike-mobile/src/Screens/Auth/PromptToAddFriendScreen.tsx`.
Calls: `user.searchUsers` (or a fixed curated list endpoint TBD).
Components: `ListItemRow` molecule, `RButton` "Follow" inline per row, `RButton` primary "Continue" at bottom.

### Paywall — plan list

Carousel of `PaywallCard` organisms. Tap → StoreKit 2 / Play Billing per ADR-0008.
RN ref: `republike-mobile/src/Screens/Auth/Paywall/PlanScreen.tsx`.
Calls: `subscription.getMyCurrentPlan()`, platform billing APIs, `subscription.verifyApplePurchase` or `verifyAndroidPurchase`.
Components: `PaywallCard`, `RButton`, `Label.title`.
Acceptance:
- Free Visiting Citizen card prominent first.
- StoreKit 2 / Play Billing flow respects platform UI (no custom payment sheet).
- Receipt verification round-trip handled with a loading state.

### Paywall — verify payment

Loading + status screen after purchase.
RN ref: `republike-mobile/src/Screens/Auth/Paywall/VerifyPaymentScreen.tsx`.

### Paywall — subscription ended

Entry point when USER_INACTIVE interceptor fires.
RN ref: `republike-mobile/src/Screens/Auth/Paywall/SubscriptionEndedScreen.tsx`.

## Issues (~17 per platform — opened after Phase 4 closes)

- 1 issue per screen × 17 screens = 17 (some can pair, e.g. Splash + Landing can share a PR)
- 1 issue for the deep-link handler hookups (reset password, email verify)
- 1 issue for StoreKit 2 / Play Billing integration (large; could be 2-3)
- 1 issue for analytics event hooks (server-side funnel counters from the analytics blog post)
- 1 tracking issue

Total: ~20 issues per platform.

## Acceptance for Phase 5

- A fresh install can: open → sign up → verify email → onboard → land on the tab bar with a real session and an active plan.
- A returning user: open → cold session restore → tab bar.
- All screens use only design system atoms / molecules / organisms — no inline color, font, or spacing literals.
- Every string is in EN, FR, IT, ES.
- IAP receipt round-trips work in TestFlight Internal / Play Internal Testing on real devices.
- USER_INACTIVE → Paywall → reactivation → tab bar works end-to-end.

## Maps to RN source

- `republike-mobile/src/Screens/Auth/*` — every screen above
- `republike-mobile/src/Services/iap/*` — IAP plumbing reference
- `republike-mobile/src/server/api/routers/auth.ts` — webapp side, contract reference
