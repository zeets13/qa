# QA Staging Release & Testing Guide (Nov 19 – Nov 24, 2026)

> **Branches Covered**
> - **Backend (`origin/feature-staging`):** up to commit `73e37c91` (Nov 24, 2026)
> - **Frontend (`origin/feature-ui`):** up to commit `a3a7e639` (Nov 24, 2026)
>
> **Focus Areas:** Modular Workspaces & Project Setup, Modular Estimation (BD), and PWD Rate Analysis (RA), plus a brief summary of other staging updates.

---

## Before Testing on Staging (DevOps / Environment Checklist)

Ask the team to verify these are applied on Staging before starting QA:
1. **Database Migrations:** `alembic upgrade head` reaches revision `d6e1f4a8b2c3` (includes workspace catalog, regions, drafts/structural discipline, estimation templates, and the `"File Upload"` / `"Review Plans"` label rename).
2. **Legacy Project Backfill:** `python scripts/backfill_project_configs.py --apply` has been run so older projects created before Workspace v1 get a valid `config` snapshot and don't appear locked out.
3. **S3 / Env Config:** Custom Estimation Template uploads (`Add Custom`) require valid S3 credentials; Redis Pub/Sub progress listener and AWS Lambda offload env vars are configured if testing legend counting/PDF processing.

---

## Quick Map: What Can Be Tested in UI vs. Backend-Only

| Feature Area | Ready in Frontend + Backend (Test in UI) | Backend-Only Right Now (API / Postman) |
|---|---|---|
| **Workspaces & Setup** | 2-tab Project Setup (`Information` + `Configuration`), Modules & Tools selection, Saved Setups, Config Excel Import/Export, Draft Projects + Autosave, Dynamic Sidebar, Step Guards & Skip Optional Steps | Project Duplication (`POST .../duplicate`), Team Member Assignment (`PUT .../team`), Start/Due Dates, `+ Custom` Discipline creation (`POST /v1/catalog/disciplines`), Company Default Region (`PATCH /v1/company/{id}/default-region`) |
| **Modular Estimation (BD)** | BD Step 1 **Quantities** (`Takeoff` / `Upload a BOQ or Excel` / `From Template` + sample Excel download), Choose Letterhead Template modal (`Team Library` + `My Templates` + `Add Custom`), Pricing Sources & Rate Zones, Sector/Currency/Language/VAT/Markup/Rounding, New Estimation UI tabs | Export-level letterhead PDF styling (`template_id` is saved in config; `tax_percent`/`markup_percent`/`rounding` are stored in config while BD sheet summary still keeps `subtotal == grand_total` for now) |
| **PWD Rate Analysis (RA)** | RA icon button in **Unit Rate** cell (Symbol Count & Cable Pricing), 3 breakdown formats (Direct Tabular, Multi-Group, Detailed Breakup A/B/C), live recalculation on field edit, **Apply Changes** saving back to row & sheet totals | Reset Row RA to Official Baseline (`DELETE .../rate-analysis` — no Reset button in UI yet), Add/Delete custom section lines (`add_lines`/`delete_lines` in `PATCH`), Search Modal standalone RA preview (`GET /v1/pwd-estimation/rate-analysis`) |

---

## 1. Workspaces & Project Setup

