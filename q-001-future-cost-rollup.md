---
num: "001"
title: "My future cost rollup errors out — 'product not found in Process Costing Spec.' What did I miss?"
tag: "ROSS"
audience: "developer"
source: "group"
date: "2026-07-16"
system: "Ross ERP 8.0"
reading_time: "7 min"
excerpt: "A future rollup only reads the exploded quantity tables — it never rebuilds them. Run the cost type 02 (Preliminary) standard rollup first, then the future rollup."
question: "I'm generating future standard costs from our weighted-average costs by running a rollup on cost type 20. The audit report errors with 'product not found in Process Costing Spec,' and a couple of the make components come back with zero-price warnings even though I keyed a future price on the raw material. What am I missing?"
restated: "Why does a future cost rollup (cost type 20) fail with 'product not found in Process Costing Spec' errors and zero-price warnings on make components, and what sequence of rollups produces correct future costs?"
fix: "A future rollup only reads the exploded quantity tables (PROCESS_SPEC_INPUT_QTYS / PROCESS_SPEC_OUTPUT_QTYS) — it never builds them. Only a cost type 02 (Preliminary) rollup runs in Standard mode and populates them. Run the 02 pass first, including the lower-level make components, then run the cost type 20 future rollup. The zero-price warnings are the same cause one level down: let the rollup derive make-item costs rather than hand-keying them."
margin_notes:
  - "02 first, then 20 ↴"
  - "future mode only READS the qty tables →"
  - "let Make items derive, don't hand-key"
reviewed: "2026-07-29"
verified: "Verified against Ross 8.0 source"
applies_when:
  - "The audit report errors with `P_01405` — 'Product … not found in Process Costing Spec' — as an error, not a warning."
  - "One or more Make components roll up to zero and raise `P_25095` — 'Price is zero for:'."
  - "A standard cost rollup behaves; only the future rollup misbehaves."
---

Your read of it is right, and it turns out to be the key to both problems. A future rollup isn't a from-scratch rollup — it leans on a snapshot that a *different* pass is supposed to build first. Skip that pass and the future rollup has nothing to stand on.

If the checklist above matches, nothing is wrong with your data or your future prices — it's the *order* the rollups ran in.

## Two rollups, two different jobs

The trap is that these look like the same program with a different number typed in. They're not — the cost type silently decides which of two very different modes you get.

| | Cost type 02 — Preliminary | Cost type 20 — Future |
| --- | --- | --- |
| Runs in | **Standard** mode (`#MODE = "S"`) | **Future** mode (`#MODE = "F"`) |
| Formula explosion | Rebuilds it from the recipe | Reuses whatever's already there |
| Quantity tables | Deletes and repopulates them | **Only reads** them |
| Prices applied | Current | Your future prices, at the valid-from date |
| When to run it | **First** | After the 02 pass |

Read that "only reads them" row twice — it's the whole answer. A future rollup assumes the exploded quantity tables are already sitting there, correct and current. Something else has to have built them.

## The mode is chosen by the cost type

You don't pick Standard or Future with a checkbox. The rollup derives it from the cost type you enter — and only cost type 02 gets Standard mode.

```dml
@program PM_U_STD_COST_ROLLUP
@note MODE IS CHOSEN FROM THE COST TYPE — 02 = STANDARD, ANYTHING ELSE = FUTURE
@reads cost_type
@risk only cost type 02 rebuilds the explosion; every other cost type reuses it
@highlight 2-3
IF (#COST_TYPE = #COST_TYPE_PRELIM)   ! cost type 02 only
    #MODE = "S"                       ! Standard — rebuilds the explosion
ELSE
    #MODE = "F"                       ! Future — reuses the explosion
END_IF
```

So cost type 20 ran in Future mode — as it's meant to. Both of your runs were future rollups. And here's why it's so easy to miss on the way in — the submission screen itself:

```screen
title: Standard/Future Cost Rollup
program: PM_U_STD_COST_ROLLUP
facility: PM_U_047
context: Company = 01
form:
  All Products = No
  Warehouse {lov} = WH01
  Product {lov} = FG100
  Cost Type {lov} = 02  Preliminary
  Valid From Date {ro} = 01-Jul-2026
  Preliminary Cost Update? = No
```

There's a **Cost Type** field — and nowhere on the screen is there a *Mode* switch. Type `02` and you're in Standard mode; type anything else and you're in Future mode, with no on-screen tell that the choice was made for you. (The one giveaway is subtle: the **Preliminary Cost Update?** row only appears *because* `02` put the run in Standard mode — enter a future cost type and it isn't there.)

## Future mode never rebuilds the explosion

Inside the incremental-cost step, every "explode the formula and rebuild the quantity tables" statement is wrapped in `IF (#MODE <> "F")`. In future mode the whole block is skipped — the exploded snapshot is neither cleared nor rebuilt.

