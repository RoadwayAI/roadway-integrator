---
name: setup-stripe-metadata
description: Populate Stripe Customer (and Checkout Session) metadata with the IDs Roadway needs to link subscription revenue back to identified users and anonymous visitors.
---

# Stripe metadata for Roadway revenue attribution

Roadway's `stg__stripe__customers` reads Stripe's `metadata` JSON and expects `user_id` plus at least one of: `anonymous_id`, `visitor_id`, `user_pseudo_id`, `distinct_id`.

Stripe metadata set **after** customer creation is not retroactively attributed — set it at creation time.

## 1. Find the customer creation call

Grep for:

- `stripe.Customer.create`
- `stripe.customers.create`
- `stripe.checkout.Session.create` / `stripe.checkout.sessions.create`

## 2. Add metadata at creation time

**Python:**

```python
stripe.Customer.create(
    email=user.email,
    metadata={
        "user_id": str(user.id),
        "anonymous_id": request.cookies.get("ajs_anonymous_id") or "",
        "user_pseudo_id": extract_ga_client_id(request.cookies.get("_ga")),
    },
)
```

**Node:**

```js
await stripe.customers.create({
  email: user.email,
  metadata: {
    user_id: String(user.id),
    anonymous_id: req.cookies.ajs_anonymous_id ?? "",
    user_pseudo_id: extractGaClientId(req.cookies._ga),
  },
});
```

`extractGaClientId` mirrors tag.js behavior — the GA4 pseudo id is the last two dot-separated segments of the `_ga` cookie:

```js
function extractGaClientId(gaCookie) {
  if (!gaCookie) return "";
  return gaCookie.split(".").slice(-2).join(".");
}
```

```python
def extract_ga_client_id(ga_cookie: str | None) -> str:
    if not ga_cookie:
        return ""
    return ".".join(ga_cookie.split(".")[-2:])
```

## 3. Stripe Checkout

If Checkout creates the Customer (rather than the app creating it first), set metadata on the session, the customer, and the subscription:

```js
await stripe.checkout.sessions.create({
  mode: "subscription",
  client_reference_id: String(user.id),
  customer_email: user.email,
  metadata: {
    user_id: String(user.id),
    user_pseudo_id: extractGaClientId(req.cookies._ga),
  },
  subscription_data: {
    metadata: { user_id: String(user.id) },
  },
  customer_creation: "always",
});
```

To also populate the Customer's own metadata, use a webhook on `checkout.session.completed` and call `stripe.customers.update(session.customer, { metadata })`. This is **only** required if downstream consumers read from `customers.metadata` rather than `checkout_sessions.metadata` — Roadway reads both, so webhook backfill is optional.

## 4. Reading cookies at checkout time

At checkout, the backend has `user.id` but `anonymous_id` / `user_pseudo_id` live in browser cookies. Options, in preference order:

1. Same-origin request → read cookies server-side directly (simplest).
2. Cross-origin → forward the cookie values in the checkout request body.
3. Forms → the Roadway tag-filled hidden inputs can be posted.

## `user_id` convention

See `../../references/required-ids-matrix.md` (`user_id` convention section). Same value must appear in Segment `identify()` and GA4 `user_id`.

## Record progress

Track progress in the `## setup-stripe-metadata` section of `.roadway/onboarding.md` per `../../references/onboarding-state.md`. Code-only skill — list checkboxes directly under the section header.

Checkboxes to track:
- [ ] `metadata` added to each Customer / Checkout Session creation call (note the files).
- [ ] `user_id` confirmed to match the convention used in Segment / GA4.
- [ ] At least one anonymous id (`anonymous_id` / `user_pseudo_id` / etc.) included alongside `user_id`.
- [ ] Checkout → Customer metadata backfill webhook added (only if the customer relies on `customers.metadata` rather than `checkout_sessions.metadata`).

## Verification

Create a test customer. Stripe Dashboard → Customers → [test customer] → scroll to "Metadata". Confirm `user_id` and at least one anonymous id are present.
