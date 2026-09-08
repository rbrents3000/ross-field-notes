---
num: "005"
title: "Five Processes for Filing Documents"
subtitle: "The implementation: a shared token subroutine, scanned-document capture, two sync directions, and the nightly reconciliation that is the only thing able to tell you a document is missing."
tag: "EMF"
audience: "developer"
date: "2026-09-08"
system: "Ross ERP 8.0"
reading_time: "10 min"
excerpt: "Every module type in these five designs is one already in service in a working configuration. Validation before storage is the load-bearing idea; write ordering decides whether a failure is recoverable; and three of the five can be built before a single open question is answered."
stats:
  - k: "Processes"
    v: "5"
    note: "module by module"
  - k: "Buildable now"
    v: "3 of 5"
    note: "no open questions"
  - k: "Untried step"
    v: "1"
    note: "a binary request body"
  - k: "Load-bearing"
    v: "Validation"
    note: "before anything is stored"
related:
  - "g-003-lift-and-shift-documents"
  - "g-004-two-directions"
key_refs:
  - "Conditional Halt"
  - "Data Store"
  - "File In"
reviewed: "2026-09-08"
verified: "Module inventory derived from a working EMF configuration"
---

> This is the plain-Markdown copy of a Field Guide. The formatted version, with the
> module diagrams and light/dark reading view, lives at
> `fieldnotes.ryanbrents.com/g/g-005-filing-processes`.

Third of three. The diagnosis is in *Documents After the Lift and Shift*; the design is in *One
Store, Two Directions*.

Design sketches only — module shapes, not exported configuration. **Every module type shown is one
already in service** in a working configuration; nothing rests on an untried capability.

## Process 1 — get access token

Called by the other four. Fetches a bearer token by client credentials, caches it, and returns it
through the process interface. The cache survives a restart, so a token is fetched once per expiry
window rather than once per run.

Build this first. There is no packaged OAuth credential type in the service dialog, but the
authorization header accepts a dynamic value, so the lifecycle is assembled by hand from two calls
and a cached value.

## Process 2 — scan ingest

The piece with no vendor replacement, and the one whose failure is silent. Reads the drop folder,
derives the key from the filename, computes a content hash, and **validates against the ERP before
storing anything**. Anything that does not resolve goes to an exceptions folder and a named human —
it is never guessed at.

There is no watermark: moving the file out of the drop folder *is* the watermark, which makes the
process naturally idempotent and means a transient failure simply leaves the file for the next run.

## Process 3 — catalog sync

Direction A. Enumerates the container, skips anything the other direction wrote, and creates an item
carrying the keys plus a link. A Conditional Halt immediately inside the loop is the guard that drops
anything tagged as having come from the other side — that single module is what stops the echo.

## Process 4 — reverse sync

Direction B, with one addition over its mirror: a step that reads the document back by key to confirm
it actually returns, rather than assuming the write succeeded. If it does not, the document is an
orphan — and you want to know on the first one, not the thousandth.

## Process 5 — nightly reconciliation

Everything else moves documents. This is the only one that can tell you a document is **missing**,
and because the ERP persists no document state, nothing else in the estate can.

Compare the catalog against transactions that should have produced a document, and publish three
counts: **orphans** (rows matching no transaction), **missing documents** (the measurable size of the
coverage gap — the number nobody has today), and **divergence** (filed one way but not another).

Make the assertion re-runnable over *any* period, not a trailing thirty days. Retention runs to
years, and a coverage hole opens the day a new user first runs a report.

## Things worth arguing about before building

- **One real exception.** Sending a binary request body outward is documented but not exercised
  anywhere observed — every existing call either receives binary or sends text. Processes 2 and 4
  both depend on it. Prove that single call before building around it.
- **Validation before storage is the load-bearing idea.** It is the difference between an exception
  queue somebody works and a slow accumulation of documents attached to nothing.
- **Order the writes so a failure is recoverable.** Bytes, then catalog row, then move the source. A
  crash between the first two leaves an orphan a reconciliation run finds and a person can fix.
  Reverse the order and you get a catalog row pointing at nothing, which nothing can fix.
- **Alert on silence, not only on errors.** A scheduled process that quietly stops firing produces no
  errors at all.
- **Paging is not optional.** Loop the continuation marker until it comes back empty, or you will
  quietly process only the first page and believe you are current.
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
