# McWay Site Management — Client Meeting Notes

**Date of Meeting:** 2026-04-14
**Project:** McWay Site Management
**Reference Pitch:** McWay_Pitch_Slides.html (3 approaches presented)

---

## 1. Meeting Context

We presented the client with **three possible approaches** for building the McWay Site Management system. The discussion centered on which approach best fits their operational reality, data accuracy needs, and user capability on the ground.

---

## 2. Client's Decision — ERP-Style Approach

The client is **inclined toward the ERP-style system** rather than a fully custom build.

**Client's reasoning:**
- ERP logic is generalized and proven — the base logic is already solid.
- Even if the ERP covers only ~50% of their exact requirements, the data it produces will be **accurate and reliable**.
- A custom system, by contrast, would demand continuous change requests and rework as requirements evolve.
- Trade-off accepted: less tailoring, higher accuracy and stability.

---

## 3. Key Client Requirement — Direct User Entry (No Data-Entry Operator)

The client was firm that **every user must enter their own data directly** into the system.

- No intermediate data-entry person should be responsible for logging activities on behalf of others.
- Example: if a **supervisor** is responsible for a material dumping entry, the supervisor himself must enter it — not a clerk, not an assistant.
- This applies across all roles: supervisor, senior engineer, camp in-charge, tanker operator, etc.
- Reason: accountability and traceability — the person closest to the activity owns the entry.

---

## 4. Conclusion — Build a Working Demo

**Agreed deliverable:** a working demo of the ERP-based system for:
- **1 site** (Kalol Sanand site)
- **1 plant/camp** (Khatraj Chokadi)

**But** — the demo must still allow the client to **add new sites, plants, camps, materials, and users** through the admin panel, so it is not hardcoded to a single setup.

---

## 5. Diesel Tracking — Major Gap Identified

The diesel logic in our earlier documentation is **missing / incorrect**. The owner raised this as a critical accountability concern. The real-world flow is more complex than what we had captured.

### 5.1 Diesel Tanker Flow

- Each site has **many machines** that consume diesel.
- Diesel is delivered by a **diesel tanker** which is operated by a **separate user** (tanker operator).
- The tanker fills up from a **petrol pump** and then travels across sites.
- The tanker may:
  - Serve **multiple sites** on the same day, **or**
  - Stay fixed at **one site** filling machines there.
- The tanker dispenses diesel directly into site machines.

**Implication:** the system must track tanker movement, fuel source (which pump), and each dispensing event (which machine, which site, how much).

### 5.2 Rented Machines — Two Scenarios

The client rents machines under two commercial models:

**a) Rented WITHOUT fuel**
- Client is responsible for fueling the machine.
- The McWay diesel tanker fills it — normal tracking applies.

**b) Rented WITH fuel (credit/advance model)**
- The rental party provides an **upfront diesel credit** (e.g., ₹1,000 of diesel given on credit).
- The machine then works across multiple sites in a day.
- At end of day, actual diesel consumption is calculated (e.g., ₹2,400 total).
- **Net cost billable = Actual − Credit already given** (e.g., ₹2,400 − ₹1,000 = ₹1,400).
- System must track:
  - Credit/loan of diesel extended to the machine
  - Actual consumption per site
  - Net payable / recoverable amount

### 5.3 Own Trucks — Petrol Pump Refueling

- Company-owned trucks (ushka, etc.) are **refueled directly at petrol pumps**, not by the tanker.
- Pump slips / records must be captured and maintained.

### 5.4 Action Item

The full diesel logic needs a **dedicated follow-up discussion** before finalizing. Core principle: **accountability must be enforced** — every litre must be traceable from source (pump) → tanker → machine → site.

---

## 6. Demo Scope & Module Plan

The demo will be built with a mix of **web (admin/computer-screen modules)** and **mobile app (field entry)**.

### 6.1 Admin Panel (Web)
Allows creation and configuration of:
- Sites
- Materials (master)
- Camps / Plants
- Users & roles
- Other master data

### 6.2 App Module (Mobile) — On-Site Dumping
- Used by supervisors on the field.
- Captures on-site material dumping entries at chainages.

### 6.3 Camp Module (Web / Computer Screen)
- Camp inventory
- Inward entries
- Outward list (to sites)
- Stock register

### 6.4 Plant Module (Web / Computer Screen)
- One plant module for plant material inward/outward tracking.

### 6.5 Senior Engineer Field Module
- Work progress entry (chainage-based).
- Road visuals / layer completion.

### 6.6 Diesel Module
- In scope for demo (basic version, pending the follow-up discussion noted in Section 5).

### 6.7 Dashboard
- Consolidated dashboard for all the activities above.

### 6.8 Out of Scope for Demo
- **Machine usage tracking** is deferred — not part of the demo. Focus stays on work progress, materials, camp/plant flow, and diesel.

---

## 7. Materials List Provided by Client

### 7.1 Kalol Sanand Site
**Camp / Plant Location:** Khatraj Chokadi

**Plant Materials (inward at plant):**
| Material | Unit | Grades / Sizes |
|---|---|---|
| Aggregate | Metric Ton | 6mm, 10mm, 20mm, 25–40mm, Storm dust |
| Cement | Metric Ton & Bags | — |
| Sand | Metric Ton | — |
| Steel | Metric Ton | Dia 8, 10, 12, 16, 20, 25, 32 mm |
| Emulsion | Metric Tonne | SS1, RS1 |

**Site Materials (dumped at site):**
- GSB
- WMM
- BM
- DBM
- BC
- SDBC
- Pipe — Dia 300, 600, 900, 1200 | Unit: numbers

**Plant Outward (from plant to site):**
- Steel
- Emulsion
- Concrete mix material — M10, M15, M20, M25, M30, M35, M40 | Unit: cubic metre
- *(How concrete is prepared — details to be shared by client later.)*

---

### 7.2 Magodi Plant

**Plant Materials (inward at plant):**
| Material | Unit | Grades / Sizes |
|---|---|---|
| Aggregate | Metric Ton | 6mm, 10mm, 20mm, 25–40mm, Storm dust |
| Cement | Metric Ton & Bags | — |
| Sand | Metric Ton | — |
| Steel | Metric Ton | Dia 8, 10, 12, 16, 20, 25, 32 mm |
| Emulsion | Metric Tonne | SS1, RS1 |
| Bitumen | Metric Tonne | VG30, VG40 |
| LDO / FO | Litres | — |

**Plant Outward Material:**
- BM
- DBM
- BC
- SDBC
- MSS
- Bitumen (VG30, VG40)
- Steel
- Emulsion

---

## 8. Open Items / Next Steps

1. **Re-discuss diesel logic** — tanker flow, rented-with-fuel credit reconciliation, truck pump refueling. Finalize before building diesel module.
2. **Concrete preparation details** — client to share how concrete mix is prepared at the plant.
3. **Build demo** covering: Admin Panel, On-Site Dumping App, Camp Module, Plant Module, Senior Engineer Work Progress, Diesel (basic), Dashboard.
4. **Exclude machine usage** from demo scope.
5. Ensure demo supports **multi-site / multi-plant creation** even though only one of each will be pre-loaded.
