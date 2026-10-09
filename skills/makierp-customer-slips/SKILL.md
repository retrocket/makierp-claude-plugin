---
name: makierp-customer-slips
description: Enters MakiERP current-account slips (cari hesap fişi) through MCP — create-only, one slip per batch row. Use when the user wants to move a customer balance directly: cash collections and payments, debit and credit notes, transfers between customers, FX-difference corrections, opening balances, late-interest invoices, self-employment receipts, credit card slips. Existing slips can never be edited or deleted through MCP.
---

# MakiERP Customer Slips

## Quick Start

1. Use `makierp-mcp-session` first if tenant orientation or access is not established. Customer slips move balances and create debt-tracking entries — confirm the tenant.
2. This surface is **create-only**: `customer_slips_create` is the only tool — no update, no delete; corrections happen in the ERP UI. Saving is **confirmation-gated**: a call without `confirmation_token` saves nothing and returns per-slip previews plus a token; only after the user explicitly approves do you repeat the call with the same slips and the token. Never confirm without showing the preview and hearing an explicit yes.
3. The tool needs a dedicated permission (`customer_slips.mcpCreate`) on top of the regular customer-slip create permission. If the call is rejected as unauthorized, tell the user to ask their administrator for the "Yapay Zeka ile Oluşturma (MCP)" permit on customer slips.

## When a Customer Slip Is the Right Document

Balances moved by an invoice, a bank slip, a safe transaction or a cheque roll already come from their own source — never re-enter those here. A customer slip is for the movement you have to write directly.

| Need | Slip type |
| --- | --- |
| Collect cash from a customer | Nakit Tahsilat |
| Pay cash to a customer | Nakit Ödeme |
| Debit / credit a customer with a reason | Borç Dekontu / Alacak Dekontu |
| Move a balance from one customer to another | Virman Fişi |
| Correct a balance difference caused by exchange rates | Kur Farkı Fişi |
| Enter an opening balance for the period | Açılış Fişi |
| Charge or receive late interest | Verilen / Alınan Vade Farkı Faturası |
| Process a self-employment receipt | Verilen / Alınan Serbest Meslek Makbuzu |
| Card collection, card refund, company card | Kredi Kartı slip types |

## Workflow

- Resolve every id before sending — `slip_type_id`, `organizational_unit_id`, `customer_id`, and per slip type `bank_account_id`, `position_id`, `project_id`. Slip type ids differ per tenant. Never invent ids.
- Every slip needs `slip_type_id`, `document_date` (inside an open period), and a `lines` array with at least one entry carrying `customer_id`. Leave `slip_no` out: the tenant numbering assigns it. Supply one only after a diagnostic says the tenant has no automatic customer slip numbering.
- `organizational_unit_id` is mandatory on every line; set it once on the header and it reaches the lines that omit it.
- **The slip type decides the direction.** Collections and credit notes are credit-only; payments and debit notes are debit-only; sending the wrong side is rejected, and never fill both sides of one line.
  - Cash collection / payment and debit / credit notes take many customers on one slip.
  - Late-interest invoices and self-employment receipts are single-customer: the header `customer_id` and every line customer must be the same.
  - Transfers need at least two lines, one debit and one credit, and the local totals must balance exactly.
  - FX-difference slips work in the local currency only and produce no debt-tracking entry.
  - Credit card slips need a header `customer_id`, `bank_account_id` and `position_id`, the customer and bank account must share a currency, and they also write a bank movement that later has to be closed — hand that closing over to `makierp-credit-card-clearing`.
- **The amount field name follows the currency basis.** `lines_currency_type` (or a line's own `original_currency_type`) picks the prefix: `local_currency_debit` / `local_currency_credit`, or the `report_` / `transaction_` pair. For a transaction-currency line with a manual rate, send `transaction_currency_exchange_rate` together with `transaction_currency_exchange_rate_handle: true`.
- Rows are validated with exactly the same rules as manual entry and saved independently: a bad row returns per-field diagnostics while the rest are created. Fix and re-send only the failed rows — never re-send rows that were already created (slip numbers are unique per slip type).
- Read the preview back to the user in business terms: slip type, date, unit, each customer with the amount that hits their balance, and the debit/credit totals. Approving totals alone is not approving the slip.
- When diagnostics point at an unresolved customer, a closed period, a missing exchange rate or an unbalanced transfer, ask the user instead of guessing.

## After Saving

Every slip except the FX-difference type also creates a debt-tracking entry (ödeme hareketi), so the amount shows up as open until it is matched against an invoice. Credit card slips additionally create a bank movement. Once a line is matched in debt tracking, or the slip is posted to accounting, its amount, customer and direction can no longer be changed at all — which is another reason to get explicit approval before confirming.

## Handover

- Use `makierp-schema-docs` before assuming a field meaning, slip type, or currency basis.
- Use `makierp-erp-data` to look up customers, balances, open items, bank accounts and organizational units to reference, and for any read-only customer question.
- Use `makierp-credit-card-clearing` to close credit card collections with a bank slip.
- Use `makierp-bank-transactions` for bank statement imports and `makierp-cheque-rolls` for cheques — those balances must not be re-entered as customer slips.
