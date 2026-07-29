---
num: "005"
title: "Can I output scrap on a job and have it carry the same cost as the finished good?"
tag: "ROSS"
audience: "developer"
source: "group"
date: "2026-07-29"
system: "Ross ERP 8.0.1"
reading_time: "8 min"
excerpt: "Pointing an item's costing spec at another product won't roll a cost onto it — the rollup only posts to a spec's principal product. Give the scrap item its own one-line costing recipe with the finished good as the input, then output it as a credited by-product."
question: "Has anyone figured out how to output scrap on a recipe/job and have it get the same cost as the finished good? We started with a manufactured part #scrap7000 and were hoping the costing spec would be 7000, but the cost won't roll up. Without making this a buy item and loading in the costs each month, is there a way in Ross to have this pick up the same cost as the #1 item? We're looking to get a true cost on scrap and output it to the job, then delete the outputs. We're on 8.0.1."
restated: "In Ross process manufacturing, how can a scrap/by-product output on a job automatically carry the same cost as the finished good, without maintaining the scrap item as a purchased item with monthly cost loads?"
fix: "Ross never posts a standard cost to an item just because you point its costing spec at another product — the rollup only posts to the principal product of a spec. To make a scrap item carry the finished good's cost with no buy-item maintenance, give the scrap item its OWN one-line costing recipe with the finished good as the sole input (qty 1, same UOM). The standard rollup then inherits the finished good's rolled cost every run. Output it on the job as a credited by-product so it is valued at that same standard — but remember the credit comes back out of the finished good's cost, which is exactly why the 'output then delete' move works as a capture-and-reverse."
margin_notes:
  - "rollup posts to the principal product only ↴"
  - "give scrap its OWN costing spec →"
  - "credited by-product = a negative input at its own cost"
  - "cost is conserved — scrap's value leaves the FG"
reviewed: "2026-07-29"
verified: "Verified against Ross 8.0 source"
applies_when:
  - "You set the scrap item up as a Make item and pointed its costing spec at the finished good, but the rollup posts no cost to it."
  - "You want the scrap valued at the finished good's cost without keying it as a Buy item every period."
  - "You plan to output the scrap on the job to capture a value, then reverse it."
key_refs:
  - "PART_MASTER_M"
  - "PROCESS_SPECIFICATIONS"
  - "RECIPE_LINES"
  - "PM_L_INCREMENTAL_COSTS"
  - "PRODUCT_WH_PRELIM_COSTS"
related:
  - "q-001-future-cost-rollup"
---

Your instinct is a good one, and it's very close to how Ross wants you to do this — the piece that's missing is *where* the costing spec has to live. Pointing the scrap item's costing spec at the finished good tells the rollup "cost this item using the finished good's recipe," and in that recipe the scrap isn't the thing being costed. So nothing lands on it.

There's a clean, self-maintaining way to get the result you're after. It comes in two parts: making the scrap item's *standard* equal the finished good, and then deciding how it behaves when you output it on the job.

## Why the costing spec alone didn't roll up

A product's cost rollup doesn't read some global "which product do I resemble" setting. It reads three fields off the item's **own** part master — the costing factory, the costing process spec, and its version — and costs the item *as the principal product of that spec*.

```dml
@program PM_L_INCREMENTAL_COSTS
@note THE ROLLUP COSTS A PART USING THE SPEC NAMED ON ITS OWN PART MASTER
@reads part_master_m
@risk point the costing spec at another product's recipe and the rollup costs THAT product, not yours
@highlight 3-5
FIND IN PART_MASTER_M /LOCK=NONE /WITH=PART_CODE=#RPART
IF (%STATUS = %SUCCESS)
    #COST_FACTORY = PART_MASTER_M(COSTING_FACTORY)
    #COST_SPEC    = PART_MASTER_M(COSTING_PROCESS_SPEC)
    #COST_VERSION = PART_MASTER_M(COSTING_PROCESS_SPEC_VERSION)
END_IF
```

When you set `#scrap7000`'s costing spec to the finished good's spec, the rollup dutifully costs *that spec* — and a spec only posts standard costs to its **principal product** and any **co-products** (outputs flagged "final product to inventory"). Your scrap line isn't one of those, so the rollup accumulates a cost and hands it to the finished good, and `#scrap7000` comes out with nothing. That's the "won't roll up."

