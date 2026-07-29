---
num: "006"
title: "How do I batch-load invoices into Ross — AP, GL, or AR batch upload?"
tag: "ROSS"
audience: "developer"
source: "group"
date: "2026-07-29"
system: "Ross ERP 8.0"
reading_time: "8 min"
excerpt: "AP batch upload loads payables and GL batch upload posts journals — neither makes a customer invoice. An invoice you send out is a receivable: stage it in the AR batch tables and let the standard post program build it. And splitting one invoice per location is usually a setting, not an import."
question: "We used to email 3 non-stock invoices to a customer at month end. They've decided they want those invoices broken up and sent to all 65 of their locations at month end. Does anyone use an import routine to batch-upload invoices into Ross that populates all of the correct tables?"
restated: "Does Ross have a batch import for invoices that populates all the right tables — and which mechanism (AP batch upload, GL batch upload, or the AR transaction batch) actually creates the customer invoices you send out, per location?"
fix: "AP batch upload loads vendor bills (payables) and GL batch upload posts journals — neither creates a customer invoice. An invoice you send out is a receivable: stage it in the AR batch tables (`AR_BATCH_TRANSACTIONS` / `AR_BATCH_TRAN_LINES`) and run Transfer Batches to Transactions (`ar_u_batches_to_transactions`), which validates and posts it into `AR_TRANSACTIONS` with its tax and GL postings. And before building anything — splitting one invoice into one-per-location is usually the `SOP_SEPARATE_INVOICE_PER_SHIP` customer flag, not an import."
reviewed: "2026-07-29"
verified: "Verified against Ross 8.0 source"
key_refs:
  - "AR_BATCH_TRANSACTIONS"
  - "AR_BATCH_TRAN_LINES"
  - "ar_u_batches_to_transactions"
  - "SOP_SEPARATE_INVOICE_PER_SHIP"
related:
  - "q-004-blanket-sales-orders"
applies_when:
  - "You reached for `AP batch upload` (or `GL batch upload`) to load invoices you *send to a customer*."
  - "You want to split one month-end invoice into one per ship-to location."
  - "You need to load invoice data from outside Ross and have it land in all the right tables."
margin_notes:
  - "AP = bills in, AR = invoices out ↴"
  - "stage the batch, never touch `AR_TRANSACTIONS` →"
  - "per-location can be a customer flag"
  - "recurring lives on AP, not AR"
---

The instinct is right — Ross *does* have a "batch upload," and reaching for it beats hand-keying 65 invoices. The catch is that there are three different things wearing that name, they land in three different ledgers, and only one of them produces an invoice you can actually send to a customer. Pick the wrong door and you'll load your data perfectly into the wrong place.

> **in plain terms** — an invoice you *email to a customer* is money they owe you: a **receivable** (AR). AP batch upload loads money *you* owe suppliers (payables), and GL batch upload just posts accounting journals. Neither one makes a customer invoice, no matter how clean your file is.

## 01 · First — is this even an import problem?

Before building anything, settle one question: **how do these invoices get created today?** It decides whether you need code at all.

- **Billed through sales orders / despatches.** Then "one invoice per location" is a **setting**, not an import. The customer-master flag `CUSTOMERS(SOP_SEPARATE_INVOICE_PER_SHIP)` tells the invoicing run to cut a **separate invoice per ship-to** instead of consolidating; the `CONSOLIDATED_IN_USE` controls govern the merging in the other direction. If the 65 locations are ship-tos under the one customer, flip the flag and the normal run splits them for you. Going from 3 to ~65 invoices a month is nothing for that run.
- **Keyed as direct, non-stock invoices** (no order, no shipment — the usual shape for a flat month-end charge). Then there's no despatch to split, so the flag doesn't apply: each location is simply **its own AR invoice document**. Now you genuinely have ~65 invoices to create — and *that* is what the AR batch is for.

> **rule of thumb** — don't build an importer until you've ruled out `SOP_SEPARATE_INVOICE_PER_SHIP`. The cheapest integration is the one you configure instead of code.

## 02 · Three doors labelled "batch upload"

