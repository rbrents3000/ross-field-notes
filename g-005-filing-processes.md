---
num: "005"
title: "Five Processes for Filing Documents"
subtitle: "The implementation: a shared environment subroutine, scanned-document capture, two sync directions, and the nightly reconciliation that is the only thing able to tell you a document is missing."
tag: "EMF"
audience: "developer"
date: "2026-09-08"
system: "Ross ERP 8.0"
reading_time: "11 min"
excerpt: "Every module named in these five designs exists and does what the design says it does — which sounds like a low bar until you count how many plausible-looking shapes quietly do not run. Validation before storage is the load-bearing idea; write ordering decides whether a failure is recoverable; and three of the five can be built before a single open question is answered."
stats:
  - k: "Processes"
    v: "5"
    note: "module by module"
  - k: "Buildable now"
    v: "3 of 5"
    note: "no open questions"
  - k: "Unsettled step"
    v: "1"
    note: "binary out over HTTP"
  - k: "Load-bearing"
    v: "Validation"
    note: "before anything is stored"
related:
  - "g-003-lift-and-shift-documents"
  - "g-004-two-directions"
key_refs:
  - "Datasection Toolbox"
  - "Text Formatting"
  - "File In"
  - "Loop"
reviewed: "2026-09-08"
verified: "Checked against the vendor manuals and a census of real configurations"
---

> This is the plain-Markdown copy of a Field Guide. The formatted version, with the
> module diagrams and light/dark reading view, lives at
> `fieldnotes.ryanbrents.com/g/g-005-filing-processes`.

Third of three. The diagnosis is in *Documents After the Lift and Shift*; the design is in *One
Store, Two Directions*.

Design sketches only — module shapes, not exported configuration. Every module named below exists
and does what it is said to do here, which sounds like a low bar until you find out how many
plausible-looking shapes quietly do not run.

## Process 1 — resolve the environment

The other four all begin by calling this one. It answers two questions at once: *where is the store*,
and *what does it want as a credential* — the two things that differ between environments and the
two things you never want scattered across four processes.

Endpoints come from a settings row rather than being typed into each process, so moving a container
or promoting from test to live is one edit in one place. The credential branch matters because the
two realistic targets want different things: a signature on the query string needs no round trip at
all, while a Graph-style endpoint wants a token fetched.

Two details are easy to get wrong. The token is extracted by a **Text Formatting** module using
`$JSONVALUE` — the Parser module handles delimited and fixed-width text and cannot read JSON. And
`Return` itself carries no payload: the `OUT` section declared on the Process Interface is what
copies the environment into the calling process; Return only releases the caller.

There is deliberately **no cache**. An earlier draft cached the token in a Data Store and read it
back on the next run. That is not buildable — the Data Store module only adds and updates, and
reading a stored value is a bracketed `$MESSAGE('[name]')$` reference rather than a module. This
fetches per run, which is also what the one real client-credentials flow in the reference set does.

## Process 2 — scan ingest

The piece with no vendor replacement, and the one whose failure is silent. It watches the drop
folder, derives the transaction key from the filename, and **validates it against the ERP before
storing anything**. Anything that does not resolve goes to an exceptions folder and a named human —
it is never guessed at.

The folder is listed once with contents off, and each file is read individually inside the loop, so
the content hash addresses the current file rather than the first one. `File In` has no age filter
of its own, so "only settled files" is a filter on `LASTMODIFIEDDATE` after it.

**Both arms link back to the loop, and that is the point.** A branch that ends, ends: the loop
advances only when a link physically reaches the Loop module. An exceptions arm that stopped at the
Move would halt the run at the first unresolvable file and still report success.

There is no watermark. Moving the file out of the drop folder *is* the watermark, which makes the
process naturally idempotent and means a transient failure simply leaves the file for the next run.

## Process 3 — catalog sync

Direction A. Enumerates the container, drops anything the other direction wrote *before* the loop
starts, and creates an item carrying the keys plus a link. Read-only against the vendor's store, so
it cannot corrupt anything or create documents the ERP will not recognise.

Paging is a **bounded Count loop**, not a "repeat until empty". The Loop module offers exactly five
sources — Datasection, Per Recipient, Count, XPath and JSON — and none of them is a while-loop, so
the page count is capped and published and the marker decides whether the body runs again. A Loop
carries at most two out-links: the body and the exit.

Strip the byte-order mark before any XML consumer, and gate the parser on what the endpoint actually
returns — an XML parser fails outright on a JSON response.

## Process 4 — reverse sync

Direction B, and the one that depends on an open question — whether a document written directly into
the store is one the ERP will actually return. Build it only after that has been observed on a real
system.

Structurally the mirror of Process 3, with one addition: a step that reads the document back by key
to confirm it actually returns, rather than assuming the write succeeded. If it does not, the
document is an orphan — and you want to know on the first one, not the thousandth. The orphan test
reads the response code out of its named message section; it is a branch condition, not a log line.

