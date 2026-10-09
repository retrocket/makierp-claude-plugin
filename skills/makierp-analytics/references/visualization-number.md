# Number Visualization

Use this for KPI cards, headline metrics, dashboard summary cards, and one-value export renderings.

## Data Shape

The query should return one row with a primary metric column. Optional comparison columns can support subtitle/delta context, but avoid crowded KPI cards.

Good examples:

- Total sales this month.
- Open invoice count.
- Cash balance in YPB.
- Collection rate percentage.

## Payload

Use `selected_viz:"number"` or dashboard card `viz_type:"number"` with:

```json
{
  "_columns": {
    "grand_total_ypb": {
      "label": "Genel Toplam (YPB)",
      "format": {"type":"currency","currency":"TRY","decimals":2,"separator_style":"1.000,00"}
    },
    "month_label": {
      "label": "Dönem",
      "format": {"type":"text"}
    }
  },
  "number": {
    "value_column": "grand_total_ypb",
    "label": "Bu Ay Satış",
    "subtitle_column": "month_label",
    "max_comparisons": 0,
    "delta_mode": "percentage",
    "direction": "higher_is_better"
  }
}
```

Allowed keys are `value_column`, `label`, `subtitle_column`, `max_comparisons`, `delta_mode`, and `direction`.

`delta_mode` is `percentage` or `difference`. `direction` is `higher_is_better`, `lower_is_better`, or `neutral`.

## UX Rules

- Use short labels; the number itself is the focus.
- Format money and percentages through `_columns`.
- Use `direction:"lower_is_better"` for debt, overdue days, defects, or waiting time.
- Use `direction:"neutral"` for balances where positive/negative is contextual.
- Do not use `number` when the user needs a ranked list or breakdown; use `bar` or `table`.
