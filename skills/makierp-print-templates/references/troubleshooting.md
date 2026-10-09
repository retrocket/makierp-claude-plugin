# Print Template Troubleshooting

Use this for requests like:

- "Şablonu yazdırınca sayfalar bomboş çıkıyor"
- "Faturayı bastırınca her satır ayrı sayfaya gidiyor"
- "Excel'e aktarınca hata verdi, neden?"

The fixes below are the short form; the rule each one comes from lives in the matching
reference (`expression-language.md`, `tables-and-pagination.md`, `data-paths.md`).

## Run These First

- `print_templates_validate` — static, saves nothing. Catches unit-less lengths, column-width and `ColSpan` sums, unsupported item types, missing/unknown `DataSetName`, a root `Body` next to `ReportSections`, dead config keys (`Filters`, `SortExpressions`, `KeepTogether`, …), and every expression through the engine's real lexer/parser (`IIf` case, single quotes, `Fields("x")` call form, `IIF`/`IsNull` arity, `Parameters!`, `Globals!PageNumber` arithmetic), plus field references against the source's columns. Always pass `mode` and — on paths 2 and 3 — `columns`. Read `skipped_checks`: a clean `ok` with a non-empty `skipped_checks` is not a clean bill of health.
- `print_templates_dry_run` — runs a **saved** template against real data, stops at the HTML stage, produces no file and no "Dışa Aktarımlar" entry. It measures what static checks cannot: `page_count == row_count` — the per-row-page accident on the report and analytics paths, and the *correct* result on record printing, so read it against the path — plus swallowed galley render errors (`render_errors`), empty pages and missing fields. Feed it what the path needs: `params` / `context` for a report key (the same real record and date range the field discovery used), and **`question_id`** on the analytics path — without it the run has no rows to render. On record types the run has no record to bind and comes back with `row_count: 0`; treat that as "not exercised", not as a pass.
- `print_templates_get(summary_only: true)` → `summary.body_items` — the **structural** view, and the only cheap way to see what the layout is actually made of: which regions sit at body level, which one binds the primary, and whether the flowing table (`repeats_header: true`) is up there or buried inside a container. Every layout symptom in "Layout And Channel" is diagnosed from this list. It is not the cheap read, though: `summary_only` lists **every** declared field, so on a field-rich template it can exceed the tool's output limit and be dropped to a file — then continue with `jq '.summary.body_items'`. For field **names** use `fields_only` instead.
- Order: validate before every create/update, dry_run right after saving — never tell the user the template is ready on a save alone.

## Warnings That Are Not Your Fault

- `DEFINITION_ENVELOPE` ("Doğrulamaya zarf gönderildi; Araçlara yalnız RDL tanımını geçin") on an **update** while you sent a bare definition: the stored `content` is itself a `{displayName, definition}` envelope, and the update path compares against it, so the warning describes the **stored** content while its wording blames the caller. Harmless — do not "fix" it by reshaping your payload. The same call on `create` with the same definition produces no warning; that asymmetry is the tell.

## Nothing Prints, Or The Job Fails

| Symptom | Cause | Fix |
|---|---|---|
| Export/mail job FAILED (static path) | One syntax error — single quotes, `Fields(`, unbalanced parens, `IIF`/`IsNull` with fewer than 3 arguments. A single bad expression drops the **whole** document, not just that cell | Catch it with `print_templates_validate`; render is not a safety net |
| Job reported success but the PDF is blank / viewer opens with 0 pages | Same class of syntax error on the galley path, where the throw is **swallowed** (`__reError` is never read on the backend) | Worse than FAILED — nobody sees an error. Validate statically; `dry_run` surfaces it as `render_errors` |
| Excel / CSV / JSON export of a report returns 422 (sync) or FAILED (queued) | There is no template of that type at all — the template gate runs **before** the format switch | Format-independent: a template must exist first, even though those formats ignore its layout |
| An export that ran fine for months fails on one record | A 2-argument `IIF` only throws when the condition comes out **false**; a data set where it was always true hid the arity error | `IIF`/`IsNull` always take exactly 3 arguments. This one is caught statically only — a dry run on friendly data cannot rule it out |
| An item never appears anywhere | Unit-less length (`"120"`) resolves to null → 0pt wide; or the item's `Type` is not one of the eight the engine renders, and the dispatcher silently returns nothing | Always write units (`"12cm"`, `"340pt"`) and keep every item to a supported type |

