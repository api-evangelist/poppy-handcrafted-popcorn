---
name: poppy-browse-catalog
description: Find Poppy Hand-Crafted Popcorn products by flavor, price or availability and read full product detail, using the store's UCP MCP endpoint.
api: Poppy Hand-Crafted Popcorn UCP Commerce API
endpoint: https://poppyhandcraftedpopcorn.com/api/ucp/mcp
transport: MCP over JSON-RPC 2.0
operations:
  - search_catalog
  - lookup_catalog
  - get_product
generated: '2026-08-26'
method: generated
source: mcp/poppy-handcrafted-popcorn-tools-list.json (live tools/list, HTTP 200)
---

# Browse the Poppy catalog

Read-only. Nothing here moves money or creates state.

## Before you call anything

Every tool requires `meta.ucp-agent.profile` — the URI of a UCP agent profile document you
publish. The server **fetches** it. If it is missing you get `-32001 invalid_profile_url`; if it
does not resolve you get `-32001 profile_unreachable`. Both return HTTP 422.

## Steps

1. **Search** — call `search_catalog` with `catalog.query`. Narrow with `catalog.filters`:
   `categories` (OR logic), `price.min` / `price.max` in **minor units**, and `available`
   (defaults to `true`, meaning sale-ready only).
2. **Page** — `catalog.pagination.limit` defaults to `10` (minimum `1`); pass
   `catalog.pagination.cursor` from the previous response to continue. Do not assume offsets.
3. **Localize** — set `catalog.context.address_country` (ISO 3166-1 alpha-2) and
   `catalog.context.currency` (ISO 4217). The store's own instructions say pricing and
   availability are wrong without them.
4. **Resolve identifiers** — `lookup_catalog` takes multiple `gid://shopify/Product/…` or
   `gid://shopify/ProductVariant/…` IDs in one request. A variant GID returns the parent product
   with that exact variant; a product GID returns it with `match: "featured"`.
5. **Detail** — `get_product` returns one complete product.

## Quoting a price to a human

Prices come back as `{"amount": 2500, "currency": "USD"}` — integer **minor units**. That is
$25.00, not $2,500. Divide by 100 for two-decimal currencies; zero-decimal currencies such as
JPY are already whole units. Convert before you say a number out loud.

## Errors

See `errors/poppy-handcrafted-popcorn-problem-types.yml`. Back off on `429` — the endpoint is
rate-limited per IP and publishes no budget headers, so you will not see it coming.
