---
name: setup-ga4-user-id
description: Ensure the customer's app sets GA4 `user_id` at signup/login so Roadway can stitch anonymous `user_pseudo_id` traffic to known users.
---

# GA4 `user_id` for Roadway signup

Roadway links GA4's anonymous `user_pseudo_id` to a known user via GA4's `user_id`. Without it, anonymous sessions stay anonymous forever.

## 1. Find the signup / login entry point

Same targets as `setup-segment-identify`. Prefer the **first post-signup render** (client-side) — that's where gtag runs with the session established.

## 2. Set `user_id`

Run in client-side JS once auth state is known:

```js
gtag('config', 'G-XXXXXXXXXX', {
  user_id: currentUser.id,
});
```

Or, if `config` already ran, use:

```js
gtag('set', { user_id: currentUser.id });
```

`G-XXXXXXXXXX` is the customer's GA4 Measurement ID — grep the repo for `G-` to find the existing call.

## 3. Fire a signup event (recommended)

```js
gtag('event', 'sign_up', {
  method: 'email',
  user_id: currentUser.id,
});
```

Only fire this on the **initial** signup, not every login.

## 4. Constraints on `user_id`

See `../../references/required-ids-matrix.md` (`user_id` convention section). Same value must appear in Segment `identify()` (if both exist) and Stripe `metadata.user_id`.

## GTM alternative

If the site tags via GTM rather than raw gtag, push to the data layer on signup:

```js
window.dataLayer.push({ event: 'login', user_id: currentUser.id });
```

Then in GTM, configure the GA4 configuration tag to read `user_id` from a Data Layer Variable.

## Record progress

Track progress in the `## setup-ga4-user-id` section of `.roadway/onboarding.md` per `../../references/onboarding-state.md`.

**External checkpoints to mark individually as the user confirms them:**
- GA4 Admin → Reporting Identity set to "Blended" or "By user-ID and device".
- GTM GA4 Configuration tag updated to read `user_id` from a Data Layer Variable, then saved + published (only if the GTM path was used).

**Code edits:** one rolled-up bullet summarizing `gtag('config', ..., { user_id })` / `gtag('set', ...)` / `dataLayer.push` additions. Note the signup file/function.

## Verification

1. GA4 → Admin → DebugView.
2. Trigger a signup in a test account with `?gtm_debug=1` appended (or Chrome GA Debugger extension).
3. Confirm:
   - `sign_up` event appears with `user_id` populated.
   - Subsequent events in the same session carry the same `user_id`.
   - `user_pseudo_id` stays stable across the anonymous → known transition.
