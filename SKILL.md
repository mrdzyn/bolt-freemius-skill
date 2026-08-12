---
name: freemius-billing-bolt
description: "Use when adding, changing, debugging, or reviewing Freemius monetization in a Bolt.new SaaS or web app. Covers Freemius SaaS/App billing, subscriptions, one-time purchases and top-up credits, pricing, secure checkout, entitlement/paywall logic, Supabase/Bolt Cloud persistence and RLS, purchase redirects, webhooks, Customer Portal, upgrades/cancellations, invoices, sandbox testing, and optional license-key activation. Uses the Freemius JavaScript/TypeScript SDK and SaaS/App integration patterns; do not substitute the WordPress SDK unless the target is actually a WordPress plugin or theme."
---

# Freemius Billing & Monetization for Bolt.new

Use this skill whenever a Bolt.new project needs Freemius for SaaS/App monetization, subscriptions, one-time purchases, top-up credits, software licensing, pricing pages, checkout, premium feature gating, subscription management, billing history, or webhook-driven entitlement synchronization.

Treat monetization and entitlement code as security-sensitive. Freemius owns the commercial/billing state; the Bolt application mirrors only the state needed to decide what the authenticated user may access.

## 1. Use the correct Freemius product path

For Bolt.new SaaS/web apps:

- Prefer the current **Freemius SaaS & Apps** integration and JavaScript/TypeScript SDK.
- Primary server SDK: `@freemius/sdk`.
- Checkout UI package when needed: `@freemius/checkout`.
- Schema validation package used by the current Freemius SDK examples: `zod`.
- Freemius React Starter Kit can be used when compatible with the project, but preserve the project's existing design system when possible.
- Do **not** install or imitate the Freemius WordPress SDK unless the target being built is actually a WordPress plugin/theme.
- If an API, SDK method, event name, or checkout behavior may have changed, check Freemius' current documentation before coding.

Official references:

- AI Skills: `https://freemius.com/help/documentation/ai/skills.md`
- AI/LLM index: `https://freemius.com/help/llms.txt`
- SaaS integration: `https://freemius.com/help/documentation/saas/saas-integration/`
- JS SDK integration: `https://freemius.com/help/documentation/saas-sdk/js-sdk/integration/`
- JS SDK installation: `https://freemius.com/help/documentation/saas-sdk/js-sdk/installation/`
- Webhooks: `https://freemius.com/help/documentation/saas-sdk/js-sdk/webhooks/`
- License activation: `https://freemius.com/help/documentation/saas/integrating-license-key-activation/`
- Official Freemius AI repository: `https://github.com/Freemius/freemius-ai`

Freemius currently publishes four complementary AI skills: `freemius-core`, `freemius-checkout`, `freemius-customer-portal`, and `freemius-troubleshooting`. This Bolt skill combines their important architecture rules with Bolt/Supabase-specific guidance.

## 2. Inspect the Bolt project before modifying it

Before writing Freemius code, inspect the existing project and determine:

1. Frontend framework and version, usually React/Vite in Bolt projects.
2. Whether the project uses Bolt Cloud, Supabase, or another backend.
3. Authentication provider and the authoritative server-side user identifier.
4. Database schema, migrations, RLS policies, and existing account/profile tables.
5. Existing server/edge functions and their runtime: Node.js, Deno, Bun, or another Fetch-compatible runtime.
6. Any existing Freemius implementation, billing tables, pricing page, paywall, or account/billing page.
7. Existing `.env.example` or secret names without exposing secret values.
8. Product monetization model: subscription, one-time/lifetime, top-up credits, or a combination.
9. Which routes/actions/features must be protected.

Reuse the existing authentication, database, and routing architecture. Do not create parallel auth or duplicate billing systems when suitable structures already exist.

## 3. Detect Freemius configuration instead of hard-coding it

If the project already has Freemius environment variables, use them. If API credentials are available server-side, use the Freemius SDK/API to retrieve product plans and pricing rather than asking the user to paste plan IDs that can be resolved automatically.

Treat the Freemius Dashboard as the commercial source of truth for:

- product
- plans
- billing cycles
- prices
- plan feature lists
- one-time/top-up offerings
- license quotas/units

Do not infer plan IDs from display text in the frontend. Do not scatter plan/pricing IDs across components.

## 4. Secrets and environment variables

Freemius secret credentials must be stored in **Bolt Secrets**, Supabase Edge Function secrets, or the deployment platform's server-side secret store.

Recommended server configuration:

