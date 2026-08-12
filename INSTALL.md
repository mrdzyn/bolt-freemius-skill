# Install the Freemius Billing skill in Bolt.new

## Import into Bolt

1. Open **Bolt.new**.
2. Open **Settings → Skills library** for a reusable workspace skill, or the project's Skills settings for project-only use.
3. Choose **Add skill → Import from file**.
4. Import `SKILL.md` or the provided ZIP package.
5. Confirm the skill name: `freemius-billing-bolt`.
6. Enable the skill for the Bolt project if required.

Invoke it manually with:

`/$freemius-billing-bolt`

Example SaaS prompt:

```text
/$freemius-billing-bolt

Integrate Freemius monetization into this Bolt SaaS.
Inspect the existing authentication, Supabase schema, RLS policies and edge functions first.

Use Freemius sandbox while building.
Use the Freemius plans already configured in my product as the source of truth.
Add:
- subscription pricing and checkout
- server-side entitlement synchronization
- secure paywalls for premium API/actions
- verified lifecycle webhooks
- an Account/Billing page with Customer Portal
- upgrades, cancellation, payment history and invoices

Keep all Freemius API/secret keys in Bolt/Supabase server secrets.
Do not expose service-role or Freemius secret credentials to the browser.
Do not create a second authentication system.
```

Example subscriptions + credits prompt:

```text
/$freemius-billing-bolt

Integrate my Freemius subscriptions and one-time top-up credits into this existing Bolt app.
Read the current Freemius product/plans from the server-side SDK rather than hard-coding pricing.
Use sandbox, synchronize entitlements by webhook, maintain an idempotent credit ledger, and enforce paid limits in backend functions.
```

## Freemius setup before integration

In the Freemius Developer Dashboard:

- Create/configure your SaaS or App product.
- Configure plans, prices, billing cycles and plan features.
- Configure top-up/one-time plans if used.
- Go to **Settings → API & Keys** and copy the official JS SDK environment configuration into secure Bolt/Supabase secrets.
- Keep `FREEMIUS_SANDBOX=true` while testing.

The skill should determine plan/pricing IDs from your existing Freemius configuration when server credentials allow it instead of requiring IDs to be hard-coded into the prompt.

## Official sources used

- https://freemius.com/help/documentation/ai/skills.md
- https://freemius.com/help/documentation/saas/saas-integration/
- https://freemius.com/help/documentation/saas-sdk/js-sdk/installation/
- https://freemius.com/help/documentation/saas-sdk/js-sdk/integration/
- https://freemius.com/help/documentation/saas-sdk/js-sdk/webhooks/
- https://freemius.com/help/documentation/saas/integrating-license-key-activation/
- https://freemius.com/help/llms.txt
- https://github.com/Freemius/freemius-ai
- https://github.com/Freemius/freemius-ai/tree/main/bolt-integration/subscription

Freemius' official repository also contains the specialized `freemius-core`, `freemius-checkout`, `freemius-customer-portal`, and `freemius-troubleshooting` skills. This Bolt skill consolidates the most important rules into one reusable Bolt.new skill and adds Bolt/Supabase-specific guardrails.
