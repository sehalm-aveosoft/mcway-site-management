# Product Requirements Document (PRD)
## McWay Site Management — Demo v1

**Author:** Product Management
**Date:** 2026-04-15
**Status:** Draft for Engineering & Design Handoff
**Platform:** Web (Admin, Camp, Plant, Senior Engineer, Reports, Dashboard) + Mobile App (On-Site Supervisor, Tanker Operator)
**Tech Stack:** Odoo 17/18 (⚠️ TBD — edition to confirm) + companion mobile app
**Reference Docs:** `_Documentation in detail.md`, `McWay_Meeting_Notes_2026-04-14.md`, `McWay_Demo_Documentation.md`

---

## Table of Contents
1. [Executive Summary & Goals](#1-executive-summary--goals)
2. [Target Personas](#2-target-personas)
3. [User Stories & Features](#3-user-stories--features)
4. [User Flows](#4-user-flows)
5. [Information Architecture & Data Model](#5-information-architecture--data-model)
6. [Success Metrics (KPIs)](#6-success-metrics-kpis)
7. [Risks & Constraints](#7-risks--constraints)
8. [Release Plan](#8-release-plan)
9. [Open Questions](#9-open-questions)

---

## 1. Executive Summary & Goals

### 1.1 The Problem
McWay runs road (and future bridge/building) construction projects across multiple sites, camps, and plants. Operations today rely on **paper records, WhatsApp messages, and Excel sheets**, causing:
- No real-time visibility into work progress, machinery, diesel, and material movement.
- Delayed or missing reports that block cost control and early problem detection.
- Untracked idle machines, breakdowns, and over-consumption of materials.
- Diesel leakage / accountability gaps — the owner cannot verify every litre from pump → tanker → machine → site.
- Data-entry operators filter field reality, reducing trust in the data.

### 1.2 The Solution
An **Odoo-based ERP** tailored for road-construction site management, with a companion mobile app for field entries. Every user enters their own data directly; a **Verifier** role reviews entries without blocking them. The system covers masters, site/camp/plant inventory and production, on-site material dumping, senior-engineer work progress, diesel traceability, reports, and an owner-only dashboard.

### 1.3 Primary Goals
| # | Goal | Measurable Target |
|---|---|---|
| G1 | Real-time operational visibility for owner | Owner dashboard updated within 15 min of any field entry |
| G2 | Diesel accountability end-to-end | 100% of diesel litres traceable pump → machine → site |
| G3 | Accurate material tracking with <5% variance | Dumped-qty variance vs plan ≤ 5% by month 3 |
| G4 | Eliminate data-entry middlemen | 100% of entries submitted by the role performing the activity |
| G5 | Reduce report preparation time | DPR available by 9 AM next morning vs current 2–3 day lag |

### 1.4 Non-Goals (Demo)
- Machine usage tracking (ON/OFF/Breakdown, idle-hours analytics) — deferred to phase 2.
- Financial accounting, vendor billing, GST posting.
- Theft detection logic / AI anomaly detection.
- Bridge and Building modules (Road only in demo).
- GPS / telemetry integrations.
- Multi-company / multi-tenant.

---

## 2. Target Personas

### 2.1 Rajesh — Owner / Upper Management
- **Role:** Company owner, oversees all sites.
- **Context of use:** Office (desktop) and on the move (tablet/mobile browser).
- **Goals:** Real-time visibility, cost control, diesel accountability, catch problems early.
- **Pain points today:** Reports arrive 2–3 days late; cannot tell which tanker dispensed how much where; suspects diesel leakage.
- **Tech comfort:** Medium; uses WhatsApp, Excel, basic dashboards.
- **When used:** Morning review, end-of-day check, ad-hoc when exceptions flagged.

### 2.2 Kiran — Senior Engineer
- **Role:** Site-level technical lead; owns work progress, verifies field data.
- **Context:** Mostly site office (laptop/web); occasional field visits.
- **Goals:** Track chainage-wise progress, verify supervisor entries, hit monthly targets.
- **Pain points:** No single source of truth for progress; manual Excel reconciliation.
- **Tech comfort:** High.
- **When used:** Daily — morning planning, midday verification, evening progress entry.

### 2.3 Mahesh — Site Supervisor
- **Role:** On-ground execution; receives and directs material dumping.
- **Context:** Entirely on site, outdoors, often poor connectivity.
- **Goals:** Quickly log dumping events without leaving the work front.
- **Pain points:** Currently writes on paper or WhatsApps photos of slips; data gets lost.
- **Tech comfort:** Low–medium; comfortable with mobile apps in Gujarati/Hindi.
- **When used:** Every time a vehicle arrives at the chainage (multiple times per day).

### 2.4 Suresh — Camp In-charge
- **Role:** Runs the camp — receives inward, dispatches outward, manages stock, diesel, labor.
- **Context:** Camp office (desktop / laptop) with occasional mobile use.
- **Goals:** Maintain accurate stock, respond to site demands, log diesel and labor.
- **Pain points:** Manual registers, reconciliation errors, lost challans.
- **Tech comfort:** Medium.
- **When used:** Continuously through the day as vehicles arrive/depart.

### 2.5 Dinesh — Plant In-charge
- **Role:** Runs the plant — raw material inward, RMC/bituminous production, outward dispatch.
- **Context:** Plant control room, desktop-based.
- **Goals:** Track raw consumption vs output, dispatch to camps/sites on schedule.
- **Pain points:** Mix calculation done on spreadsheets; no real-time stock view.
- **Tech comfort:** Medium–high.
- **When used:** Continuously; peak during morning and afternoon batches.

### 2.6 Aarti — Tanker Operator
- **Role:** Drives the diesel tanker; picks up from pump, dispenses to machines at sites.
- **Context:** Mobile-only; in a moving vehicle; often poor connectivity.
- **Goals:** Log pickups and dispenses quickly without handwritten slips.
- **Pain points:** Paper slips get lost; no proof of dispense.
- **Tech comfort:** Low; needs very simple mobile UX with minimum taps and offline support.
- **When used:** Throughout the day across multiple sites.

---

## 3. User Stories & Features

Features grouped by epic/module. Priority: **P0** = must for demo, **P1** = should, **P2** = nice-to-have.

### Epic A — Admin Panel & Masters

| ID | User Story | Priority |
|---|---|---|
| F-A01 | As an Admin, I want to create/edit Sites with chainage limits so that field modules know the scope. | P0 |
| F-A02 | As an Admin, I want to create/edit Camps and link them to one or more Sites. | P0 |
| F-A03 | As an Admin, I want to create/edit Plants and link them to Camps/Sites they supply. | P0 |
| F-A04 | As an Admin, I want to configure the Material Master (name, grade, unit, density) so no free-text exists elsewhere. | P0 |
| F-A05 | As an Admin, I want a Machine Master (name, category, ownership type, fuel average). | P0 |
| F-A06 | As an Admin, I want User & Role administration with site/camp/plant assignment. | P0 |
| F-A07 | As an Admin, I want to set estimation (length × width × thickness × density) and variance thresholds. | P1 |
| F-A08 | As an Admin, I want Rental Party & Petrol Pump masters for diesel tracking. | P0 |
| F-A09 | As an Admin, I want an audit log of all edits and verifications. | P0 |

**Functional Requirements (selected):**
- F-A01.1: Site fields — name, location (lat/long optional), type (Road/Bridge/Building), chainage_start_m, chainage_end_m, assigned senior engineer, status.
- F-A01.2: Chainage values stored in **metres (integer)**; UI accepts km with decimal, converts internally.
- F-A04.1: Material fields — name, category, grade, UoM (linked to Odoo UoM), density, source type (plant/camp/vendor).
- F-A04.2: Material unit conversions explicit (e.g., 1 bag cement = 50 kg). Defined in Odoo UoM categories.

**Non-Functional:**
- All master screens load within 2s for up to 10k records.
- Full audit trail retained 5 years.
- Role-based visibility enforced via Odoo `ir.model.access.csv` + record rules.

**Acceptance:**
- Given an Admin, when they create a Site with chainage 0–27000 m, then the Site is usable in all downstream modules within 60 seconds.

---

### Epic B — Camp Module (Web)

| ID | User Story | Priority |
|---|---|---|
| F-B01 | As a Camp In-charge, I want to record material inward with slip/royalty/weights so stock updates automatically. | P0 |
| F-B02 | As a Camp In-charge, I want to record material outward to a site with vehicle + slip details. | P0 |
| F-B03 | As the system, I want to auto-maintain opening/inward/outward/closing stock per material. | P0 |
| F-B04 | As a Camp In-charge, I want to log camp-level RMC production with inputs consumed and output produced. | P1 |
| F-B05 | As a Camp In-charge, I want to log labor arrival (party, count, purpose). | P0 |
| F-B06 | As a Camp In-charge, I want shuttering special tracking (inward, issue, return, damage, loss). | P1 |

**Functional Requirements:**
- F-B01.1: Net weight = Gross − Tare, auto-calculated, non-editable.
- F-B01.2: Source dropdown: Plant (list from master) / Vendor / Quarry.
- F-B01.3: Photo optional, max 5 MB, auto-compressed to 1080p.
- F-B02.1: Destination must be an active Site linked to this Camp.
- F-B03.1: Stock view shows real-time closing balance; low-stock flag when ≤ threshold (configurable).

**Non-Functional:**
- Inward/outward form submission ≤ 1s.
- Concurrent use by 10 camp users without lock contention (⚠️ TBD expected load).

**Acceptance:**
- Given a camp has 100 MT of aggregate, when 20 MT is dispatched outward, then closing stock = 80 MT and the outward appears on the linked site's expected inward list within 1 min.

---

### Epic C — Plant Module (Web)

| ID | User Story | Priority |
|---|---|---|
| F-C01 | As a Plant In-charge, I want to record raw material inward. | P0 |
| F-C02 | As a Plant In-charge, I want to record a production batch (output type, qty, inputs consumed, diesel). | P0 |
| F-C03 | As a Plant In-charge, I want to record outward to a Camp or Site. | P0 |
| F-C04 | As the system, I want to deduct input stock and add output stock for each production batch. | P0 |

**Functional Requirements:**
- F-C02.1: Implemented via Odoo **MRP** — each mix grade (BM, DBM, BC, SDBC, MSS, M10–M40, SS1, RS1) is a **manufactured product** with a Bill of Materials (BoM).
- F-C02.2: BoMs are editable by Admin (⚠️ concrete prep ratios TBD — client to provide).
- F-C03.1: Outward uses Odoo internal transfer (plant warehouse → camp warehouse or plant → site location).
- F-C02.3: Diesel consumed by production recorded against the plant's diesel balance.

**Non-Functional:**
- Production batch save ≤ 2s even with 10-line BoMs.

**Acceptance:**
- Given a BM batch consumes 9.5 MT aggregate + 0.5 MT bitumen and produces 10 MT BM, when the batch is confirmed, then raw stock decreases and BM stock increases exactly by those amounts.

---

### Epic D — On-Site Dumping (Mobile)

| ID | User Story | Priority |
|---|---|---|
| F-D01 | As a Supervisor, I want to log a dumping event with minimum taps on mobile. | P0 |
| F-D02 | As a Supervisor, I want to enter chainage manually (start & end in m or km). | P0 |
| F-D03 | As a Supervisor, I want to optionally attach a photo of the slip. | P0 |
| F-D04 | As a Supervisor, I want the app to work offline and sync when connectivity returns. | P0 |
| F-D05 | As a Supervisor, I want the material list pre-filtered to what can come to my site. | P1 |

**Functional Requirements:**
- F-D01.1: Fields — datetime (auto, editable), material, grade, chainage_start, chainage_end, side (LHS/RHS/Lane/Median), activity/layer, source, source_reference (optional), vehicle_no, party_name, slip_no, royalty_no, gross_wt, tare_wt, net_wt (auto), photo (optional), remarks.
- F-D01.2: Net weight auto-calculated; no free-text material/grade.
- F-D02.1: Chainage accepts `12.5 km` or `12500 m`; stored as metres integer.
- F-D04.1: Offline queue persists 7 days of entries; sync on Wi-Fi or cellular.
- F-D04.2: Conflict policy on sync — last-write-wins at entry level; entries are immutable after verification.

**Non-Functional:**
- App install size ≤ 50 MB.
- Form submission ≤ 500ms on 4G; offline submission ≤ 200ms.
- UI language: English + ⚠️ TBD Gujarati/Hindi.
- Works on Android 10+; iOS 14+ (⚠️ confirm iOS needed).

**Acceptance:**
- Given a supervisor logs a dumping event offline with a photo, when they return to coverage, then the entry and photo are synced within 2 minutes and appear in Senior Engineer's verification queue.

---

### Epic E — Senior Engineer Work Progress (Web)

| ID | User Story | Priority |
|---|---|---|
| F-E01 | As a Senior Engineer, I want to record layer-wise progress per chainage segment. | P0 |
| F-E02 | As a Senior Engineer, I want to visualize site progress as a color-coded chainage strip. | P0 |
| F-E03 | As a Senior Engineer, I want to verify supervisor dumping entries from a queue. | P0 |

**Functional Requirements:**
- F-E01.1: Method A (recommended per demo doc §9.1): segment length default 100 m (configurable per site); per segment per layer, status = Not Started / In Progress (%) / Complete.
- F-E01.2: Photos (multiple, optional); remarks; date.
- F-E02.1: Strip view — X-axis chainage, rows for each layer (GSB, WMM, BM, DBM, BC, SDBC, CC), cell color = status.
- F-E03.1: Verification queue lists unverified entries with Verify / Request Correction actions and a remark field. **Verification does not block entries.**

**Non-Functional:**
- Strip rendering for 27 km site with 7 layers (~1,890 cells) ≤ 1.5s.

**Acceptance:**
- Given 10 unverified dumping entries exist, when the SE opens the verification queue, then all 10 appear sorted by date and can be verified in ≤ 2 clicks each.

---

### Epic F — Diesel Module

| ID | User Story | Priority |
|---|---|---|
| F-F01 | As a Tanker Operator, I want to log a diesel pickup from a pump. | P0 |
| F-F02 | As a Tanker Operator, I want to log each dispense to a machine at a site. | P0 |
| F-F03 | As Admin, I want to track a rented-with-fuel machine's credit vs actual consumption. | P0 |
| F-F04 | As a Truck Driver / Camp In-charge, I want to log truck refueling directly at a pump. | P0 |
| F-F05 | As Owner, I want a per-site and per-machine diesel consumption report. | P0 |

**Functional Requirements:**
- F-F01.1: Pickup fields — date/time, pump (master), tanker (master), litres, rate, total, slip_no, invoice_ref, payment_mode, remarks.
- F-F02.1: Dispense fields — date/time, tanker (auto), site, machine (master), litres, meter_reading (optional), slip_no, photo (optional), remarks.
- F-F02.2: Tanker balance = Σ pickups − Σ dispenses; validated non-negative on dispense.
- F-F03.1: Credit record — machine, rental party, date, credit_amount (₹), credit_litres, reference_slip.
- F-F03.2: Daily reconciliation: actual_cost = Σ(litres × rate); net = actual − credit; system shows running ledger.
- F-F04.1: Truck refueling — truck, pump, litres, rate, total, slip_no, photo, odometer (optional).

**Non-Functional:**
- Dispense entry on mobile ≤ 15 seconds end-to-end (optimised tap flow).
- Offline support same as Epic D.

**Acceptance:**
- Given a rented-with-fuel machine has ₹1,000 credit and consumes ₹2,400 across 3 sites on the same day, when the day closes, then the ledger shows net billable of ₹1,400.

---

### Epic G — Reports Module (Web)

| ID | User Story | Priority |
|---|---|---|
| F-G01 | As Management, I want a Daily Progress Report (DPR) exportable as PDF. | P0 |
| F-G02 | As Management, I want material, diesel, progress, labor, and variance reports. | P0 |
| F-G03 | As Management, I want date-range filters and export to Excel. | P0 |

**Functional Requirements:**
- F-G01.1: DPR sections — materials dumped, work progress, diesel consumed, labor, machines on site, remarks. Generated via Odoo QWeb.
- F-G02.1: Variance report compares estimated (Admin) vs actual (field entries) and flags entries beyond threshold.
- F-G03.1: All reports exportable to PDF and XLSX.

**Acceptance:**
- Given a completed working day, when the user generates the DPR by 9 AM next day, then all fields are auto-populated from the prior day's entries.

---

### Epic H — Dashboard (Owner Only)

| ID | User Story | Priority |
|---|---|---|
| F-H01 | As the Owner, I want a single consolidated dashboard for all sites. | P0 |
| F-H02 | As the Owner, I want to drill down into a specific site / camp / plant. | P0 |
| F-H03 | As the Owner, I want unverified-entry flags and variance flags surfaced. | P0 |

**Functional Requirements:**
- F-H01.1: KPI tiles — active sites, dumped today, stock levels, production today, diesel today, rented-with-fuel outstanding, labor today, unverified count, cumulative progress %.
- F-H01.2: Refresh cadence: every 15 minutes (⚠️ confirm real-time vs batch).
- F-H03.1: Click-through from each flag to the underlying record list.

**Non-Functional:**
- Dashboard load ≤ 3s.

---

### Epic I — Verification Workflow (Cross-Cutting)

| ID | User Story | Priority |
|---|---|---|
| F-I01 | As a Verifier, I want a single queue of entries to verify across my scope. | P0 |
| F-I02 | As a Verifier, I want entries to remain usable even before verification. | P0 |
| F-I03 | As Admin, I want a full audit log of who entered / verified / edited records. | P0 |

---

## 4. User Flows

### 4.1 Supervisor — Log a Dumping Event (Mobile)
**Trigger:** Vehicle arrives at chainage with material.
1. Supervisor opens app, lands on Home with "New Dumping" primary button.
2. Taps "New Dumping" → form opens with site pre-filled.
3. Selects material, grade, source (Camp/Plant/Vendor).
4. Enters chainage_start, chainage_end, side, layer.
5. Enters vehicle number, slip, royalty, gross, tare (net auto-calculates).
6. (Optional) Taps camera, captures slip photo.
7. Taps Submit → entry queued locally, uploads when online.
8. Returns to Home with success toast + entry in "My Recent" list.

**Alternate paths:**
- No connectivity → entry saved to queue, badge shows "N pending sync".
- Validation error (e.g., end < start chainage) → inline error.
- Duplicate slip number → warning but not blocked.

### 4.2 Tanker Operator — Dispense Diesel (Mobile)
**Trigger:** Machine at site needs fuel.
1. Opens app → Home shows "Dispense" and "Pickup" buttons + current tanker balance.
2. Taps "Dispense" → selects Site (dropdown), Machine (dropdown filtered to that site).
3. Enters litres + slip_no.
4. (Optional) Photo of slip.
5. Submit → tanker balance decrements locally; syncs when online.

### 4.3 Senior Engineer — Verify Entries (Web)
1. Logs in, lands on "To Verify" dashboard tile showing pending count.
2. Clicks tile → queue of unverified entries across modules in scope.
3. Clicks an entry → side panel with all fields + photo.
4. Clicks Verify (or Request Correction with remark).
5. Entry moves out of queue; submitter notified.

### 4.4 Owner — Morning Review (Web)
1. Opens Dashboard → sees last 24h KPIs.
2. Spots red variance flag on Site A → drills in.
3. Sees dumping vs plan table → identifies over-consumption.
4. Opens DPR for Site A → exports PDF for morning meeting.

### 4.5 Plant In-charge — Record a Production Batch
1. Opens Plant module → Production tab → New Batch.
2. Selects output product (e.g., BM), grade, quantity.
3. System loads BoM → pre-fills expected input consumption.
4. Adjusts actual consumption if different; enters diesel used, operator, shift.
5. Confirms → stock auto-adjusts; batch appears in production log.

### 4.6 Rented-with-Fuel Reconciliation (Daily)
1. At day-end, Admin opens Rented-Machine Ledger.
2. System shows per-machine credit balance, today's consumption across sites, net payable.
3. Admin reviews, marks reconciled, exports statement for the rental party.

---

## 5. Information Architecture & Data Model

### 5.1 Core Entities (Simplified)

| Entity | Key Fields | Relationships |
|---|---|---|
| Site | id, name, type, chainage_start_m, chainage_end_m, status, senior_engineer_id | many-to-many Camps; one-to-many DumpingEntries; one-to-many ProgressEntries |
| Camp | id, name, location, in_charge_id | many-to-many Sites; many-to-many Plants; one-to-many CampInward/Outward |
| Plant | id, name, location, in_charge_id | many-to-many Camps; many-to-many Sites; one-to-many ProductionBatches |
| Material | id, name, category, grade, uom_id, density, source_types | referenced everywhere |
| Machine | id, name, category, ownership_type, fuel_avg, rental_party_id (nullable) | one-to-many DieselDispenses |
| RentalParty | id, name, contact, credit_balance | one-to-many Machines; one-to-many DieselCredits |
| PetrolPump | id, name, location | one-to-many DieselPickups, TruckRefuels |
| Tanker | id, name, current_balance | one-to-many Pickups/Dispenses |
| User | id, name, role, assigned_site_ids, assigned_camp_ids, assigned_plant_ids | — |
| DumpingEntry | id, site_id, datetime, material_id, grade, chainage_start, chainage_end, side, layer, source, vehicle, slip, royalty, gross, tare, net, photo, state | submitted_by, verified_by |
| CampInward | id, camp_id, material_id, source, vehicle, slip, royalty, gross, tare, net, state | — |
| CampOutward | id, camp_id, destination_site_id, material_id, qty, vehicle, slip, state | — |
| ProductionBatch | id, plant_id, output_material_id, qty, inputs[], diesel_consumed, operator, shift | BoM reference |
| DieselPickup | id, pump_id, tanker_id, litres, rate, slip, invoice | — |
| DieselDispense | id, tanker_id, site_id, machine_id, litres, slip, photo | — |
| DieselCredit | id, machine_id, rental_party_id, amount, litres, date, reference | — |
| TruckRefuel | id, truck_id, pump_id, litres, rate, slip, photo, odometer | — |
| ProgressEntry | id, site_id, chainage_start, chainage_end, side, layer, status, percent, date, photos[], remarks, state | — |
| LaborEntry | id, camp_or_site_id, party_name, count, purpose, date | — |
| AuditLog | id, entity, entity_id, action, user_id, timestamp, diff | — |

### 5.2 State Machines

**DumpingEntry / CampInward / CampOutward / ProgressEntry / DieselDispense:**
`draft → submitted → verified → locked`
- submitted: visible in reports & dashboard, usable.
- verified: Verifier has confirmed.
- locked: period closed by Admin; no edits.

**ProductionBatch:** `planned → in_progress → done → locked`

### 5.3 Access Control Matrix (Simplified)

| Role | Masters | Site Entry | Camp Entry | Plant Entry | Progress | Diesel | Dashboard | Verify |
|---|---|---|---|---|---|---|---|---|
| Admin | CRUD | R | R | R | R | R | — | ✓ |
| Owner | R | R | R | R | R | R | ✓ | ✓ |
| Senior Engineer | R | R (own sites) | R | — | CRUD (own sites) | R | — | ✓ (own sites) |
| Supervisor | R | CRUD (own) | — | — | — | — | — | — |
| Camp In-charge | R | — | CRUD (own camp) | — | — | R/W (camp diesel) | — | — |
| Plant In-charge | R | — | — | CRUD (own plant) | — | R/W (plant diesel) | — | — |
| Tanker Operator | R | — | — | — | — | CRUD (own tanker) | — | — |
| Verifier | R | R (scope) | R (scope) | R (scope) | R (scope) | R (scope) | — | ✓ |

Scope is enforced via Odoo record rules.

---

## 6. Success Metrics (KPIs)

### 6.1 North-Star Metric
**Diesel traceability coverage** — % of total diesel litres with a complete pump → tanker → machine → site trail. **Target: 95% by day 90.**

### 6.2 Leading Indicators
| KPI | Target (Day 30) | Target (Day 90) | Source |
|---|---|---|---|
| Daily active field users | 70% of provisioned | 90% | Odoo login logs |
| Dumping entries logged per day (digital) | 80% of paper baseline | 100% | DumpingEntry count vs manual baseline |
| Avg. time from event to digital entry | ≤ 30 min | ≤ 10 min | event timestamp vs created_at |
| Verification turnaround (submit → verified) | ≤ 24h | ≤ 4h | state transition logs |
| Mobile offline-sync success rate | ≥ 98% | ≥ 99.5% | client telemetry |

### 6.3 Lagging Indicators
| KPI | Target (Day 90) | Target (Day 180) |
|---|---|---|
| DPR availability by 9 AM next day | 95% | 99% |
| Material variance (plan vs actual) | ≤ 7% | ≤ 5% |
| Unverified-entry backlog older than 48h | < 5% of entries | < 1% |
| Owner NPS on dashboard usefulness | ≥ 7 | ≥ 8 |
| Rented-with-fuel reconciliation accuracy | 98% | 99.5% |

### 6.4 Business Outcomes (to be measured ⚠️ baselines TBD)
- Reduction in diesel cost / km of road laid.
- Reduction in monthly material over-consumption ₹.
- Time saved by owner vs manual Excel consolidation (hours/week).

---

## 7. Risks & Constraints

### 7.1 Technical Risks
| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| Odoo MRP may not fit bituminous/concrete mix edge cases | High | Medium | Prototype one BoM early; fall back to custom module if required |
| Offline-sync conflicts on mobile | Medium | Medium | Entry-level immutability after verification; last-write-wins before |
| Poor connectivity at site → delayed dashboard | High | High | 15-min refresh; offline queue + retry |
| Photo storage costs with uncompressed images | Medium | High | Server-side compression to 1080p; 5 MB per-file cap |
| Odoo scaling for 50+ concurrent field users | Medium | Low | Load test before go-live; horizontal scaling plan |

### 7.2 Product Risks
| Risk | Impact | Mitigation |
|---|---|---|
| Field users (supervisors, tanker ops) reject mobile app due to complexity | High | UX with minimum taps; 2-day onsite training; language localization |
| Verifier bottleneck if SE is overloaded | Medium | Bulk verify; delegation; dashboards for pending counts |
| Client expands scope during demo ("just add machine usage") | Medium | Strict demo scope doc; change-request process |
| Concrete-mix logic unclear | Medium | Placeholder BoMs; update after client input |

### 7.3 Business Risks
| Risk | Impact | Mitigation |
|---|---|---|
| Owner disengages after demo | High | Weekly demo checkpoints; show data they care about first (diesel) |
| Rental-party pushback on credit ledger transparency | Low–Med | Read-only statement export for external sharing |
| Regulatory — royalty number format | Low | Store as reference string; no validation in demo |

### 7.4 Dependencies
- Client to provide concrete/bituminous mix BoM ratios.
- Client to provide rental parties, petrol pumps, machines list for master seeding.
- Client to confirm Gujarati/Hindi localization need.
- Hosting infrastructure (Odoo server, mobile backend, photo storage) — ⚠️ TBD.

### 7.5 Assumptions
- Field users will have Android devices (≥ Android 10).
- Sites have intermittent 4G; full offline-first is acceptable for mobile.
- The owner is available for weekly review meetings.
- Odoo edition licensed is sufficient for required modules.

---

## 8. Release Plan

### 8.1 Milestones
| Milestone | Contents | Target Date (⚠️ TBD) | Owner |
|---|---|---|---|
| M1 — Foundations | Masters, Sites/Camps/Plants, Users & Roles, Auth | Week 3 | Backend |
| M2 — Camp + Plant | Inward/Outward, Production, Stock, Labor | Week 6 | Backend + Web UI |
| M3 — Mobile App Alpha | On-site Dumping + Tanker Diesel (offline) | Week 8 | Mobile |
| M4 — SE Progress + Dashboard | Progress entry, strip view, owner dashboard | Week 10 | Web UI |
| M5 — Reports + Verification | DPR, variance, verification queue, audit | Week 11 | Full-stack |
| M6 — Demo to Client | End-to-end demo on Kalol Sanand + Khatraj + Magodi | Week 12 | PM + All |

### 8.2 Effort Estimate (T-shirt)
- Masters + Admin: **M**
- Camp + Plant (with Odoo reuse): **L**
- Mobile App (offline-first): **XL**
- Diesel (with credit reconciliation): **L**
- SE Progress: **M**
- Dashboard + Reports: **M**
- Overall demo: **XL (~12 weeks)** assuming 2 backend + 2 mobile + 1 web + 1 QA + 1 PM.

### 8.3 Go-to-Market Checklist
- [ ] 2-day onsite training for supervisors, tanker operators, camp/plant in-charges.
- [ ] Printed quick-reference cards (laminated) in Gujarati.
- [ ] Pilot on Kalol Sanand for 2 weeks before scale-out.
- [ ] Daily standup during pilot with client owner.
- [ ] Rollback plan: keep paper registers in parallel for first 2 weeks.

### 8.4 Rollback / Kill-Switch
- Feature flags per module; disable on error.
- Nightly DB backup + 7-day retention minimum.
- Manual export to Excel available at any time.

---

## 9. Open Questions

| # | Question | Owner | Needed By |
|---|---|---|---|
| Q1 | Odoo 17 or 18? Community or Enterprise? | Tech Lead | Week 0 |
| Q2 | Mobile app — native (Flutter/RN) or PWA? iOS needed? | Tech Lead + Client | Week 1 |
| Q3 | Concrete & bituminous mix BoM ratios | Client | Week 4 |
| Q4 | SE progress method — A (recommended) or B or hybrid | Client | Week 2 |
| Q5 | Default segment length for progress (100 m?) | Client | Week 2 |
| Q6 | Localization — Gujarati / Hindi required? | Client | Week 2 |
| Q7 | Dashboard refresh — real-time or 15-min batch? | Client | Week 3 |
| Q8 | Historical Excel data migration — in scope? | Client | Week 3 |
| Q9 | Rounding rules for weights/litres/m³ | Client | Week 4 |
| Q10 | Hosting — on-prem / cloud (AWS/Azure/OVH) / Odoo.sh | Client + Tech Lead | Week 1 |
| Q11 | Concurrent user load estimate (peak) | Client | Week 2 |
| Q12 | Photo retention policy — forever or 2 years? | Client | Week 3 |
| Q13 | Notifications channel — email / SMS / WhatsApp / in-app only | Client | Week 4 |
| Q14 | Tanker GPS integration — phase 2 scope? | Client | Week 6 |
| Q15 | Rental party statement delivery — PDF email or portal access | Client | Week 6 |