### What Changed
- **Backend-driven Workspace (`GET /v1/projects/{id}/workspace`):** Sidebar steps, labels, optional/required states, and tool unlocks now come from the backend instead of hardcoded frontend rules.
- **Renamed Takeoff Steps:** `"Upload PDF"` is now **`File Upload`**, and `"Page Classification"` is now **`Review Plans`**.
- **6 Seeded Disciplines:** `Electrical`, `Architecture`, `Civil`, `Plumbing`, `Mechanical`, plus newly added **`Structural`**.
- **3 Regions & Hybrid Project Types:** `AU`, `UK`, and `BD` (e.g., `"Electrical (BD)"`, `"Structural (BD)"`) with regional currency & VAT defaults.
- **Independent Module Unlocking:** Enabling both **Takeoff** and **Estimation** lets the user jump straight into **Estimation** without waiting for Takeoff to finish.
- **Draft Projects & Autosave:** Users can **Save draft** on Tab 1 or Tab 2 without consuming plan project limits, auto-save edits after pausing, reopen drafts from Project Cards or the Project Switcher (`Draft · <name>`), and finalize them later.
- **Saved Setup Presets & Configuration Excel I/O:** Users can save reusable configurations (`Saved Setups`), download a sample `.xlsx`, export a setup/project `.xlsx`, and import an `.xlsx` via a 3-step wizard (`Preview & map` → `Validate` → `Review changes`).
- **Step Guards & Skips:** Direct URL navigation to a locked, disabled, or draft step shows a friendly **Step Blocked** screen with a redirect button instead of breaking. Optional steps (like `Legend Setup` or BD `Quantities`) can be skipped/unskipped from the sidebar.

### QA Test Checklist — Workspaces

#### A. Project Creation (`Information` & `Configuration` Tabs)
1. **Required Fields & Tab Flow**
   - Open **Project Setup** for a new project.
   - Verify the two tabs at the top: **Information** and **Configuration**.
   - Try creating without a **Project Name** → Create button should remain disabled / show validation.
   - Check the **Project Type** dropdown includes **Structural** alongside Electrical, Architecture, Civil, Plumbing, and Mechanical across regions (`AU`, `UK`, `BD`).
2. **Module Selection (Step 1 of Configuration)**
   - **Takeoff only:** Uncheck Estimation → verify **Workflow Preview** only shows Takeoff steps (`File Upload`, `Review Plans`, `Legend Setup` if applicable, `Takeoff Workspace`, `Review and Export`).
   - **Estimation only:** Uncheck Takeoff → verify **Workflow Preview** only shows Estimation steps (`Quantities` + `Cost Estimation` for BD; `Cost Estimation` only for AU/UK). Creating the project should land directly on **Quantities** (BD) or **Estimation** (AU/UK), and Takeoff routes should show **Step Blocked** if visited directly.
   - **Both Modules on:** Verify message *"Both on: Takeoff and Estimation can be worked on independently"* and verify after creation that both **File Upload** (Takeoff) and **Quantities** / **Cost Estimation** (Estimation) are immediately unlocked in the sidebar.
   - **Zero modules:** Uncheck both → verify validation error and Create disabled.
3. **Takeoff Settings (Step 2 of Configuration)**
   - For **Electrical**, verify tools include `Symbol Count` and `Cable Drawing`, plus **Legends Management** (`Self Legends` vs `Legend Library`).
   - Uncheck all Takeoff tools while Takeoff is enabled → verify validation blocks project creation.

#### B. Draft Projects & Local/Server Autosave
1. **Save Draft & Limits**
   - Enter a project name (even without finishing module settings) and click **Save draft**.
   - Verify button transitions to **Saving…** → **Draft saved**.
   - Check the Projects list and left **Project Switcher** → project appears marked as a **Draft** (`Draft · <Project Name>`).
   - *(If testing plan quota)* Verify saving a draft does **not** increment the active/monthly project quota until finalized.
2. **Reopen, Autosave & Finalize**
   - Open the draft from the dashboard card or Project Switcher → verify saved values reload cleanly.
   - Modify a setting (e.g., toggle a module or change project name) → wait briefly and verify **Draft saved** autosave triggers without errors (even if modules are temporarily empty).
   - Visit a step URL manually (e.g., `/projects/upload?projectId=<draftId>`) → verify **Step Blocked** shows *"This project is still a draft. Finish setting it up first."* with a **Finish setup** button.
   - Click the final **Create / Finalize** button → verify draft converts to an active project and redirects to the first unlocked step.
