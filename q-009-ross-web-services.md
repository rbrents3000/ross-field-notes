---
num: "009"
title: "What actually are 'Ross web services' — and how does a call get from an outside system into GEMBASE and back?"
tag: "ROSS"
audience: "developer"
source: "group"
date: "2026-08-02"
system: "Ross ERP 8.0"
reading_time: "12 min"
excerpt: "Ross web services are the ~230 rs_* wrapper programs in the RS module — each one a published operation an outside system (a WMS, a planning engine, CRM, a 3PL) calls over XML. Every wrapper does the same three things: unpack the inbound XML into GEMBASE virtual tables with XMLPACK, hand off to the very same server-logic (_s_l_) routine the screens already use, then pack the result tables — always including RS_SYS_MESSAGES — back into XML. It always exits SUCCESS; whether the call worked rides back in the data, not the status."
question: "Can you look at the documentation and code for Ross web services and make a field note for what they are and how they work, examples, etc.?"
restated: "In standard Ross ERP 8.0, what are the built-in web services, how is one structured, and what is the request/response contract an external system follows to call one?"
fix: "A 'Ross web service' is a GEMBASE wrapper program in the RS module — there are ~230 of them in vanilla 8.0 (RS_GET_SOP_ORDERS, RS_WMS_PO_RECEIPT, RS_SCM_PRODUCT_MASTER, RS_TW_CREATE_POS, and so on). Each is a published operation an outside system calls through the web-services connect host that runs on the GEMBASE application server. The transport is XML, not raw SQL: the caller sends an XML document whose bodies are Ross virtual tables (VTs), and the wrapper uses the XMLPACK external library — RECEIVE_XML_TABLES to deserialize the request into VTs, SEND_XML_TABLE to serialize results back. The wrapper itself contains almost no business logic; it validates the inbound tables and then PERFORMs the exact same server-logic (_s_l_) procedure the interactive screens call — e.g. RS_GET_SOP_ORDERS calls SOP_S_L_ORDER, RS_WMS_PO_RECEIPT calls POP_S_L_WMS_PO_RECEIPT. Two hard rules make the contract work: (1) the wrapper ALWAYS EXIT(%SUCCESS) — if it fails the connect host returns nothing at all — so (2) real success/failure is reported in-band through the #ERROR_OCCURRED flag (0 ok / 1 fatal) and a returned RS_SYS_MESSAGES table, never through the call status. To call one you send the operation's request VT(s) as XML and read back its response VT(s) plus RS_SYS_MESSAGES."
margin_notes:
  - "'web service' = one rs_* wrapper program ↴"
  - "XMLPACK marshals XML ⇆ virtual tables →"
  - "the wrapper delegates to the screens' own _s_l_ routine"
  - "always EXIT(%SUCCESS) — errors ride in RS_SYS_MESSAGES →"
  - "read vs. write is just which VTs go in and come out"
reviewed: "2026-08-02"
verified: "Verified against Ross 8.0 source"
applies_when:
  - "You're integrating an external system — a WMS, an APS/planning engine, CRM, a 3PL, a label printer — with Ross and someone said 'call the Ross web service.'"
  - "You see procedures named RS_* (RS_GET_..., RS_WMS_..., RS_SCM_..., RS_TW_...) and want to know what they are and how they're invoked."
  - "A call 'succeeds' but you get no data back, or an error, and you're not sure where the status actually lives."
related:
  - "q-010-crystal-web-services"
key_refs:
  - "XMLPACK"
  - "RS_SYS_MESSAGES"
  - "SOP_S_L_ORDER"
  - "POP_S_L_WMS_PO_RECEIPT"
  - "SYS_S_L_ADD_SEND_TO_EXT_ERROR"
  - "GEM_SELECT_LIMIT"
---

"Ross web services" sounds like it should be some separate product bolted onto the side of the ERP. It isn't. It's a **module** — the `rs` module — full of ordinary GEMBASE programs that follow one very disciplined pattern. Learn that pattern once and all ~230 of them read the same way.

The short version: each web service is a thin **wrapper** that speaks XML on the outside and speaks Ross's own internal **server-logic** procedures on the inside. The wrapper's whole job is translation — turn an inbound XML document into Ross virtual tables, call the same routine a clerk's screen would call, and turn the results back into XML. The business logic isn't in the web service at all; it's the code the application already runs.

