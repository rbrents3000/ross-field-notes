---
num: "003"
title: "Documents After the Lift and Shift"
subtitle: "When a cloud migration moves supplemental documents from SharePoint into Azure Blob, the bytes survive and everything people did outside the ERP stops working. What actually breaks, why, and a design that gives it back."
tag: "ROSS"
audience: "developer"
date: "2026-09-08"
system: "Ross ERP 8.0"
reading_time: "14 min"
excerpt: "The ERP stores no document records at all — association is metadata identity, worked out on the fly and forgotten. That makes the store swappable and it makes the loss invisible. A two-direction sync restores search and filing, and one half of it needs nothing from the vendor."
stats:
  - k: "Roles lost"
    v: "4 of 5"
    note: "only storage is replaced"
  - k: "Document rows"
    v: "Zero"
    note: "the ERP persists none"
  - k: "Vendor asks"
    v: "None"
    note: "for the half you can build now"
  - k: "Decides it"
    v: "1 test"
    note: "an afternoon on a test system"
related:
  - "g-002-crystal-web-services"
key_refs:
  - "DATA_ZONE_CONTROLS"
  - "ERPSharePointMapping.xml"
  - "DocumentService"
  - "MaxContentFileCount"
reviewed: "2026-09-08"
verified: "Verified against Ross 8.0 source and IAF 9.3 documentation"
---

> This is the plain-Markdown copy of a Field Guide. The formatted version, with the
> diagrams and light/dark reading view, lives at
> `fieldnotes.ryanbrents.com/g/g-003-lift-and-shift-documents`.

A lift and shift moves supplemental documents — signed delivery tickets, supplier paperwork,
scanned invoices — out of SharePoint document libraries and into Azure Blob Storage. The bytes
arrive intact. Almost everything people did with those documents stops working.

## The association mechanism is a metadata match

The ERP does not store documents and does not store references to them. When a screen opens it
emits the record's key values, and the document panel turns those into a query: *give me items
whose columns match these*. The labels on the file are the entire association mechanism. Nothing
is registered anywhere.

Three consequences follow, and they are unusually clean:

- **Nothing can be orphaned.** Re-pointing the store touches no transactional data, because no
  transactional data references a document. There is genuinely nothing to migrate.
- **Association is portable.** A document belongs to an order because its columns say so — so
  anything that can write a labelled file makes it appear in the ERP.
- **Nothing can detect a gap.** Because the ERP records nothing, no part of the estate can tell
  you a document that *should* exist is missing.

## Five roles, one store

SharePoint was doing five jobs at once. The replacement does one.

| # | Role | Replaced by blob? |
|---|---|---|
| 1 | Content store — holds the bytes | Yes, cleanly |
| 2 | Metadata catalog — holds the ERP keys | Barely |
| 3 | Query surface — a view for people, an API for machines | No |
| 4 | Write endpoint — accepts documents from anyone with labels | No |
| 5 | Governance boundary — permissions, retention, audit | Changes shape |

Blob is not label-less: index tags are queryable, ten per object, strings only. What it has no
version of is a front door — no page a person opens, no columns, no views, no search box. The
tooling is admin tooling.

## The narrowing is the contract, not the platform

"Limited metadata" sounds like a property of the storage. It is not. Blob carries ten queryable
tags plus an 8&nbsp;KB bag; a SharePoint library carries twenty indexed columns against a
thirty-million-item capacity. Both ends are wider than what passes between them.

The ERP emits an item name and an ordered run of key values — two to five of them, most commonly
three — and a mapping file names them positionally. What travels is the transaction's *identity*:
which company, which division, which order. Not its description. No customer name, no date, no
amount, unless the client scrapes those off the screen separately.

That is narrow, and you inherit it on any store you pick. Changing stores does not widen it.

## Half the feature is frozen, half shipped blob in 2021

The ERP-side piece really has not moved: the vendor's release comparison carries the same three
Document Connect rows unchanged through the 8.0.2 edition of March 2022, and no Azure or blob row
was ever added to it.

The client side kept moving. Blob arrived in IAF 9.3.12, March 2021, described as a new cloud model
in which an Azure app service connects to table storage, blob storage *and* SharePoint. It was not
a Document Connect decision — it rode along on a cloud-enablement wave.

Two products with one name. The ERP piece only ever emits "this screen is showing order 12345" and
stops. Everything about *where* and *how* lives in the client layer.