3. **Local Unsaved Draft Recovery**
   - Start creating a new project without clicking Save draft, refresh the page → verify the **Restored Draft Notice** banner appears with options to dismiss or **Start fresh**.

#### C. Saved Setups & Configuration Excel Import/Export
1. **Saved Setups**
   - Configure modules & settings → click **Save setup**, give it a name, save.
   - Start a new project → select the preset from the **Saved Setup** picker → verify modules, tools, and estimation settings populate accurately.
   - Delete a saved setup and verify it disappears from the list.
2. **Config Excel Import / Export**
   - Click **Export config** → verify an `.xlsx` file downloads with 2 sheets (`Project Configuration` and `Allowed Values`).
   - Open **Import configuration Excel**:
     - Click **Download sample** → verify sample `.xlsx` downloads.
     - Upload a valid `.xlsx` (≤ 5 MB) → step through **1. Preview & map** → **2. Validate** → **3. Review changes** → click **Apply** and verify form fields update.
     - Upload an invalid file type or file > 5 MB → verify clear error toast.
     - Upload an `.xlsx` with an invalid field value (e.g., wrong-region pricing source) → verify Step 2 flags validation errors and blocks advancing.

#### D. Sidebar Navigation, Step Guards & Optional Skips
1. **Step Labels & Ordering**
   - Open an active Takeoff project → confirm sidebar displays **File Upload** and **Review Plans** (not old `"Upload PDF"` / `"Page Classification"` labels), and `Takeoff` appears above `Estimation`.
2. **Sequential Locking & Completion**
   - Before uploading files, confirm **Review Plans** (and downstream takeoff steps) are locked.
   - Try navigating directly via URL to a locked step → confirm **Step Blocked** renders with *"This step is not available yet. Finish the earlier steps first."* and a **Go to the current step** button.
3. **Skip / Unskip Optional Steps**
   - On an optional step (`Legend Setup` for non-Civil disciplines, or `Quantities` in BD Estimation), use the sidebar skip control.
   - Verify the step marks as skipped and immediately unlocks the next step (`Takeoff Workspace` or `Cost Estimation`).
   - Unskip the step and verify state updates cleanly (unless the downstream step is already completed).

---

## 2. Modular Estimation (BD Flow & Settings)

### What Changed
- **BD Step 1 — Quantities (`boq_upload`):** New Estimation Step 1 exclusively for **Bangladesh (`BD`)** projects (`AU` and `UK` skip straight to `Cost Estimation`). Supports 3 quantity sources:
  1. **From Takeoff** (when Takeoff module is also enabled; syncs symbol/cable quantities into the sheet).
  2. **Upload a BOQ or Excel** (`.xlsx`, `.xlsm`, `.csv` up to 10 MB, with downloadable sample file, live column auto-mapping preview, row count & warnings, and `replace` import on continue).
  3. **From Template** (6 starter BOQ templates filtered by discipline + general blank starter).
- **Choose Template for Estimation (Letterhead/Export Templates):** New modal in Project Setup → Configuration → Estimation Settings with **Team Library** (6 built-in presets: `Corporate`, `Minimal`, `Engineering`, `Blue Modern`, `Classic Formal BOQ`, `Compact Summary`), **My Templates**, search bar, live preview, **Add Custom** upload (`.pdf`, `.xlsx`, `.docx` in UI; backend also accepts `.png`, `.jpg`/`.jpeg` up to 10 MB), and delete for custom templates.
- **Pricing Sources, Rate Zones & Estimation Settings:**
  - Region-filtered pricing sources (`PWD Schedule 2022` recommended for BD, plus `LGED`, `BWDB`, `DSCC`, `BIWTA`, and global `Company price book` & `Project rates`) OR **No pricing source** (blank estimation table).
  - **Rate Zone** selector (shows when selected source has >1 zone, e.g., PWD: Dhaka/Mymensingh, Chattogram/Sylhet, Khulna/Barishal/Gopalganj, Rajshahi/Rangpur).
  - **Sector** (`Government` / `Private` — auto-updates BD VAT between `7.5%` and `15.0%` unless manually overridden), **Currency** (`BDT`, `AUD`, `GBP`, `USD`), **Language** (`English`, `Bengali`), **Tax %**, **Markup %**, and **Rounding** (`No rounding`, `2 decimal places`, `Round to nearest 1`, `Round to nearest 10`).
