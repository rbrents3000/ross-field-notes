---
num: "011"
title: "Our lift-and-shift moves Document Connect off SharePoint and onto Azure Blob with almost no metadata. How do we keep searching and auto-filing our documents?"
tag: "ROSS"
audience: "developer"
source: "group"
date: "2026-09-02"
system: "Ross ERP 8.0"
reading_time: "16 min"
excerpt: "Document Connect is two features sharing one name, and only one of them is being taken away. The in-screen upload panel is governed by a config switch that a cloud migration flips from SharePoint to blob storage. The retrieve path is something else entirely: a URL that LB_WSS_URL_MAKER assembles from three ordinary maintenance tables and one environment variable, in the form ?FilterField1=…&FilterValue1=…. Nothing in that code is SharePoint-specific. Point it at a document finder you own, keep the blob as the vendor-supported store, and carry the metadata in an index you control — then let EMF do the filing, exactly as it does today."
question: "We're in the process of our lift and shift to the cloud. On premise, we were using SharePoint's document libraries to house our supplemental documents. Users could upload their documents in Ross, but they could also go to SharePoint to upload and search for documents as long as the proper metadata existed. We took advantage of this and created EMF processes to grab documents that were scanned in and attached them to the appropriate order in Ross. The intended path is to upload these documents into Azure Blob Storage with very limited metadata, which makes it nearly impossible to upload documents and search them the way we were able to in SharePoint. Were you using SharePoint as your Document Connect tool to associate documents to your orders when on-prem? Did you or your users ever upload documents to Document Connect without using the toolbox within Ross? Did you have to come up with a solution on top of what was provided to better manage your supplemental documents?"
restated: "In standard Ross ERP 8.0, when a cloud migration changes the Document Connect store from SharePoint document libraries to Azure Blob Storage, what is the supported mechanism for preserving document metadata, out-of-Ross search and upload, and automated attachment of scanned documents to transactions?"
fix: "Stop treating Document Connect as one feature. It is two, and they are configured independently. The in-screen DocConnect panel is governed by the iBrowser <DocumentService> block — Mode and NewUploadTo, whose documented values are SharePoint, docservice (blob) and Hybrid (both). That is the switch a lift-and-shift flips, and Hybrid is a supported setting, not a hack; note also that the product default for DocumentService is SharePoint, so blob is a deployment choice rather than the only supported path. The retrieve path is separate and it is entirely yours: LB_WSS_URL_MAKER reads a base URL from the GEM_WSS_URL system control variable, appends a per-module path from SYS_ERP_WSS_MODULE_URL, then walks SYS_ERP_WSS_MODULE_FIELDS and SYS_ERP_WSS_MAPPING_FIELDS to append ?FilterField1=<name>&FilterValue1=<value> pairs taken from the SYS_WSS_URL_PARAMETERS virtual table, and surfaces the result as an Active Link on the transaction. Not one line of that is SharePoint-specific — only the query-string convention is. So: let the blob be the store, because that is the fight you do not need; build a document index keyed on the ERP keys you already file by, and a small finder application that speaks FilterFieldN/FilterValueN; re-point GEM_WSS_URL at it, and set the module rows yourself in SYS_M_265 and SYS_M_266 without a vendor ticket. Then rebuild the scan pipeline in Aptean EMF using File In, SQL, HTTP and SQL Insert — the same modules your current processes already use — so a scanned document lands in the blob and its metadata lands in your index in one pass. You end up with more searchable metadata than SharePoint gave you, no per-library item cap, and an architecture that survives the vendor's next storage decision."
margin_notes:
  - "two features, one name — only one is changing ↴"
  - "the retrieve URL is a template you own"
  - "Mode: SharePoint | docservice | Hybrid →"
  - "SYS_M_265 needs no vendor ticket"
  - "O365 caps a library at 5,000 items"
  - "let the blob win; own the index"
reviewed: "2026-09-02"
verified: "Verified against Ross 8.0 source and IAF 9.3 documentation"
applies_when:
  - "You are moving to a hosted deployment and have been told supplemental documents will now live in Azure Blob Storage."
  - "Your users are used to opening a SharePoint library directly to upload or search, without going through the Ross toolbox."
  - "You have EMF processes that pick up scanned documents and attach them to an order, and nobody can tell you what happens to them after the migration."
  - "The DocConnect panel still opens, but the documents it lists have lost the metadata you used to filter on."