## The design: one store, two directions

Blob stays the store the ERP reads. A SharePoint site you control becomes the interface people work
in. A sync keeps them aligned, and provenance stamps stop the two directions chasing each other.

**Direction A — store out to SharePoint.** A scheduled job enumerates the container and creates a
SharePoint item carrying the ERP keys as columns. Read-only against the vendor's store, so the worst
it can do is fall behind. Two versions:

- **Catalog** — a list item with the keys and a link. The bytes never leave the store. Nothing
  duplicated, tenant storage untouched, no second copy to retain or discover. No Office editing, and
  search cannot reach inside document content.
- **Full copy** — the document itself in a library. Office editing and in-content search, at the
  cost of two retention obligations and two things that can drift.

Start with the catalog. It upgrades to a full copy by adding one fetch step; the reverse is a
deletion exercise.

**Direction B — SharePoint back into the store.** The mirror image, and the one that rests on an
open question: does a file written directly into the container become a document the ERP can find?
If the key-to-document association lives in table storage rather than on the blob object, writing a
file produces an orphan — bytes with no registration, which the panel will never return.

## Breaking the loop

Three rules, none of them novel:

1. **Stamp provenance on every write.** A tag on one side, a hidden column on the other, recording
   which direction created the item. Each direction skips what the other wrote.
2. **Compare content, not timestamps.** Carry a content hash, so a genuine re-upload is
   distinguishable from an echo.
3. **Allow edits on one side only.** Otherwise you need conflict resolution, which is a project of
   its own.

## The test that decides it

Upload one document through the ERP panel on a test transaction, then look at what landed in the
container.

- **Keys on the object** — index tags or metadata carrying company, division, number, party.
  Direction B works; copy the pattern.
- **Object is bare**, or named by an opaque identifier — the association lives elsewhere. Direction
  B as designed is dead, and the fallback is to route uploads through the ERP rather than around it.

Do not skip the test and build anyway. An orphaned document is the worst failure here because it
looks like success: the file uploads, no error appears, and it simply never shows on the order.

## Five processes

The implementation is five integration processes. Every module type in them is one already in
service in a working configuration.

1. **Get access token** — a shared subroutine. There is no packaged OAuth credential type in the
   service dialog, but the authorization header accepts a dynamic value, so the token lifecycle is
   assembled from two HTTP calls and a cached value that survives a restart.
2. **Scan ingest** — the piece with no vendor replacement. Reads the drop folder, derives the key
   from the filename, computes a content hash, and **validates against the ERP before storing
   anything**. Unresolved keys are quarantined, never guessed at. Moving the file out of the drop
   folder *is* the watermark, which makes it naturally idempotent.
3. **Catalog sync** — Direction A. A Conditional Halt immediately inside the loop is the guard that
   drops anything the other direction wrote.
4. **Reverse sync** — Direction B, with a read-back step that confirms the document actually returns
   rather than assuming the write succeeded.
5. **Nightly reconciliation** — the instrument the ERP has never had. Compares the catalog against
   transactions that should have produced a document and publishes three counts: orphans, missing
   documents, and divergence.

## Things worth arguing about before building

- **Validation before storage is the load-bearing idea.** It is the difference between an exception
  queue somebody works and a slow accumulation of documents attached to nothing.
- **Order the writes so a failure is recoverable.** Store the bytes, write the catalog row, then move
  the source file. A crash between the first two leaves an orphan a reconciliation run finds. Reverse
  the order and you get a catalog row pointing at nothing, which nothing can fix.
- **Alert on silence, not only on errors.** A scheduled process that quietly stops firing produces no
  errors at all.
- **Paging is not optional.** Both syncs enumerate collections that return a continuation marker.
- **Deletion is unhandled by design** — reconciliation surfaces it as an orphan rather than a sync
  silently deleting a record.

## What is genuinely unresolved

- **Which client type a browser session reports.** The routine supplying the panel's key values exits
  silently unless the session reports one specific client type, and the browser is separately
  enumerated. Minutes to determine on a live system; no amount of document research substitutes.
- **Whether the store is a container you can be granted access to**, or a vendor-operated app service
  in front of one. The documented service URL is an app-service host with a mandatory tenant
  identifier, which is not the shape of a storage account.
- **The service's wire contract.** No endpoint, verb, schema or auth scheme appears in any manual —
  only client-side settings pointing at it. Which is precisely why nothing here is designed against
  it.