## Rows And Pages Come Out Wrong

| Symptom | Cause | Fix |
|---|---|---|
| Every data row on its own page | A body-level table/list is bound to the primary dataset (paths 2 and 3) | Bind it to the nested dataset (`$dataset:Statement/lines`, `$dataset:Question/rows`) |
| Table shows only the first row | The table/list has **no** `DataSetName` → the region renders once against the current scope row (silent data loss) | Declare `DataSetName` on the region |
| N pages print and all of them are empty | The primary is not named `DataSet`, but the table says `DataSetName:"DataSet"` — the alias only affects the pagination decision, never data resolution | Write the real primary name (`Statement`, `Question`) |
| "Çoklu Yazdır" silently prints only the first record | Path 4 body has no **body-level** table/list bound to `DataSet` — or the bound table is wrapped in a `rectangle`/`list` container, which breaks the binding | Add an empty anchor as a direct child of Body: `{Type:'list', Name:'RecordAnchor', DataSetName:'DataSet', Width:'0cm', Height:'0cm', ReportItems:[]}`. Do **not** wrap the body in a list to fix this — it trades the missing records for a broken layout (next row) |
| Table does not paginate, overflowing rows disappear | No `Header.RepeatOnNewPage` → the table is an absolute single-page canvas and clips | `Header.RepeatOnNewPage: true` |
| Page-boundary rows ("Ara Toplam" / "Önceki Sayfadan Devir") are missing from the exported PDF, but the viewer shows them | The **export** runs on the non-galley path — a report source that returns no document rows, or a flat (non-nested) analytics template. Marked rows are removed from normal flow in **every** mode, so there they vanish entirely. Not a record-print symptom: record templates only ever render in the viewer, which is always galley | Drop the page-band rows, or move the source to document shape. The designer cannot see this loss, because the viewer is galley either way |
| Grand total repeats on every page and is missing at the end | The grand-total row carries `PageBreakRow:"Footer"` | Remove the marker from that row |

## Cells Show The Wrong Value

