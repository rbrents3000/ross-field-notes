---
num: "027"
title: "How do I tell which Ross 8.0 service pack I'm actually running?"
tag: "APTEAN"
audience: "developer"
source: "form"
date: "2026-07-23"
system: "Ross ERP 8.0"
reading_time: "5 min"
excerpt: "The folders lie. Two parameters record your installed service-pack level — everything else just tells you which patches are sitting on disk."
question: "We're on 8.0, but nobody can tell me whether we ever applied SP1 or SP2. The conversion folder has a mix of sp1 and sp2 files. How do I know what's actually live?"
fix: "Read the RENCS_VERSION and ERP_SERVICE_PACK parameters in the finance database — that's what records the installed level. The v80sp#_*.dml files in the vendor cvt tree only tell you which patches are available to apply, not which ones ran."
margin_notes:
  - "source ≠ installed ↴"
  - "RENCS_VERSION + ERP_SERVICE_PACK →"
  - "sp files = shipped, not applied"
---

This trips people up because the code tree looks like an answer and isn't one. A folder full of `v80sp2_*.dml` files tells you the SP2 patch *shipped* to you. It says nothing about whether anyone ran it.

## What actually records your level

Two global parameters in the finance database:

- **`RENCS_VERSION`** — the release, e.g. `"8.0"`.
- **`ERP_SERVICE_PACK`** — the service-pack level. Emptied at GA, then set as each SP is applied.

They're maintained by the conversion program that's meant to run *last* in any database conversion — it stamps the version and clears the SP field. Read straight from the database, that pair is the source of truth. The filesystem is not.

```dml
@program V80_UPDATE_RENCS_VERSION.DML
@note RUNS LAST IN A CONVERSION · STAMPS THE INSTALLED LEVEL
@reads rencs_version, erp_service_pack
@writes rencs_version, erp_service_pack
@risk the files on disk are NOT this — only these two params are authoritative
@highlight 3,6
! this is what records where you actually are:
IF #rencs_version <> "8.0" THEN
    SET PARAMETER rencs_version = "8.0"
END_IF
IF #erp_service_pack <> "" THEN
    SET PARAMETER erp_service_pack = ""    ! emptied at GA; set as each SP applies
END_IF
```

> **field note** — a backup that contains SP2 files while the live tree only has SP1 is a classic head-scratcher. It usually means the environment was rolled back, or refreshed from an SP1 baseline and the SP2 patch was never carried forward. The parameters settle it; the folders will just keep arguing.

## Where the SP patches live

Service packs ship as numbered conversion scripts in the vendor `cvt` tree — `v80sp1_patch.dml`, `v80sp2_patch.dml`, and a swarm of `v80sp#_<ticket>_*.dml` data scripts. Each `_patch.dml` is a metadata patch that's safe to re-run; the ticket-numbered files carry the data changes. Reading them tells you exactly *what a given SP would do* — genuinely useful when you're deciding whether to apply it — but it is not evidence that it ran.

## The practical check

1. Query `RENCS_VERSION` and `ERP_SERVICE_PACK` in finance.
2. If the SP field is blank, you're on GA — or the stamp never ran, which is worth confirming.
3. Match that against which `v80sp#_patch.dml` scripts you have on disk, and you know both what's installed and what's available to move up to.

Do this before you debug anything version-sensitive. Half of "is this a bug?" turns out to be "we're a service pack behind and didn't know it."