```dml
@program PM_L_INCREMENTAL_COSTS
@note EVERY "REBUILD THE QTY TABLES" STEP IS GUARDED BY IF (#MODE <> "F")
@writes process_spec_input_qtys, process_spec_output_qtys
@risk in future mode none of this runs — the snapshot is left exactly as it was
@highlight 1,6
IF (#MODE <> "F")
    PERFORM DELETE_EXISTING_INPUT_QTYS
    PERFORM DELETE_EXISTING_OUTPUT_QTYS
    ...
    ADD TO PROCESS_SPEC_OUTPUT_QTYS    ! never runs in future mode
END_IF
```

The unit-cost step then tries to look the product up as an *output* of the spec. When the qty tables were never populated — because no Preliminary pass ran — the row isn't there, and it logs an error:

```dml
@program PM_L_CALC_PRELIM_UNIT_COST
@note UNIT-COST STEP FINDS THE PRODUCT AS A SPEC OUTPUT — A MISSING ROW IS AN ERROR
@reads process_spec_output_qtys
@risk empty table (no prior 02 pass) → P_01405 on the audit report
@highlight 4-6
FIND IN PROCESS_SPEC_OUTPUT_QTYS
    /WITH=OUTPUT_PRODUCT=#PART_CODE

IF (%STATUS = %FAILURE)
    #LOG_SEVERITY = MESSAGE("ERROR")
    #LOG_TEXT     = MESSAGE("P_01405", #PART_CODE)   ! "Product !AS not found in Process Costing Spec"
END_IF
```

That `P_01405` is the line on your audit report. The table it reads is written *only* by a non-future rollup — so a future-only run has nothing to read.

> **field note** — "future" is a misleading name here. It doesn't mean "cost the future recipe from scratch." It means "reuse the explosion I already have, and reprice it at the future date." Building the explosion is the Preliminary rollup's job. Picture cost type 02 as assembling the skeleton and cost type 20 as putting next quarter's prices on it.

## The fix — run the Preliminary pass first

1. Run the rollup on **cost type 02 (Preliminary)** first, for the same warehouse and finished good *and* its lower-level make components — or simply "All Products." This is the Standard-mode pass that explodes the formulas and populates the quantity tables. Let it finish clean.
2. Then run the rollup on **cost type 20 (Future)**. It reuses that explosion and recomputes costs with your future prices at the cost type's valid-from date.
3. Confirm the 02 pass is against the **current active process-spec versions**. If a spec version changed after the last 02 rollup, the qty tables are stale and future mode won't see the new version — re-run 02 to refresh them.

## The zero-price warnings are the same cause, one level down

The make components that rolled up to zero are almost certainly the same defect one BOM level lower. For a Make input that carries its own costing spec, the rollup pulls the component's cost from its *spec-specific* future cost record in `PRODUCT_WH_FUTURE_COSTS` — keyed by that component's own costing factory / spec / version **and** the effective date. It does **not** read the value you typed into Product Future Cost Maintenance.

Future Cost Maintenance stores its value under a **blank** factory / spec / version key, so the parent rollup never looks at it for a Make item. With the qty tables missing, the make components rolled up to zero — and when the finished good referenced them, the rollup saw a zero cost and raised `P_25095`, *"Price is zero for:"*.

Include those components in the 02 pass so they actually roll up — the program works bottom-up by BOM level — and their future costs will compute from their own recipes plus your future raw-material price. The warnings should clear.

> **rule of thumb** — for Make items, let the rollup *derive* the future cost; don't hand-key it. Product Future Cost Maintenance is for **Buy** items — which is exactly right for the one raw material you set.

## Where this comes from

- `PM_U_STD_COST_ROLLUP` — mode chosen by cost type: 02 → Standard, anything else → Future.
- `PM_L_INCREMENTAL_COSTS` — future mode skips deleting and rebuilding the input/output qty tables; the `ADD TO PROCESS_SPEC_OUTPUT_QTYS` runs only when `#MODE <> "F"`.
- `PM_L_CALC_PRELIM_UNIT_COST` — a failed `FIND` in `PROCESS_SPEC_OUTPUT_QTYS` logs error `P_01405`.
- `IC_M_FUTURE_COST` — Future Cost Maintenance writes the blank-key record the make-input path never reads.

Two caveats worth stating plainly. This was read from Ross 8.0 source; the mechanism is standard, but on a different patch level the exact line numbers may shift — the `IF (#MODE <> "F")` guards are the stable landmark. And the link between the zero-price warnings and the missing qty tables is an inference joining two things the code proves separately: the clean test is whether the `P_25095` warnings disappear after the cost type 02 pass. If one component still warns, its own sub-components — or its spec-specific future record at that effective date — are the next place to look.
