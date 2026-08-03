---
num: "010"
title: "What are 'Crystal Web Services' in Ross, and what actually happens inside that black box when I print a Crystal report?"
tag: "CRYSTAL"
audience: "developer"
source: "form"
date: "2026-08-02"
system: "Ross ERP 8.0"
reading_time: "11 min"
excerpt: "Crystal 'web services' isn't a mystery service Ross talks to over SOAP — it's a print engine you point a browser at. Ross assembles a long query-string URL, hands it to a browser — opened directly in an IAF Desktop / iBrowser session — and a small web app on the Crystal server reads that URL, calls a Ross stored procedure back through Connect to fetch the data, renders a .rpt template to PDF, and prints/emails/files it. The 'black box' is opaque only until you read the query string — every parameter it needs is spelled out in plain text by LB_S_L_CRYSTAL_URL_MAKER, so the box tells on itself."
question: "Can you explain what 'Crystal Web Services' are in Ross — what they are, how they actually work end to end, and what's going on inside that black box? We treat it as magic that turns a menu pick into a PDF and nobody here can say what the moving parts are."
restated: "In standard Ross ERP 8.0, what is the Crystal Reports 'web service' print path, and what is the end-to-end mechanism — from a report menu pick to a finished PDF — that turns a document into a Crystal-rendered output?"
fix: "It's not a SOAP/WSDL service and there's nothing to 'call' — it's a browser-launched print URL. When a report is configured as process type Crystal, Ross loads that report's print controls (stored-procedure name, .rpt file, output type, queue, flags) into SYS_WS_PRINT_CONTROLS_VT, drops the run-time arguments into URL_PARAMS_VT as KEY=VALUE; rows, and then LB_S_L_CRYSTAL_URL_MAKER stitches all of it into one long HTTP query string built on the GEM_CRYSTAL_URL base. On a current IAF Desktop / iBrowser session it just OPEN_URLs the string; older GTC/PORTAL thin-client sessions instead write the URL into a .bat (rsiWriteFile.exe) and run rsiRunCrystal.bat. A small print web app on the Crystal/IIS server receives the URL, uses gem_connect_app to open a Ross Connect session as gembase_user, executes the named stored procedure with those params to pull the report data, feeds it to the Crystal runtime rendering the .rpt template, and then routes the result — print to report_queue × report_copies, email, save to the output directory, and/or upload to SharePoint. The box is opaque only until you read the query string: it hands you a fully labelled contract on the way in."
margin_notes:
  - "not SOAP — a browser-launched print URL ↴"
  - "process type lives on REPORT_PRINT_CONTROLS →"
  - "same retrieve-print proc feeds iRen and Crystal"
  - "the query string IS the API contract"
  - "the .rpt + the web app live off-box, not in GEMBASE"
reviewed: "2026-08-02"
verified: "Verified against Ross 8.0 source"
applies_when:
  - "A report is set to process type Crystal (not iRen) and prints as a polished PDF instead of a character report."
  - "You see rsiWriteFile.exe / rsiRunCrystal.bat fire, or a browser tab open to a long ?gem_connect_app=… URL, when you print."
  - "You're trying to trace why a Crystal document is blank, mis-queued, or not e-mailing, and need to know which moving part owns which step."
key_refs:
  - "REPORT_PRINT_CONTROLS"
  - "SYS_WS_PRINT_CONTROLS_VT"
  - "URL_PARAMS_VT"
  - "LB_S_L_CRYSTAL_URL_MAKER"
  - "LB_REPORT_PRINT_CONTROL"
  - "GEM_CRYSTAL_URL"
  - "GEM_CONNECT_APP"
  - "SYS_REPORT_PROCESS_TYPE"
---

