# McWay Site Management — Changelog

All notable changes to the project documentation and mockups are documented here.

---

## [2026-04-20] — Module 04: Plant Module

### Added
- **Module 04 spec** (`docs/Module_04_Plant_Module.md`) — 16 sections covering:
  - Raw Material Inward (plant weighbridge, vendor/quarry sources)
  - Capability-driven Production sub-modules (Bituminous / Concrete RMC / Emulsion / Steel)
  - BoM auto-fill with editable Actual Quantity column and variance tracking (green/amber/red)
  - Plant Outward — dispatches to either linked camps OR linked sites (polymorphic destination)
  - Stock Register with Raw + Finished as separate tabs
  - Explicit Scrap & Wastage module (distinct from implicit BoM variance)
  - Plant-scoped Labor entries (no cross-deployment to sites)
  - Capacity Utilization strip (today's output vs Daily Capacity per capability)
  - Verification Queue with variance flagging (> 10% = ⚠️)
  - State machine, access control, pre-loaded demo data, 16 design decisions
- **Interactive HTML mockup** (`mockups/plant-module.html`) — slate-blue theme:
  - Home with capacity utilization bars, 7 KPI tiles, quick actions
  - Raw Inward list + form (gross/tare/net auto-calc from plant weighbridge)
  - Production tabs — Bituminous, RMC, Emulsion, Steel — with card-style capability switcher
  - Production form with BoM auto-fill per grade (placeholder ratios)
  - Plant Outward form (Camp or Site destination picker)
  - Stock Register with Raw/Finished tabbed views
  - Scrap form with mandatory reason + linked batch reference
  - Verification Queue with variance ⚠️ highlighting
- **Demo data:** Magodi plant, 4 capabilities, 7 raw inward entries, 5 production batches across 3 capabilities, 4 outward dispatches, 2 scrap entries

### Design Decisions
- Capability-driven menu visibility — a plant without Emulsion capability doesn't see that screen
- BoM auto-fill is a suggestion, not a constraint — variance captured for reporting, never blocks
- Raw vs Finished as separate Odoo stock locations — operationally different, reported separately
- Plant Outward can dispatch raw materials too (rebalance scenario — e.g., extra Aggregate to camp)
- Explicit Scrap module separates physical scrap from implicit consumption variance
- No plant-to-plant transfers in demo (only 1 plant)
- Shared labor model with Camp but no "Deployed To" (plant labor = plant-only)

---

## [2026-04-20] — Module 03: On-Site Materials (Dumping)

### Added
- **Module 03 spec** (`docs/Module_03_OnSite_Materials.md`) — 18 sections covering:
  - Mobile-first dumping entry form (single scrollable screen, 23 fields)
  - Offline-first architecture with sync queue
  - Chainage input with km/m toggle + segment validation
  - Material & source filtering based on site linkages
  - Photo capture with compression
  - Verification flow (supervisor side)
  - Smart defaults (last-used material/source/side/vehicle remembered)
- **Mobile HTML mockup** (`mockups/onsite-materials.html`) — phone-frame layout with:
  - Home screen with KPI tiles + quick actions
  - Full dumping entry form with live weight display
  - My Entries list with filters (All/Today/Submitted/Verified/Correction)
  - Entry detail view with correction request display
  - Sync Queue with interactive offline/online demo
- **Machine field** as Phase 2 placeholder (disabled in demo, noted in spec)
- **Single weight field** — no Gross/Tare split at site (supervisor reads net from slip)

### Design Decisions
- Single weight field: supervisor is at site, not weighbridge — enters net directly
- Machine field planned for Phase 2: enables per-machine productivity reporting
- PWA recommended for demo (no app store friction), native for production
- Source Vendor allows free text (unlike Camp inward which requires master)

---

## [2026-04-17] — Module 02: Camp Module

### Added
- **Module 02 spec** (`docs/Module_02_Camp_Module.md`) — 16 sections covering:
  - Material Inward (weighbridge flow, source types, photo support)
  - Material Outward (site-only destination, stock validation)
  - Stock Register (auto-maintained, per material+variant, low-stock alerts)
  - RMC Production (BoM auto-fill, two-step produce→dispatch)
  - Labour Arrival & Deployment (free-text party with autocomplete)
  - Shuttering Tracking (5 transaction types, register with discrepancy check)
  - Verification Queue (bulk verify, age-based highlighting)
  - State machine (Draft→Submitted→Verified→Locked)
- **Camp HTML mockup** (`mockups/camp-module.html`) — 8 interactive views with demo data

### Design Decisions (15 total)
- Quarry = Vendor with role tag (no new entity)
- Outward captures site only, not chainage (chainage is supervisor's domain)
- One material per outward entry
- RMC: two-step flow (produce → stock → dispatch)
- Labour party name is free text with autocomplete
- Backdating allowed up to 3 days
- Plant outward ↔ camp inward NOT auto-matched (phase 2)

---

## [2026-04-16] — Module 01: Admin Portal v2

### Changed (post client-review)
- **Multi-segment chainage** — sites can have non-continuous contract segments (e.g., 1–5 km AND 7–15 km)
- **Carriageway mode** — Combined or Dual (LHS+RHS) with separate widths
- **Machine Master** stripped to permanent info only:
  - Rental terms moved to **Rental Deployments** sub-module (per-spell)
  - Assignment moved to **Machine Assignment** sub-module (one active at a time, auto-transfer)
- **Camp** — one camp can serve MANY sites (Option B); address optional; no diesel fields
- **Plant** — no tank fields; per-capability daily capacity as repeating table
- **Material** — simplified; renamed UoM→"Measured In"; Secondary UoM only for Cement/Steel
- **Tanker** — multiple authorised operators; opening balance via action, not form field; current location auto-tracked
- **Vendors** — renamed from "Rental Party"; GSTIN warning non-blocking
- **User** — single scope assignment (one site/camp/plant/tanker/none)
- **Petrol Pump master** — REMOVED
- **Linkage Principle** documented (relationships entered on owning side)

### Added
- **Admin Portal list/view mockup** (`mockups/admin-portal.html`) — all entities with demo data
- **Admin data entry forms mockup** (`mockups/admin-forms.html`) — all form screens
- **Module 01 spec v2** (`docs/Module_01_Admin_Portal.md`) — 21 sections, 62 roles

---

## [2026-04-15] — Foundation Documents

### Added
- **PRD** (`docs/PRD.md`) — Product Requirements Document with:
  - 6 personas (Owner, SE, Supervisor, Camp In-charge, Plant In-charge, Tanker Operator)
  - 9 epics (A–I), 30+ user stories with priority (P0/P1/P2)
  - 6 user flows with alternate paths
  - Data model (17 entities)
  - Success metrics (north-star: diesel traceability coverage ≥ 95%)
  - Risk matrix (technical, product, business)
  - Release plan (12-week milestones)
- **Demo Documentation** (`docs/Demo_Documentation.md`) — Full demo scope with:
  - 8 modules defined, 8 roles, 16 material types
  - Verification workflow (non-blocking, accountability-only)
  - Pre-loaded demo data specification

---

## [2026-04-14] — Project Kickoff

### Added
- **Client Meeting Notes** (`docs/Meeting_Notes_2026-04-14.md`)
- Key decisions from meeting:
  - ERP-style Odoo build chosen over custom
  - Demo scope: 1 site (Kalol Sanand), 1 camp (Khatraj Chokdi), 1 plant (Magodi)
  - Diesel logic clarified (rented-with-fuel offset, no double counting)
  - Verifier role confirmed as non-blocking
  - Machine usage tracking deferred to Phase 2

---

## Open Items (Pending from Client)

1. Concrete & bituminous mix BoM ratios — placeholder values in use
2. Estimation model — detailed discussion pending
3. Localization (Gujarati / Hindi UI)
4. Dashboard refresh cadence (real-time vs 15-min batch)
5. Password policy specifics
6. Mobile app technology choice (PWA vs native, iOS needed?)
