---
name: poppy-cart-and-checkout
description: Build a cart, turn it into a priced checkout, and complete a Poppy Hand-Crafted Popcorn purchase — with the buyer-approval and idempotency rules the store requires.
api: Poppy Hand-Crafted Popcorn UCP Commerce API
endpoint: https://poppyhandcraftedpopcorn.com/api/ucp/mcp
transport: MCP over JSON-RPC 2.0
operations:
  - create_cart
  - get_cart
  - update_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-08-26'
method: generated
source: mcp/poppy-handcrafted-popcorn-tools-list.json (live tools/list, HTTP 200) + https://poppyhandcraftedpopcorn.com/llms.txt
---

# Buy from Poppy Hand-Crafted Popcorn

## The rule that outranks everything else

**Do not complete a payment without explicit, contemporaneous buyer approval.** The store states
this in both `/llms.txt` and `/robots.txt`, and it prohibits scripted form fills and end-to-end
agent flows that finalize payment on their own. If you cannot get the buyer's approval at the
moment of payment, stop and hand off — the store's own recommendation is to route the purchase
through the Shop skill (`https://shop.app/SKILL.md`) instead.

## What is reversible, and what is not

| Step | Reversal | Window |
|---|---|---|
| `create_cart` | `cancel_cart` | not published |
| `create_checkout` | `cancel_checkout` | not published |
| `complete_checkout` | **none via API** | — |

Everything up to `complete_checkout` is free to undo and moves no money — a checkout computes
totals, taxes and shipping without charging. `complete_checkout` is the irreversible one. There
is no refund, void or cancel-order tool in the 13-tool set; after that call, reversal is a human
customer-service path (hello@poppyhandcraftedpopcorn.com, 828-552-3149).

## Steps

1. **Cart** — `create_cart` with `cart.line_items[]` (`id` = variant GID, `quantity`). Add
   `cart.buyer.email` / `phone_number` and `cart.context` for localization.
2. **Adjust** — `update_cart` to change quantities, add `cart.discounts.codes[]`, or set
   `cart.fulfillment.methods[]`. `get_cart` to re-read. `cancel_cart` to abandon.
3. **Checkout** — `create_checkout`, passing `checkout.cart_id` to carry the cart over.
4. **Fulfill** — `update_checkout` to set the shipping address and method. Note the store's
   declared constraint: **no multi-destination shipping**, and the only allowed method
   combination is `["shipping"]` alone.
5. **Confirm with the human** — read back the line items and the total, converted from minor
   units to major units. Get explicit approval.
6. **Complete** — `complete_checkout`. This call **requires** `meta.idempotency-key` in addition
   to `meta.ucp-agent.profile`; it is in the schema's `required` list and the call will not run
   without it. Generate one key per purchase intent and reuse that same key on any retry — that
   is what stops a network retry becoming a second charge.
7. **Confirm** — the response carries the order ID and a Thank You Page URL. `get_order` reads it
   back afterwards.

## Payment

Registered handlers are Google Pay (`com.google.pay`), Shopify card (`dev.shopify.card`) and
Shop Pay (`dev.shopify.shop_pay`). Pass instruments under `checkout.payment.instruments[]` as
`{id, handler_id}`. You never handle a card number.

## Errors

`-32001 invalid_profile_url` / `profile_unreachable` (HTTP 422) mean your agent profile URI is
missing or unfetchable. `-32000 AuthenticationRequired` (HTTP 403) means the call needs a valid
agent JWT — see https://shopify.dev/docs/agents/get-started/authentication. Full catalog in
`errors/poppy-handcrafted-popcorn-problem-types.yml`.
