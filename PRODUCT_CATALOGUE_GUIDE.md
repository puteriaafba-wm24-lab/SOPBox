# SOPBox product catalogue and reorder points

Current behavior is Step 16; read `STEP16_UPDATE_GUIDE.md` first for required backend account setup, invitations and server-enforced permissions. Old demo passwords no longer work. Step 15 describes separate supplier invoices, and Step 14 describes quantity-only POs, low-stock drafts and stock entry. Earlier steps remain historical references, not current authentication instructions.

## Run it from Visual Studio Code on Windows

Use **Visual Studio Code**, not Visual Studio 2022. This is a React/Vite frontend with a Python FastAPI backend, so opening `index.html` with Live Server will not start the app or its API.

1. Install the current Node.js LTS release from [nodejs.org](https://nodejs.org/) if `node -v` or `npm -v` is not recognized. Close and reopen VS Code after installing.
2. In VS Code, choose **File → Open Folder** and open the project folder that contains `src`, `README.md`, and this guide.
3. Open **Terminal → New Terminal**. In the project root, start the backend:

   ```powershell
   cd .\src\backend
   py -m venv .venv
   .\.venv\Scripts\python.exe -m pip install -r requirements.txt
   .\.venv\Scripts\python.exe -m uvicorn app.main:app --reload --port 8000
   ```

   Keep this terminal running. On the first start, SOPBox creates `src/backend/sopbox.db` and copies the starter products into it once.

4. Open a **second** VS Code terminal and start the frontend:

   ```powershell
   cd .\src\frontend
   npm install
   npm run dev
   ```

5. Open the local address printed by Vite, usually `http://localhost:5173`. Sign in using the demo login shown on the sign-in page. Do not use Live Server or open `index.html` directly.

If PowerShell says `npm` is not recognized, install Node.js LTS, restart VS Code, and verify `node -v` and `npm -v` in a new terminal.

## Add, edit, archive, and restore products

1. Sign in as the demo admin and open **Products & SOP**.
2. Use **Manage product types** to add, rename, or delete categories/types. Renaming updates the products assigned to that type. A type in use cannot be deleted; first edit those products and assign them another type. Product type selection is a searchable dropdown.
   Use **Manage stock sections** beside it to add, search, rename or safely delete storage places. Renaming keeps balances; deletion cannot discard stock or leave a product with no section.
3. Select **Add product** or **Edit**.
4. Enter the product name and type. Set the temperature limits with the +/− controls, or type values directly.
5. In **Bundle size**, enter its quantity, your base-unit name, and your bundle-unit name. For example, quantity `250`, base unit `pcs`, and bundle unit `box` means `250 pcs / box`. All three values are required.
6. Current stock is a read-only list by section, with bundles, loose units and a combined equivalent total. New products start at 0. Use Receiving Check → Allocate & receive stock to add deliveries, or Stock Count for opening/physical counts. Choose a base-unit or bundle reorder threshold; the selector uses your unit names.
   Tick one or more **Stock sections**, such as Chiller and SubStore. These are separate from the product type and determine where a product can be counted. Assignments do not duplicate its stock. Existing stock starts in Store; keep Store assigned until its physical balance is zero.
7. Select **Save product**. The changes are written to SQLite and remain after refreshing the browser or restarting the backend. Quantities are stored in base units, while the chosen reorder unit is retained for display.
8. **Archive** removes a product from future receiving checks but keeps the record available for old delivery lines. Use **Edit** on an archived product and turn on **Available for receiving** to restore it.

## Bulk edit products

1. In **Products & SOP**, open **Filter products** to tick one or more types/sections and Active, Archived or Low stock. Use the separate search box for product code/name/type.
2. Tick individual product checkboxes, or use **Select active products on this page**. Each page displays at most 50 products; selections can be retained across pages.
3. Select **Bulk edit selected**. Choose **Change product type**, **Change temperature range**, or both.
4. Only checked fields are changed. When changing the temperature range, leave a limit blank to clear that limit for every selected product.
5. Select **Apply to selected products**. The server applies the selected changes as one database transaction.

## Import products from a supplier document

1. Sign in as admin and choose **Products & SOP → Import supplier document**.
2. Upload a PDF, JPG, PNG, or WebP file up to 12 MB. The backend sends up to five PDF pages (or one image) to the configured RunPod vision endpoint.
3. Review each suggestion. Fill in or correct the product code, name, existing product type, base unit, bundle size, bundle unit, and optional temperature limits.
4. Untick any rows you do not want to add, then choose **Add selected products**. The selected rows are created together; product codes and names must be unique.
5. Imported products start with current stock and reorder point set to zero. Set reorder points in **Products & SOP**. Receive goods or record an opening count in **Stock Count** to populate stock.

The source defaults to `EXTRACTION_MODE=mock`. The worker now includes a vision-service adapter, but no model is connected or verified. Product/stocktake AI extraction remains unavailable until the backend and worker are securely configured. See `src/ai/README.md`.

## F&B demo data for backend testing

The backend seed fixture contains the 10 requested products, FD001–FD006 and DR001–DR004, across **Chiller**, **Frozen**, **Room**, and **Cold**. It includes your bundle sizes, current stock, reorder thresholds, temperature limits, and SOP descriptions. Chicken Breast and Cheddar Cheese start below their reorder points. Initial quantities are assigned to **Store**; Front and SubStore start empty when you add them.

To try the fixture without overwriting your existing database, open a new VS Code PowerShell terminal:

```powershell
cd .\src\backend
$env:SOPBOX_DB_PATH = (Join-Path $PWD "sopbox_step7_demo.db")
.\.venv\Scripts\python.exe -m uvicorn app.main:app --reload --port 8000
```

Keep this terminal open, then run the frontend in a second terminal as usual. The backend creates `sopbox_step7_demo.db` and seeds the 10 products and four product types only when that database is empty. To use the demo database again later, set the same `SOPBOX_DB_PATH` in the backend terminal before starting Uvicorn. Your normal `sopbox.db` is left unchanged.

To adjust the demo fixture before creating a fresh demo database, edit `src/backend/demo_products.json`. Add products to its `products` array and add any new category to `product_types`. Each product requires a unique `id`, `product_code`, `name`, `category`, `base_unit`, at least one `pack_units` entry (`unit` + `base_qty`), and a reorder threshold stored in base units. A threshold whose display unit is a bundle must be a multiple of that bundle's `base_qty`. Editing the JSON does not overwrite an already-created database; change products in **Products & SOP** or create a different empty demo database file to reload the fixture. Keep your actual `sopbox.db` backed up and do not delete it to test demo data.

## Low-stock alerts

The Products & SOP table shows the current stock and reorder point, but no longer displays a suggested-order quantity. The low-stock indicator compares the total stock (bundles × units per bundle + loose units) with the threshold. Set the threshold to `0` to turn off the alert for a product.

## Stock Count and stocktake

1. Open **Stock Count** and choose a **Stock section**. Store, Front, SubStore, Chiller, Frozen and Room are available by default; add custom sections as needed. Only products assigned to the section are eligible.
2. The count starts empty. Use **+ Add product**, search/filter, tick products, and choose **Done**. Only added products will be saved. Manually added products start at `0`; a blank field in an added row is saved as `0`. Remove any product you did not count. Extra loose units automatically become full bundles in summaries and saved records.
3. Review each row's calculated total and variance against stock at the selected location. Add an optional note, select **Save stocktake**, then confirm in the app.
4. Saving creates a permanent history record such as `ST-20261023-01` and updates the selected products at that location in one database transaction. Other locations and unselected products remain unchanged. The product's total is the sum of all locations. Save or clear counts before changing location; repeat the count for the next location.
5. Products without a configured required bundle size cannot be counted until their bundle size is set in **Products & SOP**.

## Spreadsheet, printed sheets, and reports

- **Download count template** creates a real XLSX for the chosen section, containing assigned products only and a separate worksheet per type. Use the type filter to limit the workbook to one type. It includes product code/name, bundle and loose counts, bundle size/units, and Stock Section. Keep headers/codes/sections unchanged.
- **Print blank count sheet** prints the currently filtered product list with space to write bundle and loose-unit counts. Select Save as PDF in the browser to keep a printable copy.
- **Upload Excel / CSV** accepts digital `.xlsx` and `.csv` files. XLSX reads all product-type tabs and skips the Instructions sheet. Both quantities blank means not counted and is skipped; enter `0` explicitly for empty stock. If one quantity is entered, the other blank quantity becomes zero. Match by product code (preferred) or exact name when code is absent. Wrong-section files are rejected. Review loaded rows before saving; uploading alone does not change inventory.
- Import one storage location at a time. Do not duplicate a product row. Maximum 500 product rows per stocktake.
- In **Stocktake history**, **Download PDF** opens a print-ready report. Choose **Save as PDF** and use the displayed `ST-YYYYMMDD-01` reference as the filename. The browser controls the final save location.
- **Download inventory summary PDF** opens a current stock report with one column per location and a combined product total. Choose **Save as PDF**. This is a live inventory snapshot, not a historical multi-location session.

### Handwritten/scanned sheets and AI

Scanned paper, photographs, and handwriting are not digital spreadsheet cells. This stocktake importer does **not** perform OCR, and renaming a scan to `.xlsx` will not make it readable. For now, enter handwritten counts in the downloaded spreadsheet or directly in Stock Count.

Stock Count now includes **Read handwritten sheet (AI)** with explicit review/correction. Its vision-service adapter still needs a real configured model. Unknown/unread values must be resolved by staff. Reviewed extra loose units are normalised when summarised/saved; extraction alone never changes stock. See `STEP10_UPDATE_GUIDE.md` and `src/ai/README.md`.

## Which files implement this section?

- `src/backend/app/models.py` — product, bulk-update, supplier-import, and stocktake validation.
- `src/backend/demo_products.json` — editable F&B seed fixture for a fresh database.
- `src/backend/app/catalogue.py` — validates and loads the JSON demo fixture.
- `src/backend/app/database.py` — SQLite storage, product management, atomic bulk updates/creation, and stocktake history/inventory transactions.
- `src/backend/app/extraction.py` — RunPod extraction client for delivery lines and supplier documents.
- `src/backend/app/main.py` — product, bulk, supplier-import, stocktake, and delivery-check APIs.
- `src/backend/app/spreadsheets.py` — section/type-based workbook templates and safe multi-tab digital imports.
- `src/ai/` — separate vision-service adapter and contracts; stocktake/product/invoice extraction needs a configured live service and staff review. Live OCR accuracy is not verified.
- `tasks/backend/`, `tasks/frontend/`, `tasks/ai/` — separate implementation task lists.
- `src/frontend/src/types.ts` — frontend product types.
- `src/frontend/src/components/ProductsPage.tsx` — catalogue, bulk editing, product form, searchable product types, supplier document upload, and reviewed import.
- `src/frontend/src/components/StockTakePage.tsx` — bundle-based physical count, variance review, and stocktake history.
- `src/frontend/src/App.tsx` — loads and saves product data through the backend API.
- `src/frontend/src/theme.css` — page and form styling.

## Receive against a purchase order

As demo admin, open **Purchase Orders → View → Receive stock** on an issued order. Tick arriving products, enter bundles/loose units, temperature and a unique delivery/DO reference; then review stock-section allocations and confirm. Ordered/received/outstanding quantities update atomically with stock. Different deliveries may share an invoice but must use distinct delivery references. See `STEP13_UPDATE_GUIDE.md`, especially its legacy-PO instructions.

**Receiving Check** is now for non-PO goods. Typed references do not link orders, and older free-text receipts are never automatically re-posted or inferred. Earlier issued POs without conversion snapshots require independent remaining-quantity review, not a guessed migration.

## Transfer existing stock

Use **Products & SOP → Transfer stock** on the product row, or open **Stock Transfers → + Transfer stock** as demo admin. Assign the product to both places first. Choose source, destination and bundles/loose units; review before/after balances and confirm. The source decreases, destination increases, and overall stock stays unchanged. History uses `TR-YYYYMMDD-01` references. See `STEP12_UPDATE_GUIDE.md` for examples, retry safety and the difference from receiving/stocktakes.

## Important demo limitation

This starter app's sign-in is demo-only. Backend inventory routes do not enforce server-side user permissions, so do not publish it for public or sensitive use. Add real authentication/authorization before deployment. Product-editor stock is read-only; receiving, stocktakes and transfers have durable records. Legacy direct product API requests can still write opening stock unless `preserve_stock` is true, so those API compatibility paths also need production access controls. Recorded staff names are browser-supplied, not verified identities.
