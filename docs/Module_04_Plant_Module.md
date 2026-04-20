# McWay Site Management — Module 04: Plant Module
## Detailed Specification for Odoo Implementation

**Version:** 1.0 (initial draft)
**Date:** 2026-04-20
**Module Owner:** Plant In-charge
**Platform:** Web (Odoo)
**Prerequisite:** Module 01 — Admin Portal (Plant Master, Material Master, Site Master, Camp Master, User Management, Lookup Lists, BoMs)

---

## Table of Contents
1. [Purpose & Scope](#1-purpose--scope)
2. [Odoo Implementation Strategy](#2-odoo-implementation-strategy)
3. [Menu & Navigation Structure](#3-menu--navigation-structure)
4. [Plant Home / Dashboard](#4-plant-home--dashboard)
5. [Sub-Module — Raw Material Inward](#5-sub-module--raw-material-inward)
6. [Sub-Module — Production (Capability-Driven)](#6-sub-module--production-capability-driven)
7. [Sub-Module — Plant Outward](#7-sub-module--plant-outward)
8. [Sub-Module — Stock Register (Raw + Finished)](#8-sub-module--stock-register-raw--finished)
9. [Sub-Module — Scrap & Wastage](#9-sub-module--scrap--wastage)
10. [Sub-Module — Labor at Plant](#10-sub-module--labor-at-plant)
11. [Sub-Module — Verification Queue](#11-sub-module--verification-queue)
12. [State Machine (All Transactional Entries)](#12-state-machine-all-transactional-entries)
13. [Access Control](#13-access-control)
14. [Pre-loaded Demo Data & Walkthrough](#14-pre-loaded-demo-data--walkthrough)
15. [Design Decisions & Assumptions](#15-design-decisions--assumptions)
16. [Open Items / Notes for Developers](#16-open-items--notes-for-developers)

---

## 1. Purpose & Scope

The **Plant Module** is the operational hub for the Plant In-charge. Plants are large-scale production and supply bases. While the Plant Master (Module 01) defines *what* a plant is, this module handles *what happens at the plant every day* — receiving raw materials, producing bituminous mixes / RMC / emulsion / steel, dispatching finished goods to camps and sites, and tracking stock (both raw and finished).

A plant differs from a camp in three ways:
1. **Larger, multi-capability production** — a plant may produce bituminous, concrete, emulsion, and steel all in one location.
2. **Two-layer inventory** — raw materials and finished products are tracked separately (distinct stock locations).
3. **Dispatch to both camps AND sites** — camps only dispatch to sites; plants can dispatch to either.

### In Scope
- Record **Raw Material Inward** (aggregate, bitumen, cement, sand, steel rods, emulsion raw, etc.) from vendors and quarries.
- Record **Production batches** per capability — each consumes raw inputs (via BoM) and produces finished goods into plant stock.
- Record **Plant Outward** — dispatch of finished goods to camps or sites.
- Auto-maintained **Stock Register** with separate Raw and Finished views.
- **Scrap & Wastage** entries for production shortfalls, rejected batches, spillage.
- **Labor at Plant** — headcount logging for plant operations.
- **Verification queue** for all plant entries.
- **Plant Home** summary screen.

### Out of Scope
- **Plant Master CRUD** — Module 01 (Admin Portal).
- **BoM definition** — Odoo MRP UI (Admin-managed). This module consumes BoMs but does not edit them.
- **Diesel** — Module 06. Diesel consumed by plant equipment is tracked there, not here.
- **Reports & Dashboard** — separate modules.
- **Material returns** — finished goods once dispatched do not come back in demo scope.
- **Multi-plant transfers** — only 1 plant in demo scope.
- **Machine productivity** — machines at the plant are assigned via Machine Assignment (Module 01), but per-batch machine tracking is Phase 2.

---

## 2. Odoo Implementation Strategy

| Entity | Odoo Base | Strategy |
|---|---|---|
| Raw Material Inward | `stock.picking` (incoming) | Picking type = Plant Raw Receipt; auto-updates raw stock |
| Production | `mrp.production` | BoM-driven; consumes raw inputs, produces finished goods into finished stock |
| Plant Outward | `stock.picking` (outgoing) | Picking type = Plant Dispatch; destination = camp OR site virtual location |
| Stock Register | `stock.quant` + custom report | Two tabs (Raw / Finished); real-time computed |
| Scrap & Wastage | `stock.scrap` (standard) + custom metadata | Standard Odoo scrap with plant-specific fields |
| Labor at Plant | Custom `mcway.labor.entry` (shared with Camp Module) | Same model; scope = plant |
| Verification | `mail.activity` + custom status field | Shared pattern across all modules |

### Dependencies (additional to Module 01)
`stock`, `product`, `mrp`, `mail`, `web` — all already dependencies of Module 01.

### Capability-Driven Visibility
Plant capabilities (set in Plant Master) drive which production screens are visible:
- **Bituminous** capability → Bituminous Production menu item.
- **Concrete (RMC)** → Concrete Production menu item.
- **Emulsion** → Emulsion Production menu item.
- **Steel** → Steel Cutting/Bending menu item.

A plant without a given capability does not see that menu item and cannot create entries for it.

---

## 3. Menu & Navigation Structure

The Plant In-charge sees a scoped menu. They are **hard-scoped** to their assigned plant (per Module 01 User Management — one user = one plant). All data is auto-filtered.

```
🏭 Plant Operations
├── Home (Dashboard)
├── Raw Material Inward
│   ├── New Inward Entry
│   └── Inward History (list)
├── Production
│   ├── Bituminous Mix          ← only if capability enabled
│   │   ├── New Batch
│   │   └── Batch History
│   ├── Concrete (RMC)          ← only if capability enabled
│   │   ├── New Batch
│   │   └── Batch History
│   ├── Emulsion                ← only if capability enabled
│   │   ├── New Batch
│   │   └── Batch History
│   └── Steel Cutting/Bending   ← only if capability enabled
│       ├── New Entry
│       └── History
├── Plant Outward
│   ├── New Dispatch
│   └── Dispatch History
├── Stock Register
│   ├── Raw Materials (summary)
│   ├── Finished Goods (summary)
│   └── Stock Ledger (all transactions)
├── Scrap & Wastage
│   ├── New Scrap Entry
│   └── Scrap History
├── Labor
│   ├── New Labor Entry
│   └── Labor History
├── Verification Queue
│   └── Pending Entries (across all sub-modules)
└── Reports (plant-level summaries)
    ├── Daily Plant Summary
    ├── Production Variance (BoM vs actual)
    └── Dispatch Reconciliation
```

> **Note:** Plant In-charge does NOT see other plants' data, Admin menus, or Camp/Site operations.

---

## 4. Plant Home / Dashboard

**Purpose:** Landing page for the Plant In-charge showing today's operational snapshot.

### 4.1 Summary Tiles

| Tile | Value | Source |
|---|---|---|
| Today's Raw Inward | Count + total MT | Raw Material Inward (today) |
| Today's Production | Count of batches + output totals by capability | Production (today) |
| Today's Outward | Count + total dispatched | Plant Outward (today) |
| Low Stock — Raw | Count of raw materials below threshold | Raw stock vs Material Master threshold |
| Low Stock — Finished | Count of finished goods below threshold | Finished stock vs Material Master threshold |
| Pending Verification | Count of unverified entries | All plant entries with state = submitted |
| Capacity Utilization | Today's output vs Daily Capacity per capability | % bar per capability |
| Labor at Plant Today | Total headcount | Labor entries (today) |

### 4.2 Quick Actions
- **+ New Raw Inward** — opens Raw Material Inward form.
- **+ New Production** — dropdown to select capability → opens relevant batch form.
- **+ New Outward** — opens Plant Outward form.
- **Review Pending** — opens Verification Queue.

### 4.3 Capacity Utilization Strip
For each capability configured on this plant, show a horizontal bar:
```
Bituminous Mix    [███████░░░] 350 / 500 MT (70%)
Concrete (RMC)    [████░░░░░░] 120 / 300 m³  (40%)
Emulsion          [██░░░░░░░░]  2 / 10 MT   (20%)
```
Click to drill into that capability's production history.

### 4.4 Recent Activity Feed
Last 10 entries across all sub-modules (inward / production / outward / scrap / labor) in reverse chronological order. Each row: timestamp, type icon, description, qty, state badge.

---

## 5. Sub-Module — Raw Material Inward

**Purpose:** Record raw materials arriving at the plant from vendors or quarries for use in production.
**Odoo Model:** `stock.picking` (type = incoming) + custom fields on `mcway.plant.inward`.

### 5.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Inward Code | Char | Auto | `PIN/YYYY/####` |
| 2 | Plant | Many2one → Plant | Auto | Pre-filled from logged-in user's assignment; read-only |
| 3 | Date & Time | Datetime | ✓ | Defaults to now; **backdating allowed up to 3 days** |
| 4 | Material | Many2one → Material Master | ✓ | Filtered to **raw materials** only (Category = Raw; excludes finished-goods categories) |
| 5 | Variant / Grade | Many2one → Product Variant | Conditional | Shown if material has variants. Mandatory if variants exist. |
| 6 | **Source Type** | Selection | ✓ | Vendor / Quarry |
| 7 | Source — Vendor/Quarry | Many2one → Vendor | ✓ | From Vendor Master; quarries filtered by role tag "Quarry/Material Supplier" |
| 8 | Vehicle Number | Char | ✓ | Free text; uppercase auto-format |
| 9 | Slip / Challan Number | Char | ✓ | — |
| 10 | Royalty Number | Char | ✗ | For aggregates, sand |
| 11 | Gross Weight | Float | ✓ | Primary UoM (usually MT) |
| 12 | Tare Weight | Float | ✓ | Primary UoM |
| 13 | **Net Weight** | Float | **Auto-computed** | = Gross − Tare. Read-only. Cannot be negative. |
| 14 | Secondary Qty | Float | Auto | For Cement & Steel (conversion from Material Master). Read-only. |
| 15 | **Stored At** | Selection | Auto | Raw stock location — set automatically based on material (e.g., Bitumen → Bitumen tank; Aggregate → Raw yard). Admin-configurable mapping. |
| 16 | Photo(s) | Binary (multi) | ✗ | Up to 3 photos. Max 5 MB each, auto-compressed to 1080p. |
| 17 | Remarks | Text | ✗ | — |
| 18 | State | Selection | Auto | Draft → Submitted → Verified → Locked (see §12) |

### 5.2 Source Type Explained

| Source Type | What it means |
|---|---|
| **Vendor** | Raw material purchased from an external supplier (e.g., Bitumen from IOCL, Cement from Ambuja) |
| **Quarry** | Raw material from a quarry (a vendor tagged as quarry) — typically aggregate, sand |

> **Design Decision:** No "Plant" or "Camp" source type here. Plants receive raw from external sources only. Inter-plant or camp-to-plant transfer is out of scope for demo.

### 5.3 Weighbridge Logic
The plant **has its own weighbridge**. Gross and Tare are captured as vehicles enter and leave. For the demo, both values are entered manually from the physical slip/weighbridge printout.

- For non-weight raw materials (e.g., Steel rods by count, Emulsion drums), Gross/Tare hide and a single **Quantity** field appears (same pattern as Camp Inward §5.4).

### 5.4 Non-Weight Raw Materials
When primary UoM is not weight-based:

| Field change | Behavior |
|---|---|
| Gross Weight | Hidden |
| Tare Weight | Hidden |
| Net Weight | Hidden |
| **Quantity** field appears | Float, mandatory, in material's UoM |
| **UoM label** | Displayed beside Quantity |

### 5.5 Business Rules

1. **Net Weight validation:** Net = Gross − Tare. If ≤ 0, block save with error.
2. **Backdating limit:** Up to 3 days past; no future dates.
3. **Duplicate slip warning:** Non-blocking warning if slip number already exists for this plant.
4. **Material filter:** Only raw materials appear. Finished goods (BM, RMC, Emulsion grades) do NOT appear here — those come from production, not inward.
5. **Variant enforcement:** Mandatory if material has variants (e.g., Aggregate 6mm / 10mm / 20mm / 40mm).
6. **On save (Submit):** State → Submitted. Raw stock increases. Entry appears in Verification Queue.
7. **Edit after submit:** Plant In-charge can edit until verified. Edits audited. Stock auto-adjusts.
8. **Edit after verified:** Only Admin can unlock.

### 5.6 UX Notes
- **Save & Add Another** button for rapid entry (multiple vehicles arriving).
- Success banner: "Inward PIN/2026/0032 — Bitumen — 18.00 MT recorded."
- Vehicle Number autocomplete from last 10 entries.
- Material + Variant can merge into one searchable dropdown.

---

## 6. Sub-Module — Production (Capability-Driven)

**Purpose:** Record production batches. Each batch consumes raw inputs (per BoM) and produces finished goods.
**Odoo Model:** `mrp.production` (standard Odoo MRP) extended via `mcway.plant.production`.

Production is split into **four sub-screens**, one per capability. Visibility is driven by the plant's configured capabilities (Plant Master §7).

### 6.1 Shared Fields (All Production Types)

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Batch Code | Char | Auto | `PPRD/<capability>/YYYY/####`; e.g., `PPRD/BIT/2026/0021` |
| 2 | Plant | Many2one → Plant | Auto | Pre-filled |
| 3 | Date & Time | Datetime | ✓ | Defaults to now; backdating up to 3 days |
| 4 | Shift | Selection | ✓ | Morning / Afternoon / Night |
| 5 | Operator / Batching-In-charge | Char | ✓ | Free text; plant operator name |
| 6 | **Output Grade** | Many2one → Product (filtered by capability) | ✓ | Grades enabled for this capability (from BoM library) |
| 7 | Output Quantity | Float | ✓ | > 0, in the grade's UoM |
| 8 | **Input Materials Consumed** | One2many → child table | ✓ | See §6.3 |
| 9 | Diesel Consumed (L) | Float | ✗ | **Informational only** — actual diesel lives in Diesel Module |
| 10 | Target Destination | Many2one (polymorphic: Camp or Site) | ✗ | Optional — which camp/site this batch is intended for (doesn't dispatch; just a hint) |
| 11 | Remarks | Text | ✗ | — |
| 12 | State | Selection | Auto | Draft → Submitted → Verified → Locked |

### 6.2 Capability-Specific Fields

| Capability | Additional Fields |
|---|---|
| **Bituminous** | **Mix Type** (BM / DBM / BC / SDBC / MSS) → filters Output Grade. **Temperature (°C)** (optional, for QA). |
| **Concrete (RMC)** | **Concrete Grade** (M10 / M15 / M20 / M25 / M30 / M35 / M40) → filters Output Grade. **Slump (mm)** (optional). |
| **Emulsion** | **Emulsion Type** (SS1 / SS2 / RS1 / RS2 / MS) → filters Output Grade. |
| **Steel (Cutting/Bending)** | **Input Steel Variant** (e.g., 8mm / 10mm / 12mm / 16mm / 20mm / 25mm rod). **Cut Length / Bend Shape** (free text). No BoM auto-fill — manual input/output. |

### 6.3 Input Materials Consumed (Child Table)

One row per raw material consumed in the batch.

| Field | Type | Mandatory | Notes |
|---|---|---|---|
| Material | Many2one → Material Master | ✓ | Filtered to raw materials appropriate for the selected output grade (via BoM) |
| Variant / Grade | Many2one → Product Variant | Conditional | e.g., Aggregate 20mm |
| BoM Quantity | Float | Auto | Expected quantity per BoM (read-only) |
| **Actual Quantity** | Float | ✓ | In material's UoM. **Editable** — plant operators adjust based on actual consumption |
| Variance | Float | Auto | = Actual − BoM Qty. Coloured green (±5%), amber (±10%), red (> 10%) |
| UoM | Display | Auto | Read-only |

### 6.4 BoM Auto-Fill

When the Plant In-charge selects an **Output Grade** and enters **Output Quantity**:

1. System looks up the BoM for that grade (defined in Odoo MRP, editable by Admin).
2. **Auto-fills** the Input Materials table with expected quantities, scaled by output quantity.
3. Plant In-charge reviews and **adjusts Actual Quantity** if consumption differed from BoM.
4. BoM Quantity is preserved alongside Actual for variance reporting.

**Example (Bituminous BM, 10 MT output):**

| Material | BoM Qty | Actual Qty | Variance |
|---|---|---|---|
| Aggregate 20mm | 4.5 MT | 4.55 MT | +1.1% (green) |
| Aggregate 10mm | 3.0 MT | 3.05 MT | +1.7% (green) |
| Sand (fine) | 2.0 MT | 2.00 MT | 0% (green) |
| Bitumen | 0.5 MT | 0.55 MT | +10% (amber) |

> **Pending from client:** Exact mix ratios per grade for BM, DBM, BC, SDBC, MSS (bituminous); M10–M40 (concrete); SS1, RS1 etc. (emulsion). Use placeholder BoMs until received. BoMs must be editable via Odoo MRP UI without code changes.

### 6.5 Placeholder BoM Values

**Bituminous (per MT of output):**
| Grade | Aggregate 20mm | Aggregate 10mm | Sand | Bitumen |
|---|---|---|---|---|
| BM | 0.45 MT | 0.30 MT | 0.20 MT | 0.05 MT |
| DBM | 0.40 MT | 0.35 MT | 0.20 MT | 0.05 MT |
| BC | 0.35 MT | 0.40 MT | 0.18 MT | 0.06 MT |
| SDBC | 0.30 MT | 0.45 MT | 0.18 MT | 0.06 MT |
| MSS | 0.25 MT | 0.50 MT | 0.18 MT | 0.06 MT |

**Concrete (per m³ of output):** Same as Camp RMC BoM (§8.4 of Module 02).

**Emulsion (per MT of output):**
| Grade | Bitumen | Water | Emulsifier |
|---|---|---|---|
| SS1 | 0.55 MT | 0.40 MT | 0.02 MT |
| RS1 | 0.60 MT | 0.35 MT | 0.02 MT |

> These are approximate. Client to provide actual ratios.

### 6.6 Stock Impact

On submit:
- **Input materials** — raw stock decreases by Actual Quantity per row. Standard validation: cannot consume more than available.
- **Output (finished good)** — finished stock increases by Output Quantity (stored as the grade-specific product variant).

### 6.7 Two-Step Flow: Produce → Dispatch

```
Step 1: Production (this sub-module)
  Bituminous BM batch: 10 MT
  Consumes: Aggregate 20mm 4.55 MT + Aggregate 10mm 3.05 MT + Sand 2.0 MT + Bitumen 0.55 MT
  Output: BM → 10 MT added to finished stock

Step 2: Plant Outward (§7)
  Material: BM, Qty: 8 MT, Destination: Kalol Sanand site
  → Finished stock decreases by 8 MT
```

> **Why two steps?** Production and dispatch happen at different times. Finished mix may be produced in the morning and dispatched in multiple trucks through the day. Two-step keeps stock accurate at every point.

### 6.8 Business Rules

1. **Visibility tied to capability.** A plant without "Bituminous" capability does not see the Bituminous Production menu.
2. **Input stock validation.** Actual consumption cannot exceed available raw stock.
3. **Output grade filter.** Only grades configured for this capability are selectable.
4. **BoM auto-fill is a suggestion, not a constraint.** Actuals can differ — variance is captured for reporting.
5. **Diesel consumed field is informational** — no deduction from any diesel stock (Diesel Module handles that).
6. **Backdating limit:** Up to 3 days.
7. **Scrap handling:** If the batch produced less than BoM-expected output (e.g., BoM expected 10 MT from inputs but only 9.2 MT came out), the shortfall of 0.8 MT is **implicit wastage** — captured by the difference between BoM-expected input consumption and actual. Explicit scrap (bad batches dumped) uses the Scrap sub-module (§9).

---

## 7. Sub-Module — Plant Outward

**Purpose:** Record finished goods dispatched from the plant to a linked camp or site.
**Odoo Model:** `stock.picking` (type = outgoing) + custom fields on `mcway.plant.outward`.

### 7.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Outward Code | Char | Auto | `POUT/YYYY/####` |
| 2 | Plant | Many2one → Plant | Auto | Pre-filled, read-only |
| 3 | Date & Time | Datetime | ✓ | Defaults to now; backdating up to 3 days |
| 4 | **Destination Type** | Selection | ✓ | Camp / Site |
| 5 | **Destination — Camp** | Many2one → Camp | If Destination=Camp | Filtered to camps this plant supplies (from Plant Master "Supplies To Camps") |
| 6 | **Destination — Site** | Many2one → Site | If Destination=Site | Filtered to sites this plant supplies directly (from Plant Master "Supplies To Sites") |
| 7 | Material | Many2one → Material Master | ✓ | Filtered to **finished goods currently in stock** at this plant (stock > 0) |
| 8 | Variant / Grade | Many2one → Product Variant | Conditional | Same logic as inward |
| 9 | Issue Quantity | Float | ✓ | Primary UoM. Cannot exceed current finished stock. |
| 10 | UoM | Display | Auto | Read-only label |
| 11 | Gross Weight | Float | ✗ | Optional — if weighbridge slip generated at dispatch. Used for cross-check against Issue Quantity. |
| 12 | Tare Weight | Float | ✗ | Optional |
| 13 | Net Weight | Float | Auto | = Gross − Tare (if provided) |
| 14 | Vehicle Number | Char | ✓ | — |
| 15 | Slip / Challan Number | Char | ✓ | Plant generates its own challan |
| 16 | Driver / Transporter | Char | ✗ | Free text |
| 17 | Receiving Person | Char | ✗ | Name at destination (camp or site) |
| 18 | Temperature at Dispatch (°C) | Float | ✗ | For bituminous dispatch (QA) |
| 19 | Photo(s) | Binary (multi) | ✗ | Up to 2 photos (loaded vehicle, challan). Max 5 MB each. |
| 20 | Remarks | Text | ✗ | — |
| 21 | State | Selection | Auto | Draft → Submitted → Verified → Locked |

### 7.2 What This Form Does NOT Capture

- **Chainage:** Not captured at dispatch. If destination is a site, chainage is recorded at site dumping (Module 03) by the Supervisor.
- **Multiple materials per entry:** Each outward entry is for **one material + one variant**. If a vehicle carries BM and tack coat emulsion, two outward entries are created.

### 7.3 Destination Filter

Only camps/sites this plant is configured to supply (via Plant Master "Supplies To") appear in the dropdowns.

- If dispatch is needed to an unlinked destination, Admin must add the linkage first.
- Enforces the hierarchy and prevents data leaks.

### 7.4 Stock Validation

- **Soft warning at 90%:** If dispatch would drop finished stock below 10% of current balance, show warning.
- **Hard block at 0:** Cannot exceed current finished stock. Error: "Insufficient finished stock. Available: X MT."
- **Reservation:** Use Odoo's `stock.quant` reservation on save to prevent simultaneous over-dispatch.

### 7.5 Plant Outward ↔ Camp Inward / On-Site Dumping Reconciliation

Plant outward and camp inward (or site dumping) are **independent entries** in the demo. They share:
- Slip/Challan Number
- Vehicle Number
- Material + Variant

Reconciliation via Reports:
- Material dispatched from plant but not recorded as received (flag mismatch).
- Variance in net weight (plant dispatched 20 MT, camp/site received 19.5 MT).

> **Phase 2 enhancement:** Auto-suggest matching inward when a downstream slip number matches an existing plant outward.

### 7.6 Business Rules

1. **One material per entry.** Simpler forms, cleaner stock math.
2. **Stock check on save.** Quantity reserved on save; released if entry is deleted/cancelled.
3. **Backdating limit:** 3 days.
4. **Duplicate slip warning:** Non-blocking.
5. **On save (Submit):** Finished stock decreases. Entry appears in Verification Queue.
6. **Edit after submit:** Allowed until verified. Stock auto-adjusts.

---

## 8. Sub-Module — Stock Register (Raw + Finished)

**Purpose:** Real-time, auto-maintained inventory view. Plant In-charge **never enters stock manually** — stock emerges from inward, production, outward, and scrap transactions.
**Odoo Model:** `stock.quant` (standard) + custom summary views.

### 8.1 Two-Tab Layout

| Tab | Contents |
|---|---|
| **Raw Materials** | Aggregate (all variants), Sand, Bitumen, Cement, Steel (rods), Emulsifier, Water, etc. |
| **Finished Goods** | Bituminous mixes (BM, DBM, BC, SDBC, MSS), Concrete grades (M10–M40), Emulsion grades (SS1, RS1), Cut/Bent Steel |

Both tabs follow the same summary view pattern (below); transactional drivers differ.

### 8.2 Summary View Columns (Raw)

| Column | Source | Notes |
|---|---|---|
| Material | Material Master | — |
| Variant / Grade | Product Variant | e.g., Aggregate 20mm |
| UoM | Material Master | MT / m³ / numbers / L |
| Opening Stock | System | Balance at start of selected date |
| Inward | Σ raw inward qty (date range) | — |
| Production Consumed | Σ consumed across all production batches | — |
| Scrap (raw) | Σ scrap entries for raw materials | — |
| **Closing Stock** | Opening + Inward − Production Consumed − Scrap | **Bold** |
| Threshold | From Material Master | Low Stock Threshold |
| Status | Computed | 🟢 OK / 🟡 Low / 🔴 Critical |

### 8.3 Summary View Columns (Finished)

| Column | Source | Notes |
|---|---|---|
| Product | Material Master (finished category) | e.g., "BM", "M25 Concrete Mix" |
| Variant / Grade | Product Variant | — |
| UoM | Material Master | MT / m³ |
| Opening Stock | System | Balance at start of selected date |
| Produced | Σ production output (date range) | — |
| Dispatched | Σ plant outward (date range) | — |
| Scrap (finished) | Σ scrap entries for finished goods | — |
| **Closing Stock** | Opening + Produced − Dispatched − Scrap | **Bold** |
| Threshold | From Material Master | — |
| Status | Computed | 🟢 OK / 🟡 Low / 🔴 Critical |

### 8.4 Stock Status Rules (shared)

| Condition | Status | Color |
|---|---|---|
| Closing ≥ Threshold × 2 | OK | Green |
| Threshold ≤ Closing < Threshold × 2 | Low | Yellow |
| Closing < Threshold | Critical | Red |
| No threshold set | OK (always) | Green |

### 8.5 Stock Ledger (Detailed Transactions)

Drill from any row (raw or finished) into full transaction history for that material + variant.

| Column | Notes |
|---|---|
| Date & Time | — |
| Transaction Type | Raw Inward / Production Consumed / Production Output / Plant Outward / Scrap / Opening Adjustment |
| Reference | Entry code (e.g., `PIN/2026/0032`, `PPRD/BIT/2026/0021`) |
| Source / Destination | Vendor name / Camp name / Site name / "Production batch" |
| Qty In | Positive entries |
| Qty Out | Negative entries |
| Running Balance | Cumulative |
| State | Submitted / Verified / Locked |
| Entered By | User name |

### 8.6 Date Range Filter
Same as Camp Module §7.4 — Today (default) / Yesterday / This Week / This Month / Custom.

### 8.7 Opening Stock (Go-Live)

On go-live, existing physical raw and finished stock is entered via **"Opening Stock Adjustment"** action (Admin-only), same mechanism as Camp Module §7.5.

### 8.8 Business Rules

1. **No manual stock edits** except one-time Opening Adjustment.
2. **Negative stock prevention** — outward, production consumption, and scrap validate against available balance.
3. **Stock per material + variant** — Aggregate 6mm and Aggregate 20mm tracked separately.
4. **Raw and Finished are separate stock locations** — Aggregate (raw) and BM (finished) cannot cross-contaminate in reports.
5. **Real-time updates.**
6. **Low stock alerts** — notification on Plant Home tile; optional email to Admin.

---

## 9. Sub-Module — Scrap & Wastage

**Purpose:** Record raw or finished material that has been scrapped, wasted, rejected, or lost. Unlike implicit BoM variance (captured in production), this is **explicit** scrap entry.
**Odoo Model:** `stock.scrap` (standard Odoo) + custom metadata in `mcway.plant.scrap`.

### 9.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Scrap Code | Char | Auto | `PSCR/YYYY/####` |
| 2 | Plant | Many2one → Plant | Auto | Pre-filled |
| 3 | Date & Time | Datetime | ✓ | Defaults to now; backdating up to 3 days |
| 4 | **Scrap Category** | Selection | ✓ | Raw-material wastage / Batch rejection / Spillage / Contamination / Other |
| 5 | Material | Many2one → Material Master | ✓ | Any material — raw or finished |
| 6 | Variant / Grade | Many2one → Product Variant | Conditional | — |
| 7 | Quantity | Float | ✓ | In material's UoM; cannot exceed available stock of that material |
| 8 | **Linked Batch** | Many2one → Production Batch | ✗ | If scrap arose from a specific batch (e.g., rejected mix) |
| 9 | **Reason** | Text | ✓ | Must explain (e.g., "Batch failed slump test", "Bitumen overheated", "Aggregate contaminated with clay") |
| 10 | Photo(s) | Binary (multi) | ✗ | Up to 2 photos |
| 11 | Disposal Method | Selection | ✗ | Disposed / Reused elsewhere / Pending disposal |
| 12 | State | Selection | Auto | Draft → Submitted → Verified → Locked |

### 9.2 Business Rules

1. **Reason is mandatory** — scrap without reason cannot be submitted.
2. **Stock validation** — cannot scrap more than available.
3. **On save (Submit):** Stock decreases. Entry appears in Verification Queue.
4. **Linked Batch optional** — scrap can arise without a specific batch (e.g., spillage during storage, contamination).
5. **Scrap affects closing stock** in the Stock Register as a separate column.
6. **High-frequency flag** — if more than 3 scrap entries are logged in a day, dashboard shows an alert for Admin/Owner review.

---

## 10. Sub-Module — Labor at Plant

**Purpose:** Log labor (contract workers) arriving at the plant for production or housekeeping activities.
**Odoo Model:** Custom `mcway.labor.entry` (same model used by Camp Module §9; scope = plant).

### 10.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Entry Code | Char | Auto | `LAB/YYYY/####` (shared sequence) |
| 2 | Plant | Many2one → Plant | Auto | Pre-filled, read-only |
| 3 | Date | Date | ✓ | Defaults to today; backdating up to 3 days |
| 4 | Party Name (Mukadam) | Char | ✓ | Free text with autocomplete from previous plant entries |
| 5 | Number of Workers | Integer | ✓ | > 0 |
| 6 | Purpose / Activity | Many2one → Activity (lookup) | ✗ | e.g., Plant operations / Loading-unloading / Maintenance / Housekeeping / General |
| 7 | Remarks | Text | ✗ | — |
| 8 | State | Selection | Auto | Draft → Submitted → Verified → Locked |

### 10.2 Design Decisions

- **Party Name is free text**, consistent with Camp Module.
- **No "Deployed To" field** — unlike camp labor, plant labor does not deploy to sites. Plant labor works at the plant.
- **No cost/wage tracking** in demo.
- **Multiple entries per day allowed.**

### 10.3 Business Rules

1. Number of Workers > 0.
2. Backdating limit: 3 days.
3. Feeds into Reports → Daily Labor Report and Dashboard.

---

## 11. Sub-Module — Verification Queue

**Purpose:** Single screen where Verifiers (Senior Engineer or Admin) review and verify entries submitted by the Plant In-charge.
**Shared pattern** — same UX as Camp Module §11.

### 11.1 Queue View

A filterable list of all **submitted but unverified** entries for this plant.

| Column | Notes |
|---|---|
| Entry Code | Clickable → opens entry detail |
| Type | Raw Inward / Production / Outward / Scrap / Labor |
| Capability | For Production entries: Bituminous / RMC / Emulsion / Steel |
| Date | — |
| Material / Description | — |
| Quantity | With UoM |
| Entered By | User name |
| Submitted At | Timestamp |
| **Age** | "2h ago", "1d 4h" |
| Action | Verify / Request Correction |

### 11.2 Verification Actions

| Action | Effect |
|---|---|
| **Verify** | State → Verified. Verifier name + timestamp recorded. Removed from queue. |
| **Request Correction** | Remark sent to Plant In-charge. Entry stays in `submitted`. Flag/badge appears on entry. Plant In-charge edits and re-submits. |

### 11.3 Bulk Verify
Select multiple entries → "Verify Selected" button.

### 11.4 Filters & Sorting

- Filter by: Type, Capability (for production), Date Range, Entered By, Age (> 24h, > 48h).
- Sort by: Date (newest first default), Age (oldest first).
- **Highlight rule:** Entries older than the system threshold (default 2 days) are highlighted orange.

### 11.5 Variance Flagging (Production-Specific)
Production entries with a variance > 10% on any input material are flagged with a ⚠️ icon in the queue, prompting closer review by the verifier.

### 11.6 Verification Rules (Recap)

1. **Verification never blocks** — Submitted entries are immediately usable in stock, reports, dashboard.
2. **Unverified entries are flagged** in Reports and Dashboard.
3. **Verifier cannot edit the entry** — only verify or request correction.
4. **Audit trail:** Who verified, when, from which IP (via `mail.thread` + `auditlog`).

---

## 12. State Machine (All Transactional Entries)

All transactional records in the Plant Module follow the same state progression as Camp Module §12:

```
┌─────────┐     Save/Submit     ┌───────────┐     Verify     ┌──────────┐     Period Close     ┌────────┐
│  Draft   │ ─────────────────→ │ Submitted │ ─────────────→ │ Verified │ ──────────────────→ │ Locked │
└─────────┘                     └───────────┘                 └──────────┘                     └────────┘
                                      │                             │
                                      │ Edit (by Plant In-charge)   │ Unlock (Admin only)
                                      ↓                             ↓
                                 Back to Submitted            Back to Submitted
```

### 12.1 State Definitions

| State | Who can edit | Stock impact | Visible in reports | In verification queue |
|---|---|---|---|---|
| **Draft** | Plant In-charge | No | No | No |
| **Submitted** | Plant In-charge | Yes | Yes (flagged as unverified) | Yes |
| **Verified** | Admin only (unlock required) | Yes | Yes (clean) | No |
| **Locked** | Nobody | Yes | Yes | No |

### 12.2 Notes

- **Draft is ephemeral** — one-click "Save & Submit" is the usual path.
- **No Rejected state** — correction requests keep entry in Submitted.
- **Locked** is set by Admin during period close.
- **Stock impact starts at Submitted** — accountability layer, not a gating mechanism.

---

## 13. Access Control

### 13.1 Model-Level Access

| Model | Plant In-charge | Admin | Owner | SE | Verifier | Others |
|---|---|---|---|---|---|---|
| Plant Raw Inward | CRUD (own plant) | CRUD | R | R | R | — |
| Plant Production (all caps) | CRUD (own plant) | CRUD | R | R | R | — |
| Plant Outward | CRUD (own plant) | CRUD | R | R | R | — |
| Plant Stock Register | R (own plant) | R | R | R | R | — |
| Plant Scrap | CRUD (own plant) | CRUD | R | R | R | — |
| Labor Entry | CRUD (own plant) | CRUD | R | R | R | — |
| Verification | — | Verify | — | Verify (own sites' plants) | Verify | — |

### 13.2 Record Rules (Scope Enforcement)

- **Plant In-charge** sees only data for their assigned plant.
- **Senior Engineer** can verify entries for plants that supply their assigned site.
- **Admin** sees all plants.
- **Owner** sees all plants (read-only).
- Enforced via Odoo `ir.rule` on `user.assignment_target` matching the plant or the plant's supplied sites.

---

## 14. Pre-loaded Demo Data & Walkthrough

### 14.1 Demo Plant
**Magodi** (`PLNT/2026/001`) — Plant In-charge: Pravin Modi. Capabilities: **Bituminous, Concrete (RMC), Emulsion, Steel**. Daily Capacity: Bituminous 500 MT, RMC 300 m³, Emulsion 10 MT, Steel 50 MT. Supplies to Kalol Sanand (site, direct) + Khatraj Chokdi (camp).

### 14.2 Pre-loaded Raw Material Inward Entries (6 entries)

| # | Code | Date | Material | Variant | Source | Vendor | Vehicle | Slip | Net Wt |
|---|---|---|---|---|---|---|---|---|---|
| 1 | PIN/2026/0001 | 2026-04-14 08:00 | Aggregate | 20mm | Quarry | Shree Construction | GJ-05-AB-1111 | SLP-0501 | 28.0 MT |
| 2 | PIN/2026/0002 | 2026-04-14 10:30 | Aggregate | 10mm | Quarry | Shree Construction | GJ-05-AB-2222 | SLP-0502 | 26.0 MT |
| 3 | PIN/2026/0003 | 2026-04-14 14:00 | Bitumen | VG-30 | Vendor | IOCL Depot | GJ-03-TK-4444 | SLP-0503 | 18.0 MT |
| 4 | PIN/2026/0004 | 2026-04-15 09:15 | Sand | — | Quarry | Shree Construction | GJ-05-AB-3333 | SLP-0504 | 22.0 MT |
| 5 | PIN/2026/0005 | 2026-04-15 11:30 | Cement | — | Vendor | Ambuja Cement | GJ-01-XX-9999 | SLP-0505 | 25.0 MT |
| 6 | PIN/2026/0006 | 2026-04-16 08:45 | Steel | 12mm rod | Vendor | Maruti Hirers | GJ-01-XX-7777 | SLP-0506 | 12.0 MT |

### 14.3 Pre-loaded Production Batches (4 entries)

| # | Code | Date | Capability | Grade | Output | Inputs Summary | State |
|---|---|---|---|---|---|---|---|
| 1 | PPRD/BIT/2026/0001 | 2026-04-15 10:00 | Bituminous | BM | 10.0 MT | Agg 20mm 4.55, Agg 10mm 3.05, Sand 2.00, Bitumen 0.55 | Verified |
| 2 | PPRD/BIT/2026/0002 | 2026-04-16 09:30 | Bituminous | DBM | 12.0 MT | Agg 20mm 4.85, Agg 10mm 4.25, Sand 2.45, Bitumen 0.62 | Submitted |
| 3 | PPRD/RMC/2026/0001 | 2026-04-16 11:00 | Concrete | M25 | 5.0 m³ | Cement 1.80 MT, Sand 3.00 MT, Agg 20mm 5.25 MT, Water 850 L | Submitted |
| 4 | PPRD/EML/2026/0001 | 2026-04-17 07:30 | Emulsion | SS1 | 2.0 MT | Bitumen 1.10 MT, Water 0.80 MT, Emulsifier 0.04 MT | Submitted |

### 14.4 Pre-loaded Plant Outward Entries (3 entries)

| # | Code | Date | Material | Variant | Destination | Qty | Vehicle | Slip |
|---|---|---|---|---|---|---|---|---|
| 1 | POUT/2026/0001 | 2026-04-15 14:00 | BM | — | Site: Kalol Sanand | 8.0 MT | GJ-05-AB-4444 | PLT-0088 |
| 2 | POUT/2026/0002 | 2026-04-16 08:00 | Aggregate | 6mm | Camp: Khatraj Chokdi | 15.0 MT | GJ-01-ZZ-3456 | PLT-0089 |
| 3 | POUT/2026/0003 | 2026-04-16 14:30 | DBM | — | Site: Kalol Sanand | 10.0 MT | GJ-05-AB-5555 | PLT-0090 |

> **Note:** Entry #2 is a raw material (Aggregate 6mm) passing through the plant — plants can also act as a raw distribution point when they over-stock or rebalance. This is supported by the model (outward allows any material currently in stock, raw or finished).

### 14.5 Expected Stock After Demo Data

**Raw (selected):**
| Material | Variant | Inward | Production Consumed | Outward | Closing |
|---|---|---|---|---|---|
| Aggregate | 20mm | 28.0 | 14.65 | 0 | 13.35 MT |
| Aggregate | 10mm | 26.0 | 7.30 | 0 | 18.70 MT |
| Bitumen | VG-30 | 18.0 | 2.27 | 0 | 15.73 MT |
| Sand | — | 22.0 | 7.45 | 0 | 14.55 MT |
| Cement | — | 25.0 | 1.80 | 0 | 23.20 MT |
| Steel | 12mm rod | 12.0 | 0 | 0 | 12.00 MT |

**Finished (selected):**
| Product | Variant | Produced | Dispatched | Closing |
|---|---|---|---|---|
| BM | — | 10.0 | 8.0 | 2.0 MT |
| DBM | — | 12.0 | 10.0 | 2.0 MT |
| M25 Concrete | — | 5.0 | 0 | 5.0 m³ |
| SS1 Emulsion | — | 2.0 | 0 | 2.0 MT |

### 14.6 Pre-loaded Scrap Entry (1 entry)

| Date | Category | Material | Qty | Reason | Linked Batch |
|---|---|---|---|---|---|
| 2026-04-15 16:00 | Spillage | Bitumen VG-30 | 0.15 MT | Leak during transfer from tank to mixer | — |

### 14.7 Pre-loaded Labor Entry (1 entry)

| Date | Party | Workers | Purpose |
|---|---|---|---|
| 2026-04-17 | Kishor Mukadam | 10 | Plant operations (loading/unloading) |

### 14.8 Demo Walkthrough (Suggested)

1. **Log in as Plant In-charge (Pravin Modi)** → lands on Plant Home showing today's tiles and capacity utilization strip.
2. **Record raw inward** — Cement, 25 MT from Ambuja Cement. Show gross/tare/net auto-calc.
3. **View Stock Register → Raw tab** — show Aggregate, Bitumen, Sand, Cement balances with drill-down to ledger.
4. **Record Bituminous Production** — BM grade, 10 MT output. Show BoM auto-fill for inputs, adjust Bitumen actual slightly (0.55 → 0.56 MT) to demonstrate variance capture. Show finished stock increase.
5. **Record Plant Outward** — BM, 8 MT to Kalol Sanand site. Show stock validation; only finished goods with stock > 0 appear. Show dispatch slip creation.
6. **View Stock Register → Finished tab** — BM closing = 2 MT after dispatch; drill into ledger.
7. **Switch capability — record RMC batch** — M25, 5 m³. Show BoM auto-fill from Concrete capability.
8. **Log a scrap entry** — Spillage 0.15 MT Bitumen with reason.
9. **Switch to Senior Engineer (Kiran)** → Verification Queue → bulk-verify today's submitted entries. Point out the variance ⚠️ icon on the BM batch.
10. **Show capacity utilization strip on Home** — after batches, Bituminous = 22 MT / 500 MT (4.4%), Concrete = 5 m³ / 300 m³ (1.7%).

---

## 15. Design Decisions & Assumptions

| # | Decision | Rationale |
|---|---|---|
| 1 | **Capability-driven production sub-menus** | Plants vary in what they produce. Hiding irrelevant screens reduces clutter and enforces business rules. |
| 2 | **BoM auto-fill with editable Actual column** | Production is never perfectly BoM-compliant. Operators adjust; variance is captured for reporting, not blocked. |
| 3 | **Two-step Produce → Dispatch** | Production and dispatch timings differ; two-step keeps finished-goods stock accurate. |
| 4 | **Raw vs Finished as separate stock tabs** | Operationally they are different — aggregate (raw) and BM (finished) don't share reporting. Odoo stock locations enforce this. |
| 5 | **No Plant-to-Plant transfers in demo** | Only one plant in scope. Add as outward→inward pair in phase 2. |
| 6 | **Plant Outward can ship either raw or finished** | Plants may rebalance raw stock to camps (e.g., extra aggregate). Filter is "stock > 0", not "finished only". |
| 7 | **Diesel consumed field is informational** | Actual diesel tracking lives in Diesel Module (Module 06). Avoid double deduction. |
| 8 | **Explicit Scrap module, implicit variance in Production** | Two distinct concepts: (a) batch consumed more/less than BoM expected (variance), (b) material physically dumped (scrap). Both tracked separately. |
| 9 | **Quarry = Vendor with role tag** | Same as Camp Module — no new master. |
| 10 | **Plant In-charge hard-scoped to assigned plant** | Per Module 01 user assignment rules. |
| 11 | **Backdating up to 3 days** | Same threshold as Camp Module for consistency. |
| 12 | **Output Grade filtered by capability** | A plant with only "Bituminous" capability cannot produce M25 concrete — dropdown is filtered accordingly. |
| 13 | **Variance highlight on Production entries > 10%** | Provides verifier a clear signal to scrutinise. Thresholds configurable. |
| 14 | **No batch-level QA tests in demo** | Temperature/slump are captured as free fields; formal QA test records (compressive strength, Marshall stability) are phase 2. |
| 15 | **Weighbridge exists at plant** | Gross/Tare is entered for raw inward. Outward weighbridge is optional (captured when available for cross-check). |
| 16 | **Plant Labor has no "Deployed To"** | Plant labor works at the plant — no cross-deployment to sites. (Camp labor differs because camp is a gathering point.) |

---

## 16. Open Items / Notes for Developers

1. **BoM ratios** — placeholders in §6.5 for BM/DBM/BC/SDBC/MSS, M10–M40, SS1/RS1 etc. Client to provide actuals. Keep editable via Odoo MRP UI.
2. **Emulsion BoM** — approximate values pending client confirmation.
3. **Steel cutting/bending BoM** — no meaningful BoM. Input = rod stock, output = cut pieces. Treat as 1:1 (minus minor cutting scrap).
4. **Variance threshold (10%)** — make configurable in System Thresholds (Module 01).
5. **Photo compression** — same spec as Camp Module (§16 item 3). 1080p, 5 MB cap.
6. **Concurrent outward race condition** — same `stock.quant` reservation pattern.
7. **Stock Register performance** — two tabs each with summary SQL views. Materialized views recommended for plants with >10k transactions.
8. **Capacity utilization calc** — today's total output per capability vs Daily Capacity from Plant Master. Roll-up across grades within the same capability.
9. **Shared `mcway.labor.entry` model** — ensure scope field supports Camp, Plant, and future Site (Module 05 may log site labor). Use a polymorphic `scope_type` + `scope_id` pattern.
10. **Opening Stock Adjustment** — one-time, Admin-only. Support bulk CSV upload for go-live.
11. **Period locking** — cross-cutting, defined in a future Period Management spec.
12. **Scrap reporting** — aggregate scrap by category (raw wastage vs batch rejection vs spillage) for Owner Dashboard. Wastage % = scrap / inward.
13. **Capability enablement after go-live** — if a plant adds a new capability (e.g., starts producing Emulsion), the menu item should appear immediately for that Plant In-charge on next login.
14. **Temperature / slump fields** — informational only in demo. If formal QA is added in phase 2, these become part of a QA Test sub-module linked to the batch.
15. **Vehicle number autocomplete** — session-level cache; consider promoting to a Vehicle Master in phase 2.
16. **Raw vs finished filter in Material dropdown** — depends on Material Master's Category. Ensure Category → Raw / Finished mapping is maintained for accurate filtering.
