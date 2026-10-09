# Tables, Bands and Pagination

Use this for requests like:

- "Ekstrede her sayfanın sonuna ara toplam, başına devir satırı koy"
- "Muavin dökümünde hesap başlığı devam sayfalarında da görünsün"
- "Tablo sayfaya sığmıyor, kolonlar kayıyor"

Dataset shape and skeletons: `skeletons.md`. Band-cell expressions: `expression-language.md`.
Symptom lookup: `troubleshooting.md`.

## The galley gate — read this before designing any band

Flowing tables, page-boundary rows and group carries only materialise in **galley** (flow)
rendering: the engine gates them on `env.galley && flow` (`render/items.tsx`, the `bandCells`
and `anyGroupRepeats` guards) — but flagged rows leave the normal flow in **every** mode.

**Galley is a property of the RENDER PATH, not of the document type.** The same template can be
galley in one channel and paged in another, so always ask "which channel is this output coming
out of?" before deciding whether a band will print.

| Render path | Galley? |
|---|---|
| **FE** — viewer preview and "Yazdır" (browser print), **any** template type | **Always.** The viewer is `PaginatedReportView` and the print iframe is `printReport`; both build `reportToGalleyHtml` and run `paginateGalleys` |
| **BE** "Dışa Aktar" / "Mail" — report key | Only when `format = pdf` **and** the source implements `ProvidesDocumentRows` (`PrintController`, `PrintTemplateExportProducer`) |
| **BE** "Dışa Aktar" / "Mail" — analytics | Only for the **nested-lines** shape (`AnalyticsExportProducer` `$isNested`); a FLAT analytics template is never galley |
| **BE** any xlsx / csv / json export | Never — galley is PDF-only |
| **BE** report key without document rows (`accounting_account_statement`) | Never — it is the one report source without document rows |

**The intersection to know: a record template (fatura / irsaliye / fiş) always renders galley**,
because record printing has no backend render path at all — `POST /{endpoint}/print` returns JSON
rows and the FE viewer renders them. So its bands do print. The "records are non-galley" rule
belongs to the backend `printByCategory` branch, which record screens never reach: those types
are not registered as print sources, and both callers of that branch gate on a report source.

On a non-galley path a table never flows at all (`regionFlows` in `items.tsx` returns false
without `env.galley`): a `PageBreakRow` row prints **nowhere**, and rows past the
first page are **clipped**. The designer never sees that loss, because the viewer is always
galley. Do not put page bands or group carries in a template whose channel is a "Never" row —
in practice, a report/analytics template that is exported rather than printed from the viewer.

## The page-1 chrome band — why an absolute header block and a flowing table collide

A record or report page is usually built the same way: absolutely positioned header boxes at the
top, one flowing table below them. What keeps the two apart on page 1 is a band the engine opens
**only** when all three of these hold (`splitGalleyBody`, `render/ReportView.tsx`):

1. the table with `Header.RepeatOnNewPage` is a **body-level** item (not inside a `list` or
   `rectangle`), **and**
2. there is at least one non-flowing item beside it (the chrome), **and**
3. that first flow table's **`Top` is positive**.

Then the chrome is emitted inside `[data-re-firstband]`, whose height is exactly that `Top`, and
it prints on page 1 only; from page 2 on, the table starts at the top of the page. Break any of
the three and the band is never created: everything falls back into the flow, the flowing table
loses its absolute position and is drawn from y=0 — **over** the header boxes.

This is the standard record-template failure and it has nothing to do with charts: condition (1)
is what a `RecordBlock` wrapper destroys (`skeletons.md`), and condition (3) is what a chart at
the top of the page competes with (`charts.md`). Diagnose it with
`print_templates_get(summary_only)` → `body_items`: if the repeating table is not listed there,
no band was ever built.

## Band structure

```json
{"Type":"table","Name":"statement","DataSetName":"Lines","TableColumns":[{"Width":"4cm"}],
 "Header":{"RepeatOnNewPage":true,"TableRows":[]},
 "Details":{"TableRows":["ONE row — the engine repeats it per data row"]},
 "Footer":{"TableRows":[]}, "TableGroups":[], "NoRowsMessage":"Kayıt bulunamadı"}
```

**`Header.RepeatOnNewPage: true` is the flowing-table switch** (`items.tsx`
`const startsFlow = !!item.Header?.RepeatOnNewPage`). Without it the table is an absolute,
single-page canvas and overflowing rows are **silently clipped** — data loss, no warning. With
it you get the repeating `<thead>`, break units, page-boundary rows and group carries.
`NoRowsMessage` is honoured on `table` only (the table renderer in `items.tsx`);
`List.NoRowsMessage` parses but
is never read.

## Page-boundary rows are a ROW CONFIG, not a separate structure

`pageBandRows` (`items.tsx`) filters the ordinary band rows on one key, `PageBreakRow`:

- `"Footer"` on a **Footer** row → the bottom "Ara Toplam" band. `"Carry"` on a **Header** row
  → the top "Önceki Sayfadan Devir" band, stamped with the **previous** page's boundary values.
