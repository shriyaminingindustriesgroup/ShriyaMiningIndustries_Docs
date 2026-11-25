# Go-live runbook: clean data, load POs and old invoices (Test, then Prod)

Agreed 2026-11-14. Rehearse on **Test** twice, then run once on **Prod** (see the plan below). Do
the steps in order: each one depends on the ones before it.

## The plan: two rehearsals on Test, then Prod

| Run | Steps | Goal / expected result |
|---|---|---|
| **Test, run 1** | Part B (clean) → C1–C4 (buyers, POs from April 2026) → C7 (old invoices) | Find out which POs are missing. On **Old invoices → Missing POs**, click **Copy list** and keep it |
| *Between runs* | Collect the PDFs of the POs on that list (including pre-April ones) | — |
| **Test, run 2** | Part B (clean) → C1–C4 with **all** POs (April onwards + the missing ones) → C7 | **Missing POs is empty**: every old invoice line is linked to a PO line (only samples and discounts stay LEGACY, by design) → Part D checks |
| **Prod** | Exactly as run 2: Part A3 → B → C → D | Same result as run 2; then start creating new invoices |

Notes:
- The **Missing POs** panel (Old invoices page) lists every PO number the old invoices mention that
  isn't in the app, with the invoices that mention it. Sample crates and "LESS DISCOUNT" lines have
  no PO number and never appear there.
- A PO uploaded **after** an old invoice was saved stays on the list under "Uploaded since, but
  already saved as LEGACY". That invoice isn't tracked against the PO. Either an Admin deletes the
  old invoice and you check it again (keeps the real invoice number), **or** use **Tracking → By PO
  → Mark shipped** on that PO, entering the **real old invoice number** (required; PO shows Fulfilled).
  Never both for the same PO, or the quantities count twice.

| | Test | Prod |
|---|---|---|
| App (UI) | https://gray-meadow-0ebc61d10-test.centralus.6.azurestaticapps.net | https://gray-meadow-0ebc61d10.6.azurestaticapps.net |
| Database (Azure SQL) | `shriyaminingdbtest` | `shriyaminingdb` |
| Code branch | `test` | `main` |

Everything runs on the existing Azure resources: no new services, no new cost.

--TestPipeline

---

## Part A: Before you start (both environments)

### A1. Test environment only: bring the test database back into the pipeline
The test pipeline currently skips the database steps (a temporary change while the test database
was unavailable: `if: github.ref != 'refs/heads/test'` on the `migrate` job in
`ShriyaMiningIndustries_api/.github/workflows/deploy-ShriyaMiningIndustriesApi.yml`).

1. Check the test database is reachable (Azure portal → SQL databases → `shriyaminingdbtest` → Query editor).
2. Remove that `if:` line (ask Claude), push to `test`, and check the pipeline's **migrate** job
   succeeds. The test database then has the same tables and procedures as prod (V018–V021: old
   invoices, buyer addresses, ship-to).

Without this, Test runs new code against an old database and the rehearsal won't be meaningful.

### A2. Deploy the final code
Push the final code to `test` (and, for Prod, to `main`) and wait for every pipeline job to pass:
API **test → migrate → deploy**, then the UI build. (Ask Claude to deploy; it reports the results.)

### A3. Make sure you can undo
Azure portal → the database → **Backups / Restore**: check that point-in-time restore covers
today. If the clean-up goes wrong, the database can be restored to a moment before it.

---

## Part B: Clean out test / junk data

**Everything under Admin's set-up pages is kept, always** (agreed 2026-11-25): users, roles and permissions,
buyers (with their ship-to addresses and invoice email lists), crate types, delivery rules and the company
profile (its Services, email templates and app settings too). The script has no way to delete them.

Open Azure portal → the database (`shriyaminingdbtest` or `shriyaminingdb`) → **Query editor**.

### Option 1 (recommended): the safe script
File: `ShriyaMiningIndustries_api/database/scripts/go-live-clear-business-data.sql`

