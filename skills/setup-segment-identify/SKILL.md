---
name: setup-segment-identify
description: Ensure the customer's app calls `analytics.identify(user_id, traits)` exactly once per new signup so Roadway can link anonymous page views to known users.
---

# Segment `identify()` for Roadway signup

Roadway's `stg__segment__users` model treats the **first `identify()` per `user_id`** as the signup timestamp. Without identify calls, there are no users in Roadway — only anonymous traffic.

## 1. Find the signup entry point

Grep for:

- `signup`, `sign_up`, `register`, `createAccount`, `createUser`
- Firebase: `createUserWithEmailAndPassword`
- Supabase: `supabase.auth.signUp`
- Clerk: `clerkClient.users.createUser`
- Auth0: `POST /dbconnections/signup`
- NextAuth: `events.createUser`

The target is the single path that runs **once per brand-new account**. Calling identify on every login is also fine — Segment dedupes — but signup is the cleanest hook.

## 2. Add `identify()`

**Client-side JS (preferred — Segment merges the browser's anonymousId automatically):**

```js
analytics.identify(user.id, {
  email: user.email,
  createdAt: user.createdAt,
});
```

**Server-side (Node):**

```js
import { Analytics } from "@segment/analytics-node";
const analytics = new Analytics({ writeKey: process.env.SEGMENT_WRITE_KEY });

analytics.identify({
  userId: user.id,
  anonymousId: req.cookies.ajs_anonymous_id,
  traits: { email: user.email, createdAt: user.createdAt },
});
```

**Server-side (Python):**

```python
import segment.analytics as analytics
analytics.write_key = os.environ["SEGMENT_WRITE_KEY"]

analytics.identify(
    user_id=user.id,
    anonymous_id=request.cookies.get("ajs_anonymous_id"),
    traits={"email": user.email, "created_at": user.created_at.isoformat()},
)
```

Server-side calls **must** pass `anonymousId` — otherwise pre-signup page views can't be stitched to the user. Read it from the `ajs_anonymous_id` cookie the Segment browser SDK sets (forwarded in the signup request, or read from cookies if the endpoint is same-origin).

## 3. `user_id` convention

See `../../references/required-ids-matrix.md` (`user_id` convention section). Same value must appear in GA4 and Stripe.

## Record progress

Track progress in the `## setup-segment-identify` section of `.roadway/onboarding.md` per `../../references/onboarding-state.md`. Code-only skill — list checkboxes directly under the section header.

Checkboxes to track:
- [ ] `analytics.identify(user_id, traits)` call added at the signup entry point (note the file/function).
- [ ] If server-side: `anonymousId` forwarded from the `ajs_anonymous_id` cookie.
- [ ] `user_id` confirmed to match the convention used in GA4 and Stripe.

## Verification

Create a test account on the customer's site. In Segment → Connections → Sources → Debugger, confirm:

- An `identify` event fires for the new account.
- It carries both `userId` and `anonymousId`.
- `anonymousId` matches the cookie value from the visitor's pre-signup page-view events on the same session.
