# Print Template Dialog Flow

Use this for requests like:

- "Fatura için yeni bir yazdırma şablonu yap"
- "Cari hesap ekstresi çıktısını değiştir"
- "Kaydettiğim analitik sorunun Excel çıktısına şablon istiyorum"

**Iron rule:** no template is produced before the data path has been answered. Guessing is forbidden; if anything stays unclear, ask again instead of assuming. An unanswered question stops the flow.

**Language rule (MCP server instruction):** user-visible messages carry no technical identifiers — "Cari Hesap Ekstresi", not `customer_statement`; "Analitik şablonu", not `analytics_export`; "belge no", not `document_no`. Keys stay inside tool arguments.

## Step 0 — Preconditions (always)

1. Establish tenant access with `session_orientation` + `tenants_list` (`makierp-mcp-session`). Templates write tenant data.
2. Ask whether the user wants to change an existing template or create a new one. Existing → `print_templates_list` → `print_templates_get` (**read the content — mandatory**) → then edit. The list never returns `content`, and `print_templates_update` demands the current `content_hash`: never overwrite a design you have not read. Two readings serve two questions — `fields_only` for field names, `summary_only` → `body_items` for structure (see Step 2A Q2). What you must **not** do is spend a full `get` on a record definition; they run to hundreds of thousands of characters.

   The hash itself does not force the read: the **confirmed** phase of your own `create` / `update` / `revert` also returns a usable `content_hash`, so a second edit in the same chain works without another `get`. Rule: hold that template's current hash → go ahead; do not hold it → `get`. A foreign write in between makes the hash mismatch and the tool refuses — that means "read it again", not "retry".

## Step 0b — Update branch (existing template)

The record already carries a `type` and the type **cannot be changed**, so the data path follows from it — **Step 1 is not asked**. State these three and get an acknowledgement:

1. *"Değişiklik **yerinde yazılır**; eski içerik üzerine geçilir, geri dönüş yalnız denetim kaydından mümkündür."*
2. *"Bu şablon şu an **varsayılan** ise varsayılanlığını kaldırmam bu türde hiç varsayılan bırakmayabilir — o zaman bu tipte yazdırma/dışa aktarım tamamen kırılır."*
3. *"Adını değiştirirsem başka bir şablonla aynı ada düşebilir; aynı türde iki aynı adlı şablon kullanıcıyı şaşırtır ve o türün başlangıç şablonunun geri alınması ada göre çalıştığı için riskli hale gelir."*

The previous `content` can still be restored afterwards with `print_templates_revert`, which reads the audit trail. A change of the default flag leaves **no** audit entry, so it cannot be reverted — say that instead of promising a full undo.

Then go straight to **Step 3**, whose wording becomes "…bir şablonu **değiştireceğim** — değişen alanlar: […]".

## Step 1 — Data path (new templates only, never skipped)

Ask verbatim:

> **Bu şablon neyi basacak?**
> 1. **Bir kaydı** — fatura, irsaliye, sipariş, çek, ambar fişi, muhasebe fişi, cari dekontu, kasa satırı ya da banka dekontu (listede satıra sağ tıklayıp "Yazdır" dediğinizde çıkan çıktı)
> 2. **Parametreli bir raporu** — cari/banka/kasa ekstresi, **muhasebe hesap ekstresi**, muavin defter, mizan, yevmiye, defter-i kebir, hesap listesi ya da maliyet analizi (tarih aralığı vb. soran raporlar)
> 3. **Kaydedilmiş bir analitik sorunun çıktısını** — soruyu Excel/PDF olarak dışa aktarırken kullanılacak düzen
> 4. **Sadece elimdeki örnek veriyle bir taslak** — henüz bir kanala bağlanmayacak

Every answer routes to a branch and carries one channel truth that MUST be said out loud:

