# Manifold DSL Reference

Use this before composing non-trivial Manifold DSL. The source schema is still the authority: call `analytics_source_describe`, use exact field paths from the response, then validate with `analytics_query_validate`.

## Survival Path

When you are dropped into an empty session and need a working query:

1. Call `analytics_sources_list` and choose a source by domain, not by guessed table name.
2. Call `analytics_source_describe` with `data_source_type` and `data_source_id`. Copy field paths, relation scopes, enum values, params, and starter DSL from that response.
3. Write the smallest raw DSL that can validate. For a basic metric, use only `select`.
4. Validate with:

```json
{
  "data_source_type": "schema",
  "data_source_id": "customers",
  "query": {
    "select": [
      {"fn":"count","args":[],"as":"customer_count","label":"Cari Sayısı"}
    ]
  }
}
```

5. Preview with the same shape using `analytics_query_preview` and optional `page`, `per_page`, and runtime `params`.
6. If `analytics_query_preview` is not in the tool list, say the DSL validates but rows cannot be confirmed in this session. Do not invent result values.

Use `data_source_type:"schema"` for schema-backed sources, `report` for report-backed sources, and `question` for saved-question sources. Use the exact `data_source_id` surfaced by source listing/describe.

## What Goes In `query`?

The `query` argument is either raw DSL or a saved-question envelope:

- Raw DSL for normal schema/report/question previews: `{"select":[...],"where":{...}}`
- Envelope when creating/updating a saved question with params: `{"dsl":{...},"_params":[...],"_param_intercepts":[...],"paginate":true}`

Do not wrap every query in `dsl`. If validation says a root key is unknown or the result looks like the backend ignored `select`, check that you sent the shape the tool expects.

## First Queries To Try

Total row count:

```json
{
  "select": [
    {"fn":"count","args":[],"as":"row_count","label":"Satır Sayısı"}
  ]
}
```

Distinct entity count:

```json
{
  "select": [
    {"fn":"count_distinct","args":[{"var":{"path":"customer_id"}}],"as":"customer_count","label":"Cari Sayısı"}
  ]
}
```

Single filtered total:

```json
{
  "select": [
    {"fn":"sum","args":[{"var":{"path":"local_grand_total"}}],"as":"grand_total_ypb","label":"Genel Toplam (YPB)"}
  ],
  "where": {
    "between": [{"var":{"path":"document_date"}},{"param":"date_from"},{"param":"date_to"}]
  }
}
```

Top-N grouped total:

```json
{
  "select": [
    {"var":{"path":"customer_name"},"as":"customer_name","label":"Cari Adı"},
    {"fn":"sum","args":[{"var":{"path":"local_net_total"}}],"as":"net_total_ypb","label":"Net Tutar (YPB)"}
  ],
  "group_by": [
    {"var":{"path":"customer_name"}}
  ],
  "order_by": [
    [{"var":{"path":"net_total_ypb"}},"desc"]
  ],
  "limit": 10
}
```

Before using any example, replace field paths with paths from `analytics_source_describe`.

## Quick Rules

- Payloads are source-relative. Do not send a root `from`; the MCP data source id chooses the source.
- Build the smallest valid query first: `select`, then `where`, then `group_by`/aggregates, then `having`, then `order_by`.
- `select`, `group_by`, `order_by`, `include`, and `union_all` are arrays. `where` and `having` are operator objects.
- Every node is structured. Use `{"var":{"path":"code"}}`, not `{"var":"code"}`.
- Computed select nodes should have stable snake_case `as` aliases and Turkish `label` values.
- Enum fields (`"type":"enum"` in `analytics_source_describe`) are projected, grouped, and compared by their case **label**; `cases[].value` is the label. Filter with it: `{"==":[{"var":{"path":"direction"}},{"literal":"Satış","type":"string"}]}`. A stored value (`"sale"`) is rewritten to its label, so older queries keep working, but write labels.
- Validate before preview/save. If no preview/execute tool is available, validation proves DSL shape and semantics only.
- If `analytics_query_validate` returns diagnostics, fix the first structural diagnostic before changing business logic. Later diagnostics are often fallout.

## Query Skeleton

```json
{
  "select": [],
  "where": {},
  "group_by": [],
  "having": {},
  "order_by": [],
  "limit": 50,
  "offset": 0,
  "include": [],
  "union_all": []
}
```

Allowed top-level keys are `include`, `select`, `where`, `group_by`, `having`, `order_by`, `limit`, `offset`, and `union_all`.