## What "a Ross web service" actually is

Three facts, and most of the confusion clears up:

- **It's a program in the RS module.** In vanilla 8.0 there are **~230** of them under `rs/` — `RS_GET_SOP_ORDERS`, `RS_WMS_GET_SHIP_NOTE`, `RS_WMS_PO_RECEIPT`, `RS_SCM_PRODUCT_MASTER`, `RS_TW_CREATE_POS`, `RS_CRM_ADD_CUSTOMER`, and so on. Each program **is** one published operation. The compiled form (`.dmc`) is what the runtime executes.
- **The wire format is XML carrying virtual tables.** A "virtual table" (VT) is Ross's in-memory table object. A request is an XML document whose contents are one or more named VTs (e.g. `RS_SOP_ORDER_SELECT_CRITERIA`); a response is an XML document of result VTs (e.g. `RS_SOP_ORDERS`) plus, always, a `RS_SYS_MESSAGES` table.
- **A connect host does the plumbing.** A web-services connect application, running against the GEMBASE application server, is the SOAP/HTTP endpoint the outside world talks to. It receives the request, launches the matching `RS_*` form on the server, streams the XML in, streams the response XML back out. Every wrapper's header calls it out by name — *"this wrapper must always exit with a %SUCCESS status, or the web connect application will return no information."*

> **in plain terms** — the "web service" is a translator standing at the door. Outside the door everything is XML over HTTP; inside the door everything is Ross virtual tables and the same server routines the screens use. The translator doesn't do the work — it just carries the message in, hands it to the department that does the work, and carries the answer back out.

> **in the system** — the wrapper is a `PROCEDURE_FORM` with a fixed three-argument signature. It links the `XMLPACK` external library, calls `RECEIVE_XML_TABLES` to populate VTs from the request, `PERFORM`s a `_s_l_` (server-logic) procedure to do the actual work, then calls `SEND_XML_TABLE` once per result table. That's the entire shape.

## The signature every wrapper shares

Open any `rs_*.dml` and the `PROCEDURE_FORM` line is the same three parameters:

| Parameter | Direction | Meaning |
| --- | --- | --- |
| `#ERROR_OCCURRED` | out | The real status. `0` (`#INFO`) = success, `1` (`#FATAL`) = failure. This — not the process exit status — is how the caller learns whether it worked. |
| `#XML_TAGS` | in | How much metadata the response XML should carry (the verbosity of the returned stream). |
| `#ROW_SELECT_COUNT` | in | Optional cap on rows returned, overriding the default `GEM_SELECT_LIMIT`. On many services this is reserved for future use. |

That uniformity is the point: the connect host invokes every operation the same way, and every operation answers the same way.

## The anatomy of one call

Strip a wrapper down and you get five blocks, in this order, every time:

```steps
1 | SETUP — become Ross, and open the envelope
do: point at the FIN database, assert a facility (security) code, read the standard message-severity parameters, then unpack the request.
sys: SET/LOCAL DATABASE FIN; PARAMETER("FIN.FACILITY_ID") = "RS_SOP_I_004"; EXTERNAL "XMLPACK" RECEIVE_XML_TABLES; CALL RECEIVE_XML_TABLES(#MSG) — the inbound XML is now sitting in virtual tables.

2 | CHECK_TABLES — validate the inbound VTs
do: confirm the request tables exist and are well-formed (e.g. exactly one header row, at least one line, all lines the same type). Bad input becomes a RS_SYS_MESSAGES entry, not a crash.
sys: START_STREAM over the request VT with /STATISTIC=#COUNT=COUNT; load a message via LB_S_L_LOAD_MESSAGE and EXIT(%FAILURE) on a bad shape.

3 | PROCESSING — hand off to the real routine
do: call the exact server-logic procedure the interactive program uses. The web service adds no business rules of its own.
sys: PERFORM "GEMSOP:SOP_S_L_ORDER" (a read) or PERFORM "GEMPOP:POP_S_L_WMS_PO_RECEIPT" (a post). Set #ERROR_OCCURRED = #FATAL if %STATUS isn't %SUCCESS.

4 | LOG — record the external touch
do: write an audit row so the interface is traceable from either side.
sys: PERFORM "GEMSYS:SYS_S_L_ADD_SEND_TO_EXT_ERROR" with a direction tag — "ROSS-WMS" when Ross is answering outward, "WMS-ROSS" when the outside system posted in.

5 | FINISH — pack the answer, and always succeed
do: serialize each result table (and RS_SYS_MESSAGES) into the response XML, commit, and exit SUCCESS no matter what happened.
sys: CALL SEND_XML_TABLE("RS_SOP_ORDERS", #XML_TAGS); CALL SEND_XML_TABLE("RS_SYS_MESSAGES", #XML_TAGS); EXIT(%SUCCESS).
```

