---
num: "028"
title: "My blanket PO inquiry shows the wrong quantities on releases. Bug?"
tag: "ROSS"
audience: "user"
source: "form"
date: "2026-07-23"
system: "Ross ERP 8.0"
reading_time: "4 min"
excerpt: "Probably not your setup. It's a known 8.0 SP2 fix — pre-SP2, blanket-release inquiry returns bad data."
question: "When I inquire on a blanket PO, the release lines show numbers that don't reconcile — quantities that are just wrong. Standard-order inquiry is fine. Did I build the blanket wrong?"
fix: "Likely not your setup. This is a known Ross 8.0 SP2 fix (ticket 477310): before SP2, purchase-order inquiry returns bad data for release-type ('R') blankets. SP2 adds a corrective view. If you're pre-SP2, that's your answer."
margin_notes:
  - "not your setup ↴"
  - "SP2 ticket 477310 →"
  - "check RENCS_VERSION first"
---

Reassuring news first: if standard-order inquiry is clean and only blanket *releases* look wrong, you very likely didn't do anything wrong. There's a documented defect with your name on it.

## What it is

Ross 8.0 SP2 ships a fix, ticket **477310**, whose reason line is blunt about it: *inquiry returning bad data for release-type blankets.* Before that fix, the purchase-order inquiry can report quantities on release-type (`"R"`) blanket lines that simply don't reconcile against the underlying detail. The blanket master and standard POs are unaffected — it's specific to the release view.

The fix is a corrective database view that the inquiry reads instead of the join that was returning the bad numbers:

```dml
@program V80SP2_PATCH.DML  ·  TICKET 477310
@note THE SP2 FIX — A CORRECTIVE VIEW THE INQUIRY READS INSTEAD
@reads pop_headers, pop_lines
@writes creates view purchase_inquiry_for_blanket
@risk absent before SP2 — that's exactly why release-type blankets read wrong
@highlight 2
! reason line, in spirit: "inquiry returning bad data for release type blankets"
ADD VIEW purchase_inquiry_for_blanket
    FROM pop_headers a, pop_lines b
    WHERE a.company_code = b.company_code
      AND a.division     = b.division
      AND a.po_number    = b.po_number
    SORT BY -a.order_date, a.po_number, b.po_line_number
```

> **field note** — this is a *display* defect, not a data defect. The releases themselves — the quantities you released, the commitments, the receipts — are fine underneath. It's the inquiry that lies. So don't go "correcting" release quantities to match a screen that's wrong; you'll turn a cosmetic bug into a real one.

## How to confirm it's this and not you

1. Check whether you're pre-SP2 — read `RENCS_VERSION` and `ERP_SERVICE_PACK` (see the note on telling your service-pack level). If the SP2 patch hasn't been applied, 477310 isn't in place.
2. Cross-check the release against the detail directly — the underlying quantities will reconcile even when the inquiry doesn't.
3. If you're already on SP2 and still see it, *then* it's worth looking at your setup — but that's the exception, not the first guess.

The move here is patience, not a rebuild: get to SP2 and the screen starts telling the truth.
