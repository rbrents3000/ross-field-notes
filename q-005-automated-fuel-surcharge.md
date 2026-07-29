---
num: "005"
title: "Can we automate a fuel surcharge with standard Ross functionality?"
tag: "ROSS"
audience: "developer"
source: "group"
date: "2026-07-29"
system: "Ross ERP 8.0"
reading_time: "8 min"
excerpt: "There's no 'fuel surcharge' object in Ross. You model it as a Miscellaneous Adjustment 'increase' code (a discrete charge line) or a Trade Promotions 'price addition' (an automatic uplift folded into the price). Standard Ross does the surcharge arithmetic — what it never does is re-index the rate to a published diesel price. That last mile is a small scheduled rate refresh."
question: "Fuel cost is high and at some point, we have to consider a fuel surcharge. Has anyone implemented an automated solution for this? We've discussed it many times through the years and have yet to come up with a way to utilize 'standard' functionality to handle that."
restated: "Does standard Ross Order Management have a way to add a fuel surcharge to customer orders automatically — ideally indexed to fuel prices — using vanilla functionality rather than a custom build?"
fix: "There is no 'fuel surcharge' object in Ross. Two standard levers carry it. A Miscellaneous Adjustment code flagged as an increase (percent, flat, or per-unit) gives you a separately-stated charge line — but it has to be attached per order and its rate is a static number. A Trade Promotions 'price addition' auto-applies an effective-dated percentage by rule — but it folds into the unit price and needs the TP module licensed. Neither re-indexes the rate to the DOE/EIA diesel price. The realistic standard answer is a Misc Adjustment percent code for the charge plus a small scheduled job that rewrites its one rate field on your cadence. The surcharge math is 100% standard; the only real 'automation' left is auto-attaching the code and auto-refreshing its rate."
reviewed: "2026-07-29"
verified: "Verified against Ross 8.0 source"
key_refs:
  - "MISCELLANEOUS_ADJUSTMENTS"
  - "SALES_ORDER_MISC_ADJS"
  - "TP_PRICINGS"
  - "sop_l_maint_misc_adjustments"
related:
  - "q-004-blanket-sales-orders"
applies_when:
  - "You want fuel to show as its own line on the invoice, not buried in the unit price."
  - "You've gone looking for a 'surcharge' object in Order Management and come up empty."
  - "You want the rate to follow the diesel index without re-keying every order."
margin_notes:
  - "the math is the easy part ↴"
  - "charge line vs. price uplift →"
  - "nothing indexes to diesel on its own"
---

Everyone who asks this expects to find a *fuel surcharge* feature — a switch you flip, a table you point at the DOE number, and orders start carrying the charge. That feature doesn't exist in Ross, and it never has. But that's not the same as "you can't do it with standard functionality." The trick is to stop looking for one object and see the request for what it actually is: three separate jobs, two of which vanilla already does, and one that no order-entry ERP does on its own.

> **in plain terms** — a fuel surcharge is just "add a bit extra, calculated the same way every time, and let it move with fuel prices." Ross can add the bit and calculate it. What Ross won't do by itself is *watch the fuel price and change the rate for you* — that part is a five-minute update someone (or a small job) makes on a schedule.

## 01 · The request is really three jobs

Break "automated fuel surcharge" into what it's actually asking for, and it stops being mysterious:

| The job | What it means | Does standard Ross do it? |
| --- | --- | --- |
| **Calculate the charge** | A percent, a flat amount, or a per-unit rate applied to the order | **Yes** — two different vanilla mechanisms |
| **Attach it automatically** | The charge lands on orders without someone keying it each time | **Partly** — one lever does, one doesn't |
| **Index the rate to fuel** | The rate follows a published diesel price on a cadence | **No** — not natively, anywhere |

The reason this has "never quite worked with standard functionality" is that no single Ross object does all three. You get the first for free, you get the second from *one* of the two levers below, and the third is always a small piece of glue. Pick your lever based on which trade-off you can live with.

| | Miscellaneous Adjustment code | TP "Price Addition" |
| --- | --- | --- |
| **Shows as its own charge line** | Yes — its own bucket + GL account | No — folded into the unit price |
| **Applied automatically by rule** | No — attached per order | Yes — the pricing engine applies it |
| **Effective-dated** | Rate is static in the code master | Yes — validity window per version |
| **Percent / flat / per-unit** | All three | Percent or amount |
| **Extra module needed** | None — base SOP | Trade Promotions, licensed + on |

## 02 · Option A — Miscellaneous Adjustments (the discrete charge line)

This is the closest thing vanilla has to an "accessorial charge," and it's the honest home for a fuel surcharge that you want customers to *see* as a fuel surcharge. A sales order, invoice, and credit note each carry a **`MISCELLANEOUS`** money bucket alongside goods and freight, and the amount in it comes from adjustment codes you define once and attach to the document.

You define the code in Miscellaneous Adjustment Code maintenance; the header row lands in **`MISCELLANEOUS_ADJUSTMENTS`** (keyed by company + `ADJUSTMENT_CODE`) and the two fields that make it a fuel surcharge are:

