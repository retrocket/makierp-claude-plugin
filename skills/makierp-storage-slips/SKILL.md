---
name: makierp-storage-slips
description: Prepares, creates, and verifies MakiERP warehouse/storage slips through MCP, including inbound, outbound, transfer, Sayım Fazlası, Sayım Eksiği, and Dönüşüm slips. Use when the user wants to enter warehouse documents from photos, scans, Excel, notes, or spoken instructions. The agent parses source material itself; these tools accept only structured drafts.
---

# MakiERP Storage Slips

## Quick Start

1. Establish tenant access with `makierp-mcp-session` and state which firm you are working in.
2. Parse images, Excel, PDFs, or notes yourself. Do not pass files to storage-slip tools.
3. Read [Storage Slip Workflow](references/storage-slip-workflow.md) before the first non-trivial slip task.
4. Call `storage_slip_types_list`; use its database id and declared stock shape.
5. Resolve item, organizational-unit, and stock-node ids with `makierp-erp-data`. Never invent or fuzzy-match ids in a write call.
6. Use `storage_slip_stock_candidates` whenever a source lot, SKT, serial, barcode, or location matters.
7. Call `storage_slips_preview` with one partial draft. Fix all diagnostics and clearly surface every warning, especially zero price.
8. Send the returned `prepared_slip` and `receipt` unchanged to `storage_slips_create`. No extra user confirmation is required unless the user asked for one.
9. Call `storage_slips_show` and compare saved stock postings with the preview.

## Hard Boundaries

- One preview/create call handles one slip. Repeat the workflow for bulk entry.
- Preview is mandatory preparation, not merely validation: it generates numbers/defaults, resolves prices, computes totals, chooses source stock, and freezes those hashes.
- A receipt expires after 15 minutes and is single-use. Any payload change, duplicate number, or stock drift requires a new preview.
- Never send `stock.reconnects` or source-linked return selections. This MCP surface rejects them even when another interface enables that feature.
- Dönüşüm (`55`) changes stock identity in place for the same item and leaf location. It needs explicit old source hashes and explicit new identities.
- Stop and ask when source evidence, item, unit, location, lot/SKT, or corrected identity remains ambiguous.
- Creation is entry-only. Do not claim this surface can edit, delete, cancel, or uncancel slips.

## Handover

- Use `makierp-schema-docs` for unfamiliar fields and enums.
- Preserve the source filename, page, row, or paper reference in `document_tracking_no` or the description when useful.
- Summarize business terms to the user in Turkish; keep ids and field keys inside tool arguments.