## A read service, end to end

`RS_GET_SOP_ORDERS` is the cleanest example — its own header documents the whole contract. You send it selection criteria; it returns matching sales orders. Note what it says it does: it just calls `SOP_S_L_ORDER` with action code 1 — the same server routine order inquiry uses.

```dml
@program RS_GET_SOP_ORDERS.DML
@note THE WEB SERVICE'S OWN HEADER DOCUMENTS THE CONTRACT — IN VT, OUT VTs, THE DELEGATE, AND THE ALWAYS-SUCCEED RULE
@highlight 2-9
! Desc: This procedure is the wrapper program for the web service RS_GET_SOP_ORDERS.
! Input Table:
!     RS_SOP_ORDER_SELECT_CRITERIA
! Out Table:
!     RS_SYS_MESSAGES
!     RS_SOP_ORDERS
! It calls SOP_S_L_ORDER with the ACTION_CODE 1.
! This wrapper must always exit with a %SUCCESS status, or the web
! connect application will return no information.
```

So the request/response is just: **one criteria table in, one data table + one messages table out.** In XML terms, the shape a caller sends and receives looks like this (company-neutral placeholders — the exact element set depends on `#XML_TAGS`):

```dml
@program REQUEST → RS_GET_SOP_ORDERS
@note THE REQUEST BODY IS SIMPLY THE INPUT VIRTUAL TABLE, RENDERED AS XML
! <RS_SOP_ORDER_SELECT_CRITERIA>
!   <ROW>
!     <COMPANY_CODE>01</COMPANY_CODE>
!     <ORDER_FROM_DATE>2026-08-01</ORDER_FROM_DATE>
!     <ORDER_TO_DATE>2026-08-02</ORDER_TO_DATE>
!   </ROW>
! </RS_SOP_ORDER_SELECT_CRITERIA>
```

```dml
@program RESPONSE ← RS_GET_SOP_ORDERS
@note THE RESPONSE IS THE OUTPUT VTs — DATA PLUS THE MANDATORY MESSAGES TABLE
! <RS_SOP_ORDERS> ...one ROW per matching order... </RS_SOP_ORDERS>
! <RS_SYS_MESSAGES>
!   <ROW><RS_MESSAGE_SEVERITY>0</RS_MESSAGE_SEVERITY>
!        <RS_MESSAGE_TEXT>Completed successfully</RS_MESSAGE_TEXT></ROW>
! </RS_SYS_MESSAGES>
```

The `#ERROR_OCCURRED` flag comes back `0`, and `RS_SYS_MESSAGES` carries no severity-1 rows. That's a successful read.

## A write service, end to end

Posting a transaction in works the same way, just with the arrow reversed — the outside system fills the request VT with data to apply. `RS_WMS_PO_RECEIPT` takes receipt lines from a warehouse system and turns them into Ross receipts. Watch the delegation: the wrapper reads the inbound `RS_WMS_PO_RECEIPT_LINES`, decides what kind of receipt it is, and calls the matching core routine — the same ones purchasing, sales returns, and warehouse transfers use.

