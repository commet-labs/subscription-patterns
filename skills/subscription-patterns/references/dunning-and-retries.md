# Dunning and Retries

How to handle failed payments, retry logic, and involuntary churn prevention. Dunning is the process of recovering revenue from failed payment attempts before the subscription is lost.

## Payment Failure Flow

When a payment fails, the subscription enters a grace period before cancellation:

```
Invoice generated → Payment attempt fails
  → Status: past_due
  → Grace period begins (access maintained)
  → Automatic retries on days 1, 3, 5, and 7
  → All retries exhausted and still unpaid
  → Status: canceled
```

### State Transitions

| Event | Status Change | Customer Access |
|-------|--------------|-----------------|
| Payment fails | `active` -> `past_due` | Maintained |
| During grace period | `past_due` | Maintained |
| Retry succeeds | `past_due` -> `active` | Maintained (never interrupted) |
| All retries fail | `past_due` -> `canceled` | Revoked |

## Grace Period

During `past_due`, the customer retains full access to their subscription. This is intentional — cutting access immediately on a transient payment failure (expired card, temporary bank hold) creates a terrible experience and increases churn unnecessarily.

The original renewal decline is day 0. Automatic retries are anchored to that instant on days 1, 3, 5, and 7. If a retry succeeds, the subscription returns to `active` without creating a new subscription. The fourth declined retry exhausts dunning and cancels the subscription.

## What Happens to Consumption

| Resource | During past_due | On Cancellation |
|----------|----------------|-----------------|
| Plan credits | Maintained | Persisted; access follows subscription state |
| Plan balance | Maintained | Persisted; access follows subscription state |
| Purchased credits | Maintained | Persisted |
| Usage counters | Continue tracking | Stopped |
| Feature access | Full access | Revoked |

Purchased credits survive cancellation. They are never forfeited because the customer paid for them separately from the plan.

## Involuntary vs Voluntary Churn

| Type | Cause | Strategy |
|------|-------|----------|
| Involuntary | Payment failure (expired card, insufficient funds) | Dunning, retries, customer notification |
| Voluntary | Customer chooses to cancel | Save offers, downgrade suggestions (see cancellation-flows.md) |

Dunning addresses involuntary churn. The customer didn't choose to leave — their payment failed. The goal is to recover the payment before resorting to cancellation.

## Customer Notification Patterns

Failed payments need proactive communication. Common notification sequence:

```
Payment fails:
  1. Immediate: "Your payment failed. Please update your payment method."
  2. Before final retry: "Action needed — update payment to keep your subscription."
  3. After cancellation: "Your subscription was canceled due to payment failure."
```

### Implementation with Webhooks

```typescript
import { Commet } from "@commet/node";

const commet = new Commet({ apiKey: process.env.COMMET_API_KEY! });

// Verify webhook signature
const payload = commet.webhooks.verifyAndParse({
  rawBody,
  signature: headers["x-commet-signature"],
  secret: process.env.COMMET_WEBHOOK_SECRET!,
});

if (!payload) {
  // Invalid signature
  return;
}

switch (payload.event) {
  case "subscription.updated":
    if (payload.data.status === "past_due") {
      // Payment failed — notify customer to update payment method
      await sendPaymentFailedEmail(payload.data.customerId);
    }
    break;

  case "subscription.canceled":
    // Could be voluntary or involuntary
    // Check if previous status was past_due for involuntary
    await sendCancellationEmail(payload.data.customerId);
    break;
}
```

## Recovery Strategies

### 1. Customer Portal for Payment Update

The simplest recovery path: send the customer to their portal where they can update their payment method.

```typescript
const portal = await commet.portal.getUrl({ customerId: "user_123" });
// Include portal.portalUrl in your "update payment method" email
```

### 2. Prevention: Monitor Subscription Status

Proactively check subscription status to catch issues early:

```typescript
const sub = await commet.subscriptions.getActive({ customerId: "user_123" });

if (sub.status === "past_due") {
  // Show in-app banner: "Payment issue — update your payment method"
  // Link to customer portal
}
```

### 3. Feature Gating as Soft Nudge

Instead of hard-blocking on `past_due`, you can show warnings while maintaining access:

```typescript
const sub = await commet.subscriptions.getActive({ customerId: "user_123" });

if (sub.status === "past_due") {
  // Show banner but don't block
  showPaymentWarningBanner();
}

if (sub.status === "canceled") {
  // Hard block — subscription is gone
  redirectToResubscribe();
}
```

## Currency and Failed Retries

Before the first successful payment, reopening checkout may resolve presentment again from the current checkout context. Once activated, renewal and dunning stay bound to the subscription's currency, saved payment method, and provider connection; a billing-address change does not silently move a retry to another rail.

## Metrics to Track

| Metric | What It Tells You |
|--------|-------------------|
| `past_due` rate | How often payments fail |
| Recovery rate | % of `past_due` that return to `active` |
| Time to recovery | How quickly failed payments are resolved |
| Involuntary churn rate | % of cancellations from payment failure |

## Gotchas

**Access continues during `past_due`.** This is by design. Do not add custom logic to block access during the grace period — it increases churn without improving recovery.

**Cancellation does not erase the stored balance row.** Access still follows subscription state. Reactivation charges first and reuses the existing relationship; it does not automatically reinitialize an already initialized balance.

**Retries use an anchored schedule.** The current automatic retry days are 1, 3, 5, and 7 after the original decline. Manual and scheduled retries share the same claim protocol, so do not build a second retry scheduler around them.
