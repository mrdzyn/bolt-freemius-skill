# Bolt.new Freemius Integration Skill

Use this skill in **Bolt.new** when you want to build or improve an app that integrates with **Freemius**.

## What this skill helps with

- Freemius SDK installation and setup guidance
- License activation/deactivation flows
- Plan and feature gating logic
- Subscription and billing event handling
- Webhook verification and processing
- Debugging Freemius API and integration issues

## Best use inside Bolt.new

When using this skill, describe:

1. Your app stack (e.g. Node.js, PHP, WordPress plugin, etc.)
2. Your current Freemius setup status
3. The exact outcome you need (e.g. "activate paid plan after checkout")
4. Any errors or logs you already have

## Suggested prompt in Bolt.new

```text
Use the Freemius integration skill to help me implement licensing and subscription checks in my app.
My stack is <your stack>. I need <goal>. Current issue: <error/details>.
Please provide step-by-step implementation guidance and code updates.
```

## Typical integration checklist

- Create/configure your product in Freemius
- Add Freemius SDK/client to your app
- Connect authentication and product identifiers
- Implement license and subscription status checks
- Gate premium features by plan/license state
- Add webhook handlers for subscription lifecycle events
- Test sandbox and production flows

## Notes

- Keep your Freemius credentials and secrets out of source control.
- Validate webhook signatures before processing events.
- Prefer server-side entitlement checks for security-critical flows.

---

If you are building in Bolt.new, attach your current integration code and the skill can guide targeted implementation updates.
