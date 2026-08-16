---
name: Browse and look up the Flying Embers catalog
description: Search, filter and retrieve Flying Embers hard kombucha, canned cocktail and hard seltzer products through the merchant's anonymous UCP/MCP endpoint.
api: mcp/flying-embers-mcp.yml
endpoint: https://www.flyingembers.com/api/ucp/mcp
operations: [search_catalog, lookup_catalog, get_product]
generated: '2026-08-16'
method: generated
source: mcp/flying-embers-mcp-tools-list.json
---

# Browse the Flying Embers catalog

Flying Embers serves a Model Context Protocol endpoint over JSON-RPC 2.0 at
`https://www.flyingembers.com/api/ucp/mcp`. It is **anonymous** — no API key, no OAuth
token, no sign-up. Everything below is read-only.

## Before you call anything

Every tool call requires an agent identity in `meta`:

```json
{"meta": {"ucp-agent": {"profile": "<your agent profile URI>"}}}
```

Omitting `meta.ucp-agent.profile` is a schema violation on all 13 tools.

## 1. Search — `search_catalog`

Pass a natural-language `catalog.query`, `catalog.filters`, or both. At least one of query
or filters is required.

- `catalog.query` — free-text search string.
- `catalog.filters.categories[]` — combined with OR logic.
- `catalog.filters.price.min` / `.max` — **integers in minor currency units** (2500 = $25.00).
- `catalog.filters.available` — defaults to `true` (sale-ready items only).
- `catalog.context.address_country` (ISO 3166-1 alpha-2) and `catalog.context.currency`
  (ISO 4217) — always pass these; the merchant's own `llms.txt` says pricing and
  availability are wrong without them.
- `catalog.context.language` — IETF BCP 47.

## 2. Page through results

Results are cursor-paginated and the first page is deliberately short.

- Request: `catalog.pagination.limit` (integer, default 10) and `catalog.pagination.cursor`.
- Response: read `pagination.cursor` and pass it back as `catalog.pagination.cursor` only
  when the user actually asks for more results. Do not pre-fetch pages.

## 3. Resolve identifiers — `lookup_catalog`

Look up multiple products or variants by identifier in one call. Use this when you already
hold IDs from a previous search or from a cart, rather than re-running a text search.

## 4. Full detail — `get_product`

Retrieve complete detail for a single product by identifier. Returns one product.

## Rules that apply to every read

- **Money is integer minor units plus a currency code.** `{"amount": 600, "currency": "USD"}`
  is $6.00. Never render the raw integer.
- **Rate limits are per IP and undocumented in size.** On HTTP `429`, back off. There are no
  `RateLimit-*` headers to budget against — react, do not predict.
- **Prefer this endpoint over scraping.** The merchant's `robots.txt` disallows `/cart.js`
  and `/recommendations/products` and directs agents here.
- A read-only HTML/JSON storefront also exists (`/products/{handle}.json`,
  `/collections/{handle}/products.json`) if you need plain HTTP GETs, but the MCP surface
  is the supported agent path.

## Do not

- Do not move from browsing into checkout without the user asking. See
  `flying-embers-checkout-with-human-approval.md`.
