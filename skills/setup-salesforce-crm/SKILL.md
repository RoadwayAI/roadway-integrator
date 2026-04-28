---
name: setup-salesforce-crm
description: Create the Salesforce custom fields Roadway needs to link anonymous visitors to Contacts and Leads, and point the customer's forms or API integrations at them.
---

# Salesforce CRM setup for Roadway

Run after `install-roadway-tag`.

## 1. Create custom fields on Contact and Lead

Create on **both** Contact and Lead — leads convert to contacts and each keeps its own field.

| Field label | API name | Type | Length |
|---|---|---|---|
| GA4 Pseudo User ID | `ga4_pseudo_user_id__c` | Text | 255 |
| Segment Anonymous ID | `segment_anon_id__c` | Text | 255 |

UI path: **Setup → Object Manager → Contact → Fields & Relationships → New → Text**. Repeat for Lead.

No programmatic creation — Metadata API is out of scope here.

## 2a. Web-to-Lead forms

Inside the existing Web-to-Lead `<form>`:

```html
<!-- Tag fills these -->
<input type="hidden" name="segment_anon_id__c" />
<input type="hidden" name="ga4_pseudo_user_id" />

<!-- Salesforce field IDs (look these up in Setup → Object Manager → Lead → Fields → [field] → URL) -->
<input type="hidden" name="00Nxxxxxxxxxxxx" id="sf_segment_anon_id" />
<input type="hidden" name="00Nyyyyyyyyyyyy" id="sf_ga4_pseudo_user_id" />

<script>
  document.querySelector('form').addEventListener('submit', function () {
    document.getElementById('sf_segment_anon_id').value =
      document.querySelector('input[name="segment_anon_id__c"]').value;
    document.getElementById('sf_ga4_pseudo_user_id').value =
      document.querySelector('input[name="ga4_pseudo_user_id"]').value;
  });
</script>
```

Replace `00Nxxxxxxxxxxxx` / `00Nyyyyyyyyyyyy` with the real Salesforce field IDs.

## 2b. Custom backend → Salesforce via API

Find the place the code creates Lead/Contact records. Forward the tag-filled hidden inputs as custom field values.

**Python (`simple_salesforce`):**
```python
sf.Lead.create({
    "LastName": last_name,
    "Company": company,
    "Email": email,
    "ga4_pseudo_user_id__c": request.form.get("ga4_pseudo_user_id"),
    "segment_anon_id__c": request.form.get("segment_anon_id"),
})
```

**Node (`jsforce`):**
```js
await conn.sobject("Lead").create({
  LastName: lastName,
  Company: company,
  Email: email,
  ga4_pseudo_user_id__c: req.body.ga4_pseudo_user_id,
  segment_anon_id__c: req.body.segment_anon_id,
});
```

The signup form must include hidden inputs named `ga4_pseudo_user_id` and/or `segment_anon_id` (the Roadway tag only looks up those names). The backend forwards their values into the Salesforce payload.

## Record progress

Track progress in the `## setup-salesforce-crm` section of `.roadway/onboarding.md` per `../../references/onboarding-state.md`.

**External checkpoints to mark individually as the user confirms them:**
- Custom field `ga4_pseudo_user_id__c` created on Contact.
- Custom field `segment_anon_id__c` created on Contact.
- Custom field `ga4_pseudo_user_id__c` created on Lead.
- Custom field `segment_anon_id__c` created on Lead.
- Lead→Contact mapping added for each custom field (Setup → Object Manager → Lead → Map Lead Fields).

**Code edits:** one rolled-up bullet summarizing Web-to-Lead hidden-input additions and/or backend Salesforce API payload changes. Capture the real Salesforce field IDs (`00N…`) that replaced the placeholders.

## Verification

Submit a test form. In Salesforce, open the new Lead/Contact and confirm the custom field is populated. If the Lead is later converted to a Contact, confirm the custom field value carries over — Salesforce's default conversion mapping matches by API name, which is why both objects need the same field.
