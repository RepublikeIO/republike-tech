# ADR 0008 — In-app purchases (subscriptions)

**Status:** Accepted (Phase 5)
**Date:** 2026-05-29

## Context

Republike subscriptions on mobile must go through the platform billing APIs — Apple and Google's policies require it for digital content. The current RN app uses `react-native-iap`. The native rewrite uses each platform's first-party billing API directly.

## Decision

| | iOS | Android |
|---|---|---|
| API | StoreKit 2 | Google Play Billing Library 6.x |
| Min OS | iOS 17 ✓ (StoreKit 2 needs iOS 15+) | API 33 ✓ (PBL 6 supports API 21+) |
| Receipt validation | Server-side via existing `subscription.verifyApplePurchase` tRPC procedure | Server-side via existing `subscription.verifyAndroidPurchase` |
| Restore | StoreKit 2 `Transaction.currentEntitlements` | PBL `queryPurchasesAsync` on resume |
| Promo codes | StoreKit `presentCodeRedemptionSheet` | Play Console promo codes (redeemed in-store) |

### Product configuration

Product IDs match the current RN setup:

- `io.republike.app.founding_citizen.yearly`
- `io.republike.app.founding_father.yearly`
- `io.republike.app.founding_consul.yearly`

Free Visiting Citizen is NOT a paid product — it's activated server-side via `user.activateFreeTier`.

### iOS — StoreKit 2 flow

```swift
import StoreKit

// 1. Load products
let products = try await Product.products(for: productIDs)

// 2. Purchase
let result = try await product.purchase()

// 3. Verify on success
switch result {
case .success(.verified(let transaction)):
    let receipt = transaction.jsonRepresentation
    try await api.subscription.verifyApplePurchase(receipt: receipt)
    await transaction.finish()
case .success(.unverified(_, let error)):
    throw IAPError.unverified(error)
// ...
}
```

The native code does NOT trust StoreKit's verification — server reverifies via App Store Server API. `transaction.finish()` only runs after the server confirms.

### Android — Play Billing 6 flow

```kotlin
val billingClient = BillingClient.newBuilder(context)
    .enablePendingPurchases()
    .setListener { result, purchases -> ... }
    .build()

val productDetails = billingClient
    .queryProductDetailsAsync(...)

val billingFlowParams = BillingFlowParams.newBuilder()
    .setProductDetailsParamsList(...)
    .build()

billingClient.launchBillingFlow(activity, billingFlowParams)

// In the listener:
suspend fun handlePurchase(purchase: Purchase) {
    api.subscription.verifyAndroidPurchase(
        purchaseToken = purchase.purchaseToken,
        productId = purchase.products.first(),
    )
    billingClient.acknowledgePurchase(...)  // server-confirmed first
}
```

Same posture: server-side verification before acknowledgement.

### Subscription state of truth

The server's `subscription.getMyCurrentPlan()` is the single source of truth. The native apps:

1. Trigger the platform purchase flow.
2. Send the receipt / token to the server.
3. Refresh the user profile (which carries `currentPlan` + `AccessContext`).
4. Update UI from the refreshed `currentUser`.

We never check StoreKit / Play Billing locally to decide what a user can access. That's `AccessContext` from the server.

### Renewal / lapsed handling

- StoreKit 2 / PBL fire transaction events for renewals — the native client forwards each to the server, which keeps `User.currentPlan` current.
- A background `Transaction.updates` listener (iOS) / billing-client renewal events (Android) runs while the app is alive.
- The server also runs `subscription` webhooks (Apple S2S notifications, Google RTDN) which are the authoritative path. The mobile updates are belt-and-suspenders.

When a subscription lapses, the next API call returns `USER_INACTIVE` (per `AccessContext`), which the API client interceptor routes to the paywall via the coordinator/dispatcher.

## Consequences

- Two billing implementations to maintain. No shared abstraction — they share the verification API but nothing else.
- No third-party library (no `react-native-iap` equivalent). We own the StoreKit / PBL code.
- Receipt verification depends on the webapp's existing endpoints — keep those alive through the rewrite.

## Maps to RN source

- `republike-mobile/src/Screens/Auth/Paywall/PlanScreen.tsx`
- `republike-mobile/src/Screens/Auth/Paywall/VerifyPaymentScreen.tsx`
- `republike-mobile/src/Screens/Auth/Paywall/SubscriptionEndedScreen.tsx`
- `republike-mobile/src/Services/iap/*` (`react-native-iap` integration)
- `republike-webapp/src/server/services/apple-payment.service.ts`
- `republike-webapp/src/server/services/google-payment.service.ts`

## Alternatives considered

- **RevenueCat.** SaaS for IAP — handles cross-platform abstraction, A/B tests, etc. Rejected — vendor lock-in, recurring cost, and another party in the receipt chain.
- **Stripe Mobile SDK.** Cannot be used for digital subscriptions on iOS / Android per store policy.
- **Shared Kotlin Multiplatform billing layer.** Considered. Rejected — KMP introduces complexity for what's already platform-specific by design.
