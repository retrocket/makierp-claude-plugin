---
name: makierp-erp-data
description: Answers read-only tenant ERP data questions through MakiERP MCP analytics and schema tools. Use when the user asks for balances, totals, stock movement, sales, purchases, finance data, or ad hoc ERP investigation without asking to save a report.
---

# MakiERP ERP Data

## Quick Start

1. Use `makierp-mcp-session` first if tenant orientation or access is not established.
2. For entity meaning, field meaning, enum meaning, or product behavior, use `makierp-schema-docs`.
3. For actual data answers, follow [ERP Data Investigator](references/erp-data-investigator.md).

## Workflow

- Use `analytics_sources_list` and `analytics_source_describe` before writing a query.
- Use `schema_list` and `schema_describe` for schema-only inspection.
- Use `makierp-dsl` before composing or debugging non-trivial Manifold DSL.
- Build the smallest read-only query, then call `analytics_query_validate` with `data_source_type`, `data_source_id`, and `query`.
- Preview with `analytics_query_preview` for bounded execution, using `page` and `per_page` when the answer needs sample rows.
- If preview is unavailable or returns diagnostics only, say what was validated and what could not be confirmed.
- Do not save questions, dashboards, collections, or exports unless the user explicitly asks.

## Handover

- Switch to `makierp-analytics` when the user wants a saved/reusable report, visualization, dashboard card/layout, collection, or export.
- Switch to `makierp-bank-transactions` when the user wants to create/update/delete bank transactions rather than inspect existing data.
