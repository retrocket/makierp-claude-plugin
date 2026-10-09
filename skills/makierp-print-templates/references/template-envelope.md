# Template Record and Content Envelope

Use this for requests like:

- "Fatura şablonunu kopyalayıp yeni bir tane oluştur"
- "Bu şablonun sadece başlığını değiştir"
- "Kaydettim ama tasarımcıda boş açılıyor"

## The record

| Field | Rule |
|---|---|
| `name` | required, ≤255 chars |
| `type` | required, one of the 17 values below — determines the data path |
| `content` | required, **a JSON string**, not an object |
| `is_default` | required boolean; writing it has side effects (see traps) |
| `max_lines_per_page` | required only for `invoice`; integer > 0 |

`condition`, `priority` and `wildcard_filters` are **routing** fields, absent from the tool schema
and never yours to author. `print_templates_create` sends `priority: 0`, `condition: null` and an
empty `wildcard_filters`; `print_templates_update` leaves the stored routing values untouched —
it writes through the model, so the key never reaches the request that would blank it (see the
last section).

## `content` holds a `{displayName, definition}` envelope

Parsed, `content` is `{"displayName": "...", "definition": { ...RDL... }}` — all 19 seed
templates use the envelope, and it is the canonical shape.

A bare RDL root (no `definition` key) is only half-tolerated, and the half that breaks is the half
the user looks at: the PDF renderer falls back to the whole array (`TemplatePrinter::definitionOf()`
— *"Fall back to the whole array for rows already saved as a bare definition"*), but the in-app
viewer reads `JSON.parse(content)?.definition` with **no** fallback (`InHouseReportViewer`,
`PersistentReportViewer`), so Yazdır opens on an empty document. Always write the envelope.