- **1 → Step 2A.** *"Bu yolda **yalnızca yazdırma** var — PDF indirme, Excel ve e-posta gönderme bu çıktı türünde bugün mevcut değil."*
- **2 → Step 2B.** *"Bu yolda yazdırma, e-posta ve dışa aktarım (PDF/Excel/CSV/JSON) üçü de var."*
- **3 → Step 2C.** *"Bu yolda yazdırma, e-posta ve dışa aktarım var. Ama şablonun **sayfa düzeni yalnız PDF'te** uygulanır. **Excel'de düzen uygulanmaz**: şablondan yalnız kolon planı ve şablon adı (sayfa/dosya adı) kullanılır; satırlar düz tablo basılır ve şablonun tablosunda olmayan bir alan kolon planında duruyorsa Excel'e o da girer. CSV/JSON'da şablon tümüyle yok sayılır."*
- **4 → Step 2D.** *"Gömülü örnek veri **yalnızca tasarım anında** okunur; gerçek baskıda veriyi her zaman uygulama gönderir. Bu şablonun **otomatik olarak seçilmesi için** 1/2/3'ten bir kanala bağlanması gerekir (kaydedildiği anda kullanıcılar onu 'Özel Yazdır' / 'farklı şablonla PDF' listesinde **görebilir**). Yine de taslak isteyeyim mi?"*
- **Unclear → produce nothing.** Ask again.

## Step 2A — Record printing

**Q1 — record type:** "Hangi kayıt türü?" Sixteen, in three families:

- **Belge (satırlarıyla birlikte basılır):** fatura / irsaliye / sipariş / çek / ambar fişi /
  millileştirme fişi / muhasebe fişi / cari hesap fişi / banka fişi / çek bordrosu /
  maliyet dağıtım fişi / satış provizyon fişi
- **Makbuz (tek satır, tek sayfa):** cari hesap hareketi / banka hesabı hareketi /
  kasa hareketi / muhasebe hesabı hareketi

→ `invoice | way_bill | order | cheque | storage_slip | customs_clearance_slip |
accounting_slip | customer_slip | bank_transaction | cheque_roll | cost_slip |
sale_provision_slip | customer_slip_line | bank_transaction_line | safe_line |
accounting_slip_line`.

**Ask which of the two a movement screen means.** Cari, banka, kasa and muhasebe hareket
ekranları basabilecekleri İKİ şey taşıyor: satırın kendi makbuzu, ve satırın bağlı olduğu
muhasebe fişi. "Kasa satırını bas" çoğu zaman makbuz demektir, ama emin değilsen sor:
*"Hareketin kendi makbuzunu mu, yoksa bağlı olduğu muhasebe fişini mi basacağız?"*

**Maliyet dağıtım fişi ile satış provizyon fişi ayrı türlerdir** ama tek ucu paylaşır; kullanıcı
"maliyet fişi" derse hangisini kastettiğini netleştir.

**Q2 — field source (mandatory; field names may never be invented).** These categories are not registered as report sources, so their schema endpoint answers **404** — there is no machine-readable column catalogue anywhere. Call `print_templates_sources(path='record')` first and eliminate options with its two flags:

- `has_existing_template = false` → option **(a) does not exist**; do not offer it.
- `sample_print_possible_without_template = false` → option **(b) is physically impossible** (that endpoint returns 422 while no template exists — chicken-and-egg); do not offer it. Both flags are false today for `way_bill`, `storage_slip` and `accounting_slip` → fall straight to **(c): produce nothing, stop.**
- Only where `sample_print_possible_without_template = true` (today only `cheque`) can (b) genuinely be asked.

Ask what survives elimination:

> "Bu türde şu anda **hangi alanların** basılabildiğini bilmem gerekiyor. Üç seçenek var: (a) Bu türde mevcut bir şablonunuz varsa onu okuyup alan adlarını oradan alayım — hangisi? (b) Yoksa uygulamada bir kaydı yazdırıp bana dönen alanları verebilir misiniz? (c) İkisi de mümkün değilse **şablon üretmiyorum** — alan adlarını uydurursam çıktı sessizce boş basar."

The only valid sources are an existing template's declared field list or a real print response. Read that list with `print_templates_get(fields_only: true)`: record definitions run from roughly 108.000 to 678.000 characters, and pulling a whole one exhausts the context for a catalogue the summary already carries. An invented field name prints a silently empty cell.

`fields_only` answers "which fields exist"; it carries **no structure**. When the question is a layout one — where the regions sit, which one binds the primary, whether the flowing table is at body level — read `print_templates_get(summary_only: true)` and look at `body_items`. That is how a broken record layout is diagnosed (`skeletons.md`, `troubleshooting.md`). `summary_only` is not the cheap option: it lists **every** declared field, so on a field-rich template it can be larger than `fields_only` and exceed the tool's output limit. When it does, the response is dropped to a file — continue with `jq '.summary.body_items'` instead of re-reading.

