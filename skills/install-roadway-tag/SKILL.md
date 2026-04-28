---
name: install-roadway-tag
description: Add the Roadway analytics tag to a customer's site so GA4 pseudo ids, Segment anonymous ids, and HubSpot form submission metadata are captured on every page.
---

# Install the Roadway tag

Hosted at `https://analytics.roadwayai.com/tag.js`. Source: https://github.com/RoadwayAI/roadway-attribution-tag.

**Prerequisite:** the customer's website must have Segment or GA4 installed. The tag's job is to fill anonymous IDs into forms and pageview payloads that flow through those pipelines into Roadway. If the Detection pass found neither, **do not install the tag** — this plugin has no applicable work; direct the customer to their Roadway rep. The router enforces this; if you are running this skill standalone, confirm the prerequisite before touching code.

## What it does (don't reimplement)

- Fills any `input[name]` matching `segment_anon_id`, `segment_anon_id__c`, `ga4_pseudo_user_id`, `user_pseudo_id`, or `pseudo_user_id` with the respective id on each page.
- Listens for HubSpot `onFormSubmit` messages and fires a `hubspot_form_submission` GA4 event carrying `hubspot_form_id` + `hubspot_utk`.
- Listens for native `<form>` submit events and fires the same event.

The customer doesn't write any of that — they only need to load the tag and add the hidden inputs (other skills handle the inputs).

## Installation options

Pick exactly one. Default to **Option A** unless GTM is already the customer's standard.

### Option A — raw script tag

Add to the site's root HTML `<head>`:

```html
<script defer src="https://analytics.roadwayai.com/tag.js"></script>
```

Framework-specific targets:

| Framework | File | Snippet |
|---|---|---|
| Next.js App Router | `app/layout.tsx` | `import Script from "next/script"` then `<Script src="https://analytics.roadwayai.com/tag.js" strategy="afterInteractive" />` inside the body of `<html>` |
| Next.js Pages Router | `pages/_document.tsx` | inside `<Head>` |
| Vite / CRA / generic SPA | `index.html` or `public/index.html` | inside `<head>` |
| Remix | `app/root.tsx` | inside `<head>` alongside `<Meta />` `<Links />` |
| Nuxt 3 | `nuxt.config.ts` | `app.head.script: [{ src: "https://analytics.roadwayai.com/tag.js", defer: true }]` |
| SvelteKit | `src/app.html` | inside `<head>` |
| Plain static site | every HTML file (or shared partial) | inside `<head>` |

### Option B — Google Tag Manager

1. Tags → New → "Custom HTML".
2. Paste `<script defer src="https://analytics.roadwayai.com/tag.js"></script>`.
3. Trigger: All Pages.
4. Save + Publish.

Prefer Option B if every other tag on the site already flows through GTM.

## Do not

- Do not load the tag from anywhere except `analytics.roadwayai.com`.
- Do not self-host — it's maintained upstream.
- Do not wrap it in your own async loader; `defer` is all it needs.

## Record progress

Track progress in the `## install-roadway-tag` section of `.roadway/onboarding.md` per `../../references/onboarding-state.md`.

**External checkpoints to mark individually as the user confirms them:**
- GTM tag saved + published (only if Option B was used).

**Code edits:** a single rolled-up bullet once the `<script>` insertion is complete. Mention the file(s) touched (e.g. `app/layout.tsx`) and whether a CSP allowlist edit was needed.

## Verification

Tell the user to:

1. Open any page of their site.
2. Append `?roadway_debug=1` to the URL.
3. Open DevTools Console — they should see `[Roadway Tag] ...` messages.

The plugin does not start a dev server; verification happens in the customer's real browser.
