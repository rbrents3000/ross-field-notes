---
num: "002"
title: "Crystal Web Services"
subtitle: "The Crystal print path isn't a SOAP service you call — it's a report engine you reach by opening a URL. One dispatch switch, two staging tables, and a URL maker that welds them into an address."
tag: "CRYSTAL"
audience: "developer"
date: "2026-08-03"
system: "Ross ERP 8.0"
reading_time: "10 min"
excerpt: "What the Crystal 'web service' print path actually is and how one print flows end to end: a report set to process type Crystal loads its controls and this run's arguments into two virtual tables, LB_S_L_CRYSTAL_URL_MAKER welds them into one HTTP query string on the GEM_CRYSTAL_URL base, and a small web app on the Crystal server renders the .rpt and calls back into Ross for the data. The box is opaque only until you read the query string."
stats:
  - k: "Print engines"
    v: "4"
    note: "iRen · Crystal · DC · RRS"
  - k: "Crystal is"
    v: "type 1"
    note: "SYS_REPORT_PROCESS_TYPE"
  - k: "Transport"
    v: "URL"
    note: "HTTP query string, no SOAP"
  - k: "Renders"
    v: "Off-box"
    note: "Crystal server, not GEMBASE"
related:
  - "q-010-crystal-web-services"
  - "g-001-ross-web-services"
key_refs:
  - "REPORT_PRINT_CONTROLS"
  - "SYS_REPORT_PROCESS_TYPE"
  - "LB_S_L_CRYSTAL_URL_MAKER"
  - "SYS_WS_PRINT_CONTROLS_VT"
  - "URL_PARAMS_VT"
  - "GEM_CRYSTAL_URL"
  - "GEM_CONNECT_APP"
reviewed: "2026-08-03"
verified: "Verified against Ross 8.0 source"
---

> This is the plain-Markdown copy of a Field Guide. The formatted version, with the
> round-trip diagram and light/dark reading view, lives at
> `fieldnotes.ryanbrents.com/g/g-002-crystal-web-services`.

"Crystal Web Services" is a grand name for a modest idea: **a report engine you reach by
opening a URL.** There is no `SOAP`, no `WSDL`, no `.asmx` you invoke from DML. When a
report is configured as the Crystal engine, Ross loads that report's controls and this
run's arguments into two staging tables, then `LB_S_L_CRYSTAL_URL_MAKER` welds them into
one long HTTP query string and hands it to a browser. A small web app on the Crystal
server renders the template and, when it needs data, calls back into Ross. The "black box"
is opaque only until you read the query string — it spells out every input on the way in.

- **One switch picks the engine.** A single field on `REPORT_PRINT_CONTROLS` —
  `SYS_REPORT_PROCESS_TYPE` — decides which of four print engines runs. Crystal is value `1`.
- **Two tables stage the request.** `SYS_WS_PRINT_CONTROLS_VT` carries the report's
  identity; `URL_PARAMS_VT` carries this run's arguments as ready-made `KEY=VALUE` fragments.
- **The URL maker just builds a string.** No networking — it concatenates the base URL, the
  controls, and the params into one address and lets the session open it.

## One switch picks the engine

Every printable document in Ross — BOL, invoice, PO, check, COA, packing list — is produced
by whichever engine `SYS_REPORT_PROCESS_TYPE` names on that report's `REPORT_PRINT_CONTROLS`
row. One value renders natively inside Ross; the other three print by handing a URL to an
off-box web app.

| Value | Engine | What it is |
| --- | --- | --- |
| 0 | iRen | The native engine — a GEMBASE report renders a character report inside Ross. The default. |
| 1 | Crystal | The subject here — a Crystal Reports web app renders a `.rpt` template to PDF, driven over a print URL. |
| 2 | DC | Data Collection — the RF / warehouse label route (LPN and sub-LPN labels). Same handoff, aimed at label stock. |
| 3 | RRS | Ross Reporting Services — the SSRS-backed path. Same URL idea, a different renderer. |

The same document can print more than one way; moving a report from iRen to Crystal is a
config flip on this one field, not a code change. When the field is blank, the dispatcher
falls back to **iRen (0)**.

## One print, end to end

1. **A report program requests a print** — e.g. `sop_r_bol_print` — and calls the dispatcher.
2. **The dispatcher reads the switch** — `SYS_REPORT_PROCESS_TYPE = 1` → the Crystal branch.
3. **The URL maker builds the URL** — `LB_S_L_CRYSTAL_URL_MAKER` stages the two virtual
   tables, welds `GEM_CRYSTAL_URL` + controls + params into one query string, and the session
   opens it with `OPEN_URL`.
