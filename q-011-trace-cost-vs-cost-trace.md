---
num: "011"
title: "Is anyone running Trace Cost successfully on 8.0?"
tag: "ROSS"
audience: "developer"
source: "group"
date: "2026-09-30"
system: "Ross ERP 8.0"
sp_note: "Trace Cost is 8.0.2 vocabulary — below that the menu says something else"
reading_time: "11 min"
excerpt: "Trace Cost arrived in 8.0.2. Trace Unit Cost reads 0 until Trace Cost Update runs, and only on closed jobs — usually that, not a defect, is why costs aren't quick."
question: "We are having issues with the Trace COST program in 8.0. Checking back to see if anyone is currently using Trace Cost successfully. We upgraded a couple of years ago and have not been able to get it working. We have made some headway (session after session with support), but not enough to be able to use it for the quick turnaround on costs that we were looking for."
restated: "Why does Trace Cost not deliver fast actual job costs on Ross ERP 8.0, and what does it take to get it working?"
fix: "Trace Cost Inquiry (IC_I_037) and Trace Cost Update (IC_U_033) arrived in 8.0.2, and the single most common reason they look broken is that nobody runs the update. Trace Unit Cost reads 0 until IC_U_033 has been run, and it populates only for jobs in Closed or History status — so on a fresh install every job shows zero and the feature appears dead on day one. It is a closed-job, post-batch number by design: the 'quick turnaround' is a batch you schedule, not a live read on an in-flight job. If your Product Costing menu says Cost Trace rather than Trace Cost, you are below 8.0.2 and looking at a different feature entirely — a Smart Client add-in whose entire Ross-side program is a single PERFORM, with no Ross code anyone can fix and a published support ceiling of 8.0.0. And in both features the color coding is driven by threshold rows you key yourself: gray means unconfigured, not broken."
margin_notes:
  - "Trace Unit Cost stays 0 until IC_U_033 runs ↴"
  - "Closed and History jobs only"
  - "the menu name dates your release"
  - "gray = no threshold row, not a defect"
  - "below 8.0.2 there is no Ross code to fix"
reviewed: "2026-09-30"
verified: "Verified against Ross 8.0 source, the data dictionary, and the shipped compatibility and change-summary guides"
applies_when:
  - "Trace Unit Cost reads 0 on every job in Job Inquiry."
  - "Trace Cost returns nothing for jobs that are still open or in progress."
  - "Your Product Costing menu says Cost Trace, not Trace Cost — and the grid opens entirely gray."
  - "Support sessions keep ending without a Ross-side defect anyone can point at."
key_refs:
  - "IC_I_037"
  - "IC_U_033"
  - "PM_I_033"
  - "PM_I_032"
  - "FACTORY_THRESHOLDS"
  - "PROCESS_SPEC_THRESHOLDS"
  - "PM_I_005"
related:
  - "q-001-future-cost-rollup"
  - "q-007-scrap-cost-matches-finished-good"
---

Start with the name, because it dates your release more precisely than anything else in the question.

## The name is a version stamp

Ross has shipped two costing-trace features whose names are near-anagrams of each other, built a decade apart on completely different technology. Which name your menu uses tells you which one you have:

| Menu reads | Facility | Arrived | What it is |
|---|---|---|---|
| Product Costing > **Trace Cost** | `IC_I_037`, plus `IC_U_033` | **8.0.2** | A native Ross inquiry. No add-in, no Smart Client. |
| Product Costing > **Cost Trace** | `PM_I_032` | 7.0 SP4 | A Smart Client add-in that Ross merely launches. |

That isn't a hair-split. Across every vendor guide I have — compatibility guides, change summaries and release comparisons spanning 2018 through 2022 — the phrase **"Trace Cost" appears in exactly two documents, and both of them are 8.0.2 documents.** Every other reference, for a decade, says "Cost Trace."

> **rule of thumb** — if you're calling it Trace Cost, you're almost certainly on 8.0.2 looking at `IC_I_037`. If your menu actually reads Cost Trace, you're below 8.0.2 and the next section doesn't apply to you — skip to *What you have below 8.0.2*.

Aptean's own heading for the 8.0.2 work says what happened: the change summary calls it **"Emulate Trace Cost Inquiry."** The native facility was built to *emulate* the older add-in, and the rename came with it.

## On 8.0.2, here's why it won't turn costs around

Four pieces arrived together:

```cards
IC_I_037 | Trace Cost Inquiry | Shows the actual unit cost of a job, organized by cost level, and walks back to prior jobs. Intermediate and final jobs together.
IC_U_033 | Trace Cost Update | Writes the trace cost of the final output product onto that product's lot number. This is the step that makes the number persist.
PM_I_033 | Job Inquiry | Gains a right-click Trace Cost option and a new Trace Unit Cost column.
PM_I_012 | Costed Process Spec Inquiry | Gains a Percent of Component column — the sum of final unit cost percent from the preceding cost level.
```