key_refs:
  - "LB_WSS_URL_MAKER"
  - "SYS_ERP_WSS_MODULE_URL"
  - "SYS_ERP_WSS_MODULE_FIELDS"
  - "SYS_ERP_WSS_MAPPING_FIELDS"
  - "SYS_WSS_URL_PARAMETERS"
  - "GEM_WSS_URL"
  - "SYS_M_265"
  - "SYS_M_266"
  - "DocumentService"
---

The premise buried in this question is that Document Connect is a single feature, and that the migration is taking it away and handing back a worse one. That premise is worth dismantling first, because once it goes, most of the problem goes with it.

Document Connect is two mechanisms sharing a marketing name. They were built at different times, they are configured in different places, and a lift-and-shift only changes one of them. The half that is changing is the half you care least about. The half that is *not* changing is a general-purpose URL builder that will happily point at anything you own — and that is the whole solution, sitting in the base product, already licensed, already installed.

## First: which Document Connect are we talking about?

| | The DocConnect panel | The WSS retrieve link |
| --- | --- | --- |
| What the user sees | The in-screen panel: document count, list, upload, delete | An Active Link on the transaction that opens a filtered document view |
| Where it's configured | iBrowser `IafConfig.xml`, the `<DocumentService>` block | Three maintenance tables plus one environment variable |
| Who can change it | Vendor, in a hosted deployment | **You**, for the tables; vendor for the variable |
| What the migration does to it | Repoints it from SharePoint to blob storage | Nothing |
| Where the metadata lives | Whatever the storage back end supports | Whatever your target URL can accept |

> **in plain terms** — one is a drawer built into the screen, and the landlord is changing what the drawer is made of. The other is a doorway with your own address written above it. You have been arguing about the drawer.

## The doorway: what LB_WSS_URL_MAKER actually does

`SYS_M_265` (*WSS Module URL and Fields*) stores a URL fragment per source module and transaction type. `SYS_M_266` (*WSS ERP Field Mapping*) maps a field on the `SYS_WSS_URL_PARAMETERS` virtual table to a field name on the target. `LB_WSS_URL_MAKER` reads the current transaction's parameters out of that virtual table and stitches the two together into one address.

Here is the part that matters — the loop that appends each mapped field as a filter pair:

```dml
@program LB_WSS_URL_MAKER.DML
@note THE URL BUILDER — APPENDS ONE FilterFieldN/FilterValueN PAIR PER MAPPED FIELD
@reads sys_erp_wss_mapping_fields, sys_erp_wss_module_fields, sys_wss_url_parameters
@risk the base URL is an SCV, not a table — changing it in a hosted environment is a vendor request
@highlight 2,3,11,13
BEGIN_BLOCK FIELD_MAPPING
    #FILTERFIELD = SYS_ERP_WSS_MAPPING_FIELDS(WSS_FIELD_CODE)   ! name on the target
    #FIELD       = SYS_ERP_WSS_MAPPING_FIELDS(ERP_FIELD_CODE)   ! name on the virtual table
END_BLOCK

BEGIN_BLOCK CHECK_VT
    #FILTERVALUE = TABLE_DATA("SYS_WSS_URL_PARAMETERS",#FIELD)  ! the live transaction value
END_BLOCK

BEGIN_BLOCK URL
    IF ( #FILTERVALUE <> "" )
        #COUNTER = #COUNTER + 1
        ! CHR(63)=?  CHR(61)==  CHR(38)=&
        #WSS_URL = #WSS_URL & CHR(38) & "FilterField" & #COUNTER & CHR(61) & #FILTERFIELD &
                              CHR(38) & "FilterValue" & #COUNTER & CHR(61) & #FILTERVALUE
    END_IF
END_BLOCK
```

And the base, assembled earlier in the same program:

