# McWay Site Management — Module 02: Camp Module
## Detailed Specification for Odoo Implementation

**Version:** 1.0 (initial draft)
**Date:** 2026-04-17
**Module Owner:** Camp In-charge
**Platform:** Web (Odoo)
**Prerequisite:** Module 01 — Admin Portal (Camp Master, Material Master, Site Master, User Management, Lookup Lists)

---

## Table of Contents
1. [Purpose & Scope](#1-purpose--scope)
2. [Odoo Implementation Strategy](#2-odoo-implementation-strategy)
3. [Menu & Navigation Structure](#3-menu--navigation-structure)
4. [Camp Home / Dashboard](#4-camp-home--dashboard)
5. [Sub-Module — Material Inward](#5-sub-module--material-inward)
6. [Sub-Module — Material Outward](#6-sub-module--material-outward)
7. [Sub-Module — Stock Register](#7-sub-module--stock-register)
8. [Sub-Module — RMC Production (Camp-Level)](#8-sub-module--rmc-production-camp-level)
9. [Sub-Module — Labor Arrival & Deployment](#9-sub-module--labor-arrival--deployment)
10. [Sub-Module — Shuttering Tracking](#10-sub-module--shuttering-tracking)
11. [Sub-Module — Verification Queue](#11-sub-module--verification-queue)
12. [State Machine (All Transactional Entries)](#12-state-machine-all-transactional-entries)
13. [Access Control](#13-access-control)
14. [Pre-loaded Demo Data & Walkthrough](#14-pre-loaded-demo-data--walkthrough)
15. [Design Decisions & Assumptions](#15-design-decisions--assumptions)
16. [Open Items / Notes for Developers](#16-open-items--notes-for-developers)

---

## 1. Purpose & Scope

The **Camp Module** is the operational hub for the Camp In-charge. While the Camp Master (Module 01) defines *what* a camp is, this module handles *what happens at the camp every day* — receiving materials, dispatching to sites, producing RMC, tracking stock, logging labor, and managing shuttering.

### In Scope
- Record material **inward** (from plants, vendors, quarries).
- Record material **outward** (to linked sites).
- Auto-maintained **stock register** (opening / inward / outward / production / closing per material+variant).
- **RMC production** at camps with a mixer (input consumption → output).
- **Labor arrival** logging.
- **Shuttering** lifecycle tracking (inward / issue / return / damage / loss).
- **Verification queue** for all camp entries.
- **Camp Home** summary screen for the Camp In-charge.

### Out of Scope
- **Camp Master CRUD** — that is Module 01 (Admin Portal).
- **Diesel** — separate Diesel Module (Module 06). Diesel stock/issue at camp level is handled there, not here.
- **Reports & Dashboard** — separate modules. Camp data feeds into them.
- **Material returns from site back to camp** — not in demo scope. If excess material comes back, it will be handled in a future phase.
- **Inter-camp transfers** — not in demo (only 1 camp). Can be added later as a special outward→inward pair.

---

## 2. Odoo Implementation Strategy

| Entity | Odoo Base | Strategy |
|---|---|---|
| Material Inward | `stock.picking` (incoming) | Picking type = Camp Receipt; auto-updates quants |
| Material Outward | `stock.picking` (outgoing) | Picking type = Camp Dispatch; destination = site virtual location |
| Stock Register | `stock.quant` + custom report | Real-time computed from quants; summary view custom |
| RMC Production | `mrp.production` (simplified) | BoM-driven; consumes inputs, produces output into camp stock |
| Labor Arrival | Custom `mcway.labor.entry` | Lightweight; no Odoo base needed |
| Shuttering Tracking | Custom `mcway.shuttering.entry` | Extends stock concept with issue/return/damage/loss types |
| Verification | `mail.activity` + custom status field | Shared pattern across all transactional models |

### Dependencies (additional to Module 01)
`stock`, `product`, `mrp`, `mail`, `web` — all already dependencies of Module 01.

---

## 3. Menu & Navigation Structure

The Camp In-charge sees a scoped menu. They are **hard-scoped** to their assigned camp (per Module 01 User Management — one user = one camp). All data is auto-filtered.

```
📦 Camp Operations
├── Home (Dashboard)
├── Material Inward
│   ├── New Inward Entry
│   └── Inward History (list)
├── Material Outward
│   ├── New Outward Entry
│   └── Outward History (list)
├── Stock Register
│   ├── Current Stock (summary)
│   └── Stock Ledger (detailed transactions)
├── RMC Production          ← only visible if camp "Has RMC Mixer" = true
│   ├── New Production Entry
│   └── Production History (list)
├── Labor
│   ├── New Labor Entry
│   └── Labor History (list)
├── Shuttering              ← only visible if camp handles shuttering
│   ├── New Shuttering Entry
│   └── Shuttering Register
├── Verification Queue
│   └── Pending Entries (across all sub-modules)
└── Reports (camp-level summaries)
    ├── Daily Camp Summary
    └── Stock Movement Report
```

> **Note:** Camp In-charge does NOT see other camps' data, Admin menus, or diesel operations (those are in the Diesel Module).

---

## 4. Camp Home / Dashboard

**Purpose:** Landing page for the Camp In-charge showing today's operational snapshot.

### 4.1 Summary Tiles

| Tile | Value | Source |
|---|---|---|
| Today's Inward | Count + total MT | Material Inward (today) |
| Today's Outward | Count + total MT | Material Outward (today) |
| Low Stock Alerts | Count of materials below threshold | Stock Register vs Material Master threshold |
| Pending Verification | Count of unverified entries | All camp entries with state = submitted |
| RMC Produced Today | Total m³ | RMC Production (today) — hidden if no mixer |
| Labor on Site Today | Total headcount | Labor entries (today) |
| Shuttering Outstanding | Units issued but not returned | Shuttering Register — hidden if N/A |

### 4.2 Quick Actions
- **+ New Inward** — opens Material Inward form.
- **+ New Outward** — opens Material Outward form.
- **+ New Production** — opens RMC form (if mixer enabled).
- **Review Pending** — opens Verification Queue.

### 4.3 Recent Activity Feed
Last 10 entries across all sub-modules (inward/outward/production/labor/shuttering) in reverse chronological order. Each row: timestamp, type icon, material/description, qty, state badge.

---

## 5. Sub-Module — Material Inward

**Purpose:** Record materials arriving at the camp from any source.
**Odoo Model:** `stock.picking` (type = incoming) + custom fields on `mcway.camp.inward`.

### 5.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Inward Code | Char | Auto | `CIN/YYYY/####` |
| 2 | Camp | Many2one → Camp | Auto | Pre-filled from logged-in user's assignment; read-only |
| 3 | Date & Time | Datetime | ✓ | Defaults to now; **backdating allowed up to 3 days** |
| 4 | Material | Many2one → Material Master | ✓ | Dropdown from active materials |
| 5 | Variant / Grade | Many2one → Product Variant | Conditional | Shown only if material has variants (e.g., Aggregate → 6mm/10mm/20mm). Mandatory if variants exist. |
| 6 | **Source Type** | Selection | ✓ | Plant / Vendor / Quarry |
| 7 | Source — Plant | Many2one → Plant | If Source=Plant | Filtered to plants that supply this camp (from Plant Master "Supplies To") |
| 8 | Source — Vendor | Many2one → Vendor | If Source=Vendor or Quarry | From Vendor Master; quarries are vendors with role tag "Quarry/Material Supplier" |
| 9 | Vehicle Number | Char | ✓ | Free text; uppercase auto-format |
| 10 | Slip / Challan Number | Char | ✓ | — |
| 11 | Royalty Number | Char | ✗ | Government compliance reference (for aggregates, sand) |
| 12 | Gross Weight | Float | ✓ | In primary UoM (usually MT) |
| 13 | Tare Weight | Float | ✓ | In primary UoM |
| 14 | **Net Weight** | Float | **Auto-computed** | = Gross − Tare. Read-only, grey background. **Cannot be negative** (validation). |
| 15 | Secondary Qty | Float | Auto | Only for Cement & Steel (uses conversion factor from Material Master). E.g., 2.5 MT cement = 50 bags. Read-only. |
| 16 | Photo(s) | Binary (multi) | ✗ | Up to 3 photos (slip, vehicle, material). Max 5 MB each, auto-compressed to 1080p. |
| 17 | Remarks | Text | ✗ | — |
| 18 | State | Selection | Auto | Draft → Submitted → Verified → Locked (see §12) |

### 5.2 Source Type Explained

| Source Type | What it means | Source field shown |
|---|---|---|
| **Plant** | Material dispatched from a linked plant | Plant dropdown (filtered to plants supplying this camp) |
| **Vendor** | Material purchased from an external supplier | Vendor dropdown |
| **Quarry** | Material from a quarry (a vendor tagged as quarry) | Vendor dropdown (same field, pre-filtered to quarry-role vendors) |

> **Design Decision:** Quarry is NOT a separate entity. It is a Vendor with the role tag "Quarry/Material Supplier". This avoids a new master and keeps reporting unified under Vendors.

### 5.3 Weighbridge Logic

The camp does not have its own weighbridge in the demo. Gross and Tare weights come from the **source weighbridge slip**. The Camp In-charge enters both values from the physical slip/challan.

- No "pending weighment" state needed.
- If the camp later gets a weighbridge, a "Weigh at Camp" action can be added (phase 2).
- For non-weight materials (pipes — unit = numbers), Gross/Tare fields are hidden and replaced with a single **Quantity** field.

### 5.4 Non-Weight Materials (Pipes, Shuttering)

When the selected material's primary UoM is **not** weight-based (e.g., Pipes in "numbers"):

| Field change | Behavior |
|---|---|
| Gross Weight | Hidden |
| Tare Weight | Hidden |
| Net Weight | Hidden |
| **Quantity** field appears | Float, mandatory, in the material's UoM |
| **UoM label** | Displayed beside Quantity (e.g., "numbers", "pieces") |

### 5.5 Plant Outward ↔ Camp Inward Matching

When source = Plant, the system does **NOT** auto-match plant outward to camp inward in the demo. They are independent entries.

**Reconciliation approach (for reports):**
- Both records share the same Slip/Challan Number and Vehicle Number.
- The Reports Module can cross-reference by slip number to flag mismatches (e.g., plant dispatched 20 MT but camp received 19.5 MT).
- Auto-matching can be added in phase 2.

### 5.6 Business Rules

1. **Net Weight validation:** Net = Gross − Tare. If result ≤ 0, block save with error "Net weight must be positive".
2. **Backdating limit:** Date cannot be more than 3 days in the past. Cannot be in the future.
3. **Duplicate slip warning:** If the same Slip Number already exists for this camp (any date), show a non-blocking warning: "Slip number already used on [date]. Continue?"
4. **Material filter:** Only active materials appear in dropdown.
5. **Variant enforcement:** If the selected material has variants defined in Material Master, the Variant/Grade field becomes mandatory.
6. **On save (Submit):** State moves to `submitted`. Camp stock increases by Net Weight (or Quantity for non-weight items). Entry appears in Verification Queue.
7. **Edit after submit:** Camp In-charge can edit a submitted entry until it is verified. Edits are audited (old value → new value logged). Stock adjusts automatically.
8. **Edit after verified:** Only Admin can unlock a verified entry for correction. Re-enters submitted state after edit.

### 5.7 UX Notes

- **Save & Add Another** button for rapid data entry (vehicles arrive back-to-back).
- After save, show a success banner: "Inward CIN/2026/0015 — Aggregate 20mm — 18.50 MT recorded."
- Vehicle Number field remembers last 10 entries for autocomplete (session-level, not persisted).
- Material + Variant selection can be combined into a single searchable dropdown showing "Aggregate — 20mm", "Cement", "Steel — 12mm", etc.

---

## 6. Sub-Module — Material Outward

**Purpose:** Record materials dispatched from camp to a linked site.
**Odoo Model:** `stock.picking` (type = outgoing) + custom fields on `mcway.camp.outward`.

### 6.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Outward Code | Char | Auto | `COUT/YYYY/####` |
| 2 | Camp | Many2one → Camp | Auto | Pre-filled, read-only |
| 3 | Date & Time | Datetime | ✓ | Defaults to now; backdating allowed up to 3 days |
| 4 | **Destination Site** | Many2one → Site | ✓ | Filtered to sites this camp serves (from Camp Master "Served Sites") |
| 5 | Material | Many2one → Material Master | ✓ | Filtered to materials currently in stock at this camp (stock > 0) |
| 6 | Variant / Grade | Many2one → Product Variant | Conditional | Same logic as inward |
| 7 | Issue Quantity | Float | ✓ | In primary UoM. **Cannot exceed current stock** (see §6.4). |
| 8 | UoM | Display | Auto | From Material Master — read-only label beside quantity |
| 9 | Secondary Qty | Float | Auto | Same as inward — for Cement & Steel |
| 10 | Vehicle Number | Char | ✓ | — |
| 11 | Slip / Challan Number | Char | ✓ | Camp generates its own challan |
| 12 | Receiving Person | Char | ✗ | Name of supervisor or person at site receiving the material |
| 13 | Photo(s) | Binary (multi) | ✗ | Up to 2 photos (loaded vehicle, challan). Max 5 MB each. |
| 14 | Remarks | Text | ✗ | — |
| 15 | State | Selection | Auto | Draft → Submitted → Verified → Locked |

### 6.2 What This Form Does NOT Capture

- **Chainage:** The Camp In-charge dispatches to a *site*, not a chainage. The Supervisor at the site records the exact chainage when the material is dumped (On-Site Dumping Module).
- **Gross/Tare weights:** Not needed for outward. The camp knows how much it's issuing from its stock. Weighing happens at inward.
- **Multiple materials per entry:** Each outward entry is for **one material + one variant**. If a vehicle carries both Aggregate 20mm and Cement, two outward entries are created.

### 6.3 Destination Site Filter

Only sites linked to this camp via Camp Master → "Served Sites" appear in the dropdown. If the camp serves 3 sites, only those 3 appear.

- If the Camp In-charge needs to dispatch to an unlinked site, they must ask Admin to add the linkage in Camp Master first.
- This enforces the hierarchy and prevents data leaks across sites.

### 6.4 Stock Validation

- **Soft warning at 90%:** If the Issue Quantity would bring remaining stock below 10% of current balance, show warning: "After this dispatch, only X MT of [Material] will remain."
- **Hard block at 0:** Issue Quantity cannot exceed current stock balance. Error: "Insufficient stock. Available: X MT."
- **Race condition note (developer):** Use Odoo's `stock.quant` reservation mechanism to prevent two simultaneous outward entries from over-issuing. The picking should reserve the quantity on draft/save, not on submit.

### 6.5 Camp Outward ↔ On-Site Dumping Linkage

Camp outward and site dumping are **independent entries** in the demo. They share:
- Slip/Challan Number (Camp generates it; Supervisor enters it at site)
- Vehicle Number
- Material + Variant

The Reports Module can cross-reference to flag:
- Material dispatched from camp but never recorded as dumped at site.
- Material dumped at site with no matching camp outward (possibly came direct from plant/vendor).

> **Phase 2 enhancement:** Auto-suggest matching when Supervisor enters a slip number that matches an existing camp outward.

### 6.6 Business Rules

1. **One material per entry.** Simpler forms, cleaner stock math, easier audit.
2. **Stock check on save.** Quantity reserved on save; released if entry is deleted/cancelled.
3. **Backdating limit:** Same as inward — up to 3 days.
4. **Duplicate slip warning:** Non-blocking, same as inward.
5. **On save (Submit):** Camp stock decreases by Issue Quantity. Entry appears in Verification Queue.
6. **Edit after submit:** Allowed until verified. Stock auto-adjusts on edit (old qty restored, new qty deducted).

---

## 7. Sub-Module — Stock Register

**Purpose:** Real-time, auto-maintained inventory view. The Camp In-charge **never enters stock manually** — stock emerges entirely from inward, outward, production, and shuttering transactions.
**Odoo Model:** `stock.quant` (standard) + custom summary view.

### 7.1 Stock Summary View (Current Stock)

A list/table showing one row per **material + variant** combination.

| Column | Source | Notes |
|---|---|---|
| Material | Material Master | — |
| Variant / Grade | Product Variant | e.g., "20mm" for Aggregate |
| UoM | Material Master | MT / m³ / numbers / etc. |
| Opening Stock | System | Balance at start of selected date (default: today) |
| Inward | Σ inward qty for the date range | — |
| Production In | Σ RMC output qty (if applicable) | Only for produced materials |
| Outward | Σ outward qty for the date range | — |
| Production Consumed | Σ RMC input consumed | Only for raw materials used in RMC |
| Shuttering Issued | Σ shuttering issues (net of returns) | Only for shuttering materials |
| **Closing Stock** | Opening + Inward + Production In − Outward − Production Consumed − Shuttering Issued | **Bold, primary column** |
| Threshold | From Material Master | Low Stock Threshold value |
| Status | Computed | 🟢 OK / 🟡 Low / 🔴 Critical |

### 7.2 Stock Status Rules

| Condition | Status | Color |
|---|---|---|
| Closing ≥ Threshold × 2 | OK | Green |
| Threshold ≤ Closing < Threshold × 2 | Low | Yellow |
| Closing < Threshold | Critical | Red |
| No threshold set | OK (always) | Green |

### 7.3 Stock Ledger (Detailed Transactions)

Drill down from any row in the summary to see every transaction that affected that material+variant.

| Column | Notes |
|---|---|
| Date & Time | — |
| Transaction Type | Inward / Outward / RMC Input / RMC Output / Shuttering Issue / Shuttering Return / Damage / Loss / Opening Adjustment |
| Reference | Entry code (e.g., CIN/2026/0015) |
| Source / Destination | Plant name / Vendor name / Site name / "RMC Production" |
| Qty In | Positive entries |
| Qty Out | Negative entries |
| Running Balance | Cumulative after this transaction |
| State | Submitted / Verified / Locked |
| Entered By | User name |

### 7.4 Date Range Filter

- Default: Today (shows opening at midnight, all today's transactions, current closing).
- Options: Yesterday / This Week / This Month / Custom Range.
- Opening stock for any date = Closing stock of previous day.

### 7.5 Opening Stock (Go-Live)

On system go-live, existing physical stock at camp needs to be entered. This is done via an **"Opening Stock Adjustment"** action (Admin-only):

- Admin navigates to Stock Register → Action → "Set Opening Stock".
- Form: Material, Variant, Quantity, Date (= go-live date), Reason ("Opening balance at go-live").
- Creates a special inward-type transaction tagged as "Opening Adjustment".
- Audited — appears in stock ledger and audit log.
- Can only be done once per material+variant. After that, stock flows from transactions.

### 7.6 Business Rules

1. **No manual stock edits.** Stock is always computed. Only "Opening Stock Adjustment" (Admin, one-time) is a direct entry.
2. **Negative stock prevention.** Outward and RMC consumption validate against available balance before deducting.
3. **Stock is per material + variant.** Aggregate 6mm and Aggregate 20mm are tracked separately.
4. **Real-time.** Every inward/outward/production save immediately updates the stock view.
5. **Low stock alert.** When a material+variant drops below its threshold, a notification appears on the Camp Home tile and optionally emails Admin (configurable).

---

## 8. Sub-Module — RMC Production (Camp-Level)

**Purpose:** When the camp has its own mixer (Camp Master → "Has RMC Mixer" = true), the Camp In-charge records concrete production batches. Each batch consumes raw inputs and produces RMC output.
**Odoo Model:** `mrp.production` (simplified) via `mcway.camp.production`.

> **Visibility:** This entire sub-module is hidden if the camp does not have an RMC mixer.

### 8.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Production Code | Char | Auto | `CPRD/YYYY/####` |
| 2 | Camp | Many2one → Camp | Auto | Pre-filled, read-only |
| 3 | Date & Time | Datetime | ✓ | Defaults to now; backdating up to 3 days |
| 4 | Shift | Selection | ✓ | Morning / Afternoon / Night |
| 5 | Operator Name | Char | ✓ | Free text (mixer operator, not necessarily a system user) |
| 6 | **Output Grade** | Many2one → Concrete Grade (lookup) | ✓ | Filtered to grades the camp produces (from Camp Master → "Output Grades Produced") |
| 7 | Output Quantity (m³) | Float | ✓ | > 0 |
| 8 | **Input Materials Consumed** | One2many → child table | ✓ | See §8.2 |
| 9 | Diesel Consumed (L) | Float | ✗ | Diesel used by the mixer for this batch. **Informational only** — actual diesel tracking is in the Diesel Module. This is a convenience field for production reporting. |
| 10 | Linked Site | Many2one → Site | ✗ | Which site this RMC is intended for (optional; actual dispatch via outward) |
| 11 | Remarks | Text | ✗ | — |
| 12 | State | Selection | Auto | Draft → Submitted → Verified → Locked |

### 8.2 Input Materials Consumed (Child Table)

One row per raw material consumed in this production batch.

| Field | Type | Mandatory | Notes |
|---|---|---|---|
| Material | Many2one → Material Master | ✓ | Filtered to raw materials: Cement, Sand, Aggregate, Water |
| Variant / Grade | Many2one → Product Variant | Conditional | e.g., Aggregate 20mm |
| Quantity | Float | ✓ | In material's primary UoM |
| UoM | Display | Auto | Read-only |

### 8.3 BoM Auto-Fill (Developer Note)

When the Camp In-charge selects an **Output Grade** (e.g., M25) and enters **Output Quantity** (e.g., 5 m³):

1. System looks up the BoM for that grade (defined in MRP, editable by Admin).
2. **Auto-fills** the Input Materials table with expected quantities: e.g., Cement 1.75 MT, Sand 3.25 MT, Aggregate 20mm 5.0 MT, Water 900 L.
3. Camp In-charge can **adjust** actual quantities if they differ from the BoM (e.g., used slightly more cement).
4. Adjusted values are saved as actuals; the BoM values are retained for variance reporting.

> **Pending from client:** Exact mix ratios per grade (M10–M40). Use placeholder values until received. BoM structure must be editable via Odoo MRP UI without code changes.

### 8.4 Placeholder BoM Values (per m³)

| Grade | Cement (MT) | Sand (MT) | Aggregate 20mm (MT) | Water (L) |
|---|---|---|---|---|
| M20 | 0.32 | 0.65 | 1.00 | 180 |
| M25 | 0.35 | 0.60 | 1.05 | 170 |

> These are approximate. Client to provide actual ratios.

### 8.5 Stock Impact

On submit:
- **Input materials** — stock decreases by consumed quantities. Standard stock validation applies (cannot consume more than available).
- **Output (RMC)** — camp stock increases by Output Quantity (stored as Concrete Mix + the grade variant).

The produced RMC then sits in camp stock until dispatched via a normal **Material Outward** entry to a site.

### 8.6 Two-Step Flow: Produce → Dispatch

```
Step 1: RMC Production (this sub-module)
  Input: Cement 1.75 MT + Sand 3.25 MT + Aggregate 5.0 MT + Water 900 L
  Output: M25 Concrete Mix → 5 m³ added to camp stock

Step 2: Material Outward (§6)
  Material: Concrete Mix — M25
  Qty: 5 m³
  Destination: Kalol Sanand site
  → Stock decreases by 5 m³
```

> **Why two steps?** Production and dispatch often happen at different times. RMC may be produced in the morning and dispatched in batches throughout the day. Two-step keeps stock accurate at every point.

### 8.7 Business Rules

1. **Only visible if camp has RMC mixer.**
2. **Input stock validation.** Cannot consume more than available stock of any input material.
3. **Output grade filter.** Only grades configured in Camp Master appear.
4. **BoM auto-fill is a suggestion, not a constraint.** Actuals can differ from BoM (practical reality — mix ratios vary by batch).
5. **Diesel consumed field is informational.** It does NOT deduct from any diesel stock. Actual diesel deduction happens in the Diesel Module.
6. **Backdating limit:** Up to 3 days.

---

## 9. Sub-Module — Labor Arrival & Deployment

**Purpose:** Log daily labor (contract workers) arriving at the camp for deployment to sites.
**Odoo Model:** Custom `mcway.labor.entry`.

### 9.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Entry Code | Char | Auto | `LAB/YYYY/####` |
| 2 | Camp | Many2one → Camp | Auto | Pre-filled, read-only |
| 3 | Date | Date | ✓ | Defaults to today; backdating up to 3 days |
| 4 | Party Name (Mukadam) | Char | ✓ | **Free text** — labor contractor name. Autocomplete from previous entries at this camp. |
| 5 | Number of Workers | Integer | ✓ | > 0 |
| 6 | Deployed To | Selection | ✓ | Camp / Site |
| 7 | Deployed To — Site | Many2one → Site | If Deployed To = Site | Filtered to served sites |
| 8 | Purpose / Activity | Many2one → Layer/Activity (lookup) | ✗ | e.g., Paving, Concrete, Excavation, Shuttering, General |
| 9 | Remarks | Text | ✗ | — |
| 10 | State | Selection | Auto | Draft → Submitted → Verified → Locked |

### 9.2 Design Decisions

- **Party Name is free text**, not a master. For the demo, labor contractors change frequently and a full master would be overhead. Autocomplete from history reduces re-typing.
- **Labor is logged at camp level.** The camp is the gathering point — workers arrive at camp and are deployed to camp work or to a site. The entry captures where they were deployed.
- **No cost/wage tracking** in demo. Just headcount per party per day per purpose.
- **Multiple entries per day allowed.** Different parties or different purposes = separate entries.

### 9.3 Business Rules

1. Number of Workers must be > 0.
2. One entry per party per purpose per deployment location per day (soft — warning if duplicate, not blocking).
3. Backdating limit: 3 days.
4. Entries feed into Reports Module → Daily Labor Report and Dashboard → "Labor on field today" tile.

---

## 10. Sub-Module — Shuttering Tracking

**Purpose:** Track the lifecycle of shuttering materials (formwork panels, props, etc.) which are reusable assets issued to sites and expected back. Unlike consumable materials, shuttering has a circular flow: inward → issue → return, with damage/loss tracking.
**Odoo Model:** Custom `mcway.shuttering.entry`.

### 10.1 Transaction Types

| Type | Direction | Meaning |
|---|---|---|
| **Inward** | + Stock | New shuttering received at camp (from vendor or another camp) |
| **Issue to Site** | − Stock | Shuttering sent to a site for use |
| **Return from Site** | + Stock | Shuttering returned after use |
| **Damage** | − Stock | Shuttering damaged beyond use (write-off) |
| **Loss** | − Stock | Shuttering lost / unaccounted |

### 10.2 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Entry Code | Char | Auto | `SHT/YYYY/####` |
| 2 | Camp | Many2one → Camp | Auto | Pre-filled, read-only |
| 3 | Date | Date | ✓ | Defaults to today; backdating up to 3 days |
| 4 | **Transaction Type** | Selection | ✓ | Inward / Issue to Site / Return from Site / Damage / Loss |
| 5 | Shuttering Type | Many2one → Material (category=Shuttering) | ✓ | From Material Master; e.g., "Steel Plate 1.2m×0.6m", "Prop 3m" |
| 6 | Quantity | Integer | ✓ | Number of pieces/units |
| 7 | Site | Many2one → Site | Conditional | Mandatory for Issue / Return. Hidden for Inward / Damage / Loss. Filtered to served sites. |
| 8 | Source Vendor | Many2one → Vendor | Conditional | Shown only for Inward (who supplied it) |
| 9 | Condition on Return | Selection | If Return | Good / Minor Damage / Major Damage |
| 10 | Slip / Reference | Char | ✗ | — |
| 11 | Photo | Binary | ✗ | Especially useful for Damage entries |
| 12 | Remarks | Text | ✗ | Required for Damage and Loss (must explain) |
| 13 | State | Selection | Auto | Draft → Submitted → Verified → Locked |

### 10.3 Shuttering Register (Summary View)

| Column | Notes |
|---|---|
| Shuttering Type | e.g., "Steel Plate 1.2m×0.6m" |
| Total Received | Σ Inward qty (all time) |
| Currently at Camp | Available stock |
| Issued to Sites | Σ Issue − Σ Return (net outstanding per site) |
| Damaged | Σ Damage |
| Lost | Σ Loss |
| **Total Accounted** | At Camp + Issued + Damaged + Lost (should = Total Received) |
| Discrepancy | Total Received − Total Accounted (should be 0) |

Drill down any row to see full transaction history for that shuttering type.

### 10.4 Business Rules

1. **Issue quantity cannot exceed available stock at camp.**
2. **Return quantity cannot exceed quantity issued to that specific site.**
3. **Damage and Loss require remarks** (explain what happened).
4. **Discrepancy > 0 is flagged** in the summary as a data integrity issue.
5. Shuttering materials are defined in Material Master with Category = "Shuttering". No separate master.

---

## 11. Sub-Module — Verification Queue

**Purpose:** Single screen where Verifiers (Senior Engineer or Admin) review and verify entries submitted by the Camp In-charge.
**Shared pattern** — same UX across all modules (Camp, Plant, On-Site Dumping, etc.).

### 11.1 Queue View

A filterable list of all **submitted but unverified** entries for this camp.

| Column | Notes |
|---|---|
| Entry Code | Clickable → opens entry detail |
| Type | Inward / Outward / RMC Production / Labor / Shuttering |
| Date | — |
| Material / Description | Material name + variant, or labor party, or shuttering type |
| Quantity | With UoM |
| Entered By | User name |
| Submitted At | Timestamp |
| **Age** | Time since submission (e.g., "2h ago", "1d 4h") |
| Action | Verify / Request Correction |

### 11.2 Verification Actions

| Action | Effect |
|---|---|
| **Verify** | State → Verified. Timestamp + verifier name recorded. Entry removed from queue. |
| **Request Correction** | Remark sent to Camp In-charge. Entry stays in `submitted` state. A flag/badge appears on the entry in the Camp In-charge's view. Camp In-charge edits and re-submits. |

### 11.3 Bulk Verify

Select multiple entries → "Verify Selected" button. Useful for end-of-day review when many entries are pending.

### 11.4 Filters & Sorting

- Filter by: Type, Date Range, Entered By, Age (> 24h, > 48h).
- Sort by: Date (default newest first), Age (oldest first — for catching stale entries).
- **Highlight rule:** Entries older than the system threshold (default 2 days, from Module 01 System Thresholds) are highlighted in orange.

### 11.5 Verification Rules (Recap)

1. **Verification never blocks.** Submitted entries are immediately usable in stock, reports, and dashboard.
2. **Unverified entries are flagged** in Reports and Dashboard with a visual indicator.
3. **Verifier cannot edit the entry** — only verify or request correction.
4. **Audit trail:** Who verified, when, from which IP. Logged via `mail.thread` + `auditlog`.

---

## 12. State Machine (All Transactional Entries)

All transactional records in the Camp Module follow the same state progression:

```
┌─────────┐     Save/Submit     ┌───────────┐     Verify     ┌──────────┐     Period Close     ┌────────┐
│  Draft   │ ─────────────────→ │ Submitted │ ─────────────→ │ Verified │ ──────────────────→ │ Locked │
└─────────┘                     └───────────┘                 └──────────┘                     └────────┘
                                      │                             │
                                      │ Edit (by Camp In-charge)    │ Unlock (Admin only)
                                      ↓                             ↓
                                 Back to Submitted            Back to Submitted
```

### 12.1 State Definitions

| State | Who can edit | Stock impact | Visible in reports | In verification queue |
|---|---|---|---|---|
| **Draft** | Camp In-charge | No | No | No |
| **Submitted** | Camp In-charge | Yes | Yes (flagged as unverified) | Yes |
| **Verified** | Admin only (unlock required) | Yes | Yes (clean) | No |
| **Locked** | Nobody | Yes | Yes | No |

### 12.2 Notes

- **Draft is ephemeral.** In practice, the Camp In-charge fills the form and clicks "Save & Submit" which goes directly to Submitted. Draft exists only for partially-filled forms that are saved but not yet ready.
- **No "Rejected" state.** A correction request keeps the entry in Submitted. The Camp In-charge edits and it stays Submitted until verified.
- **Locked** is set by Admin during period close (e.g., month-end lock). Prevents retroactive edits.
- **Stock impact starts at Submitted.** This is intentional — entries are operationally real the moment they're submitted. Verification is an accountability layer, not a gating mechanism.

---

## 13. Access Control

### 13.1 Model-Level Access

| Model | Camp In-charge | Admin | Owner | SE | Verifier | Others |
|---|---|---|---|---|---|---|
| Camp Inward | CRUD (own camp) | CRUD | R | R | R | — |
| Camp Outward | CRUD (own camp) | CRUD | R | R | R | — |
| Stock Register | R (own camp) | R | R | R | R | — |
| RMC Production | CRUD (own camp) | CRUD | R | R | R | — |
| Labor Entry | CRUD (own camp) | CRUD | R | R | R | — |
| Shuttering Entry | CRUD (own camp) | CRUD | R | R | R | — |
| Verification | — | Verify | — | Verify (own sites' camps) | Verify | — |

### 13.2 Record Rules (Scope Enforcement)

- **Camp In-charge** sees only data for their assigned camp. They cannot access any other camp's records.
- **Senior Engineer** can verify entries for camps that serve their assigned site.
- **Admin** sees all camps.
- **Owner** sees all camps (read-only).
- Enforced via Odoo `ir.rule` based on `user.assignment_target` matching the camp or the camp's served sites.

---

## 14. Pre-loaded Demo Data & Walkthrough

### 14.1 Demo Camp
**Khatraj Chokdi** (CAMP/2026/001) — serves Kalol Sanand, has RMC mixer (M20, M25).

### 14.2 Pre-loaded Material Inward Entries (5 entries)

| # | Date | Material | Variant | Source | Vehicle | Slip | Gross | Tare | Net |
|---|---|---|---|---|---|---|---|---|---|
| 1 | 2026-04-15 | Aggregate | 20mm | Vendor: Maruti Hirers | GJ-01-XX-1234 | SLP-0401 | 25.0 | 7.0 | 18.0 MT |
| 2 | 2026-04-15 | Cement | — | Vendor: Maruti Hirers | GJ-01-XX-5678 | SLP-0402 | 32.0 | 12.0 | 20.0 MT |
| 3 | 2026-04-15 | Sand | — | Quarry: Shree Construction | GJ-01-YY-9012 | SLP-0403 | 28.5 | 8.5 | 20.0 MT |
| 4 | 2026-04-16 | Aggregate | 6mm | Plant: Magodi | GJ-01-ZZ-3456 | PLT-0088 | 22.0 | 7.0 | 15.0 MT |
| 5 | 2026-04-16 | Steel | 12mm | Vendor: Maruti Hirers | GJ-01-XX-7890 | SLP-0404 | 15.0 | 5.0 | 10.0 MT |

### 14.3 Pre-loaded Material Outward Entries (3 entries)

| # | Date | Material | Variant | Destination | Qty | Vehicle | Slip |
|---|---|---|---|---|---|---|---|
| 1 | 2026-04-16 | Aggregate | 20mm | Kalol Sanand | 10.0 MT | GJ-01-XX-1234 | CHN-0201 |
| 2 | 2026-04-16 | Cement | — | Kalol Sanand | 5.0 MT | GJ-01-XX-5678 | CHN-0202 |
| 3 | 2026-04-16 | Steel | 12mm | Kalol Sanand | 3.0 MT | GJ-01-XX-7890 | CHN-0203 |

### 14.4 Expected Stock After Demo Data

| Material | Variant | Inward | Outward | Closing |
|---|---|---|---|---|
| Aggregate | 20mm | 18.0 | 10.0 | 8.0 MT |
| Aggregate | 6mm | 15.0 | 0 | 15.0 MT |
| Cement | — | 20.0 | 5.0 | 15.0 MT |
| Sand | — | 20.0 | 0 | 20.0 MT |
| Steel | 12mm | 10.0 | 3.0 | 7.0 MT |

### 14.5 Pre-loaded RMC Production (1 entry)

| Date | Grade | Output | Cement Used | Sand Used | Aggregate 20mm Used | Water |
|---|---|---|---|---|---|---|
| 2026-04-16 | M25 | 2.0 m³ | 0.70 MT | 1.20 MT | 2.10 MT | 340 L |

Updated stock after production:
- Cement: 15.0 − 0.70 = **14.30 MT**
- Sand: 20.0 − 1.20 = **18.80 MT**
- Aggregate 20mm: 8.0 − 2.10 = **5.90 MT**
- Concrete Mix M25: 0 + 2.0 = **2.0 m³** (new)

### 14.6 Pre-loaded Labor Entry (1 entry)

| Date | Party | Workers | Deployed To | Purpose |
|---|---|---|---|---|
| 2026-04-16 | Ramesh Mukadam | 12 | Kalol Sanand | Concrete work |

### 14.7 Demo Walkthrough (Suggested)

1. **Log in as Camp In-charge (Suresh)** → lands on Camp Home showing today's tiles.
2. **Record an inward** — Aggregate 10mm from Magodi Plant, 16 MT. Show gross/tare/net auto-calc.
3. **Record an outward** — Aggregate 20mm, 5 MT to Kalol Sanand. Show stock validation.
4. **View Stock Register** — show real-time balances, drill into ledger for Aggregate 20mm.
5. **Record RMC Production** — M20, 3 m³. Show BoM auto-fill, adjust cement slightly. Show stock impact.
6. **Log labor** — New party, 8 workers, deployed to camp for shuttering work.
7. **Switch to Senior Engineer (Kiran)** → open Verification Queue → bulk-verify today's entries.
8. **Show low-stock alert** — set Aggregate 20mm threshold to 10 MT (currently 5.90 MT after production), refresh → alert appears.

---

## 15. Design Decisions & Assumptions

These decisions were made based on available documentation and practical demo needs. Flag to client if any need revision.

| # | Decision | Rationale |
|---|---|---|
| 1 | **Quarry = Vendor with role tag**, not a separate entity | No new master needed; quarries are just material suppliers |
| 2 | **Outward captures site only, not chainage** | Chainage is the Supervisor's domain (On-Site Dumping Module) |
| 3 | **One material per outward entry** | Simpler form, cleaner stock math, easier audit trail |
| 4 | **No material returns in demo** | Matches on-site dumping scope decision; add in phase 2 |
| 5 | **Stock per material + variant** | Aggregate 6mm and 20mm are operationally different; must be tracked separately |
| 6 | **RMC production → camp stock → outward (two-step)** | Production and dispatch timing differ; two-step keeps stock accurate |
| 7 | **Labor party name is free text with autocomplete** | Full labor contractor master is overhead for demo; autocomplete reduces re-typing |
| 8 | **Labor logged at camp level with deployment destination** | Camp is the gathering point; deployment can be to camp or site |
| 9 | **Camp In-charge hard-scoped to assigned camp** | Per Module 01 user assignment rules (one user = one scope) |
| 10 | **Backdating allowed up to 3 days** | Practical for missed entries (e.g., vehicle arrived late evening, logged next morning) |
| 11 | **Plant outward ↔ camp inward NOT auto-matched** | Independent entries; reconciliation via reports by slip number. Auto-match in phase 2. |
| 12 | **Weighbridge weights come from source slip** | Camp does not have its own weighbridge in demo |
| 13 | **Diesel consumed by RMC mixer is informational only** | Actual diesel tracking lives in Diesel Module; avoids double deduction |
| 14 | **Shuttering = standard stock with additional transaction types** | No new inventory model; just issue/return/damage/loss on top of inward/outward |
| 15 | **No inter-camp transfers in demo** | Only 1 camp in demo scope; can add later as outward→inward pair |

---

## 16. Open Items / Notes for Developers

1. **RMC mix ratios** — pending from client. Use placeholder BoMs (§8.4). Keep editable via Odoo MRP UI.
2. **Diesel consumed by mixer** — confirm with Diesel Module spec whether this field should eventually trigger a diesel deduction or remain purely informational.
3. **Photo compression** — implement server-side compression to 1080p for all uploaded photos. Cap at 5 MB per file, 3 files per entry.
4. **Concurrent outward race condition** — use `stock.quant` reservation on save to prevent over-issue when two outward entries are created simultaneously.
5. **Stock Register performance** — for large camps with many materials, the summary view should use SQL aggregation, not Python loops. Materialized view recommended if > 10k transactions.
6. **Shuttering types** — confirm with client which shuttering items to pre-load. Placeholder: "Steel Plate 1.2m×0.6m", "Prop 3m", "H-Frame", "Plywood Sheet".
7. **Backdating limit (3 days)** — make this configurable in System Thresholds (Module 01) so it can be adjusted per client preference.
8. **Secondary UoM display** — for Cement (MT → bags) and Steel (MT → pieces), show the secondary quantity as a read-only helper. Conversion factor comes from Material Master.
9. **Vehicle number autocomplete** — session-level cache of last 10 entries. Consider promoting to a Vehicle Master in phase 2 if the client tracks vehicles.
10. **Opening Stock Adjustment** — one-time, Admin-only. Consider a bulk-upload option (CSV) for go-live with many materials.
11. **Period locking** — the "Locked" state is set by Admin during month-end close. Implement as a date-range lock: all entries within the locked period cannot be edited. Exact mechanism to be defined in a cross-cutting "Period Management" spec.
12. **Notification on correction request** — when Verifier clicks "Request Correction", the Camp In-charge should see a badge/notification on the entry. In-app notification for demo; email notification optional.