- **`ADJUSTMENT_INC_OR_DEC = 1`** — an *increase* (a charge, `+`). Anything else is an allowance (a discount, `−`). This one flag is the difference between a surcharge and a discount.
- **`ADJUSTMENT_CALCULATION_TYPE`** — how it's figured:

```cards
1 | Percent | A percentage of the line/order value — `MISC_ADJUSTMENT_PERCENT`. The usual choice for fuel.
2 | Flat value | A fixed amount per document — `MISC_ADJUSTMENT_VALUE`.
3 | Value per UOM | A per-unit rate × quantity, in `ADJUSTMENT_UNIT` (e.g. per cwt, per case).
```

The master also carries a default rate and **min/max guard-rails** (`MINIMUM_ADJUSTMENT_PERCENT` / `MAXIMUM_ADJUSTMENT_PERCENT`), a GL account (`ADJUSTMENT_ACCOUNT`) so the surcharge posts to its own revenue line, and a description flag. When the code is attached to a document, the calc lives in `sop_l_maint_misc_adjustments` — and it's exactly the arithmetic you'd write by hand:

```dml
@program SOP_L_MAINT_MISC_ADJUSTMENTS.DML
@note BLOCK CALC_TOTAL_VALUE · A PERCENT SURCHARGE ON THE LINE VALUE
@reads miscellaneous_adjustments  (adjustment_inc_or_dec, adjustment_calculation_type)
@writes sales_misc_adjustments_vt.misc_adjustment_total_value
@risk the rate is a static field — nothing here looks at a fuel index
@highlight 10-13
#LINE_VALUE_DISCOUNTED = #LINE_VALUE
IF (SALES_MISC_ADJUSTMENTS_VT(ADJUSTMENT_INC_OR_DEC) <> 1)
    #TOTAL_VALUE_MULTIPLIER = -1.0          ! an allowance
ELSE
    #TOTAL_VALUE_MULTIPLIER = 1.0           ! a charge — this is us
END_IF

IF (SALES_MISC_ADJUSTMENTS_VT(ADJUSTMENT_CALCULATION_TYPE) = '1')
    ! Percent
    #TOTAL_VALUE = (#LINE_VALUE_DISCOUNTED / 100.00) &
                * SALES_MISC_ADJUSTMENTS_VT(MISC_ADJUSTMENT_PERCENT)
    SALES_MISC_ADJUSTMENTS_VT(MISC_ADJUSTMENT_TOTAL_VALUE) &
                = ROUND((#TOTAL_VALUE * #TOTAL_VALUE_MULTIPLIER), #CURRENCY_DPS)
END_IF
```

The stored charge rides on **`SALES_ORDER_MISC_ADJS`** (line `0` = a header-level charge on the whole order; `> 0` = a specific line), and it copies forward order → invoice → credit note, staying adjustable until it's invoiced. It rolls into the header total in its own term, so the customer sees goods, freight, and the surcharge as distinct amounts.

```steps
1 | Define the surcharge code
where: Miscellaneous Adjustment Code maintenance
do: create a code — e.g. "FUEL" — as a percent (type 1), increase (`+`), with your current rate as the default and a sane min/max. Point it at a fuel-surcharge revenue account.
sys: writes `MISCELLANEOUS_ADJUSTMENTS` — `ADJUSTMENT_INC_OR_DEC = 1`, `ADJUSTMENT_CALCULATION_TYPE = 1`, `ADJUSTMENT_DEFAULT_PERCENT`, `ADJUSTMENT_ACCOUNT`.

2 | Attach it to the order
where: sop_l_maint_misc_adjustments
do: on the order (header or line), add the FUEL code. The rate defaults in; the amount calculates.
sys: writes `SALES_ORDER_MISC_ADJS`; `CALC_TOTAL_VALUE` figures `MISC_ADJUSTMENT_TOTAL_VALUE`.

3 | It flows to the invoice
do: nothing manual — the adjustment copies onto the invoice and posts to its own GL account.
sys: carried by `sop_l_load_misc_adj_vt`; posted by `sop_l_load_misc_gl_post`.
```

> **caution** — this is a lookup-code, not a rule. Vanilla has **no engine that auto-attaches** a Misc Adjustment by customer, date, or fuel index — someone (or a default-in customization at order create) has to put the code on the order. And the rate is a plain field in the code master: standard Ross will never change it for you. Great for a *visible, GL-clean charge*; not, by itself, "automatic."

## 03 · Option B — Trade Promotions "Price Addition" (the automatic uplift)

If your priority is *"it just applies itself, I don't want order entry touching it,"* the vanilla lever is the **Advanced Pricing** engine in Trade Promotions. Its pricing records aren't only discounts — one record type is a **price addition**, i.e. an uplift the pricing engine adds during order entry, the same way it resolves contract and promotion prices.

You set it up as a `TP_PRICINGS` record with the addition type, and it gives you the two things Option A lacks:

- **Rule-driven** — the pricing engine (`GEMTP:TP_L_ADVANCED_PRICINGS`, reached from `sop_s_l_order_lines` during price resolution) applies it automatically by scope: all products, or a commodity class / brand / product class. No per-order keying.
- **Effective-dated** — each record has a `VALID_FROM_DATE` / `VALID_TO_DATE` window and an active/inactive status, and the rate is a percent (`TP_PRICING_CALCULATION_TYPE = 1` → `TP_PRICING_PERCENT`). To change the rate for a new period you activate a new dated version.

> **caution** — two real catches. **(1)** It needs the **Trade Promotions module** licensed and switched on (`AR_CONTROLS(TP_PROMOTIONS_IN_USE)`), so it's not "base SOP." **(2)** The addition is **folded into the line's unit price** — it makes the price bigger; it does *not* emit a separate "Fuel Surcharge" line on the order or invoice. If the customer needs to see the surcharge stated separately, this isn't the one.

> **field note** — this is the fork in the road worth deciding up front: **a visible charge line (Option A) or hands-off application (Option B) — vanilla gives you one or the other, not both.** Most shippers who want fuel *shown* pick the Misc Adjustment and solve the "attach it" part with a small default; those who just want margin protected and don't care about the line pick the TP addition.

## 04 · The part that genuinely isn't standard — indexing to diesel

Here's the piece the question gets exactly right, and it's worth saying plainly: **no version of this is fully "standard," because the automation people actually want is the rate following fuel prices — and nothing in Ross reads a fuel index.** This isn't a Ross gap so much as an ERP-industry one; the shops that do it "automatically" all bolt a rating layer on (TMS rate matrices, accessorial masters), not core order entry.

The good news is the indexing logic is trivial and well-standardized in freight. Two common models:

- **Percentage matrix (the LTL model).** A table maps the **DOE/EIA weekly retail diesel price** into bands, each band a surcharge percent. Diesel between \$4.00–\$4.09 → 34%, \$4.10–\$4.19 → 34.5%, and so on. You publish the table; each week the current diesel price picks the row.
- **Peg formula (the truckload model).** Pick a base ("peg") diesel price where the surcharge is zero, then add a set amount for every increment above it — e.g. one cent per mile for each 6-cent rise over the peg.

Either way the output is a single number: this period's surcharge rate. Feeding that number into Ross is one field write:

- Option A: update `ADJUSTMENT_DEFAULT_PERCENT` (or the per-UOM rate) on the FUEL code.
- Option B: activate a new effective-dated `TP_PRICINGS` version with the new percent.

```checklist
[x] Calculate the charge {master-data} — standard Ross, either lever above.
[x] Attach / apply it {master-data} — Option A per-order (or a default), Option B by rule.
[ ] Index the rate to diesel {developer} — not standard; a scheduled refresh writes one field.
where: a small job (or a person) on your chosen cadence
```

> **rule of thumb** — the surcharge itself is a config exercise; the "automation" reduces to **one recurring rate update**. If a person spends five minutes a week changing one number, you've solved it with pure standard functionality. If you want even that hands-off, the *only* thing worth customizing is a thin scheduled job that reads the published diesel price and rewrites that one field — a few lines against a public number, not a surcharge subsystem. Build the small thing, not the big one.

## 05 · So which do I reach for?

- **Customers must see fuel as a stated line; you can live with attaching a code (or defaulting it in):** Miscellaneous Adjustment, percent, increase. This is the answer most of the time.
- **You want it applied automatically and don't need a visible line, and you already run Trade Promotions:** a TP price addition, effective-dated.
- **Either way, you want the rate to track diesel:** accept that this one step lives outside vanilla — do it by hand on a cadence, or wrap it in a tiny scheduled refresh.

The reason the "standard functionality" conversation keeps stalling is that it's been framed as *find the fuel-surcharge feature*. There isn't one — but there are two perfectly standard ways to carry the charge, and the only genuinely non-standard bit is a single number that changes on a schedule. That's a much smaller thing to build than the subsystem everyone's been picturing.

## 06 · Where this comes from

- `sop_l_maint_misc_adjustments` — Misc Adjustment entry + the `CALC_TOTAL_VALUE` percent/flat/per-UOM math; `MISCELLANEOUS_ADJUSTMENTS` is the code master, `SALES_ORDER_MISC_ADJS` the stored charge.
- `sop_l_load_misc_adj_vt` / `sop_l_load_misc_gl_post` — copy the adjustment order → invoice → credit note and post it to its own GL account.
- `sop_s_l_order_lines` → `GEMTP:TP_L_ADVANCED_PRICINGS` — where the Trade Promotions engine resolves a `TP_PRICINGS` price addition into the line price during order entry (gated by `AR_CONTROLS(TP_PROMOTIONS_IN_USE)`).
- The diesel index itself — the DOE/EIA weekly retail on-highway diesel price — is the public number the rate tracks; Ross reads none of it, which is why the refresh is the one piece you own.
