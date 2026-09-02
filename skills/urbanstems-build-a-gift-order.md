---
name: urbanstems-build-a-gift-order
description: >-
  Assemble an UrbanStems cart, attach the recipient's delivery destination, apply a discount code,
  and convert it into a priced checkout — all reversibly, without paying.
api: UrbanStems Commerce MCP API
endpoint: https://274513-24.myshopify.com/api/ucp/mcp
transport: mcp
auth: none
operations:
  - create_cart
  - update_cart
  - get_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
generated: '2026-09-02'
method: generated
source: mcp/urbanstems-ucp-mcp-tools.json (live tools/list, 2026-09-02)
---

# Build an UrbanStems gift order

Everything in this skill is reversible. No money moves until `complete_checkout`, which is a
different skill (`urbanstems-complete-purchase-safely`).

Every call carries `meta.ucp-agent.profile`.

## 1. Create the cart

`create_cart` requires `cart.line_items`. Each line item needs `quantity` and `item.id`, where
`item.id` is a **Product Variant** GID (`gid://shopify/ProductVariant/...`) — not a product GID.

```json
{
  "meta": { "ucp-agent": { "profile": "https://example.com/agent" } },
  "cart": {
    "line_items": [ { "quantity": 1, "item": { "id": "gid://shopify/ProductVariant/…" } } ],
    "buyer": { "email": "buyer@example.com" },
    "context": { "address_country": "US", "currency": "USD" }
  }
}
```

The returned cart ID has the form `gid://shopify/Cart/abc123?key=secret`. **The `key` is a
capability** — whoever holds that string can read and modify the cart. Do not log it, do not put it
in a URL you show the buyer, do not hand it to another agent.

`create_cart` is **not** idempotent and accepts no idempotency key. If a call times out, do not
blindly retry — you will create a second cart. Prefer `get_cart` on the ID you have; if you have no
ID, retry once and `cancel_cart` whichever cart you end up not using.

## 2. Set the recipient

This is a gifting store — the buyer and the recipient are usually different people. The recipient
address goes in `fulfillment`, not in `buyer`:

```json
"fulfillment": {
  "methods": [{
    "type": "shipping",
    "line_item_ids": ["…"],
    "destinations": [{
      "first_name": "…", "last_name": "…", "phone_number": "…",
      "street_address": "…", "extended_address": "…",
      "address_locality": "…", "address_region": "…",
      "postal_code": "…", "address_country": "US"
    }]
  }]
}
```

Then select it with `selected_destination_id`.

What this merchant supports, read from its own `/.well-known/ucp`:

- `method_combinations: [["shipping"]]` — **shipping only**. There is no pickup method. Do not offer one.
- `multi_destination: []` — **one destination per order**. A buyer sending to three recipients needs
  three orders; build three carts, not one cart with three destinations.
- `address_country` is ISO 3166-1 alpha-2.

## 3. Update, don't rebuild

`update_cart` takes the cart `id` plus a `cart` object. Note the replacement semantics:

- `discounts.codes` **replaces** the previously submitted codes. Send the full list every time; send
  `[]` to clear all codes. Codes are case-insensitive.
- Only prompt for a discount code if the buyer mentions having one. The contract says so explicitly.
- Line items are addressed by their line-item `id` for updates.

`update_cart` is self-reversing: re-send the previous values to undo. Nothing here is permanent.

## 4. Attribution

If the buyer arrived through a campaign or referral, pass `attribution` (`referring_domain`,
`utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term`, `click_id_tag`/`click_id_value`).
It is forwarded to the cart for the merchant's reporting. Do not fabricate values — omit the block
if you do not genuinely have them.

## 5. Convert to a checkout

Two options:

- **From the cart** — `create_checkout` with `checkout.cart_id`. The cart's `line_items`, `context`
  and `buyer` win; overlapping fields in the checkout payload are ignored. This is the option to
  prefer, because it keeps one source of truth.
- **Direct** — `create_checkout` with `checkout.line_items`, skipping the cart entirely.

The checkout response is where you get authoritative **totals, taxes and applicable discounts**.
Quote those to the buyer, not the sum of the line prices — and remember every amount is minor units
paired with a currency code (`{"amount": 2500, "currency": "USD"}` is $25.00).

`create_checkout` is also not idempotent. If you end up with a duplicate, call `cancel_checkout` on
the one you are abandoning.

## 6. Reversal

| You did | Undo with | Window |
|---|---|---|
| `create_cart` | `cancel_cart` | any time before payment |
| `create_checkout` | `cancel_checkout` | any time before `complete_checkout` |
| `update_cart` / `update_checkout` | re-send prior values | while the object is open |

After `complete_checkout` there is **no reversal in this API at all**. Read
`urbanstems-complete-purchase-safely` before you go further.

## Hand-off

Confirm the full picture with `get_checkout` — line items, totals, discounts, taxes, delivery
destination — and read it back to the buyer before asking for payment approval.
