# Trials and Intro Offers

Two distinct mechanisms for reducing the barrier to a first subscription. They serve different goals: trials let customers evaluate the product for free, intro offers reduce the price for the first N billing cycles.

## Free Trials

A trial gives the customer full plan access for a fixed number of days with no charge. The customer provides a payment method upfront (via setup checkout) but is not billed until the trial ends.

### How Trials Work

```
Customer subscribes to plan with trial
  → Status: pending_payment
  → Customer completes setup checkout (payment method, no charge)
  → Status: trialing (trialEndsAt set, normalized to midnight UTC)
  → Full access, no charge, usage accumulates
  → Trial expires (billing cron detects trialEndsAt <= now)
  → First invoice generated at current plan price
  → Status: active
```

### Trial Configuration

Trial duration is set on the plan price (`trialDays`). When the customer completes setup checkout, the subscription stores `trialEndsAt` calculated from that moment.

### Trial Conversion

When the trial ends, the system charges the price **in effect at that moment** — not the price when the trial started. Trials are not a price commitment. If the founder changes the plan price mid-trial, the customer pays the new price.

### Skipping Trials

Sometimes you want to bypass the trial for a specific customer:

```typescript
const { data: sub } = await commet.subscriptions.create({
  externalId: "user_123",
  planCode: "pro",
  skipTrial: true,
});
// sub.checkoutUrl -> redirects to payment (not setup) checkout
```

### During a Trial

- Full feature access at the plan's limits
- Usage accumulates (metered features track consumption)
- No invoices or charges
- Customer can cancel at any time with no charge

## Intro Offers

An intro offer is a discount applied to the first N billing cycles of a paid subscription. It activates automatically for eligible customers.

### How Intro Offers Work

```
New customer subscribes
  → System checks eligibility (no previous subscriptions)
  → Intro offer applied: discounted price for N cycles
  → subscription.introOfferEndsAt set
  → Each invoice checks: is introOfferEndsAt > now?
  → Yes: apply discount line to invoice
  → After N cycles: normal pricing automatically
```

### Discount Types

| Type | How It Works | Example |
|------|-------------|---------|
| Percentage | Reduces invoice by X% | 50% off = $99 plan billed at $49.50 |
| Fixed amount | Reduces invoice by $X | $30 off = $99 plan billed at $69 |

Percentage values use basis points (10000 = 100%). A 25% discount is stored as `2500`.

### Eligibility

Intro offers are strictly for new customers. One lifetime per customer.

| Customer Type | Eligible? |
|---------------|-----------|
| New customer (no previous subscriptions) | Yes |
| Had a subscription before (active, canceled, expired) | No |
| Active customer upgrading/downgrading | No |
| Had a trial but didn't convert | No |

The rule is simple: if ANY previous subscription exists in any state, the customer is not eligible.

### Intro Offer + Plan Change

When a customer changes plans, the intro offer is **lost**. The proration credit is based on what the customer actually paid (the discounted amount), not the list price. The new plan starts at normal price with no intro offer.

```
Customer on Starter $99/mo with 50% intro offer (paying $49.50)
  Upgrades to Pro $199/mo on day 15 (15/30 days remaining)

  Credit: $49.50 x (15/30) = $24.75  (based on what was paid)
  Charge: $199 (full price, no intro offer on new plan)
  Billed: $174.25
```

### Toggle Behavior

Turning off `introOfferEnabled` on a plan only affects **new subscriptions**. Existing subscriptions keep their discount running until `introOfferEndsAt`.

## Trials + Intro Offers Together

Trials and intro offers are **independent and can be combined**. A plan can have both: the customer gets a free trial first, then pays the discounted intro offer price for N cycles, then transitions to normal pricing.

```
Plan: Pro $99/mo, 14-day trial, 50% off for 3 months

Timeline:
  Day 0-14:   Trial (free, full access)
  Month 1-3:  $49.50/mo (intro offer)
  Month 4+:   $99/mo (normal price)
```

However, at checkout, intro offers and promo codes are **mutually exclusive**. A customer either gets the intro offer or applies a promo code, never both on the same checkout.

## Gotchas

**Trial price is not locked.** The price charged at trial end is whatever the plan costs at that moment. If you raise prices during someone's trial, they pay the new price.

**Intro offer credit uses effective price.** When calculating proration credits for a plan change, the credit is based on the discounted amount the customer actually paid, not the list price.

**One intro offer per lifetime.** Even if the customer cancels and resubscribes months later, they do not get another intro offer. The system checks for any previous subscription in any state.

**Intro offers apply per-currency.** Plans can define different intro offers per currency. At checkout, the system uses the currency-specific intro offer if one exists, otherwise falls back to the base plan's intro offer.

## Code Examples

### Create a subscription (trial auto-applies from plan config)

```typescript
const { data: sub } = await commet.subscriptions.create({
  externalId: "user_123",
  planCode: "pro",
  successUrl: "https://app.example.com/welcome",
});

// If plan has trialDays: customer completes setup checkout (no charge)
// If plan has intro offer and customer is eligible: discount auto-applies
// sub.checkoutUrl -> redirect customer here
```

### Check subscription status during trial

```typescript
const { data: sub } = await commet.subscriptions.get("user_123");

if (sub.status === "trialing") {
  // Customer is in free trial
  // sub.currentPeriod.end = trial end date
}
```

### Check feature access (works the same during trial)

```typescript
const { data } = await commet.features.check({
  code: "advanced_analytics",
  externalId: "user_123",
});
// data.allowed = true (full access during trial)
```
