---
name: roadway-integrator
description: Onboard a codebase to Roadway's attribution pipeline. Detect the customer's CRM, analytics, auth, and billing stack, then dispatch to focused sub-skills that install the Roadway tag, create CRM fields, instrument signup, and populate Stripe metadata.
---

# Roadway Integrator

Use this skill when a user asks to "set up Roadway", "integrate with Roadway attribution", or "onboard to Roadway".

This skill is a router. Its job is to figure out which sub-skills apply, and in what order:

- `install-roadway-tag` — add the `//analytics.roadwayai.com/tag.js` script.
- `setup-hubspot-crm` — HubSpot properties + hidden form fields + optional GTM wiring.
- `setup-salesforce-crm` — custom fields on Contact/Lead.
- `setup-segment-identify` — `analytics.identify()` at signup.
- `setup-ga4-user-id` — GA4 `user_id` at signup.
- `setup-stripe-metadata` — IDs on Stripe Customer metadata.

If the customer's site lacks the analytics pipeline this plugin requires, direct them to their Roadway rep.

## Resume check (do this first)

Before anything else, check whether this codebase has been onboarded in a prior session.

1. Look for `.roadway/onboarding.md` at the repo root.
2. **File absent** → go to Detection pass.
3. **File present** → parse it (format spec: `./references/onboarding-state.md`):
   - Read the `## Detected stack` block and print it verbatim. Skip the Detection pass entirely — treat this as source of truth.
   - Collect every `## <skill-name> — <status>` section. Print a two-part summary:
     ```
     Already done:
       - install-roadway-tag (2026-04-23)
       - setup-hubspot-crm (2026-04-23)
     In progress:
       - setup-ga4-user-id
         [ ] `gtag('config', ..., { user_id })` at signup
     ```
   - Ask: `Continue from here? (yes / redo <skill-name> / start over)`.
4. Handle the answer:
   - **yes** → skip every `done` skill during Dispatch; resume every `in progress` skill from its first unchecked item.
   - **redo <skill-name>** → delete that section from `.roadway/onboarding.md`, then continue. The skill will run from scratch.
   - **start over** → delete `.roadway/onboarding.md` entirely, then go to Detection pass.

## Detection pass (first run only)

Skip this step if `.roadway/onboarding.md` already contained a `## Detected stack` block — the resume check printed it.

Run these greps in parallel. Do not edit anything until the user has confirmed the detected stack.

1. **CRM**
   - HubSpot: `@hubspot/api-client`, `hubspot-api-python`, `hbspt.forms.create`, `js.hsforms.net`, `_hsenc=`.
   - Salesforce: `jsforce`, `simple_salesforce`, `salesforce-bulk`, `Web-to-Lead`, `salesforce.com/servlet/servlet.WebToLead`.
   - Neither found → no CRM-specific skill runs.

2. **Analytics**
   - Segment: `@segment/analytics-next`, `analytics.load`, `analytics.identify(`, `cdn.segment.com/analytics.js`.
   - GA4: `gtag(`, `googletagmanager.com/gtag/js?id=G-`, `G-[A-Z0-9]+`.
   - GTM: `googletagmanager.com/gtm.js`, `GTM-[A-Z0-9]+`.

   **GTM disambiguation.** GTM loads analytics at runtime from its own admin UI, invisible to source-code greps. If GTM is detected but neither Segment nor GA4 appears in source, do not classify analytics yet. Ask the user:

   > GTM is installed (GTM-XYZ), but neither GA4 nor Segment shows up in your source code. GTM usually loads one of them at runtime. Is GA4 or Segment configured inside your GTM container?
   >
   > - If yes, share the GA4 measurement ID (`G-…`) or confirm Segment is the one inside — we'll proceed as if that source is present.
   > - If no, this plugin has no work to do here — contact your Roadway rep.

   Record the answer in the Analytics row below:
   - GA4 via GTM → `Analytics: GA4 via GTM (GTM-XYZ, G-ABC123)` — proceed as if GA4 is present.
   - Segment via GTM → `Analytics: Segment via GTM (GTM-XYZ)` — proceed as if Segment is present.
   - Neither inside GTM → `Analytics: none (GTM detected but empty per user)` — the no-analytics short-circuit below will fire.

3. **Auth / signup entry point**
   - `analytics.identify(`, `setUserId(`, `signUp`, `register`, `createUser`, Firebase `createUserWithEmailAndPassword`, Supabase `auth.signUp`, Clerk `users.createUser`, Auth0 `/dbconnections/signup`, NextAuth `events.createUser`.
   - Target the function that runs **once** per brand-new account. That's where identify/setUserId must fire.