## Process 5 — nightly reconciliation

Everything above moves documents. **This is the only one that can tell you a document is missing**,
and because the ERP persists no document state, nothing else in the estate can.

Compare the transactions that should have produced a document against what the catalog holds, and
publish three counts: **missing documents** (the measurable size of the coverage gap — the number
nobody has today), **orphans** (rows matching no transaction), and **divergence** (filed one way but
not another).

That is **two joins, not one** — a left outer for missing and a right outer for orphans, plus an
inner join and a field comparison for divergence. There is no full outer join to lean on. Each
comparison copies before it filters, because a filter merges into the primary data section unless
the operation performs a copy as well; without that, the first comparison destroys the input to the
other two.

The three run **in sequence on a single thread**, not as parallel branches. When a processing path
splits, a new process object is created for each path — so three branches meeting at one Email Out
would send three separate emails, each carrying only its own count.

Make the assertion re-runnable over *any* period, not a trailing thirty days. Retention runs to
years, and a coverage hole opens the day a new user first runs a report.

## Things worth arguing about before building

The first four are the ones that cost time if you meet them at the keyboard instead of here. Each
describes a shape that looks right on a canvas and then fails without an error.

- **A branch that ends, ends.** A loop advances only when a link physically reaches the Loop module.
  An arm of a loop body that simply terminates stops the loop where it stands — and the run reports
  success.
- **A halt inside a loop ends the loop.** Conditional Halt stops an EMF process *thread*, and the
  loop body's thread is the one carrying the link back to the Loop. To skip an item, filter before
  the loop, or use a mutually exclusive pair of conditional links whose arms both rejoin.
- **Concurrent branches that meet, run twice.** A module three concurrent branches point at executes
  three times, each carrying only its own branch's sections. Mutually exclusive conditional arms are
  the exception — only one runs, so they may safely share one Return.
- **Filter merges unless it also copies.** Three filters off one joined set without copies means the
  first destroys the input to the other two, silently and with plausible-looking output.
- **Two ways to send a file, and they are not interchangeable.** Over SOAP the file is
  base64-encoded into the XML body — the shape used here, and one that appears many times over in
  working configurations. Over REST you may instead point the request body at the file's own section
  with a real content type, but a data section handed straight to an HTTP module needs a text
  formatting bridge first.
- **Binary out is the one genuinely unsettled step.** The documented binary route out of an HTTP
  module is a `POST` with content type `application/octet`. A production configuration does drive
  object storage with `PUT` and a real content type — so it works in the field, but on precedent
  rather than documentation. Prove it with one call before building on it.
- **Modules that are not what they sound like.** `Data Store` only adds and updates. `Parser` cannot
  read JSON. The relational join and filter module is the Datasection Toolbox; Data Transformation
  converts sections to XML or JSON and cannot join. `File In` has no age filter.
- **Branch on links, not on halt conditions.** Set the SQL module's halt condition to *Never* and put
  the decision on the out-links: a row-count test on one, "run when all other links using conditions
  have not run" on the other. A halt-on-no-data stops the thread rather than branching.
- **Paging has no precedent here.** Designed, not observed — nothing in the reference set pages.
- **Endpoints belong in a settings row**, and credentials never go in a request URI — publish them to
  a section and inject an authorization header.
- **Validation before storage is the load-bearing idea.** It is the difference between an exception
  queue somebody works and a slow accumulation of documents attached to nothing.
- **Order the writes so a failure is recoverable.** Bytes, then catalog row, then move the source. A
  crash between the first two leaves an orphan a reconciliation run finds and a person can fix.
  Reverse the order and you get a catalog row pointing at nothing, which nothing can fix.
- **Alert on silence, not only on errors.** A scheduled process that quietly stops firing produces no
  errors at all.
- **Give a poison file somewhere to go** after a third failed attempt, or it is retried forever.
- **Deletion is unhandled by design** — reconciliation surfaces it as an orphan rather than a sync
  silently deleting a record.

## Sequencing the build

The five are not equally ready. Three depend on nothing that is still open; two cannot responsibly be
built until the store question in the design guide is answered. That ordering matters more than any
estimate.

**Step 1** settles the questions. **Step 2** builds processes 1, 2 and 5, none of which depend on the
answers. **Step 3** builds the sync pair, whose shape step 1 decides. Steps 2 and 3 are separable, so
the work can stop after either without leaving something half-built.

Beyond the processes themselves, a build produces the SQL — key validation, the catalog schema and
its indexes, the three reconciliation comparisons — the store setup (columns, content types, indexed
fields, default views), and a runbook covering what to configure, what to test, and what to alert on.

Field names, endpoints, credentials and environment identifiers are per-deployment and always need
setting locally. That is configuration rather than redesign, but it is real work. And nothing can be
validated against real data until it runs against real data.
