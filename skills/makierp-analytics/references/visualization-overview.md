# Visualization Overview

Use this before saving `selected_viz`, dashboard card `viz_type`, export `render_mode`, or `visualization` payloads.

Supported visualization types are:

- `table`
- `pivot`
- `number`
- `pie`
- `bar`
- `line`
- `area`
- `combo`
- `scatter`
- `heatmap`

Dashboard cards with `question_id` must use one of those types. Dashboard cards without `question_id` may only use `text` or `heading`.

## Required Sequence

1. Validate and preview the query.
2. Copy exact preview `columns[].key` values.
3. Choose a visualization type from the data shape.
4. Build `visualization._columns` for labels and formatting.
5. Add the matching config key: `visualization.table`, `visualization.number`, `visualization.bar`, etc.
6. Save `selected_viz` on questions, `viz_type` on dashboard cards, and `render_mode` on exports.

If `selected_viz:"bar"` is sent, `visualization.bar` must exist. If `render_mode:"pivot"` is sent, `visualization.pivot` must exist.

## Choosing A Type

- `number`: one primary metric, optionally one comparison/delta.
- `table`: row lists, audit views, drill-down output, or mixed column types.
- `pivot`: cross-tab summaries with row fields, column fields, and value fields.
- `bar`: category comparison, top-N, grouped or stacked totals.
- `line`: trend over an axis ordered in time, and nothing else. A line between
  unordered categories draws a slope that says the value travelled from one to
  the other, which never happened.
- `area`: a line whose filled region is itself a quantity — width times height
  has to be a real total. Shade turnover, not a ratio.
- `combo`: two or more metrics where bar+line or dual axes clarify scale. The
  line rule applies to each `line` series in it.
- `scatter`: correlation/outlier analysis with numeric x and y, optional size.
  Both axes are measured, so neither may be a category or a date.
- `pie`: part-to-whole with a small number of categories.
- `heatmap`: numeric correlation matrix or compact pairwise intensity view.
- `boxplot`: the spread of one numeric column, optionally one box per category.
  Needs UNAGGREGATED rows — it derives its own quartiles.
- `map`: one point per row from a latitude and a longitude column. Needs rows at
  the granularity of the coordinate — a grouped query averages two addresses into
  a point between them. Rows without a fix are dropped, so the point count is not
  the row count.

## Shared `_columns`

Use `_columns` as an object keyed by preview column key. Every visible computed column should have a Turkish label.

```json
{
  "_columns": {
    "month": {
      "label": "Ay",
      "format": {"type":"date","preset":"month_year"}
    },
    "net_total_ypb": {
      "label": "Net Tutar (YPB)",
      "alignment": "right",
      "format": {"type":"currency","currency":"TRY","decimals":2,"separator_style":"1.000,00"}
    }
  }
}
```

Format types are `number`, `currency`, `percentage`, `date`, `boolean`, and `text`. For percentage values stored as ratios, set `multiply:true`. For text columns, use `max_width`, `truncate`, and `link_detection` when needed.

## UX Rules

- Prefer `table` until the preview proves a chart shape.
- Do not chart ids, UUIDs, raw enum values, or untranslated aliases.
- Use YPB/IPB/RPB in labels when a money column has a currency basis.
- Hide helper/sort columns in tables, but keep them in DSL when needed for ordering.
- Avoid pie charts for long-tail categories; use bar with `limit` instead.
- Pick the mark from what the axis is, not from what looks livelier: a line
  needs time, an area needs an accumulating quantity, a scatter needs two
  measures. For an unordered axis, `"type":"plot"` draws the same points as
  unconnected markers.
- Saving a chart that breaks one of those returns a warning
  (`viz.line_requires_temporal_x`, `viz.area_product_not_meaningful`,
  `viz.scatter_requires_numeric_axes`) and still saves. Read it and fix the
  chart rather than shipping a picture that claims something the data does not.
- For dashboards, use `viz_overrides` only when a card should differ from the saved question's default visualization.