- **Updated BD Estimation UI:** Streamlined 3-tab bar (**Symbol Count**, **Load Calculate**, **Cable Pricing**) and updated container styling.

### QA Test Checklist — Modular Estimation

#### A. Estimation Settings in Project Setup
1. **Sector & VAT Auto-Default (BD)**
   - Select `Electrical (BD)` → open Estimation settings.
   - Switch **Sector** between **Government** and **Private** → verify default VAT updates (`7.5%` for Government, `15%` for Private).
   - Manually type a custom Tax % and Markup % and change Currency / Language / Rounding → verify summary line updates accurately.
2. **Pricing Source & Rate Zone**
   - Verify `PWD Schedule 2022` is pre-selected as recommended for BD, and **Rate Zone** dropdown lists all 4 BD zones.
   - Select **No pricing source** (blank table) → create project → confirm estimation sheet opens cleanly without pre-populated catalog errors.
3. **Choose Template Modal**
   - Click **Choose Template** in Estimation settings.
   - Verify **Team Library** lists the 6 presets (`Corporate`, `Minimal`, `Engineering`, `Blue Modern`, `Classic Formal BOQ`, `Compact Summary`).
   - Test search filtering and preview panel on the right.
   - Switch to **My Templates** → click **Add Custom** and upload a `.pdf`, `.xlsx`, or `.docx` (≤ 10 MB) → verify it appears, can be selected, and can be deleted (2-click confirm). Built-in Team Library presets should not be deletable.

#### B. BD Step 1 — Quantities Page (`/projects/estimation/quantities`)
1. **Visibility by Region & Module**
   - Create an **Electrical (BD)** project with Estimation enabled → confirm **Quantities** appears as Step 1 under Estimation in the sidebar.
   - Create an **Electrical (AU)** or **Electrical (UK)** project with Estimation enabled → confirm **Quantities** does **not** appear; Estimation starts directly at **Cost Estimation**.
   - If Estimation-only BD project (no Takeoff), confirm **From Takeoff** option is not available/defaulted and **Upload a BOQ or Excel** is selected by default.
2. **Option 1: Upload a BOQ or Excel**
   - Click **Download the sample file** → confirm `quantiify-boq-sample.xlsx` downloads.
   - Drag/drop or browse an `.xlsx`, `.xlsm`, or `.csv` file → confirm preview displays:
     - Total **rows read** (and skipped rows if any).
     - Auto-detected column mapping (`item_code`, `item_name`, `description`, `unit`, `quantity`, `unit_rate`, etc.).
   - Click **Continue to Estimation** → confirm rows populate the BD Estimation sheet and **Quantities** step is marked completed.
   - Negative tests: try uploading `.pdf` or `.txt` (should reject with *"Choose an .xlsx, .xlsm or .csv file."*), file > 10 MB (should reject), or an empty/unparseable file (`Continue` stays disabled).
3. **Option 2: From Template**
   - Select **From Template** → verify starter templates match the project's discipline (e.g., Electrical shows `Commercial Electrical Fitting BOQ`, `Distribution Board & Main Cabling BOQ`, plus `Blank Starter Estimation Sheet`; Structural shows `Structural & Civil Works BOQ`).
   - Pick a template → click **Continue to Estimation** → confirm starter rows appear in **Symbol Count** / **Cable Pricing** tabs.
4. **Option 3: From Takeoff**
   - On a project with both Takeoff and Estimation enabled (after doing Takeoff), choose **From Takeoff** → click **Continue to Estimation** → confirm Takeoff symbols & cables sync into the Estimation sheet.

