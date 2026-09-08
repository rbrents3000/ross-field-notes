---
num: "005"
title: "Five Processes for Filing Documents"
subtitle: "The implementation: a shared environment subroutine, scanned-document capture, two sync directions, and the nightly reconciliation that is the only thing able to tell you a document is missing."
tag: "EMF"
audience: "developer"
date: "2026-09-08"
system: "Ross ERP 8.0"
reading_time: "12 min"
excerpt: "Every module named in these five designs exists and does what the design says it does — which sounds like a low bar until you count how many plausible-looking shapes quietly do not run. Validation before storage is the load-bearing idea; write ordering decides whether a failure is recoverable; and two of the five can be built before a single open question is answered."
stats:
  - k: "Processes"
    v: "5"
    note: "module by module"
  - k: "Buildable now"
    v: "2 of 5"
    note: "the rest write to the store"
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
verified: "Checked against the vendor manuals and a de-duplicated census of real configurations"
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

**Processes 2, 3 and 4 open by calling this one.** It answers two questions at once: *where is the
store*, and *what does it want as a credential* — the two things that differ between environments
and the two things you never want scattered across four processes. Process 5 never touches the
store, so it does not call this at all.

Endpoints come from a settings row rather than being typed into each process, so moving a container
or promoting from test to live is one edit in one place. The credential branch matters because the
two realistic targets want different things: a signature on the query string needs no round trip at
all, while a Graph-style endpoint wants a token fetched.

**Two OUT parameters, not one.** Message sections and data sections are declared in two separate
lists on the Process Interface, so a single parameter cannot carry both the settings row (a data
section) and the token (a message section). Two more details are easy to get wrong: the token is
extracted by a **Text Formatting** module using `$JSONVALUE` — the Parser module handles delimited
and fixed-width text and cannot read JSON — and `Return` carries no payload at all. The OUT sections
are what reach the caller; Return only releases it.

There is deliberately **no cache**. An earlier draft cached the token in a Data Store and read it
back on the next run. That is not buildable — the Data Store module only adds and updates, and
reading a stored value is a bracketed `$MESSAGE('[name]')$` reference rather than a module. This
fetches per run, which is also what the one real client-credentials flow in the reference set does.

One honest wrinkle on the signature arm: a shared-access signature *is* a credential on a query
string, which the notes below otherwise tell you to avoid. Object-storage REST offers no header
alternative, so keep the signature in a stored section, compose the URI from it so the secret is
never typed into a module, and treat the URI as sensitive wherever it is logged.

## Process 2 — scan ingest

The piece with no vendor replacement, and the one whose failure is silent. It watches the drop
folder, derives the transaction key from the filename, and **validates it against the ERP before
storing anything**. Anything that does not resolve goes to an exceptions folder *and* an emailed
owner — it is never guessed at, and it is never left silently in a folder nobody watches.

The folder is listed once with contents off, and each file is read individually inside the loop, so
the content hash addresses the current file rather than the first one. `File In` has no age filter
of its own, so "only settled files" is a filter on `LASTMODIFIEDDATE` after it.

**The archive Move on the success arm is what makes this idempotent**, and it is the step most
easily left out. File In is stateless and a date filter selects *settled* files, not unprocessed
ones — so without moving the file after a successful store, every run re-ingests everything it has
already filed. Moving it out of the drop folder *is* the watermark.

**Both arms link back to the loop.** A branch that ends, ends: the loop advances only when a link
physically reaches the Loop module. An exceptions arm that stopped at the Move would halt the run at
the first unresolvable file and still report success.

## Process 3 — catalog sync

Direction A. Enumerates the container, drops anything the other direction wrote *before* the loop
starts, and creates an item carrying the keys plus a link. Read-only against the vendor's store, so
it cannot corrupt anything or create documents the ERP will not recognise.