- Flagged rows **never print in the normal flow**, and their cells must be plain textboxes.
- Values are **not aggregates**. Each cell reads a precomputed running field carried on every
  data row (`=Fields!cum_debit.Value`). The backend embeds those `cum_*` fields; they stay out
  of `DataSets[].Fields` so xlsx stays clean.
- **Never flag the grand-total row.** A flagged row leaves the flow, so marking it deletes the
  closing total from the last page.

**In a grouped table `"Footer"` is not "every page".** When a group header repeats
(`RepeatOnNewPage: true`, the muavin/kebir pattern) the paginator makes the band group-aware
(`paginate.ts` `injectPageBands` + its `footerOverride`): "Ara Toplam" prints only on pages
whose **bottom group continues** onto
the next page. If the group ends on that page nothing prints — the group Footer is already the
closing row. In a flat flowing table the plain rule holds: every page except the table's last.

**Dead shape:** the `Table.PageBreakFooter` / `Table.PageBreakCarry` **sections** are never read
(`rdl/types.ts`, `@deprecated`); they survive only for the designer's open-time migration.

## Grouping (`TableGroups`)

```json
"TableGroups":[{
  "Group":{"GroupExpressions":["=Fields!account_code.Value"]},
  "Header":{"RepeatOnNewPage":true,"TableRows":[["=\"Hesap: \" & Fields!account_code.Value"]]},
  "Footer":{"TableRows":[["Toplam:","=Fields!account_subtotal_debit.Value"]]},
  "ContinuationLabel":" (Devamı)"}]
```

- Every entry is a nesting **level**; all levels render, outermost first.
- **The engine never sorts or regroups the data.** Rows must arrive already sorted and
  contiguous by the group key from the backend; a group breaks wherever the key value changes.
  (Expression-level aggregates do exist — see the scope bullet below — but they only sum what
  the partition already contains.)
- A group band's scope is the group's **first row** (`items.tsx` pushes that scope), Header and
  Footer alike.
  Read a precomputed subtotal field directly (`=Fields!account_subtotal_debit.Value`) or use an
  argless aggregate (`=Sum(Fields!debit.Value)`). **Never `=First(...)` in a group band** —
  argless `First` reads the root collection, not the group, and silently prints the wrong number
  (`expression-language.md`).
- `Header.RepeatOnNewPage: true` plus `ContinuationLabel` reprints the group header on
  continuation pages as "Hesap: 760 (Devamı)". The label is config, not hardcoded.

## Keys the engine never reads

`Table.Filters`, `SortExpressions` (table/list/group), `DataSet.Filters`, `List.Group`,
`Grouping.PageBreak`, `RepeatToFill`, `KeepTogether`, `ConsumeWhiteSpace`, `CanShrink`,
`Style.ShrinkToFit`, `PageHeader.PrintOnFirstPage` / `PrintOnLastPage`,
`TableSection.PrintAtBottom`, `Report.Layers`, `Page.Columns`. Writing them makes the template
*look* filtered or sorted while the data prints exactly as it arrived. **Filtering and sorting
happen on the data side**, never in the definition.

## Page geometry

Defaults (`rdl/parse.ts` `DEFAULT_PAGE`): `8.5in × 11in`, 1in margins. Resolution is
**per property**:
`ReportSections[i].Page` > `report.Page` > default. `order.json` declares `8.5in × 11in` at the
root and `8.27in × 11.69in` on the section — the section wins.

**Units:** `in | cm | mm | pt | px` only (`LENGTH_RE`, `layout/units.ts`), or a bare `0`. A
bare number (`"120"`) parses to `null` → `toPt` returns 0 → **the item is invisible**. CSS
units such as `%` or `em` count as 0 in geometry math and break pagination. Use one unit per
template (report family `cm`, record family `in`).

**Content width = `PageWidth − LeftMargin − RightMargin`**

| Page | Size | Content |
|---|---|---|
| A4 portrait | 21 × 29.7 cm | **19 cm** |
| A4 landscape | 29.7 × 21 cm | **27.7 cm** |
| Wide statement (9–11 columns) | 31.5 × 44.6 cm | **29.5 cm** |
| Kebir (landscape) | 42 × 29.7 cm | **40 cm** |

**Two invariants — violations are silent:**

1. `Σ TableColumns[].Width == table.Width`. The table renders as `tableLayout: fixed` with an
   explicit `<colgroup>` (`items.tsx`), so a mismatched total gets redistributed by the
   browser: columns drift and the right-hand ones get cut. Matching the *content* width is
   **not** required — record templates deliberately use a narrow, offset table (`order.json`:
   section 8.27in, margins 0, table 7.6667in at `Left 0.8cm`). The second rule is a bound, not
   an equality: `table.Left + table.Width <= content width`.
2. Every band row: `Σ ColSpan (default 1) == TableColumns.length`.

`print_templates_validate` checks both, plus unitless lengths, a missing
`Header.RepeatOnNewPage`, misplaced `PageBreakRow` flags, the dead sections and the dead keys
above — run it before every save.

**`CanGrow`:** a textbox without it renders `white-space: nowrap` + `overflow: hidden`
(`items.tsx`) — one line, overflow clipped. Set `CanGrow: true` on every cell that must
wrap (descriptions, addresses, account names).
