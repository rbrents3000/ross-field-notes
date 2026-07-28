---
num: "003"
title: "What has to be set up before blanket orders will work?"
tag: "ROSS"
audience: "user"
source: "form"
date: "2026-07-24"
system: "Ross ERP 8.0"
sp_note: "reviewed vs SP2"
reading_time: "6 min"
excerpt: "Blanket orders turn on with standard POP — no separate switch. Set the company controls, activate POP for the division, seed the code tables (incl. requisition type BK), grant PO security, and ensure every line has GL postings before activating."
question: "We want to start using blanket orders, but before anyone can create one I need to know what has to be configured — control flags, security, GL. What's the actual checklist, in the order I'd set it up?"
restated: "What configuration — control flags, code tables, security rights, GL/commitment setup, and master data — must be in place before blanket orders can be created, activated, and released in Ross?"
fix: "Blanket orders don't need a separate 'blanket module' switch — they turn on with standard Purchase Order Processing (POP). Set the company controls, activate POP for the division, seed the code tables (including requisition type BK), grant the four PO security rights, wire GL/commitment accounts if funds are in use, and make sure every blanket line has GL postings before you activate."
reviewed: "2026-07-24"
verified: "Verified against the 8.0 SP2 code baseline"
key_refs:
  - "pop_m_controls"
  - "gl_m_controls"
  - "POP_REQUISITION_TYPES"
related:
  - "q-002-blanket-purchase-orders"
margin_notes:
  - "turns on with POP ↴"
  - "requisition type BK →"
  - "every line needs GL postings"
---

Blanket orders don't need a separate "blanket module" switch — they turn on with standard **Purchase Order Processing (POP)**. What actually matters is that POP is active for the division, security is granted, and — if you run fund/commitment accounting — the commitment flags and GL accounts are set. Work top to bottom; each layer depends on the one above it.

> **in plain terms** — there's nothing exotic to install. You're switching POP on for the division, making sure the code tables and security exist, and (only if you track commitments) wiring up the GL side. If a blanket later won't activate, it's almost always a missing GL posting on a line — see the last section.

```legend
required | Must be set for blanket orders to work.
if-funds | Only when commitment / encumbrance accounting is on (FUND_IN_USE = YES).
master-data | Records that must already exist.
optional | Turn on only if you want that feature.
gotcha | A common reason blankets won't activate or release.
```

## 01 · Company controls

```checklist
[ ] `FUND_IN_USE` {required} — Decide yes/no. Gates the entire commitment / encumbrance machinery and whether the blanket commit flags below are even enterable.
where: gl_m_controls
[ ] `PRICING_METHOD_STATE` {required} — Inclusive / exclusive / optional pricing; drives how blanket line prices and the max-value cap are entered.
where: sys_m_controls
[ ] `COMPANY_SECURITY_ACTIVE` {required} — If YES, the transaction-type create right (section 4) is enforced; if no, that check is skipped. Either way, set it deliberately.
[ ] `GL_COMMIT_TAX` · `SYS_BUDGETS_IN_BASE` · `SYS_POSTING_DATE_REQD` {if-funds} — Committed tax, base-currency conversion of commit values, and posting-date capture. Only meaningful when funds are in use.
where: gl_m_controls / sys_m_controls
[ ] `ATP_IN_USE` {optional} — Turn on only if you want releases to feed available-to-promise; also needs the matching part/warehouse flags (section 6).
where: sys_m_controls
```

## 02 · Division / POP controls

```checklist
[ ] `POP_ACTIVE` {required} — Must be YES — the master gate. If POP isn't active for the division, blanket entry and release are refused outright.
where: pop_m_activate_module
[ ] `POP_CURRENT_YEAR` · `POP_CURRENT_PERIOD` {required} — Must be set before POP can be activated.
where: pop_m_controls
[ ] `POP_BLANKET_COMMIT_FLAG` {if-funds} — What gets pre-committed when a blanket is entered: N = none, L = line value, M = maximum line value (default N). Only enterable when FUND_IN_USE = YES.
where: pop_m_controls
[ ] `POP_PRE_COMMITMENT_STAGE` {if-funds} — Enables the pre-commitment (encumbrance) stage. Forced off when funds aren't in use.
where: pop_m_controls
[ ] `DEF_REQUISITION_TYPE` {required} — Default requisition type, validated against `POP_REQUISITION_TYPES` — the table that must contain the blanket type `BK` (section 3).
where: pop_m_controls
[ ] Pricing, currency & tax defaults {required} — `CONTRACT_PRICES_IN_USE`, the division base currency, and the tax defaults that flow onto released lines.
where: pop_m_controls
```

