# Proration Logic

How to calculate fair charges when a subscription changes mid-cycle. Proration ensures customers pay only for what they use and get credit for what they don't.

## Core Formula

Proration credit is always **time-based**, regardless of consumption model. This is the simplest and fairest approach: you paid for N days, used M, we refund the rest proportionally.

```
Credit = (remainingDays / totalDays) x effectivePrice
```

- `remainingDays`: days left in the current billing period from the change date
- `totalDays`: total days in the current billing period
- `effectivePrice`: what the customer actually paid (accounts for intro offers)

## Upgrade Proration

On upgrade, the customer gets a credit for unused time and pays full price for the new plan.

### Basic Example

```
Current plan: Starter $29/mo (paid January 1, 30-day period)
New plan: Pro $99/mo
Change date: January 15 (15 days remaining)

Credit: $29 x (15/30) = $14.50
New charge: $99 (full price, period resets to Jan 15 - Feb 15)
Net billed: $84.50
```

### With Intro Offer

The credit is based on the **effective price** (what was actually paid), not the list price.

```
Current plan: Starter $99/mo with 50% intro offer (paying $49.50)
Change date: day 15 (15 days remaining of 30)

Credit: $49.50 x (15/30) = $24.75  (not $99 x 15/30)
```

### With Extra Seats

Seats have their own proration calculated separately using the same time-based formula.

```
Plan: $29/mo, 5 included seats, $25/extra
Customer has 8 seats (3 extra = $75/mo in seat charges)
Change date: day 15 (15 days remaining of 30)

Plan credit: $29 x (15/30) = $14.50
Seats credit: $75 x (15/30) = $37.50
Total credit: $52.00
```

### Complete Example: Credits + Seats + Boolean

```
Starter $29/mo (Credits model, paid Jan 1):
  50 included credits
  5 included seats, $25/extra
  SSO: not included

Customer state (day 15, 15 remaining of 30):
  planCredits = 10 (used 40)
  purchasedCredits = 20
  seats = 8 (3 extra = $75/mo)

Upgrade to Pro $99/mo:
  200 included credits
  10 included seats, $20/extra
  SSO: included

Calculation:
  Plan credit: (15/30) x $29 = $14.50  (time-based, not credit-based)
  Seats credit: (15/30) x $75 = $37.50
  New plan charge: $99
  Net charge: $99 - $14.50 - $37.50 = $47.00

After upgrade:
  planCredits = 200 (reset to new plan)
  purchasedCredits = 20 (preserved, always intact)
  SSO: enabled
```

### With Pending Overage

If the customer has accumulated usage overage on the old plan, that overage is settled during the upgrade at the old plan's rates.

```
Starter $29/mo, 10K API calls, $0.02/extra
Customer used 15K calls (5K overage)
Change date: day 15

Plan credit: $29 x (15/30) = $14.50
Pending overage: 5K x $0.02 = $100 (old plan's overage rate)
New charge: $99
Total: $99 - $14.50 + $100 = $184.50
```

## Downgrade Proration

No proration on downgrade. The customer keeps their current plan until the period ends, then switches to the new plan at the new price. They already paid for the current period and should enjoy it fully.

## New Plan Always Charges Full Price

Upgrades reset the billing period. The new plan is **always charged at full price** for a fresh cycle. There is no partial charge for the new plan — the period restarts from the upgrade date.

## Edge Cases

### Same-Day Upgrade (Day 1 of Period)

Credit is approximately 100% of the old plan. Customer pays roughly the price difference.

```
Upgrade on day 1 of 30-day period:
  Credit: $29 x (29/30) = $28.03
  New charge: $99
  Net: $70.97
```

### Last-Day Upgrade

Minimal credit. Period restarts anyway.

```
Upgrade on day 29 of 30-day period:
  Credit: $29 x (1/30) = $0.97
  New charge: $99
  Net: $98.03
```

### Multiple Upgrades in Same Period

Each upgrade is calculated independently based on the current plan at that moment. There is no accumulation of credits from previous upgrades — each change stands alone.

```
Day 1: Subscribe to Starter $29/mo
Day 10: Upgrade to Pro $99/mo (credit on $29, charge $99)
Day 20: Upgrade to Enterprise $299/mo (credit on $99, charge $299)
```

### Upgrade Then Downgrade Request

If a customer upgrades and then immediately requests a downgrade, the downgrade is scheduled for the end of the new period. No refund is issued for the upgrade.

## Price Changes and Proration

Price changes (made by the founder) do not trigger mid-cycle proration. The principle: the customer already paid for the current period. Any price change — increase or decrease — applies at the next renewal.

```
Day 1: Price = $99, Customer A subscribes
Day 10: Founder changes price to $119
Day 20: Founder changes price to $109

Customer A's current period: $99 (what they paid at start)
Customer A's next renewal: $109 (price in effect at renewal time)
```

Each customer pays what was in effect at the start of **their** period.

## Code Examples

### Check current period and remaining days

```typescript
const { data: sub } = await commet.subscriptions.get("user_123");

const { start, end, daysRemaining } = sub.currentPeriod;
// Use daysRemaining to estimate proration before showing upgrade UI
```

### Present upgrade cost preview via customer portal

```typescript
const { data } = await commet.portal.getUrl({ externalId: "user_123" });
// Portal shows prorated cost comparison before customer confirms
```

### Track usage to understand pending overage before upgrade

```typescript
const { data: features } = await commet.features.list("user_123");

for (const feature of features) {
  if (feature.type === "metered" && feature.overage > 0) {
    // This overage will be settled on upgrade
    // feature.current = total used
    // feature.included = plan's included amount
    // feature.overage = current - included
  }
}
```