| Symptom | Cause | Fix |
|---|---|---|
| Conditional cell is silently blank | `IIf` / `iif` in lowercase — an unrecognized call name returns null with no error | `IIF` in uppercase; every builtin name is case-sensitive |
| Cell prints `[object Object]` | Dotted bang: `Fields!position.code.Value` — the engine treats `.code` as a property and ignores it | `Fields.Item("position.code").Value` |
| Total cell is empty | The value was never embedded on the rows; there are no report parameters | Backend `rows()` must embed `total_*` on every row, read with `=First(Fields!total_x.Value)` |
| Group subtotal is identical in every group (the first group's number) | Argument-less `First(...)` in a group band reads the **root** collection, not the group scope | `=Fields!account_subtotal_debit.Value`, or argument-less `=Sum(Fields!debit.Value)` |
| Cell prints `"NaN"` | Arithmetic on a missing or empty-string field | `IIF(IsNumeric(x), x, 0)` — `IsNothing` is not enough, it only covers null/undefined |
| Page-number arithmetic or comparison does not work | In flowing mode `Globals!PageNumber` / `!TotalPages` are sentinels, not numbers | Concatenate with `&` only: `="Sayfa: " & Globals!PageNumber & " / " & Globals!TotalPages` |
| Boolean column always prints "Evet" | Every string except `'true'`, `'false'` and `''` is truthy, so a `'0'` flag is true | `=IIF(Fields!f.Value = 1, "Evet", "Hayır")` when the flag arrives as text; ask the user if the column's real shape is unknown |
| Enum column prints `0` | The backend row carries an int-backed PHP enum's `->value` | `$enum?->prettyName() ?? ''` — a backend fix, not a template fix |

## Layout And Channel

| Symptom | Cause | Fix |
|---|---|---|
| The line table climbs to the **top of the page** and prints over the absolutely-positioned header boxes | The page-1 chrome band was never built. Read `print_templates_get(summary_only)` → `body_items`: if the repeating table (`repeats_header: true`) is not a body-level item — typically because the body was wrapped in a `list` — no band is created and the table loses its absolute position, so it draws from y=0. The other two ways to lose the band: no non-flowing item beside the table, or the table's `Top` is `0cm` (`tables-and-pagination.md`) | Flatten the body: the table and the header boxes both become direct children of Body, the table keeps a positive `Top`, and an empty `RecordAnchor` list carries the primary binding (`skeletons.md`) |
| Columns shifted, right-hand ones cut off | `Σ TableColumns[].Width ≠ table.Width`, or a band row whose `Σ ColSpan ≠ column count` | Restore both invariants; the browser silently redistributes mismatched widths |
| Text clipped to a single line | No `CanGrow` on the textbox | `CanGrow: true` |
| Chart overlaps the table | Charts are absolute chrome and never flow; the page-1 chrome band only forms when the first flowing table's `Top` is positive (`tables-and-pagination.md`) | Put the chart above **and** set the flowing table's `Top` to at least `chart.Top + chart.Height` |
| Junk columns in the xlsx | Meta fields were declared in `DataSets[0].Fields` | `Fields` lists only the visible columns, in export order, with their labels |
| Printed barcode does not scan | `Symbology` is never read — every barcode item is drawn as Code128, so a `Code_39`/`EAN13` item is not rejected, it prints the wrong encoding | Only `Code_128auto`/`_A`/`_B`/`_C` are real; another symbology cannot be printed from a template at all |
| A seed-file edit has no visible effect | Someone edited the JSON under `print-templates/`; the resolver reads the tenant's DB row, not the file. A live tenant is **not** re-seeded and its seed migration exits as soon as a template of that type exists, so a corrected starter file never reaches a tenant that already has one (`data-paths.md`) | Fix the stored row itself — `print_templates_get` → `print_templates_update` — per tenant. Never report "dosyayı düzelttim" as the end of the job. Templates written through these tools go straight to the DB row and never have this problem |
| "Şablonum uygulanmadı" on an analytics export | The format is `csv`, `json`, `svg` or `png`; the template gate only fires for `pdf` and `xlsx` | Choose `pdf` or `xlsx`; the built-in layout is used otherwise, with no error |
| Analytics export rejected outright | The gate is **reference**-based: every `Fields!x` / `Fields.Item("x")` inside an expression, plus every `Chart.Config` column key, must exist among the columns the question produces today | Rebind the expressions themselves — editing only the `DataSets[0].Fields` list leaves the export failing. Keep that list in step too, since it is the xlsx column plan |

## Stranded Analytics Template

An `analytics_export` template whose columns have drifted cannot be reached from any UI: the type is hidden from the "Rapor Şablonları" index by a server-side filter the user cannot lift, and the "Şablon ile Çıktı" modal — the only place with a delete action — drops incompatible templates from its list. MCP is the recovery path: `print_templates_get` by id → rebind every field reference to the question's current column keys → `print_templates_update`, or `print_templates_delete` if it should go. This is also why column validation is blocking, not advisory, before saving an analytics template.

## Reporting Back

Describe the symptom and the fix in the user's terms, never with identifiers or file paths: "Çıktı boş geliyordu çünkü bir hücrenin formülü hatalıydı; düzelttim ve şablonu gerçek veriyle denedim." Ask instead of guessing when the fix depends on data shape (flag columns, enum columns, whether a source returns document rows).
