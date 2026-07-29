---
num: "004"
title: "Is there a blanket sales order in Ross, or do I fake it?"
tag: "ROSS"
audience: "developer"
source: "form"
date: "2026-07-24"
system: "Ross ERP 8.0"
reading_time: "8 min"
excerpt: "There's no blanket sales order. The quantity-and-price commitment lives in Sales Contract Pricing (it draws down like a blanket); the orders named CONINV / CONSHP are a different animal — contract manufacturing / customer-owned goods."
question: "Purchasing has blanket orders — agree a quantity and a price, then release against it over time. I need the same thing for a customer on the sell side. Is there a blanket sales order, or am I building it myself?"
restated: "Does Ross have a blanket sales order equivalent to the blanket PO, and if not, which sell-side feature gives a customer a capped quantity at a fixed price drawn down over time?"
fix: "No 'blanket sales order' exists. The quantity-and-price commitment lives in Sales Contract Pricing, which draws down like a blanket — a pricing agreement applied to ordinary orders, not a separate document. The orders literally named 'Contract' — CONINV and CONSHP — are unrelated: they're the contract-manufacturing / customer-owned-goods flow."
reviewed: "2026-07-24"
verified: "Verified against Ross 8.0 source"
key_refs:
  - "SALES_CONTRACT_PRICES"
  - "SOP_PRICES"
  - "sop_l_get_prices"
related:
  - "q-002-blanket-purchase-orders"
applies_when:
  - "You went looking for a blanket *sales* order and only found `CONINV` / `CONSHP`."
  - "A customer wants a standing quantity-and-price agreement you can draw down over time."
  - "A customer swears they had a contract rate, but the order booked at list price."
margin_notes:
  - "no blanket sales order ↴"
  - "pricing IS the blanket →"
  - "CONINV/CONSHP = customer-owned"
---

Purchasing gets the clean version of this: a blanket PO is a real document — a master agreement with a release engine bolted on. Go looking for the same thing on the sell side and you'll find two features that both say *contract*, and neither is quite what you came for.

> **in plain terms** — there's no order type called a "blanket sales order." The commitment you want — a capped quantity at a fixed price, drawn down over time — rides on the **price**, not on a special document. Set it up as a sales *contract price* and let ordinary orders consume it.

## 01 · The one that actually behaves like a blanket

**Sales Contract Pricing** is the mechanism that behaves like a blanket PO: a customer agrees to a **quantity at a fixed price over a date window**, and normal sales orders draw it down until it's exhausted or expires. It's a *pricing agreement* applied to ordinary orders — not a separate order document.

Turn it on with `AR_CONTROLS(CONTRACT_PRICES_IN_USE) = Y` per division (in `sop_m_controls`); optionally `CONTRACT_PRICES_OVERRIDE` to let operators edit a contract-derived price. Contracts are defined in `SYS_M_SALES_CONTRACT_PRICES`.

| Table | Role | Commitment fields — the blanket bit |
| --- | --- | --- |
| `SALES_CONTRACT_PRICES` / `_LINES` | Definition — what you set up | `CONTRACT_QTY`, `CONTRACT_QTY_ORD_TD`, `CONTRACT_QTY_OUTSTANDING`, `VALID_FROM/TO_DATE`, status O/C |
| `SOP_PRICES` / `_LINES` | Runtime — what the engine reads | Same fields; contract rows carry `PRICE_TYPE = "1"` |

```steps
1 | Define the contract price
where: SYS_M_SALES_CONTRACT_PRICES
do: agree a quantity cap at a fixed price over a date window for the customer / product.
sys: writes `SALES_CONTRACT_PRICES` / `_LINES` — `CONTRACT_QTY`, `VALID_FROM/TO_DATE`, status O. Contract rows surface at runtime in `SOP_PRICES` with `PRICE_TYPE = "1"`.

2 | Enter a normal sales order
do: take an ordinary customer order — there is no special "blanket" order type to choose.

3 | Pricing resolves — the contract wins first
where: sop_l_get_prices
do: nothing manual; the engine tries the contract tier before promotion, customer, and list price.
sys: a contract lookup requires `CONTRACT_QTY_OUTSTANDING ≥ requested qty` and today inside the validity window. Priority is contract → promotion → customer/product → list (unless `PROMOTION_OVERRIDE_CONTRACTS`).

4 | Draw it down
where: sop_l_contract_price_qty_update
sys: `CONTRACT_QTY_ORD_TD += qty`; `CONTRACT_QTY_OUTSTANDING` shrinks by the same amount.

5 | Auto-close
sys: the contract stops being used once `CONTRACT_QTY_OUTSTANDING` hits zero or the date passes `VALID_TO_DATE`.
```

At order entry the pricing engine tries the contract tier first — and only if there's quantity left on the clock:

```dml
@program SOP_L_GET_PRICES.DML
@note SALES PRICE RESOLUTION · THE CONTRACT TIER IS TRIED FIRST
@reads sop_prices  (price_type "1" = contract)
@writes sop_prices.contract_qty_ord_td  (the draw-down)
@risk an exhausted contract is skipped silently — the order books at list price
@highlight 3-4
! resolution order, most specific first:
! 1) contract  2) promotion  3) customer/product  4) list
FIND sop_prices /WITH=PRICE_TYPE = "1" &                 ! the contract tier
                /WITH=CONTRACT_QTY_OUTSTANDING >= #QTY &  ! the cap — the blanket part
                /SORTED_BY=-CUSTOMER_NUMBER
IF (%STATUS = %SUCCESS)
    #PRICE = CONTRACT_PRICE
    CONTRACT_QTY_ORD_TD = CONTRACT_QTY_ORD_TD + #QTY      ! draw it down
END_IF
```

The running balance maps almost one-for-one onto the blanket PO you already know: `CONTRACT_QTY` ≈ `ORDER_QUANTITY`, `CONTRACT_QTY_ORD_TD` ≈ `POP_QTY_RELEASED`, `CONTRACT_QTY_OUTSTANDING` ≈ the outstanding-to-release balance. The difference is that it's a price applied to normal orders, not a distinct document with a release engine.

> **field note** — the failure mode is quiet. When a contract runs out of quantity, the engine doesn't stop you — it skips the contract, falls through to list price, and the order books anyway at the wrong number. If a customer swears they had a contract rate and the order says otherwise, check `CONTRACT_QTY_OUTSTANDING` before you check your setup.

## 02 · The one that only sounds like a blanket

Ross has order types named **CONINV** and **CONSHP**, reached through their own entry modes that stamp `SALES_ORDER_HEADERS(SALES_ORDER_TYPE)` — `SOP_T_001D` "Contract Invoice Order" → `CONINV`, `SOP_T_001E` "Contract Ship Order" → `CONSHP`. The name is a trap: these aren't a quantity commitment, they're the **contract-manufacturing / customer-owned-goods** flow.

```cards
CONINV | Contract Invoice | Invoice directly from the order — no despatch, no ship note. Stock moves at invoice time; does not commit stock (`sop_l_do_ic_commitments` exits immediately).
CONSHP | Contract Shipment | Zero-value shipment of customer-owned stock from the `IC_STATUS_CQOH` bucket (not normal QOH), then invoiced from that despatch with frozen pricing.
```

There's no master, no releases, no agreed cap — each is a single standalone document, and both simply draw down the standard `SALES_ORDER_LINE_QTYS(ORDER_QUANTITY_OUTSTANDING)`, closing when consumed. On CONSHP the price is forced to `0` ("Customer Owned Status"), the part is forced non-taxable, and discounts/promotions are off.

> **caution** — reach for `CONINV` / `CONSHP` expecting a blanket and you'll end up modelling toll manufacturing by accident. They exist to invoice and ship goods the customer already owns — not to hold a quantity-and-price commitment.

## 03 · Three mechanisms, side by side

| | Blanket PO (POP) | Sales Contract Pricing | Contract Orders (CONINV/CONSHP) |
| --- | --- | --- | --- |
| **Purpose** | Buy a capped qty at a price over time | Sell a capped qty at a price over time | Invoice / ship customer-owned goods |
| **Qty commitment** | Yes — `ORDER_QUANTITY` | Yes — `CONTRACT_QTY` | No commitment concept |
| **Draw-down field** | `POP_QTY_RELEASED` | `CONTRACT_QTY_ORD_TD` | `ORDER_QUANTITY_OUTSTANDING` |
| **Is it a document?** | Yes — N master + R releases | No — a price on normal orders | Yes — a standalone order type |
| **Dedicated engine** | Yes — release program | Pricing engine + qty draw-down | Invoice-from-despatch flow |

## 04 · So which do I reach for?

> **rule of thumb** — if the customer's ask is "a standing quantity at a locked price, drawn down as they order," that's **Sales Contract Pricing** — the real blanket analog. If it's "we hold or process goods the customer owns," that's `CONINV` / `CONSHP`. A true blanket *document* with releases, the way the PO side works, doesn't exist on the sell side — you approximate it with contract pricing and let normal orders be the "releases."

The short version: purchasing got a purpose-built blanket subsystem; sales splits the same needs across a pricing feature and a contract-manufacturing order flow. Once you stop looking for the word *blanket* on a sales order, the sell-side version is right where it should be — one layer down, in the price.

## 05 · Where this comes from

- `sop_l_get_prices` — sales price resolution; the contract tier (`PRICE_TYPE = "1"`) is tried first, gated by `CONTRACT_QTY_OUTSTANDING` and the validity window.
- `sop_l_contract_price_qty_update` — writes the draw-down (`CONTRACT_QTY_ORD_TD += qty`).
- `sop_m_controls` — `CONTRACT_PRICES_IN_USE` / `_OVERRIDE`; `SYS_M_SALES_CONTRACT_PRICES` defines the contracts.
- `sop_t_order_maintenance` modes `SOP_T_001D` / `SOP_T_001E` — stamp the `CONINV` / `CONSHP` order types.

The buy-side counterpart — the real blanket-document engine — is covered in [the blanket purchase-order rundown](/q/q-002-blanket-purchase-orders).