## Raw DSL Or Saved-Question Envelope?

For schema-backed sources, tools usually accept the raw DSL directly:

```json
{
  "select": [
    {"fn":"count","args":[],"as":"row_count","label":"Satır Sayısı"}
  ]
}
```

For saved questions, or when declaring `_params`, use the envelope shape:

```json
{
  "dsl": {
    "select": [
      {"fn":"count","args":[],"as":"row_count","label":"Satır Sayısı"}
    ]
  },
  "_params": [],
  "_param_intercepts": [],
  "paginate": true
}
```

Do not put `dsl` inside a raw DSL payload. Do not put clause keys like `select` beside `dsl` unless the specific MCP tool says it accepts that compatibility shape.

MCP validation/preview tool call shape:

```json
{
  "data_source_type": "schema",
  "data_source_id": "invoices",
  "query": {
    "select": [
      {"fn":"count","args":[],"as":"invoice_count","label":"Fatura Sayısı"}
    ]
  },
  "params": {}
}
```

Saved-question source preview with runtime params:

```json
{
  "data_source_type": "question",
  "data_source_id": "42",
  "query": {
    "select": [
      {"var":{"path":"customer_name"},"as":"customer_name","label":"Cari Adı"}
    ]
  },
  "params": {
    "date_from": "2026-01-01",
    "date_to": "2026-01-31"
  }
}
```

## Node Shapes

Column on the current source:

```json
{"var":{"path":"customer_id"}}
```

Column from an included alias:

```json
{"var":{"scope":"customer","path":"name"}}
```

Adapter namespace hint when source describe shows it, usually for relation-scoped projections:

```json
{"var":{"ns":"relation","scope":"lineModel","path":"code"}}
```

Literal:

```json
{"literal":10,"type":"number"}
```

Runtime parameter for filter RHS, function args, or case values:

```json
{"param":"date_from"}
```

Function:

```json
{"fn":"sum","args":[{"var":{"path":"local_net_total"}}],"as":"net_total_ypb","label":"Net Tutar (YPB)"}
```

Case expression:

```json
{
  "case": [
    {
      "when": {">=": [{"var":{"path":"local_net_total"}},{"literal":100000,"type":"number"}]},
      "then": {"literal":"yüksek","type":"string"}
    }
  ],
  "else": {"literal":"normal","type":"string"},
  "as": "amount_band",
  "label": "Tutar Bandı"
}
```

## How Do I Count?

Row count:

```json
{
  "select": [
    {"fn":"count","args":[],"as":"customer_count","label":"Cari Sayısı"}
  ]
}
```

Count non-null values in one column:

```json
{
  "select": [
    {"fn":"count","args":[{"var":{"path":"tax_number"}}],"as":"tax_number_count","label":"Vergi No Sayısı"}
  ]
}
```

Count distinct values:

```json
{
  "select": [
    {"fn":"count_distinct","args":[{"var":{"path":"customer_id"}}],"as":"customer_count","label":"Cari Sayısı"}
  ]
}
```

Grouped count:

```json
{
  "select": [
    {"var":{"path":"city"},"as":"city","label":"Şehir"},
    {"fn":"count","args":[],"as":"customer_count","label":"Cari Sayısı"}
  ],
  "group_by": [
    {"var":{"path":"city"}}
  ],
  "order_by": [
    [{"var":{"path":"customer_count"}},"desc"]
  ]
}
```

## How Do I Sum By A Dimension?

If a query selects both aggregates and plain columns, every plain selected column must appear in `group_by`.

```json
{
  "select": [
    {"var":{"path":"customer_code"},"as":"customer_code","label":"Cari Kodu"},
    {"var":{"path":"customer_name"},"as":"customer_name","label":"Cari Adı"},
    {"fn":"sum","args":[{"var":{"path":"local_net_total"}}],"as":"net_total_ypb","label":"Net Tutar (YPB)"}
  ],
  "group_by": [
    {"var":{"path":"customer_code"}},
    {"var":{"path":"customer_name"}}
  ],
  "order_by": [
    [{"var":{"path":"net_total_ypb"}},"desc"]
  ],
  "limit": 20
}
```

Use aggregate functions in `select`, `having`, or `order_by`. Use `where` for row-level filters before grouping. Use `having` for aggregate filters after grouping.

## How Do Aliases Interact With Clauses?

