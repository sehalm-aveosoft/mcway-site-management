# McWay Site Management — Demo Documentation

**Version:** Demo v1
**Date:** 2026-04-15
**Reference Documents:**
- `_Documentation in detail.md` (original full-scope document)
- `McWay_Meeting_Notes_2026-04-14.md` (client meeting notes)

---

## 1. Purpose of This Document

This document defines the **demo build** of the McWay Site Management system based on the ERP-style approach agreed with the client on 2026-04-14. It is structured module-by-module and captures every feature, field, role, workflow, and open item required to deliver a working demo.

The demo is intentionally narrower than the full-scope document (`_Documentation in detail.md`), but **must still support dynamic creation** of sites, camps, plants, users, and materials through the Admin Panel — nothing is hardcoded.

---

## 2. Demo Scope Summary

### 2.1 In Scope
| Module | Platform | Primary User |
|---|---|---|
| Admin Panel | Web | Admin / Management |
| Camp Module | Web | Camp In-charge |
| Plant Module | Web | Plant In-charge |
| On-Site Materials Module | Mobile App | Supervisor |
| Senior Engineer Work Progress | Web | Senior Engineer |
| Diesel Module | Web + Mobile | Tanker Operator / Camp In-charge / Admin |
| Reports Module | Web | Management / Senior Engineer |
| Dashboard | Web | Owner / Upper Management only |

### 2.2 Pre-loaded Demo Data
- **1 Site:** Kalol Sanand
- **1 Camp:** Khatraj Chokdi (serves Kalol Sanand)
- **1 Plant:** Magodi (supplies camps)
- All material masters, user roles, and sample users configured for the above.

### 2.3 Out of Scope for Demo
- **Machine usage tracking** (ON/OFF/Breakdown, idle time, utilization analytics) — deferred to phase 2.
- Machine **master data** is still included (required so diesel can be issued to a named machine).
- Financial accounting, vendor billing, theft detection logic.

---

## 3. System Hierarchy (Entity Relationships)

For this demo, **Site = Project** (no separate Project layer). The top-level entity is the Site itself, and Camps/Plants link to it.

```
Site (e.g., Kalol Sanand)
│
├── Can be served by 0, 1, or many Camps
├── Can also be supplied directly by a Plant (if no camp is needed)
│
├── Camp (e.g., Khatraj Chokdi)
│   ├── Serves one or more Sites (a single camp can feed multiple sites)
│   ├── Has its own RMC/mixing capability (small-scale)
│   └── Receives supply from Plants
│
└── Plant (e.g., Magodi)
    ├── Supplies one or more Camps
    ├── Can also supply Sites directly
    └── Has large-scale RMC / bitumen mixing production
```

### Key Rules
- **A site can have multiple camps, or none.**
- **A plant supplies camps** (and sometimes supplies sites directly).
- **Two sites can share one camp** placed between them.
- **Both Camp and Plant can produce RMC**, but the Plant produces at a much larger scale.
- Camp ↔ Site and Plant ↔ Camp/Site linkages are configurable in the Admin Panel.

---

## 4. User Roles (Demo)

| Role | Platform | Primary Responsibility |
|---|---|---|
| **Admin** | Web | System configuration, masters, user creation, thresholds |
| **Upper Management / Owner** | Web | Dashboard view, approvals, reports, high-level oversight |
| **Senior Engineer** | Web | Work progress entry, verification of site entries |
| **Supervisor** | Mobile App | On-site material dumping entries |
| **Camp In-charge** | Web | Camp inward/outward, diesel, inventory, labor |
| **Plant In-charge** | Web | Plant inward/outward, production (RMC, mixes), diesel |
| **Tanker Operator** | Mobile/Web | Diesel pickup from pump, dispensing to machines |
| **Verifier** | Web | Verifies records submitted by other roles (see §4.1) |

### 4.1 Verifier Role — Important Behavior
- The Verifier **cannot restrict or block** entries — entries are always permitted at the time of submission.
- The Verifier's job is to **review and mark entries as "Verified"** after they are submitted.
- An unverified entry is still usable in the system, but it will be flagged in reports and on the dashboard until verification is completed.
- Verification provides an accountability audit trail, not a hard gate.

### 4.2 Direct User Entry Principle
Every user enters their **own** data. No data-entry operator or clerk enters on behalf of others. The role that performs the activity is the role that records it.

