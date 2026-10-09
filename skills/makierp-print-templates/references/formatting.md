# Formatting, Locale And Header Images

Use this for requests like:

- "Tutarlar 1.234,56 biçiminde görünsün"
- "Şablonun üstüne firma logomuzu koy"
- "Tarih sütunu gün.ay.yıl olsun"

## Two Format Layers — `ValueFormat` Wins

A textbox can carry two formats, and the structured one **overrides** the legacy one:

| Key | Shape | When to use |
|---|---|---|
| `ValueFormat` | structured object (analytics format config, e.g. `{"type":"currency","currency_column":"currency_code","decimals":2}`) | new templates needing currency-from-column, compact numbers, zero styling, custom Evet/Hayır labels |
| `Style.Format` | .NET format string (`"N2"`) | everything the seeded corpus uses; keep it when editing an existing template |

- `ValueFormat` sits on the **textbox item itself**, next to `Value` and `Style` — never inside `Style`. A `Style.ValueFormat` is an unknown key: silently ignored.
- When `ValueFormat` is present, that cell's `Style.Format` is **ignored**.
- "Automatic" is not a value — it means **deleting the `ValueFormat` key**. Never write `"auto"` or an empty object as a sentinel: an empty object is still present, so it wins over `Style.Format` and drops the cell to the unformatted display string.
- None of the shipped templates use `ValueFormat`; the seeded corpus is pure `Style.Format`. Leave those cells alone unless the user asks for something only `ValueFormat` can do.

## `Style.Format` Codes

`N2` grouped, 2 decimals (tr-TR `1.234,56`) · `N0` · `F2` (no grouping) · `N4` for exchange rates · `D3` · `P2` · `E`. Dates: `dd.MM.yyyy`, datetime `dd.MM.yyyy HH:mm`.

## Locale Traps

- **`c` / `c2` / `c3` print NO currency symbol.** The engine has no tenant-currency source and refuses to invent one, so it prints the bare grouped number and reports a warning. Put the symbol in the data or in the column header, or use `ValueFormat` with `currency_column`. Say: "Para birimi simgesini şablon basamıyor; simgeyi sütun başlığına yazayım mı?"
- **Month and day names are English only** — `MMMM` prints "January", never "Ocak"; `MMM`/`dddd` likewise. For Turkish use `dd.MM.yyyy`, or a label the data already computed.
- **Separators follow the tenant's number-format preference** (tr-TR / en-US). Never bake separators into the data: send the raw number and format with `N2`. A pre-formatted `"1.234,56"` is text — it will not re-group, right-align or total correctly, and it pollutes the Excel output.
- **`FontFamily` is meaningless.** The engine pins `ReportSans` with `!important` across the viewer, the print iframe and the export bundle so page breaks match everywhere. Do not promise a font change.
- **Never format `Globals!PageNumber` / `Globals!TotalPages`.** In flowing mode they are text sentinels that the paginator restamps per page, so a numeric `Style.Format` is a silent no-op, and a format that rewrites the text (a `date` or `boolean` `ValueFormat`) eats the sentinel — the page number then never prints at all. Concatenate them with `&` and nothing else (see `expression-language.md`).

## Every `Style` Value May Be An Expression

Style values are evaluated **per row**, exactly like `Value`:

```json
"FontWeight": "=IIF(Fields!has_children.Value, \"Bold\", \"Normal\")"
```

The designer canvas cannot preview this (it has no row data); real output works. Mention that when the user says the styling looks wrong in the designer.

## Format The Cell, Not The Value

A repeated starter mistake: hand-building the display text made boolean columns print `true`/`false` and numbers `1234.5670000`. Keep `Value` a plain field reference so the Excel output stays clean, and put presentation in the format:

- number → `Style.Format: "N2"`
- date → `"dd.MM.yyyy"`; datetime → `"dd.MM.yyyy HH:mm"`
- boolean → `=IIF(<ref>, "Evet", "Hayır")` — the truthy form. `<ref> = true` is **wrong**: the value often arrives as the string `"true"`/`"false"` and the case mismatch prints "Hayır".

> **The truthy `IIF` is not universally right — the TEXT `'0'` prints "Evet".** The engine's boolean coercion maps only `'true'` to true and `'false'`/`''` to false; **every other string is true**. So a 0/1 flag that arrives as a numeric string is truthy. There, use `=IIF(Fields!f.Value = 1, "Evet", "Hayır")` — a comparison where both sides look numeric is compared numerically. The truthy form is safe only when the value is a real boolean, the string `'true'`/`'false'`, or the **number** 0/1. If the column's real shape is unknown, **ask**: "Bu alan Evet/Hayır bilgisini nasıl tutuyor — doğru/yanlış olarak mı, 0/1 olarak mı?" Guessing is not allowed.

## Logo And Letterhead Images

Two halves that must match by name:

```json
"EmbeddedImages": [{ "Name": "logo", "MIMEType": "image/png", "ImageData": "<base64 — NO 'data:' prefix>" }]
```
```json
{ "Type": "image", "Name": "Logo", "Source": "Embedded", "Value": "logo",
  "Left": "1cm", "Top": "0.5cm", "Width": "4cm", "Height": "1.5cm", "Sizing": "FitProportional" }
```

- The engine builds the `data:` URI itself from `MIMEType` + `ImageData`; base64 that already carries a `data:` prefix renders broken.
- If `Value` does not match an `EmbeddedImages[].Name`, the image prints **nothing, silently**. `print_templates_validate` catches this; a `Value` written as an expression (`=…`) cannot be checked statically.
- `Sizing`: `FitProportional` keeps the aspect ratio, `Fit` stretches to the box, `Clip` and `AutoSize` do not scale, and an omitted `Sizing` behaves like `FitProportional`.
- **`Style.BackgroundImage` is never drawn** by the engine — the type exists, no renderer reads it. One shipped invoice template keeps a 611 KB `EmbeddedImages` entry that is referenced **only** from `Style.BackgroundImage`, so those 611 KB have never printed a single pixel. Always point an `image` item at the embedded name instead.
- Size: images are by far the largest part of a definition. Newly added `ImageData` has an upper bound (512 KB per image, 1 MB per template); an image carried over from the saved record is exempt.
- **On update the existing `EmbeddedImages` block must be carried over.** If a new definition arrives without it while the saved record has one, the block is re-attached or the update is refused — dropping a letterhead silently is forbidden. Read the record with `print_templates_get` first and tell the user: "Mevcut antet görselini olduğu gibi koruyorum."

## `definition.Name`

`Name` is the Excel **sheet name** (trimmed to 31 characters) and the document title; when empty the output falls back to `Export`. Keep it in sync with the template record's name — `print_templates_create` / `print_templates_update` fill it from the record name when empty and warn when the two disagree. Never copy the template-generator leftover `multisectionreport1`: all nine record-print seeds carry it, and a new template must get a real name the user recognizes.