Let's retire the phrase "black box" for a second, because it's doing you a disservice. A black box implies you can't see in. The Crystal print path is more like a vending machine with the glass front intact: you can watch the entire transaction, you're just not the one who restocks it. Ross does all its work *in front of* the glass — assembling a request in plain text — and then slides that request through the slot to a machine on another server. The only genuinely hidden part is the machine's internal wiring, and even that announces itself, because Ross has to spell out every button it's pressing.

So "Crystal Web Services" is a slightly grand name for a modest idea: **a report engine you reach by opening a URL.** No SOAP envelope, no WSDL, no `.asmx` you invoke from DML. Ross builds a web address, hands it to a browser, and a small web application on the Crystal server does the rest. Let's open the glass.

## First, the fork in the road: one switch picks the engine

Every printable document in Ross (BOL, invoice, PO, check, COA, packing list, and so on) is produced by whichever engine a single field on `REPORT_PRINT_CONTROLS` names. That field — `SYS_REPORT_PROCESS_TYPE` — has four possible values: one *native* engine that renders inside Ross, and three that print by handing a URL to an off-box web app:

```cards
0 | iRen | The classic engine — a native GEMBASE REPORT_FORM renders a character/columnar report right inside Ross. No browser, no external server.
1 | Crystal | The subject of this note — Ross builds a URL and a Crystal Reports web app renders a .rpt template to PDF off-box.
2 | DC | The Data Collection path — the RF/warehouse label route (LPN and sub-LPN labels). Same web-service handoff as Crystal, but aimed at label stock rather than documents.
3 | RRS | The SQL Server Reporting Services (SSRS) path — same URL idea, different renderer and a couple of extra parameters.
```

> **in plain terms** — the same document can print more than one way (iRen, Crystal, or RRS — the DC value is for warehouse labels), and which way is a configuration choice, not a code change. Flip the report's process type and the *layout* engine changes; the *data* usually doesn't.

The report driver asks `LB_REPORT_PRINT_CONTROL` which value is set, then branches to the matching service block — `CRYSTAL_SERVICE`, `IREN_SERVICE`, `SSRS_SERVICE`, or the DC path. Here's the routing logic, condensed — iRen is the `ELSE`, the fallback when nothing else matches:

```dml
@program LB_REPORT_PRINT_CONTROL.DML
@note THE ROUTER — READS THE REPORT'S PROCESS TYPE AND HANDS BACK WHICH ENGINE
@reads report_print_controls
@risk if no control row exists the code defaults to iRen — a Crystal report with a missing control silently prints the old way
@highlight 1,8
IF (REPORT_PRINT_CONTROLS(SYS_REPORT_PROCESS_TYPE) = PARAMETER("CRYSTAL_REPORT"))
    #R6 = PARAMETER("CRYSTAL_REPORT")   ! "1"
ELSE_IF (REPORT_PRINT_CONTROLS(SYS_REPORT_PROCESS_TYPE) = PARAMETER("DC_REPORT"))
    #R6 = PARAMETER("DC_REPORT")        ! "2" — Data Collection / RF labels
ELSE_IF (REPORT_PRINT_CONTROLS(SYS_REPORT_PROCESS_TYPE) = PARAMETER("RRS_REPORT"))
    #R6 = PARAMETER("RRS_REPORT")       ! "3"
ELSE
    #R6 = PARAMETER("IREN_REPORT")      ! "0" — the default when nothing matches
END_IF
```

> **field note** — the default is iRen. If a report is *supposed* to be Crystal but its control row is missing or mistyped, it doesn't error — it quietly falls back to the character report. That's the first thing to check when "the Crystal version" mysteriously reverts to the old look.

## The two-stage assembly: controls, then arguments

When a report resolves to Crystal, Ross runs its `CRYSTAL_SERVICE` block, which does two very different kinds of loading before anyone touches a URL.

