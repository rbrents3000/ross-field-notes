---
num: "011"
title: "Is anyone running Trace Cost successfully on 8.0?"
tag: "ROSS"
audience: "developer"
source: "group"
date: "2026-09-30"
system: "Ross ERP 8.0"
sp_note: "8.0.2 replaces the add-in with native facilities"
reading_time: "11 min"
excerpt: "Two features, nearly the same name. Cost Trace is a Smart Client add-in with no Ross code to fix. Trace Cost Inquiry, native in 8.0.2, replaces it."
question: "We are having issues with the Trace COST program in 8.0. Checking back to see if anyone is currently using Trace Cost successfully. We upgraded a couple of years ago and have not been able to get it working. We have made some headway (session after session with support), but not enough to be able to use it for the quick turnaround on costs that we were looking for."
restated: "Why does Trace Cost stop working after an upgrade to Ross ERP 8.0, and what is the supported way to get fast actual-versus-standard job costs?"
fix: "There are two different features here with almost the same name, and which one you have decides whether your problem is fixable at all. Cost Trace (PM_I_032, from 7.0 SP4) is a Smart Client add-in — its entire server-side program is a one-line handoff to the client, so there is no Ross program to debug, and its published availability statement stops at 8.0.0. Trace Cost Inquiry (IC_I_037) and Trace Cost Update (IC_U_033) are native facilities added in 8.0.2 that do the same work with no Smart Client involved. Below 8.0.2, the way off the support treadmill is to get to 8.0.2. Already there, the thing most likely stopping you is that Trace Cost Update has never been run — Trace Unit Cost reads 0 until it is, and only on Closed and History jobs. And in both features the color coding is driven by threshold rows you key yourself: gray means unconfigured, not broken."
margin_notes:
  - "two features, one name — settle which you have first ↴"
  - "PM_I_032 reads zero tables"
  - "8.0.2 makes it native → IC_I_037"
  - "gray = no threshold row, not a defect"
  - "Trace Unit Cost stays 0 until IC_U_033 runs"
reviewed: "2026-09-30"
verified: "Verified against Ross 8.0 source, the data dictionary, and the shipped compatibility and change-summary guides"
applies_when:
  - "Cost Trace opens but the job grid is entirely gray, or the cost columns carry no color at all."
  - "You get a message saying Cost Trace can only be launched from Smart Client."
  - "You are on 8.0.2 and the Trace Unit Cost column reads 0 on every job."
  - "Support sessions keep ending without a Ross-side defect anyone can point at."
key_refs:
  - "PM_I_032"
  - "IC_I_037"
  - "IC_U_033"
  - "FACTORY_THRESHOLDS"
  - "PROCESS_SPEC_THRESHOLDS"
  - "PM_M_011"
  - "PM_I_005"
related:
  - "q-001-future-cost-rollup"
  - "q-007-scrap-cost-matches-finished-good"
---

Before anything else: the name is doing real damage here. Ross ships **two** costing-trace features whose names are anagrams of each other, they were built a decade apart on completely different technology, and the one most people find on the menu is the one nobody can fix. Settling which one you're actually looking at is most of the answer.

## Which one do you have?

| | Cost Trace | Trace Cost Inquiry |
|---|---|---|
| Facility | `PM_I_032` | `IC_I_037`, plus `IC_U_033` to update |
| Menu | Product Costing > Cost Trace | Product Costing > Trace Cost |
| Arrived in | 7.0 SP4 | 8.0.2 |
| Runs in | Smart Client add-in only | Native Ross facility |
| Shows you | Actual vs standard, traffic-lighted against thresholds | Actual unit cost by cost level, walking back through prior jobs |
| Server-side program | A one-line launcher | A real Ross inquiry |
| Lives or dies on | Smart Client, IAF 9.1+, and the add-in being registered | Nothing but Ross |

> **in plain terms** — Cost Trace is a separate Windows program that Ross *starts for you*. Trace Cost Inquiry is a Ross screen. When a Windows program that Ross merely starts stops working, Ross has almost nothing to tell you about why.

Aptean's own name for the 8.0.2 work says the quiet part out loud: the change-summary heading is **"Emulate Trace Cost Inquiry."** The new native facility was built to *emulate* the old add-in.

## Why Cost Trace has nothing to fix on the Ross side

This is the whole program behind `PM_I_032`. Not an excerpt — the whole thing, minus the copyright block.

