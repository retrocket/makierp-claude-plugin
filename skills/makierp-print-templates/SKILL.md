---
name: makierp-print-templates
description: "Reads, creates, updates and deletes MakiERP print templates (RDL) through MCP — record prints (invoice, waybill, order, cheque, storage/customs/customer/bank/cheque-roll/cost slips and the four movement receipts), parameterised report and ledger layouts, and analytics export layouts. Use when the user wants a new print layout, a change to an existing one, or an inventory of the templates a tenant already has. Read this before producing a template: no template is written without asking the user which data path it feeds."
---

# MakiERP Print Templates

## Quick Start

1. Use `makierp-mcp-session` first if tenant orientation or access is not established. A template decides what every printed invoice, statement or export looks like — confirm the tenant.
2. **Iron rule — the data path is asked, never guessed.** There are four paths (record print / parameterised report / analytics export / draft) and they are not interchangeable: the path fixes the template `type`, the dataset shape, and which output channels exist at all. Ask the question in [Dialog Flow](references/dialog-flow.md) verbatim and follow its branch. If any answer stays unclear, **stop and ask again** — do not assume, do not produce a template.
3. **Never invent a field name.** Exactly four sources are legitimate, one per path: the report's own column schema (`print_templates_sources`), an existing template's declared fields (`print_templates_get` with `fields_only`), a real print response the user hands you, or the analytics preview columns — and `print_templates_create` makes you declare which one you used in `field_source`. On the draft branch the user's own embedded sample data is the source, and its keys are the field names ([Dialog Flow](references/dialog-flow.md) Step 2D). Several record types have none of them; then produce nothing and say so: "Alan adlarını uyduramam — bu türde hangi alanların basılabildiğini göremiyorum, o yüzden şablon üretmiyorum." An invented key does not fail; it prints a silently blank cell.
4. **Writing is two-phase, always.** `print_templates_create`, `_update`, `_delete` and `_revert` called **without** `confirmation_token` save nothing — they return a dry preview of what would change, its warnings and a token. Show that preview to the user in Turkish wording, wait for an explicit yes, then repeat the **same** payload plus the token. A tokenless response is not "saved"; never report it as a finished template. The token is **single-use and tied to the session**: it dies with the payload it previewed, and it also dies when tenant access is renewed in between ("Onay kodu geçersiz, süresi dolmuş ya da gönderilen bilgiler önizlenenden farklı"). Then redo the token-less preview and confirm the new token — never treat the rejection as a save failure. On a long template job ask for a generous access window up front.
5. Speak product language, never identifiers: "Cari Hesap Ekstresi", not `customer_statement`; "belge no", not `document_no`. Keys belong in tool arguments only.

## The Four Paths

| The template prints | `type` | Print | Mail | Export | Layout applies to |
|---|---|---|---|---|---|
| A record — invoice, waybill, order, cheque, storage slip, customs clearance slip, accounting slip, the customer/bank/cheque-roll/cost slips, and the four movement receipts | one of 16 record types | yes | no | no | the browser print |
| A parameterised report — statements, ledgers, trial balance, cost analysis | one of 10 report keys | yes | yes | yes | PDF only (`xlsx`/`csv`/`json` ignore the design — and in a **grouped** report they also drop the grouping itself, yet still require a template) |
| A saved (or ad-hoc) analytics question's output | `analytics_export` | yes | yes | yes | PDF only; `xlsx` takes the column plan and the name |
| A draft on sample data, bound to no channel yet | any enum value, `is_default` forced `false` | no | no | no | design time only |

Customer slips, bank transactions, cheque rolls and cost/sale-provision slips are record types of
their own, and so are the four **movement receipts** — a single cari, bank-account, safe or
accounting-account movement printed as a one-line makbuz (`customer_slip_line`,
`bank_transaction_line`, `safe_line`, `accounting_slip_line`). Those screens also still offer
printing the *linked* accounting slip, which is a different type on a different endpoint: when the
user says "kasa satırını bas", they mean the makbuz, not the muhasebe fişi. Ask which one if the
wording is ambiguous.

The four movement types carry **no nested lines** — the record IS the line, so their templates have
a single `DataSet` and no `$dataset:DataSet/lines`.