They look interchangeable from the menu. They are not — each lands in a different ledger.

| Door | What it loads | Lands in | Makes a customer invoice? | Reach for it when |
| --- | --- | --- | --- | --- |
| **AP batch upload** | Vendor bills (payables) | AP open items | **No** | You're loading *supplier* invoices into AP to pay them |
| **GL batch upload** | Journal lines (DR/CR) | GL accounts only | **No** | You're posting journals/accruals with no subledger |
| **AR transaction batch** | Customer invoices (receivables) | `AR_TRANSACTIONS` (+ tax + GL) | **Yes** | You're creating invoices to send to a customer |

```cards
AP | AP batch upload | Loads vendor bills you owe. Populates AP — not a customer invoice. This is the one most people reach for by name, and it's the wrong ledger for billing out.
GL | GL batch upload | Uploads journal lines straight to GL accounts. No subledger, no open item, no document to email. It books the revenue and gives the 65 locations nothing to receive.
AR | AR transaction batch | The receivables side. Stages invoice headers + lines, then posts them as real customer invoices with tax and GL. The only door that produces something you can send.
```

> **caution** — GL batch upload is the sneaky wrong answer. It'll happily post the revenue to the right accounts, so the numbers look done — but no AR open item exists, nothing ages on the customer's account, and there's no invoice to print or email. Right money, no document.

## 03 · The door that makes a customer invoice: the AR batch

Ross's AR batch is a **staging area with a posting engine** — exactly the shape you want. You build the invoices in the batch tables, then one program validates them and turns them into live receivables.

| Table | Role |
| --- | --- |
| `AR_BATCH_TRANSACTIONS` | The **batch** — one per run: its type (`TTYPE_IN` for invoices), control total, posting period, status |
| `AR_BATCH_TRAN_LINES` | **One row per invoice** — customer, invoice number, amount, tax, terms, description (one per location) |
| `AR_BATCH_GL_POSTINGS` | The **GL distribution** — revenue and AR accounts |
| `AR_BATCH_TAX_TRANSACTIONS` | **Tax** detail per transaction |
| `AR_TRANSACTIONS` | The **live** AR ledger — written by the post program, never by you |

> **caution** — pick the right *shape* of invoice first. The AR batch creates **financial open items**: a customer, an amount, tax, terms, a description, and a GL distribution. That's exactly right for a flat non-stock charge — a fee, a service, a monthly management charge. If your invoice instead needs itemised **non-stock lines** — quantities, unit prices, per-line tax — that's the **AR Sales Invoice** (`ar_u_invoice`, non-stock lines via `LINE_TYPE_NONSTOCK`), which posts to `AR_TRANSACTIONS` directly. The batch won't hold line detail; the Sales Invoice has no bulk loader. Know which one your "invoice" really is before you build.

```steps
1 | Stage the invoices in a batch
where: ar_t_maintain_batch_transactions
do: open an invoice batch and add one row per location — the customer, the invoice amount, its tax and terms, a description, and the revenue/AR distribution. This is where an import feeds the rows instead of a person keying them.
sys: writes one `AR_BATCH_TRANSACTIONS` batch header (typed `TTYPE_IN`, with a control total you enter) and one `AR_BATCH_TRAN_LINES` per invoice, plus `AR_BATCH_TAX_TRANSACTIONS` / `AR_BATCH_GL_POSTINGS`. Nothing has hit the ledger yet — the batch carries a control total and a status flag.

2 | Review, then post the batch
where: ar_u_batches_to_transactions
do: print the edit list to eyeball it, then run "Transfer Batches to Transactions" to turn the staged batch into real invoices.
sys: it validates the batch — it must **balance to its control total** and post into an **open period** — then writes `AR_TRANSACTIONS` with its tax and GL postings. Open items now exist on the customer's account and can be printed or emailed per location.
note: `ar_r_batch_transactions_report` is the edit list / audit — run it before posting to catch an out-of-balance or wrong-period batch on paper, not in the ledger.
```

> **in the system** — the AR batch tables *are* the interface tables. That's the whole answer to "populates all of the correct tables": you populate `AR_BATCH_*`, and the post program fans the result out to the ledger, the tax lines, the GL postings, and the customer's open-item balance — consistently, every time.