> **in plain terms** — the costing spec is "which recipe do I get costed by," not "which product's price do I copy." A recipe costs the product it's built to make. To get a cost on the scrap item, the scrap item needs to be the star of *some* recipe.

## The rule underneath all of this: cost is conserved

Before the fix, here's the idea that makes the whole thing click. A job accumulates one pool of cost — ingredients plus resources. Ross then hands that pool **out** to the outputs. It never invents extra cost for an output; whatever you assign to the scrap is subtracted from what's left for the finished good.

> **caution** — you cannot make the scrap carry the finished good's full cost *and* leave the finished good's cost unchanged **on the same job**. Any value the scrap picks up comes out of the finished good. That constraint is the real reason your plan ends in "…then delete the outputs" — you're capturing a valuation, then handing the cost back.

## How Ross values each kind of output

Every output line on a recipe (`RECIPE_LINE_TYPE = 5`) is one of five things, and the type decides how — or whether — it gets cost:

```dml
@program PM_I_COSTED_PROCESS_SPEC
@note THE FIVE KINDS OF OUTPUT A RECIPE LINE CAN BE
@highlight 2-6
IF (#RECIPE_LINE_TYPE = 5)
    ! 1. Final product to Inventory      (principal product / co-product)
    ! 2. Final product to Customer
    ! 3. Interstage product to next stage
    ! 4. Credited Byproduct              (credits the job at its OWN cost)
    ! 5. Non-Credited Byproduct          (affects yield only, no cost)
END_IF
```

| Output kind | How it's costed | Effect on the finished good |
| --- | --- | --- |
| Principal product | Gets the pool, less any co-product / by-product shares | — |
| **Co-product** (final to inventory) | Share of the pool: `pool × (COST_ALLOCATION_PERCENT / 100) / output_qty` | **Dilutes it** — the pool is split |
| **Credited by-product** | Credited at the **by-product item's own cost** (posts as a negative input) | Reduces it by the credit |
| Non-credited by-product | No cost — yield only, display unit cost only | None |

Two takeaways. A **co-product** *can* be given a rolled standard by the shared spec — but it's an allocation, so to make its unit cost equal the finished good you'd have to split by quantity, which drops the finished good's unit cost. A **credited by-product** is the one whose value comes from *its own item cost*, not from the job pool — and that's the hook we want.

## The fix — give the scrap item its own costing recipe

Make `#scrap7000` the principal product of a tiny recipe whose only input is the finished good. Then its standard rolls up *to* the finished good automatically, forever.

```steps
1 | Build a one-line costing recipe for the scrap item
where: pm_m_recipes
do: create a recipe whose principal product is `#scrap7000`, with a single ingredient line — the finished good `#7000`, quantity 1, same cost UOM, 100% yield.
sys: writes `PROCESS_SPECIFICATIONS` with `PRINCIPAL_PRODUCT = #scrap7000` and one `RECIPE_LINES` input (`RECIPE_LINE_TYPE = "1"`).

2 | Point the scrap item's costing spec at that recipe
where: item / part master costing tab
do: set the scrap item's costing factory, costing process spec, and version to the recipe you just made.
sys: sets `PART_MASTER_M(COSTING_PROCESS_SPEC)` + `COSTING_FACTORY` + `COSTING_PROCESS_SPEC_VERSION`.

3 | Run the standard cost rollup
where: ic_u_product_cost_update
do: roll up the scrap item (and let the finished good roll first, bottom-up by BOM level).
sys: a Make input contributes its own rolled standard, so the scrap inherits the finished good's cost.
```

Why it works: when the rollup processes the scrap recipe, its single input is a Make item, so it pulls that input's **rolled standard cost** straight from `PRODUCT_WH_PRELIM_COSTS` —

```dml
@program PM_L_INCREMENTAL_COSTS
@note A MAKE INPUT CONTRIBUTES ITS OWN ROLLED STANDARD COST
@reads product_wh_prelim_costs
@risk let the finished good roll first — bottom-up by BOM level — or the input reads stale
@highlight 2
IF (#MAKE_BUY = PARAMETER("MAKE_FLAG"))
    #STD_COST = SCH:PRODUCT_WH_PRELIM_COSTS(STANDARD_COST)   ! scrap inherits the FG's rolled cost
END_IF
```

— so `#scrap7000`'s standard equals `#7000`'s standard on every rollup, with **no buy-item record and no monthly cost entry.** When the finished good's cost changes, the next rollup carries it straight through.

> **field note** — this is the standard "pass-through" costing recipe: a Make item whose recipe is one unit of another item. It's the supported way to say "cost me the same as that item," and it's a live link, not a one-time copy.

## Outputting it on the job — and why "then delete" behaves

Now add `#scrap7000` as an output on the finished good's *production* recipe and flag it **Credit job with by-product = Yes**. A credited by-product is run through the very same costing routine as an ingredient — as a negative input, valued at its own standard (which you've just made equal to the finished good):