- `select` aliases become output column keys.
- Computed select aliases may be referenced in `having` and `order_by`.
- Computed select aliases may not be referenced in `where` or `group_by`; repeat the source expression there.
- Do not alias a computed select to the same name as an existing source field.
- Do not reuse the same alias twice.

Valid aggregate alias in `having` and `order_by`:

```json
{
  "select": [
    {"var":{"path":"customer_id"},"as":"customer_id","label":"Cari"},
    {"fn":"sum","args":[{"var":{"path":"local_net_total"}}],"as":"net_total_ypb","label":"Net Tutar (YPB)"}
  ],
  "group_by": [
    {"var":{"path":"customer_id"}}
  ],
  "having": {
    ">": [{"var":{"path":"net_total_ypb"}},{"literal":10000,"type":"number"}]
  },
  "order_by": [
    [{"var":{"path":"net_total_ypb"}},"desc"]
  ]
}
```

Invalid pattern: `where` on `net_total_ypb` or `group_by` on `net_total_ypb`.

## How Do I Bucket By Date?

Repeat the same `date_trunc` expression in both `select` and `group_by`.

```json
{
  "select": [
    {"fn":"date_trunc","args":[{"literal":"month","type":"string"},{"var":{"path":"document_date"}}],"as":"month","label":"Ay"},
    {"fn":"sum","args":[{"var":{"path":"local_grand_total"}}],"as":"grand_total_ypb","label":"Genel Toplam (YPB)"}
  ],
  "group_by": [
    {"fn":"date_trunc","args":[{"literal":"month","type":"string"},{"var":{"path":"document_date"}}]}
  ],
  "order_by": [
    [{"var":{"path":"month"}},"asc"]
  ]
}
```

Allowed `date_trunc`, `date_add`, `date_sub`, and `date_diff` units are `minute`, `hour`, `day`, `week`, `month`, `quarter`, and `year`.

## How Do I Filter Rows?

`where` and `having` nodes must contain exactly one operator key. Comparison operators use operand arrays.

```json
{
  "where": {
    "and": [
      {">=": [{"var":{"path":"document_date"}},{"param":"date_from"}]},
      {"<=": [{"var":{"path":"document_date"}},{"param":"date_to"}]},
      {"in": [{"var":{"path":"status"}},[{"literal":"approved","type":"string"},{"literal":"closed","type":"string"}]]}
    ]
  }
}
```

Allowed filter operators are `and`, `or`, `if`, `exists`, `!`, `!!`, `==`, `!=`, `>`, `>=`, `<`, `<=`, `in`, `contains`, `starts_with`, `ends_with`, `fuzzy`, and `between`.

Common filters:

```json
{"contains":[{"var":{"path":"customer_name"}},{"literal":"anonim","type":"string"}]}
```

```json
{"between":[{"var":{"path":"document_date"}},{"param":"date_from"},{"param":"date_to"}]}
```

```json
{"!!":{"var":{"path":"cancelled_at"}}}
```

```json
{"!":{"var":{"path":"cancelled_at"}}}
```

For `in`, the RHS is a list of node objects or a single param reference. Do not wrap an array inside one `literal`.

```json
{"in":[{"var":{"path":"status"}},[{"literal":"open","type":"string"},{"literal":"closed","type":"string"}]]}
```

Conditional filter node:

```json
{
  "if": [
    {"==":[{"literal":true,"type":"boolean"},{"param":"only_open"}]},
    {"==":[{"var":{"path":"status"}},{"literal":"open","type":"string"}]},
    true
  ]
}
```

Use `if` sparingly. Most optional filters are better represented by params/defaults handled by the analytics source tooling.

## How Do I Filter Groups?

`having` is only valid with `group_by`. Use it for aggregate aliases or grouped expressions.

```json
{
  "select": [
    {"var":{"path":"customer_id"},"as":"customer_id","label":"Cari"},
    {"fn":"count","args":[],"as":"transaction_count","label":"İşlem Sayısı"}
  ],
  "group_by": [
    {"var":{"path":"customer_id"}}
  ],
  "having": {
    ">": [{"var":{"path":"transaction_count"}},{"literal":10,"type":"number"}]
  }
}
```

## How Do I Join Related Data?

Use `include` when `analytics_source_describe` exposes a relation/scope. Include aliases are required. Do not use include `from` or `mode`; use `match` when parent survival matters.

