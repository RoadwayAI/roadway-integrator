# What `//analytics.roadwayai.com/tag.js` does

Upstream: https://github.com/RoadwayAI/roadway-attribution-tag
Local copy (same behavior): `webapp/public/tag.js` in the Roadway backend repo.

## Fills hidden form inputs

Every 1.5 seconds after page load, the tag looks up these input names and fills them:

| Input name | Source |
|---|---|
| `segment_anon_id` | `window.analytics.user().anonymousId()` |
| `segment_anon_id__c` | same |
| `ga4_pseudo_user_id` | last two dot-separated segments of the `_ga` cookie |
| `user_pseudo_id` | same |
| `pseudo_user_id` | same |

The interval retries forever, so lazy-mounted forms are handled automatically.

## Listens for HubSpot form submissions

`window.addEventListener("message", ...)` for messages where `event.data.type === "hsFormCallback"` and `event.data.eventName === "onFormSubmit"`. On match, fires a GA4 event:

```
gtag('event', 'hubspot_form_submission', {
  hubspot_form_id,
  hubspot_utk
});
```

If `gtag` isn't available it falls back to `dataLayer.push({ event: 'hubspot_form_submission', ... })`.

Host-specific quirk: for hostnames containing `canibuild.com`, `hubspot_form_id` is replaced with the `hubspotutk` cookie value.

## Listens for native form submissions

A global `submit` listener fires the same `hubspot_form_submission` event for non-HubSpot `<form>` elements. This is how Roadway still captures conversion signals from custom-built forms.

## Debug mode

Append `?roadway_debug=1` to any URL — logs `[Roadway Tag] ...` messages to the console.

## Validates GA4 configuration

5 seconds after load, the tag scans `window.dataLayer` for a `config` entry whose id starts with `G-`. Logs success or failure — useful only when `roadway_debug=1` is set.

## What the tag does NOT do

- Does not create CRM properties or fields.
- Does not call `analytics.identify()` or `gtag('config', ..., { user_id })`.
- Does not instrument Stripe customers.
- Does not write to the customer's backend.

Those gaps are what the other skills in this plugin close.
