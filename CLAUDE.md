# PO–Invoice–DO Mapping (RVN 1020) — Item Mapping Validation

> Read this before doing anything else in this project. It's a portable brief — if this folder
> gets copied to another computer, this file travels with it and should give a fresh Claude
> session everything it needs without the user re-explaining from scratch.

**On-screen page title/heading is now "Invoice data Validation"**, not "Item Mapping Validation"
— the `<title>` tag and the visible `<h1>` in `index.html` were changed at the user's request.
"Item Mapping Validation" is kept everywhere else in this file (and in the Claude Design project
name, §6) as this screen's internal/project name — only the text a user actually sees on the
page changed, not the screen's identity in documentation. Don't "fix" this apparent inconsistency
by renaming one to match the other without asking; it's intentional.

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

**Confirmed granularity**: 1 row in this screen = 1 line item, matched between exactly one PO
and one Invoice. An invoice with N line items produces N rows (all sharing the same Invoice
No., and usually the same PO No.) — the app never groups/collapses rows by document. IT's
extraction pipeline is expected to hand over data already flattened to this row shape (see §5's
column spec) rather than one-row-per-document.

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
flag that it's expanding scope beyond what's confirmed. (This came up directly: the user asked
about giving Procurement a real URL to type in. Answer given — that requires real hosting +
an access-control decision, both IT's call, not the intern's to build. A temporary same-LAN
`python3 -m http.server` demo was offered as a stopgap, not a solution.)

## 3. Confirmed UX/design decisions — don't re-litigate these

These came from real back-and-forth with the user/reviewers. Several are deliberate reversals
of a first instinct — don't re-suggest the rejected option without flagging that it was already
considered.

- **2-color confidence flag (green/red), not 3-color.** User action is binary (review or
  don't) — a middle "yellow" wouldn't change what the user does. Exact Match always hits 100%
  and lands in green.
- **Fuzzy "pass" now requires a 3-tier rule, not just confidence ≥ threshold** (added after a
  follow-up meeting with IT). A row is green/pass only if `matchType === "Exact"` **or**
  (`Fuzzy` AND `confidence ≥ 92` AND a new `Character Error Rate` field `≤ 7.5`). Anything else
  — confidence ≥92 but CER between 7.5–9, confidence <92, or CER missing — is red/"ต้องตรวจ".
  Still the same binary 2-color display as the bullet above; only the *computation* behind
  "pass" got stricter. Implemented as one shared `isRowPassing(matchType, confidence,
  characterErrorRate)` helper used everywhere a pass/fail decision is made (auto-fill,
  bulk-confirm, the badge) — do not reintroduce a second ad hoc threshold check anywhere.
  `characterErrorRate` is parsed from a new required import column (placeholder header text
  `"Character Error Rate"`, exact name still pending IT confirmation) but is **never rendered
  anywhere on screen and never included in the exported file** — IT's own input file already
  carries confidence/CER, so echoing it back into the export would just duplicate it
  (confirmed with the user, not an oversight).
- **No sticky/frozen columns**, anywhere. This was tried (freezing Invoice No./PO No./Supplier
  Name during horizontal scroll) and abandoned after repeated rendering bugs (transparent
  backgrounds let other columns bleed through during scroll, survived multiple fix attempts).
  **Do not suggest re-adding column freeze** — it's a closed decision, not an oversight.
- **No standalone "Overview" dashboard page anymore.** An earlier version had one (4 stat cards
  + confidence bars + a "go to table" button) — it was later removed entirely in a design
  re-sync. Now it's a single table view with a compact inline stats bar (ทั้งหมด / ยืนยันแล้ว /
  Exact-Fuzzy counts) plus loading-skeleton / error / empty states depending on data state.
  Don't reintroduce the old two-view navigation unless explicitly asked.