```dml
@program LB_WSS_URL_MAKER.DML
@note THE BASE — AN ENVIRONMENT VARIABLE PLUS A ROW YOU MAINTAIN
@reads sys_erp_wss_module_url
#WSS_BASE_URL = GET_SCV("GEM_WSS_URL")                          ! the host — a vendor-set SCV
#DEFAULT_URL  = COMPRESS(TRIM(SYS_ERP_WSS_MODULE_URL(URL)))     ! the path — your maintenance row
#WSS_URL      = #WSS_BASE_URL & CHR(47) & #DEFAULT_URL          ! CHR(47) = /
```

So a sales order inquiry produces something shaped like this:

```
https://<GEM_WSS_URL>/<your module path>?FilterField1=Company&FilterValue1=1
   &FilterField2=Division&FilterValue2=11&FilterField3=Order_x0020_Number&FilterValue3=385344
```

> **in the system** — `FilterFieldN`/`FilterValueN` is SharePoint's list-view filter convention, which is why it looks like SharePoint's. But the program has no knowledge of SharePoint. It concatenates strings from three tables and an SCV. Any web application that can read a query string can be the thing on the other end.

> **field note** — this is the leverage. You do not need permission to change what Document Connect retrieves, because `SYS_M_265` and `SYS_M_266` are ordinary maintenance screens on your own system. Only the host portion — the `GEM_WSS_URL` SCV, set in the application environment — needs the vendor to touch it, and it is a single value changed once.

## The drawer: the switch you're being asked to flip

The panel is governed by the `<DocumentService>` block in the iBrowser configuration. Its documented values:

```cards
SharePoint | Upload to SharePoint | The URL is taken from iBrowser/SharePointSite. This is the documented default value for DocumentService.
docservice | Upload to blob storage | The URL is taken from iBrowser/DocumentService/ServiceUrl. Requires Tenant to be set. This is what a lift-and-shift moves you to.
Hybrid | Upload to both | Writes the document to both SharePoint and blob storage. Requires Tenant. A supported setting, not a workaround.
```

Two details from the same documentation set that are worth having in your pocket before the conversation:

- **The product default is `SharePoint`.** Blob is a deployment decision, not the only supported configuration.
- **SharePoint Online is a supported target**, via `SharePointOnline`, `SharePointDomain` and `SharePointOnlineEmailADSource` (UPN or EMAIL), and the shipped deployer tool provisions content types against SharePoint Online as well as on-premise. "We are in the cloud now" is not, by itself, an argument that SharePoint is unavailable.

> **caution** — two constraints cut the other way, and you should raise them yourself rather than be surprised by them. The vendor's own compatibility guide footnotes that Office 365 **limits each library to 5,000 items** for this integration — on a document library that accumulates orders, that is a real ceiling, and it is very likely the actual reason blob is being pushed. And the delete function in the DocConnect panel is documented as working **only** in `docservice` mode; in SharePoint and Hybrid modes the icon is hidden.

## Three ways out, ranked

| | What it is | Cost | Why it wins | Why it loses |
| --- | --- | --- | --- | --- |
| **A. Ask for SharePoint mode** | Have the vendor set `NewUploadTo` to SharePoint against your own tenant | One ticket | Nothing changes; you keep everything | You are asking, not deciding; 5,000-item cap; no delete |
| **B. Hybrid dual-write** | `Mode` set to `Hybrid` — blob is the system of record, SharePoint is the searchable copy | One ticket | Supported; vendor keeps its store; you keep your search | Two copies of every document — storage, drift, and a genuine retention headache |
| **C. Own the index** | Blob stays the store; you build the metadata index and the finder, and re-point the retrieve URL | A modest build | You control it; no caps; survives the next storage change | You own the uptime |

**A is worth doing first because it costs a ticket.** Ask, and be ready for the 5,000-item answer. **B is a good bridge and a poor destination** — storing every document twice to make one of the copies searchable is a liability you will be explaining to your auditors later. **C is the recommendation**, and the rest of this note is how to build it.

## The design: let the blob win, own the index

The insight is that the migration is only taking the *storage* away. It is not taking the *keys* away — you still know the company, division, order number and customer for every document, because Ross is where those come from. Metadata was never really SharePoint's to give you; SharePoint was just where you happened to keep it.

So separate the two concerns:

```states
Blob storage | the bytes — vendor-supported, uncapped, cheap
Your index | the keys — one small row per document, fully searchable
Your finder | the doorway — speaks FilterFieldN/FilterValueN
Aptean EMF | the filing clerk — writes both, on every scan
```

