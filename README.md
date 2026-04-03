# Subscription Patterns

Universal subscription billing patterns for SaaS products. How proration works, how to implement free trials, when to apply plan changes, how to handle failed payments, and more.

Framework-agnostic patterns with [@commet/node](https://commet.co) code examples.

## Install

```bash
npx skills add commet-labs/subscription-patterns
```

## What's Inside

| Pattern | Description |
|---------|-------------|
| Trials & Intro Offers | Free trials, discounted first cycles, conversion to paid |
| Upgrades & Downgrades | Plan changes, when to apply immediately vs at renewal |
| Proration Logic | Mid-cycle credits and charges, time-based calculation |
| Dunning & Retries | Failed payment handling, retry schedules, grace periods |
| Cancellation Flows | Immediate vs end-of-period, reactivation, save offers |
| Add-ons & Extras | Feature extensions, mid-cycle activation, consumption matching |

## Key Principle

Every subscription change follows one rule:

| Change Type | When Applied |
|-------------|--------------|
| Benefits customer | Immediately |
| Hurts customer | At renewal |

Upgrades are immediate with proration. Downgrades wait until the period ends. Price increases take effect at renewal. This principle drives every pattern in this skill.

## Links

- [Commet](https://commet.co) — Billing and payments for SaaS
- [Commet SDK](https://github.com/commet-labs/commet) — SDKs and libraries
- [Commet Skills](https://github.com/commet-labs/commet-skills) — Full Commet agent skills

## License

Apache-2.0