#### C. BD Estimation Guardrails
- Confirm **Export PDF** and **Export CSV** work on Estimation-only BD projects (regression fix `fd99a222` ensures Takeoff is not required for BD Estimation exports).
- Confirm BD Estimation / BOQ endpoints block `AU`/`UK` projects if hit directly (`403`).

---

## 3. PWD Rate Analysis (RA)

### What Changed
- **Backend Calculation Engine & 4,632 Official PWD Items:** Added full rate analysis breakdown data (`Dhaka & Mymensing PWD Rate Analysis.json`), calculation engine (`ROUND_HALF_UP`, 2 decimals for components/subtotals, nearest integer BDT for final `unit_rate_say`), and 5 API endpoints under `/v1/pwd-estimation/`.
- **Inline RA Popover Editor in Estimation Tables:** Rows matched to a PWD item that has a rate analysis (`has_rate_analysis: true`) now show a split-table icon button in the **Unit Rate** cell on both **Symbol Count** and **Cable Pricing** tabs.
  - *Note:* Subhead 13 (*Spare Parts For Vehicles Repair*) has no rate analysis in the PWD book, so those rows will **not** show the RA button.
- **3 Supported Breakdown Layouts:**
  1. **Direct Tabular (Single Group):** e.g., `2.1.2.1` (11 KV RMU), `6.20.11` (LED Tube Lamp) — shows `"Standard Rate"` + cost component table.
  2. **Direct Tabular (Multi-Group):** e.g., `8.1.1.1.1` / `8.1.1.29` (Passenger Lifts) — shows selectable **Rate Groups** radio pills (e.g., `"Price up to 3 stop"` vs `"Next per stop"`).
  3. **Detailed Breakup (Sections A, B, C + Summary):** e.g., `1.1.2.2` (Fan Point Wiring), `1.18.1.1.1` (Underground Cable), `1.21.1` (TC Cable) — shows collapsible **Section A (Materials)**, **Section B (Labours)**, **Section C**, and the **Summary** cost component table below.
- **Known VAT Difference to Watch For (Expected Behavior):**
  - The official PWD Rate Analysis book uses **`8.11%`** (`0.0811` = 7.5% grossed-up VAT), e.g., `1.1.2.2` = **`1,097` BDT**.
  - The PWD Schedule Catalog uses **`11.11%`** (`0.1111` = 10% grossed-up VAT), e.g., `1.1.2.2` = **`1,127` BDT**.
  - So opening RA on an unedited item may show `1,097` (at `8.11%`), while the table's catalog rate shows `1,127`. Changing the VAT `Factor %` field in RA from `8.11` to `11.11` will recalculate to `1,127`!

### QA Test Checklist — Rate Analysis

1. **Trigger Visibility & Icon State**
   - Open an Electrical (BD) Estimation project with matched PWD rows (e.g., `1.1.2.2`, `2.1.2.1`, `8.1.1.1.1`).
   - Verify the RA icon button appears inside the **Unit Rate** cell on **Symbol Count** and **Cable Pricing** rows that have `has_rate_analysis = true`.
   - Verify unmatched / custom rows and Subhead 13 rows do **not** show the RA icon.
   - Verify clicking the RA icon opens the **PWD Rate Analysis Breakdown** popover without triggering inline cell editing on the unit rate cell.
