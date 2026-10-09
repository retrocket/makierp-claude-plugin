# Template Skeletons

Use this for requests like:

- "Sipariş formu şablonu hazırla"
- "Cari hesap ekstresi için yeni bir şablon yap"
- "Defter-i kebir çıktısında bir de özet tablosu olsun"

Copy the skeleton that matches the data path (`data-paths.md`), fill it, then run
`print_templates_validate` before `print_templates_create`. Every JSON block below is the
**definition object only** — it goes inside the `{"displayName", "definition"}` envelope, itself a
JSON string in `content` (see `template-envelope.md`).

## Rule zero — the two skeletons never mix

Two shapes only; every record template uses (A) and every report template uses (B).
**When `ReportSections` is present and non-empty the engine ignores the root-level `Body`,
`PageHeader` and `PageFooter` completely** (`rdl/parse.ts` `buildSection` reads only `section.*`;
the root body feeds a synthesized `Continuous` section only when `ReportSections` is missing or
empty). So (A) puts **everything** inside `ReportSections[0]`, and (B) must not contain
`ReportSections` **at all**. The root `Page` is still read, but **per property**: a section `Page`
key overrides the root's same key, missing keys fall through.

## Skeleton A — record printing (path 4: invoice, order, waybill, storage slip, customs clearance slip, accounting slip, cheque)

Canonical example: `customs-clearance-slip.json`. The body is **flat**: the header textboxes and
the line table are both direct children of `Body`, and an empty, zero-size `RecordAnchor` list
carries the primary binding that gives every record its own page.

```json
{
  "Name": "Millileştirme Fişi", "Width": "0in", "Layers": [{ "Name": "default" }],
  "CustomProperties": [{ "Name": "DisplayType", "Value": "Page" }, { "Name": "SizeType", "Value": "Default" }, { "Name": "PaperOrientation", "Value": "Portrait" }],
  "Page": { "PageWidth": "8.5in", "PageHeight": "11in", "TopMargin": "0in", "RightMargin": "0in", "BottomMargin": "0in", "LeftMargin": "0in", "Columns": 1, "ColumnSpacing": "0.5in" },
  "DataSources": [{ "Name": "DataSource" }], "DataSets": [
    { "Name": "DataSet", "Query": { "DataSourceName": "DataSource", "CommandText": "jpath=$.[*]" }, "Fields": [{ "Name": "slip_no", "DataField": "slip_no" }] },
    { "Name": "lines", "Query": { "DataSourceName": "$dataset:DataSet/lines" }, "Fields": [{ "Name": "name", "DataField": "name" }] }
  ],
  "ReportSections": [{
    "Type": "Continuous", "Name": "ContinuousSection1", "Width": "20.283cm",
    "Page": { "PageWidth": "8.27in", "PageHeight": "11.69in", "TopMargin": "0in", "RightMargin": "0in", "BottomMargin": "0in", "LeftMargin": "0in", "Columns": 1, "ColumnSpacing": "0in" },
    "Body": { "Height": "4.1cm", "ReportItems": [
      { "Type": "table", "Name": "Table1", "DataSetName": "lines", "Left": "0.8cm", "Top": "3.5cm", "Width": "7.6667in", "Height": "0.5in", "TableColumns": [], "Header": { "RepeatOnNewPage": true, "TableRows": [] }, "Details": { "TableRows": [] } },
      { "Type": "textbox", "Name": "ValueLeft1", "Value": "=Fields!slip_no.Value", "CanGrow": true, "Left": "0cm", "Top": "0.844cm", "Width": "6cm", "Height": "0.5cm" },
      { "Type": "list", "Name": "RecordAnchor", "DataSetName": "DataSet", "Left": "0cm", "Top": "0cm", "Width": "0cm", "Height": "0cm", "ReportItems": [] }
    ] }
  }]
}
```

Table columns/rows/cells are left empty for brevity — fill them from `tables-and-pagination.md`.
Field names in the example are the real keys of `customs-clearance-slip.json`; take yours from the
sources listed in the skill, never from intuition. Cheques are the one variant:
`"CommandText": "jpath=$"`, since that endpoint returns one object.

