---
name: makierp-orders
description: Prepares and creates MakiERP sales and purchase orders through MCP from whatever the user has — an Excel order form, a pasted list, a photographed fax, an email table or spoken instructions. Use when the user wants to enter an order, turn a counterparty's order form into an order, or bulk-enter ordered quantities. The agent parses the source material itself; these tools accept only structured drafts.
---

# MakiERP Orders

## Quick Start

1. Establish tenant access with `makierp-mcp-session` and state which firm you are working in.
2. Parse the Excel, PDF, photo, mail or notes yourself. Do not pass files to order tools.
3. Read [Order Workflow](references/order-workflow.md) before the first non-trivial order.
4. Call `order_types_list`; use its database id, direction and return flag.
5. Resolve the cari, warehouse and any other ids with `makierp-erp-data`. Never invent or fuzzy-match an id in a write call.
6. When the source uses the counterparty's own article codes, call `item_customer_codes_resolve`. Treat its `unmatched` suggestions as candidates to confirm with the user, never as matches. Once the user identifies one, offer to register it with `item_customer_codes_create` so the same question never comes back.
7. Call `orders_preview` with one partial draft. Resolve every entry in `missing_fields` by asking the user, then preview again. Fix all diagnostics.
8. Show the returned `totals` and every warning to the user in business terms and **wait for their explicit approval**.
9. Only then send the returned `prepared_order` and `receipt` unchanged to `orders_create`.

## Hard Boundaries

- One preview/create call handles one order. Repeat the workflow for each further order.
- **Never call `orders_create` without explicit user approval of the previewed totals.** An order is a commercial commitment; preparing one is your job, committing to it is the user's.
- Preview is mandatory preparation, not merely validation: it fills defaults, resolves prices, applies campaigns and promotions, computes VAT and totals, and freezes that exact payload.
- A receipt expires after 15 minutes and is single-use. Any payload change needs a new preview **and a new approval**.
- `missing_fields` is not an error list — it is the set of facts only the user can supply. Ask; never fill them with a plausible guess.
- Never silently accept a zero-price warning. Say out loud which line has no price and what that makes the order total.
- A price is never carried across currencies. When the cari's only price for a line is in another currency, preview drops it to zero and warns; ask the user for a unit price in the order's own currency rather than quoting a converted one.
- Goods and services cannot share one order. Split them into two orders and say why.
- Orders are saved as *öneri* (suggestion). Confirming, dispatching and invoicing stay in the user's hands; do not claim this surface can confirm, edit, cancel or delete an order.

## Teaching the Cari Code Map

An unmatched code is a one-time gap, not a permanent one. When the user tells you which item a code means, offer to register it:

> "SAP-99183" için eşleştirme yok. Bunu Çikolata 80g olarak kalıcı kaydedeyim mi? Bir daha sormam.

- `item_customer_codes_create` registers it. Always ask whether the cari orders that code by piece or by case, and set the ordering unit accordingly — this is the difference between 10 adet and 10 koli on every future order.
- `item_customer_codes_update` corrects one. Pointing a code at a **different item** is confirmation-gated, because every future order carrying that code would silently follow it. Show the before/after and get an explicit yes.
- When a cari stops using a code, retire it with `item_customer_codes_update` and `active: false`. Reach for `item_customer_codes_delete` only when the user genuinely wants the record gone; it is confirmation-gated too.

Never register a mapping on your own reasoning. The user names the item; you record it.

## Asking Well

When `missing_fields` comes back, ask one compact question covering everything open, in the user's own language, using the `candidates` where they were returned:

> Sipariş için iki bilgi eksik: hangi depodan karşılanacak (Merkez Depo / Girne Deposu) ve hangi cari adına girilecek?

Do not ask field-by-field, and do not name the underlying fields.

## Handover

- Use `makierp-schema-docs` for unfamiliar fields and enums.
- Keep the source reference — file name, sheet row, mail subject — in `document_tracking_no`, and the counterparty's own order number in `document_no`.
- Summarize in Turkish business terms; ids and field keys stay inside tool arguments.
