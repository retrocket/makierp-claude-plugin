# Dashboard Designer

Use this for requests like:

- "Bu dashboard'a satış KPI'ı ekle"
- "Aylık tahsilat kartlarını tek sekmede topla"
- "Cari filtresini dashboard parametresi yap"

## Workflow

1. Inspect the existing dashboard, tabs, cards, params, and candidate saved questions with `analytics_dashboards_list` / `analytics_dashboards_show` and `analytics_questions_list` / `analytics_questions_show`.
2. If a needed saved question does not exist or is not polished, use `analytics-question-builder.md` first.
3. Treat dashboard changes as tenant mutations and inspect existing dashboards before writing.
4. Use saved-question preview column keys when choosing card visualization behavior.
5. Design with saved questions that have already been validated, previewed, and labeled.
6. Use `visualization-overview.md` and the type-specific visualization reference before setting `viz_type` or `viz_overrides`.
7. Read each card's current computed question surface, then bind question params to compatible dashboard params deliberately. Do not copy inherited definitions and do not promote unbound fields implicitly.
8. Update/create through `analytics_dashboards_create` / `analytics_dashboards_update`, then `analytics_dashboards_show` again to verify the persisted structure.

## Tool Navigation

- `analytics_collections_list`: find the `collection_id` that will own the dashboard.
- `analytics_dashboards_list`: find existing dashboard ids and avoid duplicates.
- `analytics_dashboards_show`: inspect existing tabs/cards and card `question_surface` values before updating, then verify after saving.
- `analytics_questions_list` / `analytics_questions_show`: choose stable `question_id` values and inspect current computed surfaces for cards.
- `analytics_query_preview`: preview the saved question when card labels, column keys, or chart suitability are unclear.

## Payload Shape

Dashboard create accepts `name`, `collection_id`, optional `description`, optional
`full_width`, optional `params`, and required `tabs`. On update, supplying `tabs`
replaces the tab/card tree; omitting it preserves the current tree. A params-only
update is still validated against the effective existing cards and bindings.

Use dashboard params like:

```json
{
  "_gridCols": 24,
  "fields": [
    {"name":"date_from","type":"date","label":"Başlangıç Tarihi"},
    {"name":"date_to","type":"date","label":"Bitiş Tarihi"},
    {
      "name":"customer_ids",
      "type":"select",
      "label":"Cariler",
      "multiple":true,
      "options_mode":"query",
      "options_source":{
        "type":"dsl",
        "source_type":"schema",
        "source_id":"customers",
        "value_column":"id",
        "label_column":"name"
      }
    }
  ]
}
```

Dashboard fields are ordinary typed runtime declarations. They may use manual,
query-backed, or arbitrary select choices, but must not contain `kind`,
`column`, `relation`, `effect`, or `bind_to_param`. A query-backed option source
only returns value/label/sublabel choices; it never becomes a card's analytical
query and never replaces the saved question.

Each tab has `name`, optional `description`, and `cards`. Each card needs `layout` and usually `question_id`:

```json
{
  "name": "Satış",
  "cards": [
    {
      "question_id": 42,
      "title": "Aylık Satış",
      "viz_type": "table",
      "layout": {"x":0,"y":0,"w":12,"h":6},
      "param_bindings": {"params":{"date_from":"date_from","date_to":"date_to"}}
    }
  ]
}
```

The binding map direction is always:

```text
question parameter -> dashboard parameter
```

In the example, the question's `date_from` receives the dashboard's
`date_from`. Matching names are not an implicit binding. Only fields saved in
`params.fields` appear in the global dashboard bar. A question field omitted
from `param_bindings.params` stays card-local; it is not silently promoted to a
global field.

A card-local parameter can carry its own saved value in `param_values`, which
mirrors the binding envelope:

```json
{
  "question_id": 42,
  "layout": {"x":0,"y":0,"w":12,"h":8},
  "param_values": {"params":{"category":"100.006.001"}}
}
```

That is what lets one dashboard show the same report twice for two different
inputs. A name may appear in `param_bindings.params` OR `param_values.params`,
never both — bound means the panel owns the value, local means the card does,
and asking for both is rejected at
`tabs.{i}.cards.{j}.param_values.params.<name>`. Each name must exist in the
card question's resolved surface and each value is type-checked against that
declaration. Beyond the saved value a viewer may still override any of a card's
parameters for their own session; those runtime overrides are not persisted.

Runtime values are presence-aware. An omitted dashboard value allows its
dashboard default; a present nonblank value is sent through the binding; a
present native JSON `null` explicitly clears the dashboard default. An optional
blank dashboard field sends no override and may fall through to the question
default. A required blank dashboard field blocks only its bound cards even when
the question has a default. `0` and `false` are values. An unused required field
may show a warning, but it does not block unrelated cards.

Every data card must execute `source_type:"question"` with its saved
`question_id`. The question resolver—not the dashboard field or option query—
remains authoritative for inherited pin/rename/map routing, defaults, explicit
clears, requiredness, and column/relation effects. Do not reconstruct a card
from the question's raw underlying source or DSL.

Use the persisted card shape from `analytics_dashboards_show` as the authority when updating. If an update replaces tabs/cards, preserve unrelated cards from the existing dashboard payload.

## Safety

- Dashboard create/update/delete mutates tenant data. Inspect before mutation and keep changes limited to the requested dashboard.
- Do not create throwaway questions or raw cards. Every card should point to a stable, polished saved question.
- State which dashboard, tab, and card changed in the final answer.