---

## 5. Module 1 — Admin Panel (Web)

Used by Admin and Upper Management to configure the system before operations and maintain control during execution.

### 5.1 Site, Camp & Plant Setup
- Create / edit **Sites** (Site = top-level entity; no separate Project layer in demo):
  - Name, location
  - Type (Road / Bridge / Building — demo focus = Road)
  - Chainage start and end (e.g., 0 km – 27 km)
  - Responsible senior engineer, supervisors
  - Status: active / inactive / closed
- Create / edit **Camps**:
  - Name, location
  - Link to one or more sites it serves
  - Responsible Camp In-charge
- Create / edit **Plants**:
  - Name, location
  - Link to camps (and/or sites) it supplies
  - Responsible Plant In-charge
  - Production capabilities (RMC, bitumen mix, etc.)

### 5.2 Material Master
All materials, grades, and units used in the system are defined here — no free-text entries elsewhere.

Material fields:
- Material name
- Category (aggregate / binder / structural / pipe / fuel / etc.)
- Grades / sizes / types
- Unit of measurement (MT, m³, litre, bag, numbers)
- Density / conversion factor (for volume ↔ weight conversions)
- Source type(s): Plant / Camp / External vendor
- Standard specification reference (optional)

**Pre-loaded materials for demo** — see §12 (Materials List Appendix).

### 5.3 Machine Master
Even though machine *usage* is out of scope, machine master data **is** required so diesel can be issued to a named machine.

Fields:
- Machine name / code
- Category (JCB, roller, paver, loader, excavator, dumper, truck, mixer, tanker, etc.)
- Ownership: Company-owned / Rented without fuel / Rented with fuel
- Capacity / specification
- Expected fuel average (litres/hour or litres/km)
- Current assignment (site / camp / plant)

### 5.4 User & Role Administration
- Create users, assign role.
- Map users to sites / camps / plants.
- Reassign when structures change.
- Every user must have login credentials (direct entry principle).

### 5.5 Requirement & Estimation Configuration
Included in demo so that variance/dashboard values are meaningful.
- Estimated material quantity per site / stretch:
  - **Formula:** Length × Width × Thickness × Density
- Daily / weekly / monthly work targets
- Expected diesel requirement
- Expected machine deployment count
- Baseline values used by the Reports Module for variance calculations

### 5.6 Approval & Verification Workflow Control
- Configure which entry types require Verifier review.
- Lock / reopen submitted data for corrections.
- Audit log of all edits and approvals.
- Historical Excel bulk upload (optional).

### 5.7 System Rules & Thresholds
- Variance tolerance % (e.g., 2–5% between estimated vs actual material).
- Idle machine alert thresholds (used by phase 2, configurable now).
- Report frequencies (daily / weekly / monthly).
- Low-stock thresholds at camp / plant.

---

## 6. Module 2 — Camp Module (Web)

The Camp is a storage + small-scale production base serving one or more sites. The Camp Module manages inventory, diesel stock, labor, and small-scale RMC operations.

### 6.1 Material Inward at Camp
Recorded when material arrives **at the camp** from a plant, vendor, quarry, or stockyard.

Fields:
- Date & time
- Material type (dropdown from master)
- Grade / size (dropdown from master)
- Vehicle number
- Slip number
- Royalty number (government compliance)
- Gross weight
- Tare weight
- **Net weight** = Gross − Tare (auto-calculated)
- Source (Plant / Vendor / Quarry — dropdown)
- Photo (optional)

System actions:
- Updates camp inventory stock (opening + inward = updated balance).
- Logs into inward history for audit.
- Validates material against master.

### 6.2 Material Outward from Camp to Site
Recorded when stored material is dispatched from camp to a site chainage.

Fields:
- Date & time
- Material type & grade
- Destination site + chainage location
- Vehicle number
- Issue quantity
- Slip number / challan
- Receiving supervisor name (optional)

System actions:
- Reduces camp inventory.
- Links this outward entry to the corresponding On-Site Material dumping entry (vendor → camp → site traceability).

### 6.3 Inventory & Stock Register
Automatically maintained by the system:
- Opening stock
- Inward quantity
- Outward quantity
- Closing stock
- RMC stock (when camp has a mixer)

