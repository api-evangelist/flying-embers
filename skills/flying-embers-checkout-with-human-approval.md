---
name: Complete a Flying Embers checkout with human approval
description: Drive the UCP checkout lifecycle on Flying Embers' MCP endpoint, honoring the merchant's idempotency key and its explicit human-approval-before-payment rule.
api: mcp/flying-embers-mcp.yml
endpoint: https://www.flyingembers.com/api/ucp/mcp
operations: [create_checkout, update_checkout, get_checkout, complete_checkout, cancel_checkout, get_order]
generated: '2026-08-16'
method: generated
source: mcp/flying-embers-mcp-tools-list.json
---

# Complete a Flying Embers checkout

> **Read this first.** Flying Embers publishes an explicit rule in both `/llms.txt` and
> `/robots.txt`: *"Checkouts are for humans. Do NOT complete checkout, payment, or order
> placement automatically — no scripted form fills, browser automation, or end-to-end agent
> flows that finalize payment without an explicit, contemporaneous human approval step."*
> Treat that as a hard precondition on `complete_checkout`, not as advice. If you cannot
> obtain contemporaneous buyer approval at the moment of payment, stop and route the
> purchase through the Shop skill (`https://shop.app/SKILL.md`) instead, as the merchant
> directs.

Endpoint: `https://www.flyingembers.com/api/ucp/mcp` (anonymous JSON-RPC 2.0 / MCP).
Every call carries `meta.ucp-agent.profile`.

## Sequence

1. **`create_checkout`** — required: `meta`, `checkout`. Returns line items, totals,
   discounts and taxes. Build from a cart you already reconciled with `get_cart`.
2. **`update_checkout`** — required: `meta`, `checkout`, `id`. Set the shipping address and
   shipping method. Address fields follow `street_address`, `extended_address`,
   `address_locality`, `address_region`, `postal_code`, `address_country` (ISO 3166-1
   alpha-2), `first_name`, `last_name`.
   This merchant's UCP profile declares `allows_multi_destination.shipping: false` and
   `allows_method_combinations: [["shipping"]]` — one destination, shipping only.
3. **`get_checkout`** — required: `meta`, `id`. Re-read the authoritative totals and show
   the user the real amount, in the real currency, before asking for approval.
4. **Ask the human.** Present the final total, the shipping address and the payment
   instrument. Get an explicit yes, in the moment.
5. **`complete_checkout`** — required: `meta` (including **`meta.idempotency-key`**),
   `id`, `checkout`.
   - `meta.idempotency-key` is a **required string** on this tool and this tool only. Mint
     one key per purchase intent and reuse the *same* key on every retry of that one
     purchase. Minting a new key on retry is how an agent double-charges someone.
   - Returns the order ID and Thank You Page URL, or the errors encountered.
6. **`get_order`** — required: `meta`, `id`. Confirm and report back.
7. **`cancel_checkout`** — required: `meta`, `id`. Use when the user backs out.

## Payment instruments

`checkout.payment.instruments[]` entries require `id`, `handler_id` and `type`.
`type` is `card` for credit/debit cards or `token` for wallet payments. `handler_id`
references a handler this merchant actually advertises in `/.well-known/ucp`:

- `gpay` — Google Pay (`com.google.pay`), cards VISA / MASTERCARD / AMEX / DISCOVER, full
  billing address and phone number required.
- `shopify.card` — Shopify card handler (`dev.shopify.card`), brands visa, master,
  american_express, discover, diners_club.

Do not handle raw card numbers. Use the handler-issued instrument.

## Failure handling

- HTTP `429` — rate-limited per IP. Back off; do not hammer.
- JSON-RPC `error` objects carry `code` and `message`. There is no published error-code
  registry for this surface, so surface the message rather than mapping it.
- `complete_checkout` may return errors *inside* the result payload rather than as a
  JSON-RPC error. Check both.
- If a `complete_checkout` call times out, **re-send with the identical
  `meta.idempotency-key`** and then verify with `get_order`. Never treat a timeout as a
  failed charge.
