# Cartesian Visualization

Use this for `bar`, `line`, `area`, `combo`, and `scatter` visualizations.

## Data Shape

- `x_column`: category/date bucket/numeric x-axis.
- `series`: one or more numeric value columns.
- `series_column`: optional split column for dynamic series.
- `size_column`: scatter bubble size.

Use `date_trunc` in DSL for time buckets and sort ascending for trend charts.

## Bar

Best for top-N and category comparisons.

```json
{
  "bar": {
    "x_column": "customer_name",
    "series": [
      {"column":"net_total_ypb","label":"Net Tutar (YPB)","axis":"y_left","type":"bar"}
    ],
    "orientation": "horizontal",
    "bar_mode": "grouped",
    "show_legend": false,
    "show_value_labels": true
  }
}
```

Use horizontal bars for long labels. Use `bar_mode:"stacked"` or `percent_stacked` only when segments are comparable parts of a whole.

## Line

A line says the gap between two points is a distance you can travel: the value
moved from here to there. That only holds when the x axis is ordered in time, so
only put a line on an axis that is a date bucket or something derived from one —
an hour, day, week, month, quarter or year. Sales representatives, customers,
products, warehouses and currencies have no such order; sorting them by value
does not create one, it just makes the slope a picture of the sort.

```json
{
  "line": {
    "x_column": "month",
    "series": [
      {"column":"net_total_ypb","label":"Net Tutar (YPB)","axis":"y_left","type":"line"}
    ],
    "curve": "monotone",
    "show_points": true,
    "show_legend": true,
    "legend_position": "top"
  }
}
```

A time axis is usually written as `format_date(date_trunc('month', document_date), 'MM.YYYY')`,
which returns text. That is still a time axis and is treated as one: the output
column is marked temporal and the marker travels with it, including through a
saved question used as another question's source. If you build a time label some
other way — `concat` of a year and a month, say — say so explicitly with
`"$": {"temporal": true}` on the select item.

Saving a line over an unordered axis returns the warning
`viz.line_requires_temporal_x`. It is advice, not a rejection: the question is
still created. To keep the categories, set `"type":"plot"` on the series and the
same points are drawn as unconnected markers (see **Plot series** below).

## Area

An area is a line plus a claim about the region under it: width times height is
a real quantity that accumulated. Daily turnover works — a day of sales times a
day of time is a month of sales. A rate does not: a percentage multiplied by
elapsed days is not an amount, and the shaded region means nothing. Everything
the Line section says about the x axis applies here too, since an area is a line.

A second y axis is fine, and left and right may carry different scales and
units. The rule is per series, not per chart: fill the series whose x times y is
itself a total, and draw the rest as plain lines on the other axis. Avoid
filling many overlapping series.

Filling a series whose declared format is a percentage — in `_columns` or on the
axis it is bound to — returns `viz.area_product_not_meaningful`.

## Combo

Use when two metrics need different marks or axes.

```json
{
  "combo": {
    "x_column": "month",
    "series": [
      {"column":"net_total_ypb","label":"Net Tutar (YPB)","type":"bar","axis":"y_left"},
      {"column":"profit_margin_percent","label":"Kar Marjı (%)","type":"line","axis":"y_right"}
    ],
    "axes": {
      "y_left": {"title":"Tutar (YPB)","format":{"type":"currency","currency":"TRY","decimals":0}},
      "y_right": {"title":"Kar Marjı","format":{"type":"percentage","decimals":1,"multiply":false}}
    },
    "show_legend": true
  }
}
```

## Scatter

Use for correlation or outliers. A scatter places a point by measuring both of
its coordinates, so `x_column` and every series column must be numeric. A
category or a date on the x axis has no distance to measure and the chart
renders empty; saving one returns `viz.scatter_requires_numeric_axes`. To
compare a measure across categories use `bar`; to keep a category axis with
unconnected markers use a `plot` series.

`size_column` is optional and must also be numeric; `size_range` is
`[min, max]` in pixels, smallest first.

```json
{
  "scatter": {
    "x_column": "sales_count",
    "series": [
      {"column":"net_total_ypb","label":"Net Tutar (YPB)"}
    ],
    "size_column": "customer_count",
    "size_range": [4, 24],
    "show_legend": true
  }
}
```

## Common Keys

Allowed cartesian keys include `x_column`, `series_column`, `series`, `size_column`, `size_range`, `axes`, `orientation`, `bar_mode`, `stacking`, `curve`, `show_points`, `show_legend`, `legend_position`, `show_value_labels`, `goal_line`, `stack_values`, and `series_overrides`.

Each entry in `series` (and each value in `series_overrides`) accepts `column`, `type`, `axis`, `color`, `label`, `visible`, `showArea`, `symbol`, `symbolSize` and `showValueLabels`.

## Plot series

Set `"type":"plot"` on a series to draw unconnected markers rather than a line. Reach for it when the x axis is categorical — customers, products, sales reps — because a line between unrelated categories implies a trend that does not exist. It works in `bar`, `line`, `area` and `combo` charts and keeps the category axis, unlike the `scatter` viz type which requires two numeric axes.

```json
{
  "combo": {
    "x_column": "temsilci",
    "series": [
      {"column":"net_satis_ypb","label":"Net Satış (YPB)","type":"bar","axis":"y_left"},
      {"column":"tahsilat_yuzdesi","label":"Tahsilat (%)","type":"plot","axis":"y_right","symbol":"star","symbolSize":14,"showValueLabels":true}
    ]
  }
}
```

`symbol`: `circle` (default), `triangle`, `diamond`, `rect`, `roundRect`, `pin`, `arrow`, `star`. `symbolSize`: 4-40, default 8 — raise it when `showValueLabels` is on so the number clears the marker. `showValueLabels` overrides the chart-level `show_value_labels` for that one series; omit it to inherit.

Axis `format` supports number, currency, percentage, and date. Use `_columns` for column labels and default formatting, and axes for chart-specific titles/ranges.

Chart exports support `svg`, `png`, and `pdf`.
