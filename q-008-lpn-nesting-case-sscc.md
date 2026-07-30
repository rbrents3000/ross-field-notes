---
num: "008"
title: "An EDI customer wants a unique SSCC on every case. Is LPN nesting the right tool — and can I limit it to just that customer?"
tag: "ROSS"
audience: "developer"
source: "group"
date: "2026-07-30"
system: "Ross ERP 8.0"
reading_time: "11 min"
excerpt: "LPN nesting is exactly how Ross puts a unique SSCC on every case — a parent pallet LPN over child case LPNs, each drawing its own serial, feeding the pallet→case hierarchy in the 856. But the switch lives on the Product Master, not the customer. There is no 'nest for customer X only' flag anywhere in vanilla Ross, so once you enable a shared product it nests for everyone who ships it. You scope the customer at the EDI/label layer, not in the nesting setup."
question: "We have a new requirement from one of our EDI customers to include a unique SSCC barcode on each individual case. We don't do case-level SSCCs today, and we were wondering whether this is a good use of LPN nesting (parent/child LPNs — also something we don't do). Is this a good application for LPN nesting? What limitations can we put on it? Can we restrict it to a single customer? Can we restrict it to products on that customer's orders only? The products in question are not customer-specific — if nesting is enabled at the product level, what downstream impact would that have on shipping those same products to other customers? We've struggled to find detailed instructions on implementing this or its impact on other customers/products/shipping, and we're open to other options — this is just what came to mind."
restated: "In Ross, is LPN nesting the correct mechanism for putting a unique SSCC on each shipped case, and can that behavior be scoped to a single EDI customer (or that customer's products) without affecting how the same, non-customer-specific products ship to everyone else?"
fix: "Yes — LPN nesting is the vanilla mechanism that produces case-level SSCCs: a parent (pallet) license plate over child (case) license plates, each plate drawing its own SSCC from the warehouse's serial counter, and at ship confirm the child cases become the case-level load carriers under the pallet in the 856 ASN hierarchy. But it is turned on per PRODUCT, not per customer. The three nesting fields (PLPN Entry at, No. of CLPNs, CLPN Default Quantity) plus Serialization in Use all live on the PRODUCT_MASTER; the customer master has no nesting, SSCC, or GS1 field at all. So you cannot natively say 'nest only for customer X.' Enable it on a product and it nests wherever that product is built into cases — for every customer. The customer-level scoping you're after lives one layer out, in EDI: build cases (nested case transfer) and emit the case-SSCC label + the 856 SU/load-carrier loop only for the trading partners who require it. If you truly need physical isolation, the only structural lever is a separate product/pack code — which fragments the item and is rarely worth it."
margin_notes:
  - "case SSCC = a child (case) LPN under a parent (pallet) LPN ↴"
  - "nesting switch is on the PRODUCT_MASTER, not the customer →"
  - "No. of CLPNs is capped by system param DC_MAX_CLPNS"
  - "enable a shared product and it nests for everyone"
  - "scope the customer in EDI/labels, not in the nesting setup"
reviewed: "2026-07-30"
verified: "Verified against Ross 8.0 source"
applies_when:
  - "An EDI trading partner requires a unique SSCC (GS1 18-digit) on each individual case, not just the pallet."
  - "You are considering parent/child (nested) license plates for the first time and need to know what it touches."
  - "The products that need case SSCCs are shared items that also ship to customers who do not want case-level serialization."
key_refs:
  - "PRODUCT_MASTER"
  - "DC_PLPN_LEVEL"
  - "DC_CLPN_COUNT"
  - "DC_CLPN_DEFAULT_QUANTITY"
  - "GS1_IN_USE"
  - "DC_SSCC_COUNTER"
  - "DC_C_NESTED_CASE_TRANSFER"
  - "ASN_LOAD_CARRIERS"
  - "SOP_S_L_SHIP_CONFIRM"
---

Your instinct is right: **LPN nesting is the vanilla Ross mechanism for putting a unique SSCC on each case.** That's precisely what parent/child license plates are for — a pallet plate on top, a case plate underneath for each carton, and every plate carries its own SSCC. When you confirm the shipment, those case plates become the case-level load carriers *under* the pallet in the 856 ASN. So you've reached for the correct tool.