### Why flat, and never a `RecordBlock` wrapper

The engine needs a **body-level region bound to the primary** — that is the whole requirement, and
`bodyHasPrimaryRegion` accepts a `table` **or** a `list`, so a zero-size empty list satisfies it
without drawing anything. The six invoice templates get it a second legitimate way: their
body-level `Table1` binds to `DataSet` directly. Do not re-wrap a working invoice template.

Wrapping the body in a `list RecordBlock` also satisfies the binding — and **breaks the layout**.
The page-1 header band is built by `splitGalleyBody`, which scans **body-level items only** for a
table with `Header.RepeatOnNewPage`. Put that table inside the wrapper and the body has exactly one
item (the list), no flow table is found, no band is opened, and `regionFlows` then pulls the
repeating table into the flow and strips its absolute position: the line table is drawn from y=0
and prints **on top of** the header boxes. This was measured on the real engine — wrapper: pages per
record ✓ but header band ✗; flat + `RecordAnchor`: both ✓.

`order.json` in the seed folder still carries the old wrapper. It is not a counter-example: the
wrapper was added the day **after** the order template was seeded, and a seed migration never
updates an existing row, so that shape has never been rendered in production. The order template
tenants actually print is flat.

Templates already stored in a tenant may predate all of this and bind nothing at all: if
`print_templates_get(summary_only)` shows no primary-bound region among `body_items`, that tenant
prints only the first record of a multi-record selection — and correcting the seed file does not
correct the stored row (`data-paths.md`).

## Skeleton B — reports and statements (paths 2 and 3)

Canonical example: `customer-statement.json`. The body table binds to the **nested** dataset,
never to the primary. `PageHeader` is optional (`accounting-account-statement.json` has none).

```json
{
  "Name": "Cari Hesap Ekstresi", "Width": "31.5cm",
  "Page": { "PageWidth": "31.5cm", "PageHeight": "44.6cm", "TopMargin": "1cm", "LeftMargin": "1cm", "RightMargin": "1cm", "BottomMargin": "1cm" },
  "DataSources": [{ "Name": "DataSource" }], "DataSets": [
    { "Name": "Statement", "Query": { "DataSourceName": "report:customer_statement", "CommandText": "" }, "Fields": [{ "Name": "document_date" }, { "Name": "transaction_balance" }] },
    { "Name": "Lines", "Query": { "DataSourceName": "$dataset:Statement/lines", "CommandText": "" }, "Fields": [{ "Name": "document_date" }, { "Name": "transaction_balance" }] }
  ],
  "PageHeader": { "Height": "2.5cm", "ReportItems": [] },
  "Body": { "Height": "40cm", "ReportItems": [{
    "Type": "table", "Name": "statement", "DataSetName": "Lines", "Left": "0cm", "Top": "0cm", "Width": "29.5cm", "Height": "40cm",
    "TableColumns": [], "Header": { "RepeatOnNewPage": true, "TableRows": [] }, "Details": { "TableRows": [] }, "Footer": { "TableRows": [] }
  }] }
}
```

### Multi-collection variant — Defter-i Kebir

**Never assume a single nested dataset.** A source can embed as many collections as it likes in its
one document row; `accounting-general-ledger.json` uses three datasets and two body tables
(`GeneralLedgerReportSource` returns `['lines' => …, 'summary' => …]`).

```json
"DataSets": [
  { "Name": "Statement", "Query": { "DataSourceName": "report:accounting_general_ledger", "CommandText": "" } },
  { "Name": "Lines",     "Query": { "DataSourceName": "$dataset:Statement/lines", "CommandText": "" } },
  { "Name": "Summary",   "Query": { "DataSourceName": "$dataset:Statement/summary", "CommandText": "" } }
],
"Body": { "ReportItems": [
  { "Type": "table", "Name": "statement", "DataSetName": "Lines",   "Top": "0cm", "Height": "24cm" },
  { "Type": "table", "Name": "summary",   "DataSetName": "Summary", "Top": "0cm", "Height": "4cm" }
]}
```