**Q2b — a real `sample_row` (record path).** Without one, `print_templates_validate` puts the field-existence gate in `skipped_checks` and its `ok: true` means only "the structure parses". There is no endpoint that hands you a record row on this path (this is also why `print_templates_dry_run` answers `row_count: 0`). Harvest one from a sibling template of the same type:

1. `print_templates_get(id)` → `definition.DataSources[0].ConnectionProperties.ConnectString`.
2. The value is prefixed `jsondata=[…]` and contains literal `\n` escapes — unescape before parsing as JSON, or the parse fails.
3. Trim one record down to the relations you print plus its `lines`, and pass it as `sample_row`.

Then the gate actually runs and `referenced_fields` is resolved against real data.

**Q3 — default:** "Yeni şablon bu türün **varsayılanı** olsun mu? (Evet dersem, aynı türdeki diğer tüm şablonların varsayılanlığı düşer.)" For the **first** template of a type `is_default` defaults to `true`, and that is stated rather than applied silently: *"bu türün ilk şablonu olduğu için varsayılan yapıyorum"*. If the user still insists on `false`, answer plainly *"o zaman bu şablon hiçbir kanaldan basılamaz"* — every channel that gets no explicit template id falls back to the type's default. Two exceptions keep `false`: the draft branch (Step 2D) and the analytics type (Step 2C).

For `invoice` the question widens — purchase and sale invoices are **one single category**, and direction is separated only by a routing condition the agent cannot write in v1:

> *"Bu şablon varsayılan olsun mu? Evet dersem **hem alış hem satış** faturalarının varsayılanı olur ve aynı türdeki diğer şablonların varsayılanlığı düşer. Yalnız birine (ör. sadece satış) uygulanmasını istiyorsanız bunu uygulamadaki şablon ekranından **koşul** ile yapmanız gerekiyor."*

**Q4 (invoice only) — `max_lines_per_page`:** "Bir sayfaya kaç satır sığsın?" Warn that the number is enforced in two places: the invoice print endpoint, **and sale-invoice save validation** — a sale invoice using this template cannot be saved at all once its line count exceeds the value. An empty or zero value falls back to 100. Do not propose needlessly small numbers, and tell the user *"bu sayı yalnız baskıyı değil, bu şablonu kullanan faturanın kaydını da sınırlar"*.

**Q5 — not asked in v1.** Routing rules (condition, wildcard filters, priority) are absent from the create/update tool schema on purpose: a written priority would outrank every existing conditional template (the resolver picks the **lowest** priority), and a condition match overrides both the customer default and the category default. Wildcard filters are not a free filter either — they are a map keyed by customer custom-field ids. On update, existing values are carried over untouched. If the user wants conditional routing: "bunu uygulamadaki şablon ekranından yapmanız gerekiyor."

## Step 2B — Parameterized report

This whole branch runs through tools; the agent issues no raw HTTP. `print_templates_sources` is the only surface it calls.

**Q1 — report:** `print_templates_sources(path='report')` → show the returned **Turkish labels** ("Cari Hesap Ekstresi", "Muavin Defter", …): "Hangi rapor? — [liste]". If the wanted report is missing: "yeni bir rapor türü eklemek şablonla yapılamaz, geliştirme gerektirir."

**Q1b — required params (mandatory, asked BEFORE any sample fetch).** Read the `required: true` entries from the tool's `params` and ask the user for **real values** (customer / bank account / safe / waybill record and/or a date range). Nine of the ten report keys demand at least one required param — the chart-of-accounts list is the only exception — and missing values make the sample endpoints answer 422 `Zorunlu alanlar eksik.` For the accounting account statement the account is **not** a param: it goes into the context bag, and when it is missing you get **no 422, just a silently empty result**. If the user cannot supply values, no sample data can be fetched → **produce nothing** (the report-side twin of Step 2A option (c)).

**Q2 (agent-only):** `print_templates_sources(path='report', key=<key>)` → `columns` (the only valid source of flat field names) + `presentation` + `params`. Never derive template structure from `presentation`; it only configures the preview table and the size gate.

**Q2b (agent-only):** `print_templates_sources(path='report', key, params, context)` → `sample_row` + `nested_collections`. The tool builds these server-side from the source's template rows, not from the flat preview-data endpoint: that endpoint always returns flat rows and carries no nested collection key, and the schema endpoint returns only the flat columns. Reports like Defter-İ Kebir expose a second collection (a summary) that never shows up in the schema.

