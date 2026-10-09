# ERP Data Investigator

Use this for questions like:

- "Son 1 haftadaki satış toplamları"
- "Bu carinin bakiyesi ne?"
- "Depoya göre stok hareketlerini grupla"
- "Geçen ay en çok satan malzemeler"

## Workflow

1. Treat the request as tenant-scoped. Do not try to pick tenant context from tool arguments.
2. If the user asks what an entity, field, enum, or product behavior means, use `docs_search` / `docs_read` first.
3. Use `analytics_sources_list` to find candidate sources, then `analytics_source_describe` before composing a query.
4. For schema-only investigation, `schema_list` and `schema_describe` may be enough; for actual data answers, use analytics sources.
5. Compose the smallest read-only query that answers the question. Use returned source columns only.
6. Call `analytics_query_validate` before any preview.
7. Call `analytics_query_preview` for bounded execution. Do not save collections, questions, dashboards, or exports unless the user explicitly asks.
8. Explain the answer with the filters, date range, source, and any uncertainty. If currency is involved, use `makierp-schema-docs` and name YPB/IPB/RPB explicitly.

## Tool Navigation

- `analytics_sources_list`: discover source ids and whether a source is `schema`, `report`, or `question`.
- `analytics_source_describe`: get fields, columns, relation scopes, params, and starter DSL. Do not query before this.
- `analytics_query_validate`: pass `data_source_type`, `data_source_id`, `query`, and optional runtime `params`.
- `analytics_query_preview`: same shape as validate, plus optional `page` and `per_page` for bounded execution.
- `schema_list` / `schema_describe`: use when you only need structure or field paths, not rows.
- `docs_search` / `docs_read`: use when a business term, enum, status, currency basis, or screen behavior needs explanation.

## Query Patterns

For a total or count, return one aggregate row. For a ranking, add the dimension to `select` and `group_by`, then sort by the aggregate alias and set `limit`. For sample lists, select only the fields needed to answer and cap with `limit`.

When composing Manifold DSL, use `makierp-dsl` even if you are not saving a question. The DSL syntax is the same; the difference is that this workflow remains read-only.

## Answer Style

- State the tenant/source, filters, date range, currency basis, and preview row count.
- If the preview returns diagnostics, report the first actionable diagnostic and do not provide a numeric answer.
- If only validation was possible, say the query validates but execution was not available in this session.
- If the user asks to keep the answer for later, hand off to `makierp-analytics`.

## Safety

- This is a read-only workflow. Avoid `analytics_*` mutations except list/show inspection actions.
- Do not invent columns, enum values, source ids, or relation names.
- If the user asks for a reusable report, visualization, dashboard card, or export, switch to `makierp-analytics`.
