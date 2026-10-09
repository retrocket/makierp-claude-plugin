# Expression Language

Use this for requests like:

- "Şablondaki müşteri adı boş çıkıyor"
- "Toplam satırına genel toplamı yazdır"
- "Vergi numarası yoksa etiketi de görünmesin"

Every expression starts with `=`. Most traps below are silent at render time, so run `print_templates_validate` before `print_templates_create` / `print_templates_update`.

## Iron Rule: `Fields!x` Takes The Data Key, Never The Label

`x` is the key of the row object, i.e. `DataSets[*].Fields[].DataField`. `Fields[].Name` is a **label**, so an expression never takes a column heading. **This is the inverse of standard RDL.** The engine does not read `DataSets[*].Fields` while rendering; `Fields!x` is resolved straight off the raw row object.

In analytics starter templates `Name` is the Turkish label and `DataField` is the key, so `Fields!<Name>` both misses the real key and raises a `LexError` on the Turkish characters — and a lex error takes the whole document down. Always take keys from `print_templates_sources`.

The label/key split only bites where **both** are present. In the report and record corpus most `Fields` entries carry `Name` alone, and there `Name` **is** the row key — both sides resolve the key as `DataField ?: Name`. So a seed template writing `=Fields!document_date.Value` against a `Name`-only entry is correct, not a mistake to "fix".

`DataSets[0].Fields` is not a declaration, it is the **xlsx/csv/json column plan**: columns come out in that order, the row key is `DataField ?: Name`, the heading is `Name`. Fields present in the row data but absent from that list still resolve in cells (`tenant_name`, `total_*`, `cum_*`) and stay out of the spreadsheet — so never list meta fields there, or they become junk columns. Leaving the list out entirely is worse than trimming it: the export then dumps **every raw row key** under its technical name, meta fields included.

## The Two Accepted Syntaxes

| Form | Rule |
|---|---|
| `Fields!alan_adi.Value` | name must be an ASCII identifier: `[A-Za-z_][A-Za-z0-9_]*` |
| `Fields.Item("noktali.yol").Value` | argument must be **double**-quoted |

Selection rule, the canonical one used by the designer:

```
/^[A-Za-z_][A-Za-z0-9_]*$/.test(name) ? `=Fields!${name}.Value` : `=Fields.Item("${name}").Value`
```

A dot, a space, a Turkish character, a `-`, or a leading digit makes `Fields.Item("…")` **mandatory**. Two forms are rejected outright: `Fields("x")` → `ParseError: Expected dot but got '('`, and `Fields.Item('x')` with single quotes → `LexError`.

## Dotted Bang Trap: `[object Object]`

`=Fields!position.code.Value` raises **no error**. The parser takes `position` after the `!`, swallows `.code` as a field property, and evaluation ignores the property entirely (only `IsMissing` is special). The expression returns the `position` **object** and the cell prints `[object Object]` (blank when the relation is null). Write `=Fields.Item("position.code").Value`.

Rule to check: the name after the bang, up to the first dot, must match the identifier regex, with no extra property between it and a trailing `.Value`. The canonical `=Fields!x.Value` is **valid** — that `.Value` is a property, not part of the name. "Any dot is an error" would reject correct code.

## Two Error Classes

| Class | Examples | Result |
|---|---|---|
| Unrecognized function name | `IIf`, `iif`, `sum`, `StDev`, `Sqrt` | **silent empty cell**, no error anywhere |
| Syntax error | single quotes, `Fields(`, missing paren, `IIF` with 2 args (only when the condition is FALSE) | **the whole document falls** |

A syntax error is never one blank cell: nothing wraps expression evaluation in a `try/catch`, so one bad expression kills every item on the page. What the user sees depends on the path:

- **Static/paged path** — the error surfaces and the export is marked **FAILED**.
- **Flowing/galley path** — the error is **swallowed**: the body is never written, the backend only waits for the ready flag, and the export completes **successfully with an empty PDF**. Worse than FAILED, because nobody notices. Galley-only expressions (page-boundary band cells, `ContinuationLabel`, group carry) also pass the paged pre-render cleanly, so only a static check finds them.
- **Viewer** — opens with **no error and 0 pages**.