And here is the sentence from the change summary that explains most of the frustration, emphasis mine:

> The Jobs Inquiry grid shows a new column called Trace Unit Cost, that shows the trace unit costs **only for the jobs with statuses as Closed and History**. **If the trace cost is not updated in the Trace Cost Update facility, this column shows a value of 0.**

Two independent gates, and a fresh install fails both:

| Gate | What it means |
|---|---|
| `IC_U_033` has never run | Every Trace Unit Cost reads **0**. Not blank, not an error — zero, which looks exactly like a broken feature. |
| Job status isn't Closed or History | Open and in-progress jobs carry no trace cost at all, by design. |

> **caution** — this is the expectation mismatch that kills the business case. Trace Cost is a **closed-job, post-batch** number. The "quick turnaround" is a batch somebody schedules, not a live read on a job that's still running. If the goal is costs on a job that closed this morning, `IC_U_033` has to have run since it closed.

So before another support session, the question worth answering internally is simply: **has anyone ever run Trace Cost Update, and on what schedule?** If the answer is no or nobody knows, that alone explains a screen full of zeroes, and no amount of support time will change it.

- [ ] Confirm `IC_U_033` exists on the Product Costing menu — if it doesn't, you're not on 8.0.2.
- [ ] Run it once manually against a known-closed job, then re-open `IC_I_037` on that job.
- [ ] If the number appears, the fix is scheduling, not support.
- [ ] If it's still 0, check the job is genuinely Closed or History, not just finished on the floor.

## What you have below 8.0.2

If your menu reads Cost Trace, this is what's behind it — and it's the reason support sessions go nowhere. Here is the *entire* server-side program for `PM_I_032`, minus the copyright block:

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

One hop further is the actual launch:

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

The guard is absolute — anything other than `IAF DESKTOP GUI` gets message `P_38928` and nothing else. Its shipped text, from the service pack that introduced the feature, is exactly *"Cost Trace can only be launched from Smart Client."* The add-in name isn't hard-coded either; it comes from a `GLOBAL, READ_ONLY` system parameter `COST_TRACE_NAME`, shipped as `ErpDesktop`.

> **in the system** — there are exactly three Ross-side things that can be wrong here: the client type isn't the IAF Desktop GUI, `COST_TRACE_NAME` doesn't match a registered add-in, or the add-in isn't deployed to the workstation. Everything past that is the add-in and the database it queries directly, and neither is Ross application code anyone can patch for you. If a session has you checking cost categories, job data or spec versions to explain Cost Trace, the session is in the wrong layer.

The compatibility guide adds a bound worth knowing:

> Cost Trace is available to all existing Ross customers running a minimum of IAF 9.1 with Smart Client and **Ross ERP 7.0 SP4 through Ross ERP 8.0.0** with Smart Client.

Compare that to its neighbors on the same page — Trace Express Viewer reads "6.3 SP5 **and later versions**," Enterprise Viewer "6.4 SP4 **and later versions**." Cost Trace is the only Smart Client add-in in the list with a **closed upper bound**, and it stayed frozen at 8.0.0 through the Q2 2022 edition, long after 8.0.1 and 8.0.2 shipped. The same page also notes Smart Client is *required* on 6.4–7.0.x but only an **option** on 7.1 and 8.0 — so a site that modernized its client stack during the 8.0 upgrade quietly removed the one runtime this feature can't start without.

I'd stop short of calling that a formal desupport notice; Aptean never published one I've seen. But a bounded statement that never moved, no defect history after the introducing release, and a native replacement two point releases later all point the same way. Getting to 8.0.2 is the answer, and then the previous section applies.

## Either way: gray means unconfigured

This one catches people on both sides of 8.0.2, and it's worth ruling out before booking anything. The color coding is not a health indicator — it's **threshold-driven variance coloring**, and the thresholds are data you key yourself.

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

**Gray is the default state of an unconfigured system**, and at a glance it is indistinguishable from a screen that isn't working. A site that turned the feature on, never keyed thresholds, and saw a colorless grid has been looking at correct output the whole time.

| Table | Keyed by | Maintained in |
|---|---|---|
| `FACTORY_THRESHOLDS` | Company + Factory + Threshold Type | Maintain Factories (`PM_M_011`) |
| `PROCESS_SPEC_THRESHOLDS` | Company + Factory + Process Spec + Threshold Type | Create Process Spec (`PM_M_023`) |

