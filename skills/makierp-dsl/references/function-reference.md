# Manifold DSL Function Reference

This reference gives stable usage rules for Manifold DSL function nodes. Exact function availability can change with the backend, so validate every query with `analytics_query_validate` before previewing or saving.

## Function Node Rules

- Shape is `{"fn":"name","args":[...]}`.
- `args` is always a list, even when empty.
- In `select`, set stable `as` aliases and Turkish `label` values for computed outputs.
- Scalar functions can be used in `select`, `where`, `group_by`, `having`, `order_by`, function args, and case values unless validation says otherwise.
- Aggregate functions without `over` trigger grouped aggregate semantics.
- Aggregate functions with `over` behave as window aggregates.
- Pure window functions require `over` and are allowed only in `select` and `order_by`.
- Nested functions are allowed, but their argument types are still validated.

## Function Index

| Function | Args | Returns | Use for |
|----------|------|---------|---------|
| `sum` | one numeric expression | number | Totals. |
| `count` | zero or one expression | number | Row count with `args: []`, non-null count with one arg. |
| `count_distinct` | one expression | number | Unique value count. |
| `avg` | one numeric expression | number | Average. |
| `min` | one expression | same as input | Smallest value. |
| `max` | one expression | same as input | Largest value. |
| `corr` | two numeric expressions | number | Pearson correlation. |
| `add`, `sub`, `mul`, `div` | two or more numeric expressions | number | Arithmetic chains. |
| `mod`, `power` | two numeric expressions | number | Remainder and exponentiation. |
| `abs`, `ceil`, `floor`, `sqrt` | one numeric expression | number | Numeric cleanup. |
| `round` | numeric expression, integer literal precision | number | Decimal rounding. |
| `concat` | two or more expressions | string | Text composition. |
| `lower`, `upper`, `trim` | one string expression | string | Text normalization. |
| `length` | one string expression | number | Text length. |
| `substring` | string, start number, optional length number | string | Text slicing. |
| `replace` | string, search string, replacement string | string | Text replacement. |
| `date_trunc` | unit literal, date/datetime expression | same as date arg | Time buckets. |
| `extract`, `date_part` | part literal, date/datetime expression | number | Date part as number. |
| `format_date` | date/datetime expression, format literal | string | Date display key. |
| `date_add`, `date_sub` | unit literal, date/datetime expression, numeric amount | date/datetime | Date arithmetic. |
| `date_diff` | unit literal, start date/datetime, end date/datetime | number | Temporal distance. |
| `now` | no args | datetime/string runtime value | Current timestamp comparisons. |
| `cast` | expression, target type literal | target type | Type coercion. |
| `coalesce` | two or more expressions | same as first arg | First non-null fallback. |
| `ifnull` | expression, fallback expression | same as first arg | Two-arg null fallback. |
| `nullif` | two expressions | same as first arg | Turn matching value into null. |
| `greatest`, `least` | two or more expressions | same as first arg | Clamp or compare values. |
| `row_number`, `rank`, `dense_rank` | no args, requires `over` | number | Window ranking. |
| `lag`, `lead` | expression, optional integer offset, optional default | same as first arg | Previous/next row values. |
| `first_value`, `last_value` | one expression | same as input | Window first/last value. |
| `nth_value` | expression, positive integer literal position | same as input | Window nth value. |

## Aggregates

Row count:

```json
{"fn":"count","args":[],"as":"row_count","label":"Satır Sayısı"}
```

Non-null count:

```json
{"fn":"count","args":[{"var":{"path":"tax_number"}}],"as":"tax_number_count","label":"Vergi No Sayısı"}
```

Distinct count:

```json
{"fn":"count_distinct","args":[{"var":{"path":"customer_id"}}],"as":"customer_count","label":"Cari Sayısı"}
```

Sum:

```json
{"fn":"sum","args":[{"var":{"path":"local_net_total"}}],"as":"net_total_ypb","label":"Net Tutar (YPB)"}
```

