# Data Paths

Use this for requests like:

- "Cari hesap ekstresine kendi şablonumuzu yapalım"
- "Bu analiz sorusunun PDF'i bizim antetimizle çıksın"
- "Fatura çıktımızı yeniden tasarla"

A print template is bound to exactly one data path. The path decides three things at once: the template's `type`, the shape of `DataSets[*].Query.DataSourceName`, and which output channels exist. Never guess it — `dialog-flow.md` makes asking mandatory.

## The engine rule that never changes

The renderer reads **only the `$dataset:` prefix** of `Query.DataSourceName`, the primary dataset is **always `DataSets[0]`**, and rows are injected from outside. `report:`, `question:`, `jpath=`, `jsondata=` are pipeline and designer conventions — the engine ignores them, and no backend code reads them either. Still write them exactly as the path expects: the designer parses them, and the whole shipped corpus follows the convention.

`schema:<id>` is the fourth ref the designer still recognises. It is **not a fifth data path** — it is the residue of an abandoned feed design with no consumer anywhere. Never bind a dataset to it.

Nested datasets are **not mandatory, they are the correct shape when the source produces a document** — the engine never forces them, and flat is a supported, live branch. Take the answer from `print_templates_sources`: `provides_document_rows: true` (11 of the 12 report keys) → nested required and binding a body region to the primary is an **error**; `false` (today only `accounting_account_statement`) → flat is correct and nested is an **error**, because the reference resolves to an empty collection and the template prints blank. Run `print_templates_validate` before writing; it applies this rule per key.

## Matrix

| # | Path | `type` | Data from | `DataSets[0].Query` | Nested | Param/Context | Yazdır | Mail | Dışa Aktar | Formats |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Embed | — (lives inside another path) | Designer memory | `DataSourceName:'<source>'` + `CommandText:'jpath=$.[*]'` | optional | — | ❌ | ❌ | ❌ | — |
| 2 | Analytics | `analytics_export` | Export request | `DataSourceName:'question:<id>'` | `$dataset:Question/rows` | export payload | ✅ | ✅ | ✅ | template only in `pdf`,`xlsx` |
| 3 | Report key | one of 12 keys | `ReportSource` | `DataSourceName:'report:<key>'` | `$dataset:Statement/lines` (may be several) | `paramSurface()` + context | ✅ | ✅ | ✅ | `pdf\|xlsx\|csv\|json`, template required in all |
| 4 | Record print | one of 16 record types | `POST /{endpoint}/print` | `DataSourceName:'<source>'` + `CommandText:'jpath=$.[*]'` | `$dataset:DataSet/lines` | record ids | ✅ | ❌ | ❌ | browser print only |

## Path 1 — Embedded / URL sample data

Not a production path: design-time sample data with **no print channel yet**. This is an unwritten capability, not a leftover — do not refuse the user, explain it: *"Bu veriyi şablona gömebiliriz ama bugün yalnız tasarım sırasında okunuyor; şablonun basılacağı bir kanal henüz yok."* Save it as a draft only after an explicit yes. The embed shape has no `type` of its own, so a draft saved this way still carries a real enum value plus a forced `is_default: false` (`dialog-flow.md` Step 2D).

- Embedded: `DataSources[i].ConnectionProperties = {DataProvider:'JSONEMBED', ConnectString:'jsondata=[...]'}`.
- URL: `{DataProvider:'JSON', ConnectString:'endpoint=<url>[;format=csv]'}` — fetched in the browser at design time only.
- Dataset bind: `CommandText:'jpath=$.[*]'` for a list, `jpath=$` for a single object.
- `print_templates_update` writes existing `ConnectionProperties` back **verbatim**. Never drop a structure you did not author. `print_templates_validate` treats embedded data as a note, never an error.
- Keep sample data small and representative. A 100 KB `jsondata=` blob only inflates the payload.

## Path 2 — Analytics (`analytics_export`)

`DataSets[0]` is `{Name:'Question', Query:{DataSourceName:'question:<id>'}}`, `DataSets[1]` is `{Name:'Rows', Query:{DataSourceName:'$dataset:Question/rows'}}` (a column literally named `rows` shifts the path to `_rows`). Tables, charts and pivots bind to `Rows`; root textboxes read row 0. `Fields` carry `{Name: column label, DataField: column key}`.