**Q3 — grouping:** "Bu raporda **gruplama** ister misiniz (ör. her hesap için ayrı başlık ve alt toplam), yoksa düz liste mi?" If the answer is yes, say the Excel consequence in the same breath — grouping lives in the template, so it exists in the PDF only: *"Gruplamayı şablona koyuyorum; Excel/CSV çıktısında hesap başlıkları, devir blokları ve ara toplamlar yer almaz, satırlar tek düz liste olarak iner."* **Q3b — summary table:** "Ana listenin altında ayrı bir **özet/toplam tablosu** ister misiniz?" — ask this one only when Q2b actually reported a second collection.

**Q4 (conditional) — page-boundary rows:** "Her sayfanın altında **ara toplam**, her sayfanın başında **önceki sayfadan devir** satırı ister misiniz?" Ask only when the source emits cumulative fields: the boundary rows read row-embedded `cum_*` values, not aggregates. Seven sources have them (customer / bank account / safe statements, subsidiary ledger, trial balance, journal, general ledger); three do not (accounting account statement, account list, detailed cost analysis). Verify against the `sample_row` keys from Q2b — if absent, do not ask, and if the user insists say *"bu raporun verisinde kümülatif alan yok, sınır satırları boş basar"*. When grouping was chosen and the group header repeats on new pages, the subtotal becomes group-aware: explain it as *"ara toplam yalnız bir hesap sayfa sonunda BÖLÜNDÜĞÜNDE basılır"*.

**Q5 — geometry:** "Sayfa **dikey mi yatay mı**, kaç kolon basacağız?" Propose landscape/wide paper at eight or more columns. **Q6 — default:** same wording as Step 2A Q3.

Also say verbatim: *"Excel/CSV/JSON çıktısında bu şablonun **düzeni** uygulanmaz — kolonlar raporun kendi şemasından gelir; şablondan yalnız **adı** kullanılır (Excel'de sayfa adı olur). Ama **şablon yine de gereklidir**: bu raporun hiç şablonu yoksa Excel/CSV/JSON dışa aktarımı da çalışmaz."*

## Step 2C — Analytics

**Q1 — which question:** "Hangi kaydedilmiş soru için?" — list with `analytics_questions_list` and show the user the **question names**.

**Ad-hoc sub-branch (no saved question):** the analytics preview and export surfaces accept `schema | report | question` as the data source type, so a template can bind to an ad-hoc query over a schema/report source instead of a saved question. The template type stays the analytics one; only where the column keys come from changes (same triple in both branches), and the compatibility gate behaves identically. Offer this branch, or have the user save the question first.

**Q2 (agent-only) — three steps, not one call.** There is no schema endpoint on this path; `print_templates_sources(path='analytics')` only returns the chain below as a note. `analytics_query_preview` requires `data_source_type` **and** `data_source_id` **and** `query` together:

```
analytics_questions_list   → pick the question (show the user NAMES)
analytics_questions_show   → take the question's `query` body verbatim
analytics_query_preview    → data_source_type:'question' + data_source_id:'<id>' + that query
                             → output column keys
```

Saying "the preview refuses a question id" is **wrong** — what it refuses is being called with an id and no `query`.

The backend compatibility gate is **reference-based**: it scans only the field references used in expressions plus native chart column keys. The declared field list is never read, so a declared field no expression uses is not a rejection reason (it only shapes the Excel column plan). Rule: **every field referenced in an expression** must exist among the question's output columns; dotted references also match on their root key.

**Q3 — content:** "Şablonda **tablo** mu, **sayı kartı** mı, ikisi birden mi olsun?" Chart/pivot starter blocks are switched off today — do not offer charts unprompted.

**Q4 — format:** "Bu şablonu **hangi formatta** kullanacaksınız — PDF mi Excel mi?" If the user says CSV or JSON, warn that the template plays no part in those formats.

**Q5 — default:** `false`. In the analytics path the template is always selected by explicit id and the default flag has no observed effect here (open question).

## Step 2D — Draft

A channel-less draft **may be saved** — but never without an explicit warning and approval. The warning is not a chat sentence: it is delivered through the `warnings` array returned by the token-less call in Step 4 and approved with the `confirmation_token`.

Say verbatim:

