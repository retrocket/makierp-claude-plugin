# Table And Pivot Visualization

Use this for row lists, grouped summaries, cross-tabs, export-ready tables, and dashboard table cards.

## Table

Use `selected_viz:"table"` or export `render_mode:"table"` with config under `visualization.table`:

```json
{
  "table": {
    "density": "default",
    "column_order": ["customer_code", "customer_name", "net_total_ypb"],
    "hidden_columns": ["customer_id"],
    "show_row_numbers": true,
    "show_zebra_rows": true,
    "show_cell_borders": false
  }
}
```

Allowed table keys are `density`, `column_order`, `hidden_columns`, `show_row_numbers`, `group_by`, `group_totals`, `grand_total`, `show_cell_borders`, `show_zebra_rows`, and `conditional_formatting`.

Use `density:"compact"` for dense operational lists, `default` for normal reports, and `comfortable` only for sparse dashboard views.

## Grouped Table And Totals

```json
{
  "table": {
    "group_by": [["region"], ["customer_group"]],
    "group_totals": [
      {"net_total_ypb": "sum"},
      {"net_total_ypb": "sum"}
    ],
    "grand_total": {"net_total_ypb": "sum", "invoice_count": "sum"},
    "column_order": ["region", "customer_group", "customer_name", "net_total_ypb"]
  }
}
```

Totals are per-column aggregator maps, not booleans. `grand_total` maps column keys to an aggregator for the trailing total row; `group_totals` is an array index-parallel to `group_by` — entry N configures the subtotal row of group level N, and a missing or empty entry disables that level's subtotals. Aggregators are `sum`, `avg`, `median`, `min`, `max`, and `count`. Number/currency columns accept all six; date columns accept `avg`, `median`, `min`, `max`, and `count`; other columns accept only `count`. Columns absent from a map get no total cell.

Only group by columns present in preview output. Prefer `sum` for monetary and quantity columns; use `count` to show row counts on a text column.

## Conditional Formatting

```json
{
  "table": {
    "conditional_formatting": [
      {
        "column": "overdue_amount_ypb",
        "operator": "gt",
        "value": 0,
        "target_columns": ["customer_name", "overdue_amount_ypb"],
        "highlight_row": false,
        "format": {"mode":"single","background":"#FEF2F2","color":"#991B1B","bold":true}
      }
    ]
  }
}
```

Operators are `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `contains`, `not_contains`, `is_null`, `not_null`, and `between`.

## Pivot

Use `selected_viz:"pivot"` or export `render_mode:"pivot"` when rows need cross-tab summarization:

```json
{
  "pivot": {
    "row_fields": ["salesperson_name"],
    "column_fields": ["month"],
    "value_fields": ["net_total_ypb"],
    "show_row_totals": true,
    "show_column_totals": true
  }
}
```

`row_fields`, `column_fields`, and `value_fields` must be preview column keys. Pivot exports support `csv`, `xlsx`, and `pdf`.

## Export Fit

- `table` works well with `json`, `csv`, `xlsx`, and `pdf`.
- `pivot` works well with `csv`, `xlsx`, and `pdf`.
- For Excel, use `export_options.xlsx_sheet_name` and `export_options.xlsx_as_table` when the user cares about workbook polish.
- To deliver a TABBED workbook instead of one flat sheet, set `export_options.xlsx_split_by` to a preview column key: each distinct value becomes its own worksheet, in first-appearance order, and `export_options.xlsx_split_summary` (default `true`) appends an `Özet` sheet with one row per worksheet plus a grand total. Each sheet closes with its own total row — the question's `visualization.table.grand_total` spec when it declares one, otherwise a `sum` over the currency columns. Splitting needs `render_mode: "table"` and `format: "xlsx"`, and is refused alongside `template_id`. Reach for it when the user asks for a report "per currency", "per warehouse", "per representative" as separate pages.
