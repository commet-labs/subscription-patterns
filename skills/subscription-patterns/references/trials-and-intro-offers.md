# Trials and Introductory Placement

A trial and an introductory discount are phases of the same independent Offer model. A `free_trial` phase delays the first charge; a discount phase changes the plan-base amount for one or more billing cycles. Introductory placement determines automatic eligible-customer selection.

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

Create an Offer with a first `free_trial` phase. Attach a compatible Offer to one base plan price for automatic introductory selection, or pass it directly with `offerId`. The `trialDays` and `customTrialDays` request fields remain compatibility shortcuts; accepted terms are still recorded as an Offer Application.

### Trial Conversion

When the trial ends, the system charges the price **in effect at that moment** — not the price when the trial started. Trials are not a price commitment. If the founder changes the plan price mid-trial, the customer pays the new price.

### Skipping Trials

Sometimes you want to bypass the trial for a specific customer:

```typescript
const sub = await commet.subscriptions.create({
  customerId: "user_123",
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

An Introductory Offer is not a separate catalog type. It is an independent Offer attached to one base plan price through introductory placement. It may include an optional first free-trial phase and at most one finite discount phase. It activates automatically for eligible customers unless checkout receives an explicit `offerId`.

### How Intro Offers Work

```
New customer subscribes
  → System checks current eligibility
  → Price's automatic Introductory Offer is selected
  → Accepted phases are persisted as an immutable Offer Application
  → Discount applies to the plan-base line for its configured cycles
  → After the phases finish: normal pricing automatically
```

### Discount Types

| Type | How It Works | Example |
|------|-------------|---------|
| Percentage | Reduces invoice by X% | 50% off = $99 plan billed at $49.50 |
| Fixed amount | Reduces invoice by $X | $30 off = $99 plan billed at $69 |

Percentage values use basis points (10000 = 100%). A 25% discount is stored as `2500`.

### Eligibility

Current eligibility excludes a customer who already has an `active` or `past_due` subscription in the organization. Historical canceled subscriptions do not create a lifetime ban.

| Customer Type | Eligible? |
|---------------|-----------|
| No active or past-due subscription | Yes |
| Active or past-due subscription | No |
| Previously canceled subscription | Yes, if no current blocker remains |
| Active customer upgrading/downgrading | No |

### Intro Offer + Plan Change

When a customer changes plans, the prior Offer Application ends. The proration credit is based on what the customer actually paid, not the list price. An immediate plan change may apply a new direct `offerId`; otherwise the new plan starts at normal price. A scheduled plan change does not accept an Offer.

```
Customer on Starter $99/mo with 50% intro offer (paying $49.50)
  Upgrades to Pro $199/mo on day 15 (15/30 days remaining)

  Credit: $49.50 x (15/30) = $24.75  (based on what was paid)
  Charge: $199 (full price when no new offerId is supplied)
  Billed: $174.25
```

### Catalog Changes

Updating, deactivating, or archiving an Offer changes future selection only. Existing subscriptions keep the accepted phases stored in their Offer Application. That immutable boundary applies to the Offer terms, not the selected plan price: renewals still read the current catalog amount of the selected price.

## Trials + Intro Offers Together

One Offer can contain the trial and introductory discount in order. The customer gets the free trial first, then pays the discounted price for N cycles, then transitions to normal pricing.

```
Plan: Pro $99/mo, 14-day trial, 50% off for 3 months

Timeline:
  Day 0-14:   Trial (free, full access)
  Month 1-3:  $49.50/mo (intro offer)
  Month 4+:   $99/mo (normal price)
```

At checkout, Commet resolves one Offer source. Omitting `offerId` allows the price's automatic introductory placement when the customer is eligible. Passing a direct `offerId` overrides it. Promo Codes reference an independent single-discount Offer, but validation rejects the code with `intro_offer_active` while an eligible automatic Introductory Offer applies.

## Gotchas

**Trial price is not locked.** The price charged at trial end is whatever the plan costs at that moment. If you raise prices during someone's trial, they pay the new price.

**Intro offer credit uses effective price.** When calculating proration credits for a plan change, the credit is based on the discounted amount the customer actually paid, not the list price.

**Eligibility is not a lifetime-history rule.** It currently blocks active and past-due subscriptions. Verify the creation service before changing this policy.

**Currency-specific Offer phases are explicit.** `amount_off` and `fixed_price` phases carry an amount for each supported currency. Commet does not silently fall back across currencies.

## Code Examples

### Create a subscription (trial auto-applies from plan config)

```typescript
const sub = await commet.subscriptions.create({
  customerId: "user_123",
  planCode: "pro",
  successUrl: "https://app.example.com/welcome",
});

// If the selected Offer starts with free_trial, checkout saves the payment method
// without charging it.
// If the selected price has an automatic Intro Offer and the customer is
// eligible, it applies when offerId is omitted.
// sub.checkoutUrl -> redirect customer here
```

### Check subscription status during trial

```typescript
const sub = await commet.subscriptions.getActive({ customerId: "user_123" });

if (sub.status === "trialing") {
  // Customer is in free trial
  // sub.currentPeriod.end = trial end date
}
```

### Check feature access (works the same during trial)

```typescript
const access = await commet.featureAccess.get({
  code: "advanced_analytics",
  customerId: "user_123",
});
// access.allowed = true (full access during trial)
```
