---
num: "002"
title: "How do blanket purchase orders work in Ross, end to end?"
tag: "ROSS"
audience: "developer"
source: "form"
date: "2026-07-24"
system: "Ross ERP 8.0"
sp_note: "SP1 tree; one inquiry fix in SP2"
reading_time: "9 min"
excerpt: "Blanket orders ride the standard PO tables, tagged by POP_ORDER_TYPE — N (master), R (release), P (plain PO). Create Inactive, activate, release against, receive, close."
question: "We want to use blanket / contract purchasing — agree a quantity and a price with a supplier, then release against it over time. Before I build any reports or integrations around it, how does the whole thing actually work in Ross: the tables, the order types, the lifecycle, and the guardrails?"
restated: "How do blanket (contract) purchase orders work end to end in Ross — the order types, tables, lifecycle, release workflow, and the limits that govern releasing?"
fix: "Blanket orders don't live in dedicated tables — they ride the standard PO tables (POP_HEADERS / POP_LINES / POP_LINE_DETAILS), distinguished by one field, POP_ORDER_TYPE: N = the master/contract, R = a release against it, P = an ordinary PO. Create the master Inactive, activate it (every line needs GL postings), release against the Active master, then receive and invoice like any PO — and close it when it's spent."
reviewed: "2026-07-24"
verified: "Verified against Ross 8.0 source"
key_refs:
  - "POP_HEADERS"
  - "POP_LINES"
  - "POP_LINE_DETAILS"
  - "POP_VENDOR_RELEASES"
related:
  - "q-003-blanket-orders-setup"
margin_notes:
  - "one field: POP_ORDER_TYPE ↴"
  - "master N, release R →"
  - "activate needs GL postings"
---

Purchasing gets the clean version of the "standing agreement" idea: a blanket PO is a master agreement with a release engine bolted on — agreed quantity, agreed price, and a running balance you call off against. Here's the whole thing end to end, from the data model up.

> **in plain terms** — a blanket order is a standing deal with a supplier: agree once on an item, a total quantity, a price and a time window, then "call off" smaller batches — **releases** — against it whenever you need them, with no re-negotiating each time.

## 01 · Three order types, one set of tables

Since the v4.3 redesign (conversion `v43_e09946` dropped the old `POP_BLANKET_ORDER_*` tables), blankets no longer live in dedicated tables. They ride the **standard PO tables**, distinguished by one field — `POP_ORDER_TYPE`, whose domain is exactly three values.

```cards
N | Non-released Blanket | The master / contract — the blanket itself. Holds the agreed max quantity and value.
R | Released Blanket | A release / call-off issued against a blanket. What actually gets received and invoiced.
P | Purchase Order | An ordinary standalone PO. Not a blanket — shown for contrast.
```

> **field note** — a blanket *requisition* isn't a `POP_ORDER_TYPE` value; it's flagged in the requisition subsystem as `REQUISITION_TYPE = "BK"`. The generate program refuses to turn a `"BK"` requisition into a plain `"P"` order.

## 02 · A blanket moves through four states

State lives in `POP_HEADERS(POP_BLANKET_STATUS)`. The release program will only touch a blanket that is **Active**.

```states
Inactive | STATUS "I"
Active | STATUS "A"
Releasing | type "R" rows
Closed | STATUS "C"
```

> **caution** — the two creation paths disagree on the starting state. **Order Maintenance** always creates the blanket *Inactive* and forces an explicit activation step; **generate-from-requisition** stamps it *Active* immediately. Status codes are driven by the global `STATUS_*` parameters — there's no hard `CHECK` constraint on the column, so don't assume a fixed domain in queries.

## 03 · Where the numbers live

The blanket **master** is a header + line (`N`). Each **release** is captured as child `POP_LINE_DETAILS` rows (stamped `R`) — one per scheduled delivery date — which roll up onto the master line and into a per-release ledger.

| Field | Table | Meaning |
| --- | --- | --- |
| `ORDER_QUANTITY` | POP_LINES | The blanket **maximum** agreed qty for the line — the release cap. |
| `POP_ORDER_TOTAL_CURRENCY` | POP_LINES | The blanket **maximum value** for the line — the value cap. |
| `POP_QTY_RELEASED` | POP_LINES | Cumulative qty released to date; incremented by each release. |
| `POP_RELEASED_VALUE` | POP_LINES | Cumulative value released to date. |
| `PO_QTY_OUTSTANDING` | POP_LINES | Computed: `ORDER_QUANTITY − accepted − QC − quarantine − closed − received − returned + credited`. |
| `POP_MIN / MAXIMUM_RELEASE_QTY` | POP_LINES | Per-release floor & ceiling (releases only). |
| `POP_EFFECTIVE_DATE` / `EXPIRY_DATE` | POP_HEADERS | Valid window; no release before effective, blank expiry = open-ended. |
| `RELEASE_NO`, `REQUIRED_DATE` | POP_LINE_DETAILS | Sequential release number + one scheduled delivery date per row. |
| `POP_QTY_RELEASED`, `POP_RELEASED_VALUE` | POP_VENDOR_RELEASES | Per-release ledger, keyed by vendor + release number. |

## 04 · The workflow, step by step

