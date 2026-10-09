---
name: makierp-dsl
description: Composes, validates, and debugs MakiERP Manifold DSL query payloads for analytics sources. Use when the user or another skill needs DSL syntax, count/aggregate/group_by/having/include/params/order_by/window function guidance, query validation diagnostics, or function details before ERP data or analytics work can proceed.
---

# MakiERP DSL

## Quick Start

1. Use `analytics_sources_list` and `analytics_source_describe` first; the source description is the authority for fields, scopes, params, and starter DSL.
2. Read [DSL Reference](references/dsl-reference.md) for query shape and diagnostics.
3. Read [Function Reference](references/function-reference.md) for functions, aggregates, date helpers, windows, and computed metrics.

## Workflow

- Build the smallest valid raw DSL first, then add filters, aggregates, grouping, includes, params, and ordering.
- Pass `data_source_type`, `data_source_id`, `query`, and optional `params` to `analytics_query_validate`.
- Preview with `analytics_query_preview` only after validation is clean.
- Fix the first structural diagnostic before changing business logic; later diagnostics are often fallout.
- Use stable snake_case `as` aliases and Turkish `label` values for computed select nodes.

## Handover

- Return to `makierp-erp-data` for read-only answers after the query previews.
- Return to `makierp-analytics` for saved questions, visualizations, dashboards, exports, and collections.
- Use `makierp-schema-docs` before assuming field meanings, enum values, relations, labels, or currency basis.

## Boundaries

- This skill only handles Manifold DSL and function syntax.
- Do not save questions, dashboards, exports, or tenant records from this skill.
- Do not invent source ids, field paths, relation scopes, or function names.
