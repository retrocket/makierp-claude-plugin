---
name: makierp-schema-docs
description: Explains MakiERP schemas, fields, enums, relations, and documented ERP behavior using MCP docs/schema tools. Use when the user asks what something means, how a model is structured, which fields exist, or how documented product behavior works.
---

# MakiERP Schema Docs

## Quick Start

Follow [Schema Docs Guide](references/schema-docs-guide.md) when explaining product structure or behavior.

## Workflow

- Use `docs_search` first for documented behavior; search Turkish business terms, schema ids, enum names, and `mcp` tags when relevant.
- Use `docs_read` with the exact `kind` and `name` returned by search.
- Use `schema_list` and `schema_describe` for current schema metadata, especially field paths and relation names used in analytics DSL.
- Use exact schema field, relation, enum, and tool names from MCP responses.
- If the user asks for data totals after the explanation, switch to `makierp-erp-data`.

## Boundaries

- Do not invent columns, enum values, relation names, or business rules.
- Do not treat old docs as more authoritative than current schema metadata.
- Distinguish documented behavior from schema-derived inference when answering.
