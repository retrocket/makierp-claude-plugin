# How MakiERP Actually Prices a Line

Read this before explaining why a price resolved the way it did. Several of these
rules contradict what the screens imply.

## Price card selection

A document line loads every card that survives the filter, then takes the first one.

**The filter** — a card has to be active, match the direction, and satisfy all of:

- the product, and either the exact unit or a convertible sibling unit;
- the customer code pattern, where empty, `*` and NULL all mean "everyone";
- the customer's custom-field filters, which are ANDed one by one;
- the warehouse, where NULL means "every warehouse";
- the date, where the window is honoured **only when both ends are set or both are
  empty**. A card with just a start date never reaches a document at all.

**The order** — `priority` ascending, and nothing else. An empty priority sorts last.

That has three consequences worth stating out loud:

1. **A customer-specific card does not automatically beat a general one.** Both pass
   the filter; only priority separates them. To make the specific card win it needs a
   *lower* priority number.
2. **Cards sharing a priority resolve unpredictably.** There is no tie-breaker, so the
   winner is whatever row order the database returns. The same document can price
   differently on different days.
3. **Currency is never filtered on.** A card in EUR and a card in TRY compete directly,
   and whichever wins decides the currency the line is priced in.

Three columns look like they narrow a card but are never consulted: payment plan,
organizational unit and project.

## Unit conversion

A convertible card can serve a different unit in the same unit group. The price scales
**inversely** to the quantity — a case of 12 at 120 becomes 10 per piece.

Conversion factors are looked up in order: the product's own unit row, then the unit-set
template. If neither carries usable factors the conversion silently proceeds at 1:1, and
the case price lands on a piece line unchanged. That is why
`convertible_without_factors` is a critical finding rather than a cosmetic one.

## VAT

`vat_inc` is taken from the winning card, not from the product. VAT is stripped with the
product's own rate, chosen by direction and whether the document is a return.

## Campaigns

Campaigns are matched on the customer code, payment plan code, slip no, document no and
description — all wildcards — plus an exact direction match and the date window. The
same both-ends-or-neither rule applies to the window. Warehouse and quantity are not
columns; they are expressed inside the campaign's own condition formula.

**Every matching campaign is applied.** Priority only decides the order, and the order is
deterministic: priority, then campaign id, then placement, then line id.

**They compound rather than add.** Each campaign recalculates against the base the
previous one left behind, so two 10% campaigns come to 19%.

A campaign's formula produces an **amount**, not a percentage; the percentage shown on
the discount line is derived afterwards. Campaigns cannot override a unit price — they
can only take an amount off, or add a gift line priced as a 100% discount.

## Settings that do not take effect

These are stored, shown in the UI and documented, but the engine never reads them:

- **`exclusive`** — a campaign marked exclusive still stacks with every other match.
- **`customer_application_count`** — the per-customer usage limit never reaches the
  query, so the campaign runs without limit.
- **`customer_trading_group_code`** — not part of the matching filter.

When an audit reports these, frame them as "this setting has no effect", not as bad data.

## Where the tools get their numbers

- Realised cost comes from the outbound unit cost recorded on the invoice line.
- Realised discounts are separate child lines hanging off the product line by placement:
  product `3`, its discounts `3.1`, `3.2`. Slip-wide discounts have no parent and are
  reported as general.
- The defined price a realised line started from is the card recorded on the line, which
  is why a line priced by hand has none.
