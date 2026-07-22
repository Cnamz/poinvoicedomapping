# PO–Invoice–DO Mapping (RVN 1020) — Item Mapping Validation

> Read this before doing anything else in this project. It's a portable brief — if this folder
> gets copied to another computer, this file travels with it and should give a fresh Claude
> session everything it needs without the user re-explaining from scratch.

## 1. Who/what this is for

**RVN 1020** is a subsidiary of Tipco Asphalt Group — a trading company that sources and sells
road construction materials (asphalt-related) and road furniture (guardrails, signage,
barriers). The user is an **intern** working on the Procurement workflow, not a full-time
employee.

**Existing manual workflow** (before this system): Trading Team sends purchase request →
Procurement finds supplier & requests quotation → approved quotation goes to Sales Team →
Procurement tracks PO/DO/Invoice documents → Sales Operations does goods receiving →
Accounting (AP) verifies documents and pays supplier.

**Pain point**: high volume of PO/DO/Invoice documents; manual PO-DO-Invoice matching is slow
and error-prone.

**Project objective**: build a **PO–Invoice–DO Mapping** system to (1) auto-reconcile
PO/Invoice data, (2) reduce manual work, (3) let Procurement monitor remaining PO balance.

**Full solution architecture (4 tasks + 1 future task)** — this repo only covers a slice of
this:
1. Document Processing — upload PO/DO/Invoice images to SharePoint → AI classification &
   extraction → normalize/structure data
2. Matching Engine — Header Mapping (match same-transaction document sets) → Line Mapping
   (match at item level using Oracle ERP master data) → compute Reconciliation data (incl.
   remaining PO balance)
3. Data Validation — 3 failure cases (Header/Line/Reconciliation mapping fail), each with a
   decision flow → manual review by user
4. Data Presentation — generate reports from validated data
5. (Future) Oracle ERP Integration — push validated data back into Oracle

**Confirmed detailed flow**: OCR Extraction → **Item Mapping** (this screen, user validation,
inserted before Header Mapping) → Header Mapping → Line Mapping → Reconciliation.

**Known risks**: master data quality (stale/incomplete Supplier Master hurts matching
accuracy), document format variability across suppliers (affects OCR), UOM mismatches between
supplier and RVN naming.

## 2. What's actually in this repo

This repo is **only the Item Mapping Validation screen** — step 2-3 of the 5-stage pipeline
above. It's a single self-contained static web page:

- `index.html` — the whole app (React, inline via Babel-standalone, no build step)
- `styles.css` — the "Classical" design-system tokens/components (serif editorial look:
  Cormorant Garamond headings, Lora body, muted near-white background, accent gold/brown)