4. **The print web app receives it** — on the Crystal server, off-box, not in GEMBASE. It
   parses the URL and renders the `.rpt` template to PDF.
5. **It calls home for the data** — opening a Ross Connect session (named by
   `gem_connect_app`, running as `gembase_user`) and executing `sys_stored_procedure` — the
   same proc iRen would use — to pull the rows.
6. **It routes the output** — print to `report_queue` × `report_copies`, e-mail, save to the
   output directory, or upload to SharePoint; an optional second stored procedure marks the
   documents printed.

Ross gathers the data and describes the job; Crystal draws the picture and delivers it.

> **field note** — on a current `IAF Desktop` or `iBrowser` session the maker returns the URL
> and the session opens it directly. Only the old `GTC` / `PORTAL` thin-client sessions —
> which can't open a URL on their own — use a second form that writes the string into a
> `.bat` and runs it client-side. Modern IAF Desktop deployments never touch that path.

## Two tables get filled first

Before the URL maker runs, the calling program stages the request across two in-memory
tables — one for the report's *identity*, one for *this run's arguments*.

**`SYS_WS_PRINT_CONTROLS_VT` — the report's identity.** Its standing configuration:
everything true *every* time you print it. Filled by `FILL_WS_VT`, which clears the table
and reloads it from the report's `REPORT_PRINT_CONTROLS` row — the stored-procedure name(s),
`.rpt` template, output directory, file type, e-mail, save / debug / run-as-service flags,
and the SharePoint flag.

**`URL_PARAMS_VT` — this run's arguments.** One row per argument, each a ready-made fragment
tagged with a `SEQUENCE`, streamed into `params=` in order:

```dml
! one row per argument, a ready-made fragment + a SEQUENCE
URL_PARAMS_VT(SYS_URL_PARAMETER) = "company_code=" & #CC & "&"
URL_PARAMS_VT(SEQUENCE)          = 1
STORE URL_PARAMS_VT
```

A second table, `URL_PARAMS_2_VT`, is used only when a second stored procedure runs during
the print. The report name, print queue, copies, and Ross user don't live in either table —
they travel as the four call parameters to the maker (`#P1`–`#P4`).

## The URL is the contract

```
{GEM_CRYSTAL_URL}?
  gem_connect_app=…&        ! call-home app (Ross Connect)
  sys_stored_procedure=…&   ! the proc that fetches the data
  params=…&                 ! URL_PARAMS_VT rows, in SEQUENCE order
  report_file_name=…&       ! the .rpt template to render
  report_queue=…& report_copies=…&
  gembase_user=…&           ! who the Connect session runs as
  …output directory · file type · email · save · debug…&
  iaf_environment=…         ! DEV output can't print on PROD
```

Read left to right, the query string *is* the API contract for the whole print job — every
input the renderer receives is spelled out.

| Parameter | Source |
| --- | --- |
| `gem_connect_app` | SCV `GEM_CONNECT_APP` — the app the web service calls back through Ross Connect |
| `sys_stored_procedure` | `SYS_WS_PRINT_CONTROLS_VT` — the main data proc |
| `params` | `URL_PARAMS_VT` rows, concatenated in `SEQUENCE` order |
| `report_file_name` | call parameter `#P1` — the `.rpt` to render |
| `report_queue` · `report_copies` | call parameters `#P2` · `#P3` |
| `gembase_user` | call parameter `#P4` — the Ross user (lower-cased on UNIX) |
| output directory, file type, extension, email, save, debug, run-as-service, template, `auto_upload_sharepoint` | `SYS_WS_PRINT_CONTROLS_VT` fields |
| `iaf_environment` | SCV `IAF_ENVIRONMENT` — so DEV output can't print on PROD |

> **rule of thumb** — if a Crystal report misbehaves, turn on `SYS_DEBUG_MODE`, print it, and
> read the URL. A blank report is almost always a bad `params=` or a proc that returned
> nothing; a mis-queued one is a bad `report_queue=`; a "prints in DEV, not PROD" is
> `iaf_environment=`.

## A quiet fallback

Two conditions route a report you *meant* for Crystal back through the native iRen engine
instead — silently, with no error. So "I set the report to Crystal, but it still prints the
old way" is usually not a Crystal fault at all; it's the dispatcher never choosing Crystal
in the first place:

- the `REPORT_PRINT_CONTROLS` row resolves a blank `SYS_REPORT_PROCESS_TYPE` — the dispatcher
  defaults to iRen (`0`);