Factory-level rows apply by default; spec-level rows override them when the job's process specification has its own. Neither table carries a foreign key in or out, so nothing warns you and nothing cascades — an empty table is a perfectly valid state producing a perfectly useless screen.

- [ ] Query `FACTORY_THRESHOLDS` for your company and factory. Empty means the grid can only ever be gray.
- [ ] Check `PROCESS_SPEC_THRESHOLDS` for the specs on the jobs you're testing — a spec-level row silently overrides the factory row.
- [ ] Confirm all four band columns are populated. They default to `0`, and a row of zeroes is not a threshold.
- [ ] Confirm Cost Category Groups (`PM_M_016`) look right — `0` Unallocated, `1` Material, `2` Labor, `3` Machine.

## The path that needs none of this

Worth knowing, because it's already installed and it survives every client and release decision you make.

The threshold rows you key in `PM_M_011` and `PM_M_023` aren't private to either trace feature. Three native inquiries read them directly:

| Facility | Name |
|---|---|
| `PM_I_003` | Job Master |
| `PM_I_005` | Job Stage Cost |
| `PM_I_029` | Job Status |

Since 7.0 SP4 each carries **Job Cost Summary** and **Job Stage Cost Summary** options showing the same thresholds, total standard, total actual and total variance — by cost category group, and by job stage and cost category group. (One placement note from the 8.0 guide: between Process Spec Cost `PM_I_009` and Job Stage Cost `PM_I_005`, only Job Stage Cost offers the two summary options.)

Alongside them, **Job Exception Inquiry** (`PM_I_034`) compares actual inputs and outputs against planned and lists the jobs and lines that violate the factory or process-spec limits — material inputs against the Material Cost threshold, labor against Labor, machine against Machine, miscellaneous against Material, and material outputs against Unit Cost. Job campaigns are excluded.

> **rule of thumb** — if what you need is "show me the jobs whose cost went sideways, fast," `PM_I_034` plus the two summary windows gets you there today, on any client, on any 8.0.x, with no batch update and no add-in. The graphical supply-chain picture is what the trace features add on top — and on 8.0.2 that picture still costs you a scheduled `IC_U_033`.

## So, to answer the question as asked

```steps
1 | Read your menu
where: Product Costing
do: If it says Trace Cost you're on 8.0.2 and the next two steps are yours. If it says Cost Trace, you're below 8.0.2 and step 4 is yours.
sys: Trace Cost is IC_I_037; Cost Trace is PM_I_032.

2 | Find out whether Trace Cost Update has ever run
where: IC_U_033
do: Run it once by hand against a job you know is Closed, then reopen the inquiry on that job. A number appearing means the whole problem was scheduling.
note: Until it runs, Trace Unit Cost is 0 on every job — and only Closed and History jobs ever carry a value.

3 | Rule out the free explanation
where: PM_M_011 / PM_M_023
do: Confirm threshold rows exist for the factory and specs you're testing. An all-gray grid with no threshold rows is working exactly as designed.
sys: FACTORY_THRESHOLDS and PROCESS_SPEC_THRESHOLDS, threshold types 01/02/03/11/12, four band columns each.

4 | If you're below 8.0.2, make 8.0.2 the target
where: Upgrade planning
do: Ask by facility ID — Trace Cost Inquiry (IC_I_037) and Trace Cost Update (IC_U_033). They need no Smart Client and no add-in registration.
note: There is essentially no Ross-side code in PM_I_032 to fix, so sessions against it have very little to work with.

5 | Stand up the fallback that already works
where: PM_I_034, PM_I_005, PM_I_029, PM_I_003
do: Put Job Exception Inquiry and the Job Cost Summary / Job Stage Cost Summary windows in front of the people who need numbers now.
note: Same thresholds, same variance math, no dependency on any of the above.
```

> **field note** — if you're on 8.0.2 and `IC_U_033` is running on a schedule and the numbers are still wrong, that's a genuine defect worth a ticket, and it's a much better ticket than "Trace Cost doesn't work" because it names a facility, a batch and an expected value. If you're below 8.0.2, get the support statement in writing — *is Cost Trace (PM_I_032) supported on our exact release, and against which Smart Client and IAF versions?* The published ceiling is 8.0.0. That answer either unblocks the sessions or ends them, and both beat another month of neither.

One aside, since this landed in a thread about tracing for audits: the slow-and-inconsistent **Trace Express** complaints are a separate feature with a separate fix list. The 8.0.1 release notes carry several Trace Express corrections, and 8.0.1 also introduces a **Trace Recall** capability. If an audit window is the real deadline, that's a different conversation and a more promising one — the fixes exist there.
