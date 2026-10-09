---
name: makierp-pricing
description: Investigates MakiERP pricing and campaigns through MCP — why a line came out at a given price, which price cards contradict each other, which campaigns quietly stack, where margin is thin, and how much is being discounted by hand. Use when the user asks what a product sells for, why a price is wrong or inconsistent, whether campaigns overlap, which products lose money, or who is giving discounts away. All of these tools are read-only.
---

# MakiERP Pricing & Campaigns

## Quick Start

1. Use `makierp-mcp-session` first if tenant orientation or access is not established.
2. Pick the tool by the question, not by the table — see the map below.
3. Read [Pricing Rules](references/pricing-rules.md) before explaining *why* a price
   resolved the way it did. The selection rule is not what most people assume.

## Which tool answers which question

| The user asks | Tool |
|---|---|
| "What do we sell this at?" / "Why did this line come out at that price?" | `price_simulate` |
| "What is this product's price?" / "How does the list price compare to what we really get?" | `item_price_overview` |
| "Find the nonsense prices" / "Which prices contradict each other?" | `prices_consistency_audit` |
| "Do our campaigns overlap?" / "Which campaigns are set up wrong?" | `campaigns_consistency_audit` |
| "Which products make no money?" | `margin_audit` |
| "Who is discounting?" / "How much did we give away last month?" | `realized_discount_audit` |

## Working rules

- **Price selection is decided by priority alone, and a lower number wins.** There is
  no scoring and no rule that a customer-specific card beats a general one. When two
  cards share a priority the winner is whatever the database returns first — say so
  plainly rather than picking one.
- **`price_simulate` is the tool that explains.** It reports every candidate card with
  the reason it lost, then the campaigns in the order they applied. Prefer it over
  reasoning from raw price rows.
- **Campaigns do not pick a winner.** Every match applies, one after another, each
  working on the base the previous one left. Two 10% campaigns come to 19%, not 20%.
- **Campaigns need a customer.** `price_simulate` and `margin_audit` only evaluate them
  when a customer is given; without one the answer covers the price card alone.
- **Audits scan everything by default.** Narrow with `item_codes`, `item_category_id` or
  `brand_id` when the user is asking about one product, brand or category. The full
  finding count is always in `meta` — never present a page as the whole picture.
- **Findings come back without their evidence.** You get the rule, the product or
  campaign, and a `magnitude` saying how bad that instance is; findings are ranked by
  it. Ask for `detail: true` only after narrowing, when you need the competing cards
  themselves — the evidence is what makes a tenant-wide response unreadable.
- **Findings carry their own Turkish wording.** Relay `summary` and `suggestion` as
  written; they already avoid column names and enum codes.
- **Some findings are about the system, not the data.** The exclusive flag, the
  per-customer usage limit and the trading-group code are stored but never applied.
  Report these as "this setting does not take effect", not as a data error.

## Reading the numbers

- `YPB` is the local currency basis, `RPB` the reporting one. Amounts are converted at
  the date being asked about. A card whose currency has no rate that day comes back
  without a normalized amount — report it as unrated rather than comparing it anyway.
- `margin_audit` reports two cost bases side by side: the realised cost of the stock
  that went out, and the defined purchase price. Neither is complete on its own, and a
  row where one is missing says so instead of leaving a blank.
- `realized_discount_audit` separates hand-typed discounts from campaign ones, and
  shows how far the realised price drifted from the card the line started from.

## Handover

- Use `makierp-schema-docs` before assuming what a field or enum means.
- Use `makierp-erp-data` for plain totals — sales by month, a customer's turnover —
  rather than these tools.
- These tools never change a price, a campaign or a document. Anything the user wants
  changed has to be done in the application.