## 04 · A repeating monthly bill? Mind the recurring gap

The scenario is a *month-end* invoice — the same charge, every period. That's the textbook use for a **recurring** feature, so it's worth knowing where Ross put one and where it didn't.

> **field note** — the asymmetry strikes again. On the **payables** side Ross ships a recurring-charge subsystem: define the charge once in `ap_t_maintain_recurring_charges`, then generate each period's postings with `ap_u_recurring_to_transactions`. There is **no AR equivalent** — no "recurring customer invoice" generator. So a standing month-end *customer* bill can't lean on a built-in generator the way a standing *vendor* bill can. On the sell side your repeatable options are: a **saved batch you re-post** each month (same rows, new period), the **per-ship-to invoicing run** if it's order-based, or a small **custom recurring feed** into the AR batch. The convenience lives on payables — same story as the batch upload.

## 05 · If the invoice data comes from outside Ross

There's no packaged "AR invoice upload" screen the way AP has one — but you don't need the vendor to ship you one, because the batch tables are a documented staging target and the post program is the standard engine. The integration is small:

```steps
1 | Land the rows in the batch
do: from your spreadsheet / feed, write one `AR_BATCH_TRANSACTIONS` header per location and its `AR_BATCH_TRAN_LINES`, tagged to a batch number — either directly or by driving `ar_t_maintain_batch_transactions` through a service.

2 | Let Ross do the rest
where: ar_u_batches_to_transactions
do: run `ar_u_batches_to_transactions`.
sys: it validates and posts — real invoices, tax, GL, open items. Your code never touches a live table.
```

This is the same architecture as the **AP invoice import** you already know — staging tables → validate → post into the subsystem — just one ledger over. That's why AP batch upload *felt* right: correct shape, wrong side of the house.

> **field note** — the safe rule for any Ross import: land your data in the subsystem's **batch / interface** tables and run the **standard update program**. Never `INSERT` into a live transaction table (`AR_TRANSACTIONS`, `AP_TRANSACTIONS`, GL postings) yourself — you'll skip the validation, the tax build, and the balance updates the update program exists to do, and quietly corrupt the subledger.

## 06 · So which do I reach for?

> **rule of thumb** — if you're emailing it to a **customer**, it's **AR** — and the mechanism is the **AR transaction batch**, posted by `ar_u_batches_to_transactions`. AP batch upload and GL batch upload are real, useful features aimed at *payables* and *journals* respectively; they just don't make a customer invoice. And check `SOP_SEPARATE_INVOICE_PER_SHIP` first — the per-location split you're after may already be a flag away.

The short version: three doors, one ledger each. The one that ends with an invoice in a customer's inbox is the AR batch — stage it, post it, done.

## 07 · Where this comes from

- `ar_t_maintain_batch_transactions` — enter / maintain AR batch invoices (the staging screen); an invoice batch is typed `TTYPE_IN`.
- `ar_r_batch_transactions_report` — the batch edit list / audit; review a batch before posting.
- `ar_u_batches_to_transactions` — "Transfer Batches to Transactions"; validates the batch (control total, open period) and posts it into `AR_TRANSACTIONS` with tax and GL postings.
- `ar_u_invoice` — the AR Sales Invoice update engine for lined direct invoicing (non-stock lines via `LINE_TYPE_NONSTOCK`); posts to `AR_TRANSACTIONS` directly.
- `ap_t_maintain_recurring_charges` / `ap_u_recurring_to_transactions` — AP's recurring-charge generator, shown for contrast: there is no AR counterpart.
- `sop_t_auto_inv_from_desp` — order/despatch invoicing; reads `CUSTOMERS(SOP_SEPARATE_INVOICE_PER_SHIP)` to split one invoice per ship-to.
- `gl_s_l_batch_upload` — the GL batch-upload path (journals only; no subledger).

The sell-side pricing counterpart — standing quantity-and-price agreements a customer draws down — is covered in [the blanket sales orders rundown](/q/q-004-blanket-sales-orders).
