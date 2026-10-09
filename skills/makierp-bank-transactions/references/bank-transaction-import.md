# Bank Transaction Import

Use this for requests like:

- "Bu banka ekstresini içeri aktar" (with an attached statement)
- "Şu hareketleri banka fişi olarak gir"
- "Bu banka hareketini düzelt / sil"

You parse the statement file yourself into rows. These tools speak structured JSON only — they never read a file. The tools take **ids, not codes or names**, so resolve every id first.

## The Loop

1. Parse the statement into candidate rows: date, description, amount, and direction (money **into** the account vs **out of** it).
2. Discover ids (next section). Pick the slip type per row from its direction; resolve the bank account, organizational unit, and any slip-type-specific references.
3. `bank_transactions_preview` with all rows. Read each result: `transaction` (computed) or `diagnostics`.
4. Resolve every diagnostic. If a row needs a customer/account you cannot map confidently, ask the user — do not guess an id.
5. `bank_transactions_create` the clean rows. Report per-row ids and any rows that still failed.
6. Re-preview and re-create only the fixed rows.

## Discover Ids First

Use the analytics/schema tools (see `makierp-erp-data` / `makierp-schema-docs`). All of these are queryable schemas:

- `slip_types` — the bank-transaction slip types. Filter to the bank-transaction model and match by `code`/`name` (catalog below). Use the row's `id` as `slip_type_id`. Ids differ per tenant; never hardcode them.
- `bank_accounts` — the account the statement belongs to → `bank_account_id`. Note its `currency_code`.
- `organizational_units` → `organizational_unit_id`.
- `customers` → `customer_id` (only for customer-facing types like incoming/outgoing transfers). Customer currency must match the bank account currency.
- `slip_type_process_types` → `slip_type_process_type_id`. Most types require one; it must belong to the chosen slip type.

If you cannot resolve an id, send the row to `preview` anyway and read the diagnostic, or ask the user. Never invent ids.

## Direction: Debit vs Credit

For a bank account, **debit = money in, credit = money out.**

- Money arriving in the account (incoming transfer, collection, interest) → enter the amount as `*_currency_debit`.
- Money leaving the account (outgoing payment, fee, cheque payment) → enter it as `*_currency_credit`.

Each line carries one side; the other is prohibited — except the general type (allows either per line) and the transfer type (debit total must equal credit total across lines). Verify the direction with `preview`: the response balances (`local_currency_debit_balance` / `local_currency_credit_balance`) must match what the statement says happened.

## Slip Type Catalog

Standard bank-transaction slip types (resolve the per-tenant `id` by `code`/`name`):

| Code | Name (TR) | Use for | Money | Side | Extra required |
|------|-----------|---------|-------|------|----------------|
| `01` | Banka İşlem Fişi | General movement: fees, interest, charges with no counterparty doc | in or out | debit or credit | process type |
| `02` | Banka Virman Fişi | Transfer between your own bank accounts | both | balanced (≥2 lines) | process type |
| `03` | Gelen Havale/EFT | Incoming transfer/EFT from a customer | in | debit | customer, position, process type |
| `04` | Gönderilen Havale/EFT | Outgoing transfer/EFT to a customer | out | credit | customer, process type |
| `05` | Banka Açılış Fişi | Opening balance | in or out | debit or credit | process type |
| `06` | Banka Kur Farkı Fişi | FX revaluation difference | in or out | debit or credit | `lines_currency_type` must be `local`; no process type |
| `16` | Banka Alınan Hizmet Faturası | Bank service fee you were charged | out | credit | process type |
| `17` | Banka Verilen Hizmet Faturası | Service you billed through the bank | in | debit | process type |
| `18` | Bankadan Çek Ödemesi | Cheque payment from the bank (one account) | out | credit | process type; all lines share one account |
| `20` | Bankadan Gider Pusulası | Expense voucher from the bank | out | credit | process type |
| `21` | Bankadan Müstahsil Makbuzu | Producer receipt from the bank | out | credit | process type |