Correlation:

```json
{"fn":"corr","args":[{"var":{"path":"local_credit"}},{"var":{"path":"local_debit"}}],"as":"debit_credit_corr","label":"Borç/Alacak Korelasyonu"}
```

Aggregates are allowed in `select`, `having`, and `order_by`. Bare aggregates need grouped semantics when mixed with plain columns. `sum` and `avg` require numeric inputs; `min`, `max`, `count`, and `count_distinct` accept broader inputs.

## Arithmetic

Arithmetic functions take numeric expressions.

```json
{"fn":"sub","args":[{"var":{"path":"local_debit"}},{"var":{"path":"local_credit"}}],"as":"balance_ypb","label":"Bakiye (YPB)"}
```

Variadic arithmetic is left-to-right:

```json
{"fn":"add","args":[{"var":{"path":"local_net_total"}},{"var":{"path":"local_tax_total"}},{"var":{"path":"local_discount_total"}}],"as":"computed_total_ypb","label":"Hesaplanan Toplam (YPB)"}
```

Guard division by zero with `nullif`:

```json
{"fn":"div","args":[{"var":{"path":"local_profit"}},{"fn":"nullif","args":[{"var":{"path":"local_revenue"}},{"literal":0,"type":"number"}]}],"as":"profit_ratio","label":"Kar Oranı"}
```

Round needs an integer literal precision:

```json
{"fn":"round","args":[{"var":{"path":"local_net_total"}},{"literal":2,"type":"number"}],"as":"rounded_net_total_ypb","label":"Yuvarlanmış Net Tutar (YPB)"}
```

## Strings

Concatenate labels:

```json
{"fn":"concat","args":[{"var":{"path":"code"}},{"literal":" - ","type":"string"},{"var":{"path":"name"}}],"as":"customer_display","label":"Cari"}
```

Normalize for grouping/search keys:

```json
{"fn":"lower","args":[{"fn":"trim","args":[{"var":{"path":"city"}}]}],"as":"city_key","label":"Şehir Anahtarı"}
```

Substring with integer literals:

```json
{"fn":"substring","args":[{"var":{"path":"code"}},{"literal":1,"type":"number"},{"literal":3,"type":"number"}],"as":"code_prefix","label":"Kod Öneki"}
```

Replace text:

```json
{"fn":"replace","args":[{"var":{"path":"phone"}},{"literal":" ","type":"string"},{"literal":"","type":"string"}],"as":"phone_clean","label":"Telefon"}
```

String functions validate string columns when the argument is a direct `var`.

## Dates

Allowed date units for `date_trunc`, `date_add`, `date_sub`, and `date_diff`: `minute`, `hour`, `day`, `week`, `month`, `quarter`, `year`.

Allowed extract/date_part parts: `year`, `quarter`, `month`, `week`, `day`, `dow`, `doy`, `hour`, `minute`, `second`.

Bucket:

```json
{"fn":"date_trunc","args":[{"literal":"month","type":"string"},{"var":{"path":"document_date"}}],"as":"month","label":"Ay"}
```

Extract year:

```json
{"fn":"extract","args":[{"literal":"year","type":"string"},{"var":{"path":"document_date"}}],"as":"year","label":"Yıl"}
```

Format date:

```json
{"fn":"format_date","args":[{"var":{"path":"document_date"}},{"literal":"YYYY-MM","type":"string"}],"as":"year_month","label":"Yıl/Ay"}
```

Date range relative to now:

```json
{">=":[{"var":{"path":"document_date"}},{"fn":"date_sub","args":[{"literal":"day","type":"string"},{"fn":"now","args":[]},{"literal":30,"type":"number"}]}]}
```

Date difference:

```json
{"fn":"date_diff","args":[{"literal":"day","type":"string"},{"var":{"path":"due_date"}},{"var":{"path":"payment_date"}}],"as":"payment_delay_days","label":"Gecikme (Gün)"}
```

