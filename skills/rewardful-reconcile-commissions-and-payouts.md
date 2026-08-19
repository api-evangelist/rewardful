---
name: Reconcile Rewardful commissions and settle a payout
description: Walk due commissions for an affiliate, group them against a payout, mark the payout paid in one call, and confirm settlement.
api: https://api.getrewardful.com/v1
docs: https://developers.rewardful.com/rest-api/payouts
generated: '2026-08-14'
method: generated
source: >-
  Grounded in the Rewardful developer documentation (rest-api/commissions,
  rest-api/payouts). Rewardful publishes no OpenAPI, so steps name the documented HTTP
  method + path; every path, parameter and state value below appears verbatim in the docs.
operations:
  - GET /v1/commissions
  - GET /v1/commissions/:id
  - GET /v1/payouts
  - GET /v1/payouts/:id
  - PUT /v1/payouts/:id/pay
  - PUT /v1/commissions/:id
---

# Reconcile Rewardful commissions and settle a payout

## Before you start

- **This skill moves money-adjacent state.** `PUT /v1/payouts/:id/pay` marks a payout AND every commission inside it as paid. There is no sandbox and no idempotency key — treat it as a one-way action and require human confirmation before calling it.
- Auth: HTTP Basic, API Secret as username, empty password.
- Rate limit 45 req / 30s. A full commission walk on a large program will hit it — page deliberately.

## Steps

1. **List what is owed.** `GET /v1/commissions?state[]=due&affiliate_id=<uuid>&expand[]=sale`
   - `state` accepts `due`, `pending`, `paid`, `voided`; pass `state[]` repeatedly for several.
   - `limit` defaults to 25, maximum 100. Walk `pagination.next_page` until it is `null`; check `pagination.total_count` first to size the job.
   - Each commission carries `amount` (minor units), `currency`, `state`, `due_at`, `paid_at`, `voided_at`, plus embedded `campaign` and `sale`. The `sale` carries `charge_amount_cents`, `refund_amount_cents`, `tax_amount_cents` and `sale_amount_cents` — reconcile against those, not against `charge_amount_cents` alone, because refunds are netted by Rewardful.

2. **Find the payout that bundles them.** `GET /v1/payouts?affiliate_id=<uuid>&state[]=due&expand[]=commissions`
   - Payout states are `pending`, `due`, `processing`, `paid`.
   - A payout is a bundle of payable commissions for ONE affiliate. Confirm the commission ids in `payout.commissions[]` match the set you just reconciled before settling.

3. **Settle in one call.** `PUT /v1/payouts/:id/pay`
   - Returns the updated payout. The state in the response is usually **`processing`**, not `paid` — the settlement is queued. That is documented and normal; do not retry because you did not see `paid`.
   - Re-read with `GET /v1/payouts/:id` (or wait for the `payout.paid` webhook) to confirm final state.

4. **Only adjust individual commissions when you must.** `PUT /v1/commissions/:id` accepts `paid_at` and `due_at` as ISO 8601 strings. The docs explicitly say this is **no longer the preferred way to mark a commission paid** — it is still supported, but `PUT /v1/payouts/:id/pay` is the batch-correct path. Use the single-commission update only to re-date something.

5. **Deleting is permanent.** `DELETE /v1/commissions/:id` permanently deletes a commission and triggers recalculation of the affiliate's and campaign's financial stats. There is no undo and no soft-delete state.

## Watch for

- **No idempotency.** If `PUT /v1/payouts/:id/pay` times out, do NOT blind-retry — `GET /v1/payouts/:id` first and look for `processing`/`paid`.
- **Amounts are integers in minor units** with a sibling `currency`; never mix currencies when summing across a program.
- **Push confirmation** is available via the `commission.paid`, `payout.paid` and `payout.failed` webhook events. `payout.failed` is the only signal that a queued settlement did not complete — subscribe to it if you settle programmatically.

## Errors

`401` invalid secret · `404` unknown UUID for this account · `422` `{"error":..., "details":[...]}` validation · `429` rate limited, back off on the `RateLimit` headers.