A draft has no channel of its own, but it is not invisible: once saved, users can pick it from "Özel Yazdır" (a menu cheques and accounting slips do not have) and the different-template PDF lists. What it never gets is being chosen automatically — tell the user that, not "it is connected to nothing".

## Workflow

- New template: run the dialog flow, take the field names from the source named in Quick Start 3, `print_templates_validate`, then the two-phase `print_templates_create`.
- Existing template: `print_templates_list` → `print_templates_get` (the list never returns `content`) → edit → `print_templates_update` with the current `expected_content_hash`. The `type` is refused by default, so the path question is skipped — the path follows from the type. (`allow_type_change: true` exists for one case only: moving a draft onto its channel once that channel is built.)
- Validate before every save and read the `skipped_checks` list: a check that could not run is not a check that passed.
- Run `print_templates_dry_run` after saving whenever the user expects real output. It is the only automatic detection of a render error the PDF path swallows, and the only confirmation of a page-per-row layout against real rows.

## Surface Map

- The mandatory question order, per branch, with the exact sentences to say: [Dialog Flow](references/dialog-flow.md).
- Which of the four paths applies, its endpoints, and which channels/formats it really has: [Data Paths](references/data-paths.md).
- The stored record, the `content` envelope, the double-encoding trap, CRUD payloads and the type enum: [Template Envelope](references/template-envelope.md).
- The two skeletons (record vs report), the `RecordAnchor` binding, the multi-collection ledger shape, and the page-per-row accident: [Skeletons](references/skeletons.md).
- Expression syntax, what is supported, and the silent null/NaN traps: [Expression Language](references/expression-language.md).
- Table bands, `RepeatOnNewPage`, page-boundary rows, groups, the page-1 header band and page geometry: [Tables And Pagination](references/tables-and-pagination.md).
- Number/date formats, locale limits, and logo/`EmbeddedImages` handling: [Formatting](references/formatting.md).
- The native `chart` item and when it may be used: [Charts](references/charts.md).
- A symptom that already happened (blank cells, empty PDF, one page per line, failed export): [Troubleshooting](references/troubleshooting.md).

## Tools

- `print_templates_list` — read: which templates exist per type, which one is default. Returns no `content`.
- `print_templates_get` — read: the full template plus its summary, field catalogue and `content_hash`. Required before an update whose current hash you do not already hold. Two different readings, do not mix them up: `fields_only` for **field names**, `summary_only` → `body_items` for **structure** (which regions sit at body level, which one binds the primary, whether the flowing table is up there) — that is the diagnosis view for a broken layout. `summary_only` lists every declared field, so on a field-rich template it can be the more expensive of the two and may exceed the tool output limit; when it does, continue from the dropped file with `jq '.summary.body_items'`.
- `print_templates_sources` — read: report keys, their columns and params, record types and their field-source flags, the analytics column chain.
- `print_templates_validate` — read: structural and expression checks on a definition before it is stored.
- `print_templates_dry_run` — read: runs a saved template against real data and reports page count, empty pages, missing fields and swallowed render errors. Produces no file and no queue job.
- `print_templates_create` — **writes.** New template record, confirmation-gated.
- `print_templates_update` — **writes.** In-place overwrite of an existing template, confirmation-gated.
- `print_templates_delete` — **writes.** Removes a template, confirmation-gated, and refuses while blockers exist.
- `print_templates_revert` — **writes.** Restores a previous `content` from the audit trail, confirmation-gated.

There is no tool for adding a new report key or a new engine capability — both are development work. When the user asks for a report that is not in `print_templates_sources`, say so and let them pick from the existing list.

### Arguments that are easy to miss

