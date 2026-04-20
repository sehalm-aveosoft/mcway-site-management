# McWay Site Management — Module 01: Admin Portal
## Detailed Specification for Odoo Implementation

**Version:** 2.0 (post client-review)
**Date:** 2026-04-16
**Module Owner:** Admin / Upper Management
**Platform:** Web (Odoo)

---

## Table of Contents
1. [Purpose & Scope](#1-purpose--scope)
2. [Odoo Implementation Strategy](#2-odoo-implementation-strategy)
3. [Menu & Navigation Structure](#3-menu--navigation-structure)
4. [Linkage Principle (IMPORTANT)](#4-linkage-principle-important)
5. [Sub-Module — Site Master](#5-sub-module--site-master)
6. [Sub-Module — Camp Master](#6-sub-module--camp-master)
7. [Sub-Module — Plant Master](#7-sub-module--plant-master)
8. [Sub-Module — Material Master](#8-sub-module--material-master)
9. [Sub-Module — Machine Master](#9-sub-module--machine-master)
10. [Sub-Module — Machine Assignment](#10-sub-module--machine-assignment)
11. [Sub-Module — Rental Deployments](#11-sub-module--rental-deployments)
12. [Sub-Module — Tanker Master](#12-sub-module--tanker-master)
13. [Sub-Module — Vendors](#13-sub-module--vendors)
14. [Sub-Module — User Management](#14-sub-module--user-management)
15. [Sub-Module — Master Configuration (Lookup Lists)](#15-sub-module--master-configuration-lookup-lists)
16. [Sub-Module — Estimation](#16-sub-module--estimation)
17. [Sub-Module — System Thresholds](#17-sub-module--system-thresholds)
18. [Sub-Module — Audit Log](#18-sub-module--audit-log)
19. [Common UI Patterns](#19-common-ui-patterns)
20. [Access Control](#20-access-control)
21. [Pre-loaded Demo Data](#21-pre-loaded-demo-data)

---

## 1. Purpose & Scope

The **Admin Portal** is the foundation layer of McWay Site Management. All operational modules (Camp, Plant, On-Site Dumping, Senior Engineer Progress, Diesel, Reports, Dashboard) consume master data defined here.

### In Scope
Create, edit, deactivate masters: **Sites, Camps, Plants, Materials, Machines, Tankers, Vendors, Users**; configure **Machine Assignments, Rental Deployments, Estimations, Thresholds, Lookup Lists**; maintain **Audit Log**.

### Out of Scope
- Transactional data (inward/outward/dumping/dispensing) — other modules.
- Diesel is its own module (links back to sites/camps/machines/vendors — not defined here).
- Reports & dashboard — separate modules.

---

## 2. Odoo Implementation Strategy

The Admin Portal is built as a custom Odoo module `mcway_site_management` reusing standard Odoo modules.

| Entity | Odoo Base | Strategy |
|---|---|---|
| Site | Custom `mcway.site` | Road-specific fields (multi-segment chainage, LHS/RHS widths) |
| Camp | Extend `stock.warehouse` | Warehouse type=Camp; stock locations auto-created |
| Plant | Extend `stock.warehouse` + `mrp` | Warehouse type=Plant; production via BoMs |
| Material | Extend `product.template` | Grades as product attribute variants |
| Machine | Extend `fleet.vehicle` | Master only holds permanent info |
| Machine Assignment | Custom `mcway.machine.assignment` | One active assignment per machine at a time; history retained |
| Rental Deployment | Custom `mcway.rental.deployment` | Per-rental-spell terms (with/without fuel, rate, dates) |
| Tanker | Extend `fleet.vehicle` sub-type=tanker | Multi-operator authorised list |
| Vendor | `res.partner` + tag Vendor | Suppliers of machines/materials/credits |
| User | `res.users` + custom `res.groups` | Single assignment (site/camp/plant/tanker/none) |
| Lookup Lists | Custom `mcway.lookup.*` models | Admin-editable dropdown values |
| Estimation | Custom `mcway.estimation` | ⚠️ TBD — discussion pending |
| Threshold | Custom `mcway.threshold.config` | Global + per-site overrides |
| Audit | OCA `auditlog` + `mail.thread` | Full change history |

### Dependencies
`base`, `mail`, `web`, `stock`, `product`, `mrp`, `fleet`, `contacts`, `auditlog` (OCA).

---

## 3. Menu & Navigation Structure

```
📋 McWay Admin
├── Dashboard
├── Masters
│   ├── Sites
│   ├── Camps
│   ├── Plants
│   ├── Materials
│   ├── Machines
│   ├── Tankers
│   └── Vendors
├── Operations Setup
│   ├── Machine Assignments
│   └── Rental Deployments
├── Users & Roles
│   └── Users
├── Master Configuration (Lookup Lists)
│   ├── Material Categories
│   ├── Machine Categories
│   ├── Layers / Activities
│   ├── Source Types
│   ├── Production Capabilities
│   ├── Payment Terms
│   └── Variant Templates (Concrete Grades, Steel Dia, Aggregate Size, Bitumen Grade, Emulsion Type, Pipe Dia)
├── Planning
│   └── Estimations  (⚠️ TBD — client discussion pending)
├── System Rules
│   └── Thresholds
└── Audit Log
```

---

## 4. Linkage Principle (IMPORTANT)

Linkages between entities are always entered on the **owning side**. The "other side" shows them as **read-only derived info**. This prevents double entry and keeps data consistent.

| Linkage | Entered On | Shown (read-only) On |
|---|---|---|
| Senior Engineer / Supervisor / Verifier → Site | **User form** (Assigned Site) | Site form → "Assigned Team" tab |
| Camp → Site(s) it serves | **Camp form** (Served Sites) | Site form → "Camps Serving This Site" |
| Plant → Camp(s) / Site(s) it supplies | **Plant form** (Supplies To) | Site / Camp form → "Plants Supplying" |
| Machine → Site / Camp / Plant | **Machine Assignment** record | Site / Camp / Plant form → "Machines On Site" |
| Vendor → Rented Machine | **Rental Deployment** record | Machine form → deployment history |
| Tanker → Site (current location) | **Diesel Dispense** record (transactional) | Tanker form → "Current Location" (auto) |

**Rule of thumb:** If an entity is "owned" by another (e.g., a Camp belongs to Sites, a Supervisor belongs to a Site), the ownership is declared on the child. The parent only displays it.

---

## 5. Sub-Module — Site Master

**Purpose:** Define a construction site.
**Odoo Model:** `mcway.site`

### 5.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Site Code | Char | Auto | `SITE/YYYY/####` |
| 2 | Site Name | Char | ✓ | Unique |
| 3 | Type | Selection | ✓ | Road / Bridge / Building |
| 4 | Status | Selection | ✓ | Active / Inactive / Closed |
| 5 | Location (City/Area) | Char | ✓ | — |
| 6 | State | Many2one | ✓ | `res.country.state` |
| 7 | Latitude | Float | ✗ | ±90 |
| 8 | Longitude | Float | ✗ | ±180 |
| 9 | Carriageway Mode | Selection | ✓ | Combined / Dual (LHS+RHS) |
| 10 | Road Width — Combined (m) | Float | If Combined | — |
| 11 | Road Width — LHS (m) | Float | If Dual | — |
| 12 | Road Width — RHS (m) | Float | If Dual | — |
| 13 | Chainage Segments | One2many | ✓ | ≥ 1 (see §5.2) |
| 14 | Total Length (km) | Computed | — | Σ segments ÷ 1000 |
| 15 | Planned Start Date | Date | ✗ | Optional |
| 16 | Planned End Date | Date | ✗ | > Start if set |
| 17 | Description / Notes | Text | ✗ | — |

### 5.2 Chainage Segments (child)

A site can have multiple non-continuous contract segments (e.g., 1–5 km + 7–15 km).

| Field | Type | Mandatory | Validation |
|---|---|---|---|
| Segment # | Integer | Auto | — |
| Start Chainage (m) | Integer | ✓ | ≥ 0 |
| End Chainage (m) | Integer | ✓ | > Start |
| Length (m) | Computed | — | End − Start |
| Remarks | Char | ✗ | — |

Rules: segments within a site cannot overlap; list auto-sorted by start chainage.

### 5.3 Derived (Read-only) Tabs

- **Assigned Team** — users whose "Assigned Site" = this site (role + name).
- **Camps Serving This Site** — camps that include this site in Served Sites.
- **Plants Supplying This Site** — plants listing this site in Supplies-to-Sites.
- **Machines On Site** — active Machine Assignments targeting this site.
- **Estimation** — estimation records for this site.
- **Change Log** — chatter + audit.

### 5.4 Business Rules
- Cannot delete a site with any transactional or estimation history.
- Team, Camp, Plant linkages **are not editable here** — set on the owning side.
- Deactivation hides from dropdowns but preserves data.

---

## 6. Sub-Module — Camp Master

**Purpose:** Camps store materials, optionally mix RMC, and serve one or more sites.
**Odoo Model:** Extend `stock.warehouse` (`mcway.camp`).

### 6.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Camp Code | Char | Auto | `CAMP/YYYY/###` |
| 2 | Camp Name | Char | ✓ | Unique |
| 3 | Address | Char | ✗ | Optional |
| 4 | Latitude | Float | ✗ | — |
| 5 | Longitude | Float | ✗ | — |
| 6 | **Served Sites** | Many2many → `mcway.site` | ✓ | **At least 1.** One camp can serve multiple sites. |
| 7 | Camp In-charge | Many2one → `res.users` | ✓ | Role = Camp In-charge |
| 8 | Storage Capacity (MT) | Float | ✗ | — |
| 9 | Has RMC Mixer | Boolean | — | If true, shows 6.2 fields |
| 10 | Mixer Capacity (m³/hr) | Float | Conditional | If RMC=true |
| 11 | Output Grades Produced | Many2many → Concrete Grades (lookup) | Conditional | e.g., M20, M25 |
| 12 | Status | Selection | ✓ | Active / Inactive / Closed |
| 13 | Remarks | Text | ✗ | — |

### 6.2 RMC Mixer at Camp — Developer Note

> **Developer Note:** When "Has RMC Mixer" is enabled, the camp becomes a mini-production unit. Production entries in the Camp Module will consume inputs (cement + sand + aggregate + water) via a BoM and produce concrete output.
>
> **Pending from client:** actual mix formulation (cement/sand/aggregate/water ratios per grade M10–M40). Use placeholder BoM values until the client provides exact ratios. Keep BoM structure editable so ratios can be tuned without code changes.

### 6.3 What This Form Does NOT Cover (Per Client Decision)
- **No Diesel Tank fields here.** Diesel is a separate module that links to camps on its own side.
- **No Plant "Supplied By" field here.** Plants declare which camps they supply from the Plant form.

### 6.4 Auto-Created Stock Locations (Odoo)
On save: `CAMP/<code>/Stock`, `CAMP/<code>/Scrap`, and if RMC=true: `CAMP/<code>/RMC Output`.

### 6.5 Inventory Flow (Explanation)
The Camp Master only **defines** the camp. Inventory is managed entirely in the transactional Camp Module:
1. Materials defined in Material Master → appear as dropdown in Camp Inward.
2. Camp In-charge records **Inward** (receive from vendor/plant) → stock ↑.
3. Camp In-charge records **Outward** (issue to site) → stock ↓.
4. System maintains Opening / Inward / Outward / Closing stock per material in real time.
5. No stock fields are entered at the master — stock emerges from transactions.

### 6.6 Business Rules
- Cannot delete a camp with any inward/outward/production history.
- Changing "Served Sites" after transactions exist is allowed, but a warning is shown.
- Deactivation hides the camp from dropdowns.

---

## 7. Sub-Module — Plant Master

**Purpose:** Plants are large-scale production + supply bases that feed camps and/or sites directly.
**Odoo Model:** Extend `stock.warehouse` + `mrp` BoMs.

### 7.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Plant Code | Char | Auto | `PLNT/YYYY/###` |
| 2 | Plant Name | Char | ✓ | Unique |
| 3 | Address | Char | ✓ | — |
| 4 | Latitude | Float | ✗ | — |
| 5 | Longitude | Float | ✗ | — |
| 6 | Plant In-charge | Many2one → `res.users` | ✓ | Role = Plant In-charge |
| 7 | Status | Selection | ✓ | Active / Inactive |
| 8 | Production Capabilities | Many2many → Capabilities (lookup) | ✓ | Bituminous / Concrete (RMC) / Emulsion / Steel |
| 9 | Daily Capacity per Capability | One2many → `mcway.plant.capacity` | ✗ | See §7.2 |
| 10 | Supplies To (Camps) | Many2many → `mcway.camp` | ✗ | — |
| 11 | Supplies To (Sites direct) | Many2many → `mcway.site` | ✗ | — |
| 12 | Remarks | Text | ✗ | — |

### 7.2 Daily Capacity (Child Table)
One row per capability the plant has. Allows distinct capacity per product line.

| Field | Type | Notes |
|---|---|---|
| Capability | Many2one → Capability lookup | From parent's selected capabilities |
| Daily Capacity | Float | — |
| Unit | Selection | MT / m³ / Litres — auto by capability |

Example:
| Capability | Capacity | Unit |
|---|---|---|
| Bituminous Mix | 500 | MT |
| Concrete (RMC) | 300 | m³ |

### 7.3 What Production Capabilities Do
Capabilities drive which **production-entry screens** appear in the Plant Module. A plant without "Concrete (RMC)" won't see a Concrete Production screen. They also filter which output materials appear in Plant Outward dropdowns.

### 7.4 What This Form Does NOT Cover (Per Client Decision)
- **No Diesel Tank fields.** Diesel in its own module.
- **LDO/FO is a material**, tracked through inventory — not declared here. (Container type — tank or barrel — is not relevant to the system.)
- **Emulsion and Steel** appear as raw and/or output materials **in inventory**, not as declared fields here. The capability only says "plant can produce/cut steel"; the actual quantities flow through inventory.

### 7.5 Auto-Created Stock Locations
`PLNT/<code>/Raw`, `PLNT/<code>/WIP`, `PLNT/<code>/Finished`, `PLNT/<code>/Scrap`.

### 7.6 BoMs
BoMs per grade (BM, DBM, BC, SDBC, MSS, M10–M40, SS1, RS1) are created in Odoo MRP.

> **Developer Note:** Initial BoMs are placeholders. **Pending from client:** exact mix ratios. Keep BoMs editable via the MRP UI so ratios can be tuned without code.

---

## 8. Sub-Module — Material Master

**Purpose:** Single source of truth for every material.
**Odoo Model:** `product.template` + attributes for variants.

### 8.1 Form Fields (simplified)

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Material Code | Char | Auto | `MAT/####` |
| 2 | Material Name | Char | ✓ | Unique |
| 3 | Category | Many2one → Material Category (lookup) | ✓ | — |
| 4 | Material Type | Selection | Auto / hidden | Raw / Produced / Consumable — auto-derived from Category; hidden unless Admin overrides |
| 5 | Measured In (Primary UoM) | Many2one → `uom.uom` | ✓ | "UoM" renamed to "Measured In" in the UI for clarity |
| 6 | Secondary UoM | Many2one → `uom.uom` | ✗ | **Only used for Cement & Steel** (hidden for other categories) |
| 7 | Conversion Factor | Float | Conditional | Visible only if Secondary UoM is set. Hint: *"How many primary units equal 1 secondary unit? E.g., 1 bag cement = 50 kg = 0.05 MT → enter 0.05."* |
| 8 | Density (kg/m³) | Float | ✗ | — |
| 9 | Specification Reference | Char | ✗ | e.g., IS-383, MoRTH |
| 10 | Low Stock Threshold | Float | ✗ | Unit = Primary UoM (displayed beside field) |
| 11 | Source Type(s) | Many2many tags | ✓ | Plant / Camp / External Vendor (from lookup) |
| 12 | Variants | One2many → `product.attribute.value` | ✗ | See §8.2 |
| 13 | Active | Boolean | — | Default true |

### 8.2 Removed Fields (per client decision)
- **Status** (replaced by standard Active boolean).
- **Track in Stock** — always true; removed from UI.
- **Applicable At** — no longer a user input. Auto-derived by the system based on Category + Material Type (e.g., "Bituminous Mix" → Plant Output; "Aggregate" → Camp + Plant + Site).

### 8.3 Variants (Grades / Sizes)
- Variants pull from variant-template lookup lists where a template exists: Concrete Grades, Steel Diameters, Aggregate Sizes, Bitumen Grades, Emulsion Types, Pipe Diameters.
- Admin can type a free-text variant if it's not in the template.
- When a supervisor/operator selects this material in a downstream form, only the variants marked here appear.

### 8.4 How Lookup Templates Flow Through (Example)
1. Admin → Master Configuration → **Steel Diameters** = [8, 10, 12, 16, 20, 25, 32 mm].
2. Admin creates Material "Steel" with Category = Steel.
3. In the Variants section, Steel Diameters appear as checkboxes. Admin checks 8, 12, 16.
4. In Supervisor's Dumping Entry, picking "Steel" shows only 8mm, 12mm, 16mm.
5. If a new size (40mm) arrives, Admin adds it to Steel Diameters once; it's available on next edit.

### 8.5 Material Type Auto-Derivation Rules
| Category | Type |
|---|---|
| Aggregate, Cement, Sand, Emulsion (raw), Bitumen, Fuel | Raw |
| Concrete Mix, Bituminous Mix | Produced |
| Pipe, Steel, Shuttering | Consumable |

---

## 9. Sub-Module — Machine Master

**Purpose:** Register all machines (permanent info only).
**Odoo Model:** Extend `fleet.vehicle`.

### 9.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Machine Code | Char | Auto | `MCH/####` |
| 2 | Registration No. / Name | Char | ✓ | Unique |
| 3 | Category | Many2one → Machine Category (lookup) | ✓ | — |
| 4 | Make / Brand | Char | ✗ | — |
| 5 | Model | Char | ✗ | — |
| 6 | Capacity / Specification | Char | ✗ | e.g., "1.0 cum bucket" |
| 7 | **Ownership Class** | Selection | ✓ | Owned / Rented |
| 8 | Expected Fuel Average | Float | ✗ | — |
| 9 | Fuel Unit | Selection | If #8 set | L/hr / L/km |
| 10 | Photo | Binary | ✗ | — |
| 11 | Remarks | Text | ✗ | — |
| 12 | Status | Selection | ✓ | Active / Inactive / Breakdown |

### 9.2 What Moved OUT (per client decision)
- **Rental Party / Rental Rate / With-or-Without Fuel** — now captured **per deployment** in the Rental Deployments sub-module (§11). A single rented machine may have many deployments over time with different vendors and fuel terms.
- **Assignment (Site / Camp / Plant)** — moved to Machine Assignment sub-module (§10).

Machine Master now holds **only permanent info**. Dynamic, time-bound info lives in the two supporting sub-modules.

---

## 10. Sub-Module — Machine Assignment

**Purpose:** Assign a machine to one Site, Camp, or Plant at a time. Provide a clean transfer flow with history.
**Odoo Model:** Custom `mcway.machine.assignment`.

### 10.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Assignment Code | Char | Auto | `ASN/YYYY/####` |
| 2 | Machine | Many2one → Machine | ✓ | — |
| 3 | **Target Type** | Selection | ✓ | Site / Camp / Plant |
| 4 | Target Site | Many2one → Site | If Target=Site | Filtered list |
| 5 | Target Camp | Many2one → Camp | If Target=Camp | — |
| 6 | Target Plant | Many2one → Plant | If Target=Plant | — |
| 7 | Start Date | Date | ✓ | — |
| 8 | End Date | Date | ✗ | Filled when assignment closes |
| 9 | Default Operator | Many2one → User | ✗ | Soft suggestion |
| 10 | Status | Selection | Auto | Active / Closed |
| 11 | Remarks | Text | ✗ | — |

### 10.2 Rules
- **Only one Active assignment** per machine at any time (enforced by Odoo constraint).
- Creating a new assignment for a machine that already has an active one triggers a **"Transfer"** flow: system auto-closes the existing assignment (sets End Date = today) and creates the new one.
- Full history retained — machine's assignment timeline viewable on its form.

### 10.3 UX
- Form is a two-step picker: first choose Target Type; then the relevant Site/Camp/Plant dropdown appears.
- "Transfer Machine" action from the Machine form opens this form pre-filled with the machine.

---

## 11. Sub-Module — Rental Deployments

**Purpose:** Capture time-bound rental terms for a rented machine. One deployment = one rental spell with one vendor.
**Odoo Model:** Custom `mcway.rental.deployment`.

### 11.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Deployment Code | Char | Auto | `RDP/YYYY/####` |
| 2 | Machine | Many2one → Machine (Ownership=Rented) | ✓ | Filter: rented only |
| 3 | Vendor | Many2one → Vendor | ✓ | — |
| 4 | Fuel Mode | Selection | ✓ | With Fuel / Without Fuel |
| 5 | Rental Rate (₹/day) | Float | ✓ | — |
| 6 | Start Date | Date | ✓ | — |
| 7 | End Date | Date | ✗ | Open-ended if blank |
| 8 | Remarks | Text | ✗ | — |

### 11.2 Rules & Use
- A rented machine must have an open Rental Deployment in order to work on site.
- When fuel mode = **With Fuel**, every diesel we pour into the machine is logged as **recoverable from the vendor** (see Diesel Module — netted against rental payable).
- When fuel mode = **Without Fuel**, diesel consumption is our cost; no vendor recovery.
- Daily rental bills accrue from Start Date to End Date × rate.

### 11.3 Example — Rented With Fuel
- Rate ₹2,400/day (quote includes diesel, vendor's responsibility).
- We fuel the machine via our tanker / at a pump: ₹1,000.
- System records ₹1,000 as "recoverable from vendor".
- End-of-day bill: ₹2,400 rental − ₹1,000 diesel we already paid = **₹1,400 net payable to vendor**.
- The ₹1,000 is **not** an additional expense; it is offset against the vendor's bill.

---

## 12. Sub-Module — Tanker Master

**Purpose:** Diesel tankers used to fuel machines across sites.
**Odoo Model:** `fleet.vehicle` sub-type=tanker.

### 12.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Tanker Code | Char | Auto | `TNK/###` |
| 2 | Registration No. | Char | ✓ | Unique |
| 3 | Tank Capacity (L) | Float | ✓ | > 0 |
| 4 | Current Balance (L) | Float | System-maintained | Read-only; adjust via "Opening Stock Adjustment" action |
| 5 | **Authorised Operators** | Many2many → Users (role=Tanker Operator) | ✓ | Multiple; handles absence |
| 6 | Current Location (Site) | Many2one → Site | Auto | Read-only; auto-updates from latest dispense |
| 7 | Status | Selection | ✓ | Active / Inactive / Maintenance |
| 8 | Photo | Binary | ✗ | — |
| 9 | Remarks | Text | ✗ | — |

### 12.2 Changes from v1 (per client decision)
- **Opening Balance field removed from creation form.** Fresh tanker starts at 0. If diesel is already inside on go-live, Admin uses "Opening Stock Adjustment" action after save (one-time entry with reason, audited).
- **Multiple operators** — any authorised operator can log dispenses; the logged-in user stamps the entry automatically. Solves the "absent default operator" problem.
- **Current Location** auto-tracked — every dispense the operator picks the site; system updates tanker's current location. A tanker may remain at one site all day, or move across multiple sites.

---

## 13. Sub-Module — Vendors

**Purpose:** External parties we deal with. Renamed from "Rental Party".
**Odoo Model:** `res.partner` with tag `Vendor`.

### 13.1 Why We Need Vendors
Vendors appear in multiple places:
- **Rental Deployments** — vendor provides a rented machine.
- **Camp / Plant Inward** — vendor supplies cement, aggregate, bitumen, etc.
- **Diesel credit offset (rented-with-fuel)** — when we fuel a vendor's machine, the ₹ are offset against what we owe the vendor.
- Future: material vendor bills, payments.

Without a vendor master every inward / rental entry would have free-text names — inconsistent and un-reportable.

### 13.2 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Vendor Code | Char | Auto | `VEN/###` |
| 2 | Vendor Name | Char | ✓ | — |
| 3 | Contact Person | Char | ✗ | — |
| 4 | Phone | Char | ✗ | Optional |
| 5 | Email | Char | ✗ | Optional |
| 6 | GST Number | Char | ✗ | Warning on invalid GSTIN format, not blocking |
| 7 | Address | Text | ✗ | — |
| 8 | Payment Terms | Many2one → Payment Terms (lookup) | ✗ | Net 30 default |
| 9 | **Vendor Roles** | Many2many tags | ✗ | Machine Rental / Material Supplier / Diesel Supplier / Other |
| 10 | Current Net Balance (₹) | Computed | — | (Bills − Payments − Offsets) |
| 11 | Status | Selection | ✓ | Active / Inactive |

### 13.3 Linked Records (Read-only Tabs)
- **Machines Provided** — rented machines where this vendor is on the active/past Rental Deployment.
- **Deployment History** — full timeline of rental deployments.
- **Diesel Offsets Outstanding** — ₹ we paid for their machines' diesel, not yet settled.
- **Inward Supply History** — materials supplied to camps/plants (from transactional modules).

---

## 14. Sub-Module — User Management

**Purpose:** Create users with role + **single scope assignment**.
**Odoo Model:** `res.users` + custom `res.groups`.

### 14.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | User Code | Char | Auto | `USR/###` |
| 2 | Full Name | Char | ✓ | — |
| 3 | Login (Username) | Char | ✓ | Unique |
| 4 | Email | Char | ✗ | Optional |
| 5 | Phone | Char | ✗ | Optional |
| 6 | Password | Password | ✓ on create | Min 8; complexity policy |
| 7 | Preferred Language | Selection | — | English / Gujarati / Hindi |
| 8 | Role | Selection | ✓ | Admin / Owner / Senior Engineer / Supervisor / Camp In-charge / Plant In-charge / Tanker Operator / Verifier |
| 9 | **Assignment Type** | Selection | Conditional | Site / Camp / Plant / Tanker / None |
| 10 | Assignment Target | Many2one (polymorphic) | Conditional | One specific Site OR Camp OR Plant OR Tanker |
| 11 | Mobile App Access | Boolean | Auto by role | Supervisor / Tanker Operator default true |
| 12 | Web Access | Boolean | Auto by role | Other roles default true |
| 13 | Status | Selection | ✓ | Active / Inactive / Locked |
| 14 | Last Login | Datetime | Auto | Read-only |

### 14.2 Assignment Rules (Per Client Decision)
A user can be assigned to **only one** Site / Camp / Plant / Tanker at a time. No multi-site users in the demo.

Form flow:
1. Choose **Role**.
2. Choose **Assignment Type** (auto-limited by role — e.g., Admin/Owner/Verifier can be None).
3. Choose the specific Site / Camp / Plant / Tanker.

Admin can change a user's assignment at any time — last-write wins, previous assignment closed automatically, audited.

### 14.3 Role → Assignment Type Guidance
| Role | Allowed Assignment Types |
|---|---|
| Admin | None |
| Owner / Upper Management | None |
| Senior Engineer | Site |
| Supervisor | Site |
| Camp In-charge | Camp |
| Plant In-charge | Plant |
| Tanker Operator | Tanker |
| Verifier | Site (usually) |

---

## 15. Sub-Module — Master Configuration (Lookup Lists)

**Purpose:** Admin-editable dropdown values used everywhere.

### 15.1 Fixed vs Configurable

| Category | Fixed (dev-owned) | Configurable (Admin) |
|---|---|---|
| Material | Material Type | Material Category, Source Type |
| Machine | Ownership Class, Fuel Unit, Status | Machine Category |
| Site | Type, Status, Carriageway Mode | — |
| Plant | — | Production Capability |
| Layer/Progress | Side (LHS/RHS/Lane/Median) | Layer / Activity |
| Commercial | — | Payment Terms, Vendor Roles |
| User | Role, Assignment Type | Preferred Language |

### 15.2 Configurable Lists

| List | Used In |
|---|---|
| Material Category | Material Master |
| Machine Category | Machine Master |
| Layer / Activity | Dumping, Progress, Estimation |
| Source Type | Material, Camp Inward |
| Production Capability | Plant Master |
| Payment Terms | Vendor |
| Vendor Roles | Vendor |
| Concrete Grades | Variant template (Concrete Mix) |
| Steel Diameters | Variant template (Steel) |
| Aggregate Sizes | Variant template (Aggregate) |
| Bitumen Grades | Variant template (Bitumen) |
| Emulsion Types | Variant template (Emulsion) |
| Pipe Diameters | Variant template (Pipe) |

### 15.3 Lookup Row Fields
Sequence · Name (✓) · Short Code · Description · Active.

### 15.4 Rules
- Deletion blocked if value is referenced anywhere; deactivate instead.
- Reordering via sequence; affects dropdown display only.
- Renaming updates all references automatically (m2o).

---

## 16. Sub-Module — Estimation

> **⚠️ TBD — detailed discussion pending with client.** Keep the current fields as preliminary. Do not finalise database constraints until client confirms the estimation model (segment-wise vs stretch-wise vs layer-wise, daily/weekly/monthly targets calculation, machine and diesel expectations).

**Odoo Model:** Custom `mcway.estimation`.

Preliminary fields: Site, Stretch Start/End, Layer, Width, Thickness, Material, Density, Estimated Qty (computed), Daily Target, Expected Machines, Expected Diesel, Remarks.

Formula: `Length × Width × Thickness × Density`.

---

## 17. Sub-Module — System Thresholds

Global tolerances used by reports and dashboard alerts.

| Field | Default | Purpose |
|---|---|---|
| Variance Tolerance (%) | 5.0 | Alert when actual vs estimated deviation exceeds |
| Unverified Entry Alert (days) | 2 | Flag entries unverified beyond |
| Low-Stock Alert (%) | 15 | Warn when stock drops below % of capacity |
| DPR Cutoff Time | 09:00 | Next-day DPR expected by |
| Report Frequency | Daily | Default report cadence |

---

## 18. Sub-Module — Audit Log

Every create / write / delete / state-change across all Admin and transactional models. Uses OCA `auditlog` + `mail.thread`.

Fields: Timestamp · User · Model · Record ID · Record Name · Action · Changed Fields (JSON diff) · IP.

**Retention:** 5 years. Exportable CSV. Admin-only.

---

## 19. Common UI Patterns

- **Breadcrumbs** on every form: `McWay Admin > Masters > <Entity> > <Record>`.
- **Status badges** — color-coded (Active=green, Inactive=grey, Closed=red, Breakdown=orange, Locked=dark).
- **Sticky toolbar + footer** with Save / Save & Add Another / Cancel.
- **Required markers** red `*`.
- **Auto-computed fields** displayed read-only with grey background.
- **Conditional fields** use Odoo attrs (hide/show based on other fields).
- **Tag-input** for many2many.
- **Bulk actions** from list view (activate, deactivate, export).

---

## 20. Access Control

### 20.1 Model-Level (ir.model.access.csv summary)

| Model | Admin | Owner | SE | Supv | Camp | Plant | Tanker | Verifier |
|---|---|---|---|---|---|---|---|---|
| Site, Camp, Plant, Material, Machine, Vendor, Tanker, Lookup, Threshold | CRUD | R | R | R | R | R | R | R |
| User | CRUD | R | — | — | — | — | — | — |
| Machine Assignment | CRUD | R | R | — | R | R | — | R |
| Rental Deployment | CRUD | R | — | — | — | — | — | — |
| Estimation | CRUD | R | R (own site) | — | — | — | — | R |
| Audit Log | R | R | — | — | — | — | — | — |

### 20.2 Record Rules
Scope enforced via `assignment_target` on user → filters Sites/Camps/Plants/Tankers accordingly.

---

## 21. Pre-loaded Demo Data

### 21.1 Sites
| Code | Name | Type | Segments | Carriageway |
|---|---|---|---|---|
| SITE/2026/001 | Kalol Sanand | Road | 0–27000 m (single segment) | Combined, 7.5 m |

### 21.2 Camps
| Code | Name | Served Sites | RMC? |
|---|---|---|---|
| CAMP/2026/001 | Khatraj Chokdi | Kalol Sanand | Yes (capacity 30 m³/hr, grades M20/M25) |

### 21.3 Plants
| Code | Name | Capabilities | Daily Capacity | Supplies |
|---|---|---|---|---|
| PLNT/2026/001 | Magodi | Bituminous, RMC, Emulsion, Steel | BM 500 MT, RMC 300 m³ | Khatraj Chokdi |

### 21.4 Materials — seed from client list
(Same as earlier — aggregates, cement, sand, steel, emulsion, bitumen, LDO/FO, GSB, WMM, BM, DBM, BC, SDBC, MSS, Concrete Mix, Pipe.)

### 21.5 Machines (sample)
5 representative machines (Excavator, JCB, Roller, Paver, Dumper). One "Rented" for deployment demo.

### 21.6 Tankers
1 tanker, capacity 10,000 L, 2 authorised operators.

### 21.7 Vendors
2 vendors — Shree Construction (Machine Rental), Maruti Hirers (Machine Rental + Material Supplier).

### 21.8 Users
Admin, Owner, SE, 2 Supervisors, Camp In-charge, Plant In-charge, 2 Tanker Operators, Verifier.

### 21.9 Rental Deployments
1 active deployment — rented Paver from Maruti Hirers, With Fuel, ₹25,000/day, started 2026-04-10.

### 21.10 Machine Assignments
5 active assignments — all machines assigned to Kalol Sanand.

---

## 22. Open Items / Notes for Developers

1. **RMC mix formulation** (Camp & Plant) — pending from client. Keep placeholder BoMs editable.
2. **Estimation model** — to be finalised with client before coding hard constraints.
3. **Localization** (Gujarati / Hindi) — to be confirmed.
4. **GSTIN validation** — warning, not blocking.
5. **Password policy** — confirm complexity rules.
6. **Opening stock for tankers** — use standalone "Opening Stock Adjustment" action, not a field on Tanker form.
7. **Vendor diesel offset** — implement as an accounting-like ledger on the Vendor; each "we-fuelled-their-machine" event is a negative entry against their rental payable.
8. **Machine transfer flow** — auto-closes active assignment on new assignment creation; UI must show both records in the history.