```dml
@program PM_L_INCREMENTAL_COSTS
@note A CREDITED BY-PRODUCT IS COSTED THROUGH THE SAME ROUTINE AS AN INGREDIENT
@reads recipe_lines
@risk its credit is its OWN item cost, not a share of the job pool
@highlight 2-3
IF (RECIPE_LINES(DESTINATION_TYPE) <> (PARAMETER("SOURCE_TYPE_STAGE")) AND &
    RECIPE_LINES(CREDIT_JOB_WITH_BYPRODUCT) = PARAMETER("LANGUAGE_YES"))
    PERFORM UPDATE_MATERIAL_COSTS      ! the same routine ingredient lines use
END_IF
```

Report the scrap output on the job and it's valued at that "true cost." Because it credits (reduces) the job's cost pool, the finished good on *that* job costs a little less — the conservation rule again. Reverse or remove the output and the cost flows back to the finished good. That's why your "output it, then delete the outputs" sequence works: it's a **capture-and-reverse** to read a true scrap value without permanently moving cost off the finished good. Job outputs are reversible in `pm_u_reverse_job_partial_close` and editable in `pm_m_amend_jobs` (`JOB_OUTPUTS`).

- [ ] Want the scrap to *stay* in inventory at the finished good's value? Don't reverse it — let the credited by-product output stand. The scrap sits in stock at its standard (= the finished good) and the finished good absorbs the credit.
- [ ] Want only a report figure, no lasting cost effect? Output, read it, reverse it.

## There's no "copy another item's cost" switch

Worth saying plainly, because it's the thing people go looking for: vanilla Ross has **no** "reference product cost," "like item," or "copy cost from another product" feature at the item level. The only cost-copy logic in the rollup copies values **between cost types for the same product** (e.g. Preliminary → Standard), never between two products. The manual alternative is exactly the one you're trying to avoid — a Buy item with costs keyed each period in Future/Standard Cost Maintenance. The pass-through costing recipe above is the supported way to get a **live** equal-cost link instead.

## Where this comes from

- `PART_MASTER_M(COSTING_PROCESS_SPEC / COSTING_FACTORY / COSTING_PROCESS_SPEC_VERSION)` — the per-item "costing spec" the rollup reads to decide which recipe costs the item.
- `PROCESS_SPECIFICATIONS(PRINCIPAL_PRODUCT)` — a spec posts a standard cost to its principal product (and co-products), not to arbitrary outputs.
- `PM_L_INCREMENTAL_COSTS` — costs ingredients and credited by-products through `UPDATE_MATERIAL_COSTS`; a Make input pulls its rolled standard from `PRODUCT_WH_PRELIM_COSTS`.
- `PM_I_COSTED_PROCESS_SPEC` — enumerates the five output kinds and how each is valued; a credited by-product shows as a negative input, `COST_ALLOCATION_PERCENT` forced to 0.
- `RECIPE_LINES(CREDIT_JOB_WITH_BYPRODUCT / COST_ALLOCATION_PERCENT / FINAL_PRODUCT_FLAG)` — the flags that classify an output.
- `IC_U_PRODUCT_COST_UPDATE` — the standard cost rollup driver.

A couple of honest caveats. This was read from Ross 8.0 source; the mechanism is standard and unchanged through 8.0.1, though exact line numbers shift by patch level — the stable landmarks are the `COSTING_PROCESS_SPEC` read and the `CREDIT_JOB_WITH_BYPRODUCT` branch. And if you run an actual/average cost type rather than a rolled standard, the same structure holds but the input cost is read from the warehouse average/last cost instead of the preliminary-cost table — tell me which cost type you actually roll into and I'll trace that exact path.
