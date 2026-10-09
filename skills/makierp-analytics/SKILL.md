---
name: makierp-analytics
description: "Builds and manages the MakiERP analytics surface through MCP: collections, saved questions, visualizations, dashboards, exports, and scheduled email delivery. Use when the user asks for analytics reports, saved questions, charts, KPI cards, dashboards, dashboard params, exports, recurring report emails, visualization polishing, or reusable reporting artifacts."
---

# MakiERP Analytics

## Quick Start

1. Use `makierp-mcp-session` first if tenant orientation or access is not established.
2. Start with the surface map below, then load the specific reference for the artifact you are creating.
3. Use `makierp-dsl` before composing or debugging non-trivial Manifold DSL.
4. Use `makierp-schema-docs` before field, enum, relation, currency, label, or product-behavior assumptions.

## Surface Map

- Read-only answer only: use `makierp-erp-data`, not this skill.
- Saved/reusable report: [Analytics Question Builder](references/analytics-question-builder.md).
- Manifold DSL syntax, functions, and diagnostics: use `makierp-dsl`.
- Visualization selection and shared `_columns`: [Visualization Overview](references/visualization-overview.md).
- Tables and pivots: [Table And Pivot Visualization](references/visualization-table-pivot.md).
- KPI/number cards: [Number Visualization](references/visualization-number.md).
- Bar, line, area, combo, and scatter charts: [Cartesian Visualization](references/visualization-cartesian.md).
- Pie and correlation heatmap charts: [Pie And Heatmap Visualization](references/visualization-pie-heatmap.md).
- Distribution box plots: [Box Plot Visualization](references/visualization-boxplot.md).
- Coordinate maps: [Map Visualization](references/visualization-map.md).
- Dashboard tabs/cards/params: [Dashboard Designer](references/dashboard-designer.md).
- Export jobs and render modes: [Analytics Export Builder](references/analytics-export-builder.md).
- Scheduled delivery and dynamic audiences: [Scheduled Analytics Mail](references/scheduled-analytics-mail.md).
- Public share links for a saved question: see Sharing below (`analytics_question_shares_*`).

## Workflow

- Inspect collections with `analytics_collections_list` before any artifact create.
- Inspect existing questions/dashboards with list/show tools before updating or creating near-duplicates.
- Describe analytics sources before writing DSL, then use `makierp-dsl` for syntax and validation.
- Preview with `analytics_query_preview`; use exact preview `columns[].key` values in visualization, dashboard, and export payloads.
- For `selected_viz` or export `render_mode`, include the matching `visualization.<type>` config.
- After create/update, show the saved artifact and report id/name, source, params, visualization type, and assumptions.
- For recurring email, inspect the question first, choose only the requested recipient sources, select one format supported by the saved question view plus any tabular data attachments, call `analytics_mail_schedules_preview` with the structured calendar rule, then create/update only after the summary and next occurrences match the request. Report the timezone, files, audience switches, and next occurrences. Use immediate run only when explicitly requested.

## Sharing A Saved Question

A saved question can be shared through a **tenant-controlled bearer link** that intended B2B recipients can open without a MakiERP account. Anyone holding the link can use it within the deployment's access boundary. Use these when the user asks to share, publish, send, or get a link to a report.

- `analytics_question_shares_create` — publish a question. Optional expiry as an ISO 8601 datetime; omit for a link that never expires. A question can have only one active link at a time: if one already exists, the tool returns an error carrying that existing link — surface that link instead of retrying.
- `analytics_question_shares_list` — the links the current user can manage, filterable by state (active/expired/revoked), collection, name search, or a single question.
- `analytics_question_shares_revoke` — stop a link from working for anyone holding it.
- `analytics_question_shares_set_expiry` — change a link's expiry, or clear it so it never expires.

Report the share link, its state, and its expiry back to the user in plain terms.

The public GET producer currently accepts query-string `params`, but the
caller-input policy, allowlist, URL/log/cache exposure rules, transitive access
policy, and required-input UX are not approved. The current public producer also
remains outside the common saved-question execution path. Native JSON `null` is
the explicit-clear signal on HTTP/MCP execution surfaces; there is no approved
GET encoding that extends that contract to a public link. Existing public GET
transport and encoding remain unchanged. Do not append, recommend, or promise
runtime params on a share URL, and do not claim saved-question parameter parity
for public links. Share lifecycle management remains supported while this
product/security boundary is unresolved.

## Boundaries

- Questions, collections, dashboards, and exports mutate tenant state.
- A share link exposes the question's current data on a public URL to anyone who holds it; confirm the user wants that before creating one.
- Public-share runtime params and GET null/default semantics are unresolved; manage the link lifecycle only and never invent a URL encoding or input policy.
- Do not save raw generated labels, unpreviewed DSL, or visualization configs using guessed column keys.
- Do not look for an export list/update/delete tool; this MCP surface exposes export create and show only.
- Scheduled-mail run sends private external email. Do not call `analytics_mail_schedules_run` merely to preview a definition; show the schedule and its next occurrences instead.
