# Cancellation Flows

How to handle subscription cancellations, reactivation, and retention. Cancellation is not always the end — the right flow can save subscriptions or make re-subscription easy.

## Two Types of Cancellation

| Type | Behavior | Use Case |
|------|----------|----------|
| End-of-period (default) | Access continues until period ends | Customer wants to leave but already paid |
| Immediate | Access ends now | Customer demands immediate cancellation or compliance requirement |

End-of-period is the default and recommended approach. The customer paid for the current period and should be able to use it.

## End-of-Period Cancellation

```
Customer requests cancellation
  → cancelAt set to currentPeriodEnd
  → Status remains: active (with pending cancellation)
  → Customer retains full access until period ends
  → Period ends
  → Status: canceled
  → Access revoked
```

The customer can continue using all features until the billing period expires. No credit is issued because they are using what they paid for.

### Code Example

```typescript
await commet.subscriptions.cancel({
  subscriptionId: "sub_xxx",
  reason: "Too expensive",
  // immediate defaults to false -> cancels at period end
});
```

## Immediate Cancellation

```
Customer requests immediate cancellation
  → Status: canceled (now)
  → Access revoked immediately
  → Unused time credited (prorated refund)
```

### Code Example

```typescript
await commet.subscriptions.cancel({
  subscriptionId: "sub_xxx",
  reason: "Duplicate account",
  immediate: true,
});
```

## What Happens to Resources on Cancellation

| Resource | End-of-Period | Immediate |
|----------|--------------|-----------|
| Feature access | Until period end | Revoked now |
| Plan credits | Until period end, then reset to 0 | Reset to 0 now |
| Plan balance | Until period end, then reset to 0 | Reset to 0 now |
| Purchased credits | **Preserved** | **Preserved** |
| Usage counters | Continue until period end | Stopped |
| Seats | Active until period end | Released |

Purchased credits are never forfeited. They survive cancellation and are restored if the customer reactivates.

## Reactivation

A canceled customer can subscribe again, but it is treated as a **new subscription** with specific rules:

| Aspect | Behavior |
|--------|----------|
| Pricing | Current plan price (not what they previously paid) |
| Intro offer | **Not eligible** (had a subscription before) |
| Purchased credits | Restored (they were preserved) |
| Plan credits/balance | Fresh allocation from new plan |
| Deprecated plans | Cannot re-subscribe to plans with `isPublic = false` |

### Code Example

```typescript
// Customer who canceled wants to come back
const { data: sub } = await commet.subscriptions.create({
  externalId: "user_123",
  planCode: "pro", // Must be a public plan
  successUrl: "https://app.example.com/welcome-back",
});
// sub.checkoutUrl -> new payment checkout (no intro offer)
```

## Save Offers (Retention Strategies)

Before completing cancellation, present alternatives:

### 1. Downgrade Instead of Cancel

```
Customer wants to cancel Pro $99/mo
  → Offer: "Would you like to switch to Starter at $29/mo instead?"
  → If accepted: downgrade scheduled for end of period
  → Customer keeps Pro until period ends, then moves to Starter
```

```typescript
// Check available plans for downgrade suggestion
const { data: plans } = await commet.plans.list();
const cheaperPlans = plans.filter(
  (plan) => plan.prices[0].price < currentPlanPrice
);
```

### 2. Portal Self-Service

Let customers manage their own cancellation through the portal, which can present plan alternatives:

```typescript
const { data } = await commet.portal.getUrl({ externalId: "user_123" });
// Portal shows plan comparison before confirming cancellation
```

### 3. Pause Instead of Cancel

If your system supports pausing, offer it as an alternative. Paused subscriptions skip billing with no access, but the subscription is not terminated.

| Aspect | Pause | Cancel |
|--------|-------|--------|
| Status | `paused` | `canceled` |
| Access | No | No |
| Billing | Skipped | Stopped |
| Resume | Same subscription, current price | New subscription, current price, no intro offer |
| Plan credits | Frozen | Lost |

On resume from pause, the price in effect at time of resuming applies. Pausing does not freeze the price.

## Reactivation Within Period

If a customer cancels with end-of-period and then changes their mind before the period ends, the cancellation can be reversed. The customer stays on their current plan as if nothing happened.

## Data Retention

After cancellation:
- Customer record persists (for billing history, compliance)
- Subscription record persists with `canceled` status
- Invoices and payment history retained
- Purchased credits retained on the customer
- Plan credits, balance, and usage data cleared

## Cancellation Reasons

Tracking why customers cancel helps identify patterns:

```typescript
await commet.subscriptions.cancel({
  subscriptionId: "sub_xxx",
  reason: "Missing feature: SSO", // Free-text reason
});
```

Common categories to analyze:
- Too expensive (opportunity for downgrade offer)
- Missing features (product feedback)
- Switching to competitor (competitive intelligence)
- No longer needed (seasonal or project-based)
- Technical issues (support opportunity)

## Gotchas

**Reactivation is a new subscription.** The customer does not get their old subscription back. They go through checkout again with current pricing and no intro offer.

**Deprecated plans block resubscription.** If the plan was set to `isPublic = false` after the customer originally subscribed, they cannot resubscribe to it. They must choose a currently public plan.

**Purchased credits persist indefinitely.** Even across cancellation and reactivation cycles, purchased credits are never lost. They restore automatically when the customer creates a new subscription.

**Price at resume is not frozen.** If a customer pauses and the price changes before they resume, they pay the new price. Pausing is not a price lock.
