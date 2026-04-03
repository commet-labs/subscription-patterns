---
name: subscription-patterns
description: Use when implementing subscription billing — free trials, intro offers, plan upgrades/downgrades, proration logic, dunning and payment retries, cancellation flows, plan groups, or add-ons. Contains patterns for the full subscription lifecycle.
license: Apache-2.0
metadata:
  author: commet
  version: "1.0.0"
  homepage: https://commet.co
  source: https://github.com/commet-labs/subscription-patterns
references:
  - references/trials-and-intro-offers.md
  - references/upgrades-downgrades.md
  - references/proration-logic.md
  - references/dunning-and-retries.md
  - references/cancellation-flows.md
  - references/addons-and-extras.md
---

# Subscription Billing Patterns

Universal patterns for the full subscription lifecycle. These apply to any billing system — the examples use `@commet/node` but the logic is the same everywhere.

## Subscription State Machine

Every subscription moves through these states:

```
                    ┌─────────────────────────────────────┐
                    │                                     v
draft → pending_payment → trialing → active → canceled
                                       │
                                       v
                                     paused → past_due → expired
```

**One active subscription per customer.** States that block a new subscription: `draft`, `pending_payment`, `trialing`, `active`, `paused`, `past_due`. States that allow a new one: `canceled`, `expired`.

## Quick Reference

| Task | File |
|------|------|
| Implement free trials | [trials-and-intro-offers.md](references/trials-and-intro-offers.md) |
| Add intro offer discounts | [trials-and-intro-offers.md](references/trials-and-intro-offers.md) |
| Handle plan upgrades | [upgrades-downgrades.md](references/upgrades-downgrades.md) |
| Handle plan downgrades | [upgrades-downgrades.md](references/upgrades-downgrades.md) |
| Change billing interval | [upgrades-downgrades.md](references/upgrades-downgrades.md) |
| Calculate proration | [proration-logic.md](references/proration-logic.md) |
| Handle mid-cycle plan changes | [proration-logic.md](references/proration-logic.md) |
| Implement failed payment retries | [dunning-and-retries.md](references/dunning-and-retries.md) |
| Build grace period logic | [dunning-and-retries.md](references/dunning-and-retries.md) |
| Cancel a subscription | [cancellation-flows.md](references/cancellation-flows.md) |
| Reactivate a canceled subscription | [cancellation-flows.md](references/cancellation-flows.md) |
| Add purchasable add-ons | [addons-and-extras.md](references/addons-and-extras.md) |
| Prorate mid-cycle add-on activation | [addons-and-extras.md](references/addons-and-extras.md) |

## Key Principle

Every subscription change follows one rule:

| Change Type | When Applied |
|-------------|--------------|
| Benefits customer | **Immediately** |
| Hurts customer | **At renewal** |

This means: upgrades are immediate (customer gets value now). Downgrades wait until the period ends (customer keeps what they paid for). Price increases take effect at renewal (customer paid for the current period already). Price decreases also apply at renewal for consistency.

This single principle drives every pattern in this skill and eliminates ambiguity about when any change should take effect.