```dml
@program RS_WMS_PO_RECEIPT.DML
@note THE WRAPPER RECEIVES XML → VALIDATES → ROUTES BY TYPE TO THE CORE _s_l_ ROUTINES → SENDS XML BACK
@reads rs_wms_po_receipt_lines
@writes po_receipts / customer_returns / wh_transfer_receipts (via the delegates)
@risk the wrapper adds no receiving logic of its own — it reuses the application's routines
@highlight 3,6,9,12
CALL RECEIVE_XML_TABLES(#MSG)            ! inbound XML → the RS_WMS_PO_RECEIPT_LINES virtual table
...
PERFORM CHECK_TABLES                     ! at least one line? all lines the same WMS PO type?
...
BEGIN_CASE (#WMS_PO_TYPE)
    CASE (31)  PERFORM "GEMPOP:POP_S_L_WMS_PO_RECEIPT"    ! a PO receipt
    CASE (81)  PERFORM "GEMSOP:SOP_S_L_WMS_RETURN_RECEIPT" ! a customer return
    CASE (60)  PERFORM "GEMIC:IC_L_WH_XFER_RECV_DRIVER"    ! a warehouse-transfer receipt
END_CASE
...
CALL SEND_XML_TABLE("RS_SYS_MESSAGES", #XML_TAGS)   ! response = just the status/messages
EXIT(%SUCCESS)                                       ! ...always
```

A write service typically returns only `RS_SYS_MESSAGES` — the caller needs to know *whether it posted*, not a data set. If `#ERROR_OCCURRED` comes back `1`, the transaction rolled back and the reason is in the messages table.

> **field note** — the reason a web service can safely reuse a screen's routine is that the `_s_l_` ("server logic") procedures were built to be UI-independent in the first place. The screen is one caller; the web service is another. Both hand the same routine the same virtual tables. That's why the integration surface is *wide* (hundreds of operations) without being *deep* (almost no logic lives in `rs/`).

## The contract, stated plainly

Two rules govern every operation, and they're the ones that trip people up:

> **caution** — a Ross web service **always exits SUCCESS**, even when the business operation failed. If the wrapper itself exited with failure, the connect host would treat the call as dead and return *nothing* — no data, no error, just silence. So failure is deliberately moved *into the payload*.

That means the only correct way to check a result is:

- [ ] Read `#ERROR_OCCURRED` — `0` = the operation succeeded, `1` = it failed.
- [ ] Inspect the returned `RS_SYS_MESSAGES` table — any row with `RS_MESSAGE_SEVERITY = 1` is a fatal error; lower severities are informational.
- [ ] **Never** infer success from the fact that the call "returned 200" / completed — it always does.

`RS_SYS_MESSAGES` is the universal envelope: every service clears it on entry, loads messages into it as it runs (via `LB_S_L_LOAD_MESSAGE`), and ships it back in the response. It's the one table you can count on being present in every reply.

## What's actually published — the catalog

The `rs/` module isn't a random pile; it's organized by the external system it serves. In vanilla 8.0 the families break down roughly like this:

| Family | Prefix | ~Count | What it's for |
| --- | --- | --- | --- |
| Planning / APS | `rs_scm_*` | ~48 | Feed an advanced-planning engine: product master, demand, forecasts, locations, work/transfer orders. Mostly outbound extracts. |
| Warehouse (WMS) | `rs_wms_*` | ~32 | Two-way WMS integration: get ship notes / PO data / customer data out; post receipts, ship confirms, adjustments back in. |
| Generic reads | `rs_get_*` | ~28 | Query services — orders, invoices, customers, products, balances, lot numbers, requisitions. |
| Document retrieval | `rs_retrieve_print_*` | ~17 | Hand a document (BOL, pick, PO, invoice, GRN, checks) to an external print/output service. |
| Third-party / EAM | `rs_tw_*` | ~11 | Create POs, GRNs, AP invoices, journal entries, vendors, products for a linked third-party system. |
| Print flagging | `rs_flag_*` | ~11 | Mark documents as printed once an external printer has them. |
| CRM | `rs_crm_*` | ~4 | Sync customers, addresses, and product lists with a CRM. |
| RF / data collection | `rs_rf_export_*` | ~4 | Export jobs, products, allocations, test requirements to RF/mobile. |
| Everything else | mixed | — | Project accounting (`rs_pa_*`), production/labor (`rs_job_*`, `rs_labor_*`, `rs_pm_*`), tax reporting (`rs_sii_*`), inventory moves, and more. |

Two conventions worth internalizing from that list:

- **Verb tells direction.** `get` / `retrieve` / `_export_` pull data *out* of Ross; `create` / `receipt` / `confirm` / `increase` / `update` push transactions *in*. The audit tag in the LOG step matches — `"ROSS-WMS"` for outbound answers, `"WMS-ROSS"` for inbound posts.
- **The VT name is the payload name.** A service's input and output tables are named for the service (`RS_WMS_PO_RECEIPT_LINES`, `RS_SOP_ORDERS`, `RS_SCM_PRODUCT_MASTER`). If you know the operation, you know what tables its XML will contain.

## How you'd actually call one

From the integrating system's side, the flow is ordinary web-services work — the Ross-specific part is only *what goes in the body*:

```steps
1 | Discover the operation
do: get the endpoint and operation name from the connect host's published services. The operation name is the RS_* program name.
note: which operations are exposed is a deployment/licensing setting on the connect host — the module ships hundreds; a given site publishes the subset it uses.

2 | Build the request document
do: render the operation's input virtual table(s) as XML — one <ROW> per record, columns as child elements. For a read that's the selection-criteria table; for a write it's the data table (e.g. the receipt lines).
sys: this is exactly what RECEIVE_XML_TABLES will deserialize on the Ross side.

3 | Invoke and read the response
do: POST the request; read back the response document — the operation's output table(s) plus RS_SYS_MESSAGES.
sys: each table you get was written by a CALL SEND_XML_TABLE on the Ross side.

4 | Check status the Ross way
do: treat #ERROR_OCCURRED = 1 or any RS_SYS_MESSAGES row at severity 1 as a failure — regardless of the transport-level result.
note: the call itself will report success even on a business failure. Always look in the payload.
```

> **rule of thumb** — if you can call one Ross web service you can call them all. Pick the operation, shape the input VT as XML, send, then read `#ERROR_OCCURRED` + `RS_SYS_MESSAGES`. The only thing that changes service to service is which virtual tables appear in the request and the reply.

## Where this comes from

- `rs/src/rs_get_sop_orders.dml` — the reference read service; its header documents the in/out virtual tables (`RS_SOP_ORDER_SELECT_CRITERIA` → `RS_SOP_ORDERS` + `RS_SYS_MESSAGES`), the delegate (`SOP_S_L_ORDER`, action 1), and the always-`%SUCCESS` rule.
- `rs/src/rs_wms_po_receipt.dml` — the reference write service; receives `RS_WMS_PO_RECEIPT_LINES`, routes by WMS PO type to `POP_S_L_WMS_PO_RECEIPT` / `SOP_S_L_WMS_RETURN_RECEIPT` / `IC_L_WH_XFER_RECV_DRIVER`, returns `RS_SYS_MESSAGES`.
- `rs/src/rs_wms_get_ship_note.dml` — a second read service; shows the same skeleton (`RECEIVE_XML_TABLES` → `CHECK_TABLES` → `SYS_S_L_WMS_GET_SHIP_NOTE` → several `SEND_XML_TABLE` calls) and the "must exit %SUCCESS or Connect returns no information" note verbatim.
- `XMLPACK` external library — `RECEIVE_XML_TABLES(DSTRING)` deserializes the request XML into virtual tables; `SEND_XML_TABLE(DSTRING, LONG)` serializes a result table into the response. Linked via `EXTERNAL "XMLPACK" ...` in every wrapper.
- `RS_SYS_MESSAGES` — the standard response/status table; `RS_MESSAGE_SEVERITY = 1` marks a fatal error. Present in every service's reply.
- `SYS_S_L_ADD_SEND_TO_EXT_ERROR` — the external-interface audit hook every wrapper calls in its FINISH block, tagged with the traffic direction.
- The three-argument signature `(#ERROR_OCCURRED, #XML_TAGS, #ROW_SELECT_COUNT)` and the `GEM_SELECT_LIMIT` row cap — identical across the `rs/` module.

One honest caveat, as always: this is read from standard Ross 8.0 source, and the mechanism is vanilla, but the published operation list and endpoint are **deployment settings** — a given site exposes only the services it has turned on, and the copyright banners across the module span the Ross Systems / CDC / Aptean eras, so a specific wrapper's exact validation lines drift by patch level. The stable landmarks are the ones above: the `RS_*` wrapper shape, `XMLPACK` in/out, delegation to a `_s_l_` routine, the always-`%SUCCESS` exit, and status carried in `#ERROR_OCCURRED` + `RS_SYS_MESSAGES`.
