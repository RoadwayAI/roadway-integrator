---
name: setup-hubspot-crm
description: Create the HubSpot contact properties and hidden form fields Roadway needs to link anonymous visitors to CRM contacts. Also wire GTM when the customer uses GA4 with HubSpot embedded forms.
---

# HubSpot CRM setup for Roadway

Run after `install-roadway-tag`.

## 1. Create HubSpot contact properties

Add whichever applies (both if the customer uses both analytics sources):

| Property label | Internal name | Type | Hidden |
|---|---|---|---|
| GA4 Pseudo User ID | `ga4_pseudo_user_id` | Single-line text | Yes |
| Segment Anonymous ID | `segment_anon_id` | Single-line text | Yes |

UI path: **Settings → Properties → Contact properties → Create property**.

If the user wants automation, ask for a HubSpot private app token with `crm.schemas.contacts.write` and POST to `https://api.hubapi.com/crm/v3/properties/contacts`:

```json
{
  "name": "ga4_pseudo_user_id",
  "label": "GA4 Pseudo User ID",
  "type": "string",
  "fieldType": "text",
  "groupName": "contactinformation",
  "formField": true
}
```

Otherwise print the UI steps and move on — don't block on a token they might not have.

## 2. Add hidden fields to HubSpot embedded forms

For each HubSpot form that captures leads/signups:

**Marketing → Forms → [form] → Add field → select the new property → set visibility to Hidden → Publish.**

Repeat for every form. The Roadway tag writes to these hidden fields at page load; HubSpot submits them as contact properties.

## 3. Native `<form>` elements POSTing to the customer's backend

If the customer posts to their own backend and then creates the HubSpot contact via API, add these hidden inputs to the form markup:

```html
<input type="hidden" name="segment_anon_id" />
<input type="hidden" name="ga4_pseudo_user_id" />
```

In the backend call to HubSpot, forward the submitted values as properties:

```ts
await hubspotClient.crm.contacts.basicApi.create({
  properties: {
    email,
    firstname,
    lastname,
    segment_anon_id: req.body.segment_anon_id,
    ga4_pseudo_user_id: req.body.ga4_pseudo_user_id,
  },
});
```

## 4. GA4 + HubSpot forms → GTM wiring (optional)

Only if the customer uses **GA4 + HubSpot embedded forms** and needs the `hubspot_form_submission` event in GA4.

In GTM, create three objects:

| Name | Type | Config |
|---|---|---|
| `Roadway Custom Event - hubspot_form_submission` | Trigger → Custom Event | Event name: `hubspot_form_submission` |
| `DL - hubspot_utk` | Variable → Data Layer Variable | Variable name: `hubspot_utk` |
| `Roadway GA4 - HubSpot Form Submission` | Tag → GA4 Event | GA4 config tag: the existing one. Event name: `hubspot_form_submission`. Event parameter: `hubspot_utk` = `{{DL - hubspot_utk}}`. Trigger: the custom event above. |

Save + Publish the container.

## Record progress

Track progress in the `## setup-hubspot-crm` section of `.roadway/onboarding.md` per `../../references/onboarding-state.md`.

**External checkpoints to mark individually as the user confirms them** (include the HubSpot portal ID on each):
- Property `ga4_pseudo_user_id` created in HubSpot portal &lt;portal-id&gt; (if applicable).
- Property `segment_anon_id` created in HubSpot portal &lt;portal-id&gt; (if applicable).
- Each embedded form updated: "Form '&lt;name&gt;' set new property to Hidden and published."
- GTM objects saved + published (only if section 4 ran).

**Code edits:** one rolled-up bullet summarizing native `<form>` hidden input additions and any backend HubSpot API call changes.

## Verification

Submit a test form on the customer's site. In HubSpot, open the new contact and confirm the `ga4_pseudo_user_id` / `segment_anon_id` property is populated. For the GTM path: in GA4 DebugView, confirm the `hubspot_form_submission` event appears.