`GET reports/print/{key}/schema` lists only the flat `columns()`, so `summary` keys are **not** in
it — read the real document shape with `print_templates_sources` before binding them.

### Analytics variant (path 2)

Same shape as (B) with `report:<key>` replaced by `question:<id>` and `Lines` by `Rows`. Every
table/chart/pivot binds to `Rows`; root textboxes (number cards) read row 0.

```json
"DataSets": [
  { "Name": "Question", "Query": { "DataSourceName": "question:42", "CommandText": "" }, "Fields": [{ "Name": "Müşteri", "DataField": "customer_name", "Type": "string" }] },
  { "Name": "Rows",     "Query": { "DataSourceName": "$dataset:Question/rows", "CommandText": "" }, "Fields": [{ "Name": "Müşteri", "DataField": "customer_name", "Type": "string" }] }
]
```

`Fields[].Name` is the **label**, `DataField` the data key, `Type` designer metadata the engine
ignores. If a result column is literally named `rows`, the nested path becomes `_rows` (then
`__rows`, …) — never let the path collide with a column key.

### One-off exception — `accounting_account_statement`

That source does not implement `ProvidesDocumentRows`, so a flat single-dataset shape is the
**correct** one for it: a body `list` bound to `Statement`, no `Lines` dataset, no `PageHeader`.
A nested `$dataset:Statement/lines` resolves to `[]` there and prints empty — do not copy this
shape for the other nine report keys.

## One page per record — the rule flips with the data path

The engine gives every primary row its own page when — and only when — a **top-level** `table` or
`list` in the body binds to the primary dataset (`ReportView.tsx` `bodyHasPrimaryRegion` matches
`DataSetName === primaryName || DataSetName === 'DataSet'`).

- **Paths 2 and 3:** body tables bind to the nested dataset — binding to the primary prints one
  page per transaction. The single exception is the flat `accounting_account_statement` shape
  above, whose body `list` binds to the primary by design and carries exactly that per-row-page
  risk; never copy it onto one of the nine document-shaped keys.
- **Path 4:** a primary-bound top-level region is **mandatory** — the `RecordAnchor` list, or the
  body-level line table itself where it binds `DataSet` (the invoice shape). Without it the engine
  builds a single instance rooted at `[data[0]]`, so "Çoklu Yazdır" quietly prints **only the first
  record** of the selection. Satisfying it by wrapping the body in a list costs the page-1 header
  band — see "Why flat, and never a `RecordBlock` wrapper" above.

| Silent mistake | What the engine does |
|---|---|
| `DataSetName` omitted on the region | Not bound to the primary at all: the region renders **once** against the current scope row (`items.tsx` `regionCollection`: `dataSetName == null` → `{rows: [scope.row], single: true}`), so a body table prints a single row — silent data loss, not one-page-per-row |
| `DataSetName: "DataSet"` while the primary is named `Statement` | The `'DataSet'` alias is honoured **only** in the pagination decision, not in data resolution (`evaluate.ts` `resolveCollection` matches `name === ctx.primaryName`, then falls back to a path read on the innermost row → `[]`). Result: N completely **blank** pages. This is exactly what mixing skeletons (A) and (B) produces |
| Primary-bound region wrapped in a `rectangle` or another `list` | `bodyHasPrimaryRegion` looks at **top-level** body items only: no per-record paging, root collapses to `[data[0]]`, only the first record prints. The primary-bound region must be a direct child of the body |
| Flowing line table wrapped in a `list` to make the pages work | Pages come out right, the **layout** does not: with no body-level flow table the page-1 chrome band is never built, and the table is drawn over the absolutely-positioned header boxes (`tables-and-pagination.md`, "The page-1 chrome band"). Keep the body flat and add `RecordAnchor` |
| Nested dataset name misspelled in `$dataset:Parent/path` | The collection resolves to `[]` and the table prints nothing (or its `NoRowsMessage`) |

One non-table trigger exists: a `chart` carrying `Chart.PerRow`. It is a backward-compatibility
flag that was taken out of the product and **never** belongs in a new template — `charts.md`.