**Stage one — the print controls (the *how*).** As it resolves the engine, `LB_REPORT_PRINT_CONTROL` (through its `FILL_WS_VT` helper) clears and reloads `SYS_WS_PRINT_CONTROLS_VT` from that report's `REPORT_PRINT_CONTROLS` row. This is the report's standing configuration — everything that's true every time you print it, regardless of *which* BOL. (A few values from the same row — the report name, print queue, and copy count — ride along as call parameters rather than in the VT, but they all originate from that one control row.)

| Control field | What it tells the web app |
| --- | --- |
| `SYS_STORED_PROCEDURE` | The Ross stored procedure that gathers the report's data |
| `SYS_STORED_PROCEDURE_2` | An optional second proc to run after (e.g. flag-as-printed) |
| `REPORT_FILE_NAME` | The Crystal `.rpt` template to render |
| `SYS_REPORT_FILE_TYPE` / `SYS_EXTENSION` | Output format — PDF, etc. |
| `SYS_REPORT_OUTPUT_DIRECTORY` | Where the rendered file lands |
| `REPORT_QUEUE` / `REPORT_COPIES` | Print destination and count |
| `EMAIL` · `SYS_SAVE_REPORT` · `AUTO_UPLOAD_SHAREPOINT` | Route the output: e-mail it, keep it, push it to SharePoint |
| `SYS_RUN_AS_SERVICE` · `SYS_DEBUG_MODE` · `SPLIT_REPORT_FLAG` | Run unattended, verbose, or one-file-per-document |

**Stage two — the run-time arguments (the *what*).** Then Ross drops the specifics of *this* run into `URL_PARAMS_VT`, one row per parameter, each a literal `KEY=VALUE;` string. Here's the BOL driver doing exactly that — note it's just building strings, nothing clever:

```dml
@program SOP_R_BOL_PRINT.DML
@note THE RUN-TIME ARGUMENTS FOR THIS PRINT, ONE KEY=VALUE; ROW AT A TIME
@writes url_params_vt
@highlight 3-4,8-9
CLEAR_BUFFER URL_PARAMS_VT
URL_PARAMS_VT(SEQUENCE) = 1
URL_PARAMS_VT(SYS_URL_PARAMETER) = "COMPANY_CODE=" & #COMPANY_CODE & ";"
ADD TO URL_PARAMS_VT
...
CLEAR_BUFFER URL_PARAMS_VT
URL_PARAMS_VT(SEQUENCE) = 3
URL_PARAMS_VT(SYS_URL_PARAMETER) = "SYS_CALLING_FUNCTION=" & #FACILITY_ID & ";"
ADD TO URL_PARAMS_VT
```

> **in the system** — this two-table split is the whole design. `SYS_WS_PRINT_CONTROLS_VT` is the report's *identity* (which proc, which template, where it goes); `URL_PARAMS_VT` is *this invocation's* arguments (which company, which shipping notes). The URL maker's only job is to weld the two into one address.

## The weld: LB_S_L_CRYSTAL_URL_MAKER

This is the program people mean when they say "the Crystal web service" — and it's almost anticlimactic. It reads the base address from a system config value, then concatenates every control and every parameter into one long query string. There is no networking here at all; it's string-building:

```dml
@program LB_S_L_CRYSTAL_URL_MAKER.DML
@note BUILD THE PRINT URL — BASE ADDRESS + CONNECT APP + PROC + PARAMS + ROUTING
@reads sys_ws_print_controls_vt, url_params_vt
@risk if GEM_CRYSTAL_URL isn't set, there's no base address and the whole path fails with P_13293
@highlight 1,4,7
#BASE_URL = GET_SCV("GEM_CRYSTAL_URL")
#PARTIAL_URL = #PARTIAL_URL & #BASE_URL & "?"
! the fat-app name the web app uses to call back into Ross
#PARTIAL_URL = #PARTIAL_URL & "gem_connect_app=" & #FAT_APP_NAME & "&"
! which proc to run, and its arguments (the KEY=VALUE; rows, joined)
#PARTIAL_URL = #PARTIAL_URL & "sys_stored_procedure=" & #SP1 & "&"
#PARTIAL_URL = #PARTIAL_URL & "report_file_name=" & #RPT & "&"
```