Neither class is visible from the rendered output alone. That is why every expression is parsed statically before a write goes through — and "every expression" is wider than textbox `Value`: `Visibility.Hidden`, every `Style.*` entry, item-level `HorizontalAlignment` / `VerticalAlignment`, `GroupExpressions`, `NoRowsMessage` and `ContinuationLabel` are compiled the same way and down the same document.

## `IIF` / `IsNull` — Uppercase, Lazy, Exactly 3 Arguments

- Case matters for every builtin: `IIF` and `Format` work, `IIf` / `iif` / `format` do not — they fall through to the "unknown call" branch and print an empty cell without complaining.
- `IsNull` is **not** a null test, it is an alias of `IIF`. The only null test is `IsNothing(x)`. (A 1-argument `IsNull` sits in the builtin table but the special form intercepts the name first — unreachable dead code.)
- Both take **exactly 3 arguments**. Fewer throws a `TypeError`, and a 2-argument `IIF` throws **only when the condition evaluates FALSE** — it can run for weeks on data where the condition is always true, then crash on a different record. Never ship a 2-argument `IIF`.

## Supported Functions

| Category | Functions |
|---|---|
| Scope | `First(expr[,"scope"])`, `Last(...)`, `Previous(...)`, `RowNumber()`, `CountRows([scope])` |
| Aggregate | `Sum/Avg/Count/Min/Max/Median(expr[,"scope"])` |
| Color | `ColorScale(value,min,max,minColor,maxColor)` |
| Conditional | `IIF(cond,a,b)` — lazy |
| Null | `IsNothing(x)`, `IsDate(x)`, `IsNumeric(x)` |
| Date | `DateTime.Parse`, `CDate`, `ToDateTime`, `Year/Month/Day/Hour/Minute/Second/Weekday` |
| Number | `Abs, Round([n]), Ceiling, Floor, Sign, CInt, CLng, CDbl, Val, Int, Fix` |
| Text | `Format(v,fmt), Trim/LTrim/RTrim, UCase/LCase, Len, Left/Right/Mid, Replace, CStr, CBool` |
| Value methods | `.ToString([fmt]), .Substring, .Replace, .Trim, .ToUpper/.ToLower, .PadLeft/.PadRight, .IndexOf, .Contains, .StartsWith, .EndsWith, .Length` |
| Globals | Seven members, no more: `Globals!PageNumber`, `!TotalPages`, `!ReportName`, `!ExecutionTime`, `!TenantName`, `!UserName`, `!PrintDate`. The last three come from the print metadata, so they are empty strings wherever that metadata is absent — fine in a page header, never as data. Any other member is rejected |
| Code.* | only `FormatCurrencyValue`, `FiveDecimalFormatCurrencyValue`, `FormatDateDayMonthHourMinute` |

**Not supported — silently `null`:** `Parameters!x`, `ReportItems!x`, `Variables!x` (the parser folds `Globals!`, `Parameters!`, `ReportItems!` and `Variables!` into one globals lookup, and the bag carries no entry under those names), plus `StDev`, `Var`, `Aggregate`, `Sqrt`, `Pow`, two-argument `Min(a,b)`, `Log`, and any other unknown call.

Because there are no report parameters, every report-level value — company name, customer code/name, date range, grand total — is embedded by the data source on **every row** and read with `=First(Fields!tenant_name.Value)`. That is the corpus-wide pattern.

## Scope Semantics — Two Different Rules, Do Not Merge Them

| Call | Scope when no scope argument is given |
|---|---|
| `Sum/Avg/Count/Min/Max/Median`, `CountRows()` | the **innermost** data scope (table Footer = the region's rows, group Footer = that group's rows) |
| `First/Last/Previous` | the **root** collection |