```text
FREEMIUS_PRODUCT_ID=
FREEMIUS_PUBLIC_KEY=
FREEMIUS_SECRET_KEY=
FREEMIUS_API_KEY=
FREEMIUS_SANDBOX=true
PUBLIC_APP_URL=
```

Rules:

- `FREEMIUS_SECRET_KEY` is server-only.
- `FREEMIUS_API_KEY` / bearer token is server-only.
- Never put secret values in `VITE_*`, `NEXT_PUBLIC_*`, browser code, logs, sample data, screenshots, or committed files.
- Do not commit `.env` containing secrets.
- Keep sandbox/live credentials and product configuration clearly separated.
- Never silently continue if required billing configuration is missing.

### Sandbox rule

Use a dedicated `FREEMIUS_SANDBOX` flag.

**Never derive Freemius sandbox mode from `NODE_ENV`.** A Bolt deployment may have `NODE_ENV=production` while still intentionally testing Freemius sandbox checkout. Default development/testing to sandbox and switch to live only deliberately.

Example:

```ts
export const IS_FREEMIUS_SANDBOX =
  process.env.FREEMIUS_SANDBOX !== 'false';
```

Adapt environment access to Deno (`Deno.env.get`) when using Supabase Edge Functions.

## 5. Install the Freemius SDK in the correct runtime

For Node/standard package-manager projects, current Freemius documentation uses:

```bash
npm install @freemius/sdk @freemius/checkout zod
```

Only install `@freemius/checkout` if the application actually needs the browser checkout package.

For Supabase/Deno Edge Functions, use the runtime-supported npm import style where appropriate, for example:

```ts
import { Freemius } from 'npm:@freemius/sdk';
```

Verify package/runtime compatibility instead of forcing Node-only patterns into an edge runtime.

Create one reusable server-side Freemius client module, not multiple ad-hoc instances.

Conceptual example:

```ts
import { Freemius } from '@freemius/sdk';

export const freemius = new Freemius({
  productId: process.env.FREEMIUS_PRODUCT_ID!,
  apiKey: process.env.FREEMIUS_API_KEY!,
  secretKey: process.env.FREEMIUS_SECRET_KEY!,
  publicKey: process.env.FREEMIUS_PUBLIC_KEY!,
});
```

Keep Freemius business logic in server-side service modules. Routes/edge functions should mainly authenticate, validate input, call a service, and shape the response.

## 6. Entitlement database model

For Bolt Cloud/Supabase, create or adapt a local entitlement mirror. A common name is:

`user_fs_entitlement`

Recommended fields include:

```text
id
user_id
fs_license_id      UNIQUE
fs_plan_id
fs_pricing_id
fs_user_id
type               subscription | lifetime/one_time
expiration          nullable
is_canceled
created_at
updated_at
```

Add fields if current SDK output or your product requirements need them, but avoid copying the entire Freemius object into your database.

### Supabase RLS

Enable RLS.

Required policy intent:

- authenticated users may read only their own entitlement rows when frontend reading is necessary;
- users may not insert, update, or delete their own billing entitlement rows;
- entitlement writes come only from trusted server/edge functions using appropriate server credentials;
- server authorization must not rely on the frontend's RLS-visible row alone for sensitive premium operations.

Do not expose a Supabase service-role key in browser code.

## 7. Freemius is the billing source of truth

The local entitlement table is a synchronized mirror/cache, not the authoritative billing provider.

When processing a purchase or license update:

1. Receive a trusted license/purchase identifier from a verified Freemius flow.
2. Retrieve the authoritative purchase/license details through the Freemius SDK/API.
3. Convert/map the result into the application's entitlement record.
4. UPSERT by a stable unique Freemius identifier such as `fs_license_id`.
5. Never grant a paid tier solely from client-submitted plan, price, license, or success data.

Where supported by the current SDK, use the purchase/entitlement helper methods rather than rebuilding Freemius billing logic manually.

## 8. Pricing page

The pricing page should reflect the plans and prices configured in Freemius.

Preferred architecture:

```text
Browser pricing UI
       ↓
GET /api/checkout/pricing (or Bolt/Supabase equivalent)
       ↓
Trusted server calls Freemius
       ↓
Return display-safe plans/pricing/features + server-created checkout context
```

Rules:

- Do not call secret-authenticated Freemius APIs directly from the browser.
- Prefer retrieving current plans/prices from Freemius rather than duplicating them in frontend constants.
- Support the monetization modes actually present in the product.
- Subscriptions may have monthly/annual cycles.
- One-time/top-up plans should not be forced into subscription semantics.
- Use Freemius plan feature lists as inputs to entitlement/paywall decisions where applicable.
- Keep pricing display and checkout plan selection consistent.