Tracked categories (demo): aggregates (6mm/10mm/20mm/25–40mm/storm dust), sand, cement, steel (8–32mm dia), emulsion (SS1/RS1), shuttering, pipes, RMC, plus any custom-configured materials.

Shuttering special handling: track inward, issue, return, damage, loss.

Future: low-stock alerts, reorder suggestions.

### 6.4 Diesel Stock & Issue (Camp-Level)
See full Diesel Module in §10 — camp operations are part of that flow.

### 6.5 RMC (Camp-Level, Small-Scale)
When the camp has its own mixer:

Fields:
- Concrete grade (M10, M15, M20, M25, M30, M35, M40)
- Quantity produced (m³)
- Input materials consumed (aggregates, cement, sand, water — with quantities)
- Diesel consumed by mixer
- Shift & operator details
- Linked site / chainage where it was used

**Note:** *How concrete is prepared exactly (mix ratios, input transformation logic) — to be provided later by the client.* Entries may evolve when that logic is finalized, because for example aggregate + cement mixing produces concrete and the output material differs from the input.

### 6.6 Labor Arrival & Deployment
Fields:
- Date
- Camp / site location
- Party name (Mukadam)
- Number of labor brought
- Purpose of work / remarks
- Optional link to activity (paving, concrete, excavation, shuttering)

### 6.7 Verification
- Camp In-charge submits all entries.
- Verifier (Senior Engineer or assigned role) reviews and marks verified.
- Entries are not blocked prior to verification.

---

## 7. Module 3 — Plant Module (Web)

The Plant is a large-scale production and supply base. A Plant supplies camps and can also supply sites directly.

### 7.1 Plant Material Inward
Recorded when raw material arrives at the plant.

Fields: same as Camp inward (§6.1) — date, material, grade, vehicle, slip, royalty, gross/tare/net, source, photo optional.

**Pre-configured materials** for plant inward (see §12):
- Aggregate (6/10/20/25–40mm/storm dust) — MT
- Cement — MT + bags
- Sand — MT
- Steel — MT (dia 8/10/12/16/20/25/32 mm)
- Emulsion — MT (SS1, RS1)
- Bitumen — MT (VG30, VG40) *(Magodi only)*
- LDO / FO — litres *(Magodi only)*

### 7.2 Plant Production
Because the plant **transforms** raw materials into mixed products, the production flow tracks input consumption and output produced.

**Production types:**
- **Bituminous mixes:** BM, DBM, BC, SDBC, MSS
- **Concrete mixes (RMC):** M10, M15, M20, M25, M30, M35, M40 (unit: m³)
- **Emulsion (SS1, RS1)**
- **Steel cutting/bending** (if done at plant)

Fields per production batch:
- Date, shift, operator
- Output type, grade, quantity
- Input materials consumed (with grade & quantity — multiple lines per batch)
- Diesel / LDO / FO consumed
- Remarks

System actions:
- Reduces plant inventory of inputs.
- Increases plant inventory of output product.
- Logs production history for reporting.

**Note on concrete & mix preparation logic:** *Exact ratios and transformation rules to be shared later by the client. Once received, the input-consumption calculations will be updated.*

### 7.3 Plant Outward
Recorded when produced or raw material is dispatched from the plant to a camp or a site.

Fields:
- Date & time
- Destination type (Camp / Site) + name
- Material / product + grade
- Quantity, unit
- Vehicle number
- Slip / challan
- Gross / tare / net weight (for weight-based materials)

**Pre-configured plant outward items:**
- From Khatraj Chokdi camp (as per client): Steel, Emulsion, Concrete mix (M10–M40)
- From Magodi plant: BM, DBM, BC, SDBC, MSS, Bitumen (VG30/VG40), Steel, Emulsion

### 7.4 Plant Inventory & Stock Register
Maintained automatically — opening, inward, production-in, production-out, outward, closing — separately for raw materials and produced products.

### 7.5 Verification
Same model as Camp — Plant In-charge enters; Verifier reviews.

---

## 8. Module 4 — On-Site Materials Module (Mobile App)

Used by Supervisors on the field to record **material dumping at specific road chainages**. The site is not a storage location; no stock is maintained here.