2. **Verify All 3 Formats Render Properly**
   - **Detailed Breakup (e.g., `1.1.2.2` Fan Point Wiring):**
     - Header displays item code pill (`1.1.2.2`), description, `Unit` (`Per point`), and `Base` qty (`1.0`).
     - Collapsible **Section A**, **Section B** (and **Section C** if present) appear above the Summary table, can be expanded/collapsed, and show line items with formulas and section totals.
   - **Direct Tabular Single Group (e.g., `2.1.2.1` or `6.20.11`):**
     - No Section A/B/C; shows `Standard Rate` pill and the Cost Component table (`Ref ID`, `Component Label`, `Formula / Source Refs`, `Factor %`, `Amount`, and `Math Verified Final Rate`).
     - For `6.20.11`, check blank direct-input fields (e.g., `Installation`) accept numeric input.
   - **Direct Tabular Multi-Group (e.g., `8.1.1.1.1` Passenger Lift):**
     - Radio pills allow switching between Rate Groups (e.g., `"Price up to 3 stop"` ↔ `"Next per stop"`).
     - Switching groups updates the Cost Component table and **Math Verified Final Rate** immediately.
3. **Live Recalculation & Validation**
   - Edit an editable **Qty**, **Rate**, **Factor %**, or **Amount** field.
   - Wait ~300ms → verify `"Recalculating…"` briefly appears and all dependent subtotals, section totals, summary components, and **Math Verified Final Rate** update automatically.
   - **VAT Check (`1.1.2.2`):** Change VAT `Factor %` from `8.11` to `11.11` → confirm **Math Verified Final Rate** recalculates from `1,097` to **`1,127`**.
   - **Invalid Input:** Enter a negative number or non-numeric text → confirm inline validation message (*"Enter a non-negative number or clear the field."*) and **Apply Changes** is disabled.
   - **Cancel:** Click **Cancel** or press `Esc` → confirm popover closes and the sheet's `Unit Rate` is unchanged.
4. **Save (`Apply Changes`) & Persistence**
   - Make an edit and click **Apply Changes** → confirm button shows `"Saving…"`, toast `"Rate analysis saved."` appears, and popover closes.
   - Verify the row's **Unit Rate** and **Price** (and sheet summary totals) immediately update to match **Math Verified Final Rate**.
   - Verify the RA trigger icon now has the **edited** highlight state (`(edited)` in title/aria-label, light blue background).
   - Refresh the browser page and reopen the RA popover for that row → confirm custom edits persist!
5. **Switching PWD Item Clears Custom RA**
   - On a row with an edited RA, use the PWD item selector / search to assign a **different** PWD item (or run Quick Optimize overwriting it).
   - Confirm the old custom RA is cleared, `is_rate_analysis_edited` resets, and reopening RA loads the new item's official baseline.

---

## 4. Other Recent Staging Updates (Nov 19 – Nov 20)

Quick awareness items that also landed in the last 5 days:
- **Legend Canvas Copy/Paste (`feature-ui`):** Users can now select single or multiple legend boxes on the Legend Detection canvas, right-click → **Copy**, and paste with `Ctrl+V` / `Cmd+V` (works with Undo/Redo and unsaved draft state). Canvas resize handles & cursor behavior were also refined.
- **Legend Counting Progress via Redis Pub/Sub (`feature-staging`):** Legend counting background tasks now stream progress updates over Redis Pub/Sub instead of polling only. Worth a quick smoke check on Legend Counting live progress bar.
- **PDF/Image Processing Offload to AWS Lambda (`feature-staging`):** PDF/image processing jobs offload to AWS Lambda. Worth a smoke check on **File Upload** & **Review Plans** with a multi-page PDF.
- **Backend-Only APIs Not Yet Wired in Frontend UI (Do Not Log as Missing UI Bugs Unless Requested):**
  - Project Duplication (`POST /v1/projects/{id}/duplicate`)
  - Project Team Member Assignment (`GET/PUT /v1/projects/{id}/team`) & Project Start/Due Dates
  - `+ Custom` Company Discipline creation (`POST /v1/catalog/disciplines`)
  - Company Default Region setting (`PATCH /v1/company/{id}/default-region`)
  - Reset Row Rate Analysis (`DELETE /v1/pwd-estimation/project/{id}/row/{row_id}/rate-analysis`) & Add/Delete custom breakdown lines inside RA
