---
num: "001"
title: "Ross Web Services"
subtitle: "The ~230 published operations an outside system calls to read from and write to Ross — XML in, XML out, run over the application's own server logic."
tag: "ROSS"
audience: "developer"
date: "2026-08-02"
system: "Ross ERP 8.0"
reading_time: "9 min"
excerpt: "What the rs_* web services are and how one call flows end to end: XML unpacked into virtual tables, delegated to the same server-logic routine the screens use, packed back to XML — and why the call always returns success even when it failed."
stats:
  - k: "Operations"
    v: "~230"
    note: "programs in the rs/ module"
  - k: "Transport"
    v: "XML"
    note: "⇄ Ross virtual tables"
  - k: "Gateway"
    v: "Connect"
    note: "on the GEMBASE app server"
  - k: "Logic lives"
    v: "Elsewhere"
    note: "in the screens' own routines"
related:
  - "q-009-ross-web-services"
key_refs:
  - "XMLPACK"
  - "RS_SYS_MESSAGES"
  - "SOP_S_L_ORDER"
  - "POP_S_L_WMS_PO_RECEIPT"
reviewed: "2026-08-02"
verified: "Verified against Ross 8.0 source"
---

> This is the plain-Markdown copy of a Field Guide. The formatted version, with the
> round-trip diagram and light/dark reading view, lives at
> `fieldnotes.ryanbrents.com/g/g-001-ross-web-services`.

"Ross web services" sounds like a bolt-on product. It isn't. It's a **module** — the `rs`
module — full of ordinary GEMBASE programs that all follow one disciplined pattern. Learn
that pattern once and all ~230 of them read the same way.

Each web service is a thin **wrapper** that speaks XML on the outside and Ross's own
internal **server-logic** routines on the inside. Its whole job is translation: turn an
inbound XML document into Ross **virtual tables**, call the same routine a clerk's screen
would call, then turn the results back into XML. The business logic isn't in the web
service at all — it's the code the application already runs.

## What a Ross web service actually is

- **It's a program, not a black box.** Each operation is one `RS_*` program —
  `RS_GET_SOP_ORDERS`, `RS_WMS_PO_RECEIPT`, `RS_SCM_PRODUCT_MASTER`, `RS_TW_CREATE_POS`,
  and so on. The compiled form is what the runtime executes.
- **The payload is virtual tables as XML.** A request is one or more named virtual tables
  (e.g. `RS_SOP_ORDER_SELECT_CRITERIA`); a response is result tables plus, always, an
  `RS_SYS_MESSAGES` status table.
- **A "Connect" gateway does the plumbing.** A web-services connect application on the
  GEMBASE application server is the SOAP/HTTP endpoint: it receives the request, launches
  the matching `RS_*` program, and streams the XML back.

## One round trip, end to end

1. **Outside system** builds the request table as XML and sends it.
2. **Connect gateway** (SOAP/HTTP on the app server) launches the operation.
3. **The `RS_*` wrapper** runs five blocks:
   1. **Unpack** — `RECEIVE_XML_TABLES` turns the request XML into virtual tables.
   2. **Validate** the inbound tables — right shape, right counts.
   3. **Delegate** — `PERFORM` the core `_s_l_` routine. *This is where the work happens.*
   4. **Log** — write an audit row tagged with direction (`"ROSS-WMS"` outbound,
      `"WMS-ROSS"` inbound).
   5. **Pack** — `SEND_XML_TABLE` writes each result table plus `RS_SYS_MESSAGES`, then
      `EXIT(%SUCCESS)`.
4. **Outside system** reads `#ERROR_OCCURRED` and `RS_SYS_MESSAGES` — the real status.

Every wrapper shares the same three-argument signature: `#ERROR_OCCURRED` (out: 0 = ok,
1 = fatal), `#XML_TAGS` (in: response metadata verbosity), and `#ROW_SELECT_COUNT` (in:
optional row cap over `GEM_SELECT_LIMIT`).

## The one rule that trips everyone

A Ross web service **always exits `%SUCCESS`**, even when the business operation failed. If
the wrapper itself exited with failure, the Connect gateway would treat the call as dead
and return *nothing* — no data, no error, just silence. So failure is deliberately moved
*into the payload*:

- Read `#ERROR_OCCURRED` — `0` = it worked, `1` = it failed.
- Inspect `RS_SYS_MESSAGES` — any row at `RS_MESSAGE_SEVERITY = 1` is a fatal error.
- **Never** infer success from "the call returned 200 / completed." It always does.

## A read, and a write

A **read** — send selection criteria in, get matching sales orders back:

```
IN   RS_SOP_ORDER_SELECT_CRITERIA
OUT  RS_SOP_ORDERS        (one row per order)
OUT  RS_SYS_MESSAGES      (status — always present)

PERFORM SOP_S_L_ORDER     (action code 1 — the app's own read)
```

A **write** — a warehouse posts receipt lines in, Ross applies them and routes by type:

```
IN   RS_WMS_PO_RECEIPT_LINES
OUT  RS_SYS_MESSAGES      (did it post? — status only)

CASE 31  PERFORM POP_S_L_WMS_PO_RECEIPT      (a PO receipt)
CASE 81  PERFORM SOP_S_L_WMS_RETURN_RECEIPT  (a customer return)
CASE 60  PERFORM IC_L_WH_XFER_RECV_DRIVER    (a transfer receipt)
```

A write typically returns only `RS_SYS_MESSAGES` — you need to know *whether it posted*,
not a data set. `#ERROR_OCCURRED = 1` means it rolled back.

## What's published — the catalog

The `rs/` module is organized by the external system it serves. Two tells: the **verb
signals direction** (`get`/`retrieve`/`export` pull data out; `create`/`receipt`/`confirm`/
`update` push transactions in), and the **table name matches the operation name**.

| Family | Prefix | ≈ | What it's for |
| --- | --- | --- | --- |
| Planning / APS | `rs_scm_*` | 48 | Feed an advanced-planning engine: product master, demand, forecasts, locations, work & transfer orders. Mostly outbound extracts. |
| Warehouse (WMS) | `rs_wms_*` | 32 | Two-way: get ship notes, PO & customer data out; post receipts, ship confirms, adjustments back in. |
| Generic reads | `rs_get_*` | 28 | Query services — orders, invoices, customers, products, balances, lots, requisitions. |
| Document retrieval | `rs_retrieve_*` | 17 | Hand a document (BOL, pick, PO, invoice, GRN, checks) to an external print/output service. |
| Third-party / EAM | `rs_tw_*` | 11 | Create POs, GRNs, AP invoices, journal entries, vendors, products for a linked system. |
| Print flagging | `rs_flag_*` | 11 | Mark documents as printed once an external printer has them. |
| CRM | `rs_crm_*` | 4 | Sync customers, addresses, and product lists with a CRM. |
| RF / data collection | `rs_rf_export_*` | 4 | Export jobs, products, allocations, test requirements to RF / mobile. |
| Everything else | mixed | — | Project accounting, production & labor, tax reporting, inventory moves, and more. |

## How you'd actually call one

1. **Discover the operation** — get the endpoint and operation name from the Connect
   gateway's published services. The operation name is the `RS_*` program name; which ones
   are exposed is a per-site setting.
2. **Build the request** — render the operation's input table(s) as XML, one `<ROW>` per
   record. For a read that's the selection criteria; for a write it's the data.
3. **Invoke and read the response** — POST it; read back the output table(s) plus
   `RS_SYS_MESSAGES`.
4. **Check status the Ross way** — treat `#ERROR_OCCURRED = 1`, or any severity-1 message,
   as failure regardless of the transport result.

Rule of thumb: if you can call one, you can call them all. Only the tables in the request
and reply change from service to service.

## Where this comes from

- `rs/src/rs_get_sop_orders.dml` — reference read service; its header documents the in/out
  virtual tables and the delegate (`SOP_S_L_ORDER`, action 1).
- `rs/src/rs_wms_po_receipt.dml` — reference write service; routes by type to
  `POP_S_L_WMS_PO_RECEIPT` / `SOP_S_L_WMS_RETURN_RECEIPT` / `IC_L_WH_XFER_RECV_DRIVER`.
- `XMLPACK` external library — `RECEIVE_XML_TABLES` deserializes the request into virtual
  tables; `SEND_XML_TABLE` serializes a result table into the response.
- `RS_SYS_MESSAGES` — the standard status table; `RS_MESSAGE_SEVERITY = 1` marks a fatal
  error. Present in every reply.

This is read from standard Ross 8.0 source; the mechanism is vanilla, though the published
operation list and endpoint are per-deployment settings.