**The script's `@Confirm` is already set to `N'DELETE ALL BUSINESS DATA'`: running it deletes
straight away** (it lists the row counts before deleting and after). Everything is deleted in one
transaction: if anything fails, nothing is deleted.

1. Make sure you're in the right database (`shriyaminingdbtest` or `shriyaminingdb`).
2. Paste the script and run it.
3. Check the counts at the end: invoices and POs 0; buyers, crate types, delivery rules, users and roles unchanged.

To only see the counts without deleting, change the line to `DECLARE @Confirm NVARCHAR(50) = N'';`.

Settings at the top of the script:

| Setting | Default | Meaning |
|---|---|---|
| `@ClearLogs` | `1` | Clears the error log, event log, email log and API payload log |

Always kept: users, roles, permissions, buyers (ship-to addresses, invoice email lists), crate types, delivery
rules, the company profile, error codes, and the migration journal. (Until 2026-11-25 buyers and crate types had
`@KeepBuyers` / `@KeepCrateTypes` switches that could delete them; they were removed.)

### Option 2: the manual script (the same deletes as the safe script, without row counts or the confirm step)
```sql
SET XACT_ABORT ON;   -- any error rolls back everything
BEGIN TRAN;

-- Invoices and everything hanging off them (emails to buyers point at invoices).
DELETE FROM app.InvoiceReviewDecision;
DELETE FROM app.InvoiceReviewRecipient;
DELETE FROM app.InvoiceReviewRequest;
DELETE FROM app.InvoiceContainerFit;
DELETE FROM app.InvoiceStatusChange;
DELETE FROM app.InvoiceSnapshot;
DELETE FROM app.InvoiceLine;
DELETE FROM app.LegacyInvoiceUpload;
DELETE FROM app.InvoicePurchaseOrder;
DELETE FROM app.Invoice;

-- POs: break the two loops (PO line <-> its latest upload row, PO <-> its current PDF) first.
UPDATE sales.POLine SET LatestImportRowId = NULL WHERE LatestImportRowId IS NOT NULL;
UPDATE sales.PurchaseOrder SET CurrentDocumentId = NULL, ShipToAddressId = NULL
  WHERE CurrentDocumentId IS NOT NULL OR ShipToAddressId IS NOT NULL;
DELETE FROM sales.POEdit;         -- Edit PO history
DELETE FROM sales.POStatus;       -- stored PO statuses
DELETE FROM sales.POImportRow;
DELETE FROM sales.POLine;
DELETE FROM sales.POImportBatch;
DELETE FROM sales.PODocument;     -- the stored PDFs
DELETE FROM sales.PurchaseOrder;

-- Optional: the logs (the safe script does this when @ClearLogs = 1).
-- DELETE FROM log.ErrorLog; DELETE FROM log.ApplicationEvent; DELETE FROM app.EmailLog; DELETE FROM log.PayloadLog;

-- Buyers, crate types, delivery rules, users, roles and the company profile are never deleted.
COMMIT;
```
Order matters (children before parents): a wrong order stops on a foreign key and the whole transaction
rolls back, so nothing is half-deleted.

---

## Part C: Set up and load the data (in this order)