When unsure which code a statement line maps to, describe the slip type via `makierp-schema-docs` rather than guessing. System slip types (cash bridges, credit-card clearing, FX buy/sell) are managed elsewhere and are not created through these tools.

## Request Shape

`preview` and `create` take `{ "transactions": [ <slip>, ... ] }`. `update` takes `{ "transactions": [ { "id": 99, "transaction": <slip> } ] }` (omit `slip_type_id` — the type cannot change). `delete` takes `{ "ids": [99, 100] }`.

Every response is `{ "results": [ ... ] }`, one entry per input row in order. Preview/create/update entries carry `index` plus either `transaction` (the computed/saved slip with totals, balances, and lines) or `diagnostics`. Delete entries carry `id`, `deleted` (bool), and `objections` (strings).

### Slip fields

| Field | Required | Notes |
|-------|----------|-------|
| `slip_type_id` | yes | A bank-transaction slip type (not system-defined). Drives every per-line rule. |
| `document_date` | yes | `YYYY-MM-DD`. Must fall inside an open accounting period. |
| `lines_currency_type` | yes | `local`, `transaction`, or `report` — the basis amounts are entered in. Must be `local` for FX difference (`06`). |
| `organizational_unit_id` | recommended | Propagates to lines that omit it. |
| `description` | no | Free text; use it for the statement line memo. |
| `position_id` | type `03` only | Required at slip level for incoming transfers. |
| `slip_no` | create: no / update: yes | Auto-assigned on create; required when updating. |

### Line fields

| Field | Required | Notes |
|-------|----------|-------|
| `bank_account_id` | yes | Active bank account. |
| `organizational_unit_id` | yes | Inherited from the slip when omitted. |
| `<basis>_currency_debit` / `<basis>_currency_credit` | per type | `<basis>` matches `lines_currency_type` (`local`/`transaction`/`report`). Debit = money in, credit = money out. One side per line. |
| `slip_type_process_type_id` | most types | Required by every type except FX difference; must belong to the slip type. |
| `customer_id` | types `03`, `04` | Customer currency must equal the bank account currency. |
| `position_id` | type `03` | Required on the line too. |
| `account_id`, `throw_account_id`, `expense_group_id`, `project_id` | no | Active when set; can be filled later during accounting. |
| `transaction_currency_exchange_rate_handle` / `transaction_currency_exchange_rate` | no | Leave the handle off to let the system look up the rate. Only set the handle `true` together with a rate (> 0) to pin a manual rate. |

## Currency Basis

`lines_currency_type` picks the basis you type amounts in, and the line fields use the matching prefix:

- `local` → `local_currency_debit` / `local_currency_credit` (the tenant's local currency, YPB).
- `transaction` → `transaction_currency_debit` / `transaction_currency_credit` (the account/transaction currency, İPB).
- `report` → `report_currency_debit` / `report_currency_credit` (reporting currency, RPB).

The system converts across bases using the exchange rate for `document_date`. Example: a EUR account, a 1 000 EUR incoming payment, entered in transaction basis:

```json
{ "lines_currency_type": "transaction",
  "lines": [ { "bank_account_id": 8, "organizational_unit_id": 3, "slip_type_process_type_id": 4, "customer_id": 12, "position_id": 2, "transaction_currency_debit": 1000.00 } ] }
```

## Worked Examples

These are **minimal** — exactly the fields each slip type requires, nothing more (see "Minimal vs complete" below for what a real import adds). Replace every `<…>` with a discovered id. All amounts are in the chosen basis.

Incoming transfer from a customer (`03`, money in → debit):

```json
{
  "slip_type_id": "<id of code 03>",
  "document_date": "2026-03-14",
  "lines_currency_type": "local",
  "organizational_unit_id": "<ou>",
  "position_id": "<position>",
  "description": "Gelen havale - ACME Ltd",
  "lines": [
    { "bank_account_id": "<account>", "organizational_unit_id": "<ou>", "customer_id": "<customer>", "position_id": "<position>", "slip_type_process_type_id": "<process type>", "local_currency_debit": 15000.00 }
  ]
}
```

Outgoing transfer to a customer (`04`, money out → credit):

```json
{
  "slip_type_id": "<id of code 04>",
  "document_date": "2026-03-14",
  "lines_currency_type": "local",
  "organizational_unit_id": "<ou>",
  "description": "Gönderilen havale - tedarikçi ödemesi",
  "lines": [
    { "bank_account_id": "<account>", "organizational_unit_id": "<ou>", "customer_id": "<customer>", "slip_type_process_type_id": "<process type>", "local_currency_credit": 8000.00 }
  ]
}
```

General bank fee (`01`, money out → credit):

```json
{
  "slip_type_id": "<id of code 01>",
  "document_date": "2026-03-14",
  "lines_currency_type": "local",
  "organizational_unit_id": "<ou>",
  "description": "Hesap işletim ücreti",
  "lines": [
    { "bank_account_id": "<account>", "organizational_unit_id": "<ou>", "slip_type_process_type_id": "<process type>", "local_currency_credit": 45.00 }
  ]
}
```

Transfer between own accounts (`02`, balanced — credit the source, debit the destination):

```json
{
  "slip_type_id": "<id of code 02>",
  "document_date": "2026-03-14",
  "lines_currency_type": "local",
  "organizational_unit_id": "<ou>",
  "description": "Hesaplar arası virman",
  "lines": [
    { "bank_account_id": "<source account>", "organizational_unit_id": "<ou>", "slip_type_process_type_id": "<process type>", "local_currency_credit": 10000.00 },
    { "bank_account_id": "<destination account>", "organizational_unit_id": "<ou>", "slip_type_process_type_id": "<process type>", "local_currency_debit": 10000.00 }
  ]
}
```

### Minimal vs complete

The examples above pass validation as-is, but a real statement import should also set, whenever the statement provides them:

- `document_no` (slip level) — the bank's reference / transaction number, so the record traces back to the statement line.
- `description` (slip and/or per line) — the statement memo.
- `account_id` and/or `expense_group_id` (per line) — the GL counter-account and expense group, so the entry is accounting-ready instead of needing a later fill.

These are optional for validation but make the imported records complete. Prefer to populate them rather than ship bare minimal slips.

Fuller incoming example (`03`) with the recommended fields:

```json
{
  "slip_type_id": "<id of code 03>",
  "document_date": "2026-03-14",
  "document_no": "EFT-2026-0098431",
  "lines_currency_type": "local",
  "organizational_unit_id": "<ou>",
  "position_id": "<position>",
  "description": "Gelen havale - ACME Ltd",
  "lines": [
    { "bank_account_id": "<account>", "organizational_unit_id": "<ou>", "customer_id": "<customer>", "position_id": "<position>", "slip_type_process_type_id": "<process type>", "account_id": "<gl account>", "description": "ACME fatura tahsilatı", "local_currency_debit": 15000.00 }
  ]
}
```

## What Preview Returns

A valid row's `transaction` includes the computed `local_debit_total` / `local_credit_total`, the `*_balance` fields, the resolved currency/exchange-rate per line, and the line breakdown. Check that the balance direction and amount match the statement before creating. An invalid row returns `diagnostics`, each `{ code, path, message, severity, stage }` — the `path` points at the offending field (e.g. `lines.0.slip_type_process_type_id`).

## Update & Delete Rules

- **Update** revalidates the whole slip. The slip type is fixed — omit `slip_type_id` and the existing type is kept; sending a different one is rejected. An accounted transaction cannot be edited (diagnostic). `slip_no` is required on update.
- **Delete** runs guards and returns `objections` instead of throwing: a transaction that is accounted, cancelled, or matched to a payment comes back `deleted: false` with the reasons. Relay them; do not retry blindly.

## Safety

- Create, update, and delete mutate tenant financial records. Preview first, and confirm the working tenant before writing.
- Batches are per-row: report exactly which rows were created, which were skipped, and why. Never claim a whole statement imported when some rows returned diagnostics.
