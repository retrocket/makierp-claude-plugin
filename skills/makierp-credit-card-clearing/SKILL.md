---
name: makierp-credit-card-clearing
description: Closes MakiERP credit card blocks against the current account (Kredi Kartı Sihirbazı) through MCP — create-only, one bank slip per statement day. Use when the user hands over a POS/merchant statement (POS hesap ekstresi, blokaj çözülme dökümü) and wants the blocked credit card balances cleared. Existing clearings can never be edited or deleted through MCP.
---

# MakiERP Credit Card Clearing

## Quick Start

1. Use `makierp-mcp-session` first if tenant orientation or access is not established. This writes tenant financial data — confirm the tenant.
2. This surface is **create-only**: `credit_card_clearings_create` is the only tool — no update, no delete, no separate preview; corrections happen in the ERP wizard UI. Saving is **confirmation-gated**: a call without `confirmation_token` saves nothing and returns the match report plus per-day previews and a token; only after the user explicitly approves do you repeat the call with the same input and the token. Never confirm without showing the report and hearing an explicit yes.
3. The tool needs three permits: "Kredi Kartı Sihirbazı" on payment transactions, "Oluşturma" on bank transactions, and the dedicated "Yapay Zeka ile Kredi Kartı Bloke Kapama (MCP)" on bank transactions. If the call is rejected as unauthorized, name the missing one to the user.

## What The Operation Does

A credit card sale puts money into the bank account's **blocked** balance. The bank releases it days or weeks later. This tool records that release: per credit card group it writes **exactly two lines** — one reversing the blocked balance and one debiting the current account — into a single bank slip per statement day.

- `clearing_type: customer_credit_card` → slip type `07`, blocked balance "Kredi Kartı Bloke".
- `clearing_type: company_credit_card` → slip type `08`, blocked balance "Firma Kredi Kartı Bloke".

The two cannot be mixed in one call. You never send slip type ids, balance types, amounts, customers or bank accounts per line — the ERP derives all of it from the originating sale.

## Workflow

- **You parse the statement**; the tool speaks JSON only and never reads files. Send `bank_account_id`, `clearing_type`, and `days`, each day carrying `document_date` (the day money reached the account), a **required** `description` in the user's own wording, and `rows` of `{amount, sale_date?, reference?}`.
- **Always send `sale_date` when the statement has it.** A POS release row normally prints the original sale date next to it (`BT: 02/03/2026`). That date narrows matching to the sale day itself instead of the whole window — on accounts where one amount recurs dozens of times it is the difference between a decisive match and an ambiguous row.
- **Matching is the tool's job.** It checks each amount against the open credit card groups on that bank account. Amounts match a whole **group total** — one card payment including all its installments — never a single installment row.
- **Per-row amounts often match nothing, and that is normal.** Banks bundle a day's card payments into a handful of release rows while the ERP holds one record per payment. When that is the shape, send **one row per sale day** carrying that day's **total** across all its release rows: if it equals the sum of every open group sold that day, the tool clears them all in one go. Check this first by comparing day totals before you try row-by-row.
- **Read the report before confirming.** Every day comes back with `matched`, `ambiguous` and `unmatched`. Only matched rows are ever written.
  - `ambiguous` — several open groups share that total, or the only match falls outside the date window. Never guess: show the candidate list (customer, sale date, day gap, installments) and let the user pick, then re-run that row with `group_uuids` pinned.
  - `unmatched` — nothing fits. Report it with the reason; do not invent a record. A group already carrying a clearing link is already cleared.
- **Fees are not clearing rows.** Commission, BSMV, POS rental and sweep/transfer lines on a POS statement are ordinary bank movements — leave them out and enter them with `makierp-bank-transactions` instead.
- **Refunds reverse both legs** automatically from the payment transaction's own type. Never try to flip debit and credit yourself.
- **`slip_no`** is assigned by the tenant's own numbering. Only supply it per day if a diagnostic says the tenant has no automatic bank slip numbering.
- **Reconcile.** After each day compare the created slip total with the statement day total, and at the end list every skipped row with its reason.

Read `references/credit-card-clearing-playbook.md` before driving a real statement.

## Handover

- Use `makierp-schema-docs` before assuming a field meaning, balance type, or slip type.
- Use `makierp-erp-data` for read-only questions: which groups are still open, what a bank account's blocked balance is, which sales a customer paid by card.
- Use `makierp-bank-transactions`, not this skill, for ordinary bank statement lines — including the POS commission and sweep rows that sit alongside the block releases.
