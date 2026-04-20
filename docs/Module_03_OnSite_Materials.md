# McWay Site Management — Module 03: On-Site Materials (Dumping)
## Detailed Specification for Mobile App Implementation

**Version:** 1.0 (initial draft)
**Date:** 2026-04-20
**Module Owner:** Supervisor (field)
**Platform:** Mobile App (Android primary; PWA/iOS TBD)
**Prerequisite:** Module 01 — Admin Portal (Site Master, Material Master, User Management, Lookup Lists)

---

## Table of Contents
1. [Purpose & Scope](#1-purpose--scope)
2. [Platform & Technical Strategy](#2-platform--technical-strategy)
3. [Navigation & Screen Structure](#3-navigation--screen-structure)
4. [Home Screen](#4-home-screen)
5. [Sub-Module — Dumping Entry](#5-sub-module--dumping-entry)
6. [Sub-Module — My Entries](#6-sub-module--my-entries)
7. [Sub-Module — Sync Queue (Offline)](#7-sub-module--sync-queue-offline)
8. [Offline-First Architecture](#8-offline-first-architecture)
9. [Chainage Input & Validation](#9-chainage-input--validation)
10. [Material & Source Filtering](#10-material--source-filtering)
11. [Weight Entry & Non-Weight Materials](#11-weight-entry--non-weight-materials)
12. [Photo Capture](#12-photo-capture)
13. [Verification Flow (Supervisor Side)](#13-verification-flow-supervisor-side)
14. [State Machine](#14-state-machine)
15. [Access Control](#15-access-control)
16. [Pre-loaded Demo Data & Walkthrough](#16-pre-loaded-demo-data--walkthrough)
17. [Design Decisions & Assumptions](#17-design-decisions--assumptions)
18. [Open Items / Notes for Developers](#18-open-items--notes-for-developers)

---

## 1. Purpose & Scope

The **On-Site Materials Module** is a mobile-first application used by **Supervisors** on the construction site. When a vehicle arrives at a chainage with material, the Supervisor records the dumping event in real time — material, quantity, location, slip details, and an optional photo.

The site is **not a storage location** — no stock is maintained here. Material arrives, gets dumped at a chainage, and the entry is recorded. The system provides traceability from source (camp/plant/vendor) → site chainage.

### In Scope
- **Dumping Entry:** Record each material arrival + dumping at a chainage.
- **My Entries:** View and edit own submitted entries.
- **Sync Queue:** Offline entry queue with background sync.
- **Verification feedback:** See which entries are verified, which have correction requests.

### Out of Scope
- **Material returns / rejections** — not in demo.
- **Stock tracking at site** — site is not a storage point; material is consumed on arrival.
- **Work progress** — separate module (Senior Engineer Work Progress, Module 05).
- **Diesel operations** — separate module (Module 06).
- **Camp outward ↔ site dumping auto-matching** — manual reconciliation via reports; auto-match in phase 2.
- **Machine usage / idle tracking** — deferred to phase 2.

---

## 2. Platform & Technical Strategy

### 2.1 Technology Decision (Pending)

| Option | Pros | Cons |
|---|---|---|
| **PWA (Progressive Web App)** | No app store; works on any device; easier updates | Limited offline capabilities; no native camera access on older devices |
| **React Native / Flutter** | True offline-first; native camera; push notifications | Separate codebase; app store review; installation friction |
| **Odoo Mobile + custom views** | Reuses Odoo backend; less code | Limited offline; poor UX for field workers |

> **Recommendation for demo:** PWA with service worker for offline queue. Simpler to build, no app store needed. Revisit for production if offline reliability is insufficient.
>
> **Client decision pending:** Confirm PWA vs native; confirm if iOS is needed (§18 Open Items).

### 2.2 Backend Integration

- All data syncs to the same Odoo backend as Camp/Plant modules.
- Dumping entries create `mcway.dumping.entry` records.
- Material + variant dropdowns pulled from `product.template` / `product.product`.
- Chainage validation against `mcway.site` segments.
- Photos uploaded as `ir.attachment` linked to the entry.
- Offline queue uses IndexedDB (PWA) or SQLite (native) with background sync.

### 2.3 UX Principles for Field Use

| Principle | Why | How |
|---|---|---|
| **Minimum taps** | Supervisor is standing in sun, wearing gloves, vehicle waiting | Pre-fill site, auto-suggest last-used material/source, large touch targets |
| **Offline-first** | Sites have poor/no connectivity | Queue locally, sync when online, never lose an entry |
| **No free text for material** | Data consistency | All dropdowns from master; search-as-you-type |
| **Large fonts & high contrast** | Outdoor readability | Min 16px body text; no low-contrast grey-on-grey |
| **Language support** | Workers speak Gujarati/Hindi | UI language toggle (English + Gujarati + Hindi — TBD) |
| **Fast submission** | Entry should take < 60 seconds | One-screen form, auto-calculations, smart defaults |

---

## 3. Navigation & Screen Structure

The Supervisor sees a simple bottom-tab mobile layout. No sidebar — everything is bottom navigation.

```
┌──────────────────────────────────┐
│ [Header: Site name, connectivity]│
├──────────────────────────────────┤
│                                  │
│         [Screen content]         │
│                                  │
├──────────────────────────────────┤
│  🏠 Home  │  ➕ New  │  📋 My   │
│           │ Dumping  │ Entries  │
└──────────────────────────────────┘
```

| Tab | Screen | Purpose |
|---|---|---|
| **Home** | Dashboard tiles + quick action | Landing screen |
| **New Dumping** | Dumping entry form | Primary action — record a dumping event |
| **My Entries** | List of own entries | Review, edit, see verification status |

Additional screens (not tabs):
- **Entry Detail** — tap an entry from My Entries to view/edit.
- **Sync Queue** — accessible from Home when offline entries are pending.
- **Profile / Settings** — language, sync status, logout.

---

## 4. Home Screen

**Purpose:** Quick snapshot of today's activity and primary action button.

### 4.1 Header Bar (Persistent)
| Element | Notes |
|---|---|
| Site name | Pre-filled from user assignment (e.g., "Kalol Sanand") |
| Connectivity indicator | 🟢 Online / 🔴 Offline / 🟡 Syncing |
| Sync badge | "3 pending" — count of unsynced entries |
| User avatar + name | Tap for profile/settings |

### 4.2 Today's Summary Tiles

| Tile | Value | Notes |
|---|---|---|
| Entries Today | Count of dumping entries submitted today | Tap → My Entries filtered to today |
| Total Dumped | Sum of net weight/qty across today's entries | "124.5 MT" |
| Pending Sync | Count of entries not yet synced | Tap → Sync Queue |
| Correction Requests | Count of entries with verifier remarks | Tap → filtered My Entries |

### 4.3 Primary Action
Large prominent button: **"+ New Dumping Entry"** — opens the dumping form.

### 4.4 Recent Entries (Quick List)
Last 5 entries in reverse chronological order:
- Material + variant, net weight, chainage range, time ago, state badge.
- Tap → Entry Detail.

---

## 5. Sub-Module — Dumping Entry

**Purpose:** Record a material dumping event at a specific chainage on the site.
**Odoo Model:** Custom `mcway.dumping.entry`.

### 5.1 Form Fields

The form is a **single scrollable screen** — no tabs, no multi-step wizard. Designed for speed.

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Entry Code | Char | Auto | `DMP/YYYY/####` — generated on sync, placeholder shown as "DMP/pending" until synced |
| 2 | Site | Display | Auto | Pre-filled from logged-in user's assigned site; read-only |
| 3 | Date & Time | Datetime | ✓ | Defaults to now; editable. **Backdating allowed up to 3 days.** Cannot be in the future. |
| 4 | **Material** | Search-select | ✓ | Dropdown from Material Master. Search-as-you-type. Pre-filtered to materials applicable to this site (see §10). |
| 5 | **Variant / Grade** | Select | Conditional | Shown only if selected material has variants. Mandatory if shown. |
| 6 | **Chainage Start** | Integer/Float | ✓ | In metres or km (see §9). Validated against site segments. |
| 7 | **Chainage End** | Integer/Float | ✓ | > Chainage Start. Validated against site segments. |
| 8 | **Side** | Select | ✓ | LHS / RHS / Both / Lane / Median |
| 9 | **Layer / Activity** | Select | ✓ | From Lookup List: GSB / WMM / BM / DBM / BC / SDBC / MSS / CC / Pipe-laying / Other |
| 10 | **Source Type** | Select | ✓ | Camp / Plant / Direct Vendor |
| 11 | Source — Camp | Select | If Source=Camp | Filtered to camps serving this site |
| 12 | Source — Plant | Select | If Source=Plant | Filtered to plants supplying this site (direct or via camp) |
| 13 | Source — Vendor | Text/Select | If Source=Vendor | Vendor from master or free text for one-off |
| 14 | Source Reference | Char | ✗ | Optional: camp outward challan number (for traceability) |
| 15 | Vehicle Number | Char | ✓ | Auto-uppercase. Autocomplete from recent entries. |
| 16 | Party / Vendor Name | Char | ✗ | Free text — name of the driver / transporter |
| 17 | Slip Number | Char | ✓ | — |
| 18 | Royalty Number | Char | ✗ | Government compliance (for aggregates, sand) |
| 19 | **Weight / Quantity** | Float | ✓ | Single field. Supervisor enters the weight directly (no Gross/Tare split — weighbridge slip already shows net weight). UoM label auto-set from Material Master (MT / m³ / Numbers). Highlighted in large font. |
| 20 | **Machine** | Select | ✗ | **Phase 2 — not in demo.** Will show machines assigned to this site (from Machine Assignment). Captures which machine was used for dumping/laying. Placeholder shown as disabled in demo UI. |
| 21 | Photo | Camera/Gallery | ✗ | Optional. Up to 2 photos (slip + dumping location). Max 5 MB each, auto-compressed to 1080p. |
| 22 | Remarks | Text | ✗ | Short note field |
| 23 | State | Selection | Auto | Draft → Submitted → Verified → Locked (see §14) |

### 5.2 Form Layout (Mobile Optimized)

The form is organized into collapsible sections for scannability, but all sections are visible by default (no hidden tabs):

```
┌─ WHAT ─────────────────────────────────┐
│ Material [search dropdown]             │
│ Variant  [dropdown — if applicable]    │
│ Layer    [dropdown]                     │
└────────────────────────────────────────┘
┌─ WHERE ────────────────────────────────┐
│ Chainage Start [    ] — End [    ] m   │
│ Side  [LHS] [RHS] [Both] [Lane] [Med] │
└────────────────────────────────────────┘
┌─ FROM ─────────────────────────────────┐
│ Source [Camp ▼]  → [Khatraj Chokdi ▼]  │
│ Source Ref [optional challan #]        │
└────────────────────────────────────────┘
┌─ VEHICLE & SLIP ───────────────────────┐
│ Vehicle No [          ]                │
│ Party      [          ]                │
│ Slip No    [          ]                │
│ Royalty No [          ] (optional)     │
└────────────────────────────────────────┘
┌─ WEIGHT ───────────────────────────────┐
│         ┌──────────────────┐           │
│         │  Weight: [18.0] MT│ ← big    │
│         └──────────────────┘           │
│  (single field — from weighbridge slip)│
└────────────────────────────────────────┘
┌─ MACHINE (Phase 2) ───────────────────┐
│  Machine [disabled — coming soon]      │
└────────────────────────────────────────┘
┌─ PHOTO & NOTES ────────────────────────┐
│ [📷 Take Photo]  [🖼 Gallery]          │
│ Remarks [                    ]         │
└────────────────────────────────────────┘
         ┌──────────────────┐
         │   ✅ SUBMIT      │ ← sticky bottom
         └──────────────────┘
```

### 5.3 Smart Defaults & Speed Optimizations

| Optimization | How |
|---|---|
| **Site pre-filled** | From user assignment — never changes during session |
| **Last-used material remembered** | If supervisor dumped "Aggregate 20mm" 5 minutes ago, it's pre-selected for the next entry |
| **Last-used source remembered** | Camp/Plant selection persists across entries |
| **Last-used side remembered** | LHS/RHS selection persists |
| **Vehicle autocomplete** | Last 20 vehicle numbers cached, shown as suggestions |
| **Chainage auto-suggest** | After first entry, start chainage of next entry = end chainage of previous |
| **Date/time auto-filled** | Current datetime, one tap to adjust |
| **Submit = instant save** | No "Save" then "Submit" — one button does both. Entry goes directly to Submitted. |

### 5.4 Business Rules

1. **No free text for material.** All materials come from Material Master dropdown. Search-as-you-type to find quickly.
2. **Chainage validation.** Start and End must fall within one of the site's defined segments (see §9).
3. **Weight must be positive.** Value must be > 0 (blocked if not).
4. **Duplicate slip warning.** If same slip number exists for this site (any date), non-blocking warning shown.
5. **Backdating limit.** Up to 3 days. Cannot be in the future.
6. **One material per entry.** Each dumping event records one material arrival.
7. **Entry is usable immediately on submit.** No verification gate.
8. **Edit after submit.** Supervisor can edit until verified. Edits audited.
9. **Offline entries.** Queued locally; synced when online. Entry code assigned on sync.

---

## 6. Sub-Module — My Entries

**Purpose:** Supervisor views their own dumping entries, filters by date/state, and edits if needed.

### 6.1 List View

| Column/Info | Notes |
|---|---|
| Material + Variant | "Aggregate 20mm" |
| Net Weight / Qty | "18.50 MT" — large, bold |
| Chainage | "km 3.2 – 3.5 (LHS)" |
| Layer | "GSB" |
| Time | "08:30 AM" or "2h ago" |
| State badge | Draft / Submitted / Verified / Correction Requested |
| Sync indicator | ✅ Synced / 🔄 Pending sync / ❌ Sync failed |

### 6.2 Filters

- **Date:** Today (default) / Yesterday / This Week / Custom.
- **State:** All / Submitted / Verified / Correction Requested.
- **Material:** Filter by specific material.

### 6.3 Entry Detail View

Tap an entry to open the full detail view:
- All fields from the entry form displayed (read-only if verified).
- Photo displayed inline.
- **Verification status section:**
  - If Verified: "✅ Verified by Kiran Shah on 17 Apr 2026, 16:30"
  - If Correction Requested: "⚠️ Correction requested by Kiran Shah: 'Check tare weight — seems too low for this vehicle type.' " — Supervisor can then edit and re-submit.
- **Edit button** (if state = Submitted, not yet Verified).

### 6.4 Correction Flow

1. Senior Engineer reviews the entry in Verification Queue and clicks "Request Correction" with a remark.
2. Supervisor sees a badge on the entry in My Entries: "⚠️ Correction Requested".
3. Supervisor taps the entry, reads the remark, edits the field(s), and re-submits.
4. Entry returns to "Submitted" state in the verification queue.
5. Old values are preserved in audit log.

---

## 7. Sub-Module — Sync Queue (Offline)

**Purpose:** When the supervisor is offline, entries are saved locally and queued for sync. This screen shows queue status and allows retry.

### 7.1 Queue View

| Column | Notes |
|---|---|
| Entry (material + qty) | "Aggregate 20mm — 18.50 MT" |
| Created at | Timestamp of local save |
| Status | Pending / Syncing / Failed / Synced |
| Action | Retry (for failed) / View |

### 7.2 Sync Behavior

- **Auto-sync** when connectivity returns. Background sync every 30 seconds when online.
- **Manual sync** button ("Sync Now") available.
- **Failed entries** show error reason (e.g., "Server error — retry in 5 min").
- **Conflict resolution:** Last-write-wins at entry level. Entries are immutable after verification.
- **Queue persistence:** IndexedDB / SQLite. Survives app restart. Entries retained up to 7 days if unsyncable.

### 7.3 Offline Indicators

| State | UI Indicator |
|---|---|
| **Online** | Green dot in header; normal operation |
| **Offline** | Red dot + "Offline" text in header; yellow banner on Home: "You are offline. Entries will be saved locally and synced when connectivity returns." |
| **Syncing** | Yellow dot + "Syncing..." in header; progress indicator |
| **Sync complete** | Brief green toast: "3 entries synced successfully" |

---

## 8. Offline-First Architecture

### 8.1 What Works Offline

| Feature | Offline? | Notes |
|---|---|---|
| Create new dumping entry | ✅ Yes | Saved to local queue |
| View my entries (already synced) | ✅ Yes | Cached locally |
| View my entries (pending sync) | ✅ Yes | From local queue |
| Edit a pending-sync entry | ✅ Yes | Edit before sync |
| Material / variant / source dropdowns | ✅ Yes | Master data cached on login; refreshed daily when online |
| Chainage segment validation | ✅ Yes | Site segments cached |
| Take photo | ✅ Yes | Stored locally; uploaded on sync |
| Verify entries | ❌ No | Verification is web-only (Senior Engineer) |
| See verification status | ❌ No | Requires server sync |
| Search all site entries | ❌ No | Only own entries cached |

### 8.2 Master Data Cache

On login (and daily refresh when online):
- Material list + variants (only active materials)
- Site details (name, segments, chainage ranges)
- Camps serving this site (names + codes)
- Plants supplying this site (names + codes)
- Vendor list (names)
- Lookup lists (layers/activities, side options)
- Last 20 vehicle numbers

Cache size: estimated < 500 KB for demo scope. No concern.

### 8.3 Photo Handling (Offline)

- Photos captured offline are stored as compressed blobs in IndexedDB.
- On sync, photos upload first, then the entry submits with the attachment reference.
- If photo upload fails but entry sync succeeds, entry is marked "synced (photo pending)".
- Auto-compression to 1080p on capture (client-side).
- Max 5 MB per photo.

---

## 9. Chainage Input & Validation

### 9.1 Input Format

The Supervisor enters chainage as a **number with a unit toggle**:

| Mode | Input | Stored as |
|---|---|---|
| **Kilometres** (default) | `3.5` | 3500 m (integer) |
| **Metres** | `3500` | 3500 m (integer) |

A toggle button switches between km and m. Default: km (more natural for field workers).

### 9.2 Validation Rules

| Rule | Validation | Error Message |
|---|---|---|
| Start < End | Chainage End must be > Chainage Start | "End chainage must be greater than start" |
| Within segment | Both start and end must fall within one of the site's defined segments | "Chainage 28.5 km is outside the site's defined segments (0–27 km)" |
| Non-negative | Start >= 0 | "Chainage cannot be negative" |
| Reasonable range | End − Start ≤ 2000 m (2 km) | Warning (non-blocking): "Are you sure? This entry covers 2.5 km — that's unusually long." |

### 9.3 Site Segment Reference

For Kalol Sanand: segment 0–27,000 m (single segment, 27 km).

If a site had multiple segments (e.g., 1–5 km and 7–15 km), the validation would reject chainage 6.0 km (falls in the gap).

### 9.4 Chainage Auto-Suggest

After the first entry in a session:
- **Next entry's start chainage** = previous entry's end chainage (auto-filled, editable).
- This supports the common workflow where a supervisor moves linearly along the road.

---

## 10. Material & Source Filtering

### 10.1 Material Filter Logic

Not all materials are relevant at every site. The dropdown shows only materials that can arrive at this site:

| Material Source Type (from Material Master) | Shown at Site? |
|---|---|
| Materials with Source = "Camp" | ✅ If a camp serving this site exists |
| Materials with Source = "Plant" | ✅ If a plant supplying this site (directly or via camp) exists |
| Materials with Source = "External Vendor" | ✅ Always (vendors can deliver directly) |

**Example for Kalol Sanand:**
- Camp (Khatraj Chokdi) supplies: Steel, Emulsion, Concrete Mix → shown.
- Plant (Magodi) supplies: BM, DBM, BC, SDBC, MSS, Bitumen, Steel, Emulsion → shown.
- Vendor can supply anything → all raw materials shown.
- Result: all materials appear (demo has comprehensive linkages), but in a larger deployment, this filter becomes important.

### 10.2 Source Filter Logic

When the Supervisor selects a material:
- **Source Camp dropdown** shows only camps that serve this site AND handle this material.
- **Source Plant dropdown** shows only plants that supply this site (or supply a camp that serves this site) AND produce/stock this material.
- **Source Vendor** is always available.

### 10.3 Layer / Activity Filter

The Layer/Activity dropdown is **not filtered** — all layers from the lookup list are shown. The supervisor picks the appropriate one. In phase 2, layers could be filtered based on material type (e.g., "BM" material → auto-suggest "DBM" or "BC" layer).

---

## 11. Weight / Quantity Entry

### 11.1 Single Weight Field (Simplified)

Unlike Camp Inward (which captures Gross/Tare/Net from the weighbridge), the **On-Site dumping form uses a single weight field**. The Supervisor reads the net weight from the weighbridge slip and enters it directly.

**Why no Gross/Tare split here:**
- The Supervisor is not at the weighbridge — they receive the vehicle at site.
- The weighbridge slip (from camp or plant) already shows the net weight.
- Fewer fields = faster entry = less friction in the field.

### 11.2 UoM Auto-Detection

The single field's label and unit change based on the selected material:

| Material Category | UoM Label | Example |
|---|---|---|
| Aggregate, Cement, Sand, Steel, BM, DBM, BC, etc. | **MT** | "Weight: 18.0 MT" |
| Concrete Mix (RMC) | **m³** | "Quantity: 5.0 m³" |
| Pipe | **Numbers** | "Quantity: 25 Nos" |

The form shows one field — "Weight" for weight-based materials, "Quantity" for count/volume-based — with the UoM label beside it.

### 11.3 Secondary UoM Display

For materials with a secondary UoM (Cement: MT→bags, Steel: MT→pieces):
- Below the weight field, show a read-only helper: "≈ 360 bags" or "≈ 1,250 pcs".
- Conversion uses the factor from Material Master.

### 11.4 Machine Field (Phase 2 Placeholder)

A **Machine** dropdown field is planned for phase 2 to capture which machine was used for dumping/laying at this chainage. In the demo:
- The field is shown but **disabled** with a "Coming in Phase 2" hint.
- When enabled, it will show machines assigned to this site (from Machine Assignment module).
- This enables per-machine productivity and fuel-efficiency reporting in phase 2.

---

## 12. Photo Capture

### 12.1 Photo Options

| Action | Notes |
|---|---|
| **Take Photo** | Opens device camera. Auto-compressed to 1080p. |
| **Choose from Gallery** | Opens gallery picker. |
| **Skip** | Photo is optional. |

### 12.2 Constraints

- Max 2 photos per entry (slip + dumping location).
- Max 5 MB per photo (enforced client-side with compression).
- Auto-compression: if photo > 1080p or > 5 MB, compress on capture.
- Photos stored locally when offline; uploaded on sync.
- EXIF data preserved (date/time/GPS for audit).

### 12.3 Photo Display

- Thumbnail shown in the entry form after capture.
- Tap thumbnail to view full-size.
- Delete button on thumbnail to remove.

---

## 13. Verification Flow (Supervisor Side)

### 13.1 How Verification Works (Recap)

Verification is done by the **Senior Engineer** on the web (Module 05). The Supervisor only sees the result:

| State | What Supervisor Sees |
|---|---|
| **Submitted** | Blue badge: "Submitted". Entry is live and usable. |
| **Verified** | Green badge: "Verified ✅". Verifier name + timestamp shown. Entry locked from edits. |
| **Correction Requested** | Orange badge: "⚠️ Correction". Verifier's remark displayed. Edit button available. |
| **Locked** | Grey badge: "Locked 🔒". Period closed. No edits possible. |

### 13.2 Notification on Correction Request

When a Senior Engineer requests a correction:
- If online: push notification / in-app notification to the Supervisor.
- Badge count on "My Entries" tab increments.
- Entry highlighted in the list with orange indicator.

### 13.3 Correction Workflow

1. Supervisor sees "⚠️ Correction Requested" on entry.
2. Taps entry → sees verifier's remark (e.g., "Tare weight seems too low for this vehicle type").
3. Taps "Edit" → modifies the field(s).
4. Taps "Re-submit" → entry returns to Submitted state.
5. Senior Engineer sees it again in their verification queue.

---

## 14. State Machine

Same pattern as Camp Module — consistent across all transactional modules.

```
┌─────────┐     Submit      ┌───────────┐     Verify     ┌──────────┐     Period Close     ┌────────┐
│  Draft   │ ─────────────→ │ Submitted │ ─────────────→ │ Verified │ ──────────────────→ │ Locked │
└─────────┘                 └───────────┘                 └──────────┘                     └────────┘
                                  │                             │
                                  │ Edit (by Supervisor)        │ Unlock (Admin only)
                                  ↓                             ↓
                             Back to Submitted            Back to Submitted
```

### 14.1 Mobile-Specific Notes

- **Draft does not exist in normal mobile flow.** Tapping "Submit" creates the entry directly as Submitted. Draft only exists if the app crashes mid-form (auto-saved partial entry).
- **Offline entries** are created as "Submitted (pending sync)". On sync, they transition to full Submitted on the server.
- **No "locked" indicator needed on mobile** in the demo — period locking is an Admin action that rarely affects daily workflow.

---

## 15. Access Control

### 15.1 Scope

- **Supervisor** is hard-scoped to their assigned site (from User Management, Module 01).
- They can ONLY create entries for their own site.
- They can ONLY see their own entries (not other supervisors' entries at the same site).
- They can ONLY edit entries in Submitted state (not Verified or Locked).

### 15.2 Model Access

| Model | Supervisor | Admin | SE | Verifier |
|---|---|---|---|---|
| Dumping Entry | CRUD (own entries, own site) | CRUD | R (own site) | R (own site scope) |

### 15.3 Data Scope on Mobile

The mobile app only downloads data relevant to the logged-in supervisor:
- Their own entries (for "My Entries" tab).
- Master data for their assigned site (materials, camps, plants, vendors, layers, segments).
- No access to other users' data, other sites' data, or admin functions.

---

## 16. Pre-loaded Demo Data & Walkthrough

### 16.1 Demo Supervisor
**Mahesh Desai** (USR/004) — Supervisor assigned to Kalol Sanand. Language: Gujarati. Mobile app access.

### 16.2 Pre-loaded Dumping Entries (8 entries)

| # | Code | Date | Material | Variant | Chainage | Side | Layer | Source | Vehicle | Slip | Net Wt | State |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | DMP/2026/0001 | 2026-04-15 08:15 | Aggregate | 20mm | 1000–1200m | LHS | GSB | Camp: Khatraj Chokdi | GJ-01-XX-1234 | SLP-0401 | 18.0 MT | Verified |
| 2 | DMP/2026/0002 | 2026-04-15 09:30 | Aggregate | 20mm | 1200–1400m | LHS | GSB | Camp: Khatraj Chokdi | GJ-01-XX-1234 | SLP-0402 | 17.5 MT | Verified |
| 3 | DMP/2026/0003 | 2026-04-15 11:00 | Cement | — | 1000–1200m | LHS | CC | Vendor: Maruti Hirers | GJ-01-XX-5678 | SLP-0403 | 5.0 MT | Verified |
| 4 | DMP/2026/0004 | 2026-04-16 07:45 | GSB | — | 1400–1600m | LHS | GSB | Vendor: Shakti Aggregates | GJ-05-AB-3333 | SLP-0410 | 22.0 MT | Verified |
| 5 | DMP/2026/0005 | 2026-04-16 10:00 | WMM | — | 1000–1200m | LHS | WMM | Plant: Magodi | GJ-01-ZZ-3456 | PLT-0088 | 20.0 MT | Submitted |
| 6 | DMP/2026/0006 | 2026-04-16 14:30 | Steel | 12mm | 1200–1400m | RHS | CC | Camp: Khatraj Chokdi | GJ-01-XX-7890 | CHN-0203 | 3.0 MT | Submitted |
| 7 | DMP/2026/0007 | 2026-04-17 08:00 | Aggregate | 20mm | 1600–1800m | LHS | GSB | Camp: Khatraj Chokdi | GJ-01-ZZ-7777 | CHN-0210 | 8.0 MT | Submitted |
| 8 | DMP/2026/0008 | 2026-04-17 09:45 | Pipe | 600mm | 2000–2100m | LHS | Pipe-laying | Vendor: Maruti Hirers | GJ-01-XX-1234 | SLP-0420 | 12 Nos | Correction Requested |

### 16.3 Correction Request Example (DMP/2026/0008)

- **Verifier remark (Kiran Shah):** "Quantity seems low for 600mm pipes over 100m. Expected ~25 based on spacing. Please verify count."
- **Expected supervisor action:** Mahesh checks the physical count, either confirms 12 is correct with a remark, or corrects to the actual count.

### 16.4 Demo Walkthrough (Suggested)

1. **Log in as Mahesh Desai (Supervisor)** → Home screen shows today's tiles (2 entries, 28 MT dumped).
2. **Tap "New Dumping Entry"** → form opens with Kalol Sanand pre-filled.
3. **Record Aggregate 20mm from Khatraj Chokdi camp:**
   - Material: Aggregate → Variant: 20mm → Layer: GSB.
   - Chainage: 1.8 km – 2.0 km, Side: LHS.
   - Source: Camp → Khatraj Chokdi. Source Ref: CHN-0210.
   - Vehicle: GJ-01-ZZ-7777, Slip: SLP-0425.
   - Weight: 18.0 MT (from weighbridge slip).
   - Take photo of slip.
   - Submit → success toast.
4. **View My Entries** → see the new entry at top with "Submitted" badge.
5. **Tap entry DMP/2026/0008 (Pipe)** → see "⚠️ Correction Requested" with Kiran Shah's remark. Edit quantity to 25. Re-submit.
6. **Simulate offline:**
   - Toggle offline indicator to red.
   - Create another entry → saved locally with "Pending Sync" badge.
   - Open Sync Queue → show "1 pending".
   - Toggle back to online → auto-sync → success toast.

---

## 17. Design Decisions & Assumptions

| # | Decision | Rationale |
|---|---|---|
| 1 | **Mobile-first, single-screen form** | Supervisor is standing in sun with a vehicle waiting. No multi-step wizards. |
| 2 | **No stock at site** | Site is a consumption point, not storage. Material arrives and is dumped. |
| 3 | **One material per entry** | Consistent with Camp Module. One vehicle trip = one material type = one entry. |
| 4 | **No returns/rejections in demo** | Keeps the mobile app simple. Add in phase 2. |
| 5 | **Submit = instant Submitted** | No Draft→Submit two-step on mobile. One tap to submit. Draft only for crash recovery. |
| 6 | **Chainage in km (default)** | Workers think in km, not metres. System stores in metres internally. |
| 7 | **Last-used values remembered** | Speed optimization — most entries in a session are similar material/source/side. |
| 8 | **Chainage auto-suggest** | Next entry starts where previous ended — linear road progression. |
| 9 | **Backdating up to 3 days** | Consistent with Camp Module. Covers late entries from previous day. |
| 10 | **Photo optional** | Making it mandatory would slow down every entry. Supervisors take photos when they can. |
| 11 | **Camp outward ↔ site dumping NOT auto-matched** | Independent entries, same as Camp Module decision. Match by slip number in reports. |
| 12 | **Supervisor sees only own entries** | Privacy + scope control. Senior Engineer sees all entries for the site. |
| 13 | **PWA recommended for demo** | No app store friction. Service worker for offline. Revisit for production. |
| 14 | **Single weight field (no Gross/Tare)** | Supervisor reads net weight from weighbridge slip. No weighbridge at site — splitting into Gross/Tare adds friction without value here. Camp Inward retains Gross/Tare because it's the receiving point. |
| 15 | **Machine field as Phase 2 placeholder** | Machine assignment to dumping enables per-machine productivity analytics. Shown disabled in demo UI so client sees it's planned. Will pull from Machine Assignment module. |
| 16 | **Source Vendor can be free text** | Unlike Camp inward (where vendor must be from master), one-off vendors may deliver directly to site. Allow free text with autocomplete from vendor master. |

---

## 18. Open Items / Notes for Developers

1. **Mobile technology decision** — Client to confirm PWA vs native (React Native / Flutter). iOS needed? Impacts offline reliability and camera access.
2. **Language localization** — Gujarati / Hindi UI strings needed. Confirm which languages are priority.
3. **Push notifications** — For correction requests. Requires Firebase (Android) or APNs (iOS) if native. Web Push if PWA.
4. **Offline queue retry strategy** — Exponential backoff on failed sync (30s → 1m → 5m → 15m). Alert user if entry is unsyncable for > 6 hours.
5. **Photo upload reliability** — Photos are large. If connectivity drops mid-upload, retry the photo separately. Entry can sync without photo first.
6. **Chainage overlap detection** — Should the system warn if two entries at the same chainage + side + layer exist on the same day? This could indicate a duplicate. Non-blocking warning.
7. **Entry code assignment** — On offline, use a client-generated UUID. On sync, server assigns the sequential `DMP/YYYY/####` code. The UUID is retained as an internal reference.
8. **Batch sync** — When many entries are queued (e.g., 20 entries from a full day offline), sync should batch them (5 at a time) to avoid timeout.
9. **Session timeout** — Mobile app should not log out the user aggressively. Token refresh every 24h. Offline entries are tied to the user even if session expires — they sync on next login.
10. **Supervisor handoff** — If Supervisor A starts a shift and Supervisor B takes over, each logs in with their own account. No shared accounts.
11. **Minimum Android version** — Android 10+ (API 29). Covers ~95% of devices in the field.
12. **Screen size** — Design for 5.5"–6.5" screens (standard Android). Test on 5.0" minimum.
13. **Dark mode** — Not needed for demo. Outdoor use = high brightness, light mode is better.
14. **Accessibility** — Minimum touch target 48x48dp. High contrast text. No reliance on color alone for state.
15. **Machine field (Phase 2)** — When enabled, the Machine dropdown should pull active assignments for this site from `mcway.machine.assignment`. UI is already present (disabled). Backend field `machine_id` (Many2one → Machine) should be added to the model now (nullable) so no migration is needed when Phase 2 activates it. Consider auto-suggesting the last-used machine for speed.