- **The template engages only for `pdf` and `xlsx`.** With `csv`, `json`, `svg` or `png` the `template_id` is **silently ignored** and the built-in layout prints. There is no error message; the user just reports "şablonum uygulanmadı".
- **In `xlsx` the RDL layout is not applied at all.** The writer projects `DataSets[0].Fields` over the raw rows; `definition.Name` becomes the sheet name and file title. Bands, charts and page geometry exist only in PDF. The column key is `DataField ?: Name` and the header the user reads is `Name`, so `Name` must be the Turkish label that belongs in Excel. Leave `Fields` empty and the writer falls back to the raw row keys — technical keys as headers.
- **One exception to "`Name` is the Excel header":** when the body contains a `list`, the writer matches every list cell bound as `=Fields!x.Value` to a header textbox at the **same `Left` offset**, and that textbox's text **overrides** `Fields[].Name` for that column (`TemplateRenderer::headerLabels()`). Table-based templates never take this path. So if an Excel header comes out unexpectedly, look for a list plus a header textbox before touching `Fields`.
- **Choosing a template silently discards the user's `export_options`.** Paper size, orientation, margins, font size, page numbers, repeated headers, titles, `xlsx_sheet_name`, `xlsx_as_table` and the rest are never passed on. Geometry comes from the template's `Page`, the title from `definition.Name`; the one surviving option is `filename` (falling back to `title`), which still names the downloaded file. So "kağıdı yatay yapalım" is a **template `Page`** edit, not an export option.
- **The backend does not check the template's type here** — asymmetric with paths 3 and 4, where a mismatched id is ignored and the default is used instead. An invoice template's id is accepted. The tool must verify `type === 'analytics_export'` itself.
- The only real guard is the hard column gate, and it is **reference-based**: every field the template actually references (`Fields!x`, `Fields.Item("x")`) plus native chart config columns must exist among the question's output columns, otherwise the export is refused with *"Seçilen şablon, bu sorunun üretmediği kolonlara bağlı…"*. Do not confuse it with the declaration-based check.
- Row ceilings: PDF **5.000** rows on the backend (enforced regardless of the template), viewer **1.000** (**100** with a per-row chart) — above the viewer limit the flow falls straight to a queued PDF. Past 5.000 both channels fail; the fix is narrowing the question, not the template.
- **Yazdır and Dışa Aktar do not print the same rows.** The viewer gets the page currently loaded on the question screen; export and mail re-run the query and print everything. Say so: *"Ekrandan yazdırdığınız çıktı sayfadaki satırları, indirdiğiniz dosya tüm satırları içerir."*
- This type never appears in the "Rapor Şablonları" index, and the "Şablon ile Çıktı" modal hides incompatible templates — a template whose question columns changed becomes unreachable from every UI. MCP is the recovery path.
- `/reports/print/analytics_export/*` returns 404; there is no schema endpoint for this path.

## Path 3 — Report key

Twelve registered keys: `customer_statement`, `accounting_account_statement`, `bank_account_statement`, `safe_statement`, `accounting_account_list`, `accounting_subsidiary_ledger`, `accounting_trial_balance`, `accounting_journal`, `accounting_general_ledger`, `detailed_cost_analysis`, `project_trial_balance`, `position_permission_scope`.

`DataSets[0]` is `{Name:'Statement', Query:{DataSourceName:'report:<key>'}}`; the body table binds to the nested `Lines`, never to the primary — the single exception is the flat key described below (`accounting_account_statement`), where the reverse holds. **One report can return more than one nested collection**: Defter-i Kebir has three datasets (`Statement`, `Lines` = `$dataset:Statement/lines`, `Summary` = `$dataset:Statement/summary`) and two body tables. Without that shape the ledger reports cannot be built.

| Endpoint | Returns | Accepts `template_id` |
|---|---|---|
| `GET reports/print/keys` / `{key}/schema` / `{key}/params` / `{key}/templates` | keys, flat columns, param surface, template list | — |
| `POST reports/print/{key}/data` | flat `rows` only — **never** nested keys | ❌ |
| `POST reports/print/{key}/preview` | `{definition, rows}` or a too-large envelope | ❌ |
| `POST reports/print/{key}` (sync) | raw bytes | ❌ |
| `POST reports/print/{key}/mail` \| `/export` | `202 {export_id}` | ✅ |