The catch — and it's the whole substance of your remaining four questions — is **where the switch lives.** Nesting is a property of the **product**, not the customer. Ross has no "nest for this customer only" flag anywhere in the standard data model. That single fact drives the answers to restriction, product scoping, and downstream impact, so most of this note is about working *with* that constraint rather than against it.

## What "LPN nesting" actually builds

A license plate (LPN) in Ross is a tracked, serial-bearing container — and in a GS1 shop, that serial *is* an SSCC. Nesting simply lets one plate be the **parent** of others:

- **Parent LPN** — the pallet. One SSCC for the whole pallet.
- **Child LPNs** — the cases on that pallet. One SSCC each.

The parent↔child link is a single field on the license-plate record, `DC_PARENT_LICENSE_PLATE_ID`. Every plate — parent or child — draws its own SSCC from the warehouse serial counter, so "a unique SSCC on every case" falls straight out of the model: each case is its own child plate, each child plate its own SSCC. Nothing about that is customer-specific; it's a physical packaging structure.

> **in plain terms** — nesting doesn't "add barcodes to cases." It makes each case a real, serialized container in Ross, and the SSCC is what that container is called. The pallet is the parent container; the 856 later reports the cases *inside* it.

## Where you turn it on — the Product Master, not the customer

This is the pivot for everything you asked. The nesting configuration is four fields, and they all sit on the **Product Master** (`PRODUCT_MASTER`) — a product-level, company-wide record. There is no warehouse and no customer in this setup:

```screen
title: LPN Nesting
program: SYS_M_PRODUCT_MASTER
facility: SYS_M_048B
toc: Inventory Control > Master > Product Master — LPN Nesting
context: Product = FG1000 | Description = Shared finished good | Company = 01
form:
  PLPN Entry at {lov} = 2 — End  ! PRODUCT_MASTER.DC_PLPN_LEVEL {WORD} — nesting mode: 0 Not in Use / 1 Beginning / 2 End
  No. of CLPNs = 12  ! PRODUCT_MASTER.DC_CLPN_COUNT {WORD} — child (case) plates per parent; capped by system param DC_MAX_CLPNS
  CLPN Default Quantity = 24.000  ! PRODUCT_MASTER.DC_CLPN_DEFAULT_QUANTITY {INVENTORY_QTY} — default units per each case plate
  Serialization in Use = Yes  ! PRODUCT_MASTER.GS1_IN_USE {YES_NO} — GS1/SSCC serialization on/off for this product
```

- **PLPN Entry at** (`DC_PLPN_LEVEL`) — the on/off + mode switch. Domain is `0 / 1 / 2`: **0 = Not in Use** (no nesting), **1 = Beginning**, **2 = End**. The two non-zero modes select *where in the LPN build* the parent (pallet) plate is introduced — beginning vs. ending level — but for your purposes the load-bearing distinction is simply **0 vs. non-zero**: non-zero means this product nests.
- **No. of CLPNs** (`DC_CLPN_COUNT`) — child plates per parent, i.e. cases per pallet.
- **CLPN Default Quantity** (`DC_CLPN_DEFAULT_QUANTITY`) — the default quantity per child plate, i.e. units per case.
- **Serialization in Use** (`GS1_IN_USE`) — whether GS1 serialization (the SSCC/GTIN machinery) applies to this product.

Here's the field itself, straight from the metadata — note the domain and that its default is off:

```dml
@program FIN_FIELDS
@note THE NESTING MODE IS A PER-PRODUCT CONTROL FLAG — OFF BY DEFAULT
@highlight 6-8
ADD FIELD DC_PLPN_LEVEL
    /DATATYPE=WORD
    /DESCRIPTION="Indicates at which level PLPN is applied."
    /PROMPT="PLPN Entry at"
    /HELP="0 - Nested LPN is not in use"
    /HELP="1 - PLPN is used at beginning level"
    /HELP="2 - PLPN is used at ending level"
    /DEFAULT="0"
    /DOMAIN='"0","1","2"'
    /INITIAL_VALUE="0"
```

And here is the part that answers "can we restrict it to a single customer" at the schema level — the three fields are added to `PRODUCT_MASTER`, alongside ordinary product attributes, with **no customer key in sight**:

```dml
@program FIN_TABLES
@note THE NESTING FIELDS HANG OFF THE PRODUCT MASTER — NOT THE CUSTOMER
@reads product_master
@risk there is no customer-scoped variant of these fields anywhere in vanilla Ross
@highlight 2-4
   ADD TABLE PRODUCT_MASTER
       /ADD_FIELD=DC_PLPN_LEVEL
       /ADD_FIELD=DC_CLPN_COUNT
       /ADD_FIELD=DC_CLPN_DEFAULT_QUANTITY
       /ADD_FIELD=GS1_IN_USE
```

The customer master (`sop_s_l_customer.dml` / `CUSTOMERS`) carries **zero** references to PLPN, CLPN, nesting, SSCC, case labels, or GS1. There is simply no place to hang "this customer wants case SSCCs." That's not an oversight you can toggle around — it's the shape of the model.

## The limits you *can* put on it

Within the product-level setup, you get real, enforced controls — this is the honest answer to "what limitations can we put on LPN nesting":

| Control | Field / param | What it bounds |
| --- | --- | --- |
| Cases per pallet | `DC_CLPN_COUNT` ("No. of CLPNs") | How many child plates a parent gets |
| Hard ceiling on cases | system param `DC_MAX_CLPNS` | Upper bound the entry is clamped to |
| Units per case | `DC_CLPN_DEFAULT_QUANTITY` | Default quantity on each child plate |
| Whether it nests at all | `DC_PLPN_LEVEL` = 0 | Off switch, per product |
| Whether SSCCs are drawn | `GS1_IN_USE` | GS1 serialization on/off, per product |

The `DC_MAX_CLPNS` ceiling is enforced in two places — both the Product Master entry and the nested case transfer clamp to it, so you cannot exceed it by any path:

```dml
@program DC_C_NESTED_CASE_TRANSFER
@note NO. OF CLPNS IS CLAMPED TO THE SYSTEM MAXIMUM, AND NESTING IS REQUIRED
@risk if the product's PLPN level is 0, the nested case transfer refuses to run
@highlight 1-2
IF (#DC_CLPN_COUNT > PARAMETER("DC_MAX_CLPNS"))
    #DC_CLPN_COUNT = PARAMETER("DC_MAX_CLPNS")
END_IF
...
IF (#DC_PLPN_LEVEL = 0)
    MESSAGE/BELL P_36550        ! "Nested LPN is not in use."
    GOTO PART_CODE_OR_EAN
END_IF
```

That second guard is the important one for scoping: **nested case transfer is only available for products whose `DC_PLPN_LEVEL` is non-zero.** A product left at 0 can't be nested at all. So the product flag *is* your master enable — it's just not a customer flag.

## Where each SSCC number comes from

When a plate is created, its SSCC is pulled from a **per-warehouse** counter, `DC_SSCC_COUNTER`, keyed by company + warehouse, and (if the standard-format flag is set) finished with the GS1 18-digit check digit:

```dml
@program DC_C_LPN_PRINT
@note EACH PLATE DRAWS THE NEXT SSCC FROM THE PER-WAREHOUSE COUNTER
@reads dc_sscc_counter
@risk the counter is company+warehouse scoped, not customer scoped — every plate consumes one
@highlight 1-3
FIND IN DC_SSCC_COUNTER /LOCK=WRITE &
    /WITH=COMPANY_CODE=#COMPANY_CODE &
    /WITH=WAREHOUSE=#WAREHOUSE
...
#LPN_ROOT_NUMBER = DC_SSCC_COUNTER(DC_NEXT_SSCC_NUMBER)
IF (DC_SSCC_COUNTER(DC_USE_SSCC_STANDARD_FORMAT))
    PERFORM "GEMDC:DC_V_COMMON" DC_GET_CHECK_DIGIT ((#LPN_ROOT_NUMBER), (18), #CHECK_DIGIT)
    #NEXT_LPN = (#LPN_ROOT_NUMBER & #CHECK_DIGIT)
END_IF
```

Two consequences worth internalizing: the counter is **shared across all shipments out of that warehouse** (so every nested product, for every customer, draws from the same pool — plan the SSCC company prefix / range accordingly), and the case-building step also stamps each case with its own GS1 serial as it goes:

```dml
@program DC_C_NESTED_CASE_TRANSFER
@note EACH CASE IS STAMPED WITH ITS OWN GS1 SERIAL AS IT IS BUILT
@highlight 1
DC_CASE_TRANSFER(DC_CASE_SERIAL_NUMBER) = #GS1_SN
```

## How the case SSCCs reach the 856 ASN

