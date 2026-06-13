# Rewardful (rewardful)

Rewardful is a SaaS affiliate and referral tracking platform that enables companies to create, manage, and scale affiliate and customer referral programs tightly integrated with Stripe. It provides a REST API for programmatically managing campaigns, affiliates, referrals, commissions, and payouts, with zero percent transaction fees across all plan tiers.

APIs.json: https://raw.githubusercontent.com/api-evangelist/rewardful/refs/heads/main/apis.yml

Naftiko: https://github.com/naftiko/fleet?utm_source=api-evangelist&utm_medium=readme&utm_campaign=rewardful-api-evangelist&utm_content=repo

## Tags

- Affiliate Tracking
- Referral Programs
- SaaS
- Stripe
- Commissions
- Payouts

## APIs

- **Rewardful REST API** — REST API for managing campaigns, affiliates, referrals, commissions, payouts, and webhooks. Base URL: `https://api.getrewardful.com/v1`. Authentication via HTTP Basic Auth using the API Secret as the username.

## Plans / Rate Limits / FinOps

- **Plans:** Three tiers — Starter ($49/mo, up to $7,500 affiliate revenue), Growth ($99/mo, up to $15,000 affiliate revenue), Enterprise ($149+/mo, unlimited affiliate revenue). All plans include a 14-day free trial and zero percent transaction fees. Annual billing provides 2 months free.
- **Rate Limits:** 45 requests per 30-second window. HTTP 429 responses include RateLimit headers for backoff guidance.
- **FinOps:** Fixed monthly subscription with no per-request or per-transaction costs. Annual billing reduces spend by ~16.7%. Plan tier is governed by monthly affiliate revenue managed.

## Timestamps

- Created: 2026-06-13
- Modified: 2026-06-13

## Common

| Type | URL |
|------|-----|
| Website | https://www.rewardful.com/ |
| Documentation | https://developers.rewardful.com/ |
| GitHub Org | https://github.com/rewardful |
| LinkedIn | https://www.linkedin.com/company/rewardful/ |
| Blog | https://www.rewardful.com/articles |
| Pricing | https://www.rewardful.com/pricing |
| Status Page | https://status.rewardful.com/ |
| X | https://x.com/getrewardful |

## Maintainers

- Kin Lane — kin@apievangelist.com
