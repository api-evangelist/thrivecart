---
name: thrivecart-onboard-an-affiliate
description: Create a ThriveCart affiliate, register and approve them for products, set custom
  commissions, and track the three different identifiers that all refer to the same person.
api: thrivecart-api
base_url: https://thrivecart.com/api/external
operations:
  - ping
  - searchAffiliates
  - readAffiliateInfo
  - createNewAffiliate
  - registerAffiliateForAProduct
  - approveAnAffiliateForAProduct
  - specifyCustomCommissions
  - markAffiliateAsFavorite
generated: '2026-08-12'
method: generated
source: openapi/thrivecart-api-openapi.yml + data-model/thrivecart-data-model.yml
---

# Onboard a ThriveCart affiliate

The affiliate surface is the richest part of the ThriveCart API — nine write operations — and the
one with the most identifier confusion. Get the identifiers straight first.

## Three identifiers, one affiliate

| Field | What it is | Where it appears |
|---|---|---|
| `affiliate_id` | the vanity/link id. **"may be updated by the system"** on creation | path param on `/affiliates/{affiliate_id}/…` |
| `affiliate_user_id` | the numeric user id | event trigger fields (`affiliate_commission_*`) |
| `email` | sign-in address | `createNewAffiliate`, `readAffiliateInfo` |

`readAffiliateInfo` (`POST /affiliate`) accepts **any of the three** in its `affiliate_id` field —
it is the disambiguator. Because `affiliate_id` can be rewritten by ThriveCart at creation time,
never assume the id you requested is the id you got: read it back.

## Steps

1. `GET /ping` — confirm the token and account.

2. **Check for an existing affiliate.** `GET /affiliates` with `query` set to the name, email or
   affiliate id, `page=1`, `perPage` up to **25** (not 100 — this endpoint's cap is lower than
   `/transactions`).

3. **Create.** `POST /affiliates` (urlencoded) with `email` and `product_ids` (an array of at
   least one product id). Optional: `name`, `first_name`, `last_name`, `company`, `country`,
   `affiliate_id` (desired), `auto_approve`, `parent_affiliate`. Returns **201**.

4. **Read back the real id.** `POST /affiliate` with the email. Persist the `affiliate_id` you
   get back, not the one you asked for.

5. **Register for more products.** `POST /affiliates/{affiliate_id}/register` with `product_ids`,
   plus optional `auto_approve`, `trigger_emails` (defaults to true) and `parent_affiliate`.
   Set `trigger_emails=false` for bulk migrations so you do not mail a thousand people.

6. **Approve or reject.** `POST /affiliates/{affiliate_id}/approve` or `.../reject` with
   `product_ids` and optional `trigger_emails`. Skip this if you set `auto_approve` in step 3 or 5.

7. **Custom commissions.** `POST /affiliates/{affiliate_id}/custom_commissions` with `product_id`
   and `commission_object`. Pass `null` as `commission_object` to remove custom commissions for
   that product. This changes payout economics — treat it as a consequential write.

8. **Optional: flag.** `POST /affiliates/{affiliate_id}/favorite` / `.../unfavorite`.

## Verify the money side

Commission events fire only in **live** mode:

```json
{"event": "affiliate_commission_earned",
 "target_url": "https://yoursite.example/webhooks/tc/",
 "trigger_fields": {"affiliate_user_id": [93810, 145813]}}
```

Note the trigger field is `affiliate_user_id`, not `affiliate_id` — the third identifier again.

To see the commission rates an affiliate would earn on an offer, call
`GET /products/{product_id}/pricing_options?affiliate_id=<id>` — the `affiliate_id` query
parameter returns commissions tailored to that specific affiliate.

## Cautions

- `POST /affiliates/{affiliate_id}/delete` is destructive and has no confirmation step or
  idempotency key.
- `POST /affiliate` and `GET /affiliates` are reads; everything else here writes.
- 60 requests per minute per account. Bulk onboarding a large affiliate list will hit it — pace
  at roughly one call per second and expect no `Retry-After` to tell you when to resume.