`format_date` accepts a restricted PostgreSQL-like token set: `YYYY`, `YYY`, `YY`, `Y`, `MONTH`, `MON`, `MM`, `DAY`, `DY`, `DD`, `D`, `HH24`, `HH12`, `HH`, `MI`, `SS`, `MS`, `US`, `AM`, `PM`, `TZ`, `TZH`, `TZM`, `Q`, `WW`, `IW`, plus separators.

## Nulls And Fallbacks

First non-null:

```json
{"fn":"coalesce","args":[{"var":{"path":"customer_name"}},{"literal":"(adsız)","type":"string"}],"as":"customer_name","label":"Cari Adı"}
```

Two-arg fallback:

```json
{"fn":"ifnull","args":[{"var":{"path":"local_net_total"}},{"literal":0,"type":"number"}],"as":"net_total_ypb","label":"Net Tutar (YPB)"}
```

Division guard:

```json
{"fn":"nullif","args":[{"var":{"path":"quantity"}},{"literal":0,"type":"number"}]}
```

Clamp-like helpers:

```json
{"fn":"greatest","args":[{"var":{"path":"local_balance"}},{"literal":0,"type":"number"}],"as":"positive_balance_ypb","label":"Pozitif Bakiye (YPB)"}
```

## Casts

Allowed cast target literals are `string`, `number`, `boolean`, `date`, and `datetime`.

```json
{"fn":"cast","args":[{"var":{"path":"document_no"}},{"literal":"string","type":"string"}],"as":"document_no_text","label":"Belge No"}
```

Cast only when validation or source shape requires it. Prefer native typed fields from `analytics_source_describe`.

## Windows

Window spec shape:

```json
{
  "partition_by": [{"var":{"path":"customer_id"}}],
  "order_by": [[{"var":{"path":"document_date"}},"asc"]]
}
```

Ranking:

```json
{"fn":"row_number","args":[],"over":{"partition_by":[{"var":{"path":"customer_id"}}],"order_by":[[{"var":{"path":"document_date"}},"desc"]]},"as":"row_no","label":"Sıra"}
```

Lag with offset and default:

```json
{"fn":"lag","args":[{"var":{"path":"local_net_total"}},{"literal":1,"type":"number"},{"literal":0,"type":"number"}],"over":{"partition_by":[{"var":{"path":"customer_id"}}],"order_by":[[{"var":{"path":"document_date"}},"asc"]]},"as":"previous_net_total_ypb","label":"Önceki Net Tutar (YPB)"}
```

First value:

```json
{"fn":"first_value","args":[{"var":{"path":"document_date"}}],"over":{"partition_by":[{"var":{"path":"customer_id"}}],"order_by":[[{"var":{"path":"document_date"}},"asc"]]},"as":"first_document_date","label":"İlk Belge Tarihi"}
```

Window functions are allowed in `select` and `order_by`. They require `over`; `row_number`, `rank`, and `dense_rank` take no args. `lag`/`lead` take 1-3 args; the offset must be an integer literal. `nth_value` position must be a positive integer literal.

## Function Diagnostics

- `dsl.unknown_function`: function name is not registered.
- `dsl.invalid_function_arity`: wrong number of args; check the index table.
- `dsl.invalid_function_args`: `args` must be a list.
- `dsl.invalid_function_arg`: argument shape or role is invalid.
- `dsl.invalid_function_literal`: literal value is the wrong kind, such as non-integer precision.
- `dsl.invalid_function_column_type`: source field type is wrong for the function.
- `dsl.invalid_function_interval`: `date_trunc` interval is not allowed.
- `dsl.invalid_function_date_unit`: `date_add`, `date_sub`, or `date_diff` unit is not allowed.
- `dsl.window_not_supported`: scalar function was given `over`.
- `dsl.window_required`: window-only function is missing `over`.