### 8.1 Dumping Entry Fields
- Date & time (auto-filled, editable)
- Material type & grade (dropdown from master)
- **Chainage start & end** — **manual input** (km or metres)
- Side: LHS / RHS / Lane / Median
- Activity / layer: GSB / WMM / BM / DBM / BC / SDBC / CC / Pipe-laying / etc.
- Source: Camp / Plant / Direct vendor (dropdown)
- Source reference (which camp/plant outward slip — optional linking)
- Vehicle number
- Party / vendor name
- Slip number
- Royalty number
- Gross / tare / net weight (or quantity + unit for non-weight items like pipes)
- **Photo of slip / dumping location — OPTIONAL**
- Remarks

### 8.2 Scope Rules
- Supervisor records **only dumping**. Returns / rejections are not captured in the demo.
- All material names and grades come from the Material Master — no free text.
- Entries are submitted directly by the supervisor — no intermediary.

### 8.3 Verification
- Supervisor submits entry.
- Senior Engineer (acting as Verifier) reviews and marks verified.
- Entries are not blocked; verification is post-hoc accountability.

---

## 9. Module 5 — Senior Engineer Work Progress (Web)

Used by Senior Engineer to record and monitor **work progress** against planned chainage and layers.

### 9.1 Progress Entry — Proposed Methods
Two methods are proposed; **client to choose one** (or approve a hybrid).

**Method A — Layer % per Chainage Segment (recommended)**
- The site chainage is divided into configurable segments (default 100 m or 500 m buckets).
- For each segment, for each layer (GSB / WMM / BM / DBM / BC / SDBC / CC), the Senior Engineer enters one of:
  - Not Started
  - In Progress (with % complete, 0–99)
  - Complete (100%)
- Visual: a colored chainage strip showing layer-wise progress across the site.

**Method B — Quantity-Based (auto-calculated %)**
- System stores **planned quantity** per layer per segment (from Admin Panel estimation).
- Senior Engineer enters **executed quantity** per layer per segment.
- % = Executed / Planned × 100 (auto-calculated).
- More objective but requires accurate estimates upfront.

> **My recommendation:** start with Method A for demo simplicity; Method B can be added in phase 2 once estimates stabilize.

