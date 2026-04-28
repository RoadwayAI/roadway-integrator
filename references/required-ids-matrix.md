# Required IDs for Roadway attribution

Canonical source: https://docs.roadwayai.com/required-ids-for-attribution

## Core identifiers

| ID | Who produces it | Where it must land |
|---|---|---|
| `anonymousID` / `visitor_id` | Roadway tag — reads `_ga` cookie (GA4) or `ajs_anonymous_id` (Segment) | Hidden form field → CRM custom property. |
| `userID` / `user_id` | Customer auth system at signup | Segment `identify()` or GA4 `user_id`. |
| `customerID` / `customer_id` | Billing provider (Stripe) | Stripe Customer id. |
| `contactID` / `contact_id` | CRM (HubSpot/Salesforce) | CRM contact id. |

## Per-source requirements

| Source | Required fields |
|---|---|
| Segment page events | `anonymous_id`, `original_timestamp`, `context_page_url`, optional `user_id` |
| Segment identifies | `user_id`, `anonymous_id`, `timestamp` (first identify per user_id = signup) |
| GA4 events export | `user_pseudo_id`, event timestamp, `page_location`, optional `user_id` |
| HubSpot contacts | `id`, `createdat`, custom property `ga4_pseudo_user_id` and/or `segment_anon_id` |
| HubSpot UTK mapping | `hubspot_utk`, `ga4_pseudo_user_id`, `hubspot_contact_id` (Roadway's backend maintains this via the `hubspot_form_submission` GA4 event the tag fires) |
| Salesforce Contact / Lead | `Id`, `CreatedDate`, custom fields `ga4_pseudo_user_id__c` and/or `segment_anon_id__c` |
| Stripe Customer | `id`, `created`, `metadata.user_id` + one of `anonymous_id`/`user_pseudo_id`/`visitor_id`/`distinct_id` |
| Google Ads | `campaign_id`, `segments_date`, `metrics_cost_micros`, `gclid` |
| Meta Ads | `campaign_id`, `date_start`, `spend`, `fbclid` |
| LinkedIn / TikTok / Reddit Ads | `campaign_id`, date, spend |

## `user_id` convention

The `user_id` passed to Segment `identify()`, GA4 `gtag('config', ..., { user_id })`, and Stripe `metadata.user_id` must be:

- **Identical across all three sources.** If they diverge, Roadway can't join across them.
- **Stable per account.** Set once at creation; never changes across sessions or devices.
- **Non-PII.** Use the internal database primary key as a string — never email or username.
- **≤ 255 chars.** (Segment caps at 255, GA4 at 256, Stripe metadata values at 500. 255 satisfies all three.)

## Stack → plugin actions

| Stack | Tag | CRM | Signup | Revenue |
|---|---|---|---|---|
| HubSpot + GA4 | raw script or GTM | HubSpot property `ga4_pseudo_user_id` (+ 3 GTM objects) | GA4 `user_id` | Stripe metadata |
| HubSpot + Segment | raw script | HubSpot property `segment_anon_id` | Segment `identify()` | Stripe metadata |
| Salesforce + GA4 | raw script | Contact/Lead `ga4_pseudo_user_id__c` | GA4 `user_id` | Stripe metadata |
| Salesforce + Segment | raw script | Contact/Lead `segment_anon_id__c` | Segment `identify()` | Stripe metadata |
| Custom CRM, no Stripe, or other missing first-class source | raw script (if analytics present) | — (skipped; customer's Roadway rep handles) | identify/user_id if a signup exists | — (skipped; rep handles) |
| No Segment and no GA4 | — (plugin exits; contact Roadway rep) | — | — | — |