- Formats are `pdf | xlsx | csv | json`; anything unknown is **silently coerced to `pdf`** (`'excel'` returns a PDF, not a 422).
- **A template is required in every format** — the resolver runs before the format branch. Content is rendered only in PDF; `xlsx`/`csv`/`json` build a flat table from the source's own `columns()` and drop every band, chart and page setting. **On a grouped report that also drops the grouping**: group headers, per-group devir blocks and group subtotals live in `TableGroups`, so the spreadsheet is one flat run with nothing marking where one account ends and the next begins — visible only if the source itself carries the group's identity in `columns()`. Warn before the export, not after. But `definition.Name` is still used as the xlsx sheet name and title. Never tell the user "bu formatta şablon gerekmiyor". Missing template → 422 *"Bu rapor için bir şablon tasarlanmamış…"* on sync/preview, FAILED on the queued channels.
- Because `preview` and sync `print` reject `template_id`, a **non-default** template can only be tried through Dışa Aktar → "Farklı şablonla PDF". Plan the smoke test accordingly.
- Required params are satisfied by the `params` **or** the `context` bag; missing ones return 422 with a `missing` list. The UI hides a param already present in context.
- The `preview` size gate is not a failure. When the source paginates and the cheap page estimate passes its limit (every source ships 50 pages / 40 rows today), the endpoint answers with the `too_large` envelope and the UI queues a PDF export instead. Report it as designed behaviour: *"Bu aralık ekranda gösterilemeyecek kadar uzun, çıktı Dışa Aktarımlar listesine alınacak."*
- **`accounting_account_statement` is the exception**: `account_id` is *not* on its param surface and is read from `context` only, with no fallback. A call without it returns **an empty result, not a 422** — an empty report there usually means missing context. It is also the only source that does not provide document rows, so its correct shape is **flat** (nested is an error), it prints without galley, and the render-parity warning applies to it.
- **`position_permission_scope` always returns one document**, even when the position has no granted ability: the lines are empty and `scope_note` says why (system administrator, no role, only inactive roles), so the header and the signature block still print. Put the signature on the table's own `Footer` rows and give the table a `NoRowsMessage`; the position comes from `position_id` in the context (opened from a position) or the params (report catalog). Its header fields (`position_code`, `member_name`, `role_names`, `granted_count`, …) ride on every row and stay out of `columns()`, so the spreadsheet keeps its six columns.
- `POST /{key}/data` can never show you the nested collection keys; take them from `print_templates_sources`, which reads the document shape directly. Fields that exist only inside a secondary collection are a validator **warning**, not an error, because `/schema` lists flat columns only.
- `presentation` describes the preview table, not the template. Reuse only its labels (`devirLabel`, `pageTotalLabel`, `grandTotalLabel`); never derive a header band from `columnGroups` or a devir row from `showDevir` — the devir row is data the source itself emits. `showPageTotals: false` likewise forbids nothing: two shipped templates declare it and still carry a `PageBreakRow` footer.

## Path 4 — Record print

Sixteen types. There is **no mail and no export** on this path — only browser printing.

| `type` | Endpoint | `payload_key` |
|---|---|---|
| `invoice` | `POST invoices/print` | `invoices` |
| `way_bill` | `POST waybills/print` | `waybills` |
| `order` | `POST orders/print` | `orders` |
| `storage_slip` | `POST storage-slips/print` | `storage_slips` |
| `cheque` | `POST finance/cheques/print` | `cheques` |
| `accounting_slip` | `POST accounting/slips/print` | `accounting_slips` |
| `customs_clearance_slip` | `POST customs-clearance-slips/print` | `customs_clearance_slips` |
| `customer_slip` | `POST customers/slips/print` | `customer_slips` |
| `bank_transaction` | `POST finance/banks/transactions/print` | `bank_transactions` |
| `cheque_roll` | `POST finance/cheque-rolls/print` | `cheque_rolls` |
| `cost_slip` | `POST costSlips/print` | `cost_slips` |
| `sale_provision_slip` | `POST costSlips/print` | `cost_slips` |
| `customer_slip_line` | `POST customers/slip-lines/print` | `customer_slip_lines` |
| `bank_transaction_line` | `POST finance/banks/lines/print` | `bank_transaction_lines` |
| `safe_line` | `POST safes/lines/print` | `safe_lines` |
| `accounting_slip_line` | `POST accounting/slips/lines/print` | `accounting_slip_lines` |

`customer_slip`, `safe_line` and `bank_transaction` were dropped on 2026-08-10 and came back
on 2026-08-25 with endpoints of their own, alongside six new siblings.

Two shapes live on this path, and they bind differently:

- **Slips** print a document header WITH its lines, so they carry a nested dataset — `lines`
  for every one of them except the cheque roll, whose lines are `transactions`
  (`$dataset:DataSet/transactions`).
- **The four movement receipts** (`*_line`) print ONE row as a single-page makbuz. The record
  IS the line, so they have a single `DataSet` and **no** nested collection at all; a
  `$dataset:DataSet/lines` there resolves to nothing.

