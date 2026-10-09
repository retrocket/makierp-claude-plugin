# Pie And Heatmap Visualization

Use this for part-to-whole charts and compact numeric matrix views.

## Pie

Pie charts work only when the category count is small and the user needs composition, not ranking. For top-N comparisons, prefer `bar`.

```json
{
  "pie": {
    "rings": ["category_name"],
    "value_column": "net_total_ypb",
    "show_legend": true,
    "show_total": true,
    "show_labels": true,
    "show_percentages": "legend",
    "decimal_places": 1,
    "min_slice_percent": 2
  }
}
```

Allowed keys are `rings`, `value_column`, `show_legend`, `show_total`, `show_labels`, `show_percentages`, `decimal_places`, `min_slice_percent`, `colors`, `slice_labels`, and `slice_hidden`.

`rings` accepts up to three preview column keys. `show_percentages` is `off`, `legend`, `chart`, or `both`.

Use `slice_labels` to polish raw enum/category values and `slice_hidden` to suppress noise. Use `colors` only when a stable business color exists.

## Heatmap

Heatmap is for numeric correlation/intensity columns, not geographic maps.

```json
{
  "heatmap": {
    "columns": ["sales_total_ypb", "collection_total_ypb", "overdue_total_ypb"],
    "show_values": true,
    "decimal_places": 2
  }
}
```

Allowed keys are `columns`, `show_values`, and `decimal_places`. `columns` accepts up to 32 numeric preview column keys.

## Export Fit

Pie and heatmap render as charts. Use export `format:"svg"`, `png`, or `pdf`, with matching `render_mode`.

## UX Rules

- Do not use pie with more than roughly seven visible slices.
- Do not use pie for negative values.
- Use `_columns.<value>.format` to control money/percentage display.
- For heatmap, ensure selected columns share a comparable numeric meaning.
