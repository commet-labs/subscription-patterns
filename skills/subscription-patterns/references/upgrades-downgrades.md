# Upgrades and Downgrades

How to handle plan changes within a subscription. The core question is always: does this change benefit or hurt the customer?

## The Rule

| Change | When Applied | Why |
|--------|-------------|-----|
| Upgrade (more expensive plan) | **Immediately** | Customer wants more value now |
| Downgrade (cheaper plan) | **At end of period** | Customer already paid for current period |

This eliminates all ambiguity. You never need to decide "should this be immediate?" — just compare the prices.

## Upgrades

An upgrade means the customer moves to a more expensive plan. It takes effect immediately with proration.

### What Happens on Upgrade

1. **Billing period restarts** from the change date
2. **Allocations reset** to the new plan (credits, balance, usage counters)
3. **Credit issued** for unused time on the old plan
4. **Full charge** for the new plan (new period starts fresh)
5. **Pending overage** from old plan is settled

```
Customer on Starter $29/mo (paid January 1)
  Upgrades to Pro $99/mo on January 15

  Old plan credit: $29 x (15 remaining / 30 total) = $14.50
  New plan charge: $99 (full price, period resets to Jan 15 - Feb 15)
  Net charge: $84.50
```

### Upgrade with Pending Usage

If the customer has accumulated overage on the old plan, that overage is settled at the old plan's rates as part of the upgrade.

```
Starter $29/mo, 10K API calls included, $0.02/extra
  Customer used 15K calls (5K overage)
  Upgrades to Pro $99/mo

  Plan credit: $14.50
  Pending overage: 5K x $0.02 = $100 (old plan's rate)
  New charge: $99
  Total billed: $99 - $14.50 + $100 = $184.50
```

### Feature Access on Upgrade

Immediate. The customer gets the new plan's features, limits, and allocations the moment the upgrade processes. Old usage counters reset.

## Downgrades

A downgrade means the customer moves to a cheaper plan. It takes effect at the end of the current billing period.

### What Happens on Downgrade

1. **Customer keeps current plan** for the remainder of the period
2. **No credit issued** (customer enjoys what they paid for)
3. **At renewal**, subscription switches to the new plan at the new price
4. **New plan's allocations** apply from the next period

### Downgrade with Seats Exceeding New Limit

If the customer has 8 seats and downgrades to a plan that includes 5 seats, the change is allowed with a clear warning. The customer sees: "You have 8 active seats. New plan includes 5. Extra seats cost $25/each."

The system does not force seat removal. Extra seats become paid seats on the new plan.

### Downgrade with Usage Exceeding New Limit

Similarly, if current usage exceeds the new plan's included amount, the change is allowed with a warning showing estimated overage costs at the new plan's pricing.

## Billing Interval Changes

Changing the billing interval (monthly/yearly) on the same plan follows the same benefit/hurt rule.

| Change | When Applied | Reasoning |
|--------|-------------|-----------|
| Monthly to yearly | **Immediate** | Commitment upgrade, usually better per-month price |
| Yearly to monthly | **At end of period** | Commitment downgrade, customer paid for the year |

### Upgrade + Interval Change Simultaneously

Treated as a normal upgrade (more expensive = immediate). The system evaluates the total cost change, not each dimension separately.

### Downgrade + Interval Change Simultaneously

Treated as a downgrade (cheaper = at renewal). Customer enjoys their full current period.

## Plan Groups

Plan changes only happen within the same **plan group**. A plan group defines which plans are upgrade/downgrade paths of each other (e.g., Starter, Pro, Enterprise). You cannot change from a plan in one group to a plan in another group — that requires canceling and creating a new subscription.

## Code Examples

### Create a subscription the customer can later upgrade

```typescript
const { data: sub } = await commet.subscriptions.create({
  customerId: "user_123",
  planCode: "starter",
  billingInterval: "monthly",
  successUrl: "https://app.example.com/welcome",
});
// sub.checkoutUrl -> redirect customer to complete payment
```

### Check current subscription before presenting upgrade options

```typescript
const { data: sub } = await commet.subscriptions.get("user_123");

// sub.plan.name = "Starter"
// sub.plan.basePrice = 2900 (cents)
// sub.currentPeriod.daysRemaining = 15
// sub.features -> current feature limits and usage
```

### List available plans for upgrade/downgrade UI

```typescript
const { data: plans } = await commet.plans.list();

// Filter to same plan group, show price comparison
// plans[].prices -> billing interval options
// plans[].features -> feature comparison
```

### Generate customer portal for self-service plan changes

```typescript
const { data } = await commet.portal.getUrl({ customerId: "user_123" });
// data.portalUrl -> customer can upgrade/downgrade from here
```

The customer portal handles plan changes, proration preview, and payment automatically. Customers see a comparison of their current plan vs available plans, with prorated amounts calculated before they confirm.
