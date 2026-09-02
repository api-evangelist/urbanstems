---
name: urbanstems-complete-purchase-safely
description: >-
  Complete an UrbanStems checkout with explicit buyer approval and an idempotency key, confirm the
  order, and correctly tell the buyer what can and cannot be undone afterwards.
api: UrbanStems Commerce MCP API
endpoint: https://274513-24.myshopify.com/api/ucp/mcp
transport: mcp
auth: buyer-approved payment instrument
operations:
  - get_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-09-02'
method: generated
source: >-
  mcp/urbanstems-ucp-mcp-tools.json (live tools/list, 2026-09-02),
  https://urbanstems.com/.well-known/ucp, https://urbanstems.com/agents.md,
  https://urbanstems.com/policies/refund-policy
---

# Complete an UrbanStems purchase safely

This is the only irreversible operation UrbanStems exposes. Treat it accordingly.

## The rule that comes before everything else

From the merchant's own agent instructions, verbatim:

> **Checkout requires human approval.** Agents must not complete payment without explicit buyer
> consent. If you cannot get contemporaneous buyer approval at the moment of payment, install
> https://shop.app/SKILL.md and route the purchase through Shop Pay instead.

"Contemporaneous" is the operative word. Consent gathered earlier in the conversation, or a standing
authorization, is not what this asks for. If the buyer is not present at the moment you call
`complete_checkout`, do not call it.

## 1. Read the checkout back first

`get_checkout` returns line items, totals, discounts and taxes. Read the **total** to the buyer in
major units — divide `amount` by 100 for USD/EUR — along with the recipient address and the delivery
method. Get their approval against those numbers.

## 2. Attach a payment instrument

Payment goes in `checkout.payment.instruments[]` on `update_checkout` or `complete_checkout`. Each
instrument requires `id`, `handler_id` and `type`.

The handlers **this merchant** accepts, from its `/.well-known/ucp`:

| `handler_id` | Handler | Notes |
|---|---|---|
| `gpay` | `com.google.pay` 2026-01-11 | Google Pay API v2. `PAN_ONLY` and `CRYPTOGRAM_3DS`. VISA, MASTERCARD, AMEX, DISCOVER. Billing address required, in FULL format, with phone number. Gateway `shopify`. |
| `shopify.card` | `dev.shopify.card` 2026-01-15 | visa, master, american_express, discover, diners_club |
| `shop_pay` | `dev.shopify.shop_pay` 2026-04-08 | Shop Pay, shop_id 69340168440 |

Do not offer a handler that is not in that list.

Schema rules the server enforces:

- If `handler_id` is `apple-pay`: `type` must be `card`, and both `billing_address` and `credential`
  are required, with `credential.type` = `apple_pay_token` and a `payment_data` object carrying
  `version`, `data`, `signature` and a `header` with `public_key_hash`, `transaction_id`, and one of
  `ephemeral_public_key` or `wrapped_key`.
- Otherwise `credential` requires `token` and `type`, where `type` is a namespaced dot-notation token
  type (e.g. `google.pay`, `merchant.token`).
- `display` (`brand`, `last_digits`, `expiry_month`, `expiry_year`, `funding_source`) is what you show
  the buyer. `expiry_month`/`expiry_year` are required for card instruments.

You never handle a raw card number. The instrument you attach is already a token produced by the
handler.

## 3. Complete — with an idempotency key

`complete_checkout` is the **one** tool that requires `meta.idempotency-key`:

```json
{
  "meta": {
    "ucp-agent": { "profile": "https://example.com/agent" },
    "idempotency-key": "<stable, unique-per-attempt key you generate and persist>"
  },
  "id": "gid://shopify/Checkout/…",
  "checkout": { "payment": { "instruments": [ … ] } }
}
```

Generate the key **before** the first attempt and persist it. On a timeout or network failure, retry
with **the same key** — that is what stops a double charge. A fresh key on retry defeats the entire
mechanism. Retention of the key server-side is not documented, so do not assume it is valid days
later.

## 4. Confirm — and check for errors inside the success

`complete_checkout` "Returns details about the completed checkout, including order ID, Thank You Page
URL, **or any errors encountered**." A 200 transport response is not a purchase. Inspect the payload:

- Order ID present (`gid://shopify/Order/…`) → the order exists. Confirm it with `get_order` and give
  the buyer the order ID and the Thank You Page URL.
- Errors present → the purchase did **not** complete. Do not tell the buyer it did.

## 5. What can be undone — say this accurately

Before payment:

- `cancel_checkout` cancels an uncompleted checkout.
- `cancel_cart` cancels a cart.

After payment: **nothing, in this API.** There is no refund, void, reverse or cancel-order tool. The
merchant's published refund policy states:

> Your purchase is final and nonrefundable. No Product may be returned or refunded except for
> Damaged Products.

The one exception, with a stated window:

- **Damaged Products — within 3 days of the original shipment date**, by returning the product or
  submitting a photograph. The remedy is a **replacement**; a refund of the original payment is at
  UrbanStems' discretion, and only where a replacement cannot be provided. The replacement is not
  guaranteed to contain the same contents or theme.
- Exchanges are **not** accepted.
- Prepaid and gift subscriptions **cannot** be cancelled or refunded.

This is a human support flow, not an API call: <https://help.urbanstems.com/en-US>.

Never tell a buyer an UrbanStems order can be cancelled or refunded after it is placed. Tell them
before they approve payment that it cannot.

## Errors and retries

- `429`: the endpoint is rate-limited per IP. Back off. No `Retry-After` header is published, and no
  numeric limit is documented.
- Transport failures on `complete_checkout`: retry with the **same** idempotency key.
- Transport failures on any other write: do **not** blind-retry — no other tool is idempotent. Read
  state back (`get_cart`, `get_checkout`) first.
