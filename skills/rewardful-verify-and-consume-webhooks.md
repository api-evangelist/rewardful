---
name: Verify and consume Rewardful webhooks
description: Stand up a Rewardful webhook receiver that verifies the HMAC-SHA256 signature, handles the 33 documented event types, and survives the 3-day retry window.
api: https://api.getrewardful.com/v1
docs: https://developers.rewardful.com/webhooks/overview
generated: '2026-08-14'
method: generated
source: >-
  Grounded in the Rewardful developer documentation (webhooks/overview, webhooks/endpoints,
  webhooks/requests, webhooks/event-types, webhooks/signed-webhooks) and the captured
  catalog in asyncapi/rewardful-webhooks.yml. Rewardful publishes no AsyncAPI document.
operations:
  - POST <your endpoint> (inbound from Rewardful)
  - GET /v1/commissions/:id
  - GET /v1/payouts/:id
---

# Verify and consume Rewardful webhooks

## Setup

Webhook endpoints are configured in the dashboard at `https://app.getrewardful.com/webhooks`, not through the API — there is no REST endpoint for managing webhook subscriptions. Your endpoint must be **HTTPS** and must return **`200`**; any other status is a failure.

Select only the event types you need. There are 33 across seven object families: `affiliate.*` (4), `affiliate_link.*` (3), `affiliate_coupon.*` (5), `referral.*` (4), `sale.*` (4), `commission.*` (5), `payout.*` (6). The full list is in `asyncapi/rewardful-webhooks.yml`.

## Payload shape

Every delivery is a JSON POST with three root keys:

- `object` — the record that triggered the event, structurally identical to the REST representation of that object type
- `event` — `{ id, type, created_at, api_version }`
- `request` — `{ id }`, the only correlation identifier Rewardful publishes

## Steps

1. **Read the raw body first.** The signature is computed over the **raw request body**, so capture it before any JSON parsing or middleware re-serialization.

2. **Verify `X-Rewardful-Signature`.** Compute `HMAC-SHA256(raw_body, endpoint_signing_secret)` as a hex digest and compare it to the header. The Signing Secret is per-endpoint, from the dashboard Webhooks page. Reject with `401` if the header is missing or does not match. Use a constant-time comparison.

3. **Know what the signature does NOT cover.** There is no timestamp and no nonce in the scheme, so it proves authorship but not freshness — a captured delivery can be replayed. Defend yourself: deduplicate on `event.id`, and ignore events whose `event.created_at` is far in the past.

4. **Ack fast, work later.** Return `200` as soon as you have verified and enqueued. Anything else is treated as a failure and Rewardful retries with exponential backoff for **up to 3 days**, then abandons the delivery.

5. **Assume at-least-once, unordered delivery.** No ordering guarantee is documented, and the retry policy makes duplicates normal. Make handlers idempotent on `event.id`, and treat `object` as a snapshot — when the money matters, re-read the authority with `GET /v1/commissions/:id` or `GET /v1/payouts/:id` instead of trusting the payload.

6. **Route the events that matter.**
   - Partner lifecycle: `affiliate.created` (no welcome email is sent for API-created affiliates — send yours here), `affiliate.confirmed`, `affiliate.updated`, `affiliate.deleted`
   - Attribution: `referral.created`, `referral.lead`, `referral.converted`
   - Revenue: `sale.created`, `sale.refunded`, `commission.created`, `commission.voided`
   - Settlement: `payout.due`, `payout.paid`, **`payout.failed`** — the only signal that a queued settlement did not complete

## Gotchas

- The `sale` object has no REST collection and no object-reference page; `sale.*` events are the only place you see it standalone.
- `event.api_version` is `"v1"` and there is no published API changelog, so pin your parsing defensively and tolerate unknown fields.