```dml
@program PM_I_COST_TRACE_SHELL
@note THE ENTIRE SERVER-SIDE PROGRAM BEHIND FACILITY PM_I_032
@risk no table is read here — the add-in queries the database on its own
@highlight 4
PROCEDURE_FORM START_NEW_INQUIRY

    BEGIN_BLOCK CALL_CT
        PERFORM "GEMLB:LB_L_CPANEL" CP_TYPE_EXEC_CT
    END_BLOCK

END_FORM
```

The data dictionary agrees: `PM_I_032` has **zero core tables, zero reference tables, zero calls out and zero calls in**. Every other costing inquiry in the module reads dozens of tables. This one reads none, because the add-in opens its own database connection and does the querying itself.

Follow the `PERFORM` one hop and you reach the actual launch:

```dml
@program LB_L_CPANEL
@note WHERE COST TRACE IS ACTUALLY LAUNCHED
@reads system parameter COST_TRACE_NAME
@risk any client that is not the IAF Desktop GUI drops straight to the message
@highlight 8-15
PROCEDURE_FORM CP_TYPE_EXEC_CT

    BEGIN_BLOCK SETUP
        SET/LOCAL DATABASE FIN
    END_BLOCK

    BEGIN_BLOCK CE
        IF( %THIN_CLIENT_TYPE = "IAF DESKTOP GUI" )
            CPANEL &
                /EXECUTE &
                /ADDIN=PARAMETER("COST_TRACE_NAME") &
                /NAME="CostTraceModule.NewInquiry"
        ELSE
            MESSAGE/IDENTIFIER/SEVERITY/BELL P_38928
        END_IF
    END_BLOCK

END_FORM
```

Two things in there are worth writing on a sticky note.

The **guard** is absolute. If `%THIN_CLIENT_TYPE` is anything other than `IAF DESKTOP GUI`, there is no fallback path — you get message `P_38928` and nothing else. Its shipped text, from the service pack that introduced the feature, is exactly: *"Cost Trace can only be launched from Smart Client."*

The **add-in name** is not hard-coded. It comes from a system parameter, shipped `GLOBAL, READ_ONLY` with the value `ErpDesktop`:

```dml
@program V70SP4_PATCH
@note HOW THE MESSAGE AND THE ADD-IN NAME WERE SEEDED IN 7.0 SP4
@writes system messages, system parameters
@highlight 9-12
PROCEDURE_FORM 89018_COST_TRACE

    BEGIN_BLOCK ADD_MESSAGAE
        PERFORM ADD_MODIFY_MESSAGE("A", #REASON,"I","P_38928", "Cost Trace can only be launched from Smart Client.")
    END_BLOCK

    BEGIN_BLOCK ADD_PARAMETER
        BEGIN_SIGNAL_TO_STATUS
            ADD PARAMETER COST_TRACE_NAME &
                /REASON=#REASON &
                /FLAGS=GLOBAL,READ_ONLY,NODATES &
                /VALUE="ErpDesktop"
        END_SIGNAL_TO_STATUS
    END_BLOCK

END_FORM
```

> **in the system** — if a support session has you checking Ross setup, cost categories, job data or spec versions to explain why Cost Trace misbehaves, the session is in the wrong layer. There are exactly three Ross-side things that can be wrong: the client type isn't the IAF Desktop GUI, `COST_TRACE_NAME` doesn't match a registered add-in, or the add-in isn't deployed to the workstation. Everything past that is the add-in and the database it queries directly — and neither of those is Ross application code anyone can patch for you.

That layering also explains something you may have noticed: Trace Express, which is built exactly the same way (`IC_I_007D`, help text *"performs a shell program that uses Smart Client tab to call the Trace Express Inquiry"*), collects defect after defect in the 8.0.1 release notes — data not loading for a lot, an Oracle `ORA-01747`, a crash retrieving trace data. Cost Trace collects none. Across the vendor guides I have, `PM_I_032` appears in exactly one document: the 7.1 change summary that introduced it. It shows up in no release-note defect list at all. Read that how you like, but "quiet" and "healthy" are not the same word.

## The support statement has an upper bound

The compatibility guide is unusually specific, and it is the single most useful sentence in this whole answer:

> Cost Trace is available to all existing Ross customers running a minimum of IAF 9.1 with Smart Client and **Ross ERP 7.0 SP4 through Ross ERP 8.0.0** with Smart Client.

Now compare it to its neighbors on the same page:

| Add-in | Stated availability |
|---|---|
| Trace Express Viewer | Ross ERP 6.3 SP5 **and later versions** |
| Enterprise Viewer | Ross ERP 6.4 SP4 **and later versions** |
| Ross Reporting Services | Ross ERP 6.4 SP2 (open-ended) |
| **Cost Trace** | **7.0 SP4 *through* 8.0.0** |

Cost Trace is the only Smart Client add-in in the list with a **closed upper bound**, and that bound stayed frozen at 8.0.0 through the Q2 2022 edition of the guide — long after 8.0.1 and 8.0.2 had shipped.

> **caution** — the same page carries a second line that matters just as much: Smart Client is *required* for Ross ERP 6.4 through 7.0.x, but is only an **option** for 7.1 and 8.0. So a site that upgraded to 8.0 and modernized its client stack at the same time has quietly removed the one runtime Cost Trace cannot start without. The upgrade didn't break the feature so much as walk away from it.

I'd stop short of calling this a formal desupport notice — Aptean never published one that I've seen. But a bounded availability statement that never moved, no defect history after the introducing release, and a native replacement shipped two point releases later all point the same direction.

## If the grid is gray, it isn't broken

This one catches people, and it's worth ruling out before you book another session. Cost Trace is not a trace in the Trace Express sense — it doesn't discover anything. It is **threshold-driven variance coloring**, and the thresholds are data you have to key yourself.

Five threshold types, stored with these codes:

| Code | Threshold type |
|---|---|
| `01` | Material Cost Variance |
| `02` | Labor Cost Variance |
| `03` | Machine Cost Variance |
| `11` | Unit Cost Variance |
| `12` | Yield Percent |

Each row carries four bands — `TH_LOW_LIMIT`, `TH_LOW_LIMIT_WARNING`, `TH_HIGH_LIMIT_WARNING`, `TH_HIGH_LIMIT` — and those four numbers are the entire traffic light:

| Color | What it means |
|---|---|
| Green | The value falls between the low warning and the high warning. |
| Yellow | The value sits in a warning band — between low warning and low, or between high warning and high. |
| Red | The value is below the low limit or above the high limit. |
| Gray | No threshold has been defined. |

**Gray is the default state of an unconfigured system**, and it is indistinguishable at a glance from a screen that isn't working. A site that turned the feature on, never keyed thresholds, and saw a colorless grid has been looking at correct output the whole time.

Where the rows live:

| Table | Keyed by | Maintained in |
|---|---|---|
| `FACTORY_THRESHOLDS` | Company + Factory + Threshold Type | Maintain Factories (`PM_M_011`) |
| `PROCESS_SPEC_THRESHOLDS` | Company + Factory + Process Spec + Threshold Type | Create Process Spec (`PM_M_023`) |

Factory-level rows apply to a job by default; spec-level rows override them when the job's process specification has its own. Neither table carries a foreign key in or out, so nothing warns you and nothing cascades — an empty table is a perfectly valid state that produces a perfectly useless screen.

- [ ] Query `FACTORY_THRESHOLDS` for your company and factory. Empty means the grid can only ever be gray.
- [ ] Check `PROCESS_SPEC_THRESHOLDS` for the specs on the jobs you're testing — a spec-level row silently overrides the factory row.
- [ ] Confirm all four band columns are populated. They default to `0`, and a row of zeroes is not a threshold.
- [ ] Confirm Cost Category Groups (`PM_M_016`) look right — `0` Unallocated, `1` Material, `2` Labor, `3` Machine. Descriptions are editable; the groups themselves can't be added or deleted.

That's a ten-minute check, and it is worth doing before your next call.

## What 8.0.2 actually gives you

Four related pieces, all native:

```cards
IC_I_037 | Trace Cost Inquiry | Shows the actual unit cost of a job, organized by cost level, and walks back to prior jobs. Intermediate and final jobs together.
IC_U_033 | Trace Cost Update | Writes the trace cost of the final output product onto that product's lot number. This is the step that makes the number persist.
PM_I_033 | Job Inquiry | Gains a right-click Trace Cost option and a new Trace Unit Cost column.
PM_I_012 | Costed Process Spec Inquiry | Gains a Percent of Component column — the sum of final unit cost percent from the preceding cost level.
```

And here is the gotcha that speaks directly to wanting a quick turnaround on costs:

