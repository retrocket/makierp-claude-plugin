# Schema And Docs Guide

Use this for explaining MakiERP product structure, schemas, fields, enums, and documented behavior.

Examples:

- "Cari ne demek?"
- "customer schema hangi alanları içeriyor?"
- "Fatura durum enum değerleri neler?"
- "KDV Matrahı hangi kolon?"
- "Bu ekranın kullanım dokümanı var mı?"

## Workflow

1. Start with `docs_search`. For MCP-specific agent docs, search with tag `mcp`; for product docs, search by Turkish business term and entity name.
2. Use `docs_read` with `kind: "page"`, `kind: "schema"`, or `kind: "enum"` after search gives a stable id/name.
3. Use `schema_list` and `schema_describe` when compiled docs are missing or when current tenant schema metadata matters.
4. For labels, currency basis, and ERP vocabulary, prefer current docs and schema metadata over memory.
5. Give a concise explanation with source names and distinguish documented conventions from inferred schema facts.

## Search Strategy

- Start broad with the Turkish business term: `cari`, `fatura`, `stok`, `tahsilat`, `KDV`, `banka`.
- Add schema ids or field words after the first search result: `customer balance`, `invoice status`, `bank transaction slip type`.
- Use `kind:"schema"` or `kind:"enum"` when the user asks for fields or allowed values.
- Use tag `mcp` when the user asks how an agent should use tools or DSL.

## Schema Strategy

Use `schema_list` to discover exact schema identifiers. Use `schema_describe` to read fields, relation names, related schemas, filterability/sortability, and paths for analytics DSL. Current schema metadata is authoritative for field existence. Product docs are authoritative for business meaning when they exist.

## Answer Style

- Name the exact docs/schema entries used.
- For fields, include path, label, type, and relation/enum context when available.
- For enums, list allowed values exactly as returned.
- For currency fields, explain YPB/IPB/RPB when the source identifies the basis.
- If docs and schema disagree, say so and prefer schema for current field availability.

## Boundaries

- This workflow explains and orients. It does not preview tenant data unless the user asks for actual data.
- Do not create or update analytics artifacts. Route report-building requests to `makierp-analytics`.
