---
name: makierp-payment-plans
description: Reads, defines and tests MakiERP payment plans (ödeme planı) through MCP — read and create only. Use when the user wants to see how due dates are decided, define standard or per-customer payment terms, or check which plan an invoice would get. Existing plans can never be edited or deleted through MCP.
---

# MakiERP Payment Plans

## Quick Start

1. Use `makierp-mcp-session` first if tenant orientation or access is not established. Payment plans decide the due date of every future invoice — confirm the tenant.
2. Four tools: `payment_plans_list` and `payment_plans_show` read, `payment_plans_create` defines, `payment_plans_simulate` tests. There is **no update and no delete** — corrections happen in the ERP UI.
3. Always `payment_plans_list` before defining anything. A plan is never chosen on its own merits; it has to outrank every other plan, so you cannot write a correct one without seeing the others.
4. `payment_plans_create` is **confirmation-gated**: a call without `confirmation_token` saves nothing and returns per-plan previews plus a token. Show the preview — due dates, instalment amounts, conflicts — and only after an explicit yes repeat the call with the same plans and the token.
5. The create tool needs a dedicated permission (`payment_plans.mcpCreate`) on top of the regular create permission. If the call is rejected as unauthorized, tell the user to ask their administrator for the "Yapay Zeka ile Oluşturma (MCP)" permit on payment plans.

## How A Plan Gets Picked

In order, first hit wins: the document's own plan → a plan whose **condition** holds → the **default** plan → the customer card's plan.

- **A plan with a condition beats every plan without one, whatever the priority values say.** This is the rule that surprises people. Among plans of the same kind, the lowest `priority` number wins; ties fall to insertion order, so always set a priority.
- An exception plan must therefore have a *lower* priority number than the standard plan it overrides.
- A condition that never mentions the document type covers purchase invoices too, and will displace the purchasing plans. Scope it with `slip_type.code` when you mean sales only.
- The tool refuses `is_default`. Promoting a default re-routes everything that was falling through, so it stays a UI decision.

## Writing Conditions

A condition is a list of `{attribute, operator, value}` objects joined by the strings `"and"` / `"or"`, with nested arrays for grouping (`and` binds tighter than `or`). Attributes are paths on the document — `customer.code`, `position.code` (the salesperson), `slip_type.code`, `warehouse.code`, `has_special_vat_base`. Operators: `equals`, `not_equals`, `in`, `not_in`, `greater_than`, `less_than`, `contains`, `starts_with`, `ends_with`, `between`, `empty` and their negations.

The same customer can carry different terms under different salespeople, so exception conditions normally pair both: `customer.code in [...]` **and** `position.code equals ...`.

Group companies are not reachable from a condition — only the customer's own fields are loaded. Expand the group members into the code list yourself, using `makierp-erp-data` over the customers source ("Üst Cari" / "Alt Cariler").

## Writing Lines

Each line sets one instalment: `sequence`, `type` (`cash`, `cheque`, `credit_card`, `installment`, `dbs`, `bank_transfer`, `no_action`), a `formula`, and `day`/`month`/`year` offsets. A signed value like `"+30"` shifts from the invoice date; an unsigned `"15"` pins that day of the month.

`formula` splits the invoice total over `P1` net total, `P2` VAT base, `P3` VAT, `P4` the amount still unallocated, `P6` expenses — **`P5` is reserved, never use it**. Functions: `min`, `max`, `round` (precision argument required), `ceil`, `flor` (floor), `absolute`. The last line is normally just `P4`, otherwise the remainder becomes a separate instalment on the invoice date — the preview reports this as `unallocated_amount`.

`applicable_weekdays` (1=Monday..7=Sunday) pushes an instalment landing outside those days forward; the preview reports the shift, so a "30 day" plan visibly becoming 32 is caught before saving.

## Verifying

After creating, run `payment_plans_simulate` with real cases — a customer, the salesperson, optionally the document type and a net total. Check three things: the intended documents land on the new plan, everyone else's do not, and the purchase side still resolves to its own plan. `also_matched` shows which conditions also held but were outranked, which is how you diagnose a plan that lost.

## Handover

- Use `makierp-erp-data` to resolve customer codes, salesperson positions and group hierarchies before writing conditions.
- Use `makierp-schema-docs` before assuming a field meaning or an enum value.