Paging is a **bounded Count loop**, not a "repeat until empty". The Loop module offers exactly five
sources — Datasection, Per Recipient, Count, XPath and JSON — and none of them is a while-loop, so
the page count is capped and published, and a conditional link flagged as exiting the loop carries
the marker decision. A Loop carries at most two out-links: the body and the exit.

**A loop does not accumulate; you have to accumulate it yourself.** A module that *creates* a data
section re-creates it on every iteration, so without a merge step only the last page would reach the
filter and every earlier page would be dropped without an error. Merge each page into a run-scoped
accumulator — the manual documents this case for loops specifically, including what happens on the
first pass, so there is no special case to write.

Strip the byte-order mark before any XML consumer, and gate the parser on what the endpoint actually
returns — an XML parser fails outright on a JSON response.

## Process 4 — reverse sync

Direction B, and the one that depends on an open question — whether a document written directly into
the store is one the ERP will actually return. Build it only after that has been observed on a real
system.

Structurally the mirror of Process 3, and now literally so: the same BOM strip, page parse and
accumulator, because a SOAP list response needs turning into something a loop can walk just as much
as a REST one does. Note the paging marker rides in the **payload** here, not the URI — a Web
Service module takes its endpoint from the WSDL and has no Resource URI field to put it in.

The addition over its mirror is a step that reads the document back by key to confirm it actually
returns, rather than assuming the write succeeded. **That read-back has two arms**: a document that
comes back is stamped synced, and anything else records an orphan and is never stamped. If you draw
only the success arm, the orphan case either dead-ends inside the loop or gets marked synced
forever.

**One word does a lot of work in both sync processes.** `origin` is the provenance stamp from the
design guide: a tag on the object in the store and an indexed column on the library item, each
recording which direction created it. Process 3 skips items whose origin says the library wrote them;
Process 4 skips items whose origin says the store did. Define it once, in both places, before either
process runs — it is the only thing stopping the two from feeding each other forever.

## Process 5 — nightly reconciliation

Everything above moves documents. **This is the only one that can tell you a document is missing**,
and because the ERP persists no document state, nothing else in the estate can.

Compare the transactions that should have produced a document against what the catalog holds, and
publish three counts: **missing documents** (the measurable size of the coverage gap — the number
nobody has today), **orphans** (rows matching no transaction), and **divergence** (filed one way but
not another).

That is **three joins, not one** — a left outer for missing, a right outer for orphans, an inner join
plus a field comparison for divergence. There is no full outer join to lean on.

**Each comparison takes two Datasection Toolbox modules, not one.** Operations inside a single module
run in fixed tab order, which puts *filter before join* — and join writes a new third section,
leaving its inputs untouched. So a join-and-filter module filters its input and emits the entire
join, which would make the "missing" count the total transaction count. One module joins; a second
copies and filters. The copy matters too: a filter without one mutates the section in place.

**An Email Out with no recipient module preceding it sends nothing at all.** Recipients are gathered
from recipient modules earlier in the flow, and with none the process runs green and delivers
silence — the worst possible failure for the one process whose entire output is a message.

The three comparisons run **in sequence on a single thread**, not as parallel branches. When a
processing path splits, a new process object is created for each path — so three branches meeting at
one Email Out would send three separate emails, each carrying only its own count.

Make the assertion re-runnable over *any* period, not a trailing thirty days. Retention runs to
years, and a coverage hole opens the day a new user first runs a report.

## Things worth arguing about before building

The first six are the ones that cost time if you meet them at the keyboard instead of here. Each
describes a shape that looks right on a canvas and then fails without an error.

- **A branch that ends, ends.** A loop advances only when a link physically reaches the Loop module.
  An arm of a loop body that simply terminates stops the loop where it stands — and the run reports
  success.
- **A loop does not accumulate.** A module that *creates* a data section re-creates it every
  iteration, overwriting the last. Merge into a run-scoped accumulator.
- **Toolbox runs in tab order.** Filter fires before join, and join writes
  a new third section — so one module cannot join and then filter its own join output. Each
  comparison is two modules, and a filter without a copy mutates the section in place.