> "Bu şablonu kaydedebilirim, ama bilmen gereken bir şey var: **şu an bu şablonu basacak bir kanal yok.** Verisini kendi içinden (gömülü örnek veri) ya da verdiğin adresten alan bir şablonu yazdırma/dışa aktarma özelliği henüz eklenmedi — bugün çıktı alınırken veriyi her zaman uygulama gönderiyor. Şablon kayıtlı kalır ve bu özellik eklendiğinde hazır olur. Kaydedeyim mi?"

If approved, save; if not, do not save and suggest which real channel (1/2/3) it could bind to. The `type` must still be a valid enum value, and the draft branch **skips no other gate**:

- **The field-source gate still applies.** In a draft the embedded sample data itself satisfies it — field names come from the keys of the JSON (or endpoint sample) the user supplied, the same source the designer builds its field catalog from. With no sample data, Step 2A Q2 (record types) or the Step 2B/2C schema step (report/analytics types) applies. **With no source at all, no field names are invented and no draft is produced.**
- **`is_default` is forced to `false` here**, even if the user asks otherwise: in a category that has no template at all today, a draft saved as default would instantly take over that type's live printing.
- **Send `channelless_draft: true` on the `print_templates_create` call.** That flag is what makes this branch behave as described: it forces `is_default` to `false` and puts the channel warning into the `warnings` array you read back to the user. Leave it out and the call is treated as an ordinary template — the normal default gate applies and no channel warning returns, so the sentence above becomes a promise the tool never kept.

Do **not** say "bu şablon şu an bir kanala bağlı değil" — it is false. A saved draft is immediately visible and selectable in the row menu's "Özel Yazdır" list and in the "farklı şablonla PDF" picker, regardless of the default flag. Say instead: *"Bu şablon varsayılan yapılmadı, yani otomatik baskıda kullanılmaz; ama kullanıcılar Özel Yazdır / farklı şablonla PDF listesinde onu görüp seçebilir."* Two record types are the exception — `cheque` and `accounting_slip` screens have no "Özel Yazdır" menu, so there a non-default template can only be tried by making it the default temporarily (`data-paths.md`).

## Template name (every branch, mandatory, last question)

> "Bu şablonun adı ne olsun?"

Never invent the name. Check the proposed name with `print_templates_list` and warn the user if the same **name + type** pair already exists — two identically named templates of one type are indistinguishable in every picker the user sees, and the starter-template migration's rollback deletes by exact `type` + `name`, so a duplicate name puts the wrong row in its path. "… (Başlangıç Şablonu)" is **not** a forbidden style: every distributed starter template is named that way. Judge the name on clarity, not on a re-seed danger — a live tenant is never re-seeded (`data-paths.md`, "How a starter template reaches a tenant").

## Step 3 — Final confirmation before writing (every branch)

Present a summary and ask for approval **before** writing:

> "Özetleyeyim: **[şablon adı]** adında, **[TR tip etiketi]** türünde bir şablon oluşturacağım. Kolonlar: [TR etiket listesi]. Sayfa: [A4 dikey/yatay]. Gruplama: [var/yok]. Sayfa sonu ara toplamı: [var/yok]. Varsayılan yapılsın mı: [evet/hayır]. Onaylıyor musunuz?"

On an update, replace "oluşturacağım" with "**değiştireceğim** — değişen alanlar: […]".

## Step 4 — Two-phase write (never skipped)

```
1. Call print_templates_create / print_templates_update WITHOUT a token → nothing is saved;
   {preview, diff, warnings, confirmation_token} comes back.
2. Show preview + diff TO THE USER (Turkish labels; no technical keys).
3. Only after explicit approval, repeat the SAME payload plus the confirmation_token.
```

A token-less response is **not** "saved". Presenting it as "şablonunuz hazır" leaves the user believing in a record that does not exist. Without approval the second call is not made. The gate covers **every** create and **every** update — not only default-flag changes.

**Token lifetime.** The token is single-use and bound to the exact payload it previewed **and** to the session it was issued in. Renewing tenant access between the two phases invalidates it: the second call answers *"Onay kodu geçersiz, süresi dolmuş ya da gönderilen bilgiler önizlenenden farklı"*. That is not a save failure and not a reason to change the payload — repeat the token-less call, show the fresh preview, and confirm with the new token. On a long template job, ask for a generous access window before starting so the two phases stay inside it.
