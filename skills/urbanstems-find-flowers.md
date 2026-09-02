---
name: urbanstems-find-flowers
description: >-
  Search the UrbanStems catalog for bouquets, plants and gifts, narrow by price and availability,
  and quote prices to a buyer without getting the currency wrong.
api: UrbanStems Commerce MCP API
endpoint: https://274513-24.myshopify.com/api/ucp/mcp
transport: mcp
auth: none
operations:
  - search_catalog
  - lookup_catalog
  - get_product
generated: '2026-09-02'
method: generated
source: mcp/urbanstems-ucp-mcp-tools.json (live tools/list, 2026-09-02)
---

# Find flowers at UrbanStems

Read-only. Nothing in this skill creates a cart, moves money, or changes anything at the merchant.

## Before you start

Resolve the endpoint from the merchant profile, not from the llms.txt:

```
GET https://urbanstems.com/.well-known/ucp
```

Read `ucp.services["dev.ucp.shopping"][0].endpoint`. As of 2026-09-02 that is
`https://274513-24.myshopify.com/api/ucp/mcp`. The endpoint printed in `/llms.txt` and `/agents.md`
(`https://urbanstems.com/api/ucp/mcp`) returns **404** — do not use it.

No credential is needed. But **every** tool call requires the agent-identity block:

```json
"meta": { "ucp-agent": { "profile": "<your agent profile URI>" } }
```

Omit it and the call fails schema validation.

## 1. Search

Call `search_catalog`. At least one of `catalog.query` or `catalog.filters` must be present.

```json
{
  "meta": { "ucp-agent": { "profile": "https://example.com/agent" } },
  "catalog": {
    "query": "peonies",
    "context": { "address_country": "US", "currency": "USD" },
    "filters": { "price": { "min": 4000, "max": 12000 }, "available": true },
    "pagination": { "limit": 10 }
  }
}
```

- `filters.price.min` / `.max` are **minor units** — `4000` means $40.00.
- `filters.available` defaults to `true` (sale-ready items only). Set it `false` only if the buyer
  explicitly wants to see out-of-stock items.
- `filters.categories[]` combine with OR.
- Pass `context.address_country` and `context.currency`; the merchant's own instructions call this
  out, because pricing and availability are localized.

## 2. Page

Results are deliberately limited. Take `pagination.cursor` from the response and send it back as
`catalog.pagination.cursor` **only when the buyer asks for more** — do not pre-fetch pages.

## 3. Resolve identifiers

If you already hold IDs, use `lookup_catalog` instead of repeated searching:

- Up to **10** IDs per request (`catalog.ids`, `minItems` 1, `maxItems` 10).
- A `gid://shopify/Product/...` returns the product with `match: "featured"`.
- A `gid://shopify/ProductVariant/...` returns the parent product with that exact variant.
- Each variant carries an `inputs` array showing which requested ID resolved to it.

## 4. Get detail before recommending

Call `get_product` with `catalog.id` for anything you are about to recommend. Search results are a
list; `get_product` is what gives you exact pricing, the relevant variant set, and real-time
availability. Use `catalog.selected` (`[{name, label}]`) to narrow options as the buyer chooses, and
`catalog.preferences` to pass their stated preferences.

## 5. Quote the price correctly

Every price in every response is an integer in ISO 4217 minor units paired with a currency code:

```json
{ "amount": 2500, "currency": "USD" }   // this is $25.00
```

Divide by 100 for two-decimal currencies (USD, EUR). Zero-decimal currencies (JPY) are already whole
units. **Never** read `amount` as dollars.

## Errors

- No rate-limit headers are returned, and no numeric limit is published. The merchant states the
  endpoint is rate-limited **per IP** and that you must back off on `429`. Back off exponentially;
  there is no `Retry-After` to read.
- Errors arrive as JSON-RPC 2.0 error objects. There is no published error vocabulary.
- A `404` from `urbanstems.com/api/ucp/mcp` means you used the wrong host — see the top of this file.

## Questions about policy

For "what is your return policy", "do you ship to X", "what are your hours", call
`search_shop_policies_and_faqs` on the storefront MCP at
`https://274513-24.myshopify.com/api/mcp` with a natural-language `query`. Do not guess policy from
the catalog.
