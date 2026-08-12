---
name: thrivecart-manage-a-subscription
description: Pause, resume or cancel a ThriveCart subscription when the API gives you no way to
  read one — recovering subscription_id from transactions or webhooks first.
api: thrivecart-api
base_url: https://thrivecart.com/api/external
operations:
  - ping
  - searchTransactions
  - pauseASubscription
  - resumeASubscription
  - cancelASubscription
generated: '2026-08-12'
method: generated
source: openapi/thrivecart-api-openapi.yml + data-model/thrivecart-data-model.yml + asyncapi/thrivecart-events-asyncapi.yml
---

# Manage a ThriveCart subscription

The hard part is not the write — it is that **ThriveCart has no subscription read endpoint**.
`subscription_id` is required by all three state-change operations and returned by none of them.
Plan for that before you start.

## Where subscription_id comes from

Only two places:

1. **`GET /transactions`** — search with the customer email or order id and
   `transactionType=rebill`. The transaction record carries the subscription context, and since
   April 2026 it also carries `subscription_current_status` (`active`, `paused`, `cancelled`,
   `completed`).
2. **Webhook payloads** — `order_rebill`, `order_rebill_failed`, `subscription_paused`,
   `subscription_resumed`, `order_rebill_cancelled`. If you operate an integration, persist
   `order_id` + `subscription_id` from these on arrival. That is the durable path; searching
   transactions is the recovery path.

## Steps

1. `GET /ping` — confirm the token and the account it resolves to.

2. Recover `order_id` and `subscription_id` as above. Check
   `subscription_current_status` first: pausing an already-paused subscription or cancelling a
   completed one is a wasted, unprotected write.

3. **Pause** — `POST /pauseSubscription` with `order_id`, `subscription_id`, and optionally
   `auto_resume`, a Unix timestamp that **must be at least 24 hours in the future**. A nearer
   timestamp is rejected.

4. **Resume** — `POST /resumeSubscription` with `order_id` and `subscription_id`.

5. **Cancel** — `POST /cancelSubscription` with `order_id` and `subscription_id`. This is
   irreversible; there is no un-cancel operation. If the customer might return, prefer a pause.

6. **Verify by event, not by read.** There is no subscription read. Subscribe to
   `subscription_paused`, `subscription_resumed` and `order_rebill_cancelled` and treat the event
   as your confirmation, or re-run `GET /transactions` and inspect
   `subscription_current_status`.

## Subscribing to the confirming events

```
POST /subscribe
{"event": "subscription_paused", "target_url": "https://yoursite.example/webhooks/tc/",
 "trigger_fields": {"mode_int": 2, "subscription": {"type": "product", "product_id": 5}}}
```

For an OAuth application, `target_url` must begin with a URL registered in your app settings —
the same rule as OAuth redirect URIs. It does not apply to account-wide API keys.

Deduplicate deliveries on `webhook_id`, a UUID that is stable across ThriveCart's retries of the
same delivery. Do not use `event_id`; it is tied to the transaction record and can be absent.

## Cautions

- No `Idempotency-Key`. A retried cancel is a second cancel attempt.
- One credential covers test and live. Check `mode_int` before acting.
- 60 requests per minute per account, with no rate-limit headers to read.