At ship confirm, `LOAD_ASN_LOAD_CARRIERS_DC` turns each shipped plate into an ASN load carrier. The qualifier is literally `"SSCC"`, and the child case points at its parent pallet via `ASN_PARENT_LOAD_CARRIER_ID` — that parent/child pair *is* the pallet→case hierarchy your customer's 856 is asking for:

```dml
@program SOP_S_L_SHIP_CONFIRM
@note SHIP CONFIRM BUILDS THE SSCC PALLET→CASE HIERARCHY FOR THE 856
@reads asn_load_carriers
@risk the parent link is what makes cases nest under the pallet in the ASN — it comes straight from the LPN
@highlight 3-6
CLEAR_BUFFER ASN_LOAD_CARRIERS
ASN_LOAD_CARRIERS(ASN_ID)                     = #ASN_ID
ASN_LOAD_CARRIERS(ASN_LOAD_CARRIER_ID)        = #ASN_LOAD_CARRIER_ID
ASN_LOAD_CARRIERS(ASN_LOAD_CARRIER_QF)        = "SSCC"
ASN_LOAD_CARRIERS(ASN_LOAD_CARRIER_TYPE)      = #DC_ASN_LOAD_CARRIER_TYPE
ASN_LOAD_CARRIERS(ASN_PARENT_LOAD_CARRIER_ID) = #DC_PARENT_LICENSE_PLATE_ID
ADD TO ASN_LOAD_CARRIERS
```

So the pipeline end to end is: **product enabled for nesting → cases built as child plates (nested case transfer) → each plate draws an SSCC → ship confirm emits the SSCC load-carrier hierarchy → the 856 reports pallet-with-cases.** Every stage keys off the product and the physical build. None of it keys off the customer.

## Answering the four scoping questions directly

Here's each of your questions against what the source actually supports:

| Your question | Answer | Why |
| --- | --- | --- |
| Good application for LPN nesting? | **Yes** | It's the native, purpose-built path to case-level SSCCs and the 856 pallet→case hierarchy. |
| Can we restrict it to a single customer? | **No — not in the nesting setup** | Nesting/SSCC/GS1 are Product Master fields; the customer master has no such field. There is no customer-level nesting switch. |
| Can we restrict it to that customer's products only? | **Only in the sense that it's product-scoped** | You enable it on specific products — but that enables it for those products *globally*, for every customer, not just this one. |
| Downstream impact for other customers on those shared products? | **Real — see below** | A shared product that's nest-enabled nests everywhere it's built into cases. |

## The downstream catch for shared products

This is the question that matters most for you, because you said the products aren't customer-specific. Turn on `DC_PLPN_LEVEL` for a shared product and here's what changes **for that product across the board**:

- **Nested case transfer becomes available and expected** for that product in the warehouse. The per-product guard (`DC_PLPN_LEVEL = 0 → refuse`) now passes, so operators *can* build parent/child structures for it regardless of who the order is for.
- **Any nested build draws SSCCs** from the shared per-warehouse counter — consuming serial numbers whether or not the destination customer wanted them.
- **If `GS1_IN_USE` is on for the product, its cases serialize** wherever they're built — the flag is on the product, so it doesn't know which customer is downstream.
- **At ship confirm, any shipment that contains nested plates emits the SSCC pallet→case hierarchy into the 856** — so a *different* customer who receives an ASN for that same product could suddenly get a case-level load-carrier loop they didn't ask for, and whose map may reject it.

> **caution** — there is no system guard that says "nest this product only when it's going to Customer X." The flag is global to the product. Whether a given shipment ends up nested is then a matter of *how the warehouse builds the LPNs* — which is procedure, not a Ross rule. Relying on operators to "only nest for the right customer" is a control you enforce with process and training, not with a switch Ross will hold for you.

## So what should you actually do

The clean pattern keeps the *configuration* product-scoped (because that's the only place it lives) while keeping the *customer-facing output* customer-scoped (because that's where Ross does have per-customer control — EDI):