**Maliyet dağıtım fişi and satış provizyon fişi share one endpoint** and one payload key, and
differ only by the record's `type` — so they are two template types over one channel, and the
endpoint refuses a bulk print that mixes them.

Body: `{"<payload_key>": [{"id": 1}], "print_template_id": <optional>}`. **Nothing here is derivable from the slug** — a waybill is `way_bill` as an enum, `waybills` in the URL and payload, `way_bills` as the viewer alias. Take the endpoint and key from `print_templates_sources(path='record')`.

- `DataSets[0]` binds with `jpath=$.[*]` (cheque: `jpath=$`, a single object). Nested lines come from `$dataset:DataSet/lines`; invoices add `vat_indicator`, `custom_fields`, `customer.addresses`, and chains like `$dataset:lines/slip.lines`.
- One page per record is **correct** here, but it does not happen by itself: a body-level `table`/`list` must be bound to the primary dataset. Without it "Çoklu Yazdır" silently prints only the first record. Get it with the flat body + empty `RecordAnchor` list from `skeletons.md` — wrapping the body in a `RecordBlock` list also binds the primary but destroys the page-1 header band, so the line table prints over the header boxes.
- These types are not in the print source registry: `/reports/print/{invoice|order|…}/*` returns 404, so there is **no machine-readable column catalog**. Field names come from an existing template or from a real print response, and `print_templates_sources(path='record')` reports both facts per type as `has_existing_template` and `sample_print_possible_without_template`. Only `cheque` prints without a template; the others answer 400. `way_bill` and `accounting_slip` ship no starter template at all, and the `storage_slip` one is not distributed (its file is not named `*.json`, so nothing picks it up) — on those three a tenant offers no field source whatsoever. Then stop and ask rather than invent fields. The nine types added on 2026-08-25 all ship a starter template, so their field names can be read straight off it.
- Template resolution runs a chain, and **not every step exists for every type**: the record's own `print_template_id` only for `invoice`/`way_bill`; `condition` + `wildcard_filters` practically only for invoices (other types get an empty parameter bag, so every P1xx/P2xx condition falls through); the customer's default works only where the card really is a cari — `invoice`, `way_bill`, `order`, `customer_slip` and `customer_slip_line` — and is a no-op everywhere else, the safe/bank/accounting movements included; the category default (`is_default`) is the one step that always works. **On the nine types added on 2026-08-25 the category default is the ONLY working step**, which is why each ships a starter template with `is_default: true`: without it every print request answers 400.
- **Cheques skip the chain entirely** — the request's `print_template_id` or the category default, nothing else. The shipped cheque template is `is_default: false`, so a seeded tenant has no cheque default at all. Ask the question and say it plainly: *"Bu türde şu an varsayılan şablon yok; siz seçmezseniz baskı şablonsuz kalır."*
- `Özel Yazdır` (picking a non-default template from the UI) exists for `order`, `invoice`, `way_bill`, `storage_slip` and `customs_clearance_slip` — though on `storage_slip` the chosen id is dropped by its request rules and the default prints instead. For `cheque` and `accounting_slip` the only way to test a new template is to make it the default temporarily — warn before doing that.
- `max_lines_per_page` is genuinely enforced on this path: on `invoices/print`, and again when a **sales invoice is saved** with this template — an invoice with more lines than the limit cannot be saved at all. Unset or 0 falls back to 100.
- The render-parity warning does **not** belong on this path: a record template has no file channel to differ from, and its one output — the viewer / browser print — is always galley (`tables-and-pagination.md`). Say *"Yazdır ile dosya çıktısı birebir aynı olmayabilir"* where it is true: a flat analytics template, and a report key whose source returns no document rows, are exported through the static renderer while the viewer shows them flowing.

## How a starter template reaches a tenant (every path)

A starter template is **printed once per type, by a tenant seed migration** under `database/migrations/tenant/`, not by a seeder: tenant provisioning runs no seeder at all, and the one seeder that does touch this table only fires while `print_templates` is completely empty. The migration's rule is the opposite of an upsert — it sees a template of that type and exits without touching anything.

Two consequences the agent must state honestly:

- **A live tenant is never re-seeded, so a stored template can never be overwritten by a re-seed.** Any advice built on that danger is wrong.
- **A fixed starter file does not reach the tenants that already got the old one.** Re-running the migration answers "Nothing to migrate"; the deployed rows stay as they are. So "dosyayı düzelttim" is only half the job — the distributed rows have to be handled separately (through these tools, per tenant), and that second half must be said out loud rather than implied.