## 03 · Transaction types and numbering

```checklist
[ ] `POP_TRANSACTION_TYPES` / `AP_TRANSACTION_TYPES` {required} — Carry the PO auto/manual numbering that generation reads. Created for you when POP is activated.
where: auto-created by pop_m_activate_module
[ ] `PO_TYPES` {required} — The PO-generation screen requires a PO type validated against this table.
where: shared code maintenance
[ ] `POP_REQUISITION_TYPES` must include `BK` {required} — Blanket requisitions carry `REQUISITION_TYPE = "BK"`; generation blocks turning a BK requisition into a plain PO, so the type must exist.
where: shared code maintenance
```

## 04 · Security

```checklist
[ ] Accessible company for `MODULE_PO` {required} — The user must have at least one PO-accessible company, or the program aborts.
[ ] Warehouse access — `ACCESS_TYPE_WAREHOUSE` for `MODULE_PO` {required} — At least one accessible warehouse.
[ ] Division access — `SYS_ALLOW_ACCESS = YES` {required} — Granted per division (`ACCESS_TYPE_DIVISION`, `MODULE_PO`).
[ ] Create-transaction right — `TTYPE_PO` {required} — `SECURITY_TRAN_TYPES_VT` must grant `CREATE_TRANSACTIONS = YES`. Enforced only when `COMPANY_SECURITY_ACTIVE = YES`. Note: the right is `TTYPE_PO`, not `TTYPE_BL`.
```

## 05 · GL and funds

```checklist
[ ] GL accounts on blanket lines flagged `FUND_IN_USE = YES` {if-funds} — Only fund-flagged accounts post commitments; an unflagged expense account commits nothing for that line.
[ ] Budget check level / period · tolerance · summary account {if-funds} — Configure on those GL accounts.
[ ] Commitment period / year on the blanket header {if-funds} — Captured at entry and used as the posting period for the pre-commitment.
[ ] Posting flow, for reference {fyi} — At activation a pre-commitment posts under `TTYPE_BL` to `PRECOM`; at release it reverses and a firm commitment posts under `TTYPE_PO` to `COMMIT`.
```

## 06 · Master data

```checklist
[ ] Vendor {master-data} — Active, not on `HOLD_PO` or `STOP_PAYMENT`, not deactivated, with a default address code.
[ ] Product master / product-warehouse {master-data} — Part set up in the warehouse with its purchase UOM, taxable code, and (if used) the ATP source flags.
[ ] Addresses · credit terms · currency · cost centers {master-data} — Invoice/delivery/vendor address codes, the credit-terms code, division base currency, and any cost centers referenced.
[ ] Warehouse order horizon {master-data} — `DAYS_BACKWARD_ORDERED` / `DAYS_FORWARD_ORDERED` classify each release's required date; the receiving warehouse must be active.
```

## 07 · Before it will activate

> **gotcha** — a blanket won't move to **Active** unless it has at least one line *and* **every** line has a GL posting record. If any line is missing one, activation reverts the status and names the failing line. Because only an Active blanket can be released against, this is the single most common "why won't it work" cause — check line-level GL postings first.

## 08 · Which service pack

> **caution** — this checklist was reviewed against the **8.0 SP2** code baseline. A few SP2-era additions around these screens — requisition-workflow approvals and invoice-matching tolerance controls in `pop_m_controls`, and the blanket-PO inquiry fix (ticket 477310) — may look different or be absent at SP1. Confirm the running level from the `RENCS_VERSION` and `ERP_SERVICE_PACK` parameters before relying on the SP2-only items.

Once this is all in place, the mechanics of creating, activating and releasing a blanket are covered in [the end-to-end rundown](/q/q-002-blanket-purchase-orders).