```steps
1 | Enable nesting + serialization on the specific products
where: sys_m_product_master — LPN Nesting
do: set PLPN Entry at to a non-zero mode, No. of CLPNs to your cases-per-pallet, CLPN Default Quantity to units-per-case, and Serialization in Use = Yes — only on the products this customer actually orders.
sys: writes PRODUCT_MASTER(DC_PLPN_LEVEL / DC_CLPN_COUNT / DC_CLPN_DEFAULT_QUANTITY / GS1_IN_USE).

2 | Build cases as nested plates only for that customer's shipments
where: dc nested case transfer (Data Collection)
do: run nested case transfer to create the pallet parent + case children for orders going to the requesting customer; ship other customers' orders of the same product without nesting.
sys: writes DC_CASE_TRANSFER + child DC_LICENSE_PLATES with DC_PARENT_LICENSE_PLATE_ID set; each plate draws an SSCC from DC_SSCC_COUNTER.

3 | Gate the case-SSCC label and the 856 load-carrier loop by trading partner
where: the customer's EDI 856 / trading-partner setup
do: emit the case-level SSCC label and the SU/case load-carrier segments only for partners who require them; suppress them for everyone else.
sys: ASN_LOAD_CARRIERS is populated at ship confirm; the outbound 856 map decides which load-carrier levels are transmitted per partner.
```

That gives your EDI customer exactly what they asked for while leaving the *transmitted* result unchanged for everyone else — because the customer-aware decision (label + which ASN levels go out) is made in EDI, where per-customer control genuinely exists.

If you need **physical** isolation rather than procedural — cases *never* nest for anyone but this customer — the only structural lever vanilla Ross gives you is a **separate product (or pack) code** for the serialized version, so the nesting flag rides a code that only this customer's orders use. Be clear-eyed about the cost: it splits the item master, fragments on-hand and forecasting, and pushes complexity into order entry. For a product that isn't otherwise customer-specific, that's usually a worse trade than disciplined process plus EDI gating. Reach for it only if an auditor or the trading partner forbids the shared-product approach outright.

> **field note** — the mental model that keeps this straight: **nesting is a packaging capability of a product; the customer requirement is an EDI output.** Ross lets you configure the first per product and the second per customer. Trying to make the first behave per-customer is fighting the model; wiring the second to the customer's 856 map is using it as designed.

## Where this comes from

- `PRODUCT_MASTER(DC_PLPN_LEVEL / DC_CLPN_COUNT / DC_CLPN_DEFAULT_QUANTITY / GS1_IN_USE)` — the four per-product nesting/serialization fields; `DC_PLPN_LEVEL` domain is `0` (off) / `1` (beginning) / `2` (end), default `0`. Surfaced on the **LPN Nesting** form of the Product Master (`sys_m_product_master`, facility `SYS_M_048B`).
- `sys_m_product_master.dml` (FORM `LPN_NESTING`) — enforces `No. of CLPNs > 0` and `≤ DC_MAX_CLPNS`, and zeroes the child fields when `DC_PLPN_LEVEL = 0`.
- `dc_c_nested_case_transfer.dml` — clamps `DC_CLPN_COUNT` to `PARAMETER("DC_MAX_CLPNS")`, refuses to run when `DC_PLPN_LEVEL = 0`, and stamps each case's `DC_CASE_SERIAL_NUMBER`.
- `dc_c_lpn_print.dml` — draws the next SSCC from `DC_SSCC_COUNTER` (company + warehouse) and applies the 18-digit GS1 check digit when `DC_USE_SSCC_STANDARD_FORMAT` is set.
- `sop_s_l_ship_confirm.dml` (`LOAD_ASN_LOAD_CARRIERS_DC`) — writes `ASN_LOAD_CARRIERS` with qualifier `"SSCC"` and `ASN_PARENT_LOAD_CARRIER_ID` = the parent plate, forming the pallet→case hierarchy in the 856.
- `CUSTOMERS` / `sop_s_l_customer.dml` — no PLPN, CLPN, nesting, SSCC, or GS1 fields; confirms there is no customer-level nesting control.

One honest caveat, same as always: this was read from Ross 8.0 source and the mechanism is standard, though exact line numbers drift by patch level. The stable landmarks are the four `PRODUCT_MASTER` fields, the `DC_PLPN_LEVEL = 0` guard in nested case transfer, the `DC_SSCC_COUNTER` draw, and the `"SSCC"` / `ASN_PARENT_LOAD_CARRIER_ID` pair at ship confirm. Your `DC_MAX_CLPNS` value and whether `DC_USE_SSCC_STANDARD_FORMAT` is set are site parameters — check yours before you quote a cases-per-pallet ceiling.