- A scope name must be a **quoted string literal**: `Sum(x, "Rows")`. A bare identifier resolves to `"null"`.
- `Count` counts values that are not null/undefined/`''`; an empty set yields `null` for every aggregate except `Count`, which yields `0`. There is **no date aggregation** — date min/max is precomputed in the data.
- **Never use argless `First(...)` in a group band.** It reads the root, so every group footer prints the **first** group's subtotal, with no error. Use `=Fields!x.Value` (the band's row is already the group's first row) or an argless `=Sum(Fields!debit.Value)`. Reserve `First` for report-level values.
- **`Previous(...)` behaves as `First(...)`** — it does not give you the previous row. Running totals and deltas must be precomputed in the data (`cum_*`).

## Operators And Literals

`&& || ! == !=` do not exist. Use `AndAlso`, `OrElse`, `Not`, `=`, `<>`. Concatenation is `&`. String literals are **double-quoted only** (`"Bold"`), escaped as `""`; a single quote is a `LexError`, which takes the whole document down.

## `Globals!PageNumber` Is A Sentinel In Flowing Mode

The FE viewer is galley for **every** template, and on the backend the PDF path is galley for document-shaped reports and nested-lines analytics templates (`tables-and-pagination.md` carries the full gate). There `Globals!PageNumber` / `!TotalPages` are placeholder tokens the paginator restamps as **text** — they are not numbers. The non-galley renders — record prints, a report whose source has no document shape (today "Muhasebe Hesap Ekstresi") and a flat analytics template — substitute real numbers instead, so the same cell behaves differently in the two renders.

The only valid use is a `&` chain, and it is the only pattern in the seed corpus: `="Sayfa: " & Globals!PageNumber & " / " & Globals!TotalPages`. `=Globals!PageNumber + 1` gives `NaN`, `=IIF(Globals!PageNumber = 1, …)` is FALSE on every page, and applying `ValueFormat` / `Style.Format` corrupts the token. All three fail silently.

## Null Relations And The `"NaN"` Cell

A `null` field prints as an empty string on its own. The damage appears in concatenation and arithmetic:

| Written as | Output when the field is null | Correct form |
|---|---|---|
| `=Fields!x.Value` | blank — fine | — |
| `="Kod: " & Fields!x.Value` | `"Kod: "` — orphan label | `=IIF(IsNothing(Fields!x.Value), "", "Kod: " & Fields!x.Value)` |
| `=Fields!a.Value + Fields!b.Value` | `"NaN"` printed in the cell | `=IIF(IsNumeric(Fields!a.Value), Fields!a.Value, 0) + IIF(IsNumeric(Fields!b.Value), Fields!b.Value, 0)` |
| `=IIF(IsNothing(Fields!a.Value),0,Fields!a.Value) + Fields!b.Value` | still `"NaN"` when the field is `''` | same — use `IsNumeric` |
| zero amount | `"0,00"` — noise in a statement | make the data source return `null` for zero |

**`IsNothing` is not enough in arithmetic.** It is true only for `null`/`undefined`, but an empty string also coerces to `NaN`, and `NaN` is printed literally. Empty strings are realistic input (text columns, blank numeric fields, analytics output). Guard arithmetic with `IsNumeric`, which accepts numeric strings and rejects `''` and text; in text concatenation `IsNothing` is the right guard.

Guard the label and the value in the **same** expression, otherwise a customer with no tax number gets a stray "Vergi No:" in the output. When a field the user asked for cannot be produced at all, say so plainly — "Bu alan seçtiğiniz veri kaynağında yok, bu yüzden şablona eklenmedi." — rather than printing an empty labelled row.

## Do Not Rewrite Existing Templates With Aggregates

None of the shipped templates use `Sum(`, `Avg(`, `Count(`, `Min(`, `Max(`, `Median(`, `CountRows(`, `ColorScale(`, `Previous(`, `Last(`, `RowNumber(`, `ValueFormat`, `Chart`, or `Parameters!`. Their canonical pattern is precompute-in-the-data plus `=First(Fields!total_x.Value)`, and it works. Aggregates are available for **new** templates; converting an existing one to aggregates is out of scope, even when it looks cleaner.
