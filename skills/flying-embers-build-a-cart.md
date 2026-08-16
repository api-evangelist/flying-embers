---
name: Build and manage a Flying Embers cart
description: Create, inspect, update and cancel a cart on the Flying Embers UCP/MCP endpoint before handing off to checkout.
api: mcp/flying-embers-mcp.yml
endpoint: https://www.flyingembers.com/api/ucp/mcp
operations: [create_cart, get_cart, update_cart, cancel_cart]
generated: '2026-08-16'
method: generated
source: mcp/flying-embers-mcp-tools-list.json
---

# Build a Flying Embers cart

The cart surface implements the UCP `dev.ucp.shopping.cart` capability on the merchant's
anonymous MCP endpoint `https://www.flyingembers.com/api/ucp/mcp`.

Every call carries `meta.ucp-agent.profile` (a URI). No credential is required.

## Sequence

1. **`create_cart`** — required arguments: `meta`, `cart`. Returns a cart with an `id`.
   Resolve product/variant identifiers first with `search_catalog` or `lookup_catalog`
   (see `flying-embers-browse-catalog.md`); do not guess identifiers.
2. **`get_cart`** — required arguments: `meta`, `id`. Read back the authoritative contents
   and totals before showing the user anything. Never render a cart you constructed
   locally.
3. **`update_cart`** — required arguments: `meta`, `cart`, `id`. Apply quantity changes,
   additions and removals. `update_cart` is **not** idempotency-keyed: a retried call is a
   second mutation, so re-read with `get_cart` after any ambiguous failure rather than
   blindly retrying.
4. **`cancel_cart`** — required arguments: `meta`, `id`. Abandon the cart when the user
   walks away.

## Reading totals

Every price is an integer in ISO 4217 minor units paired with a currency code. Pass buyer
context (`address_country`, `currency`) through the catalog calls so the products you add
are priced for the buyer's market.

## Boundaries

- The cart is not a purchase. Nothing here charges anyone.
- Move to `flying-embers-checkout-with-human-approval.md` only when the user explicitly
  asks to buy.
- On HTTP `429`, back off — the endpoint is rate-limited per IP.
- Failures come back as JSON-RPC 2.0 error objects (`error.code`, `error.message`); there
  is no RFC 9457 problem document on this surface.