- the session isn't web-capable — anything other than `IBROWSER GUI`, `IAF DESKTOP GUI`,
  `GTC`, or `PORTAL` is forced back to iRen before Crystal is ever considered.

## "It won't print" — where to look first

The URL maker fails loudly, with specific message codes. Match the code — or the symptom —
back to the knob that caused it.

| Symptom | What's happening | First thing to check |
| --- | --- | --- |
| `P_13293` · nothing prints | The base URL came back empty, so the maker exits before it builds anything. | SCV `GEM_CRYSTAL_URL` — it must hold the web app's base URL. |
| `P_91053` · "failed to start stream" | A staging table was empty when the maker ran. | The calling program — it must fill both tables (`FILL_WS_VT` plus the `URL_PARAMS_VT` rows) *before* calling the maker. |
| `P_13282` · "error creating the URL" | A wrapper error from the caller — the maker returned failure. A symptom, not the cause. | The more specific code the maker raised just before it (`P_13293` or `P_91053`) — that's the real reason. |
| `P_13283` · "environment unsupported" | The session type can't open a print URL at all, so the dispatcher refuses. | You're not on an interactive, web-capable session (e.g. a background / batch run). Web printing needs `IBROWSER GUI` / `IAF DESKTOP GUI` / `GTC` / `PORTAL`. |
| A separate window opens every print | Debug mode forces `OPEN_URL /NEW_WINDOW` instead of an in-place tab. | Field `SYS_DEBUG_MODE` on `SYS_WS_PRINT_CONTROLS_VT` — turn it off for normal printing. |
| Renders, but can't read Ross data | The web app renders the report but its call home through Ross Connect fails. | SCV `GEM_CONNECT_APP` — the `gem_connect_app` value must name a valid Connect app. |

Codes `P_13291` / `P_13292` / `P_13294` come only from the legacy `.bat` path — a missing
`GEM_BATCH_FILE`, or the client-side write helper failing — and appear only on `GTC` /
`PORTAL` sessions. A modern IAF Desktop session never hits them.

## Crystal vs its RRS sibling

Value `3` on the switch routes to `LB_S_L_RRS_URL_MAKER` — the newest of the four engines, a
later addition to the 8.0 line where the Crystal path is original-vintage code, so you'll
meet it only on more recent installs. It rhymes with the Crystal maker — same
`SYS_WS_PRINT_CONTROLS_VT`, same call-home through `GEM_CONNECT_APP` — but don't assume
they're interchangeable. It differs in four ways worth knowing.

| | Crystal | RRS |
| --- | --- | --- |
| Base URL | SCV `GEM_CRYSTAL_URL` | passed in as call parameter `#P5` (the RRS server URL) |
| Forms | two — a direct form and the legacy `.bat` form | one form, plus a `SUMMARY_PAGE_URL` wrapper |
| Run arguments | streams `URL_PARAMS_VT` into `params=` | `params=` is sent empty — it never reads the `URL_PARAMS` tables |
| Main procedure | from field `SYS_STORED_PROCEDURE` | from `SYS_GET_DOCUMENT_LIST`, plus a `document_list_name` |

Learn the Crystal maker and the *shape* transfers to all three web engines: stage the
controls, build a query string, get a browser to open it. Just don't assume the parameters
line up — RRS quietly sources several from different fields.

## Where this comes from

- `REPORT_PRINT_CONTROLS` — the per-report configuration table; `SYS_REPORT_PROCESS_TYPE`
  selects the engine (0 iRen / 1 Crystal / 2 DC / 3 RRS) and carries the stored procedures,
  the `.rpt` template, output settings, and routing flags.
- `LB_REPORT_PRINT_CONTROL` — the dispatcher that reads the process type; defaults to iRen
  when the type is blank or the session isn't web-capable.
- `LB_S_L_CRYSTAL_URL_MAKER` — the URL maker. Base address from SCV `GEM_CRYSTAL_URL`; the
  call-home app from SCV `GEM_CONNECT_APP`; environment from SCV `IAF_ENVIRONMENT`.
- `LB_S_L_RRS_URL_MAKER` — the Reporting Services sibling; base URL as a call parameter,
  `params=` empty, main procedure from `SYS_GET_DOCUMENT_LIST`.

This is standard Ross 8.0 behaviour read from source; the mechanism is vanilla, though the
`.rpt` templates and the print web application itself live off-box on the Crystal server, so
their configuration (paths, Connect app, service account, environment) is site-specific and
not visible from the DML.
