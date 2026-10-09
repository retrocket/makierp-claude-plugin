# Charts And Item Types

Use this for requests like:

- "Şablona ürün gruplarına göre bir pasta grafiği ekle"
- "Şablondaki grafik tablonun satırlarının üstüne biniyor"
- "Şablona ince bir ayraç çizgisi çizer misin?"

## The eight item types

The engine renders exactly these `Type` values: `textbox` · `rectangle` · `table` · `list` ·
`barcode` · `richtext` · `image` · `chart`. Any other type (`line`, `shape`, `checkbox`,
`subreport`, `tablix`, `sparkline`, `dvchart`, `bandedlist`, `formattedtext`, `inputfield`,
`tableofcontents`, `bullet`, `overflowplaceholder`) **produces no error and is never drawn**:
the dispatcher returns nothing for an unknown type, so the item is silently missing from every
page — there is no legacy fallback renderer behind it. For a rule or divider use a
`rectangle`, or an empty `textbox` with a single border side.

Barcodes are **Code 128 only** (`Code_128auto`, `Code_128_A/B/C`) and no caption is drawn. The
engine never reads `Symbology`: a `Code_39` or `EAN13` item is not rejected — it prints a
**wrong Code 128 barcode**, i.e. an unreadable or mis-scanning label.

`print_templates_validate` is the ONLY guard for both cases — an unsupported item `Type` and a
non-Code-128 `Symbology`. Run it before writing; the runtime will not warn you.

## Chart item shape

```jsonc
{
  "Type": "chart", "Name": "Grafik1", "DataSetName": "Rows",
  "Chart": {
    "Kind": "pie|bar|line|area|combo|scatter|heatmap|pivot",
    "Config": { /* analytics visualization config JSON */ },
    "Columns": [{ "key": "…", "label": "…", "type": "number" }],
    "Format": { "columns": { /* optional FormatConfig map */ } }
  },
  "Left": "0cm", "Top": "0cm", "Width": "19cm", "Height": "11cm"
}
```

The template stores the definition only — each render draws it from the bound collection's
current rows. `DataSetName` names the collection the chart reads, exactly as on a table: on
the report and analytics paths bind it to the nested dataset the body table already uses
(`Lines`, `Rows`), not to the primary. Omitted, it falls back to the primary collection.
Unlike a table or list, a chart bound to the primary does **not** turn the section into one
page per row. `pivot` is not an SVG chart: it renders as a plain HTML table.

## Placement — the rule that breaks templates

A chart is **absolute chrome**: it does not flow in the galley, it is never split across
pages, and it does not consume the paginator's page budget. It must be placed **above** the
flowing table — but placing it above is not enough on its own. The chart rides in the page-1
chrome band, and that band has three conditions (`tables-and-pagination.md`, "The page-1 chrome
band"); the one a chart usually breaks is the third: the band is created **only when the first
flowing table's `Top` is positive**. With `Top: "0cm"` the split is skipped, everything falls
back into the flow, and the chart is stamped over the rows. The band's height is exactly that
`Top` value.

Rule: if the chart starts at `Top: "0cm"`, the flowing table's `Top` must be at least
`chart.Top + chart.Height`.

## Sizing

The box decides how much chrome (legend, labels, padding) the chart may spend, and the
**constraining** dimension wins (OR semantics): under 300px wide OR under 200px tall → mini;
under 560px OR under 320px → compact; otherwise full. `Width`/`Height` convert at 96dpi
(1cm ≈ 37.8px), so in template units: under ~7.9cm wide or ~5.3cm tall is mini, under ~14.8cm
wide or ~8.5cm tall is compact. A wide-but-short 19×5cm box is squeezed as hard as a narrow
one — give a chart height, not only width.

On paper the legend is never paged: all entries are laid out at once and the drawing is
scaled uniformly into the box. A box too small for its series count does not drop entries, it
shrinks them until they are unreadable.

## Data and compatibility

- Charts **never appear in xlsx output**. Anything the user needs in the spreadsheet must also
  exist as table/textbox data.
- The column keys inside `Chart.Config` (`value_column`, `x_column`, `series_column`,
  `size_column`, `rings`, `columns`, `row_fields`, `column_fields`, `value_fields`,
  `series[].column`) ARE part of the print compatibility check. The `Chart.Columns` metadata
  is deliberately excluded from it.

## Do not use, do not propose again

- `Chart.PerRow` / `Chart.RowMode` still exists in the engine for backward compatibility but
  was removed from the production UI. It is the one chart flag that DOES force one page per
  data row — a 400-row statement becomes 400 pages. Never put it in a new template.
- Chart starter blocks are switched off in the product (`CHART_STARTERS_ENABLED = false`). Do
  not offer a chart on your own initiative — add one only when the user asks, and say plainly
  what it will show: "Şablona ürün gruplarına göre bir pasta grafiği ekleyeceğim."
- Charts embedded as static captured images (tested by the user, rejected, capture path removed).
- The `RdlTable.DataTable` config layer (removed — analytics table settings translate into
  native RDL structures instead).
- Scroll legends or tier-based hiding on paper: a printed chart cannot be paged through, so
  every legend entry has to be visible at once.