## 9. Secure checkout creation

Create Freemius checkout server-side for the **authenticated user**.

Recommended flow:

```text
Authenticated user clicks Subscribe
        ↓
POST server/edge checkout endpoint
        ↓
Resolve authenticated user's email/id server-side
        ↓
Validate requested plan against approved Freemius product plans
        ↓
freemius.checkout.create(...)
        ↓
checkout.serialize()
        ↓
Return hosted link and/or safe checkout options
        ↓
Browser opens Freemius Checkout
```

Conceptual SDK pattern:

```ts
const checkout = await freemius.checkout.create({
  user: {
    email: authenticatedUser.email,
    firstName,
    lastName,
  },
  planId,
  isSandbox: IS_FREEMIUS_SANDBOX,
});

const { options, link } = checkout.serialize();
```

Rules:

- Never trust an arbitrary email supplied by the browser when an authenticated email is already available.
- Validate requested plan IDs/pricing IDs against the current Freemius product configuration.
- Do not expose API/secret keys to create checkout in the browser.
- Prevent duplicate purchase actions caused by double clicks.
- Prefer Freemius' current checkout mechanisms instead of building your own payment form.

## 10. Hosted redirect and overlay completion are different flows

Do not collapse different checkout completion contracts into one handler.

If using hosted checkout return/redirection:

- use the Freemius SDK's current redirect-processing method for the signed redirect URL;
- re-fetch the authoritative purchase;
- synchronize the local entitlement;
- redirect the user to an appropriate success/account page.

If using overlay checkout:

- the checkout-completed callback should POST its relevant license/purchase identifier to a dedicated authenticated backend endpoint;
- the backend must re-fetch the purchase from Freemius before granting access;
- do not treat the overlay's client-side completion callback as the entitlement authority.

Keep hosted redirect handling and overlay purchase synchronization separate when their payload contracts differ.

## 11. Webhooks are mandatory for reliable lifecycle sync

A checkout success path only handles the initial purchase. Webhooks are required to keep access correct when billing changes later.

Create a dedicated public Freemius webhook endpoint, such as:

```text
POST /api/webhooks/freemius
```

or the appropriate Supabase Edge Function URL.

Use the Freemius SDK's webhook listener/processor and its documented authentication method. Do not invent custom signature verification when the SDK already supports the correct Freemius authentication flow.

Typical license lifecycle events the app may need include:

```text
license.created
license.updated
license.extended
license.shortened
license.cancelled
license.expired
license.plan.changed
license.quota.changed
license.deleted
```

Use only event names supported by the current SDK/docs.

For relevant events:

- retrieve/re-sync the authoritative purchase/license where appropriate;
- UPSERT the entitlement;
- delete or invalidate the mirror when the lifecycle event requires it;
- make handlers idempotent;
- return success only after the request is authenticated and processed safely.

### Raw-body warning

Do not pre-parse or mutate webhook request data in a way that breaks Freemius SDK webhook authentication. Hand the request to the current SDK processor in the form it expects.

## 12. Entitlement resolver

Centralize premium access checks.

Create reusable helpers such as:

```text
getUserEntitlement(userId)
requireEntitlement(userId, feature?)
requirePlan(userId, planKey)
requireCredits(userId, quantity)
```

The exact names may differ, but all protected backend actions should use the same trusted entitlement logic.

Important rules:

- enforce entitlements server-side;
- UI hiding/disabled buttons are UX only;
- if access is missing, return a consistent response such as HTTP `402 Payment Required` or an application-standard entitlement error;
- frontend may show a paywall when it receives that response;
- never grant access because `localStorage`, a React state variable, or client JWT custom field says `premium=true` unless that value is itself issued and refreshed from trusted server billing state.

### Subscriptions vs one-time/lifetime

Do not assume every valid entitlement is a recurring subscription. If the Freemius product sells lifetime/one-time access, ensure the resolver includes valid one-time entitlements and does not accidentally filter them out by using a subscription-only helper.

## 13. Quantitative limits and plan features

For plans that differ by limits such as:

- projects
- websites/domains
- seats
- storage
- videos
- exports
- API calls
- AI generations
- credits

store/derive the entitlement limit from a trusted plan mapping or Freemius plan feature configuration.

Enforce hard limits in backend/database logic where bypass would have commercial impact.

Frontend limit meters are informational; they are not authorization.

Unknown/unrecognized plan IDs must not fall back to the highest tier.

## 14. Top-up credits / usage credits

If the product sells one-time top-up credits:

