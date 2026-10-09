# Analytics Export Builder

Use this for requests like:

- "Aylık tahsilat CSV'si hazırla"
- "Bu soruyu dışa aktar"
- "Satış toplamları raporunu export et"

## Workflow

1. Inspect candidate questions with `analytics_questions_list` / `analytics_questions_show`. There is no export list tool; use `analytics_exports_show` only when you have an export id.
2. If the export needs a new or changed question, use `analytics-question-builder.md` first.
3. Ensure the saved question has been validated and previewed, and that labels are polished. Use `makierp-schema-docs` for Turkish labels, field meanings, and currency bases.
4. Use exact preview column keys if export output depends on table/column visualization state.
5. Use `visualization-overview.md` and the matching visualization reference before setting `render_mode`.
6. Create the export through `analytics_exports_create` only after confirming the target question and params.
7. Inspect export status after creation when the user expects a generated artifact.

## Tool Navigation

- `analytics_questions_list`: find candidate saved questions. Use `collection_id` when the user names a collection/dashboard area.
- `analytics_questions_show`: inspect the question id, current computed surface, selected visualization, export options, and column labels. Do not replace the question reference with its underlying source/query.
- `analytics_query_validate` / `analytics_query_preview`: verify the saved-question source and runtime params before exporting.
- `analytics_exports_create`: submit the export job. It returns the export object and may enqueue work.
- `analytics_exports_show`: read job status/result after create or when the user provides an export id.

## Export Create Shape

Use `analytics_exports_create` with:

```json
{
  "data_source_type": "question",
  "data_source_id": "42",
  "query": {},
  "params": {},
  "format": "xlsx",
  "delivery": "download",
  "filename": "aylik-tahsilat",
  "render_mode": "table",
  "visualization": {
    "table": {},
    "_columns": []
  },
  "per_page": 10000
}
```

Allowed formats are `json`, `csv`, `xlsx`, `pdf`, `svg`, and `png`.
`delivery` defaults to download. `per_page` is capped at 10000.

For a saved-question export, `data_source_type:"question"` plus the saved
question id is the execution authority. Keep the required MCP `query` object
empty to run that question unchanged; do not copy its underlying source or
saved DSL into a new export definition. Use the current saved visualization and
export presentation returned by `analytics_questions_show` unless the user asks
to change the question first.

Runtime `params` use the saved question's current computed surface. Omit a key
to allow its default, send a nonblank value to override it, and send native JSON
`null` to explicitly clear it. Clear suppresses an optional default/effect and
leaves a required param missing; `0` and `false` remain values. The export runs
the same pin/rename/map and closed column/relation effect pipeline as the saved
question viewer.

## Export Through a Print Template

`export_options.template_id` renders the question's rows through a designed print template instead of the built-in renderers. Use it when the user wants their own layout ("kendi fatura düzenimle", "şablonlu PDF"). Design or change the template with the `makierp-print-templates` skill.

- **Only `pdf` and `xlsx` honour it.** With `csv`, `json`, `svg`, or `png` the template is dropped without any error and the built-in output is produced — the user reads it as "şablonum uygulanmadı". If the user asks for a template with one of those formats, say the format does not carry a layout and offer PDF or Excel.
- **Find the template with `print_templates_list`, filtering `type` to `analytics_export`.** These templates are hidden from the application's own template index, so listing them through the tool is the only inventory.
- **The backend does not check the template's type.** The rule is only `exists:print_templates,id` and the producer loads the row by id, so an invoice or statement template is accepted just as readily. Verify the type yourself before passing an id.
- **The one real guard is the column gate, and it is reference-based.** Every field the template actually binds (`Fields!x.Value`, `Fields.Item("x")`, native chart config columns) must exist among the question's output columns; otherwise the export fails with *"Seçilen şablon, bu sorunun üretmediği kolonlara bağlı: …"*. A template that merely declares a field it never prints passes. A wrong-type template whose columns happen to line up is rendered silently.
- **A chosen template overrides the rest of `export_options`.** Page size, orientation, margins, font size, page numbers, repeating headers, subtitle and `include_*` switches, and the `xlsx_*` sheet options are all ignored: page geometry comes from the template's own page setup and the document name from the template definition. Only `filename` (else `title`) still names the file. So "kağıdı yatay yap" is a change to the template, not to the export request.
- In `xlsx` the template contributes its column plan and document name, not its layout; the full PDF design applies to `pdf` only.

## Safety

- Export creation mutates tenant state and may enqueue work. Be explicit about that action.
- Do not export from a question with stale visualization keys or raw generated labels.
- Report export id/status and any assumptions about params/date range.
- Do not search for non-existent `analytics_exports_list`, update, or delete tools.
