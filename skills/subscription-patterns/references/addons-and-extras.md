# Add-ons and Extras

Purchasable feature extensions that customers add to their existing subscription. Each add-on links to one feature and has its own pricing, activated mid-cycle with proration.

## What Add-ons Are

Add-ons let customers extend their subscription with features not included in their plan. Unlike plan upgrades (which replace the entire plan), add-ons layer on top of the current subscription.

```
Subscription (Pro plan)
  ├── Plan features (included)
  │   ├── API calls: 100K included
  │   ├── Team members: 10 included
  │   └── Custom branding: enabled
  └── Add-ons (purchased separately)
      ├── Priority support: $49/mo (boolean)
      └── Extra storage: $19/mo, 50GB included (metered)
```

## Consumption Model Compatibility

An add-on's consumption model must match the plan's model OR be `boolean`. This prevents mismatched billing logic (e.g., a credits-based add-on on a metered plan).

| Add-on Model | Allowed on Plan Model |
|-------------|----------------------|
| `boolean` | Any plan (metered, credits, balance) |
| `metered` | Metered plans only |
| `credits` | Credits plans only |
| `balance` | Balance plans only |

Boolean add-ons are the most common — they simply enable a feature (like priority support or SSO) regardless of the plan's consumption model.

## Activation Flow

When a customer activates an add-on mid-cycle, they pay a prorated amount for the remainder of the current billing period. At the next renewal, they pay the full add-on price.

```
Billing period: Jan 1 - Jan 31 (30 days)
Add-on activated: Jan 15 (15 days remaining)
Add-on price: $49/mo

Prorated charge: $49 x (15/30) = $24.50
  → Charged immediately via separate invoice
  → Add-on active from Jan 15

Next renewal (Jan 31):
  → Plan base + add-on full price ($49) on regular invoice
```

### Activation Steps

1. Validate consumption model compatibility with current plan
2. Resolve price to subscription currency (via exchange rate if non-USD)
3. Calculate prorated charge: `fullPrice x (daysRemaining / totalDays)`
4. If charge > $0: process payment with tax
5. Create add-on record (status = active)
6. Feature becomes available immediately

The prorated charge creates a separate invoice with type `addon_activation`.

## Deactivation

When a customer deactivates an add-on, it is turned off immediately. No refund is issued for the remaining period.

| Aspect | Behavior |
|--------|----------|
| When takes effect | Immediately |
| Refund | None |
| Feature access | Revoked |
| Next invoice | Add-on line removed |

## Billing at Renewal

At each billing cycle renewal, active add-ons are included in the regular invoice:

```
Invoice for Feb 1 - Feb 28:
  Plan base (Pro):          $99.00
  Add-on (Priority support): $49.00
  Add-on (Extra storage):    $19.00
  Subtotal:                 $167.00
  Intro offer discount:     -$0.00  (if applicable)
  Total:                    $167.00
```

Add-on lines appear after the plan base but before feature overages. Discounts (intro offers, promo codes) apply to the entire invoice including add-on lines.

## Add-on Features

Each add-on is linked to exactly one feature. The add-on defines how that feature behaves:

| Add-on Field | Purpose |
|-------------|---------|
| `basePrice` | Monthly/yearly price in settlement cents |
| `consumptionModel` | boolean, metered, credits, or balance |
| `includedUnits` | Units included (for metered add-ons) |
| `overageRate` | Per-unit overage price (for metered/balance) |
| `creditCost` | Credits per unit (for credits model) |

### Boolean Add-on

Simply enables a feature. No units, no overage.

```
Add-on: Priority Support ($49/mo)
  consumptionModel: boolean
  → Feature "priority_support" enabled
```

### Metered Add-on

Includes units with optional overage pricing.

```
Add-on: Extra Storage ($19/mo)
  consumptionModel: metered
  includedUnits: 50 (GB)
  overageRate: 5000 (rate scale: $0.50/GB over 50)
```

## One Add-on Per Feature

Each feature can only have one add-on per organization. A customer cannot activate the same add-on twice on one subscription. This prevents duplicate feature grants and billing confusion.

## Business Rules Summary

1. **One feature per add-on** per organization
2. **Model must match** plan model or be boolean
3. **Cannot activate twice** on the same subscription
4. **Mid-cycle = prorated** charge immediately
5. **Deactivation = no refund**, feature revoked
6. **Cannot archive** an add-on that has active subscriptions
7. **Soft deletion only** (preserves billing history)

## Code Examples

### Check feature access (works for plan features and add-on features)

```typescript
const { data } = await commet.features.check({
  code: "priority_support",
  customerId: "user_123",
});
// data.allowed = true if add-on is active (or included in plan)
```

### Get detailed feature info

```typescript
const { data } = await commet.features.get({
  code: "extra_storage",
  customerId: "user_123",
});
// data.current = 35 (GB used)
// data.included = 50 (from add-on)
// data.remaining = 15
// data.overage = 0
```

### Track usage against an add-on's metered feature

```typescript
await commet.usage.track({
  customerId: "user_123",
  feature: "extra_storage",
  value: 5, // 5 GB uploaded
});
```

### Customer portal for self-service add-on management

```typescript
const { data } = await commet.portal.getUrl({ customerId: "user_123" });
// Portal shows available add-ons with prorated price preview
// Customers can activate/deactivate from here
```

### List all features (plan + add-ons combined)

```typescript
const { data: features } = await commet.features.list("user_123");

for (const feature of features) {
  // feature.code, feature.name, feature.type
  // feature.allowed, feature.current, feature.included
  // Works the same whether the feature comes from a plan or add-on
}
```

## Gotchas

**Proration uses the same time-based formula as plan upgrades.** `fullPrice x (daysRemaining / totalDays)`. No special logic.

**Deactivation is immediate with no refund.** Unlike plan downgrades (which wait until period end), add-on deactivation takes effect now. The customer chose to remove it.

**Add-on pricing is in USD settlement cents.** At activation, the price is converted to the subscription's currency using the plan's exchange rate.

**Boolean add-ons work on any plan.** You do not need to check the plan's consumption model for boolean add-ons. They always work.
