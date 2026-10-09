# Analytics Question Builder

Use this for requests like:

- "Son 1 haftadaki satış toplamlarını rapor yap"
- "Aylık tahsilat raporu oluştur"
- "Cari bazında net satış sorusu kaydet"

## Required Reading

- Use this file for the full save sequence.
- Use `analytics_source_describe` output for source shape; do not rely on memory.
- Use `makierp-dsl` before composing aggregate, grouped, include, param, ordered, or function-heavy DSL.
- Use `visualization-overview.md` before choosing `selected_viz` or saving visualization payloads.
- Use `makierp-schema-docs` before Turkish labels, money columns, field meanings, enum meanings, or product behavior assumptions.
- Use exact preview `columns[].key` values before saving visualization payloads.

## Workflow

1. Discover collections with `analytics_collections_list` so create/update uses a valid `collection_id`.
2. Inspect existing questions with `analytics_questions_list` / `analytics_questions_show` before updating or creating a near-duplicate.
3. Discover sources with `analytics_sources_list`.
4. Describe the chosen source with `analytics_source_describe`. Do not invent columns, relations, enum values, or value types.
5. Compose the smallest DSL that answers the user. Computed select nodes need stable snake_case `as` aliases and Turkish `label` values. For row counts, use `{"fn":"count","args":[]}`; for distinct counts, use `count_distinct` with one var argument.
6. If params are needed, use the UI model: ordinary/free params omit `kind`; schema-aware column params use `kind:"column"` with `column_filter`; relation picker params use `kind:"relation"` with `relation_exists`. Both effects target `where`.
7. Call `analytics_query_validate` with `data_source_type`, `data_source_id`, `query`, and optional validation `params`. Fix all diagnostics before preview.
8. Call `analytics_query_preview` when available. Use exact preview `columns[].key` values in visualization configs. If the preview/execute tool is not available, validation proves DSL shape only; do not claim returned data.
9. Choose visualization through `visualization-overview.md` and the type-specific visualization reference.
10. Save through `analytics_questions_create` or `analytics_questions_update` only after preview. Include polished `name`, `description` when useful, `selected_viz`, `visualization`, and complete `visualization._columns`.
11. After saving, report the saved question id/name and the source/date/currency assumptions.

## Question Kind

- Saved Questions return top-level `query_kind:"regular"|"aggregate"`. This remembers the authoring kind, independently of `selected_viz`; there is no separate view artifact.
- Creation can omit the kind: root grouping and non-window aggregate expressions (including inside CASE or scalar functions) infer `aggregate`. Window-only projections and aggregates inside relation includes do not change the outer Question's kind.
- Replacement `query` updates without `query_kind` infer again. Metadata-only updates preserve the saved kind. Send `query_kind:"aggregate"` to retain an aggregate draft after its final measure is removed.
- Actual aggregate DSL always resolves to `aggregate`, even with stale `regular` metadata. Older Questions with no persisted kind are inferred when read and gain a stored kind on their next save.

## Parameter Contract

- Free params use canonical `type:"text"` (plus `number`, `date`, `datetime`,
  `boolean`, or `select`) and affect only their explicit `{param}` nodes or
  source bindings. Existing `string` is legacy input; do not emit it in new
  definitions.
- Column and relation params are closed conditional filter effects. They append
  one typed pre-aggregation predicate to an execution copy when active. An
  optional blank value appends nothing; a required blank value fails.
- Ordinary params may own `effect:{type:"query_fragment",query:{...}}` using
  the supported DSL clauses. Optional blank owners skip the entire fragment.
  Keep `_params_version:2` when replacing fragment-bearing queries; query-backed
  choices only supply input values and do not apply query behavior themselves.
- Runtime `params` distinguish presence: omit a key to allow its default, send
  a nonblank value to override it, and send native JSON `null` to explicitly
  clear it. Clear suppresses an optional default/effect and leaves a required
  param missing. `0` and `false` are real values.
- For a saved-question source, use the computed `surface` returned by current
  source/question inspection. Do not copy inherited declarations into the
  outer question. Use `_param_intercepts` to `pin` a hidden fixed value,
  `rename` the outer name, or `map` one compatible own param into an inherited
  target. Only computed outer-surface names accept caller values.

## Tool Navigation

- `analytics_collections_list`: find writable collection ids.
- `analytics_questions_list`: find existing questions; filter by `collection_id` when useful.
- `analytics_questions_show`: inspect the saved query, current computed surface, selected visualization, and column labels before updating.
- `analytics_sources_list` / `analytics_source_describe`: pick the source and field paths; for question sources, use the computed surface rather than owned `_params` alone.
- `analytics_query_validate` / `analytics_query_preview`: verify before save.
- `analytics_questions_create` / `analytics_questions_update`: persist only after validation and preview.

## Safety

- Saving is a tenant data mutation. Inspect existing questions first when updating.
- Never save a raw-looking result with generated aliases or English column labels.
- If the user only wants an answer, use `makierp-erp-data` instead of saving.