| Argument | Where | Why it matters |
|---|---|---|
| `field_source` | `create`, and `update` whenever you send a `definition` | Required. Names which of the four legitimate sources the field keys came from; the tool rejects a definition you cannot account for. |
| `columns` | `create` / `update` / `validate` on `analytics_export` | The real output column keys. Without them the column gate cannot run and the export is refused later, by which time the template is already stored. |
| `sample_row` | `create` / `update` / `validate` | One real row, nested collections included. It is what turns "the field exists" into "the field has a value here". Without it the field-existence check lands in `skipped_checks`, and a clean `ok` means nothing. On the report path it comes from `print_templates_sources`; on the record path there is no endpoint for it — harvest it from a sibling template's embedded sample ([Dialog Flow](references/dialog-flow.md) Step 2A Q2b). |
| `channelless_draft` | `create` on the draft branch | The flag that makes the draft legal: it forces `is_default` to `false` and returns the channel warning you must read to the user. Without it the draft branch behaves like an ordinary template. |
| `new_default_id` | `delete` when the target is the default of its type | The template that takes over. Deleting the only default without it is refused — the category would lose printing entirely. |
| `question_id` | `dry_run` on the analytics path | The dry run has no other way to know which question's rows to render. Omit it and the analytics template cannot be exercised. |
| `expected_content_hash` | `update` | Required. It is the version stamp of the content you are overwriting, and it is what makes "do not overwrite what you have not read" enforceable. `get` returns it — and so does the **confirmed phase of your own `create` / `update` / `revert`**, so a second edit in a row does not need another `get`. Rule: if you do not hold that template's **current** hash, call `get`; if a foreign write slipped in between, the hash no longer matches and the tool refuses the write — that refusal means "read it again", never "retry". |
| `allow_type_change` | `update`, rarely | Type changes are refused by default. The one legitimate use is migrating a draft once its channel exists; everything else is a mistake. |

## Non-negotiables

- **Which dataset the body region binds to inverts with the path.** On the record path a primary-bound region is **mandatory** and must be a direct child of the body; without it the engine renders a single instance and only the first record prints. Get it with a **flat body** — the line table and the header boxes stay at body level, and an empty zero-size `RecordAnchor` list carries the primary binding. Do **not** wrap the body in a `RecordBlock` list: the wrapper satisfies the binding but kills the page-1 header band, so the table climbs to the top of the page and overlaps the header boxes ([Skeletons](references/skeletons.md)). On the report and analytics paths the body table binds to the **nested** dataset (`$dataset:Statement/lines`, `$dataset:Question/rows`) — binding it to the primary prints one page per line. The exception is a report whose source returns no document shape (`provides_document_rows: false` from `print_templates_sources`, today "Muhasebe Hesap Ekstresi"): there the flat shape is the correct one and a nested binding resolves to an empty collection. The engine enforces none of this; it renders the mistake happily.
- **An omitted `DataSetName` is not a neutral default.** The region is then rendered once against the current scope row — at body level that means only the first row prints, with no error anywhere. Same for a primary-bound table wrapped in a rectangle or list: the wrapper breaks the binding.
- **One bad expression takes down the whole document.** A single quote, `Fields(`, a missing parenthesis or a wrong-arity `IIF`/`IsNull` does not degrade one cell: the static path fails the export outright, and on the galley path the error is swallowed so a **blank PDF is reported as successful**. Validate before saving; never save an expression you have not run through `print_templates_validate`.
- **`is_default` is destructive in both directions.** Promoting one demotes every other template of that type — and on invoices that covers purchase and sales alike, since they share a single type. Demoting the only default of a type breaks printing for that whole category. Both directions go through the confirmation gate and are stated to the user before they happen.
- **Excel, CSV and JSON do not apply the template's layout — but the template still has to exist.** On report exports the columns come from the report's own schema and only the template's name travels (it becomes the Excel sheet name); a report with no template at all cannot be exported in any format either. Say this before the user assumes their page design will show up in a spreadsheet.
- **On a grouped report the spreadsheet loses more than the design.** Group headers, per-group "Devreden" blocks and group subtotals live in the template's `TableGroups`, so `xlsx`/`csv`/`json` come back as one flat run of rows with no visible group boundaries — in Muavin Defter, dozens of accounts' movements with nothing saying which row belongs to which account. The group's identity is only readable if the source itself carries it in `columns()`. Say it in advance whenever the user asks for the Excel of a grouped report: *"Excel'de hesap başlıkları ve ara toplamlar yer almaz, satırlar düz liste olarak iner."*
- **Record prints have printing only.** Invoices, waybills, orders, cheques and slips have no mail and no export channel today — the output is the browser print of the viewer. Promise nothing else on that path.

## Handover

- Use `makierp-analytics` for the saved question itself (its query, params and column keys); this skill only covers the template that renders it.
- Use `makierp-schema-docs` before assuming a field meaning, an enum value or a Turkish label.
- Anything about routing — which template gets picked by condition, priority or wildcard filters — is out of scope here and stays a job for the template screen in the application.