**The index** is one table, and it is deliberately boring. One row per document:

| Column | Why |
| --- | --- |
| `doc_id` | Surrogate key |
| `company`, `division` | The filter every screen passes |
| `source_module`, `transaction_type` | Mirrors the `SYS_M_265` keys, so the finder can route |
| `doc_number` | Order, PO, invoice, ship note — whichever key the module files by |
| `customer`, `vendor` | Secondary filters your users actually search on |
| `doc_date`, `uploaded_at`, `uploaded_by` | Chronology and provenance |
| `title`, `content_type`, `file_size` | What to show in the list |
| `blob_uri` | Where the bytes are |

That is more metadata than a SharePoint content type typically carried, it costs almost nothing to store, and it is a normal database table you can index, join and report against.

**The finder** is a small web application with exactly one hard requirement: it must accept `?FilterField1=…&FilterValue1=…&FilterField2=…` and treat those pairs as a conjunctive filter. Meet that contract and Ross's existing Active Link lights up unchanged. Everything else about it is your choice — and since you are writing it, give it the search page SharePoint never quite gave you: by order, by customer, by vendor, by date range, by document type, across every module at once.

```steps
1 | User opens a transaction inquiry
where: SOP_I_002, POP_I_001, AP_I_001 — any Document Connect location
do: Ross fills SYS_WSS_URL_PARAMETERS with the current transaction's keys.
sys: The virtual table must hold exactly one row; LB_WSS_URL_MAKER fails the request otherwise.

2 | Ross builds the address
do: LB_WSS_URL_MAKER joins GEM_WSS_URL, the SYS_ERP_WSS_MODULE_URL path, and one FilterFieldN/FilterValueN pair per mapped field.
sys: Mapping comes from SYS_ERP_WSS_MODULE_FIELDS joined to SYS_ERP_WSS_MAPPING_FIELDS on FIELD_CODE.

3 | The Active Link opens your finder
do: The user clicks through and lands on a filtered document list for that exact transaction.
note: This is the step that used to land on a SharePoint filtered view. Same mechanism, new address.

4 | The finder queries the index, not the store
do: Translate the filter pairs into a query against your index table and render the matches.
sys: Never scan the blob container to answer a query — the index exists precisely so you don't.

5 | The document streams from blob
do: Each result links to a short-lived signed URL for the blob object.
note: Expire them in minutes. The finder authorizes; the storage account stays private.
```

## Rebuilding the scan pipeline in EMF

This is the part that sounds hardest and is actually the most familiar, because it is the same shape as the processes already running. Every module below is a standard Aptean EMF module:

```steps
1 | File In
where: EMF Process Builder — Data modules
do: Read the scan drop folder into a data section.
sys: Pattern, older-than and subdirectory filters are on the module; pair with a Scheduler initiator.

2 | Parser
do: Pull the key out of the filename or the barcode/cover-sheet text — order number, PO number, ship note.
sys: Delimited or fixed-width into a data section; Auto define will do most of the work.

3 | SQL
do: Look the key up in Ross and fetch the rest of the metadata — company, division, customer, date.
sys: This is also your validation gate. No matching transaction means the document is misfiled, not that the index should take it.

4 | Conditional Halt
do: Stop this branch when the lookup came back empty, and route the file to an exceptions folder instead.
note: Skipping this is how you end up with an index full of documents attached to nothing.

5 | HTTP
do: PUT the file into blob storage.
sys: The HTTP module is a full client — it posts data out, not just in. Use the vendor's document service endpoint here if you can get its specification.

6 | SQL Insert
do: Write the index row — the keys from step 3 plus the blob URI from step 5.
sys: SQL Insert writes a data section straight into a table. This is the step that makes the document findable.

7 | Move Files
do: Move the processed file out of the drop folder into an archive path.
note: Do this last. A file still sitting in the drop folder is the only reliable signal that the run failed.
```

> **rule of thumb** — write the bytes before the index row, and let a failed index write leave an orphan blob rather than an index row pointing at nothing. An orphan blob is a cleanup job. A dangling reference is a support call.

## Keeping the habit: SharePoint as a drop zone, not a system of record

