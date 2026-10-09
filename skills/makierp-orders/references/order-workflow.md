# Order Workflow

## 1. Source parsing

Order tools never read files. Turn the user's Excel sheet, PDF, photographed fax, mail table or spoken notes into structured rows yourself: one row per ordered line, carrying at least a code or a name and a quantity. Keep the sheet row numbers — they make every later question answerable ("3. satırdaki kod bizde kayıtlı değil").

One order per workflow. A file holding several caris' orders is several orders; loop.

## 2. Identity resolution

Call `order_types_list` whenever type ids are not already confirmed in this firm. Codes are stable business labels; ids are tenant database values. The response also tells you the direction and whether the type is a return.

Resolve the rest with the ERP-data skill:

- `customer_id` — the cari the order belongs to
- `node_id` — the warehouse it is supplied from, as its stock node
- optionally `payment_plan_id`, `project_id`, `customer_agreement_id`, `position_id`

If a lookup returns several plausible records, stop and ask. Never fuzzy-match inside preview or create.

### Counterparty article codes

When the form names items in the cari's own coding, call `item_customer_codes_resolve` with that `customer_id` and the rows exactly as written:

```json
{
  "customer_id": 412,
  "entries": [
    { "code": "SAP-99183", "name": "ÇİKOLATA 80G" },
    { "code": "SAP-99184" }
  ]
}
```

- `matched` rows are authoritative. Carry `item_id` into `line_model_id`, and — importantly — carry `subject_unit_id` through when the mapping supplies one: it identifies the exact concrete item unit in the named item unit set, so a quantity of 10 stays 10 cases rather than becoming 10 pieces.
- `unmatched` rows are open. Their `suggestions` are look-alikes found by our own item code, a barcode or the name, each labelled with `matched_on`. Present them to the user and use only what the user picks. A code that stays unresolved keeps the order incomplete — say so rather than dropping the line.

Once the user has named the item behind an unmatched code, offer to register it with `item_customer_codes_create` so the next order form resolves on its own:

```json
{
  "customer_id": 412,
  "item_id": 890,
  "code": "SAP-99183",
  "item_unit_id": 55,
  "description": "carinin kendi tanımı: ÇİKOLATA 80G"
}
```

Ask whether that code means a piece or a case before sending `item_unit_id`; a wrong unit here is wrong on every future order rather than just this one. Use `item_customer_codes_update` to correct a mapping — re-pointing it at another item needs the user's explicit approval — and prefer retiring a dead code with `active: false` over `item_customer_codes_delete`.

## 3. Draft preparation

Send exactly one draft to `orders_preview`. A minimal one:

```json
{
  "draft": {
    "slip_type_id": 12,
    "customer_id": 412,
    "node_id": 67,
    "document_no": "SIP-2026-4471",
    "document_tracking_no": "siparis_agustos.xlsx sheet1",
    "lines": [
      { "line_model_id": 890, "amount": 24 },
      { "line_model_id": 902, "amount": 6, "subject_unit_id": 3 }
    ]
  }
}
```

Preview fills, without being asked:

- today's document date
- the acting user's own position, and the unit that position reports into
- each line's base unit when none was given
- sequential placements and the item line type
- the transaction currency and its rate

Then it prices the order the same way the order screen does: the cari's price list, campaigns, promotions, manual discounts, VAT and totals.

## 4. Reading the response

**`missing_fields`** — the facts nobody could derive. Each entry carries the path, why it is open, and `candidates` when a short list exists. Nothing is prepared while any remain. Collect them into one question and ask the user; then preview again.

**`diagnostics`** — something sent is wrong: a closed period, a cari blocked for sales, a delisted item on a purchase order, goods and services mixed in one order. Fix and re-preview.

**`warnings`** — non-blocking assumptions the user must still see:

| code | what it means |
|---|---|
| `automatic_price_not_found_zero_used` | No price list covered this line; it is prepared at zero. Never let this pass silently. |
| `automatic_price_currency_mismatch` | The cari has a price for this line, but in another currency. It is not converted into the order currency; the line is prepared at zero instead. Ask the user for a unit price in the order currency and preview again with `pricing_mode: "pricing_unit_price"`. |
| `unit_assumed_from_item` | The item sells in several units and none was given, so the base unit was used. |

**`totals`** — document totals plus a figure set per line, campaign and promotion rows included. This is what the user approves.

## 5. Approval and creation

Show the totals and the warnings in the user's own business terms. Wait for an explicit yes.

Then send `prepared_order` and `receipt` back **unchanged** to `orders_create`. Do not re-key, reorder or "tidy" the payload: the receipt is bound to its exact bytes, and a mismatch is rejected rather than silently re-planned.

The receipt is single-use and lasts 15 minutes. If it expires, or the user asks for a change, prepare a fresh preview and get a fresh approval — an old approval does not carry over to new numbers.

## 6. Afterwards

The order is saved as *öneri* (suggestion) with its own generated number. Confirming it, dispatching it and invoicing it are the user's steps in the application. Report the saved number and total, and stop there.
