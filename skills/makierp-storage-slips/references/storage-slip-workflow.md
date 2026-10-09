# Storage Slip Workflow

## 1. Discovery and source parsing

Storage-slip tools never read files. Inspect the user's photo, PDF, Excel workbook, handwritten page, or spoken notes upstream and turn each intended document into a structured draft. Preserve page/row references so a created slip can be traced back to its evidence.

Call `storage_slip_types_list` every time type ids are not already confirmed in the current tenant. Codes are stable business labels; ids are tenant database values. The response also tells you the physical direction, movement profile, stock-input shape, target-node need, and whether source buckets can be planned automatically.

Resolve canonical ids with the ERP-data skill:

- `slip_type_id`
- `organizational_unit_id`
- `node_id` and transfer `target_node_id`
- each `line_model_id`
- optional `subject_unit_id`, project, accounts, and other normal HTTP fields

Do not fuzzy-match inside preview/create. If lookup returns multiple plausible records, stop and ask.

## 2. Candidate selection

Use `storage_slip_stock_candidates` with exact `item_id` and source `node_id` whenever the evidence names or implies an existing identity. Narrow with exact mark/lot code, SKT, barcode, leaf stock node, or identity attributes.

Candidate rows contain:

- stable bucket `hash`
- live available quantity
- warehouse and leaf location
- lot/serial mark
- SKT
- barcode
- identity attributes and party origin

Do not choose among two matching positive buckets without evidence. Ask the user or retain the ambiguity as a diagnostic.

## 3. Draft preparation

Send exactly one HTTP-shaped draft to `storage_slips_preview`. A minimal starting draft contains:

```json
{
  "slip_type_id": 123,
  "organizational_unit_id": 45,
  "node_id": 67,
  "document_date": "2026-08-10",
  "document_tracking_no": "photo-12 page 3",
  "lines": [
    {
      "line_model_id": 890,
      "amount": "4.000000"
    }
  ]
}
```

Preview may safely fill:

- `document_no` and `slip_no` without advancing durable counters
- today's document date when omitted
- tenant local transaction currency and currency type
- sequential placements and stable line group ids
- the item's unambiguous base unit
- a full-quantity inbound identity only when the item and target leaf require no choice
- automatic price, computed currency amounts and totals
- automatically planned outbound source hashes

If an automatic price cannot be found, preview changes that line to an explicit allowed manual zero price and returns `automatic_price_not_found_zero_used`. Treat this warning as prominent financial information; do not hide it just because creation is allowed.

## 4. Slip shapes

### Ordinary inbound

Use `stock.inbounds`. Quantities must add up to the line amount. Supply required mark/serial, SKT, barcode, identity attributes, and `target_node_id` when the warehouse scope does not resolve to one leaf. Preview only invents a row for completely unambiguous no-tracking stock.

### Ordinary outbound and Sayım Eksiği

You may omit stock allocations. Preview runs the authoritative planner and freezes every selected bucket into `stock.allocations`. When paper evidence specifies a lot/SKT/location, select the exact candidate hash instead of accepting generic planning.

### Transfer

Supply both header `node_id` and `target_node_id`. Preserve explicit `stock.targets` and their quantities when multiple destination leaves are involved. Preview fills each target's exact source allocations; a simple one-target transfer may use flat allocations with target node ids.

### Sayım Fazlası

Use manual `stock.inbounds`. This MCP surface deliberately forbids source-linked `stock.reconnects`, even when a tenant feature flag enables them. A no-tracking row at one unambiguous leaf may be prepared automatically; tracked stock needs its stated identity.

### Dönüşüm (`55`)

Dönüşüm is an in-place identity correction. It never changes the item and never moves between leaves.

- `stock.allocations`: exact old bucket hashes and quantities to consume
- `stock.inbounds`: corrected mark/SKT/barcode/attributes and matching quantities to birth

Both sides must equal the logical line quantity. The old sources must be the same item and same leaf. A no-op identity change is rejected. If the product itself changes, follow the user's stated Sayım Eksiği/Sayım Fazlası treatment rather than disguising it as Dönüşüm.

## 5. Preview, create, and readback

`storage_slips_preview` returns either diagnostics or all of:

- `prepared_slip`
- warnings
- resolved source rows and availability
- planned effective postings
- signed receipt and expiration

Preview makes no durable slip, stock, ledger, audit, planner-run, or generation-counter change. It does not reserve stock.

Call `storage_slips_create` with the exact `prepared_slip` and receipt. The agent may do this immediately; the receipt is an integrity gate, not a human-approval gate. Creation revalidates uniqueness and live stock under the normal transaction. It never substitutes another bucket. A changed payload, expired/replayed receipt, duplicate document number, or stock drift means: create nothing, run a new preview.

After success, call `storage_slips_show`. Compare the canonical saved stock assignments and `effective_postings` with the preview. Report the resulting fiş türü, date, document/slip number, item, quantity, warehouse/location, old/new identity where relevant, and any zero-price warning.

## 6. Ambiguity and prohibited inputs

Stop instead of guessing when:

- item lookup is not unique
- a unit conversion is unclear
- multiple source buckets still match the evidence
- a tracked inbound identity is incomplete
- a transfer destination leaf is unclear
- Dönüşüm's old or new identity is unclear

Never submit:

- `stock.reconnects`
- source allocation ids for return/reconnect sourcing
- legacy `lots`
- warehouse ids in place of stock-node ids
- hot-sales request links
- file content or image data