The second question in the original post — *did users ever upload without the Ross toolbox?* — is the one people underestimate. Users had a workflow: open the library, drag the file in, fill the columns. Take it away and they will email documents to each other instead, and you will lose them entirely.

You do not have to take it away. Keep a SharePoint library as the **front door** and let it stop being the **filing cabinet**:

- Users keep uploading exactly as they do today, into the library, filling the columns they already know.
- An EMF process on a schedule sweeps the library — `HTTP` against the Graph or REST endpoint, `XML Parser` or a `Script` step to read the response — and for each new item moves the content to blob and writes the index row.
- The item is then removed from the library.

The library never accumulates, so the 5,000-item cap becomes irrelevant — it is a queue, not an archive. Users keep a UI they already know. And the searchable system of record is your index, which is the thing that survives the next migration.

> **in plain terms** — stop asking SharePoint to remember things. Let it be the place people hand documents in, and remember them somewhere you control.

## The rollout

```checklist
[ ] Ask for SharePoint or Hybrid mode {recommended} — costs one ticket, may solve it outright
where: Vendor request against the iBrowser <DocumentService> block
[ ] Get the document service API specification in writing {required} — you cannot design step 5 without it
[ ] Stand up the index table and backfill it from the existing library metadata {required}
where: Your own database — do not put this in the ERP schema
[ ] Build the finder and make it answer FilterFieldN/FilterValueN {required}
[ ] Point SYS_M_265 and SYS_M_266 at the finder for one module {master-data} — start with sales orders
where: System > Sharepoint WSS > WSS Module URL and Fields
[ ] Request the GEM_WSS_URL SCV change {gotcha} — the one piece you cannot do yourself
[ ] Move one EMF scan process to the new pipeline and run both in parallel for a cycle {developer}
[ ] Roll the remaining modules, then convert the library to a swept drop zone
```

```legend
recommended | do this first, it is nearly free
required | the build cannot proceed without it
master-data | ordinary maintenance, no vendor involvement
gotcha | depends on someone outside your team
developer | needs process work in EMF
```

## What to get from your vendor in writing

One genuine gap: the document service exposed by `ServiceUrl` is referenced throughout the configuration documentation, but its **REST contract is not published in the standard manual set**. Everything documented tells you how to point *at* it, not how to *call* it. That single specification determines whether your EMF pipeline writes through the vendor's service — keeping the in-screen panel authoritative — or writes to blob directly, leaving the panel showing only Ross-originated uploads while the Active Link shows everything.

Ask for, and get in writing:

- [ ] The document service REST specification — endpoints, auth, and **exactly which metadata fields it accepts and returns**
- [ ] Whether `Mode` values `SharePoint` and `Hybrid` are supported in your hosted environment, and if not, why the documented default is unavailable
- [ ] Whether `GEM_WSS_URL` can be set to a customer-owned host, and the change process for it
- [ ] Whether outbound HTTP from the hosted EMF instance to your own endpoints is permitted
- [ ] Retention, backup and eDiscovery posture of the blob container, since it is displacing a governed SharePoint library

That last one is not a technical question, and it is the one that tends to change minds. A document library sitting inside a managed tenant carries retention policy, legal hold and discovery tooling. A storage container does not, unless someone has deliberately built that. If supplemental documents include anything that could be called for — signed delivery receipts, quality certificates, supplier correspondence — that difference belongs in the migration decision rather than after it.

> **caution** — do not put your index table in the ERP database. It will not survive an upgrade, it complicates every vendor support conversation, and it gives you nothing that a separate database does not. Keep the boundary clean: the ERP owns transactions, you own the document index.

## What this actually buys you

Working backwards from the three original questions: documents remain associated to orders, because association was always the ERP keys and you still have those. Users keep uploading outside the toolbox, because the library stays as a swept drop zone. Scanned documents keep filing themselves, because EMF is doing the same job with the same modules against a different destination.

And you end up somewhere better than where you started. The old arrangement made SharePoint load-bearing for search, which is why losing it hurt so much. The new one makes a small table you own load-bearing instead — no item caps, more metadata than a content type carried, and a storage back end that can be swapped again without anyone having to ask a forum what to do about it.
