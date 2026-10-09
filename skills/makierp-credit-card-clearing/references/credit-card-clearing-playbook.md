# Credit Card Clearing Playbook

Drive a real POS/merchant statement through `credit_card_clearings_create`, day by day.

## 1. Read The Statement First

A POS account statement mixes several row kinds. Only one of them is a clearing row.

| Row kind | Typical wording | What to do |
|---|---|---|
| **Block release** | `İşyeri no:…, Tutar : 7525 Komisyon : 0, Bloke No : …, BT: 02/03/2026, ÇT:31/03/2026` | **This is the clearing row.** |
| Commission / BSMV | `KOMİSYON`, `BSMV`, `POS KOMİSYON` | Not a clearing row — an ordinary bank expense. |
| POS rental / service | `POS YAZILIM/DONANIM/BAKIM ÜCRETİ`, `FİZİKİ POS KİRASI` | Not a clearing row. |
| Sweep / transfer out | `… No.lu Hesaba, Şube Virman`, `… Hesaba Para Yatırma` | Not a clearing row — a transfer between the firm's own accounts. |
| Refund / chargeback | `İADE`, `RET`, `CHARGEBACK`, or a negative release | A clearing row, but of a refund group; the tool handles the polarity. |

Everything in the "not a clearing row" rows belongs to `makierp-bank-transactions`. Never fold them into a clearing.

**Two dates per release row — send both.** `BT` is the sale/blockage date, `ÇT` is the release date; the statement usually posts the row a day after `ÇT`. The **statement day (`document_date`) is the day the money reached the account**, not `BT`. Put `BT` on the row as `sale_date`.

Sending `sale_date` is not optional in practice: without it the tool matches on amount alone inside a 45-day window, and on a busy POS account the same amount can appear dozens of times, so nearly every row comes back ambiguous. With it, the pool shrinks to that one sale day. If nothing on that day fits, the tool reports it and lists the near misses from the surrounding window — it never quietly matches something from another day.

**Amounts.** Use the figure that actually landed in the account. If the statement shows gross and commission separately, the released amount is what the ERP will match; commission is its own row.

## 2. Resolve The Account And The Card Type

- `bank_account_id` — the POS account the statement belongs to. Resolve it over the bank-accounts data source; match on the account number in the statement header. **A wrong account matches nothing** rather than clearing someone else's blocks, so a completely empty match report usually means the wrong account.
- `clearing_type` — `customer_credit_card` when customers paid with their cards (the normal POS case), `company_credit_card` when the firm's own card was used. Slip type `07` versus `08`; each accepts only its own two payment transaction types, so a mixed statement needs two separate calls.

## 3. Matching Rules

The tool matches a statement amount against a **group total**: every payment transaction sharing one `transaction_group_uuid`, i.e. one card payment with all its installments. A 3-installment 3.000 TL sale is one group of 3.000 TL, never three rows of 1.000 TL.

Resolution order per row:

1. **Pinned** — if you send `group_uuids`, those groups are used, after checking they are open, on this account, of the right card type, and not already claimed earlier in the same call. Send several uuids when the bank paid several groups in one statement line; their totals must sum to `amount`.
2. **Amount on the sale day** — when the row carries `sale_date`, only groups dated that day are considered. Nothing there → `ambiguous` listing the window's near misses; never a silent match from another day. A `sale_date` later than the statement day is rejected outright: a block cannot be released before the sale.
3. **Exact amount** — no `sale_date`: exactly one open group in the window with that total.
4. **Older than the window** — if the only groups with that total predate `max_age_days`, the row comes back `ambiguous` with a note, never matched silently. Send the sale date, pin it, or raise `max_age_days`. Groups dated *after* the statement day are never offered.
5. **Whole sale day** — when the row carries `sale_date` and its amount equals the sum of **every** open group sold that day, all of them are cleared together. This is the normal outcome on a POS account: the bank bundles a day into ~10 release rows while the ERP holds 30–250 individual payments, so no single row ever matches a single group, but the day totals agree exactly. Send one row per sale day with the day's total.
6. **Combined total** — one statement line covering several groups is searched for, but only on a small pool and only when there is exactly one possible combination. More than one combination is `ambiguous`.
7. Otherwise `unmatched`.

### Which shape is this account?

Before sending anything, compare **day totals**: sum the statement's release rows per `BT` day and compare with the open groups the ERP holds for that day. If the totals agree but the row counts differ wildly, this is a bundled account — go straight to one-row-per-day. If they disagree, the missing amount is a data problem in the ERP, and no matching mode will paper over it: report it.

A group claimed by one row can never be claimed by another, in the same day or a later one.

## 4. Drive It Day By Day

1. Lay the whole plan in front of the user first: dates, row counts, day totals, and anything you already know you will skip.
2. Get an explicit go-ahead.
3. Per day: call without a token, read the report, then call again with the same input and the token.
4. **Stop and ask** if a day's preview total disagrees with the statement day total, or if any row is `ambiguous`. Never confirm a day you cannot explain.

You may send several days in one call; the tool still writes one slip per day. Driving one day per call keeps the review small and makes a mistake cheap to contain.

## 5. Resolving An Ambiguity

The candidate list gives you customer code and name, sale date, day gap, installment count and total, closest sale first. Show it to the user in those terms and let them choose — then re-send only that row with `group_uuids: ["<the chosen uuid>"]`. Do not pick "the closest one" on the user's behalf: two identically-priced sales to different customers are indistinguishable to the tool by design.

## 6. Report Back

After the last day, give the user:

- per day: slip number, group count, total, and whether it matched the statement day total;
- every skipped row with its reason — fee row, unmatched, ambiguous, already cleared, or not visible to this user;
- the remaining open blocked balance, if they want to know what is still outstanding (`makierp-erp-data`).

## 7. Hard Boundaries

- **No update, no delete.** There is no MCP tool for either, and the generic bank transaction tools refuse credit card clearing slips. A wrong clearing must be undone in the ERP wizard UI — say so plainly.
- **Never re-send a cleared group.** A group already linked to a clearing line is closed; sending it again returns a diagnostic and creates nothing.
- **Never invent a group.** If the sale is not in the ERP, the clearing cannot be recorded — the missing sale has to be entered first.

## 8. Permission Errors

The tool needs all three, and refuses on the narrowest missing one first:

| Permit (Turkish label) | Subject |
|---|---|
| Kredi Kartı Sihirbazı | Ödeme Hareketleri |
| Oluşturma | Banka Hareketleri |
| Yapay Zeka ile Kredi Kartı Bloke Kapama (MCP) | Banka Hareketleri |

Row-level rules on the wizard permit also narrow which groups you can see at all: a group the user is not entitled to is reported as not found, not as forbidden.
