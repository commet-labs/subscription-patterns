---
name: subscription-patterns
description: Use when implementing subscription billing — free trials, intro offers, plan upgrades/downgrades, proration logic, dunning and payment retries, cancellation flows, plan groups, or add-ons. Contains patterns for the full subscription lifecycle.
license: MIT
metadata:
  author: commet
  version: "1.0.0"
  homepage: https://commet.co
  source: https://github.com/commet-labs/subscription-patterns
---

# Subscription Billing Patterns

Universal patterns for the full subscription lifecycle. These apply to any billing system — the examples use `@commet/node` but the logic is the same everywhere.

## Subscription State Machine

Every subscription moves through these states:

```
draft → pending_payment → trialing → active → canceled
                               │          │
                               └──────────┴→ past_due
                                              │
                                              ├→ active
                                              └→ canceled
```

The persisted statuses are `draft`, `pending_payment`, `trialing`, `active`, `past_due`, and `canceled`. Commet does not expose a pause/resume status.

**One active subscription relationship per customer.** Treat the active endpoint and current API errors as the source of truth when deciding whether a new subscription may be created. Do not recreate lifecycle rules from stale status lists.

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

Do not infer subscription behavior from a generic fairness rule. In Commet, plan changes are classified from billing interval, plan-group order, and paid/free transitions. Price edits affect renewal through the selected catalog price, while accepted Offer phases remain immutable. Use the linked reference for the exact operation you are implementing.