- treat the Freemius payment/purchase as the funding event;
- maintain a separate application-side credit balance or, preferably, a credit ledger;
- credit the account only from verified Freemius purchase state;
- make credit grants idempotent so a webhook retry cannot double-credit the user;
- debit credits inside a trusted transaction around the metered action;
- never let the browser directly increment its credit balance;
- keep subscription entitlement and consumable credit balance as separate concepts.

For auditability, prefer a ledger similar to:

```text
credit_transactions
- id
- user_id
- source_type        purchase | usage | adjustment | refund
- source_id          UNIQUE where applicable
- amount_delta
- balance_after      optional
- created_at
```

## 15. Customer Portal

Prefer Freemius' Customer Portal / current React Starter Kit components for post-purchase billing management instead of rebuilding subscription billing logic.

The customer portal can cover:

- current subscription
- upgrades/change plan
- cancellation
- retention coupon when available
- billing information
- payment history
- invoices

### Security boundary

- Require an authenticated user before retrieving portal data.
- Resolve the Freemius user from the authenticated account server-side.
- Do not accept an arbitrary Freemius user ID from the browser as authorization.
- Freemius credentials remain server-side.
- Use signed portal action URLs/processors provided by Freemius where available.

For the current JS SDK, the headless customer portal pattern uses one backend endpoint, commonly:

```text
GET|POST /api/portal
```

with `freemius.customerPortal.request.createProcessor(...)` or the current documented equivalent.

The browser should call the application's authenticated portal endpoint, not Freemius secret APIs directly.

## 16. Authenticated Bolt/Supabase function calls

When the frontend calls an authenticated Supabase Edge Function, attach the active user's access token.

Conceptual pattern:

```ts
const {
  data: { session },
} = await supabase.auth.getSession();

const response = await fetch(edgeFunctionUrl, {
  headers: {
    Authorization: `Bearer ${session?.access_token}`,
  },
});
```

For invoices/files returned through authenticated functions, fetch the file with authorization, convert it to a Blob, then open/download the object URL. Do not rely on a raw `<a href>` to an endpoint that requires an Authorization header.

## 17. Upgrades, downgrades, and cancellation

Use Freemius-supported subscription/Customer Portal flows instead of directly mutating local entitlement fields.

Rules:

- verify the signed-in user's ownership before any billing mutation;
- prefer a license-upgrade checkout or the current Freemius recommended upgrade flow;
- do not create a second concurrent subscription when the intent is to upgrade the existing license/subscription;
- reflect cancellations in local access according to the actual license expiration/state, not merely the timestamp when the user clicked Cancel;
- when Freemius offers a cancellation retention coupon, allow the Customer Portal flow to present it before final cancellation;
- webhook updates must ultimately reconcile the local mirror.

## 18. Optional license-key activation for downloadable apps/extensions

Use this section only when the Bolt project is the backend/control plane for a desktop app, browser extension, or SaaS flow that explicitly uses Freemius license keys.

Freemius supports license activation/deactivation/validation for SaaS & Apps. Follow the current official guide rather than inventing a key format.

Rules:

- never embed Freemius secret/API credentials in a distributable browser extension or desktop binary;
- validate license ownership/status using Freemius-supported APIs/SDK patterns;
- store only the minimum license metadata the product needs;
- account for activation quotas;
- deactivate installations when the app supports device/site deactivation;
- do not treat possession of a syntactically valid-looking key as proof of entitlement;
- if SaaS license keys are meant to be shown to customers, the Freemius Dashboard setting must explicitly allow it.

## 19. Refunds and revoked access

If a refund, chargeback, deletion, expiration, downgrade, or other event changes what the user is entitled to:

- let Freemius billing/license state drive the decision;
- update/reconcile the local mirror from trusted server processing;
- reverse consumable credits when the product's business rules require it and the refund can be uniquely tied to a credited purchase;
- never leave permanent premium access merely because the original checkout succeeded once.

## 20. Bolt/Supabase security checklist

Before considering the integration complete, verify:

- [ ] Freemius secret/API keys exist only in Bolt/Supabase server secrets.
- [ ] No secret is exposed through `VITE_*`, public config, browser bundles, or logs.
- [ ] `FREEMIUS_SANDBOX` is independent from `NODE_ENV`.
- [ ] The pricing page gets trusted/current Freemius plan data.
- [ ] Checkout is created server-side for the authenticated user.
- [ ] Plan IDs are validated.
- [ ] Checkout success re-fetches authoritative purchase information.
- [ ] Local entitlement rows are server-written.
- [ ] RLS blocks users from editing their own entitlement state.
- [ ] Protected backend routes enforce entitlement independently of the UI.
- [ ] Webhook endpoint is public and configured in the Freemius Dashboard.
- [ ] Webhook authentication is handled with the current Freemius SDK/docs.
- [ ] Lifecycle events re-sync/correct entitlements.
- [ ] Customer Portal identifies users from authenticated server context.
- [ ] Billing actions cannot target another user's Freemius account/license.
- [ ] Quantitative plan limits are enforced server-side.
- [ ] Top-up credits are idempotent and cannot be double-granted.
- [ ] Sandbox purchase was tested end-to-end.
- [ ] Cancellation/expiration was tested, not just initial purchase.
- [ ] Invoice/payment history access requires authentication.
- [ ] Live configuration was not enabled accidentally during testing.