```steps
1 | Set up the agreement
where: pop_t_order_maintenance
do: pick the vendor, set the start and end dates (end after start, or blank for open-ended), then add each item with the total quantity you're committing to and its price. Optionally cap how large or small a single call-off may be. Save.
sys: writes a `POP_HEADERS` + `POP_LINES` record with `POP_ORDER_TYPE = "N"`. Agreed qty → `ORDER_QUANTITY`, price × qty → `POP_ORDER_TOTAL_CURRENCY`, dates → `POP_EFFECTIVE_DATE` / `EXPIRY_DATE`. Saved here it lands Inactive ("I").

2 | Turn it on
do: activate the contract so it can be used. The system only lets you activate it if today is inside its date window and every line is set up correctly. (Auto-generated blankets are already active — skip this.)
sys: Activate Orders flips `POP_BLANKET_STATUS` from `I` to `A` — but only if the window covers today and every line has GL postings; then WMS is notified. Only an Active blanket can be released against.

3 | Call off what you need — the release
where: pop_u_blanket_order_release
do: when you actually need stock, raise a release against the contract: choose the contract and item, and enter one or more delivery dates and quantities. Repeat as often as you like until the agreed total is used up.
sys: each grid row becomes a `POP_LINE_DETAILS` row (type `R`) with the next sequential `RELEASE_NO`. It rolls up `POP_QTY_RELEASED` / `POP_RELEASED_VALUE`, writes the `POP_VENDOR_RELEASES` ledger, updates inventory demand/supply, and turns the GL pre-commitment into a firm commitment.

4 | Send it to the vendor
where: pop_r_order_print
do: print or transmit the release to the supplier as a purchase order.
sys: prints releases (type `R`) in a separate batch from standard POs. A master prints its committed/accepted qty; a release prints its ordered qty.

5 | Receive and invoice
do: receive the goods and process the supplier's invoice exactly as you would for any PO — there's nothing blanket-specific to learn here.
sys: releases flow through standard goods-receipt (GRN) and AP invoicing, keyed by the release's own PO number.

6 | Close it out
do: when the contract is used up or past its end date, close it so no more releases can be raised against it.
sys: Close Blanket Orders sets `POP_BLANKET_STATUS` to `C`.
```

## 05 · The rules that stop you over-releasing

> **in plain terms** — the system won't let you call off more than the agreed quantity or value, book a delivery outside the contract's dates, or release against a contract that isn't active. It stops you with a specific message each time.

| Rule | Enforced by | Message |
| --- | --- | --- |
| Cumulative release qty can't exceed the line max | `(this qty + POP_QTY_RELEASED) ≤ ORDER_QUANTITY` | `P_01240` |
| Cumulative release value can't exceed the max value | `≤ POP_ORDER_TOTAL_CURRENCY` | `P_01239` |
| A single release must sit within per-release min/max | `POP_MIN…MAXIMUM_RELEASE_QTY` | `P_13649 / P_01243` |
| Can't shrink a line below what's already released | `qty ≥ released; value ≥ released` | `P_BL010 / P_BL011` |
| Blanket must be Active (not expired, not future-dated) | `POP_BLANKET_STATUS = "A"` | `P_37302 / P_01229 / P_01230` |
| Line can't be closed for receipt or invoicing | `CLOSED_FOR_GRN / CLOSED_FOR_PI` | `P_03221` |

## 06 · Commitments and the pre-commit flag

> **in plain terms** — if your finance setup tracks commitments (encumbrances), a blanket can earmark budget up front, and each release turns that earmark into a firm commitment. If it doesn't, this section won't affect you.

If funds/encumbrance accounting is on (`COMPANY_CONTROLS(FUND_IN_USE)`), behavior is driven by `AP_CONTROLS(POP_BLANKET_COMMIT_FLAG)`:

| Flag | What gets pre-committed at blanket entry |
| --- | --- |
| `N` | **None** — the commitment is created when you release. |
| `L` | **Line value** is pre-committed (PRECOM) when the blanket is entered. |
| `M` | **Maximum line value** is pre-committed when the blanket is entered. |

On each release, the pre-commitment is **reversed** and a **firm commitment** is posted through `GL_L_FUND_UPDATE`, pro-rated across the line's expense accounts in the blanket's commitment period. Commitment is carried on `POP_GL_POSTINGS` — there is no table literally named `POP_COMMITMENT`.

## 07 · Reports and inquiry

- **Open Blankets Report** — `pop_r_open_blankets`. Lists blanket/release lines with quantity still available to draw down, showing max vs. released qty and value per PO and vendor.
- **Order Inquiry** — `pop_i_order_inquiry`. In blanket mode it shows `N` and `R` together; the release balance is `ORDER_QUANTITY − POP_QTY_RELEASED − QTY_CLOSED`, and you can drill into each `RELEASE_NO`.

> **caution** — one blanket inquiry defect was fixed in **8.0 SP2** (ticket 477310): before SP2, order inquiry returns **bad data for release-type (`R`) blankets**, corrected by a `PURCHASE_INQUIRY_FOR_BLANKET` view. If your active tree is at SP1, treat release-blanket inquiry results with caution until SP2 is applied.

## 08 · Where this comes from

- `pop_u_blanket_order_release` — the release transaction; creates the call-offs. The heart of it.
- `pop_t_order_maintenance` — create/maintain blankets & releases; Activate; Mass Close.
- `pop_t_po_generate` — generate POs / blankets / releases from requisitions.
- `pop_r_open_blankets` — Open Blanket Orders report.
- `pop_i_order_inquiry` — blanket / release order inquiry.
- `v43_e09946_blanket_orders` — the v4.3 conversion that removed the legacy `POP_BLANKET_ORDER_*` tables.

> **caution** — three watch-outs. **`"R"` is overloaded** — it's both the `POP_ORDER_TYPE` of a released order document *and* the type stamped on the `POP_LINE_DETAILS` release rows under an `"N"` master; know which layer you're reading. **`POP_BLANKET_STATUS` has no hard domain** (driven by global `STATUS_*` params, no SQL `CHECK`) — don't hard-code assumptions. And **`PO_QTY_OUTSTANDING` is computed, not stored** — don't try to `UPDATE` it.

Before you can create any of this, the module has to be configured for it — see [the setup checklist](/q/q-003-blanket-orders-setup).
