---
name: makierp-bank-transactions
description: Imports and manages MakiERP bank transactions through MCP — preview, create, update, and delete bank slips, one row per statement line. Use when the user wants to import a bank statement (Excel/PDF/CSV), log bank movements, or create, update, or delete bank transactions. The agent itself parses the statement file into rows; these tools never read files.
---

# MakiERP Bank Transactions

## Quick Start

1. Use `makierp-mcp-session` first if tenant orientation or access is not established. Bank transactions write to tenant financial data — confirm the tenant.
2. Parse the statement yourself into structured rows; there is no file-import tool.
3. Follow [Bank Transaction Import](references/bank-transaction-import.md) for the field model, slip-type rules, and the preview → create loop.

## Workflow

- Resolve every id from schema/analytics tools before sending — `slip_type_id`, `bank_account_id`, `organizational_unit_id`, and (per slip type) `customer_id`, `position_id`, `slip_type_process_type_id`. Never invent ids.
- Always `bank_transactions_preview` first. It computes amounts/balances without saving and returns per-row diagnostics. Fix every diagnostic before creating.
- `bank_transactions_create` saves each row independently — a bad row returns diagnostics while the rest are created. Re-send only the rows you fixed.
- When preview reports an unresolved customer or account, ask the user instead of guessing.
- `bank_transactions_update` cannot change the slip type and cannot edit an accounted transaction.
- `bank_transactions_delete` reports objections (accounted, cancelled, payment-matched) instead of deleting; relay them.

## Handover

- Use `makierp-schema-docs` before assuming a field meaning, enum value, slip type, or process type.
- Use `makierp-erp-data` to look up existing bank accounts, customers, or transactions to reference.
- Use `makierp-erp-data`, not this skill, when the user only asks for bank balances or transaction totals.
