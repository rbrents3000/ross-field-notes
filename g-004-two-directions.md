---
num: "004"
title: "One Store, Two Directions"
subtitle: "A design that lets the ERP keep reading cloud storage while people keep working in SharePoint — one half safe to build today, the other resting on a single question you can answer in an afternoon."
tag: "ROSS"
audience: "developer"
date: "2026-09-08"
system: "Ross ERP 8.0"
reading_time: "8 min"
excerpt: "Blob keeps the bytes and the ERP keeps reading it; a site you control becomes the interface. A catalog restores search without duplicating a single document. The reverse direction rests on whether a file written straight into the container is one the ERP will actually return — an afternoon to establish, and a disaster to assume."
stats:
  - k: "Directions"
    v: "2"
    note: "one safe, one conditional"
  - k: "Duplication"
    v: "None"
    note: "if you take the catalog"
  - k: "Loop guard"
    v: "1 module"
    note: "a provenance stamp"
  - k: "Decides it"
    v: "1 upload"
    note: "then look at the container"
related:
  - "g-003-lift-and-shift-documents"
  - "g-005-filing-processes"
key_refs:
  - "Blob index tags"
  - "Conditional Halt"
  - "List View Threshold"
reviewed: "2026-09-08"
verified: "Verified against Ross 8.0 source and IAF 9.3 documentation"
---

> This is the plain-Markdown copy of a Field Guide. The formatted version, with the
> diagrams and light/dark reading view, lives at
> `fieldnotes.ryanbrents.com/g/g-004-two-directions`.

Second of three. The diagnosis is in *Documents After the Lift and Shift*; the process
implementations are in *Five Processes for Filing Documents*.

## Let each side do what it is good at

The store keeps the bytes and the ERP keeps reading it. A site you control becomes the interface
people work in. A sync keeps the two aligned, and a provenance stamp stops them chasing each other.

**Direction A** copies out of the store into a site you own. It is read-only against the vendor's
store, so the worst it can do is fall behind — it cannot corrupt anything, and it cannot create
documents the ERP will not recognise.

**Direction B** pushes newly added documents back in, so a person dragging a file into SharePoint
sees it appear on the order inside the ERP.

## Direction A has a choice inside it

- **Catalog** — a list item with the keys and a link. The bytes never leave the store. Nothing
  duplicated, tenant storage untouched, no second copy to retain or discover. You do not get Office
  editing, and search cannot reach inside document content.
- **Full copy** — the document itself in a library. Office editing, in-content search and offline
  sync, at the cost of two retention obligations, two places a discovery request must look, and two
  things that can drift apart.

Start with the catalog. It upgrades to a full copy by adding one fetch step; the reverse is a
deletion exercise.

## Direction B rests on one question

Does a file written directly into the container become a document the ERP can find? The vendor's own
release note describes the cloud document service as connecting to *table storage, blob storage and
SharePoint* — and table storage is a very plausible home for the key-to-document association. If it
lives there rather than on the object, writing a file produces an **orphan**.

Upload one document through the panel on a test transaction, then look at what landed:

- **Keys on the object** — Direction B works. Copy the pattern.
- **Object is bare**, or named by an opaque identifier — the association lives elsewhere. Route
  uploads through the ERP instead.

Do not skip the test and build anyway. An orphan is the worst failure here because it looks like
success: the upload works, no error appears, and the document never shows on the order.

## Three rules that break the loop

1. **Stamp provenance on every write.** A tag on one side, a hidden column on the other, recording
   which direction created the item. Each direction skips what the other wrote. This alone breaks the
   cycle.
2. **Compare content, not timestamps.** A content hash distinguishes a genuine re-upload of a changed
   document from an echo of an unchanged one.
3. **Allow edits on one side only.** Otherwise you need conflict resolution, and conflict resolution
   on documents is a project of its own.

None of this is novel. Provenance marking, idempotent keys and a single writable side are the
standard shape of every bidirectional sync that has ever worked. The risk in this design is not the
loop — it is the open question above.

## The honest trade

- **Duplication**, but only if you choose the full copy. The catalog stores no second copy at all,
  which is the main reason to prefer it.
- **Read access is a hard dependency** for both directions. If the store turns out to be a shared
  service rather than a container that is yours, this design does not run.
- **Someone owns it.** A sync that silently stops is a sync that quietly stops filing documents.
  Alert on silence, not only on errors.
- **It is not a whole fix.** It gives the scanned-document pipeline somewhere good to write, which is
  most of the battle — but capture, validation and an exception queue with a named human still have
  to exist.

## Where I would start

Build Direction A as a catalog and ship it on its own. It restores the most visible loss, carries
almost no risk, needs only read access, and is useful whatever happens next. Run the test in parallel
— it costs an afternoon and it decides whether the second half is a build or a redesign.
