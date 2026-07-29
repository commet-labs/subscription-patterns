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
  id: "sub_xxx",
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
  id: "sub_xxx",
  reason: "Duplicate account",
  immediate: true,
});
```

## What Happens to Resources on Cancellation

| Resource | End-of-Period | Immediate |
|----------|--------------|-----------|
| Feature access | Until period end | Revoked now |
| Plan credits | Until period end | Access revoked now |
| Plan balance | Until period end | Access revoked now |
| Purchased credits | Persisted | Persisted |
| Usage counters | Continue until period end | Stopped |
| Seats | Active until period end | Released |

Cancellation does not erase the persisted subscription balance. Access follows subscription state.

## Reactivation

A canceled paid subscription can be reactivated through the dedicated operation. Commet charges first, then restores the same subscription relationship with a new period anchored to the reactivation date.

| Aspect | Behavior |
|--------|----------|
| Pricing | Current value of the selected catalog price |
| Offer | Optional Promotional `offerId`; accepted phases are snapshotted |
| Balance | Existing initialized balance is not automatically reset |
| Subscription identity | Same subscription record |

### Code Example

```typescript
const sub = await commet.subscriptions.reactivate({
  id: "sub_xxx",
});
// sub.status === "active" after the reactivation charge succeeds
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
const portal = await commet.portal.getUrl({ customerId: "user_123" });
// Portal shows plan comparison before confirming cancellation
```

## Reactivation Within Period

If a customer schedules an end-of-period cancellation and changes their mind before it takes effect, use `subscriptions.uncancel`. The customer stays on the current relationship without a reactivation charge.

## Data Retention

After cancellation:
- Customer record persists (for billing history, compliance)
- Subscription record persists with `canceled` status
- Invoices and payment history retained
- Purchased credits retained on the customer
- Subscription balance history retained

## Cancellation Reasons

Tracking why customers cancel helps identify patterns:

```typescript
await commet.subscriptions.cancel({
  id: "sub_xxx",
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

**Reactivation reuses the subscription.** It charges the current selected catalog price and restores state only after successful payment.

**Uncancel and reactivate are different.** `uncancel` reverses a scheduled cancellation before it takes effect. `reactivate` restores an already canceled subscription and creates a new paid period.

**Balance rows persist.** Do not add an application-side “restore” or “reset” step around reactivation.