What comes out the other end is a single URL whose query string is, in effect, the API contract for the whole print job. Read it left to right and the "black box" has already told you everything it's about to do:

| Query parameter | Source | What the web app does with it |
| --- | --- | --- |
| *(base)* | SCV `GEM_CRYSTAL_URL` | The print web app's address |
| `gem_connect_app` | SCV `GEM_CONNECT_APP` | The Ross "fat app" to call back through Connect |
| `sys_stored_procedure` | control | The proc to run to fetch the data |
| `params` | `URL_PARAMS_VT` | The `KEY=VALUE;` arguments for that proc |
| `report_file_name` | control | The `.rpt` template to render |
| `sys_stored_procedure_2` | control | Optional follow-up proc |
| `report_queue` · `report_copies` | control | Where to print, how many |
| `gembase_user` | `%USERNAME` | Which Ross user the Connect session runs as |
| `sys_report_output_directory` · `sys_report_file_type` · `sys_extension` | control | Where the file lands, and in what format |
| `email` · `sys_save_report` · `auto_upload_sharepoint` | control | Post-render routing |
| `sys_run_as_service` · `sys_debug_mode` | control | Unattended / verbose |
| `sys_ext_report_template` | control | An external report template, when one is configured |
| `iaf_environment` | SCV `IAF_ENVIRONMENT` | Which environment (so DEV output can't print on PROD) |

> **rule of thumb** — if a Crystal report misbehaves, turn on `SYS_DEBUG_MODE`, print it, and read the URL. Every input the renderer receives is in that string. A blank report is almost always a bad `params=` or a proc that returned nothing; a mis-queued one is a bad `report_queue=`; a "prints in DEV, not PROD" is `iaf_environment=`.

## Handing it to a browser

The URL maker has two forms, and which one runs depends on how the user is connected (`%THIN_CLIENT_TYPE`). On any current deployment it's the first one — the second is a legacy thin-client path most sites never hit:

```steps
1 | IAF Desktop / iBrowser (IAF DESKTOP GUI, IBROWSER GUI)
where: CREATE_URL_ONLY
do: the session already IS a browser, so there's no detour — build the whole URL string and open it directly.
sys: CREATE_URL_ONLY returns the assembled URL in a parameter; the driver calls OPEN_URL to open it in a tab (or a new window when SYS_DEBUG_MODE is on). This is the path virtually every current session takes.

2 | Legacy thin client (GTC, PORTAL)
where: CREATE_URL_FILE
do: these old thin-client sessions can't open a URL themselves, so Ross writes the URL into a .bat on the client and runs it.
sys: rsiWriteFile.exe writes the URL line-by-line into the batch file; then CLI/CLIENT "rsiRunCrystal.bat" runs it on the client. Modern IAF Desktop deployments never reach this form.
```

> **in plain terms** — on IAF Desktop the session opens the URL directly; there's nothing exotic to it. The `.bat`-file detour exists only for the old GTC/PORTAL thin client, whose browser lives on the user's PC rather than in the session — so Ross posts a note to the client saying "go open this." For practical purposes it's deprecated.

> **caution** — the two forms don't emit *quite* the same query string. The direct build (`CREATE_URL_ONLY`) appends `auto_upload_sharepoint=` and `iaf_environment=` on the tail; the legacy batch build (`CREATE_URL_FILE`) stops at `sys_ext_report_template=` and sends neither. It's rarely relevant now, but if a report ever behaved differently from a GTC/PORTAL session — SharePoint upload not firing, or environment not honored — that trailing-parameter gap is why.

## What actually happens on the far side of the glass

Here's the part that isn't in the GEMBASE source, because it runs on the Crystal/IIS server — but the URL contract tells us precisely what it must do, step for step. When the print web app receives that URL:

```steps
1 | Read the request
do: parse the query string — the connect app, the proc + params, the template, the routing flags.
note: everything it needs is in the URL; there is no hidden second call from Ross.

2 | Call back into Ross for the data
do: open a Ross Connect session using gem_connect_app, authenticated as gembase_user, and execute sys_stored_procedure with params.
sys: this is the same class of retrieve-print stored procedure Ross uses for iRen (e.g. the RETRIEVE_PRINT_* family) — it gathers headers, lines, prompts and comments into result sets.

3 | Render the template
do: hand the returned data to the Crystal Reports runtime, which opens report_file_name (.rpt) and renders it to sys_report_file_type — typically PDF — in sys_report_output_directory.
note: the .rpt is where the layout lives — logos, barcodes, boxes. Ross never sees it.

4 | Route the output
do: print it to report_queue × report_copies, and/or e-mail it, save it, and upload it to SharePoint, per the flags.
sys: an optional sys_stored_procedure_2 runs afterward — commonly the "mark these documents printed" update.
```

That's the entire trick. **Ross gathers the data and describes the job; Crystal draws the picture and delivers it.** The division of labour is the point — and it's a genuinely good design once you stop expecting a service call and start seeing a print handoff.

## When it won't print — where to look first

The URL maker fails loudly, with specific message codes. Match the code — or the symptom — back to the knob that caused it:

| Symptom | What's happening | First thing to check |
| --- | --- | --- |
| `P_13293` · nothing prints | The base URL came back empty; the maker exits before building anything. | SCV `GEM_CRYSTAL_URL`. |
| `P_91053` · "failed to start stream" | A staging table was empty when the maker ran. | The caller must fill both tables (`FILL_WS_VT` + the `URL_PARAMS_VT` rows) *before* calling the maker. |
| `P_13282` · "error creating the URL" | A wrapper error — the maker returned failure. A symptom, not the cause. | The specific code raised just before it (`P_13293` / `P_91053`). |
| `P_13283` · environment unsupported | The session type can't open a print URL. | You're not on `IAF DESKTOP GUI` / `IBROWSER GUI` / `GTC` / `PORTAL` — e.g. a background run. |
| Set to Crystal, prints native anyway | Blank `SYS_REPORT_PROCESS_TYPE`, or a non-web session, forces the fallback to iRen. | The resolved process type, and that you're on a web-capable session. |
| A separate window opens each print | `SYS_DEBUG_MODE` is on — debug opens a new window instead of a tab. | Turn off `SYS_DEBUG_MODE` on the report's controls. |
| Renders, but can't read Ross data | The web app's call home through Connect failed. | SCV `GEM_CONNECT_APP`. |

Codes `P_13291` / `P_13292` / `P_13294` are legacy-thin-client-only (the `.bat` helpers) — a modern IAF Desktop session never sees them.

## Why anyone built it this way

Two payoffs justify the extra moving parts, and they're worth stating because they explain the whole architecture:

- **Layout leaves the developer's plate.** An iRen report is a `REPORT_FORM` written in DML — every column, every heading, compiled by a developer. A Crystal report is a `.rpt` file editable in the Crystal Reports designer by anyone with the tool. Logos, barcodes, pixel-aligned pre-printed-form overlays, real fonts — all the things a customer-facing BOL or invoice needs and a character report can't do — become a design task, not a code task.
- **The data logic is shared.** Because both engines are fed by the same retrieve-print stored procedures, switching a report from iRen to Crystal is (mostly) a config flip on `SYS_REPORT_PROCESS_TYPE`. You're changing the renderer, not rewriting how the BOL is assembled. That's why the two-table split matters: the *data* half is engine-agnostic.

> **caution** — the corollary is where teams get burned: **two of the three moving parts don't live in Ross.** The `.rpt` template lives on the Crystal server; the print web app is a deployed binary on IIS. GEMBASE source will happily show you the URL being built and never hint that the template is stale, the output directory is full, the Connect app is misconfigured, or the web app is down. When a Crystal report breaks and the DML side looks fine, the DML side probably *is* fine — the failure is off-box.

## A quick sibling note: RRS and WSS

You'll see two cousins of the Crystal URL maker in the same module, and they follow the identical pattern so they're easy to reason about once you have Crystal down:

- **`LB_S_L_RRS_URL_MAKER`** — the SSRS path, and the newest of the four engines (a later addition to the 8.0 line, where the Crystal path is original-vintage code). Same skeleton, but don't assume it's interchangeable: its base URL is passed in as a call parameter (not `GEM_CRYSTAL_URL`); it has a single form plus a `SUMMARY_PAGE_URL` wrapper; it sends `params=` **empty** (it never streams the `URL_PARAMS` tables); and it sources its main procedure from `SYS_GET_DOCUMENT_LIST` (plus a `document_list_name`) rather than `SYS_STORED_PROCEDURE`.
- **`LB_WSS_URL_MAKER`** — the same URL-assembly convention applied to the web self-service/SharePoint side.

Different renderer, same skeleton: load controls, fill params, weld into a URL, hand it to a browser.

## Where this comes from

- `REPORT_PRINT_CONTROLS` (`fin_tables.gem`) — the per-report configuration table; `SYS_REPORT_PROCESS_TYPE` selects the engine (0 iRen / 1 Crystal / 2 DC / 3 RRS), and it carries `SYS_STORED_PROCEDURE`, `SYS_STORED_PROCEDURE_2`, `REPORT_FILE_NAME`, `SYS_REPORT_OUTPUT_DIRECTORY`, `SYS_REPORT_FILE_TYPE`, the routing flags (`EMAIL`, `SYS_SAVE_REPORT`, `AUTO_UPLOAD_SHAREPOINT`), and the run flags (`SYS_RUN_AS_SERVICE`, `SYS_DEBUG_MODE`, `SPLIT_REPORT_FLAG`).
- `LB_REPORT_PRINT_CONTROL` (`GET_REPORT_PROCESS_TYPE`) — the router that reads the process type; defaults to iRen when no control row is found.
- `SOP_R_BOL_PRINT.DML` — a representative driver: `CRYSTAL_SERVICE` loads `SYS_WS_PRINT_CONTROLS_VT`, `CFB_SERVICE` fills `URL_PARAMS_VT` with `KEY=VALUE;` rows, then branches on `%THIN_CLIENT_TYPE` to either write-and-run a `.bat` or `OPEN_URL` directly.
- `LB_S_L_CRYSTAL_URL_MAKER.DML` — two forms: `CREATE_URL_ONLY` (returns the URL string, for IAF Desktop / iBrowser — the current path) and `CREATE_URL_FILE` (writes the URL into a batch file via `rsiWriteFile.exe`, for the legacy GTC/PORTAL thin client). Base address from SCV `GEM_CRYSTAL_URL`; the fat-app name from SCV `GEM_CONNECT_APP`; environment from SCV `IAF_ENVIRONMENT`.
- `LB_S_L_RRS_URL_MAKER.DML` / `LB_WSS_URL_MAKER.DML` — the SSRS and web-self-service siblings, same assembly pattern.

The usual honest caveat: this is standard Ross 8.0 behaviour read from source, and the mechanism is stable, but two of the three parts — the `.rpt` templates and the print web application itself — live outside GEMBASE on the Crystal/IIS server, so their configuration (paths, Connect app, service account, environment) is site-specific and not visible from the DML. The stable landmarks are the `SYS_REPORT_PROCESS_TYPE` switch, the `SYS_WS_PRINT_CONTROLS_VT` + `URL_PARAMS_VT` pair, and the `GEM_CRYSTAL_URL` / `GEM_CONNECT_APP` config values. Your queues, output directories, and template names are yours — confirm them on your own system before quoting a path.
