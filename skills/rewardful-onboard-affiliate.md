---
name: Onboard an affiliate into a Rewardful campaign
description: Create an affiliate on a specific campaign, give them a trackable link (and optionally a coupon), then hand them a one-time SSO magic link into their dashboard.
api: https://api.getrewardful.com/v1
docs: https://developers.rewardful.com/rest-api/overview
generated: '2026-08-14'
method: generated
source: >-
  Grounded in the Rewardful developer documentation (rest-api/affiliates,
  rest-api/campaigns, rest-api/affiliate-links, rest-api/affiliate-coupons). Rewardful
  publishes no OpenAPI, so steps name the documented HTTP method + path rather than an
  operationId; every path and parameter below appears verbatim in the docs.
operations:
  - GET /v1/campaigns
  - POST /v1/affiliates
  - POST /v1/affiliate_links
  - POST /v1/affiliate_coupons
  - GET /v1/affiliates/:id/sso
---

# Onboard an affiliate into a Rewardful campaign

## Before you start

- **Auth on every call**: HTTP Basic, API Secret as the **username**, empty password — `-u YOUR_API_SECRET:`. There is no OAuth and no scoping; this secret is full account access.
- **Bodies are form-encoded**, responses are JSON. IDs are bare UUIDs.
- **No idempotency key exists.** A retried `POST /v1/affiliates` creates a SECOND affiliate. Before any retry, re-list and check whether the first attempt landed.
- **Rate limit**: 45 requests / 30 seconds per account, `429` on exhaustion with `RateLimit` headers. Sleep between bulk creates.
- **No sandbox.** Every call hits production data.

## Steps

1. **Pick the campaign.** `GET /v1/campaigns` returns all campaigns; each carries `id`, `name`, `reward_type`, `commission_percent` / `commission_amount_cents`, and `days_before_referrals_expire`. Note the `id` of the campaign you are onboarding into. If you omit `campaign_id` in step 2, the affiliate lands in the account's `default: true` campaign — decide deliberately rather than by omission.

2. **Create the affiliate.** `POST /v1/affiliates` with form fields:
   - Required: `first_name`, `last_name`, `email`
   - Recommended: `campaign_id` (the UUID from step 1), `token` (the `?via=` code — letters, numbers and dashes only)
   - Optional: `state` (defaults to `active`), `paypal_email`, `stripe_customer_id` for a customer-referrer who should receive Stripe account credits (the customer must exist in your Stripe account in livemode)

   On success you get the affiliate object including `links[]` with the generated link `token` and `url`.

   **Rewardful sends no welcome email and requires no email confirmation for API-created affiliates.** If the partner needs a welcome, send it yourself, or drive it from the `affiliate.created` webhook.

3. **Add extra links if the program needs them.** `POST /v1/affiliate_links` with `affiliate_id` and a `token`. Creating additional affiliate links beyond the first is a **Growth/Enterprise plan feature** — on Starter this call will fail; read the `422` `details[]` before retrying.

4. **Add a coupon for coupon-based attribution (optional).** `POST /v1/affiliate_coupons` assigns a payment-processor coupon/promotion code to the affiliate. The returned `external_id` (e.g. `promo_...`) is the Stripe-side identifier Rewardful sets automatically.

5. **Hand off with SSO, on demand only.** `GET /v1/affiliates/:id/sso` returns a one-time magic URL into the affiliate dashboard. It **expires after one minute**, cannot be reused, and issuing a new one invalidates all prior links. Never embed it in an HTML page or email — fetch it when the user clicks and immediately redirect.

## Error handling

| Status | Meaning | What to do |
|---|---|---|
| `401` | `{"error": "Invalid API Secret."}` | Secret missing, wrong, or sent as the password. Re-send as basic-auth username. |
| `404` | Object not found | The UUID does not belong to this account. Re-list to resolve. |
| `422` | `{"error": "...", "details": ["Email can't be blank"]}` | Read every string in `details[]`, fix the form fields, resubmit — after confirming nothing partial was created. |
| `429` | Rate limited | Back off using the `RateLimit` headers; throttle bulk onboarding. |

## Verify

`GET /v1/affiliates/:id?expand[]=campaign&expand[]=links` and confirm the campaign binding and link token are what you intended. Subscribe to `affiliate.created` and `affiliate_link.created` webhooks (HMAC-SHA256 in `X-Rewardful-Signature`) if you need confirmation pushed to you.