- **PO data removed from the web app entirely** (the user explicitly asked for this — "ไม่ต้อง
  แสดงข้อมูล PO แล้ว"). No PO No. column in the main table, no PO side in the expandable
  section, no PO↔Invoice mismatch check/badge — the expandable row is now just "Invoice
  Reference" showing Qty/Unit Price/ex.VAT/UOM as plain editable values, not a PO-vs-Invoice
  comparison. This is a UI-only removal — `poNo`/`poQty`/etc. still exist in the row data shape
  and export (see §5's spec table for what's actually asked of IT now, which also dropped PO
  from the import side). **Do not reintroduce the PO vs. Invoice comparison grid or the
  mismatch badge** — this was a deliberate, explicit request, not an oversight.
- **4 editable Invoice fields** (Qty/Unit Price/Amount ex.VAT/UOM — down from "8 PO+Invoice
  fields" before the PO removal above; the 3rd field's on-screen label is `"Amount ex.VAT"`, not
  the shorter `"ex.VAT"` an earlier version used — matches the user's preferred wording):
  click-to-edit (not permanently-open inputs), Enter/blur to save, numeric fields auto-format
  with commas on blur, UOM stays free text. Original and edited values are tracked separately
  (audit trail — an accent-colored dot marks an edited field; hovering an un-edited field shows a
  pencil icon).
- **The 6 optional Invoice-only field labels dropped their `"Invoice "` prefix on screen** —
  shown as `"Discount by Line"`, `"Discount at end of bill"`, `"Delivery fee"`, `"Weight"`,
  `"Deposit"`, `"Rounding"` in the "ข้อมูลเพิ่มเติมจาก Invoice" section. This is purely the 3rd
  element (display label) of each `OPTIONAL_IMPORT_COLUMNS` entry — the 1st element (the actual
  import header / export column name, e.g. `"Invoice Discount by Line"`) is unchanged and still
  matches IT's real file exactly (§5); don't let a future edit conflate the two.
- **Supplier/Item selection is dropdown-only, never free text** — must resolve to real
  Supplier Master entries (typos would corrupt master data), unlike the 8 editable fields
  above which have no master list to select from. The picker list and the confirmed-item
  button both display the **supplier's own item name** — the reviewer compares this against the
  OCR text on the invoice, so it must read in the same "vocabulary" the invoice uses.
  **RVN item code/name/UOM are gone from this app entirely now — not just hidden, removed from
  `SUPPLIER_MASTER`'s data shape itself.** This went through several stages: first a visible
  "RVN: code · name" line under the item button (removed at the user's request as noise
  irrelevant to what this screen validates); then RVN Item Code/Name stayed as *exported* Excel
  columns only (reasoning at the time: IT-facing, needed for downstream Oracle linking); the user
  then asked for those export columns gone too; **finally the user clarified this screen is
  scoped to supplier-item validation only and has nothing to do with RVN item codes at all** —
  at that point `rvnItemCode`/`rvnItemName`/`rvnUom` were confirmed completely unused anywhere in
  the code (verified via full-file grep before removing) and deleted from every entry in
  `SUPPLIER_MASTER` (each item object now has only `supplierItemCode`/`supplierItemName`/
  `supplierUom`). **Do not reintroduce RVN fields to `SUPPLIER_MASTER` or the export** — if a
  later pipeline stage needs an RVN/Oracle code mapping, that's a different screen's job, not
  this one's. The **item picker list itself**
  went through the same trim: first showed "RVN: code · name" as a sub-line, then briefly the
  supplier's own item code as sub-line, and finally settled on **item name only, no sub-line at
  all** — each row in the picker is just the plain `supplierItemName` text, nothing else.
- **Both Supplier and Item pickers show the current selection** — a "ตอนนี้เลือก : X" line right
  under the dialog title (above the search box), plus a checkmark + tinted background on the
  matching row in the list itself. Added after the user found it hard to verify what was already
  picked when reopening a picker on an already-resolved row (especially auto-filled ones). The
  `dialogOptions` builder tags each option with `selected: true/false` (name match for supplier,
  `supplierItemName` + `supplierItemCode` match for item) and a `dialogCurrentLabel` variable
  feeds the top-of-dialog line — both computed in `App()` right where `dialogOptions` is built,
  not inside the dialog JSX itself.
- **Two-tier confirmation state**: (1) does the row have an item selected at all (auto or
  manual), (2) did the *user* actively pick it. Rows above the confidence threshold auto-fill
  from Supplier Master and show "ระบบเลือกให้" (system-selected) until the user actually
  interacts; rows below threshold start empty, then show "ยืนยันแล้ว" (user-confirmed) once
  picked.
- **Bulk-confirm only applies to rows at/above threshold.** Below-threshold (red) rows can
  only be confirmed one at a time — a user can't accidentally bulk-skip a risky row.
- **Export flow — gating reversed after a follow-up meeting with IT.** A primary
  "เสร็จสิ้นและส่งออกข้อมูล" button in the header is now disabled **only when there's no data
  loaded at all** (`stats.total === 0`) — it is **no longer blocked** by unconfirmed/needs-review
  rows (this reverses an earlier "confirmed" decision that required every low-confidence row to
  be confirmed first; the reversal itself is the confirmed decision now, don't revert back).
  If rows are still unconfirmed, the confirm dialog shows a **non-blocking** warning ("ยังมี N
  แถวที่ยังไม่ได้ตรวจ ต้องการส่งออกต่อหรือไม่?") but the export button in that dialog stays
  enabled regardless. Confirming writes the file (see the file-handle bullet below) and clears
  the persisted `localStorage` state, plus shows a success toast. (An earlier version of this
  bullet talked about export not depending on PO/Invoice mismatches — moot now, since both the
  mismatch check and the export gating itself were removed; see the PO-removal bullet above.)
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
- **A Supplier Item Code, if OCR captured one, always means Exact match at 100% confidence —
  REVISED: only true for the built-in mock dataset now, not real imports.** Originally
  implemented as a single normalizer (`applyItemCodeExactRule`) applied to both the mock rows and
  every imported row. IT's real production file (`inv_for_validation.xlsx`, §5) proved this wrong
  for real data: it has rows with a real Supplier Item Code that IT's own pipeline still marked
  `Fuzzy`/`Need Review` (a code doesn't guarantee the OCR'd description/qty are trustworthy).
  `applyItemCodeExactRule` is now called only when building `RAW_ROWS` (mock data); real imports
  keep IT's own `Match Type`/`need_review` verbatim. **Do not re-apply it to imported rows.**
- **Item-code-first matching for auto-fill/lookup.** `findMasterItem()` tries an exact
  `supplierItemCode` match first (when OCR captured a code), then an exact `supplierItemName`
  text match, then (added after real-data testing, §5) a whitespace-stripped/lowercased
  `supplierItemName` comparison as a last resort — real `Matched Item Name` values from IT
  sometimes have spacing drift from Supplier Master's spelling even on rows IT marked as not
  needing review.
  - **Known fragility (still true for the no-code fallback path)**: name-based matching is a
    literal `===` string comparison against `Supplier Master`'s `supplierItemName`. Any
    whitespace/spelling drift between what a matching pipeline outputs and the master file's
    exact text silently breaks auto-fill — confirmed multiple times while building test data
    (e.g. "ไฮโดรลิก" vs the master's real "ไฮดรอลิก"). The user-facing implication communicated
    to the user: "Matched Item Name" values from IT's pipeline must be byte-identical to
    `Supplier Master`'s item names, *unless* the row also carries a Supplier Item Code, in
    which case the code match rescues it.
- **Per-supplier "extra" Invoice-only fields — a fixed, named closed list now, not a fully
  dynamic scan.** Real supplier invoices vary a lot beyond Qty/Unit Price/ex.VAT/UOM (Discount,
  Deposit, Freight, Rounding, Weight, ...; confirmed via a real 29-supplier config file + 15
  sample per-line dataframes the user provided — see §4). An **earlier version** of this app
  made *any* unrecognized column automatically become a per-row field (fully dynamic, scanning
  for unknown headers). **That was replaced, after a follow-up meeting with IT confirmed the
  real set is exactly 6 named fields** — `OPTIONAL_IMPORT_COLUMNS` in `index.html`: Invoice
  Discount by Line, Invoice Discount at end of bill, Invoice Delivery fee, Invoice Weight,
  Invoice Deposit, Invoice Rounding. Confirmed behavior, still true for these 6:
  - Each is optional as a *column* (may be absent from the file entirely) and optional as a
    *cell* (blank per row is fine) — shown only on rows where that supplier's file actually has
    a value for it.
  - Rendered in the "ข้อมูลเพิ่มเติมจาก Invoice" section below the main Invoice Reference grid;
    **editable**, same click-to-edit + audit-trail-dot pattern as the 4 core Invoice fields.
  - No PO counterpart (PO has none of these fields, ever), so no mismatch-warning check applies.
  - Round-tripped into the exported Excel as their own fixed 6 columns (always the same order,
    blank where a row/file didn't have that field) — no more dynamic union-of-labels logic.
  - **Anything in the file that ISN'T one of these 6, one of the required core columns, `document_id`,
    or the Character Error Rate column is now silently ignored** — the opposite of the old
    "unknown column auto-becomes a field" behavior. If IT adds a genuinely new field type in the
    future, it needs to be added to `OPTIONAL_IMPORT_COLUMNS` explicitly, not picked up
    automatically.
  - **Resolved (previously an open question): the "freight as a separate pseudo-row with no
    item code" edge case** (seen in the "ปูนซีเมนต์เอเชีย" sample — a "ค่าขนส่งถึงลูกค้า" row
    appended after the real item rows, no product/item code at all). **Decision unchanged: this
    is IT's normalization problem, not the app's** — IT's Excel output (§5) must fold any such
    charge into the appropriate real item row before it reaches this screen; the app has no
    special handling for item-less rows and isn't getting any.
- **`document_id` — imported but never shown, round-tripped straight into the export.** IT
  wants a way to link back to source documents. Parsed from a placeholder header
  (`"document_id"`, exact name still pending IT) into `row.documentId`; optional on import
  (blank-safe, not a required column since early test files may not have it); never rendered in
  any table/expandable-section JSX; exported as its own "Document ID" column.
- **Export timestamp — app-generated, not from IT's file.** Every export stamps a fresh
  "Exported At" column (Bangkok local time, `YYYY-MM-DD HH:mm`, computed via
  `formatBangkokTimestamp()` using `Intl.DateTimeFormat` with an explicit `Asia/Bangkok`
  timezone rather than trusting the browser's system clock/locale) — reflects the moment of
  that export, not the import time, and is recomputed on every re-export in the same session.
- **Default row state on first load — must be empty, not mock data.** `useState(() =>
  loadPersistedRows() || [])` — an earlier version defaulted the fallback to `RAW_ROWS` (the
  hardcoded mock dataset) whenever `localStorage` had nothing yet, which was fine during dev but
  meant a genuinely fresh deploy showed 90+ fake rows instead of the "ไม่มีรายการที่ต้องตรวจสอบใน
  ขณะนี้" empty state. Caught by the user testing a real Codespace/local deploy. **Do not
  reintroduce the `RAW_ROWS` fallback** — `RAW_ROWS`/`BASE_ROWS`/`EXTRA_ROWS` still exist in the
  file for reference/dev-console use, they're just no longer wired into the initial state.
- **File System Access API is disabled on `file://` origins — `supportsFileSystemAccess()`
  gates every use of `showOpenFilePicker`/`showSaveFilePicker`.** `location.protocol !== "file:"
  && !!window.showOpenFilePicker && !!window.showSaveFilePicker`. Root cause: the user opened
  `index.html` by double-clicking it locally on Windows/Edge (`file://...`) — `showSaveFilePicker`
  reports as available there (Chromium doesn't gate this API on `file:`), but a real test on that
  exact setup produced an export Excel couldn't open ("file format or extension is not valid")
  with no JS error thrown; likely an OS/AV-level interaction with the API's atomic swap-file
  rename on `close()`. The classic `<input type="file">` import + `XLSX.writeFile()` download
  export fallback was verified reliable in every test run (including simulated `file://` via
  Playwright) and is now what file:// always uses — the native single-handle round-trip flow
  (§5) is Codespace/http(s)-only in practice now, even though the code doesn't literally check
  for Chromium.

## 4. Data: what's real vs. mock right now

- **`FALLBACK_SUPPLIER_MASTER`** in `index.html` (renamed from `SUPPLIER_MASTER` — see the live
  loading system below) is **real data**, built from **"Supplier item master update 1.xlsx"** —
  28 real suppliers, 102 items. The source file also had linked RVN item code/name/UOM per row,
  but **the RVN fields were deliberately stripped out of `index.html`'s copy** (§3) — this app
  only keeps `supplierItemCode`/`supplierItemName`/`supplierUom`, since the user confirmed this
  screen validates supplier-item identity only and has nothing to do with RVN codes. This
  supersedes an earlier, incomplete file ("master Item supplier1.xlsx") that had the supplier
  side only, with no RVN linkage — that history is moot now that RVN isn't kept here at all.
- **Supplier Master is now live-loaded, not permanently hardcoded** — the user pointed out the
  real Supplier Master Excel gets updated by IT/procurement over time (new items added), so a
  value baked into `index.html` at dev time would silently go stale. Loads into `App()`'s
  `supplierMaster` React state (everything that used to read the `SUPPLIER_MASTER` constant —
  `findMasterItem()`, the Supplier/Item picker dialogs — now reads this state instead;
  `findMasterItem()`'s signature gained a `master` first parameter for this) via **auto-fetch on
  every page load only**: `fetch(SUPPLIER_MASTER_FILENAME)` (constant =
  `"supplier_item_master.xlsx"` — renamed from `"supplier_master.xlsx"` at the user's request once
  the real file arrived, to match IT's own filename) looks for that exact filename next to
  `index.html`. If found and parseable, it silently
  replaces the built-in fallback. **Only works over http(s)** (Codespace, GitHub Pages,
  Cloudflare Pages) — `fetch()` of a local file is blocked under a plain `file://` open, so this
  silently no-ops there (falls back to `FALLBACK_SUPPLIER_MASTER`, no error shown — this is
  expected, not a bug).
  **A manual "นำเข้า Supplier Master" `<input type="file">` button was built as a file://
  fallback, then explicitly rejected by the user right after ("คิดว่าไม่ควรมีนำเข้า excel
  supplier master แล้ว") and removed** — `handleSupplierMasterFile()`, its ref, its button/input
  JSX, and the `"import"` value of `supplierMasterSource` are all gone. **Do not re-add a manual
  Supplier Master import button** — auto-fetch is the only loading path now, besides the
  hardcoded fallback.
  A small status line under the page title ("Supplier Master: โหลดจากไฟล์ล่าสุดในโฟลเดอร์" /
  "ข้อมูลเริ่มต้นในแอป (อาจไม่ใช่ข้อมูลล่าสุด)") shows which of the two remaining sources is
  currently active, driven by `supplierMasterSource` state (values: `"file"` / `"built-in"`).
  **Parsing gotcha found during testing, don't reintroduce**: the real Excel's `"supplier name"`
  column is a short internal mnemonic code (`"JWT"`, `"KTMT"`, ...), **not** the full Thai company
  name — using it as the parsed key produced 0 auto-resolved rows (nothing matched). The real
  full name — the one that must match an Invoice file's own `"Supplier Name"` column — was in the
  **`"Oracle Supplier Name"`** column instead in that old source file.
  **Column spec revised again — a clean 4-column ask, handed to IT directly, not derived from an
  existing Excel this time.** The user showed a screenshot of how Oracle's own team stores the
  RVN↔supplier item mapping (a DFF export: header row 8, pipe-delimited `"code|name"` combined
  fields, `FG Item`/RVN-style internal codes) — evaluated and **rejected** as the format to build
  a parser around, both because it reintroduces RVN-adjacent data (§3 — explicitly out of scope)
  and because combined delimited fields are fragile to parse. Instead, agreed a clean ask to
  relay to IT: **`Supplier Name`, `Supplier Item Code`, `Supplier Item Name`, `Supplier Item
  UOM`** — one row per supplier-item pair, header on row 1, no RVN/FG Item column needed.
  `Supplier Item UOM` should exist as a column even before Oracle actually tracks UOM data (per
  the user: "IT should create the column now, values can stay blank until later") — the parser
  already handles blank UOM gracefully (many `FALLBACK_SUPPLIER_MASTER` entries already have
  `supplierUom: ""`), so no code change was needed for that specific ask, only for the header
  names below.
  `parseSupplierMasterRows()` now reads each field with **new-name-first, old-name-fallback**
  (`raw["Supplier Name"] ?? raw["Oracle Supplier Name"]`, and similarly for Item Code/Name/UOM) —
  built at the time so an old-format file (copied from "Supplier item master update 1.xlsx")
  would keep working via the fallback names until IT delivered a real file; verified both the
  old-format file (546-row invoice test: 514 auto-resolved) and a hand-built new-format file
  (with an intentionally blank UOM) parse correctly. **The real file has since arrived and is
  deployed (see below)** — the old fallback header names could now be deleted from
  `parseSupplierMasterRows()`, but were deliberately left in place regardless, not urgent to
  remove; don't be surprised to find both present, that's intentional dual-support, not leftover
  cruft.
  **Deployment note**: whoever deploys this app for real needs to place an actual
  `supplier_item_master.xlsx` (exact filename, matching `SUPPLIER_MASTER_FILENAME`) next to
  `index.html` for auto-fetch to find anything — this is a manual step outside the app itself,
  not automated by any build process (there is no build process, §1). A real copy already exists
  in this project folder (see below).
  **The real file from IT has arrived and is now the one deployed** (`supplier_item_master.xlsx`,
  "Export Worksheet" sheet, 100 rows / 27 suppliers, generated from an Oracle SQL query against
  `fnd_lookup_values` — the `SQL` sheet in the file itself documents the exact query). Only 3 of
  the 4 requested columns are present — **`Supplier Item UOM` is missing entirely** (not even as
  a blank column; Oracle genuinely doesn't track it yet, matching what the user warned) — the
  parser already handles a fully-absent UOM column gracefully, no fix needed there.
  **Real gotcha caught and fixed**: the actual header is `"Supplier Item Code "` — **with a
  trailing space** — traced to the source SQL itself (`... as "Supplier Item Code "`, space
  included in the query). An exact-string header lookup would have silently returned `""` for
  every single row's item code, forever, with no error — caught by inspecting the real file
  before wiring it up (established practice in this project after the "JWT" mnemonic-code
  incident). Fixed generally, not with a one-off key: `parseSupplierMasterRows()` now runs
  `trimmedKeys()` on every row first, stripping whitespace from every column name before any
  field lookup — covers this file and any future whitespace surprises the same way. Verified
  against the real 546-row invoice file: 515 auto-resolved (**one better** than the old hardcoded
  `FALLBACK_SUPPLIER_MASTER`'s 514 — this real data is slightly more complete). The dev copy of
  `supplier_item_master.xlsx` in this project folder is now this real file, not the old "Supplier
  item master update 1.xlsx" — but the old-header fallback names in the parser were deliberately
  left in place regardless (§ above), since this file still only has 3 of 4 columns and a future
  delivery's exact header text isn't guaranteed identical.
- The broader RVN Item Master reference (1,361 items, English descriptions, Product
  Type/Category hierarchy) lives in **"Copy of RVN Inventory Org Structure Updated_Y2026
  (3).xlsb"**, sheet **"Final"** — was never wired into the app, and per the RVN-removal decision
  above should stay that way; kept as a source file for reference only, not for reuse here.
- **`BASE_ROWS`/`EXTRA_ROWS`** (the transactional PO/Invoice/OCR rows, 114 total) are still
  **mock/synthetic transactions**, used only when no real file has been imported (§3's empty-
  default fix means they're never shown automatically anymore, just available for dev/console
  use). A real production file (`inv_for_validation.xlsx`, 546 rows) has since been received and
  is what the app is actually tested against now (§5) — the mock rows' structure is deliberately
  *not* kept in sync with the real column spec any more, don't assume they match.
- **Confirmed assumption: `Supplier Name` will always match an existing `SUPPLIER_MASTER`
  key.** There is no real-world case of OCR producing a supplier name absent from the system —
  suppliers are known/registered ahead of time; confirmed true across all 15 suppliers in the
  real file (§5). Don't design defensively for "supplier not found" (e.g. don't add fallback UI
  for it). If a supplier or item lookup ever comes up empty, that's a **data quality issue in the
  source Supplier Master file itself** (e.g. an unrelated vehicle-purchase entry under "บริษัท
  โตโยต้า ภูเก็ต มอเตอร์ส จำกัด" had a garbage `"???"` placeholder in the source spreadsheet's RVN
  column, back when this app still carried RVN fields — moot now that RVN is stripped out
  entirely, §3, but the underlying point stands: source-file quality issues aren't this app's
  job to paper over), not a gap the app needs to handle gracefully.
  - Practical trap hit repeatedly while building test/mock data: guessing a supplier's full
    registered name instead of using the real key in `SUPPLIER_MASTER` (e.g. inventing "บริษัท
    ปัตตานีคอนกรีต จำกัด" when the real entry is "บริษัท ปัตตานีโลจิสติกส์(2009) จำกัด") produces
    exactly the same symptom as a genuine "not found" case, but it's test-data error, not a
    product bug. Always grep `SUPPLIER_MASTER` in `index.html` for the real name/spelling
    before writing a supplier name into a mock/sample file.
- **A supplier-level config file** ("PO_INV_DO_Supplier_Invoice_Reconciliation_Config_1.xlsx",
  provided by the user, not saved into this repo) documents 29 real suppliers across 5
  dimensions — VAT Method, Discount Method, Deposit Method, Freight Method, Rounding Method
  (values: None / Line Inclusive / End-of-Bill / Per-Line / Embedded in Net Amount / Special) —
  plus 15 real per-line invoice-dataframe screenshots (also user-provided, not saved into this
  repo). Key takeaways already folded into §3's "extra fields" decision and into the Excel spec
  in §5; the raw files themselves live only in chat history / the session that reviewed them,
  not on disk here — if this comes up again and the specifics matter, ask the user to re-share.

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

**Session/state rule (revised)**: each new Excel pull = a new batch/session. Working state must
be cleared **only after a successful export** — refresh must NOT lose in-progress review state.
**Revised from an earlier version of this rule**: persistence used to be in `localStorage`
(survives even a full browser/tab close), but the user found this confusing during real testing
— reopening the app later (a new session, not a refresh) showed a stale batch from days earlier,
easy to mistake for current/live data. Switched to **`sessionStorage`**: still survives F5/
refresh within the same tab (the original "don't lose in-progress work" goal), but is empty again
the moment the tab/browser is actually closed and reopened — verified both cases end-to-end with
Playwright (same-context reload keeps rows; a fresh browser context starts on the empty state).
`clearPersistedRows()`/`persistRows()`/`loadPersistedRows()` all read/write `sessionStorage` now,
same `STORAGE_KEY`/`SCHEMA_VERSION` guard as before. **Do not revert to `localStorage`** — that
was the confusing behavior that prompted this change.

**Batch model — confirmed as the user's intended approach, still needs IT sign-off**: one input
Excel + one output Excel per month, each month's batch independent (no document overlap across
months). The user has settled on this as what they want to propose; it is **not yet confirmed
with IT**. If IT confirms months never mix, the incremental-vs-full-snapshot dedup question
below mostly resolves itself. Remaining edge case to flag to IT regardless: a document from
month A that failed OCR and gets reprocessed into month B's batch would still need a dedup key
(Invoice No. + PO No.).

**Known limitation — no multi-user concurrency, by design (confirmed with user, not yet a
problem but flagged for later)**: this is a single-user, client-side-only app — there is no
backend, so two people working on the same batch at the same time **do not see each other's
edits at all**. Each person's Import creates an independent in-memory copy in their own
browser; if two people Import the same input file and both edit/confirm rows, each produces
its own separate, possibly-conflicting output file on Export, with no merge or warning. This
is a direct consequence of the "no backend/database" scope boundary (§2), not a bug. Mitigation
is **process-level, not code**: one batch should have exactly one owner at a time (the monthly
batch model above already tends to enforce this naturally). If the team later needs true
concurrent multi-user editing on the same batch, that's a scope change requiring a real backend
— flag it back to the user/IT rather than trying to solve it client-side.

**Local-file-bridge import — BUILT, briefly upgraded to a unified Import+Export file handle,
then that unification was REVERSED at the user's explicit request.** The original SharePoint
workflow reasoning (IT keeps one "Output" folder holding a single round-tripped file, re-uploaded
under the same filename each time) motivated a period where Import and Export shared one
`fileHandleRef`, and Export silently overwrote whatever file was last picked/imported, with a
"เปลี่ยนไฟล์ปลายทาง" link to reset it. **That link turned out confusing in practice** — mutating a
`ref` doesn't trigger a React re-render, so clicking it visibly did nothing (a real bug, fixed
once — see the `isRowPassing`-adjacent bug-fix entry above) — and shortly after fixing that, the
user decided they'd rather **always be prompted for a destination on every single export**, no
memory at all. Current behavior:
- **`writeExportFile(displayRows)`** (no longer takes a `fileHandleRef` param) always calls
  `window.showSaveFilePicker()` fresh on Chrome/Edge — every export shows the native save dialog,
  every time. No handle is ever kept between exports. Firefox/Safari (no File System Access API)
  and any `file://` open (§3) fall back to a classic download, unchanged. The export confirm
  dialog no longer has any explanatory text for the Chrome/Edge case either (removed at the
  user's request, "ไม่ต้องมีคำพูด") — only the Firefox/Safari/`file://` fallback still shows its
  "เบราว์เซอร์นี้จะดาวน์โหลดเป็นไฟล์ใหม่ทุกครั้ง" line, since that one explains genuinely
  different behavior worth flagging; the Chrome/Edge case is self-evident once the native picker
  appears, no caption needed.
- **"นำเข้าไฟล์ Excel" (Import)**, in Chrome/Edge, still uses `window.showOpenFilePicker()` rather
  than a plain `<input type="file">` — but the picked handle is now used only to read the file
  (`handle.getFile()`) and is **not stored anywhere afterward**; it has no bearing on Export.
- **Do not reintroduce a shared/remembered file handle between Import and Export, or any
  "reuse last destination" behavior for Export** — this was tried, then explicitly reversed. If a
  future request asks for "remember my last export location" again, treat it as a genuinely new
  ask, not a revert to old code (the old `hasFileHandle`/`fileHandleRef`-in-`App()` machinery was
  fully deleted, not just hidden).
- **Columns are matched by header name**, not position — required columns must match exactly by
  name; see the column list below for exactly which. `findMissingImportColumns` /
  `parseImportedRows` in `index.html` are the source of truth.
- Validates before touching state: missing required columns → error banner listing exactly
  which ones, existing data untouched. Empty file (headers only, no data rows) → separate "no
  data" message. Unparseable/corrupt file → caught, existing data untouched either way.
- Importing while there's unconfirmed/edited in-progress work triggers a native confirm dialog
  first (each import fully replaces the row set — it's a new batch).
- The user's SharePoint access is via **"Add shortcut to OneDrive"** on the target document
  library ("IT Solution Team" → "Data Project PO-Invoice-DO Mapping") rather than the full
  OneDrive desktop sync client — confirmed this is a live link to the real SharePoint file (not
  a copy): reading it is always safe (Import never writes back on its own), and — separately —
  dropping/overwriting a file in that same shortcut folder (whether via the app's unified
  handle or manually) **does** sync it to the real shared SharePoint library automatically, no
  extra step needed, visible to the rest of the team immediately.
- **No API key/OAuth of any kind is needed for this bridge approach.** The web app never talks
  to SharePoint or Microsoft Graph directly — it only reads/writes local files, and separately,
  the OneDrive desktop client (or the browser's own file access, in the shortcut-only setup
  above) handles the real SharePoint sync in the background, completely decoupled from the app.
  An API key only becomes relevant for the "real" integration in task #5/#7 below, where the app
  itself would call the SharePoint/Graph REST API directly.
- **Known limitation, unchanged**: no true multi-user concurrency — see the dedicated bullet
  above. The single-round-tripped-file model described here actually *helps* enforce "one owner
  at a time" as a side effect, but the app still can't detect or merge two people editing
  independently if the process discipline breaks down.
- **Historical bug (now moot — the whole mechanism it applied to was later removed, see the
  "REVERSED at the user's request" bullet above): "เปลี่ยนไฟล์ปลายทาง" (change destination file)
  link in the export confirm dialog did nothing visible when clicked.** Root cause: its `onClick`
  only mutated `fileHandleRef.current = null` — mutating a `ref` does **not** trigger a React
  re-render, so the dialog's text stayed on "จะอัปเดตไฟล์เดิมที่เลือกไว้..." even though the click
  "worked" internally. Fixed at the time with a parallel `hasFileHandle` state; **both the ref and
  the state, and the link itself, were deleted entirely soon after** when Export was changed to
  always prompt fresh. Kept here only for the general lesson, which still applies to any future
  ref-driven UI in this codebase: **never gate JSX rendering on a ref's `.current` value directly
  — it won't update the screen.**

**Excel column spec — CONFIRMED against IT's real production file, not a placeholder anymore.**
The user supplied a real file, `inv_for_validation.xlsx` (546 real rows, 20 columns), generated
by IT's actual matching/AI pipeline (Path A: IT computes Match Type/Confidence/Matched Item
Name/need_review — this screen does not compute matching itself, it's a review UI over IT's
output). All header names below are **verbatim from that file**, not the earlier "(OCR)"-suffixed
chat-shorthand names this app used to expect — that suffix was **never real**, and headers have
been updated in `index.html` to match exactly. PO columns are still fully dropped (no PO data
anywhere in IT's real file either, confirming the earlier removal was correct):

| # | Column (exact header) | Required? | Notes |
|---|---|---|---|
| 1 | `Invoice No` | ✅ | blank in ~5% of real rows (25/546) — allowed, just displays blank |
| 2 | `Supplier Name` | ✅ | must match a real `SUPPLIER_MASTER` key exactly (§4) — all 15 suppliers in the real file are covered |
| 3 | `Supplier Item Code` | ✅ | blank allowed per row (53/546 blank in real data) |
| 4 | `Item Description` | ✅ | |
| 5-8 | `Invoice Qty` / `Invoice Unit Price` / `Invoice Amount ex.VAT` / `Invoice UOM` | ✅ | |
| 9 | `Match Type` | ✅ | real values are `"Exact Match"` / `"Fuzzy"` / blank (~1% of rows, no candidate found at all) — parsed via `/^exact/i` so `"Exact Match"` → internal `"Exact"`, everything else (including blank) → `"Fuzzy"` badge |
| 10 | `Matched Item Name` | ✅ | ideally byte-identical to Supplier Master's `supplierItemName`, but **~24% of real distinct values have whitespace/case drift** (e.g. `"16มมx10ม SD40"` vs Master's `"16มมx10มSD40"`) — `findMasterItem()` now falls back to a whitespace-stripped/lowercased comparison after the exact-match attempt fails (§3), specifically so IT's "no review needed" rows still auto-select despite the drift |
| 11 | `fuzzing_score` | ✅ | 0–100 float (e.g. `88.888...`), rounded on import — this is what the app calls `confidence` internally and shows as "% CONFIDENCE" |
| 12 | `character_error_rate` | ✅ | confirmed real header (was a placeholder). Feeds `isRowPassing()`'s confidence/CER rule — the app's sole pass/fail source (see below). Never shown, never exported |
| 13 | `need_review` | ✅ | Parsed into `row.needReviewRaw` but **NOT used to decide pass/fail** — see below. Kept on the row (available if a future need arises) but currently write-only. Never shown, never exported |
| 14-19 | `Invoice Discount by Line` / `Invoice Discount at end of bill` / `Invoice Delivery fee` / `Invoice Weight` / `Invoice Deposit` / `Invoice Rounding` | optional | column may be absent entirely; cells may be blank per row either way — header text matched IT's real file exactly already, no change needed here |
| 20 | `doc_id` | optional (always present in the real file, kept optional for leniency) | confirmed real header (was a placeholder `document_id`) — hidden from UI, round-tripped straight to export as "Document ID" |

**Two behavior changes this real file forced, both deliberate reversals of earlier "confirmed"
rules — do not revert either:**
- **`applyItemCodeExactRule` (a Supplier Item Code always means Exact/100%) no longer applies to
  imported files** — only to the built-in mock dataset (`RAW_ROWS`) now. Real data has rows with
  a real Supplier Item Code that IT still marked `"Fuzzy"`/`"Need Review"` (e.g. `BEW80181` →
  `"ถวดดำ #18"` vs matched `"ลวดดำ #18"`, an OCR misread the old rule would have silently hidden
  from review). Forcing Exact/100 on top of IT's own real determination would be actively wrong,
  not just redundant.
- **`isRowPassing()` ignores `need_review` entirely — pass/fail is decided solely by this app's
  own confidence≥92%/CER≤7.5% rule (Exact match always passes), applied uniformly to every row.**
  This went through two revisions before landing here: v1 trusted `need_review` as authoritative
  ("anything but Need Review passes"); the user corrected that because `"Validated by Low Score"`
  doesn't actually mean pass; v2 added a per-value carve-out that still ran the confidence/CER
  rule just for that one value. The user then pointed out the app should **always** apply its own
  rule regardless of `need_review`'s value — checked against all 546 real rows and confirmed the
  confidence/CER rule alone reproduces the exact same pass/fail split IT's `need_review` implies
  (every `"Validated"`/`"Validated by High Score"` row already clears 92%/7.5% on its own; every
  `"Need Review"`/`"Validated by Low Score"` row already misses it) — so this is not a behavior
  change from v2, just a simpler, single source of truth. `needReviewRaw` is still parsed and
  kept on the row, just unused for pass/fail. **Do not reintroduce a `needReviewRaw` branch into
  `isRowPassing()`.**

**Known real-data quirks, flagged as IT/source-data issues, not app bugs** (don't "fix" these
client-side): ~7 rows (1.3%) have completely blank `Match Type`/`Matched Item Name` — one of
these is literally a stray `"Delivery Order No."` / `"วันที่"` row that looks like a mis-parsed
table header, not a real line item. The app renders these safely (blank Match Type → "Fuzzy"
badge, `need_review = "Need Review"` correctly forces review either way) but doesn't attempt to
detect or filter out garbage rows — that's IT's extraction pipeline's job.

Anything in the file that isn't one of these 20 named columns is **hidden from the UI but passed
through untouched into the export** (the passthrough-fields mechanism, §3) — not discarded. 1 row
= 1 line item matched between one PO and one Invoice (§1). No DO columns needed for this screen —
explicitly confirmed with the user, this screen checks PO↔Invoice only (even though PO itself
isn't in the import spec above anymore, the *matching* this screen validates is still
conceptually PO↔Invoice line matching, just without displaying PO data — see the earlier
PO-removal decision in §3 if this reads as contradictory at a glance).

**Export column order/names now mirror the import spec above exactly, in the same order** —
confirmed with the user this makes downstream re-processing easier for IT (the output is
recognizably "the input file with review results appended," not a differently-shaped file). This
is a deliberate reversal of the export's earlier ad hoc column set (which had its own names like
`"Invoice No."`/`"Supplier Name (OCR)"`/`"Confidence (%)"` and omitted `character_error_rate`/
`need_review` on purpose). Current `buildExportRows()` order: all 20 columns from the table above
verbatim (including `fuzzing_score`, `character_error_rate`, `need_review`, `doc_id` — no longer
excluded), **then** this app's own added columns (`Selected Supplier`, `Selected Item`,
`Confirmed By`, `Exported At`), **then** any passthrough (unrecognized) columns. **`RVN Item
Code`/`RVN Item Name` were removed from this list entirely** (§3 — RVN is now gone from the app
end to end, not just hidden) — **`Selected Item`** (the supplier's own `supplierItemName`,
`r.selectedItem.supplierItemName`) was added right after, since without an RVN column the
export had no field at all showing which item was actually picked, only which supplier — caught
immediately after the RVN removal. `"Selected Supplier"` exports blank instead of the on-screen
placeholder text `"— ยังไม่เลือก —"` when nothing's been picked (that Thai UI string was leaking
into the Excel cell before this was caught); `"Selected Item"` is blank whenever
`r.selectedItem` is `null` (unresolved rows), no placeholder-string equivalent needed there.

**Handoff note**: when the intern's internship ends and this hands off to IT, recommended
package = code (`index.html`, `styles.css`, `vendor/`) + this file (suggest renaming to
`README.md` for a non-Claude audience — content stays the same) + the column spec above. The
source Excel files (`Supplier item master update 1.xlsx` etc.) don't strictly need to go along
since their data is already baked into `SUPPLIER_MASTER` in `index.html` — flag to IT which
source file that was built from instead, so they can check whether it's stale.

**Deployment files (added to this folder, not authored by this Claude session)**: the user
separately prepared a GitHub/Cloudflare Pages deployment package and asked to merge it with this
project — `README.md` (deployment-audience version, distinct in purpose from this file),
`DEPLOYMENT.md` (step-by-step GitHub upload + Cloudflare Pages setup), `_headers` (Cloudflare
Pages security/cache headers — nosniff, frame-deny, long cache on `vendor/*`), and `.gitignore`
(blocks `.xlsx`/`.xlsb`/`.claude/`/`.env` etc. from being committed). **Only `index.html` from
that package was rejected and replaced** — it was a stale copy (`SCHEMA_VERSION = 3`, still had
PO columns in the export, no passthrough-fields system, no "ไม่ทราบค่า" placeholder); the
`index.html` actually in this folder is always this session's latest. If a future deploy package
arrives from outside this session again, **diff its `index.html` against this folder's before
using it** — same check that caught this one.

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
- **Note**: a long list of app logic/behavior changes — the Import-Excel feature (now unified
  with Export around one file handle), the fixed optional-fields list (replacing an earlier
  fully-dynamic version), the item-code-priority matching rule, the on-screen RVN-name removal,
  the PO-data removal from the UI, the 3-tier fuzzy pass/fail rule, and the export-gating
  reversal (all §3/§5) — were built directly in `index.html` and have **not** been synced back
  into the Claude Design project. If a future design re-sync happens, these are hand-written app
  logic/behavior, not design-tool output — don't let a design sync silently revert them.

## 7. Task status (as of this file's writing)

| # | Task | Status |
|---|---|---|
| 1 | Add persistence to survive refresh | ✅ Done — now `sessionStorage`, not `localStorage` (revised, §3: refresh keeps state, closing the tab/browser doesn't) |
| 2 | Build client-side Export-to-Excel function | ✅ Done |
| 3 | Replace mock SUPPLIER_MASTER with real Item Master data | ✅ Done |
| 4 | Get a real OCR output Excel sample and match RAW_ROWS/import structure | ✅ Done — real file `inv_for_validation.xlsx` received and fully wired up (§5). The earlier `demo_mapping_PO_INV_2.xlsx` sample is confirmed **not** this screen's format (different pipeline stage) — don't reuse its structure |
| 5 | Clarify SharePoint API auth mechanism with IT | ⏳ Waiting on IT — moot for the local-file-bridge approach (confirmed no API key needed there, §5), still relevant only for the eventual real integration (#7) |
| 6 | Confirm monthly batch / file-naming convention with team | ✅ User has settled on monthly batches as their own proposal (§5) — still needs IT's formal sign-off, but no longer an open design question on this app's side |
| 7 | Build real SharePoint read/write API integration | 🚫 Blocked by #5 (and increasingly less urgent — the unified-file-handle local bridge in #8 now covers the actual described real-world workflow) |
| 8 | Build local-file-bridge Import+Export (§5) | ✅ Done — unified around one File System Access API handle (Chromium/http(s) only — disabled on `file://`, §3), so Import and Export round-trip the *same* file/filename automatically on Codespace/hosted use. `<input type="file">` + classic download used everywhere else (Firefox/Safari, or any `file://` open) |
| 9 | Send the finalized column spec to IT | ✅ Done — superseded by receiving IT's real file directly; the spec in §5 is now confirmed-real, not proposed |
| 10 | Confirm exact header text for `character_error_rate` and `doc_id` columns | ✅ Done — both confirmed from the real file (`character_error_rate`, `doc_id`), no longer placeholders |

Task #5 is now largely moot for near-term work (the local bridge in #8 needs no API key at
all); it only matters if/when a "real" always-on SharePoint API integration (#7) is revisited.
Task #6 is settled on this app's side (monthly batches), just needs IT's formal agreement.

## 8. Environment notes

- Not a git repository (as of this writing).
- To preview: use `.claude/launch.json`'s `static-server` config (`python3 -m http.server
  8000`), then open `http://localhost:8000/index.html`. Opening `index.html` directly via
  `file://` should also work in a real desktop browser now that all JS libraries are vendored
  locally (only relative `<script src>` tags, no CORS-sensitive fetch calls) — this differs
  from some sandboxed testing tools that block `file://` script execution for files outside a
  designated project folder. On the user's actual Windows machine, plain `python3` wasn't
  found (Windows' python.org installer usually registers the command as `python`, not
  `python3`) — try `python -m http.server 8000` there first.
- Claude's own **memory** (accumulated project context, this same information in finer detail)
  lives outside this folder, tied to its exact path on the original machine, and does **not**
  travel automatically if this folder is copied elsewhere — that's the whole reason this file
  exists.