4. **Payments**
   - Stripe: `stripe.Customer.create`, `stripe.customers.create`, `stripe.checkout.Session.create`, `@stripe/stripe-js`, env var `STRIPE_SECRET_KEY`.

5. **HTML host (for tag install)**
   - Next.js App Router → `app/layout.tsx`
   - Next.js Pages → `pages/_document.tsx`
   - Vite/CRA/SPA → `index.html` or `public/index.html`
   - Remix → `app/root.tsx`
   - Nuxt → `nuxt.config.ts`
   - SvelteKit → `src/app.html`
   - Static site → every `*.html`

Report findings as a short table:

```
CRM:        HubSpot (found `hbspt.forms.create` in 3 files)
Analytics:  GA4 (G-ABC123), no Segment
Auth:       signup at src/server/auth/signup.ts:42
Payments:   Stripe (stripe.Customer.create in src/billing/customer.ts:88)
HTML host:  Next.js App Router (app/layout.tsx)
```

If any of the five rows is ambiguous, ask the user one question before moving on. If two identity layers coexist (Segment + GA4), prefer Segment — it's the lower-friction identity path.

Once the user confirms the detected stack, write `.roadway/onboarding.md` with the `# Roadway onboarding` header, the intro paragraph, and the `## Detected stack` block. Format spec: `./references/onboarding-state.md`. Do not add sub-skill sections yet — each skill writes its own.

## Dispatch order

### Pre-check: no-analytics short-circuit

If the Detection pass (including the GTM disambiguation answer, if it applied) concluded that **neither Segment nor GA4 is carrying analytics — whether directly in source or via GTM** — stop and do not run any of the per-skill steps below. The Roadway tag, CRM hidden-field setup, identity instrumentation, and Stripe metadata all depend on Segment or GA4 being present to carry anonymous IDs across page loads and into form submissions / cookies. Without either, this plugin has no work to do.

Print to the user and then exit:

> Your codebase has no Segment or GA4 integration, so none of the sub-skills apply. The Roadway tag needs an analytics pipeline to flow through, and the CRM / Stripe / identity skills all depend on anonymous IDs the tag would carry.
>
> **Contact your Roadway rep** — they will set up your integration.

Do not dispatch any sub-skill. Do not update `.roadway/onboarding.md` beyond the `## Detected stack` block already written.

### Regular path

Run in this order when at least one of Segment or GA4 is present. Skip any step whose prerequisites aren't met. On resume, skip every skill whose `## <skill-name>` section in `.roadway/onboarding.md` is `done`, and resume every `in progress` skill from its first unchecked item.

1. `install-roadway-tag` — unblocks every downstream step.
2. CRM-specific: `setup-hubspot-crm` OR `setup-salesforce-crm`. If neither CRM is in use, skip this step.
3. Identity: `setup-segment-identify` OR `setup-ga4-user-id`. Requires an auth/signup entry point — if detection found none, skip (static content sites have nothing to instrument).
4. `setup-stripe-metadata` if Stripe is in use. If not, skip.
5. End with the verification checklist below.

Every sub-skill is expected to update its own section in `.roadway/onboarding.md` as it runs (see each skill's "Record progress" section and the format spec in `./references/onboarding-state.md`).

## Final verification checklist

Before declaring the integration done, confirm:

- [ ] `<script defer src="https://analytics.roadwayai.com/tag.js"></script>` loads on every page (test with `?roadway_debug=1`).
- [ ] Every lead/signup form has the right hidden input(s): `ga4_pseudo_user_id` and/or `segment_anon_id` (and the `__c` variants for Salesforce).
- [ ] CRM has the custom property/field created and it's marked as a form-visible field.
- [ ] `analytics.identify(user_id, ...)` or `gtag('config', ..., { user_id })` fires exactly once per new account.
- [ ] New Stripe customers carry `metadata.user_id` + at least one anonymous id.
- [ ] The user has shared the `user_id` / `visitor_id` / `customer_id` conventions with their Roadway contact.

## References

Shared docs (read on demand, don't inline):

- `./references/required-ids-matrix.md` — which IDs each source needs and where they land.
- `./references/tag-source.md` — exactly what `tag.js` does and doesn't do.
- `./references/onboarding-state.md` — `.roadway/onboarding.md` format and resume semantics.

External docs:

- https://docs.roadwayai.com/required-ids-for-attribution
- https://docs.roadwayai.com/required-ids-for-attribution
- https://github.com/RoadwayAI/roadway-attribution-tag (public tag source)
