---
num: "003"
title: "Documents After the Lift and Shift"
subtitle: "When a cloud migration moves supplemental documents from SharePoint into Azure Blob, the bytes survive and everything people did outside the ERP stops working. What actually breaks, and why the loss is invisible from inside the system."
tag: "ROSS"
audience: "developer"
date: "2026-09-08"
system: "Ross ERP 8.0"
reading_time: "8 min"
excerpt: "The ERP stores no document records at all — association is metadata identity, worked out on the fly and forgotten. That is what makes the store swappable, and it is also why nothing in the estate can tell you a document that should exist is missing."
stats:
  - k: "Roles lost"
    v: "4 of 5"
    note: "only storage is replaced"
  - k: "Document rows"
    v: "Zero"
    note: "the ERP persists none"
  - k: "Keys on the wire"
    v: "2–5"
    note: "identity, not description"
  - k: "Blob arrived"
    v: "Mar 2021"
    note: "in the client layer, not the ERP"
related:
  - "g-004-two-directions"
  - "g-005-filing-processes"
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

First of three. The design that addresses this is in *One Store, Two Directions*; the process
implementations are in *Five Processes for Filing Documents*.

A lift and shift moves supplemental documents — signed delivery tickets, supplier paperwork, scanned
invoices — out of SharePoint document libraries and into Azure Blob Storage. The bytes arrive intact.
Almost everything people did with those documents stops working.

## The association is a metadata match

The ERP does not store documents and does not store references to them. When a screen opens it emits
the record's key values, and the document panel turns those into a query: *give me items whose
columns match these*. The labels on the file are the entire association mechanism. Nothing is
registered anywhere, and nothing is remembered afterwards.

Three consequences follow, and the first two are unusually good news:

- **Nothing can be orphaned.** No transactional data references a document, so re-pointing the store
  touches nothing. There is genuinely nothing to migrate on the ERP side.
- **Association is portable.** A document belongs to an order because its columns say so — anything
  that can write a labelled file makes it appear in the ERP, whether that is a person, a scanner or
  a job.
- **Nothing can detect a gap.** Because the ERP records nothing, no part of the estate can tell you
  that a document which *should* exist is missing. That is true today, before any migration.

## Five roles, one store

SharePoint was doing five jobs at once. The replacement does one — which is why the loss reads as
total rather than partial, and also why most of it is recoverable somewhere else.

| # | Role | Replaced by blob? |
|---|---|---|
| 1 | Content store — holds the bytes | Yes, cleanly |
| 2 | Metadata catalog — holds the ERP keys | Barely |
| 3 | Query surface — a view for people, an API for machines | No |
| 4 | Write endpoint — accepts documents from anyone with labels | No |
| 5 | Governance boundary — permissions, retention, audit | Changes shape |

Blob is not label-less: index tags are queryable, ten per object, strings only. What it has no
version of is a **front door** — no page a person opens, no columns, no views, no search box. The
tooling is admin tooling, which is fine for an administrator and useless for a clerk.

## The narrowing is the contract, not the platform

"Limited metadata" sounds like a property of the storage. It is not. Blob carries ten queryable tags
plus an 8&nbsp;KB bag; a library carries twenty indexed columns against a thirty-million-item
capacity. Both ends are wider than what passes between them.

The ERP emits an item name and an ordered run of key values — two to five of them, most commonly
three — and a mapping file names them positionally. What travels is the transaction's *identity*:
which company, which division, which order. Not its description. No customer name, no date, no
amount, unless the client scrapes those off the screen separately.

That is narrow, and you inherit it on any store you pick. **Changing stores does not widen it** —
which is why the store choice matters far less than the argument about it suggests, and why the
write path is the whole decision.

## Half the feature is frozen; the other half shipped blob in 2021

"Document Connect has not changed in years" is half true, and the half that matters explains why
nobody planned for this.

**The ERP side is frozen.** The vendor's release comparison carries the same three Document Connect
rows unchanged through the 8.0.2 edition of March 2022, and no Azure or blob row was ever added to
it. As late as September 2020 the install guide still required the application server to sit in the
same domain as SharePoint.

**The client side kept moving.** Blob arrived in IAF 9.3.12, March 2021, described as a new cloud
model in which an Azure app service connects to table storage, blob storage *and* SharePoint. It was
not a Document Connect decision — it rode along on a cloud-enablement wave.

Two products with one name. The ERP piece only ever emits "this screen is showing order 12345" and
stops, which is precisely why the store could be swapped without touching it. Everything about
*where* and *how* lives in the client layer.

## What breaks, ranked by how much it hurts

The ranking is counter-intuitive: the one everybody leads with belongs last.

1. **Scanned paper stops filing itself.** Fails silently. Documents stop reaching their orders and
   nothing announces it — discovered months later, when somebody needs a proof of delivery for a
   claim. Highest stakes, and the piece the product never provided.
2. **Governance changes shape.** Retention, hold and discovery were inherited from the tenant. A
   storage container has its own controls — immutability, legal hold, lifecycle rules — but
   content-policy enforcement is not among them.
3. **Nobody can find or add a document.** Loud, immediate, survivable. People complain on day one
   and adapt within a week, by emailing documents to each other — which is how they disappear.

## What is genuinely unresolved

Each is cheap to settle and none can be settled by reading.

- **Which client type a session reports.** The routine supplying the panel's key values exits
  *silently* unless the session reports one specific client type, and the browser client is
  separately enumerated. Minutes to determine on a live system. If it fails, the in-ERP panel is
  unavailable to that population and the outside-the-ERP surface is not a convenience — it is the
  entire product.
- **Whose container it is.** The documented service URL is an app-service host with a mandatory
  tenant identifier, which is not the shape of a storage account you can be handed. Ask for a signed
  URL or a periodic inventory export before designing anything that reads the store directly.
- **The wire contract.** No endpoint, verb, schema or auth scheme for the document service appears
  in any manual — only client-side settings pointing at it.

If you cannot enumerate or export your own documents, that is not a search problem — it is an exit
question, and it is worth raising while a migration is still being negotiated rather than after.