**Double encoding.** The model casts `content` to `array` while validation demands a JSON string,
so a string written onto the cast column encodes a second time and reads back as a string. The
renderer compensates at runtime (`TemplatePrinter::definitionOf()` — comment: *"The array cast's
type is a lie for legacy rows"*). Decode once and you hold a string where an object is expected;
the designer opens blank.

**Reading recipe**, for anything you read yourself:

```
content → is it a string? → JSON.parse
        → still a string?  → JSON.parse ONCE MORE
        → parsed.definition ?? parsed
```

You never build this string by hand: `print_templates_create` / `print_templates_update` take
`definition` as an **object** and wrap the envelope themselves.

## The 27 valid `type` values

```
invoice · cheque · storage_slip · way_bill · accounting_slip · order · customs_clearance_slip ·
customer_slip · bank_transaction · cheque_roll · cost_slip · sale_provision_slip ·
customer_slip_line · bank_transaction_line · safe_line · accounting_slip_line ·
customer_statement · accounting_account_statement · bank_account_statement · safe_statement ·
accounting_account_list · accounting_subsidiary_ledger · accounting_trial_balance ·
accounting_journal · accounting_general_ledger · analytics_export · detailed_cost_analysis
```

`customer_slip`, `safe_line` and `bank_transaction` were dropped on 2026-08-10 and **came back on
2026-08-25** with real endpoints of their own, alongside six new siblings. Never read a type count
off the seed **file** count — they have never matched. The common slips:
irsaliye is **`way_bill`** (not `waybill`), ambar fişi is `storage_slip`, muavin is
`accounting_subsidiary_ledger`, mizan is `accounting_trial_balance`.

**`type_label` is not guaranteed.** `prettyName()` reads the compiled enum docs first (those carry
Turkish labels for all 27 cases), then `resources/lang/tr/enums.php` — which translates only the
**16 record types** — and, failing both, returns the **raw slug**. On a host whose docs cache was
never compiled, the 11 report and analytics types therefore degrade to `customer_statement` and
friends. The tool signals that state as `type_label: null` plus a `label_source` marker; never
echo a slug either way — name the template by its `name` or the Turkish wording the user already
used ("Cari Hesap Ekstresi", never `customer_statement`).

## Reading before writing is mandatory

`print_templates_list` does **not** return `content` (the index select stops at ids, name, type,
flags and routing columns), so every edit or clone starts with `print_templates_get`.

The version stamp is **`content_hash` = sha256 of `content`**, returned by
`print_templates_get` and required by `print_templates_update` as `expected_content_hash`.
`updated_at` is not a version stamp: second resolution, and one default change rewrites it on
every template of that type. A hash mismatch means "Şablon bu arada değişmiş" — re-read first.

An update is no kinder to the definition: omit `definition` and the stored one is kept whole, send
one and it **replaces** the stored content in place. The only way back is the audit trail through
`print_templates_revert`, and it restores `content` alone — an `is_default` flip is written by a
bulk query that leaves no audit row behind.

Deleting reads the same record. `print_templates_delete` refuses while a **blocker** stands: the
template is a customer's `default_print_template_id`, it is a record's own `print_template_id`, or
it is the type's `is_default` and no other template of that type exists. When a default is deleted
and the type does have others, ask **which one takes over** and pass it as **`new_default_id`** —
the handover happens inside the same write, in one transaction. Omit the argument and the delete is
refused, because the alternative is a category left with no default at all.

## Three traps

1. **PUT is not a partial patch** — at the HTTP layer. `name`, `type`, `content`, `is_default`
   and `priority` are required on every write, and what a raw request omits is overwritten.
   `print_templates_update` shields you from that: every field you do not send is filled from
   the stored record (`definition` included). That is precisely why `print_templates_get`
   first is mandatory — the tool can only carry over what it has read.
2. **Never send `max_lines_per_page: null`.** The column is `integer NOT NULL DEFAULT 15`; for
   every type except `invoice`, leave the key out of the payload entirely. `null` is a NOT NULL
   violation and fails with a 500, not a validation message. That column default is not the
   printed limit — an unset or zero value falls back to 100 at print time (`data-paths.md`).
3. **`is_default` is destructive in both directions.** `true` runs `makeDefault()`, flipping every
   other template of that type to `false` and possibly unseating the tenant's live default.
   `false` on the *only* default leaves the type with none and breaks printing for the whole
   category ("Bu rapor için bir şablon tasarlanmamış"). Both directions need explicit approval —
   name the template that loses its default status before you confirm.

## What an update must carry over verbatim

One record column and two definition blocks survive an edit only because they are carried through.
Where the stored record has them and your payload does not, the tool merges them back or refuses
the write — never "fix" a refusal by re-sending a stripped definition.

- **`wildcard_filters`** — the HTTP request class force-merges `[]` when the key is absent, so a
  quiet update **wipes** it with no error. With an empty `condition` the template drops out of the
  conditional-routing query and something else gets printed. With `condition` set the result is
  inverted and worse: the template stays in the query, but an empty filter set makes
  `checkWildcardMatch()` return `true` for **everyone**, so it outranks the correct template for
  every customer the condition admits.
- **`DataSources[].ConnectionProperties`** — the `JSONEMBED` / `endpoint=` blocks on the 8 legacy
  record templates. Never **drop** them on an update; the tool merges the stored ones back, and a
  definition that arrives without them is a definition that tried to throw away someone's sample
  data. **Reading them is a different matter and is encouraged**: on the record path that embedded
  sample is the only place a real row can be found, and it is what a usable `sample_row` is cut
  from (`dialog-flow.md` Step 2A Q2b — mind the literal `\n` escapes inside `jsondata=`). On a
  **create** their absence is normal: a new template starts with `DataSources: [{Name:'DataSource'}]`
  and nothing embedded.
- **`EmbeddedImages`** — the letterhead/logo blob, ~90% of the bytes in the largest templates.
  Dropping it silently is forbidden: either it is carried into the new definition, or the write
  is refused with "antet görselini düşürüyorsunuz".
