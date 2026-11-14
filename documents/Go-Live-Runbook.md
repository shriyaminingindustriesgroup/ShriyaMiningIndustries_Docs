# Go-live runbook: clean data, load POs and old invoices (Test, then Prod)

Agreed 2026-11-14. Run the whole thing on **Test** first as a rehearsal, then the same steps on
**Prod**. Do the steps in order: each one depends on the ones before it.

| | Test | Prod |
|---|---|---|
| App (UI) | https://gray-meadow-0ebc61d10-test.centralus.6.azurestaticapps.net | https://gray-meadow-0ebc61d10.6.azurestaticapps.net |
| Database (Azure SQL) | `shriyaminingdbtest` | `shriyaminingdb` |
| Code branch | `test` | `main` |

Everything runs on the existing Azure resources: no new services, no new cost.

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

**Users, roles and permissions are not touched.** Manage them yourself (Admin → Users / Roles).

Open Azure portal → the database (`shriyaminingdbtest` or `shriyaminingdb`) → **Query editor**.

### Option 1 (recommended): the safe script
File: `ShriyaMiningIndustries_api/database/scripts/go-live-clear-business-data.sql`

1. Paste it and run it **as it is**. It deletes nothing; it lists each table's row count and
   whether it would be deleted or kept.
2. Check the numbers.
3. Change `@Confirm` to `N'DELETE ALL BUSINESS DATA'` and run it again. Everything is deleted in one
   transaction: if anything fails, nothing is deleted.

Settings at the top of the script:

| Setting | Default | Meaning |
|---|---|---|
| `@KeepBuyers` | `0` | `0` deletes buyers and their ship-to addresses (you add the real ones in step C3). `1` keeps them |
| `@KeepCrateTypes` | `1` | Crate types are **kept**: invoice lines need them for crate sizes and the container fit check |
| `@ClearLogs` | `1` | Clears the error log and event log |

Always kept: users, roles, permissions, the company profile, crate types (by default), error codes,
and the migration journal.

### Option 2: the manual script (same result, no row counts or confirm step)
```sql
SET XACT_ABORT ON;   -- any error rolls back everything
BEGIN TRAN;

-- Invoices (uploads point at invoices; invoice lines point at PO lines)
DELETE FROM app.LegacyInvoiceUpload;
DELETE FROM app.InvoiceLine;
DELETE FROM app.InvoiceSnapshot;
DELETE FROM app.InvoiceStatusChange;
DELETE FROM app.InvoiceHistory;
DELETE FROM app.InvoiceContainerFit;
DELETE FROM app.Invoice;

-- Break the two circular links
UPDATE sales.PurchaseOrder SET CurrentDocumentId = NULL;  -- PO -> its current PDF
UPDATE sales.POLine        SET LatestImportRowId = NULL;  -- PO line -> its latest stored row

-- Children before parents
DELETE FROM sales.POStatus;       -- stored PO statuses
DELETE FROM sales.POImportRow;    -- -> POLine, POImportBatch
DELETE FROM sales.POLine;         -- -> PurchaseOrder, POImportBatch
DELETE FROM sales.PODocument;     -- -> PurchaseOrder (the stored PDFs)
DELETE FROM sales.PurchaseOrder;
DELETE FROM sales.POImportBatch;

-- Test buyers too (recommended, since buyers are added fresh in step C3)
DELETE FROM app.BuyerAddress;
DELETE FROM app.Buyer;

COMMIT;
```
Order matters: `app.LegacyInvoiceUpload` must go **before** `app.Invoice` (it points at invoices),
or the whole transaction rolls back.

---

## Part C: Set up and load the data (in this order)

| # | Step | Where | Check |
|---|---|---|---|
| C1 | **Company profile**: legal name, address, IEC, bank, declaration | Admin → Company profile | Printed on every invoice |
| C2 | **Crate types**: every crate size you use, with correct inside dimensions | Admin → Crate types | Needed for crate sizes and the 20 ft container fit check |
| C3 | **Buyers**: name, code (e.g. NSR) and PO format (PDF) for each buyer | Admin → Buyers | A PO upload needs its buyer first |
| C4 | **Upload POs from April 2026 to today.** If a PO has several versions (`_0_`, `_1_`…), upload the oldest first | PO Import | Each PO saved; versions show v1, v2… |
| C5 | On **each buyer's first PO**, the "buyer details differ" panel appears: tick the name, address, phone, fax, email, website and the **Ship To**, then **Update buyer** | PO Import → review | Admin → Buyers shows the details and ship-to addresses (Anaheim, Atlanta Warehouse…) |
| C6 | **Pre-April POs that still have something left to ship**: upload them too, plus their pre-April invoices (C7) | PO Import | Otherwise their remaining quantity is wrong |
| C7 | **Upload the old invoices from April 2026 to today**: drop all the files at once, then **Check** each one: confirm the PO line for every row, then **Save old invoice** as Delivered (or In Transit if still at sea) | Invoices → Upload old invoices | Every file shows **Saved** on Old invoices |
| C8 | Lines whose PO isn't in the app are saved as **LEGACY** (not tracked). Review them | Invoices → search "LEGACY" | Only pre-April POs, samples and discounts should be LEGACY |

Why this order:
- **POs before old invoices.** An old invoice's lines can only be matched to PO lines that are
  already in the app. Lines saved before their PO is uploaded become LEGACY and don't reduce the
  PO. Fixing them means an Admin deletes that old invoice and you check it again.
- **Old invoices before any new invoice.** New invoices continue the crate numbers after the
  highest old crate (e.g. 1011 after 1010).

---

## Part D: Check before creating new invoices

| # | Check | Where | Expected |
|---|---|---|---|
| D1 | Old invoices appear | Tracking → Invoice, tick **Show delivered**, date **Any** | Every old invoice, tagged "Old"; LEGACY lines shown as "Not tracked" |
| D2 | POs show what's really left | Tracking → PO, tick **Show fulfilled** | Fully shipped POs show Fulfilled; open POs show only the true remainder |
| D3 | Anything odd | Tracking → **Needs a look** (⚠) | Over-invoiced lines only where an old invoice really shipped more |
| D4 | Ship-to links | Tracking → **Ship to** filter | POs listed under their addresses; few or none "Not matched" |
| D5 | Crate numbers continue | Create invoice → pick lines → Review (don't save) | Crates start after the last old crate (e.g. 1011) |
| D6 | Crate sizes available | Create invoice → Review → crate size dropdown | Your crate types are listed |

When D1–D6 look right on **Test**, repeat Parts A–D on **Prod**. Then start creating new invoices.

---

## If something goes wrong

| Problem | What to do |
|---|---|
| An old invoice was saved with wrong lines | Admin: open it → **Delete old invoice**. Its quantities go back to the PO and its file returns to Old invoices to check again |
| A PO was uploaded for the wrong buyer | Clear and redo from Part B on Test; on Prod, ask Claude before changing data |
| A file can't be read | It's listed with the reason; download it, check it's the old invoice layout (INV sheet), or leave it out |
| The clean-up deleted the wrong thing | Restore the database to a moment before it (A3) |
| A pipeline step fails | Ask Claude: it reads the run log and fixes it |