> **caution** — the Trace Unit Cost column populates **only for jobs in Closed or History status**, and it reads **0** until Trace Cost Update has been run. The "quick" part of quick turnaround is a batch update somebody has to schedule. If nobody has ever run `IC_U_033`, every job on the screen shows zero and the feature looks broken on day one.

So the honest expectation-setting on 8.0.2: it gives you traced actual cost per lot, it does it without Smart Client, and it is a **closed-job, post-update** number. It is not a live look at an in-flight job.

## The path that needs none of this

Worth knowing, because it's already installed and it survives every client decision you make.

The threshold rows you key in `PM_M_011` and `PM_M_023` are not private to the add-in. Three native inquiries read them directly:

| Facility | Name |
|---|---|
| `PM_I_003` | Job Master |
| `PM_I_005` | Job Stage Cost |
| `PM_I_029` | Job Status |

Since 7.0 SP4 each of those carries **Job Cost Summary** and **Job Stage Cost Summary** options, showing the same cost thresholds, total standard, total actual and total variance — by cost category group, and by job stage and cost category group. Same thresholds, same variance math, ordinary Ross screens. (One placement note from the 8.0 guide: between Process Spec Cost `PM_I_009` and Job Stage Cost `PM_I_005`, only Job Stage Cost offers the two summary options.)

Alongside them, **Job Exception Inquiry** (`PM_I_034`) compares actual inputs and outputs against planned and lists the jobs and lines that violate the factory or process-spec limits — material inputs against the Material Cost threshold, labor against Labor, machine against Machine, miscellaneous against Material, and material outputs against Unit Cost. Job campaigns are excluded.

> **rule of thumb** — if what you need is "show me the jobs whose cost went sideways, fast," `PM_I_034` plus the two summary windows gets you there today, on any client, with no add-in registration and no batch update. The graphical supply-chain picture is what Cost Trace adds — and that picture is the part that costs you a Smart Client dependency.

## So, to answer the question as asked

Nobody should expect Cost Trace to come good on 8.0 through support sessions. It's an add-in whose entire Ross-side surface is a single `PERFORM`, whose published support ceiling is 8.0.0, and which has no post-introduction defect history to suggest anyone is still working on it. The sessions aren't failing because the problem is hard; they're failing because there's almost nothing on the Ross side to change.

```steps
1 | Settle which feature you have
where: Product Costing menu
do: Check whether the entry is Cost Trace or Trace Cost, and confirm your exact point release — 8.0.0, 8.0.1 and 8.0.2 are three different answers here.
sys: Facility PM_I_032 is the add-in; IC_I_037 is the 8.0.2 native inquiry.

2 | Rule out the free explanation
where: PM_M_011 / PM_M_023
do: Confirm threshold rows actually exist for the factory and specs you're testing. An all-gray grid with no threshold rows is working exactly as designed.
sys: FACTORY_THRESHOLDS and PROCESS_SPEC_THRESHOLDS, keyed by threshold type 01/02/03/11/12, four band columns each.

3 | Stand up the fallback that already works
where: PM_I_034, PM_I_005, PM_I_029, PM_I_003
do: Put Job Exception Inquiry and the Job Cost Summary / Job Stage Cost Summary windows in front of the people who need the numbers. Same thresholds, no add-in.
note: This is the step that gets you results this week rather than next quarter.

4 | Make 8.0.2 the target
where: Upgrade planning
do: Ask specifically about Trace Cost Inquiry (IC_I_037) and Trace Cost Update (IC_U_033), by facility ID. They are the supported replacement and they need no Smart Client.
sys: Introduced in 8.0.2 under the change-summary heading "Emulate Trace Cost Inquiry."

5 | Schedule the update before you judge it
where: IC_U_033
do: Run Trace Cost Update on a schedule that matches how fast you need the numbers. Until it runs, Trace Unit Cost is 0 everywhere.
note: And only Closed and History jobs will ever carry a value.
```

> **field note** — worth asking your account team one blunt question in writing: *is Cost Trace (PM_I_032) supported on our exact release, and if so, against which Smart Client and IAF versions?* The published statement stops at 8.0.0. Getting the answer on the record either unblocks the sessions or ends them, and both outcomes beat another month of neither.

One aside, since this came up alongside a question about tracing for audits: the slow-and-inconsistent Trace Express complaints are a **separate** problem with a separate fix list. The 8.0.1 release notes carry several Trace Express corrections, and 8.0.1 also introduces a **Trace Recall** capability. If audit-window traceability is the real deadline, that's a different conversation and a more promising one — the fixes exist there.