| # | Step | Where | Check |
|---|---|---|---|
| C1 | **Company profile**: legal name, address, IEC, bank, declaration | Admin → Company profile | Printed on every invoice |
| C2 | **Crate types**: every crate size you use, with correct inside dimensions | Admin → Crate types | Needed for crate sizes and the 20 ft container fit check |
| C3 | **Buyers**, by hand: name, code (e.g. NSR), PO format (PDF), address, phone, fax, email, website, country, and **every ship-to address** (Anaheim, Atlanta Warehouse…). Buyers are kept by the clean-up, so on later runs only check them | Admin → Buyers | A PO upload needs its buyer first. **Delivery rules**: the deploy sets up the NSR rules once (Admin → Delivery rules); they work when the buyer code is exactly **NSR** (the rules page warns if it's missing). Ship-to rules read the Ship To printed on the PO. The PO upload never changes a buyer; a PO whose Ship To matches a saved address is linked to it (for the ship-to filters) |
| C4 | **Upload POs from April 2026 to today.** If a PO has several versions (`_0_`, `_1_`…), upload the oldest first | PO Import | Each PO saved; versions show v1, v2… |
| C5 | *(Removed: the PO upload no longer offers to update buyers. Add buyer details and ship-to addresses in C3.)* | | |
| C6 | **Pre-April POs that still have something left to ship**: upload them too, plus their pre-April invoices (C7) | PO Import | Otherwise their remaining quantity is wrong |
| C7 | **Upload the old invoices from April 2026 to today**: drop all the files at once, then **Check** each one: confirm the PO line for every row, then **Save old invoice** as Delivered (or In Transit if still at sea) | Invoices → Upload old invoices | Every file shows **Saved** on Old invoices |
| C8 | Lines whose PO isn't in the app are saved as **LEGACY** (not tracked), with no check needed. Review the **Missing POs** panel | Old invoices → Missing POs (Copy list) | Run 1: the list of POs to collect. Run 2 and Prod: empty |

Why this order:
- **POs before old invoices.** An old invoice's lines can only be matched to PO lines that are
  already in the app. Lines saved before their PO is uploaded become LEGACY and don't reduce the
  PO. Fixing them means an Admin deletes that old invoice and you check it again.
- **Old invoices before any new invoice.** New invoice numbers continue after the highest
  number in the app, old invoices included (e.g. 78 after 77).

---

## Part D: Check before creating new invoices

| # | Check | Where | Expected |
|---|---|---|---|
| D1 | Old invoices appear | Tracking → Invoice, tick **Show delivered**, date **Any** | Every old invoice, tagged "Old"; LEGACY lines shown as "Not tracked" |
| D2 | POs show what's really left | Tracking → PO, tick **Show fulfilled** | Fully shipped POs show Fulfilled; open POs show only the true remainder |
| D3 | Anything odd | Tracking → **Needs a look** (⚠) | Over-invoiced lines only where an old invoice really shipped more |
| D4 | Ship-to links | Tracking → **Ship to** filter | POs listed under their addresses; few or none "Not matched" |
| D5 | Crate numbers by invoice number | Create invoice → pick lines → Review (don't save) → Invoice header | Crate starting no is the invoice's block: 78 → 101, 79 → 201 … 86 → 901, 87 → 101 again (up to 99 crates; a Manager or Admin can type another start). Old invoices keep their crate numbers |
| D6 | Crate sizes available | Create invoice → Review → crate size dropdown | Your crate types are listed |
| D7 | Invoice numbers continue | Invoices (highest number first) | The top invoice is the highest old number (e.g. 71). The first new invoice gets the next number (72), then 73… Plain numbers only, the format downstream systems use |
| D8 | Every PO has a delivery location | Purchase Orders → **Delivery** column (and its ▾) | New York · TLBR, Houston · DAL, Los Angeles · BED / Anaheim, Savannah · Atlanta Warehouse; none left blank. A PO whose items go to different places was blocked at upload |

When D1–D8 look right on **Test run 2**, repeat Parts A3–D on **Prod**. Then start creating new invoices.

---

## If something goes wrong

| Problem | What to do |
|---|---|
| An old invoice was saved with wrong lines | Admin: open it → **Delete old invoice**. Its quantities go back to the PO and its file returns to Old invoices to check again |
| A PO was uploaded for the wrong buyer | Clear and redo from Part B on Test; on Prod, ask Claude before changing data |
| A file can't be read | It's listed with the reason; download it, check it's the old invoice layout (INV sheet), or leave it out |
| The clean-up deleted the wrong thing | Restore the database to a moment before it (A3) |
| A pipeline step fails | Ask Claude: it reads the run log and fixes it |