```json
{
  "include": [
    {
      "scope": "customer",
      "as": "customer",
      "match": "optional"
    }
  ],
  "select": [
    {"var":{"scope":"customer","path":"code"},"as":"customer_code","label":"Cari Kodu"},
    {"fn":"sum","args":[{"var":{"path":"local_grand_total"}}],"as":"grand_total_ypb","label":"Genel Toplam (YPB)"}
  ],
  "group_by": [
    {"var":{"scope":"customer","path":"code"}}
  ]
}
```

`match:"optional"` keeps parent rows even when no related row exists. `match:"required"` behaves like an existence requirement for that include.

Nested include:

```json
{
  "include": [
    {
      "scope": "customer",
      "as": "customer",
      "include": [
        {"scope":"balance","as":"balance","match":"required"}
      ]
    }
  ],
  "select": [
    {"var":{"scope":"customer","path":"name"},"as":"customer_name","label":"Cari Adı"},
    {"var":{"scope":"balance","path":"local_currency_balance"},"as":"balance_ypb","label":"Bakiye (YPB)"}
  ]
}
```

For nested scopes, nest include frames. Do not send `scope` as an array.

## How Do I Sort And Page?

`order_by` is an array of tuples. Direction is optional, but use explicit `asc` or `desc`.

```json
{
  "select": [
    {"var":{"path":"document_date"},"as":"document_date","label":"Tarih"},
    {"var":{"path":"slip_no"},"as":"slip_no","label":"Fiş No"}
  ],
  "order_by": [
    [{"var":{"path":"document_date"}},"desc"],
    [{"var":{"path":"slip_no"}},"asc"]
  ],
  "limit": 50,
  "offset": 0
}
```

For aggregate results, sort by the aggregate alias after selecting it:

```json
"order_by": [[{"var":{"path":"net_total_ypb"}},"desc"]]
```

`limit` and `offset` must be non-negative integers.

## How Do I Test Whether Related Rows Exist?

Use `exists` in `where` when you need existence without projecting the related scope.

```json
{
  "where": {
    "exists": {
      "scope": "invoices",
      "where": {
        "and": [
          {"==": [{"var":{"path":"status"}},{"literal":"open","type":"string"}]}
        ]
      }
    }
  },
  "select": [
    {"var":{"path":"code"},"as":"customer_code","label":"Cari Kodu"}
  ]
}
```

Inside the `exists.where`, bare `path` values refer to the scoped related row.

## How Do I Rank Rows Or Calculate Running Values?

Window functions need `over`. Pure window functions and window value functions are allowed in `select` and `order_by`.

```json
{
  "select": [
    {"var":{"path":"customer_id"},"as":"customer_id","label":"Cari"},
    {"var":{"path":"document_date"},"as":"document_date","label":"Tarih"},
    {"fn":"row_number","args":[],"over":{"partition_by":[{"var":{"path":"customer_id"}}],"order_by":[[{"var":{"path":"document_date"}},"desc"]]},"as":"row_no","label":"Sıra"}
  ],
  "order_by": [
    [{"var":{"path":"customer_id"}},"asc"],
    [{"var":{"path":"document_date"}},"desc"]
  ]
}
```

Window aggregate example:

```json
{
  "select": [
    {"var":{"path":"customer_id"},"as":"customer_id","label":"Cari"},
    {"var":{"path":"document_date"},"as":"document_date","label":"Tarih"},
    {"fn":"sum","args":[{"var":{"path":"local_net_total"}}],"over":{"partition_by":[{"var":{"path":"customer_id"}}],"order_by":[[{"var":{"path":"document_date"}},"asc"]]},"as":"running_total_ypb","label":"Kümülatif Net Tutar (YPB)"}
  ]
}
```

Do not mix grouped aggregate mode and window mode casually. If the query has `group_by`, plain columns inside window `args`, `partition_by`, or `order_by` can still trigger grouping diagnostics unless they are grouped or aggregated.

## How Do I Build A Computed Metric?

Use nested functions. Guard division with `nullif` when the denominator can be zero.

```json
{
  "select": [
    {
      "fn": "mul",
      "args": [
        {
          "fn": "div",
          "args": [
            {"fn":"sum","args":[{"var":{"path":"local_profit"}}]},
            {"fn":"nullif","args":[{"fn":"sum","args":[{"var":{"path":"local_revenue"}}]},{"literal":0,"type":"number"}]}
          ]
        },
        {"literal":100,"type":"number"}
      ],
      "as": "profit_margin_percent",
      "label": "Kar Marjı (%)"
    }
  ]
}
```