- **Email Out needs a recipient.** A named validation in the builder, but the
  runtime consequence is silence.
- **A halt inside a loop ends the loop.** Conditional Halt stops an EMF process *thread*, and the
  loop body's thread is the one carrying the link back to the Loop. *The rule is documented; the loop
  consequence follows from it rather than from an observed example.*
- **Concurrent branches run twice.** A module three concurrent branches point at executes
  three times, each carrying only its own branch's sections. Mutually exclusive conditional arms are
  the exception — only one runs, so they may share one Return. *Also inferred from the documented
  split behaviour rather than from an observed case.*
- **One parameter, one section kind.** Message sections and data sections are declared
  separately, so a subroutine returning both needs two OUT parameters.
- **Two ways to send a file.** Over SOAP the file is base64-encoded
  into the XML body — 26 occurrences across eleven distinct configurations, 23 of them encoding a
  file. Over REST you point the request body at the file's own section with a real content type;
  base64 there would store a base64 text file rather than the document.
- **Binary out is unsettled.** The documented binary route out of an HTTP
  module is a `POST` with content type `application/octet`. A production configuration does drive
  object storage with `PUT` and a real content type — so it works in the field, but on precedent
  rather than documentation. Prove it with one call. Processes 2 and 4 both depend on it.
- **Modules that mislead.** `Data Store` only adds and updates. `Parser` cannot
  read JSON. Coming back from XML is the `XML Parser`; Data Transformation only goes the other way
  and cannot join — the relational module is the Datasection Toolbox. A Web Service module has no
  Resource URI. `File In` has no age filter.
- **Branch on links.** Set the SQL module's halt condition to *Never* and put
  the decision on the out-links — a quoted `$ROWCOUNT('<section>')$` test on one, "run when all other
  links using conditions have not run" on the other.
- **Paging has no precedent here.** Designed, not observed — nothing in the reference set pages.
- **Endpoints belong in a settings row**, and credentials go in a stored section and an authorization
  header rather than a request URI, with the signature carve-out noted under Process 1.
- **Validation before storage is the load-bearing idea.** It is the difference between an exception
  queue somebody works and a slow accumulation of documents attached to nothing.
- **Order the writes so a failure is recoverable.** Bytes, then catalog row, then move the source. A
  crash between the first two leaves an orphan a reconciliation run finds and a person can fix.
- **Alert on silence, not only on errors.** A scheduled process that quietly stops firing produces no
  errors at all.
- **Give a poison file somewhere to go** after a third failed attempt, or it is retried forever.
- **Deletion is unhandled by design** — reconciliation surfaces it as an orphan rather than a sync
  silently deleting a record.

## Sequencing the build

The five are not equally ready. Two depend on nothing that is still open; the three that write to the
store all wait on the same two answers. That ordering matters more than any estimate.

**Step 1** settles the questions — store access, and one binary write worth proving with a single
call. **Step 2** builds processes 1 and 5, neither of which touches the store. **Step 3** builds 2, 3
and 4.

Process 2 sits in step 3 rather than step 2, which is a change from an earlier draft: its store call
is the same unproven binary write flagged above, so it cannot honestly be called unconditional.
Everything up to that call can be built and tested in parallel — the drop folder, the key parse, the
validation query and the exception queue are all independent of how the bytes travel. And the
reconciliation is worth building even if none of the sync is, because it measures the gap the rest of
the work would close.

Beyond the processes themselves, a build produces the SQL — key validation, the catalog schema and
its indexes, the three reconciliation comparisons — the store setup (columns, content types, indexed
fields, default views), and a runbook covering what to configure, what to test, and what to alert on.
That runbook needs to include the Synchronized Alerts Retriever: without it, a Call that waits on a
stalled subprocess never times out.

Field names, endpoints, credentials and environment identifiers are per-deployment and always need
setting locally. That is configuration rather than redesign, but it is real work. And nothing can be
validated against real data until it runs against real data.
