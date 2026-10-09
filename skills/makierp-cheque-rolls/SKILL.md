---
name: makierp-cheque-rolls
description: Enters MakiERP cheque rolls (çek bordrosu) through MCP — create-only, one roll per batch row. Use when the user wants to record cheque entries, issue own cheques, or process cheques (collection, guarantee, bounce, return, transfer) as new rolls. Existing rolls can never be edited or deleted through MCP.
---

# MakiERP Cheque Rolls

## Quick Start

1. Use `makierp-mcp-session` first if tenant orientation or access is not established. Cheque rolls write to tenant financial data — confirm the tenant.
2. This surface is **create-only**: `cheque_rolls_create` is the only tool — no update, no delete; corrections happen in the ERP UI. Saving is **confirmation-gated**: a call without `confirmation_token` saves nothing and returns per-roll previews plus a token; only after the user explicitly approves do you repeat the call with the same rolls and the token. Never confirm without showing the preview and hearing an explicit yes.
3. The tool needs a dedicated permission (`cheque_rolls.mcpCreate`) on top of the regular cheque-roll create permission. If the call is rejected as unauthorized, tell the user to ask their administrator for the "Yapay Zeka ile Oluşturma (MCP)" permit on cheque rolls.

## Workflow

- Resolve every id from schema/analytics tools before sending — `slip_type_id`, `organizational_unit_id`, and (per slip type) `customer_id`, `bank_account_id`, `slip_type_process_type_id`, `cheque.id`. Never invent ids.
- Every roll needs `slip_no` (unique per slip type), `document_date` (inside an open period), `lines_currency_type`, and a `transactions` array with one entry per cheque.
- Slip types map to workflows:
  - `01` entry and `03` issue/endorsement record cheques: header `customer_id`, each transaction carries a full `cheque` object (portfolio_no, serial_no, due_date, transaction_total, bank fields; `type: own` cheques use `cheque_bank_account_id` instead of customer-cheque bank fields).
  - `05` send-to-collection and `07` guarantee: header `bank_account_id`, each transaction references an existing cheque via `cheque.id`.
  - `09` (customer-cheque processing) and `11` (own-cheque processing) additionally need `slip_type_process_type_id`; allowed cheque statuses depend on the process type.
  - `13` transfers between organizational units: needs `throw_organizational_unit_id` different from `organizational_unit_id`.
- Rows are validated with exactly the same rules as manual entry and saved independently: a bad row returns per-field diagnostics while the rest are created. Fix and re-send only the failed rows — never re-send rows that were already created (slip numbers are unique).
- When diagnostics point at an unresolved account, customer, or cheque status, ask the user instead of guessing.

## Bank Collection Statements (çek tahsil dökümü)

When the user hands you a bank's cheque-collection statement (Excel/PDF) plus the collecting bank-account code, build date-by-date rolls:

1. **Parse yourself.** Statement rows usually come in a few shapes — `serial(DRAWER BANK)` (cleared cheques), `<account> nolu hesaptan ödenen <serial> numaralı çek` and `<account>-<serial> nolu şube çekinin yatırım işlemi` (both also carry a cheque serial), plus non-cheque rows (EFT, transfers). Extract date, serial, and amount per row; keep non-cheque rows aside.
2. **Resolve the collecting bank account** over the bank-accounts data source. The user-supplied code is the FULL `code` value and may contain spaces and dotted segments (e.g. `001  102.01.01`) — match it exactly first; if that misses, fall back to a contains-match on the distinctive segment and confirm the pick with the user by account name.
3. **Look up every serial** over the cheques data source and verify the amount matches the statement row. No match on serial or amount → report the row, do not touch it. Duplicate serials in the statement → report, process only the first.
4. **Pick the operation from the cheque's current state**: in portfolio → a type-05 (send to collection) roll followed by a type-09/03 (collect from bank) roll on the statement date; already at the bank awaiting collection → just the 09/03 roll. Any other state (already collected, endorsed, bounced...) → report, do not touch.
5. **One batch per date**: send the 05 roll (only the portfolio cheques) and then the 09/03 roll (all of that date's cheques) in the same `cheque_rolls_create` call — rows are processed in order, so the 05 result is visible to the 09/03 row. The preview phase computes with the same ordering, so the batch previews correctly too.
6. **Approve once, then confirm per batch**: lay the whole date-by-date plan (dates, cheque counts, totals, rows to skip) in front of the user and get an explicit go-ahead; only then run the preview → confirm pair for each date's batch. If any preview disagrees with the statement, stop and ask instead of confirming.
7. **Reconcile and report**: after creating, compare per-date roll totals against statement totals and list every skipped row with its reason (non-cheque, duplicate, missing, amount mismatch, incompatible state).

## Handover

- Use `makierp-schema-docs` before assuming a field meaning, slip type, process type, or cheque status value.
- Use `makierp-erp-data` to look up existing cheques, customers, or bank accounts to reference, and for any read-only cheque questions (balances, portfolios, statuses).
- Use `makierp-bank-transactions`, not this skill, for bank statement imports.