## How Do I Use Params?

Saved-question params live beside the DSL in `_params`. There are three approved
declaration forms:

- ordinary/free params omit `kind`, feed explicit `{param}` nodes or source
  inputs, and may own an optional `query_fragment` effect;
- column params use `kind:"column"` with `column_filter`;
- relation picker params use `kind:"relation"` with `relation_exists`.

Free param types are `text`, `number`, `date`, `datetime`, `boolean`, and
`select`. Existing `string` is read as legacy text; new definitions use `text`.
Free params may use `label`, `required`, `default`, `multiple`, `options`,
`options_source`, and `bind_to_param` metadata when the UI/source supports them.

```json
{
  "dsl": {
    "where": {
      "between": [{"var":{"path":"document_date"}},{"param":"date_from"},{"param":"date_to"}]
    },
    "select": [
      {"fn":"count","args":[],"as":"transaction_count","label":"İşlem Sayısı"}
    ]
  },
  "_params": [
    {"name":"date_from","type":"date","label":"Başlangıç Tarihi","required":true},
    {"name":"date_to","type":"date","label":"Bitiş Tarihi","required":true}
  ]
}
```

The free-param predicate above remains part of the saved DSL. Marking the
declaration optional does not remove that `between` expression when its value is
missing.

Optional column filter param:

```json
{
  "dsl": {
    "select": [
      {"var":{"path":"code"},"as":"customer_code","label":"Cari Kodu"},
      {"var":{"path":"name"},"as":"customer_name","label":"Cari Adı"}
    ]
  },
  "_params": [
    {
      "name": "customer_name",
      "type": "text",
      "widget": "search",
      "kind": "column",
      "column": "name",
      "required": false,
      "effect": {
        "type": "column_filter",
        "operator": "contains",
        "target": "where"
      }
    }
  ]
}
```

Relation picker param:

```json
{
  "dsl": {
    "select": [
      {"fn":"count","args":[],"as":"transaction_count","label":"İşlem Sayısı"}
    ]
  },
  "_params": [
    {
      "name": "customer",
      "kind": "relation",
      "relation": "customer",
      "label": "Cari",
      "multiple": true,
      "effect": {
        "type": "relation_exists",
        "target": "where",
        "value_column": "id",
        "label_column": "name",
        "sub_label_column": "code"
      }
    }
  ]
}
```

Column and relation params append one typed hidden predicate to an execution
copy at runtime. Both approved effects target pre-aggregation `where` and are
AND-composed with the saved base filter. An optional blank value appends no
predicate; a required blank value fails before execution. Do not hand-write a
duplicate condition for the same effect.

An ordinary parameter may own `effect: {"type":"query_fragment","query":{...}}`.
The partial query uses the same DSL, source, symbols, functions, and typed input
bindings as its declaring Question. Its owner controls activation; an optional
blank owner skips the entire body. Explicit ParamNodes in the base query remain
unconditional. Do not also specify a Column/Relation `kind`.

```json
{
  "name": "search", "type": "text", "required": false,
  "effect": {
    "type": "query_fragment",
    "query": {
      "where": {"or": [
        {"contains": [{"var": {"path": "code"}}, {"param": "search"}]},
        {"contains": [{"var": {"path": "name"}}, {"param": "search"}]}
      ]}
    }
  }
}
```

Supported partial clauses: `where`, `having`, `include`, `select`, `group_by`,
`order_by`, `union_all`, `limit`, `offset`. Filters are whole-group AND additions.
Includes use explicit aliases: identical frames are reused; different frames
with the same alias conflict. Select appends to an explicit base projection;
variable items retain field identity (no `as`), computed items need `as`.
Identical outputs/group expressions/sorts are reused. Conflicting output aliases,
opposite sort directions, or different existing limits/offsets fail. Union branches
append without deduplication. No source changes, stages, raw SQL, recursive input
definitions, pagination, or visualization settings are allowed inside a fragment.

All bodies are symbolically validated on save, including inactive bodies. Each
must work with the base independently: combine dependent optional clauses into
one fragment. Active combinations validate again before execution. Another input
referenced by an active body must be available. Errors identify the owning
`_params` entry and body path.

Query-backed choices are separate queries; dashboard controls only feed saved
Question inputs. Nested pin/map/rename routing applies fragments exactly once in
their declaring frame. Execution `query_state` describes active/skipped fragments
and `output_parameter_dependent`; actual execution columns are authoritative.
Public CSV shares of fragment-bearing Question graphs return an explicit
unsupported response; they never omit a fragment and return unfiltered data.