## 21. Dashboard handoff Bolt cannot perform automatically

When code is ready, clearly tell the developer which Freemius Dashboard configuration still needs human setup. Common tasks include:

1. Product → Plans: confirm plans, prices, billing cycles, features, and top-up/lifetime products.
2. Settings → API & Keys: copy the official JS SDK `.env` values into Bolt/Supabase server secrets.
3. Checkout & Redirection / plan customization: configure the deployed checkout return URL if the chosen flow requires it.
4. Webhooks: register the deployed public Freemius webhook endpoint and required events.
5. SaaS license keys: enable customer license-key visibility only if the product actually needs exposed license keys.

Never ask the user to paste secret values into source code or into a public chat if Bolt Secrets can be used instead.

## 22. Testing strategy

Test in sandbox before live production.

Minimum end-to-end cases:

1. Free/no-entitlement user sees the paywall.
2. Protected backend request is denied even if the frontend is manipulated.
3. Pricing displays the intended Freemius plans.
4. Checkout opens in sandbox mode.
5. Successful purchase creates/synchronizes entitlement.
6. Page refresh/login on another browser still recognizes entitlement from the server.
7. Webhook retry does not duplicate entitlement or credits.
8. Upgrade/downgrade changes the effective plan correctly.
9. Cancellation preserves or removes access according to actual Freemius license state/expiration.
10. Expiration/revocation removes access.
11. Customer Portal opens only for the signed-in user.
12. Payment history/invoices cannot be retrieved cross-account.
13. One-time/lifetime entitlement works if offered.
14. Top-up credits are credited exactly once if offered.
15. Unknown plan/pricing IDs fail closed.

Freemius also documents testing via sandbox/test cards. A 100% coupon may be useful for specific live-style checkout testing, but it is not a substitute for sandbox and must not be confused with sandbox mode.

## 23. Troubleshooting order

When Freemius "payment worked but access didn't," diagnose in this order:

1. Was the checkout actually sandbox or live as intended?
2. Did the successful checkout return/overlay handler run?
3. Did the backend obtain a license ID/purchase identifier?
4. Could the backend retrieve authoritative purchase information from Freemius?
5. Was `user_fs_entitlement` UPSERTed for the correct authenticated user?
6. Did RLS/service-role configuration block the write?
7. Does `getUserEntitlement()` handle the entitlement type being sold?
8. Is the premium backend route calling the entitlement guard?
9. Is the Freemius webhook URL public and correctly configured?
10. Is the webhook being authenticated/processed with the request format expected by the SDK?
11. Are sandbox/live product IDs and secrets mixed?
12. Is a subscription upgrade accidentally creating a second subscription instead of upgrading the existing license?

Use current Freemius troubleshooting documentation/official `freemius-troubleshooting` skill when an SDK-specific error remains unclear.

## 24. Keep the integration maintainable

After implementation:

- centralize Freemius client/configuration;
- centralize entitlement synchronization;
- centralize access-control helpers;
- keep route handlers thin;
- avoid plan IDs spread throughout UI code;
- document which premium routes/features are gated;
- document which environment variables are required without recording secret values;
- add or update a concise `AGENTS.md`/project instruction section describing where checkout, entitlement sync, webhooks, paywall guards, and Customer Portal code live.

Do not overwrite unrelated developer instructions in an existing `AGENTS.md`.

## 25. Completion report

When Bolt finishes a Freemius integration, report:

1. Files/components/functions added or changed.
2. Database migration/table/RLS changes.
3. Secrets/environment variable names required, without exposing values.
4. Freemius Dashboard steps still required.
5. Monetization modes implemented: subscription / one-time / credits / license activation.
6. Premium features/routes now enforced server-side.
7. Sandbox tests completed and their results.
8. Remaining production-readiness risks or untested lifecycle cases.

Never claim the billing integration is production-ready if checkout, entitlement persistence, webhook lifecycle sync, and server-side premium enforcement have not all been tested.