### 9.2 Progress Entry Fields (Method A)
- Date
- Site (pre-filled for user's assigned site)
- Chainage start – end
- Side: LHS / RHS / Lane / Median
- Layer / activity
- Status: Not Started / In Progress (%) / Complete
- Photos (multiple, optional)
- Remarks

### 9.3 Visual Output
- Chainage-wise progress strip (color-coded by layer status).
- Layer-wise completion summary (e.g., GSB 80%, WMM 60% overall).
- Cumulative site progress %.

### 9.4 Verification Role
The Senior Engineer is also the **Verifier** for supervisor entries (§8.3) and optionally for Camp entries. Verification UI is provided on the web — a list of unverified entries with Verify / Reject buttons and a remark field.

---

## 10. Module 6 — Diesel Module

Diesel is the highest accountability concern raised by the owner. The demo implements the **complete traceability flow: Pump → Tanker → Machine → Site**, plus base support for rented-with-fuel credit reconciliation and truck pump refueling.

> A **dedicated follow-up discussion** with the client is still pending to finalize corner cases. The design below reflects current understanding and will be refined after that discussion.

### 10.1 Entities Involved
- **Petrol Pump** (external source)
- **Diesel Tanker** (mobile; one or many, each operated by a tanker operator)
- **Machine** (consumer; owned / rented without fuel / rented with fuel)
- **Truck** (owned vehicle, typically refueled directly at pumps, not via tanker)

### 10.2 Flow 1 — Diesel Tanker Pickup (Pump → Tanker)
Entered by: **Tanker Operator**

Fields:
- Date, time
- Petrol pump name
- Tanker name / number
- Quantity filled (litres)
- Rate per litre, total cost
- Slip number, invoice reference
- Payment mode (cash / credit / card)
- Remarks

System action: increases tanker's current diesel balance.

### 10.3 Flow 2 — Tanker Dispensing (Tanker → Machine)
Entered by: **Tanker Operator** (mobile-friendly so they can enter on the go)

Fields:
- Date, time
- Tanker name (auto-filled)
- Current site (dropdown — the tanker may serve multiple sites in a day)
- Machine name / code (dropdown from machine master)
- Quantity dispensed (litres)
- Meter reading (optional)
- Slip / acknowledgement number
- Photo of slip (optional)
- Remarks

System actions:
- Reduces tanker's balance.
- Logs dispense event against the selected machine and site.

### 10.4 Flow 3 — Rented Machine WITHOUT Fuel
- Machine is marked in master as "Rented without fuel".
- Standard Flow 2 applies — tanker dispenses normally.
- Consumption is logged against the machine; no special billing treatment.

### 10.5 Flow 4 — Rented Machine WITH Fuel (Credit Reconciliation)
This is the owner's key concern.

**Scenario:** The rental party provides the machine with a diesel credit (e.g., ₹1,000 advance). The machine works across one or more of our sites. At end of day, actual consumption is computed. The difference is payable/recoverable.

**Data captured:**
- **Credit Entry:** Machine, date, credit amount (₹), credit litres, rental party, reference slip.
- **Daily Consumption:** For each day, for each site the machine worked on, quantity consumed and cost — either from our tanker (Flow 2) or self-reported by the machine operator.
- **End-of-Day Reconciliation:**
  - Actual cost = sum of consumption × rate
  - Net payable/recoverable = Actual − Credit
  - System shows running balance per machine.

**Example (from meeting):**
- Credit = ₹1,000
- Actual = ₹2,400 across 3 sites in one day
- **Net billable = ₹1,400**

### 10.6 Flow 5 — Own Truck at Petrol Pump
Entered by: **Truck driver** (mobile) or **Camp In-charge / Admin** if drivers don't have the app in demo.

Fields:
- Date, time
- Truck name / number
- Petrol pump name
- Quantity (litres)
- Rate, total cost
- Slip / bill number
- Photo of slip (optional)
- Odometer reading (optional)
- Remarks

System action: logs consumption against the truck; no tanker involvement.

### 10.7 Diesel Reports & Reconciliation
- Daily diesel in (pump → tanker + pump → truck).
- Daily diesel out (tanker → machines, by site).
- Per-machine consumption log.
- Per-site consumption log.
- Rented-with-fuel machine ledger (credit vs actual vs net).
- Tanker balance report (opening / pickup / dispense / closing).

### 10.8 Verification
All diesel entries are verifier-reviewable. Tanker operator entries are typically verified by Camp In-charge or Admin.

---

## 11. Module 7 — Reports Module (Web)

Accessible to Management, Senior Engineer, and Admin. All reports are exportable (PDF / Excel).

### 11.1 Daily Progress Report (DPR)
- Site, date
- Materials dumped (from §8 entries): grade, quantity, chainage, layer
- Work progress updates (from §9)
- Diesel consumed (from §10)
- Labor deployed (from §6.6)
- Machines on site
- Remarks / issues

### 11.2 Material Reports
- Camp inward / outward by material & date range
- Plant inward / production / outward by material & date range
- Site-wise material dumped by layer, chainage, date range
- Planned vs Actual (variance %)

### 11.3 Diesel Report
- Per-pump pickup report
- Per-tanker log
- Per-machine consumption
- Per-site consumption
- Rented-machine credit reconciliation ledger
- Truck-at-pump log

### 11.4 Work Progress Report
- Chainage-wise, layer-wise progress snapshot
- Weekly / monthly progress deltas
- Cumulative site progress %

### 11.5 Labor Report
- Daily labor count by party, site, purpose

### 11.6 Variance & Exception Report
- Estimated vs Actual material consumption (uses Admin estimation values)
- Entries still unverified beyond configured days
- Low-stock alerts (camp / plant)

### 11.7 Report Frequency
Daily, weekly, monthly — configurable in Admin (§5.7).

---

## 12. Module 8 — Dashboard (Web, Owner/Management ONLY)

Per client instruction, the dashboard is visible **only to the owner / upper management**. Other roles use their module screens and Reports.

### 12.1 KPIs Displayed
- **Sites overview:** active sites, % progress per site
- **Materials today:** dumped qty by site & layer
- **Camp stock levels:** current closing stock per material, low-stock flags
- **Plant production today:** output qty by product type
- **Diesel today:** issued by tanker, by site, by machine
- **Rented-with-fuel ledger:** outstanding recoverable / payable
- **Labor on field today:** count by site
- **Variance flags:** items exceeding variance tolerance
- **Unverified entries:** count by module
- **Work progress:** cumulative % per site, top-5 chainage segments in progress

### 12.2 Dashboard Views
- Consolidated all-sites view
- Per-site drill-down
- Per-camp / per-plant drill-down
- Date filter (today / this week / this month / custom range)

---

## 13. Verification Workflow — System-Wide

| Entry Type | Entered By | Verified By |
|---|---|---|
| On-site material dumping | Supervisor | Senior Engineer |
| Camp inward/outward/diesel/labor | Camp In-charge | Senior Engineer or Admin |
| Plant inward/production/outward | Plant In-charge | Admin or Management |
| Work progress | Senior Engineer | Management (optional) |
| Diesel tanker entries | Tanker Operator | Camp In-charge or Admin |
| Truck pump refueling | Driver / Camp In-charge | Admin |

**Rules:**
- Entries are never blocked at submission time.
- Unverified entries are flagged in Reports and on the Dashboard.
- Verifier can mark "Verified" or add a remark requesting correction.
- Full audit log of who entered, who verified, timestamps, and edits.

---

## 14. Materials List Appendix (Pre-loaded Demo Data)

### 14.1 Kalol Sanand Site
**Camp:** Khatraj Chokdi

**Camp / Plant Inward Materials:**
| Material | Unit | Grades / Sizes |
|---|---|---|
| Aggregate | MT | 6mm, 10mm, 20mm, 25–40mm, Storm dust |
| Cement | MT & Bags | — |
| Sand | MT | — |
| Steel | MT | Dia 8, 10, 12, 16, 20, 25, 32 mm |
| Emulsion | MT | SS1, RS1 |

**Site Dumping Materials:**
- GSB
- WMM
- BM
- DBM
- BC
- SDBC
- Pipe — Dia 300, 600, 900, 1200 | Unit: numbers

**Camp Outward (Khatraj Chokdi → Site):**
- Steel
- Emulsion
- Concrete mix — M10, M15, M20, M25, M30, M35, M40 (m³)
- *(Concrete preparation logic — to be provided later by client.)*

### 14.2 Magodi Plant

**Plant Inward Materials:**
| Material | Unit | Grades / Sizes |
|---|---|---|
| Aggregate | MT | 6mm, 10mm, 20mm, 25–40mm, Storm dust |
| Cement | MT & Bags | — |
| Sand | MT | — |
| Steel | MT | Dia 8, 10, 12, 16, 20, 25, 32 mm |
| Emulsion | MT | SS1, RS1 |
| Bitumen | MT | VG30, VG40 |
| LDO / FO | Litres | — |

**Plant Outward:**
- BM
- DBM
- BC
- SDBC
- MSS
- Bitumen (VG30, VG40)
- Steel
- Emulsion

---

## 15. Open Items / To Be Provided by Client

1. **Diesel logic follow-up meeting** — finalize tanker multi-site behavior, rented-with-fuel reconciliation edge cases, and truck pump refueling workflow.
2. **Concrete preparation logic** — mix ratios, input-to-output transformation rules for Camp RMC and Plant RMC / bituminous mixes. Entries for §6.5 and §7.2 may evolve after this is received.
3. **Work progress entry method** — client to confirm **Method A** (recommended) or **Method B** (or hybrid) from §9.1.
4. **Segment length** for progress entry (default 100 m) — client to confirm.
5. **Historical data upload** — whether existing Excel/WhatsApp records should be migrated into the demo.
6. **Report export formats** — confirm PDF + Excel is sufficient.

---

## 16. Demo Delivery Checklist

- [ ] Admin Panel: masters, site/camp/plant CRUD, users, estimation, thresholds
- [ ] Camp Module: inward, outward, stock, RMC, diesel, labor
- [ ] Plant Module: inward, production, outward, stock
- [ ] On-Site Dumping App: mobile, manual chainage, optional photo
- [ ] Senior Engineer Work Progress (Method A): web, chainage segments, layer %
- [ ] Diesel Module: pump → tanker → machine; rented-with-fuel ledger; truck pump
- [ ] Reports Module: DPR, material, diesel, progress, labor, variance
- [ ] Dashboard: owner-only, consolidated + drill-down
- [ ] Verification workflow across all modules
- [ ] Machine master (no usage tracking) populated for diesel issue
- [ ] Pre-load Kalol Sanand + Khatraj Chokdi + Magodi + material masters
- [ ] Ability to add new sites / camps / plants / materials / users (not hardcoded)