When replacing a stored fragment-bearing Question query, send `_params_version: 2`
beside `dsl` and `_params`, including when intentionally removing the fragment.
Unversioned replacement is rejected to prevent old editors from dropping behavior.
Metadata-only updates need no marker.

Runtime input uses three presence states:

- omit the key to allow the declaration default;
- send a nonblank value to override the default (`0` and `false` are values);
- send native JSON `null` to explicitly clear the default. An optional effect
  then becomes inactive and a required param remains missing.

Do not assume that a public-share GET URL can express the native-null clear
state. Its parameter and encoding policy is explicitly unresolved.

Question-source inherited params:

```json
{
  "dsl": {
    "select": [
      {"var":{"path":"customer_name"},"as":"customer_name","label":"Cari Adı"}
    ]
  },
  "_param_intercepts": [
    {"param":"source_date_to","kind":"rename","rename_to":"date_to"},
    {"param":"source_status","kind":"pin","value":"approved"},
    {"param":"source_date_from","kind":"map","map_from":"date_from"}
  ],
  "_params": [
    {"name":"date_from","type":"date","label":"Başlangıç Tarihi","required":true}
  ]
}
```

The inherited surface is computed from the current source chain whenever the
question is described or executed; never copy child declarations into the
outer `_params` list. Use `pin` to hide an inherited field and supply its fixed
inner value, `rename` to expose it under another outer name while retaining its
inner name, and `map` to pipe one compatible own param into an inherited target.
Each inherited target may have at most one intercept.

For report-backed or question-backed sources, pass runtime `params` to MCP
using only names on the computed outer surface. Pinned targets are hidden and
cannot be overridden; renamed fields use the outer alias; mapped targets receive
the selected own param value. Runtime values are not persisted.

## How Do I Combine Branches?

Use `union_all` when you need rows from multiple compatible branches. Each branch must have `select`, may have `include`, `where`, `group_by`, `having`, and `order_by`, and may not define nested `union_all`.

```json
{
  "select": [
    {"literal":"sales","type":"string","as":"source","label":"Kaynak"},
    {"fn":"sum","args":[{"var":{"path":"local_grand_total"}}],"as":"total_ypb","label":"Toplam (YPB)"}
  ],
  "union_all": [
    {
      "select": [
        {"literal":"returns","type":"string","as":"source","label":"Kaynak"},
        {"fn":"sum","args":[{"var":{"path":"local_grand_total"}}],"as":"total_ypb","label":"Toplam (YPB)"}
      ]
    }
  ]
}
```

Union branches must have compatible output aliases and types.

## How Do I Avoid Common Validation Failures?

- `dsl.removed_root_from`: remove root `from`; source is selected by MCP data source id.
- `dsl.invalid_select`: `select` must be an array of operand objects.
- `dsl.invalid_where_node`: `where`/`having` must be a non-empty object with exactly one operator.
- `dsl.unknown_attribute`: call `analytics_source_describe` again and use exact field paths/scopes.
- `dsl.unknown_scope`: include scope does not exist for this source; use described relation names.
- `dsl.unknown_function`: use `function-reference.md`; do not invent function names.
- `dsl.invalid_function_arity`: check `args`; row count is `{"fn":"count","args":[]}`.
- `dsl.invalid_function_column_type`: use a field of the required type or cast when appropriate.
- `dsl.ungrouped_select_column`: add the plain selected column to `group_by` or remove it.
- `dsl.ungrouped_select_expression_column`: a computed/window expression references a plain column that must be grouped or aggregated.
- `dsl.having_without_group_by`: add `group_by` or move the condition to `where`.
- `dsl.alias_not_allowed_in_context`: use computed aliases only in `having`/`order_by`, not `where`/`group_by`.
- `dsl.select_alias_conflict`: do not alias computed output to an existing source field.
- `dsl.duplicate_select_alias`: every explicit alias must be unique.
- `dsl.invalid_order_by_item`: use `[[operand,"asc"]]` or `[[operand,"desc"]]`.
- `dsl.removed_include_mode`: remove `mode`; use `match:"optional"` or `match:"required"`.
- `dsl.legacy_scope_path`: replace include scope arrays with nested include frames.

If validation is clean but preview is unavailable, say exactly that: the DSL validates, but the current tool list does not expose execution/preview, so returned rows cannot be confirmed.
