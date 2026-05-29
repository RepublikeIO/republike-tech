# Screen inventory

Every screen in the current `republike-mobile` app, labelled by first-delivery scope. Used to size phases and to write issues with the right `Maps to RN source` references.

| Screen | RN source path | Scope | Phase | Notes |
|---|---|---|---|---|
| Splash | `src/Screens/Auth/SplashScreen.tsx` | first-delivery | 5 | thin |
| Landing | `src/Screens/Auth/LandingScreen.tsx` | first-delivery | 5 | logo + CTA |
| Login | `src/Screens/Auth/LoginScreen.tsx` | first-delivery | 5 | |
| Register | `src/Screens/Auth/RegisterScreen.tsx` | first-delivery | 5 | progressive validation |
| Email verification | `src/Screens/Auth/EmailVerificationScreen.tsx` | first-delivery | 5 | code field component |
| Verifying | `src/Screens/Auth/VerifyingScreen.tsx` | first-delivery | 5 | post-verification animation |
| Forgot password | `src/Screens/Auth/ForgotPasswordScreen.tsx` | first-delivery | 5 | |
| Reset password | `src/Screens/Auth/ResetPasswordScreen.tsx` | first-delivery | 5 | |
| Interest selection | `src/Screens/Auth/InterestScreen.tsx` | first-delivery | 5 | onboarding |
| Pre-principle (intro) | `src/Screens/Auth/PrePrincipleScreen.tsx` | first-delivery | 5 | |
| Pre-principle 1 | `src/Screens/Auth/PrePrinciple1Screen.tsx` | first-delivery | 5 | |
| Pre-principle 2 | `src/Screens/Auth/PrePrinciple2Screen.tsx` | first-delivery | 5 | |
| Moderation explanation | `src/Screens/Auth/ExplanationModerationScreen.tsx` | first-delivery | 5 | |
| Profile completion | `src/Screens/Auth/ProfileCompletionScreen.tsx` | first-delivery | 5 | |
| Add friend prompt | `src/Screens/Auth/PromptToAddFriendScreen.tsx` | first-delivery | 5 | |
| Paywall — plan list | `src/Screens/Auth/Paywall/PlanScreen.tsx` | first-delivery | 5 | StoreKit 2 / Play Billing |
| Paywall — verify payment | `src/Screens/Auth/Paywall/VerifyPaymentScreen.tsx` | first-delivery | 5 | |
| Paywall — subscription ended | `src/Screens/Auth/Paywall/SubscriptionEndedScreen.tsx` | first-delivery | 5 | |
| Home feed (new) | `src/Screens/Main/HomeFeed/NewHomeFeedScreen.tsx` | first-delivery | 6 | Agora + For You tabs |
| Home feed (legacy) | `src/Screens/Main/HomeFeed/HomeFeedScreen.tsx` | dropped | — | replaced by NewHomeFeedScreen |
| Post detail | `src/Screens/Main/Post/PostDetailScreen.tsx` | first-delivery | 6 | reactions, comments |
| Reaction stats sheet | `src/shared/components/composed/ReactionStatsBottomSheet.tsx` | first-delivery | 6 | |
| Profile (own) | `src/Screens/Main/Profile/MyProfileScreen.tsx` | first-delivery | 7 | |
| Profile (other user) | `src/Screens/Main/Profile/OtherProfileScreen.tsx` | first-delivery | 7 | |
| Edit profile | `src/Screens/Main/Profile/EditProfileScreen.tsx` | first-delivery | 7 | |
| Settings | `src/Screens/Main/Profile/SettingsScreen.tsx` | first-delivery | 7 | |
| Composer (tip-tap-based) | `src/Screens/Main/ContentCreation/ContentCreationScreenWithTenTap.tsx` | deferred | — | rich text editor (ADR-0002) |
| Composer (poll/survey) | `src/Screens/Main/ContentCreation/ContentCreationScreenPollSurveyTenTap.tsx` | deferred | — | |
| Repost composer | `src/Screens/Main/ContentCreation/RepostContentCreationScreenWithTenTap.tsx` | deferred | — | |
| Search | `src/Screens/Main/Search/*` | deferred | — | Algolia |
| Notifications | `src/Screens/Main/Notifications/*` | deferred | — | |
| Moderation queue | `src/Screens/Main/Moderation/*` | deferred | — | |
| Report content | `src/Screens/Main/Report/*` | deferred | — | |
| DMs / messaging | not yet implemented in RN | deferred | — | |
| Tip flow | embedded modal | deferred | — | |

## Tab bar plan

5 tabs in the native shell:

| Tab | First delivery | Deferred fallback |
|---|---|---|
| Feed | native | — |
| Search | placeholder | "Search coming soon" until composer/search ships |
| Composer (center button) | placeholder | opens "Composer not yet available — use the web app at www.republike.io" |
| Notifications | placeholder | "Notifications coming soon" |
| Profile | native | — |

The placeholder screens use the `EmptyTabScreen` template — one component, one string per deferred tab.
