# Build: legacy invoices, "mark as shipped before go-live", and lines without a PO

> **Status (2026-11-14): built** (API V018, Old invoices pages, Mark shipped in Tracking, "Add a line
> without a PO" on Review & save). Not built yet: entering an old invoice by hand without a file in the
> UI (the API route `POST /api/legacy-invoices` exists), and an "Old" tag in PO line History.

## Context
Shriya Mining Industries' invoicing app is going live. Before go-live, the last 12 months of purchase orders (POs) will be uploaded through PO Import. Many of those POs were already shipped, fully or partly, on invoices made in the old Excel process. The app currently treats every uploaded PO line as fully open, so a new invoice could bill quantities that were already shipped.

Goal: let users record what was shipped before go-live, so each PO line's remaining quantity, line status and PO status are correct, with a clear paper trail. New invoices must only use what is genuinely left.

Do NOT build an Excel importer for old invoices. Old invoices are entered by a person through the screens below. There are about 20 of them.

## Repositories and stack
- API: `C:\GIT\ShriyaMiningIndustries_api`. .NET 10 Azure Functions, Dapper, SQL Server stored procedures.
  - DbUp migrations in `database/migrations`: `V###` scripts run once, `R__platform-procedures.sql` reruns. The latest is V017; use V018.
  - The procedures also live as files in `database/ShriyaMiningIndustries.Database/app/StoredProcedures`, which build into a dacpac. Keep each procedure file and its copy in `R__platform-procedures.sql` identical.
- UI: `C:\GIT\ShriyaMiningIndustries_UI`. Vue 3, TypeScript, Vite, vitest.
- Invoice pages: `src/views/POImport.vue` is the Create invoice and invoice page, `src/views/InvoiceList.vue` is Invoices, `src/views/PoTracking.vue` is Tracking.

## What already works (reuse it; don't duplicate)
- An invoice line may carry a `poLineId`. Lines with one count as invoiced on that PO line.
- The API refuses over-invoicing.
- Procedures compute remaining quantities, line statuses (Open, WIP, Partial, Ready to Ship, In Transit, Delivered) and PO statuses from linked invoice lines.
- Lines without a `poLineId` are already allowed in the invoice document. They print and total, but count against no PO.
- Crate numbers run on across invoices.
  - `app.Invoice.CrateStart` and `CrateEnd` hold each invoice's range.
  - New invoices continue after the highest crate number used.
  - Overlapping ranges are refused.
- Invoices store a full document (`DocumentJson`), are validated by `LegacyInvoiceDocument.Validate`, and keep versions and status history.
- Admin and Manager may edit invoices at any status. A container fit check runs on save.

## Feature 1: Legacy invoices
A read-only record of an old invoice, linked to PO lines, so those quantities count as shipped.

### Behaviour
On Review & save of a NEW invoice (not when editing a saved one), add a tick box: "This is a legacy invoice (already shipped)". When ticked, show:
- **Invoice number:** required, the old invoice's own plain number, as printed (e.g. "60"). Up to 20 characters. Stored in the normal `InvoiceNumber` field.
- **Invoice date:** required, the old invoice's date.
- **Old crate range and container no.:** optional. The invoice's overall crate range (e.g. 960-984) is stored in the existing CrateStart/CrateEnd only when it is clean AND does not overlap any invoice already saved; otherwise CrateStart/CrateEnd stay empty and the crates are kept as text on the lines. Either way the existing crate code works unchanged (see "Crate numbering").

Saving a legacy invoice:
- **Status:** any invoice status, chosen when saving (suggested: In Transit for the last 45 days, Delivered for older). It can be changed any time, for one invoice or for several ticked at once on the old-invoices list; every change goes into the status history. The lines are read-only for everyone, and Admin/Manager "edit at any status" does not apply to them.
- **Delete old invoice (Admin only):** removes a wrongly saved old invoice. Its quantities go back to the PO lines, and its uploaded file goes back to the upload list as a draft to redo.
- **PO lines:**
  - Its lines must be linked to PO lines (picked on Select lines), with quantities in the PO line's own unit.
  - Over-invoicing is allowed on old invoices only, with a warning ("ships 60 SF more than PO line 172203-02 has left"); the PO line then shows Over-invoiced. Rare in practice. New invoices keep refusing it.
  - Remaining quantities and statuses update through the existing calculations.
- **Not assigned or checked:** no app-generated invoice number, no container fit check, not listed in "Needs attention" for a missing container, and no full document validation. Lines need only a description, quantity, unit and PO line. Rate and amount are optional.
- **Duplicates:** refused by the existing unique `InvoiceNumber`. The message says "Invoice 60 is already in the app." Old plain numbers can never clash with app numbers (`SMI-YYYY-MM-NNN`).

Display:
- **Invoices list:** shows the number with a "Legacy" tag, and is searchable like any invoice. A filter or status tab is not required.
- **Opening a legacy invoice:** a read-only page with the number, date, lines, linked PO lines, and the reference fields.
- **PO line History** (Tracking and Create invoice): shows "Invoice 60 (legacy) · 18 Sep 2026 · 144 Box".

### Data
- **V018 migration** adding to `app.Invoice` only `IsLegacy BIT NOT NULL DEFAULT 0`. No separate original-number/date columns and no `L-` numbers: the legacy number and date go in the existing `InvoiceNumber` and `InvoiceDate`.
- **Invoice numbering:** the monthly `SMI-YYYY-MM-NNN` sequence only reads numbers with that prefix, so plain legacy numbers don't affect it. Verify this.
- **Crate numbering: no code or table changes.** The next crate number stays "largest crate number in the database + 1" (`InvoiceRepository.GetNextCrateStartAsync`, never below 1000) and the overlap checks in `usp_Invoice_Create` / `usp_Invoice_Update` stay as they are. Old invoices only store a range that is clean and doesn't overlap (see Behaviour), so the overlap check never trips on them, and with invoices 71-74 entered (up to crate 1010) new invoices start at 1011.
- **Update the list, search, summary and PO-line history procedures** to return `IsLegacy`.

## What real legacy files look like (4 samples read 2026-11-14)
Old invoices 71–74, all dated 05 Nov 2026, from NSR POs that are live in the app: 172203, 172360, 167784, 172034, 172679, 172311, 172782, 172274 and 171035. They show:
- **Only the INV sheet is the invoice.** `Sheet3` in every file is a stale copy of a 2019 invoice ("SMI/NSR/INV /64 /2018-19"). Ignore it, and ignore PACKIN (packing list) and Sheet1.
- **No SKUs on lines.** Each line has crates ("CRT NO 934 - 937", "04 CRATES"), stone/finish/size ("CALIFORNIA GOLD · GUAGED · 12" X 24""), quantity, rate, amount, a pieces note ("840 PCS"), and "NSR PO # 172034". PO-line matching must suggest candidates by PO number + stone + size (tolerating typos), and a person confirms.
- **One old line can cover two PO lines.** Example: 750 liners = 250 PCS on PO 172203 + 500 PCS on PO 172360. It must be possible to split a line across PO lines.
- **Units vary per line** despite the SQFT heading: SQFT, PCS, SETS (patterns) and TONS (flagstone, "4 TONS" at 115). Each line keeps its own unit and is counted in the PO line's unit.
- **Non-PO lines:**
  - "SAMPLE CRATES · MIXED PO NUMBERS" (3 crates, 360) uses Feature 3.
  - A discount ("LESS DISCOUNT 360", subtotal 8,113.24 → total 7,753.24) needs a negative line so totals match.
- **The header PO list is unreliable** (copied between invoices, or missing a PO used on a line). Use the PO numbers on the lines.
- **Crate ranges have noise:** "951 - 952,", "CRT NO 405 TO 408" next to "951 - 952". Old numbering ran 934 → 1010 across invoices 71/72/74; invoice 73 used 1–22, for another consignee code ("CODE ; BD", port Los Angeles).
- **Prod starts clean:** before go-live, prod holds only real POs and these legacy invoices, with no test invoices. So nothing else pushes the crate counter or duplicates a shipment.

### Changes to Feature 1 because of this (these override the rules above)
- **Crate counter:** unchanged code; see "Crate numbering" under Data. Messy or overlapping old crate text is kept on the lines only.
- **Scope:** only old invoices dated from 1 Apr 2026 (one financial year), so old invoice numbers don't repeat. POs come from PO PDFs only; the PendingOrders spreadsheet is not used, so old invoices never double-count against an already-reduced PO.
- **Upload old invoices in the UI (main way in):** on the Invoices page, "Upload old invoices" lets you pick several .xls/.xlsx files at once (e.g. all 20).
  - Each file becomes a draft in a list ("3 of 20 checked"), which the user opens, checks and saves one by one.
  - Read only the INV sheet. Fill in the invoice number, date, buyer, container/vessel, lines (stone/finish/size, crates, crate range, quantity, unit, rate, PO number) and discount. The file's total is shown next to the lines' total, with a warning if they differ.
  - For each line, suggest the matching PO line(s); the user picks or splits. Nothing is saved until the user saves.
  - A file that can't be read is listed with the reason, and can be entered by hand.
  - .xls needs a reader (for example ExcelDataReader, MIT licence, no cost) or conversion to .xlsx.
  - Entering one by hand through Review & save (above) stays possible.
- **Keep the original file:** store the uploaded Excel with the invoice, in SQL like PO documents (see `usp_PODocument_Save`; no new storage service or cost). Add "Download original" on the legacy invoice page.
- **Discount line:** allow one negative "Less discount" line on legacy invoices (and on normal invoices, if approved).
- **Lines whose PO isn't in the app** (PO older than 12 months, not uploaded):
  - The line uses the fixed PO reference **LEGACY** and keeps the old PO number written on it as reference ("LEGACY · was NSR PO # 165210").
  - It counts against no PO and has no quantity limits.
  - Search for LEGACY to find all such lines later.
  - Sample crates and discounts use it too. Every other line whose PO is in the app must be matched to a PO line.

## Feature 2: "Mark as shipped before go-live" on a PO
A quick way to close out POs that were fully shipped before go-live.

- **Where:** on a PO in Tracking (PO view) and on the PO's heading in Create invoice, for Admin and Manager only. It opens a dialog.
- **The dialog:**
  - Lists the PO's lines that still have quantity open, all ticked by default, each showing what is still open in the PO line's unit. Users can untick any.
  - Optional old invoice number. When blank, the app uses "GL-<PO number>" (then "GL-<PO number>-2" on a repeat run), shown as "Shipped before go-live".
  - Date, required, defaulting to today.
- **Confirm:** creates ONE legacy invoice (Feature 1 rules) covering the full remaining quantity of each ticked line.
  - Fully ticked POs then show Fulfilled, and their lines show Delivered.
  - Nothing more can be invoiced against those lines.
- **Show a summary before confirming,** for example "3 lines · 432 Box · will be marked shipped".
- **Repeatable:** may be run again on lines still open later.

## Feature 3: "Add a line without a PO" on Review & save
For lines from old POs that are not in the app.

- **Button:** "Add a line without a PO" on Review & save. It opens a small form:
  - **Required:** PO number (text), category (existing goods categories), quantity, unit, rate, crate count.
  - **Optional:** SKU/item, stone, finish, size, reference.
- **The line is saved without a `poLineId`.** It prints, totals, gets crate numbers in table order, can have a crate size, and is included in the container fit check.
- **Tagging:** in the lines table it is tagged "No PO in the app · not tracked". Lines from POs are tagged with their PO line ID.
- **Editing:** the line can be edited or removed like other lines while the invoice is editable.
- **On legacy invoices** these lines use the fixed PO reference LEGACY (sample crates, discounts, and lines whose PO is not in the app).

## Rules to follow
- **Design first.** Before building, write a short design covering screens, data changes and the affected procedures, and wait for approval. Ask before deviating from the approved design.
- **Testing:**
  - API: xUnit tests for the duplicate rule, old crate ranges stored only when clean and non-overlapping (next crate 1011 after 71-74), over-invoicing allowed with a warning on old invoices only, Admin delete returning quantities, the discount line, remaining quantity after a legacy invoice, "mark as shipped" making a PO Fulfilled, and read-only legacy invoices.
  - UI: vitest tests for the new pure helpers.
  - Run `dotnet test` and `npx vitest run`.
  - Verify the migration against an empty database and an existing one.
- **UI conventions:**
  - Font sizes only through the shared tokens (`var(--font-size-…)`); a test enforces this.
  - A solid fill means "click me". Information uses tinted tags with a dot or plain text. Disabled controls are flat grey.
  - Labels are plain text. Write copy in plain words from the user's side.
- **Code style:** keep CRLF line endings in files that use them, and follow the existing code style and comments.
- **Commits and deploys:** do not commit, push or deploy until told. When told, push to `test`, wait for the pipeline, then push to `main`. Nothing goes out if tests fail.
- **Secrets:** don't read or print connection strings or other secrets.

## Done when
- A partly shipped legacy PO can be brought to its correct remaining quantity by entering its old invoices as legacy invoices.
- A fully shipped PO can be closed in one step.
- New invoices can only use what is left.
- Old invoice numbers show as written, never clash with new ones, and duplicates are refused.
- Lines from untraceable POs can be added to new invoices without affecting any PO.
- New invoices continue the crate numbers after the old ones (1011 after 1010) with no change to the crate code; the fit check ignores old invoices.

- Tests pass; screenshots of the three features are provided.

---

## Later (separate project, after go-live): archive of older years
Only the last 12 months need the features above. Older years (up to ~20 years of Excel invoices) are history: their POs are closed and nothing will be invoiced against them again.

When that is wanted, build an **archive import**, not more legacy invoices:
- Read only the header of each old Excel file: invoice number, date, buyer, PO numbers, container, crate ranges, total. Lines are optional and display-only.
- Keep the original file attached. Show the records in a searchable "Archive", separate from live invoices.
- Nothing is linked to PO lines. Nothing counts against POs, crate numbering or checks.
- Use a label-based reader: layouts shift between years.
  - Example file: `C:\GIT Repo\ShriyaMiningIndustries\tests\fixtures\po\Invoice_OriginalRequirementWithAllNeededData.xls`. It holds two invoices: "60" from 2026 on sheets INV/PACKIN, and "SMI/NSR/INV /64 /2018-19" on Sheet3, which is shifted one column and 8 rows.
  - Old .xls files need a reader such as ExcelDataReader, or conversion to .xlsx first.
- Files that can't be read are still archived as the original file, and listed for review.
- Before designing it, ask for:
  - the number of files;
  - the formats used (any PDFs or scans?);
  - about 10 samples from different years;
  - whether the history is for lookup only or also for reports.
- Confirm the storage cost before loading. The PO PDFs' storage account is expected; roughly 0.5 GB per 5,000 files.