- `vendor/` — React, ReactDOM, Babel-standalone, and SheetJS(XLSX) **vendored locally** (not
  CDN) so the app works fully offline — this was done deliberately because the user plans to
  eventually move this project to a work computer with unknown network/firewall policy. Only
  remaining external dependency: Google Fonts import in `styles.css` (fails gracefully to a
  system font if blocked, doesn't break functionality).
- `master Item supplier1.xlsx`, `Supplier item master update 1.xlsx`, `Copy of RVN Inventory
  Org Structure Updated_Y2026 (3).xlsb` — real source Excel files (see §4).
- `.claude/launch.json` — a `static-server` config (`python3 -m http.server 8000`) so the app
  can be previewed in a browser (needed because the in-browser Babel/React setup requires
  `http://`, not `file://`, to run reliably in most tooling).

**Scope boundaries (confirmed)** — the intern's job is only:
- The frontend/logic of this one page
- Calling an API to read the input Excel from SharePoint and write the output Excel back

**Explicitly out of scope for the intern to build**: authentication/authorization (use the
org's existing SSO — which one is still unknown, needs asking IT), production
deployment/hosting/infrastructure, a permanent database or full backend, security review /
production hardening. If a request drifts toward "add login" or "deploy this for real users,"
flag that it's expanding scope beyond what's confirmed.

## 3. Confirmed UX/design decisions — don't re-litigate these

These came from real back-and-forth with the user/reviewers. Several are deliberate reversals
of a first instinct — don't re-suggest the rejected option without flagging that it was already
considered.

- **2-color confidence flag (green/red), not 3-color.** User action is binary (review or
  don't) — a middle "yellow" wouldn't change what the user does. Exact Match always hits 100%
  and lands in green.
- **No sticky/frozen columns**, anywhere. This was tried (freezing Invoice No./PO No./Supplier
  Name during horizontal scroll) and abandoned after repeated rendering bugs (transparent
  backgrounds let other columns bleed through during scroll, survived multiple fix attempts).
  **Do not suggest re-adding column freeze** — it's a closed decision, not an oversight.
- **No standalone "Overview" dashboard page anymore.** An earlier version had one (4 stat cards
  + confidence bars + a "go to table" button) — it was later removed entirely in a design
  re-sync. Now it's a single table view with a compact inline stats bar (ทั้งหมด / ยืนยันแล้ว /
  Exact-Fuzzy counts) plus loading-skeleton / error / empty states depending on data state.
  Don't reintroduce the old two-view navigation unless explicitly asked.
- **Expandable "PO vs. Invoice Reference" row** instead of showing all 18 columns flat — a
  warning badge ("⚠ Qty, ex.VAT ไม่ตรงกัน") on the main row signals a collapsed mismatch so
  users are forced to notice rather than skip the check silently.
- **8 editable fields** (PO/Invoice × Qty/Unit Price/ex.VAT/UOM): click-to-edit (not
  permanently-open inputs), Enter/blur to save, numeric fields auto-format with commas on blur,
  UOM stays free text. Original and edited values are tracked separately (audit trail — an
  accent-colored dot marks an edited field; hovering an un-edited field shows a pencil icon).
  Mismatch badges recalculate in real time.
- **Supplier/Item selection is dropdown-only, never free text** — must resolve to real
  Supplier Master entries (typos would corrupt master data), unlike the 8 editable fields
  above which have no master list to select from.
- **Two-tier confirmation state**: (1) does the row have an item selected at all (auto or
  manual), (2) did the *user* actively pick it. Rows above the confidence threshold auto-fill
  from Supplier Master and show "ระบบเลือกให้" (system-selected) until the user actually
  interacts; rows below threshold start empty, then show "ยืนยันแล้ว" (user-confirmed) once
  picked.
- **Bulk-confirm only applies to rows at/above threshold.** Below-threshold (red) rows can
  only be confirmed one at a time — a user can't accidentally bulk-skip a risky row.
- **Export flow (confirmed)**: a primary "เสร็จสิ้นและส่งออกข้อมูล" button in the header,
  disabled (with a tooltip explaining why) until every low-confidence row is confirmed.
  Clicking opens a confirm dialog; confirming triggers a real `.xlsx` download (via SheetJS)
  and clears the persisted `localStorage` state, plus shows a success toast.
- **Export gating does NOT depend on PO/Invoice mismatches.** A row with a red mismatch
  warning can still be exported as long as its supplier+item are confirmed — the mismatch
  badge is a visual reminder, not a hard block. User explicitly confirmed this rule.
- **Deliberately NOT built**: 3-color confidence; separate PO/Invoice header color-coding or
  column reordering (the expandable row already solves the "eyes have to jump" problem); a
  separate Status column (Confidence flag already encodes it); Validated-by/Validated-date as
  visible columns (kept as background audit log — though the *exported* Excel does include a
  "Confirmed By: Auto/User" column, since that file serves a different, IT-facing audience); a
  third legend color for "no OCR code" (too many colors). UOM was *initially* going to be cut
  from this screen but was **added back** after real user feedback.
- **A debug-only "[ทดสอบ] เลือกให้ครบทุกแถว" (force-fill all rows) button** appeared in one
  design sync — it bypassed validation entirely and was **removed at the user's explicit
  request**. If a future design re-sync reintroduces it, flag it again rather than silently
  re-adding it.

## 4. Data: what's real vs. mock right now

- **`SUPPLIER_MASTER`** in `index.html` is **real data**, built from **"Supplier item master
  update 1.xlsx"** — 28 real suppliers, 102 items, with both supplier item code/name/UOM *and*
  the linked RVN item code/name/UOM in one table. This supersedes an earlier, incomplete file
  ("master Item supplier1.xlsx") that had the supplier side only, with no RVN linkage.
- The broader RVN Item Master reference (1,361 items, English descriptions, Product
  Type/Category hierarchy) lives in **"Copy of RVN Inventory Org Structure Updated_Y2026
  (3).xlsb"**, sheet **"Final"** — not wired into the app directly, useful for cross-checking
  if needed.
- **`BASE_ROWS`/`EXTRA_ROWS`** (the transactional PO/Invoice/OCR rows, 114 total) are still
  **mock/synthetic transactions** — regenerated to reference the real suppliers/items above
  (previously used fictional supplier names), but not real OCR output. A real OCR-output Excel
  sample has not been provided yet (see task #4 below) — don't assume its exact column
  structure.

## 5. The real-world workflow this app must eventually support

```
[IT] OCR output → Excel file → placed on SharePoint
        ↓
[Web app] calls an API to pull that Excel file from SharePoint
        ↓
renders it as the Item Mapping Validation page (this repo)
        ↓
Procurement team reviews/edits in the browser
        ↓
exports the reviewed data as a *separate* output Excel file
        ↓
[IT] takes that output file into Phase 2 (Header/Line Mapping)
```

**Session/state rule (confirmed)**: each new Excel pull = a new batch/session. Working state
must be cleared **only after a successful export** — refresh/tab-close/browser-close must NOT
lose in-progress review state. **Already implemented**: rows persist to `localStorage` on every
change, cleared only in `confirmExport()`. Verified end-to-end.

**User's proposed batch model** (not yet confirmed with IT): one input Excel + one output
Excel per month, each month's batch independent (no document overlap across months). If IT
confirms months never mix, the incremental-vs-full-snapshot dedup question below mostly
resolves itself. Remaining edge case to flag to IT regardless: a document from month A that
failed OCR and gets reprocessed into month B's batch would still need a dedup key (Invoice No.
+ PO No.).

**Currently being explored (not yet built)**: instead of waiting on IT to approve a SharePoint
API/auth mechanism, the user proposed using **their own computer as a manual bridge** —
SharePoint syncs to a local folder via OneDrive/SharePoint sync client, the web app gets an
"Import Excel" file-picker button (parses via the already-vendored SheetJS) to load a real
`.xlsx` from that local folder instead of the hardcoded mock rows, and the existing Export
button's downloaded file gets manually moved back into the synced folder for SharePoint to
pick up. When IT eventually wants the "real" always-on version, only the read/write step swaps
from local-file to API — the rest of the app doesn't change. Two open questions before
building this were asked and not yet answered as of this file's writing:
1. Does the user already have a SharePoint folder synced locally to test with?
2. Should the importer match columns by **header name** (flexible, adapts if the real file's
   structure shifts slightly) rather than fixed column position/order, given task #4's real
   structure still isn't confirmed?

## 6. Where the UI design source lives

The screen's UI is authored/iterated in a **Claude Design** project, not directly in this
codebase. `index.html` is a hand-translated port (Claude Design's `.dc.html` template/binding
syntax → plain React) kept in sync on request.

- **Design project URL**: https://claude.ai/design/p/4a5cd25d-0323-4a8c-bd62-51e005d691fc?file=Item+Mapping+Validation.dc.html
- **Project ID**: `4a5cd25d-0323-4a8c-bd62-51e005d691fc`
- **Target file**: `Item Mapping Validation.dc.html`
- Fetch via the `DesignSync` MCP tool (`get_file` method with this project ID). Its
  design-system bundle lives at `_ds/classical-515ce88c-6cd9-4521-aae8-7028a1a18cb3/` — its
  `styles.css` is the token/component source of truth, already ported verbatim into this
  repo's own `styles.css`.
- When asked to "sync the design" again: use this project ID directly (don't make the user
  re-paste the URL unless they give a different one), and **always re-fetch fresh and diff
  against the current index.html** — every sync so far has been a real, substantive change,
  never just cosmetic noise.

## 7. Task status (as of this file's writing)

| # | Task | Status |
|---|---|---|
| 1 | Add localStorage persistence to survive refresh | ✅ Done |
| 2 | Build client-side Export-to-Excel function | ✅ Done |
| 3 | Replace mock SUPPLIER_MASTER with real Item Master data | ✅ Done |
| 4 | Get a real OCR output Excel sample and match RAW_ROWS structure | ⏳ Waiting on IT |
| 5 | Clarify SharePoint API auth mechanism with IT | ⏳ Waiting on IT |
| 6 | Confirm monthly batch / file-naming convention with team | ⏳ Waiting on IT |
| 7 | Build real SharePoint read/write API integration | 🚫 Blocked by #5, #6 |

Tasks #4/#5/#6 can all be asked to IT in one conversation — same people need to answer all
three. Task #7 can't start until #5 and #6 have answers. In the meantime, the local-file-bridge
idea in §5 is a way to make progress without waiting on IT at all.

## 8. Environment notes

- Not a git repository (as of this writing).
- To preview: use `.claude/launch.json`'s `static-server` config (`python3 -m http.server
  8000`), then open `http://localhost:8000/index.html`. Opening `index.html` directly via
  `file://` should also work in a real desktop browser now that all JS libraries are vendored
  locally (only relative `<script src>` tags, no CORS-sensitive fetch calls) — this differs
  from some sandboxed testing tools that block `file://` script execution for files outside a
  designated project folder.
- Claude's own **memory** (accumulated project context, this same information in finer detail)
  lives outside this folder, tied to its exact path on the original machine, and does **not**
  travel automatically if this folder is copied elsewhere — that's the whole reason this file
  exists.
