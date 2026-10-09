# Visualization: Box Plot

A box plot answers a question the other charts cannot: not "how much", but "how
spread out". The box covers the middle half of the values, the line across it is
the median, the whiskers reach the furthest observation still close to the box,
and anything past that is drawn as an individual point.

Reach for it when the shape of the values is the finding — payment terms that
average 45 days but run from 15 to 180, delivery times that look fine on average
because two customers are dragging the mean, discount rates that cluster in two
places rather than one. A bar of averages hides all of that.

## The query must not be aggregated

This is the one thing that catches people. Every other chart is given a summary
to draw; a box plot is given the raw rows and derives the summary itself, because
the DSL has no percentile function — you cannot `group_by` your way to a
quartile. **One row per observation.** A query that already averages by customer
gives you one point per customer and a meaningless box.

```json
{
  "selected_viz": "boxplot",
  "visualization": {
    "_columns": {
      "temsilci": {"label": "Temsilci"},
      "vade_gun": {"label": "Vade (gün)"}
    },
    "boxplot": {
      "value_column": "vade_gun",
      "category_column": "temsilci",
      "orientation": "vertical",
      "show_outliers": true,
      "show_mean": false
    }
  }
}
```

## Keys

- `value_column` (required): the numeric column whose distribution is described.
- `category_column` (optional): draws one box per distinct value. Omit it for a
  single box over everything. Capped at 40 boxes — past that nothing is readable.
- `orientation`: `vertical` (default) or `horizontal`. Go horizontal when the
  category labels are long.
- `show_outliers` (default true): points beyond 1.5 IQR of the box, drawn
  individually. Turning them off does not fold them into the whiskers; it hides
  them.
- `show_mean` (default false): marks each box's mean with a diamond. The
  five-number summary does not include it, and mean-versus-median is often the
  whole point in skewed data.
- `axes.x.title`, `axes.y_left.title`: axis titles.

## How the summary is computed

Quartiles by linear interpolation; whiskers at the furthest real observation
within 1.5 IQR of the box, never at the fence itself, so a box never claims a
value the sample does not contain. The frontend and the PDF renderer compute this
identically — if they ever disagree, that is a bug, not a rounding difference.

## Not a box plot

- Comparing one number across categories → `bar`.
- A trend over time → `line`.
- The relationship between two measures → `scatter`.
- How two measures move together across many pairs → `heatmap`.
