---
name: thrivecart-refund-a-purchase
description: Find a ThriveCart transaction from a customer's email or order id and refund the
  correct item on it, safely, on an API that has no idempotency key.
api: thrivecart-api
base_url: https://thrivecart.com/api/external
operations:
  - ping
  - searchTransactions
  - readCustomerInformation
  - refundATransaction
generated: '2026-08-12'
method: generated
source: openapi/thrivecart-api-openapi.yml + conventions/thrivecart-conventions.yml + errors/thrivecart-problem-types.yml
---

# Refund a ThriveCart purchase

Refunds move money and **cannot be reversed**. ThriveCart's API accepts no `Idempotency-Key`
header, so a retry after a network timeout can refund twice. Read before you write, every time.

## Before you start

- You need a bearer token: an account API key (Settings > API & webhooks > API tokens) or an
  OAuth access token. Send it as `Authorization: Bearer <token>` on every call.
- Confirm which account the token resolves to with `ping`. The response gives `account_name`,
  `account_id` and `account_version`, and the response headers echo `X-ThriveCart-Account-Name`.
  Getting this wrong means refunding a stranger's customer.
- The same token reads both test and live data. There is no test-only credential. Check
  `mode_int` (1 = test, 2 = live) on whatever you are about to act on.

## Steps

1. **Validate the token.** `GET /ping`. A `401` with `{"error":"auth.missing"}` means no
   Authorization header was sent; `{"error":"invalid_token"}` means the token is expired or
   revoked.

2. **Find the transaction.** `GET /transactions` with `query` set to the customer's email or the
   order id, `transactionType=charge` (or `rebill` for a recurring payment), `page=1` and
   `perPage` up to 100. If you only have an email and want the customer record first, call
   `readCustomerInformation` — it is a `POST` that performs a read and is safe to retry.

3. **Identify the exact item.** An order can contain a product, a bump, an upsell and a downsell.
   `refundATransaction` refunds one item, addressed by `reference` in the composite form
   `<type>-<id>` — for example `product-299`. Do not guess the type: read it from the transaction
   you just fetched.

4. **Confirm, then refund once.** `POST /refund` with `order_id` and `reference` as
   `application/x-www-form-urlencoded`. Record the request locally *before* sending it, keyed on
   `order_id` + `reference`, so a crash mid-flight does not become a second refund.

5. **On timeout, do not retry.** Re-run step 2 and check whether the refund landed. There is no
   idempotency key to protect you, and no request id to correlate on.

6. **Verify.** Re-run `GET /transactions` with `transactionType=refund` and the same `query`.

## Errors you will actually hit

| Status | Body | What to do |
|---|---|---|
| 401 | `{"error":"auth.missing"}` | Add the `Authorization: Bearer` header |
| 401 | `{"error":"invalid_token"}` | Re-authorise; regenerate the API key |
| 400 | validation | Check `order_id` and the `reference` format |
| 404 | not found | The order is not in the account this token resolves to |

ThriveCart does not use RFC 9457 problem details. Errors are plain JSON with an `error` key, in
two inconsistent naming styles (`auth.missing`, `invalid_token`). Do not pattern-match on one form.

## Rate limits

60 requests per minute per account, shared across everything. No `RateLimit-*` or `Retry-After`
headers are returned, and the exhaustion status code is undocumented — budget against the number
rather than reacting to a signal.
