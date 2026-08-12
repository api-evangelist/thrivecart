---
name: thrivecart-subscribe-to-events
description: Create targeted ThriveCart event subscriptions, verify the deliveries you receive,
  and deduplicate them — without a list endpoint to fall back on.
api: thrivecart-api
base_url: https://thrivecart.com/api/external
operations:
  - ping
  - createEventSubscription
  - unsubscribeFromAnEvent
generated: '2026-08-12'
method: generated
source: asyncapi/thrivecart-events-asyncapi.yml + conventions/thrivecart-conventions.yml
---

# Subscribe to ThriveCart events

ThriveCart has **two** event surfaces with **different names for the same facts**. Pick one
deliberately.

| | Event Subscription API | Account-wide webhooks |
|---|---|---|
| Created by | `POST /subscribe` | UI: Settings > API & Webhooks |
| Encoding | JSON body | `x-www-form-urlencoded` |
| Scope | one event key, or `*` | everything in the account |
| Names | `order_created`, `order_refund_product` … | `order.success`, `order.refund` … |
| Limit | not published | 5 destinations by default |

Use the Event Subscription API when you want a filtered stream; it is the only programmatic option.

## Steps

1. `GET /ping` — confirm the token and account.

2. **Subscribe.** `POST /subscribe` with a JSON body:

   ```json
   {"event": "order_payment_product",
    "target_url": "https://yoursite.example/webhooks/tc/",
    "trigger_fields": {"mode_int": 2, "base_product": [5, 19, 30]}}
   ```

   `event` is one of the 21 keys (or `*` for all). `target_url` is your HTTPS endpoint.
   `trigger_fields` narrows delivery — every event page documents its own set.

   For an **OAuth application**, `target_url` must begin with a URL registered in your app's URL
   settings, exactly like an OAuth redirect URI, or the call errors. Account-wide API keys are
   exempt.

3. **Receive.** Deliveries arrive as HTTP POST with a JSON body. Answer 2xx.

4. **Verify authenticity.** The payload carries `thrivecart_secret` (your account's secret word,
   from Settings > API & Webhooks > ThriveCart order validation) and `thrivecart_account`.
   Compare `thrivecart_secret` against a hard-coded value on your side. There is **no HMAC
   signature header and no timestamp**, so a leaked secret is a permanent forgery capability —
   store it as a secret, and rotate it if it ever appears in a log.

5. **Deduplicate on `webhook_id`.** It is a UUID identifying that specific delivery and it is
   stable across ThriveCart's retries. `event_id` still exists for backward compatibility but is
   tied to the transaction record and may be missing on some event types — do not key on it.

6. **Unsubscribe.** `POST /unsubscribe` with the endpoint `url`. There is **no list endpoint**,
   so you must persist every `target_url` you register. If you lose that record you cannot
   enumerate or clean up your subscriptions through the API.

## The 21 event keys

`order_created` · `order_payment_product` · `order_payment_bump` · `order_payment_upsell` ·
`order_payment_downsell` · `order_rebill` · `order_rebill_failed` · `order_rebill_completed` ·
`order_rebill_cancelled` · `order_refund_product` · `order_refund_bump` · `order_refund_upsell` ·
`order_refund_downsell` · `subscription_paused` · `subscription_resumed` · `cart_abandoned` ·
`affiliate_approved` · `affiliate_rejected` · `affiliate_commission_earned` ·
`affiliate_commission_payout` · `affiliate_commission_refund`

## Trigger-field shapes

- `mode_int`: `1` test only, `2` live only
- `base_product`: numeric or array — products (payment/refund of a product, cart abandonment)
- `purchase.bump_id` / `purchase.upsell_id` / `purchase.downsell_id`: numeric or array
- `subscription.type`: `product` | `upsell` | `downsell`, then the matching
  `subscription.product_id` / `upsell_id` / `downsell_id`
- `product_id`: affiliate approval / rejection
- `affiliate_user_id`: affiliate commission events

## Things that will bite you

- Affiliate payouts only happen in **live** mode, so `affiliate_commission_*` never fires in test
  mode. You cannot test that path end to end.
- Retry behaviour is acknowledged but no schedule, backoff or attempt cap is published, and there
  is no delivery log or replay endpoint. Build your own dead-letter handling.
- Payload field schemas are not documented for any event. Capture a real delivery before writing
  a parser.
