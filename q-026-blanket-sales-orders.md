---
num: "026"
title: "Is there a blanket sales order in Ross, or do I fake it?"
tag: "ROSS"
audience: "developer"
source: "form"
date: "2026-07-23"
system: "Ross ERP 8.0"
reading_time: "6 min"
excerpt: "There's no blanket sales order. The commitment you want lives in Sales Contract Pricing; the thing named 'Contract order' is a different animal entirely."
question: "Purchasing has blanket orders — agree a quantity and a price, then release against it over time. I need the same thing for a customer on the sell side. Is there a blanket sales order, or am I building it myself?"
fix: "No 'blanket sales order' exists. The quantity-and-price commitment lives in Sales Contract Pricing, which draws down like a blanket. The orders literally named 'Contract' — CONINV and CONSHP — are unrelated: they're for contract manufacturing and customer-owned goods."
margin_notes:
  - "two 'contracts', not one ↴"
  - "pricing IS the blanket →"
  - "CONINV/CONSHP = customer-owned"
---

Purchasing gets the clean version of this. A blanket PO is a real document — a master agreement with a release engine bolted on: agreed quantity, agreed price, and a running balance you call off against. Go looking for the same thing on the sell side and you'll find two features that both say *contract*, and neither is quite what you came for.

## The one that actually behaves like a blanket

It's **Sales Contract Pricing** — and the giveaway is that it isn't an order type at all. You agree a quantity at a price over a date window, and ordinary sales orders draw it down until it's exhausted or it expires. The commitment rides on the *price*, not on a special document.

The running balance is the whole trick, and it maps almost one-for-one onto the blanket PO you already know:

| Blanket PO (POP) | Sales Contract Pricing |
| --- | --- |
| `ORDER_QUANTITY` (the cap) | `CONTRACT_QTY` |
| `POP_QTY_RELEASED` | `CONTRACT_QTY_ORD_TD` |
| outstanding to release | `CONTRACT_QTY_OUTSTANDING` |
| release against the master | a normal order consumes the contract |

At order entry the pricing engine tries the contract tier first — before promotion, customer, or list price — and only if there's quantity left on the clock:

```dml
@program SOP_L_GET_PRICES.DML
@note SALES PRICE RESOLUTION · THE CONTRACT TIER WINS FIRST
@reads sop_prices  (price_type "1" = contract)
@writes sop_prices.contract_qty_ord_td  (the draw-down)
@risk an exhausted contract is skipped silently — the order books at list price
@highlight 4-5
! resolution order, most specific first:
! 1) contract   2) promotion   3) customer/product   4) list
FIND sop_prices
    WHERE price_type = "1"                  ! the contract tier
      AND contract_qty_outstanding >= :qty  ! the cap — this is the blanket part
      AND :order_date BETWEEN valid_from AND valid_to
    SORT BY -customer_number, -valid_from
IF found THEN
    price = contract_price
    contract_qty_ord_td = contract_qty_ord_td + :qty   ! draw it down
END_IF
```

> **field note** — the failure mode is quiet. When a contract runs out of quantity, the engine doesn't stop you. It skips the contract and falls through to list price, and the order books anyway — just at the wrong number. If a customer swears they had a contract rate and the invoice says otherwise, check `contract_qty_outstanding` before you check your setup.

## The one that only sounds like a blanket

Ross has order types named **CONINV** ("Contract Invoice") and **CONSHP** ("Contract Shipment"), reached through their own entry modes, that stamp `SALES_ORDER_TYPE`. The name is a trap. These aren't a quantity commitment — they're the **contract-manufacturing / customer-owned-goods** flow.

- **CONSHP** ships at *zero value* from a dedicated customer-owned inventory bucket. The stock isn't yours to sell; you're moving goods the customer already owns.
- **CONINV** invoices straight from the order with no despatch, moving stock at invoice time.

There's no master, no releases, no agreed cap. Reach for these expecting a blanket and you'll end up modelling toll manufacturing by accident.

## So which do I reach for?

- **A customer standing agreement** — quantity and price locked over time, drawn down as they order: **Sales Contract Pricing.**
- **Holding or processing goods the customer owns:** CONINV / CONSHP.
- **A true blanket *document* with releases, the way the PO side works:** it doesn't exist. You approximate it with contract pricing and let normal orders be the "releases."

The short version: purchasing got a purpose-built blanket subsystem, and sales got the same idea split across a pricing feature and a manufacturing flow. Once you stop looking for the word *blanket* on a sales order, the sell-side version is right where it should be — one layer down, in the price.
