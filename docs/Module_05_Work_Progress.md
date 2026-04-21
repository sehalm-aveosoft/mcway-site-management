# McWay Site Management — Module 05: Senior Engineer Work Progress
## Detailed Specification for Odoo Implementation

**Version:** 1.0 (initial draft)
**Date:** 2026-04-21
**Module Owner:** Senior Engineer
**Platform:** Web (Odoo)
**Prerequisite:** Module 01 (Site Master, Lookup Lists — Layers), Module 03 (On-Site Dumping entries feed the Verification Queue), Module 02 & 04 (entries from camp/plant may also route to SE for verification when they affect a site the SE owns).

---

## Table of Contents
1. [Purpose & Scope](#1-purpose--scope)
2. [Odoo Implementation Strategy](#2-odoo-implementation-strategy)
3. [Menu & Navigation Structure](#3-menu--navigation-structure)
4. [Home / Dashboard](#4-home--dashboard)
5. [Sub-Module — Progress Entry (Method A)](#5-sub-module--progress-entry-method-a)
6. [Sub-Module — Chainage Strip View](#6-sub-module--chainage-strip-view)
7. [Sub-Module — Layer-Wise Summary](#7-sub-module--layer-wise-summary)
8. [Sub-Module — Progress History](#8-sub-module--progress-history)
9. [Sub-Module — Consolidated Verification Queue](#9-sub-module--consolidated-verification-queue)
10. [Method A vs Method B (Design Note)](#10-method-a-vs-method-b-design-note)
11. [Segment Model & Layer Hierarchy](#11-segment-model--layer-hierarchy)
12. [State Machine](#12-state-machine)
13. [Access Control](#13-access-control)
14. [Pre-loaded Demo Data & Walkthrough](#14-pre-loaded-demo-data--walkthrough)
15. [Design Decisions & Assumptions](#15-design-decisions--assumptions)
16. [Open Items / Notes for Developers](#16-open-items--notes-for-developers)

---

## 1. Purpose & Scope

The **Work Progress Module** is the Senior Engineer's (SE) command centre. It answers two questions every day: *"how much road is done?"* and *"are the supervisors' entries accurate?"*

The SE is responsible for:
1. **Recording work progress** against the site's planned chainage, layer by layer (GSB → WMM → BM → DBM → BC → SDBC → CC).
2. **Visualising progress** as a colour-coded chainage strip — a single screen that shows at a glance what's done, what's in progress, and what hasn't started.
3. **Verifying downstream entries** — dumping entries from Supervisors (Module 03), and — when relevant — camp/plant entries that flow into the SE's assigned site.

### In Scope
- **Progress Entry** — layer-wise progress per chainage segment (Method A: status = Not Started / In Progress % / Complete).
- **Chainage Strip View** — interactive colour-coded grid: X = chainage, Y = layers, cell colour = status.
- **Layer-Wise Summary** — cumulative completion per layer (e.g., GSB 80%, WMM 60%).
- **Cumulative Site Progress** — overall site completion %.
- **Progress History** — audit-style list of all progress entries.
- **Consolidated Verification Queue** — all submitted entries across modules that require SE sign-off, for the sites this SE owns.

### Out of Scope
- **Planned quantity / estimation** — lives in Admin Portal → Estimation sub-module (Method B prerequisite). Progress here is entered as % or status, not quantity-based, in the demo.
- **Rework / snag list** — not in demo scope.
- **BOQ / billing** — separate concern; Reports module surfaces progress, but billing is phase 2.
- **Machine productivity** — deferred to phase 2 (depends on per-machine capture, which is itself Phase 2).

---

## 2. Odoo Implementation Strategy

| Entity | Odoo Base | Strategy |
|---|---|---|
| Progress Entry | Custom `mcway.progress.entry` | One record per chainage-segment × layer × date; state-managed |
| Chainage Segments | Derived from Site Master multi-segment chainage + configurable bucket size | Materialized on site creation / segment edit; one `mcway.chainage.bucket` per bucket |
| Layer Lookup | Lookup list (Module 01) | GSB / WMM / BM / DBM / BC / SDBC / CC — order matters (construction sequence) |
| Strip View | Custom QWeb + JS grid | Client-side render; server supplies bucket × layer matrix as JSON |
| Verification Queue | Union query across `mcway.dumping.entry`, `mcway.camp.*`, `mcway.plant.*` | Filtered to entries whose site (direct or via camp/plant linkage) is assigned to this SE |

### Dependencies (additional to Modules 01–04)
`web`, `mail`, `auditlog` — already in place.

### Performance Requirement
Per PRD: strip rendering for 27 km site with 7 layers (~1,890 cells) must complete ≤ 1.5 s. Approach: a single SQL query returning `{bucket_id, layer_id, status, percent}` with client-side painting on canvas or CSS grid.

---

## 3. Menu & Navigation Structure

The SE sees a scoped menu. They may be assigned to **one or more sites** (unlike the Supervisor who is one-to-one). Work progress is scoped to their assigned sites only.

```
📐 Work Progress
├── Home (Dashboard)
├── Site Selector (visible if SE owns >1 site)
├── Progress Entry
│   ├── New Entry
│   └── Bulk Entry (grid)          ← fast path; see §5.5
├── Chainage Strip View            ← hero visual
├── Layer-Wise Summary
├── Progress History
├── Verification Queue
│   ├── Dumping Entries
│   ├── Camp Entries (if SE's site is served by any camp)
│   └── Plant Entries (if SE's site is supplied by any plant)
└── Reports (site-level progress)
    ├── Daily Progress Report
    └── Cumulative Layer Report
```

> **Note:** SE does NOT see Admin menus, cannot create sites, and cannot edit Supervisor or Camp In-charge data — only verify or request correction.

---

## 4. Home / Dashboard

**Purpose:** Single-screen snapshot of the SE's sites.

### 4.1 Site Context Strip (Persistent)
If the SE has multiple sites, a dropdown/pill bar at the top lets them switch. For single-site SEs, this is a read-only label.

### 4.2 Summary Tiles

| Tile | Value | Source |
|---|---|---|
| Cumulative Site Progress | `%` overall | Weighted avg of all layers (see §7.3) |
| Today's Progress Entries | Count | Progress entries (today) |
| Layers at Risk | Count where behind schedule (if target exists) | §7.4 |
| Pending Verification | Count across all sub-queues | §9 |
| Dumping Entries This Week | Count (all states) | Module 03 feed |
| Active Chainage | e.g., "km 1.2–2.0" (where most recent progress activity happened) | Last N entries |
| Unverified Dumpings | Count (stale > 2d highlighted) | Module 03, state=submitted |

### 4.3 Mini Strip (Top of Home)
A compressed version of the Chainage Strip View — one-line-per-layer, hover for detail, click to open full Strip View.

### 4.4 Quick Actions
- **+ New Progress Entry** — opens Progress Entry form.
- **Open Strip View** — full strip visualisation.
- **Review Pending** — opens Consolidated Verification Queue.

### 4.5 Recent Activity Feed
Last 10 progress entries + last 5 verification actions (verify / correction request). Each row: timestamp, type icon, description, state badge.

---

## 5. Sub-Module — Progress Entry (Method A)

**Purpose:** Record layer-wise progress at a specific chainage segment.
**Odoo Model:** `mcway.progress.entry`.

### 5.1 Form Fields

| # | Field | Type | Mandatory | Notes |
|---|---|---|---|---|
| 1 | Entry Code | Char | Auto | `PRG/YYYY/####` |
| 2 | Site | Many2one → Site | Auto / Select | Pre-filled if SE owns one site; dropdown if many |
| 3 | Date | Date | ✓ | Defaults to today; **backdating allowed up to 3 days** |
| 4 | **Chainage Start** | Integer (metres) | ✓ | Must align to segment boundary (snaps to nearest bucket) |
| 5 | **Chainage End** | Integer (metres) | ✓ | > Start; can span multiple buckets (see §5.3) |
| 6 | **Side** | Select | ✓ | LHS / RHS / Both / Lane / Median |
| 7 | **Layer** | Many2one → Layer lookup | ✓ | GSB / WMM / BM / DBM / BC / SDBC / CC |
| 8 | **Status** | Selection | ✓ | Not Started / In Progress / Complete |
| 9 | **Percent Complete** | Integer (0–100) | Conditional | Required if Status = In Progress. Auto = 0 if Not Started, 100 if Complete. |
| 10 | Photo(s) | Binary (multi) | ✗ | Up to 4 photos. Max 5 MB each, auto-compressed to 1080p. |
| 11 | Remarks | Text | ✗ | — |
| 12 | State | Selection | Auto | Draft → Submitted → Verified → Locked (see §12) |

### 5.2 Status → Percent Mapping

| Status | Percent | UX |
|---|---|---|
| **Not Started** | 0 | Percent field hidden |
| **In Progress** | 1–99 | Percent field shown, mandatory; slider + number input |
| **Complete** | 100 | Percent field hidden (auto 100%) |

### 5.3 Chainage Span & Bucketing

Progress is stored at the **bucket level** (default 100 m). If the SE enters an entry spanning multiple buckets:

- Example: Chainage 1200–1600 m, Layer = GSB, Status = Complete.
- The system **creates or updates 4 bucket records** (1200–1300, 1300–1400, 1400–1500, 1500–1600), each stamped Complete @ 100%.
- Each bucket record is atomic — if a subsequent entry updates just one bucket, the others are untouched.

**Storage model:** `mcway.progress.bucket` — one row per `(site, bucket_start, side, layer)`. The `mcway.progress.entry` form creates/updates these buckets on save and stores the *entry* as an audit record pointing at the affected buckets.

### 5.4 Side Handling

- If Side = **Both**, the bucket record is flagged as covering both LHS and RHS (logical union).
- If Side = **Lane** or **Median**, separate rows are stored for those carriageway parts (matches Module 01 multi-side Site Master).
- The strip view can be filtered to a specific side or show combined.

### 5.5 Bulk Entry Grid (Speed Path)

For the common case of "I completed layer X from chainage A to B this morning", a **grid-entry form** is offered:

```
Date: [2026-04-21]   Site: [Kalol Sanand]   Side: [LHS ▼]   Layer: [GSB ▼]

   Chainage (each cell = 100 m bucket)
   ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
   │ 0.0│ 0.1│ 0.2│ 0.3│ 0.4│ 0.5│ 0.6│ 0.7│ 0.8│ 0.9│  ← km markers
   │ ✅ │ ✅ │ ✅ │ 🟡 │ ⬜ │ ⬜ │ ⬜ │ ⬜ │ ⬜ │ ⬜ │
   │100 │100 │100 │ 60 │  0 │  0 │  0 │  0 │  0 │  0 │  ← % (editable)
   └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘

   [Select range: drag across cells] [Set %: 100] [Apply]
```

- Click or drag cells to select.
- Set status/percent on selected cells in one action.
- One save = many `progress.bucket` rows updated.
- Works offline is **not required** — this is a web screen, not mobile.

### 5.6 Business Rules

1. **Chainage within site segments.** Entry chainage must fall within the site's defined segments (reuses Module 01 validation).
2. **Layer order warning (non-blocking).** If the SE records "BC complete" for a bucket where "BM" is still 40%, show a non-blocking warning: "BM at this chainage is 40%. Typical sequence is GSB → WMM → BM → DBM → BC → SDBC → CC. Continue?"
3. **Percent range.** 0–100 integer. In Progress status forces 1–99 (not 0 or 100).
4. **Monotonic percent (recommended, non-blocking).** If a bucket's layer was 60% yesterday and today's entry sets it to 40%, warn: "Progress is moving backwards for this bucket/layer. Possible correction — proceed?"
5. **Photo recommended** (not mandatory) for Complete status — the verifier relies on evidence.
6. **Backdating limit:** 3 days past; no future dates.
7. **On submit:** State → Submitted. Bucket(s) updated immediately. Appears in Verification Queue (self-verified for progress? see §13.1).
8. **Edit after submit:** SE can edit until verified (self-sign-off or admin sign-off — see §12).

---

## 6. Sub-Module — Chainage Strip View

**Purpose:** The hero visual. A colour-coded grid where each cell represents one 100 m bucket × one layer. Shows the entire site's progress at a glance.
**Odoo Model:** Server query, client-side render.

### 6.1 Layout

```
        km 0.0  0.5   1.0   1.5   2.0   2.5   3.0   3.5   4.0 …
       ┌───────────────────────────────────────────────────────┐
  GSB  │███████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
       │100 100 100 100 60  0   0   0   0   0   0   0   …      │
  WMM  │█████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
  BM   │███████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
  DBM  │███░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
  BC   │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
  SDBC │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
  CC   │███░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
       └───────────────────────────────────────────────────────┘
```

### 6.2 Cell States & Colours

| Status | Colour | Example |
|---|---|---|
| Not Started | Light grey (`#e0e2e7`) | 0% |
| In Progress 1–33% | Red-orange (`#f4a261`) | 20% |
| In Progress 34–66% | Yellow (`#f4c430`) | 50% |
| In Progress 67–99% | Light green (`#8bc34a`) | 80% |
| Complete 100% | Dark green (`#198754`) | 100% |
| Not Applicable | Hatched pattern | e.g., layer CC not planned for this segment |

### 6.3 Interactions

| Action | Behavior |
|---|---|
| Hover cell | Tooltip: "GSB @ km 0.3–0.4 (LHS) — In Progress 60% — last updated 2026-04-19 by Kiran Shah" |
| Click cell | Opens a side drawer with the full bucket history (all entries affecting this bucket × layer) |
| Drag across cells | Selects a range; "Quick Update Status" button appears |
| Layer row click | Filters Layer-Wise Summary to that layer |
| km marker click | Opens Progress History filtered to that chainage ± 100 m |

### 6.4 Filters
- **Side:** All / LHS / RHS / Lane / Median (tabs)
- **Date Cursor:** "as of today" (default) / "as of 2026-04-15" — see §6.5
- **Layers visible:** multi-select chips to show/hide specific layers
- **Zoom:** 100 m / 500 m / 1 km bucket aggregation for long sites

### 6.5 Time Travel
A date cursor shows the site's state as of that date. Server query filters progress buckets to `date ≤ cursor`. Useful for retrospective analysis and client reporting.

### 6.6 Export
- **PNG** — strip rendered to image (for WhatsApp / email to owner)
- **PDF** — landscape, with header (site name, date, overall %) and the strip + layer summary table

---

## 7. Sub-Module — Layer-Wise Summary

**Purpose:** Numeric roll-up per layer across the whole site.

### 7.1 Summary Table

| Layer | Buckets Total | Not Started | In Progress (avg %) | Complete | Overall % |
|---|---|---|---|---|---|
| GSB | 270 | 240 | 8 @ 55% | 22 | 9.8% |
| WMM | 270 | 260 | 3 @ 40% | 7 | 3.0% |
| BM | 270 | 267 | 2 @ 30% | 1 | 0.6% |
| DBM | 270 | 270 | 0 | 0 | 0.0% |
| BC | 270 | 270 | 0 | 0 | 0.0% |
| SDBC | 270 | 270 | 0 | 0 | 0.0% |
| CC | 270 | 267 | 0 | 3 | 1.1% |

### 7.2 Cell Calculation
`Overall % = (Σ percent across all buckets for this layer) / (bucket count × 100) × 100`

Layers can be **excluded per segment** (e.g., "CC not planned for segment 2") — those buckets are counted as "N/A" and removed from the denominator.

### 7.3 Cumulative Site Progress
Weighted average across all planned layers:
```
Cumulative % = Σ (layer weight × layer %) / Σ (layer weights)
```
Layer weights are configurable in Admin → System Thresholds (default: all equal). Typical weighting may give more weight to BC / DBM (costlier layers).

> **Pending from client:** whether to enable weighted cumulative or simple average. Demo uses equal weights.

### 7.4 Behind-Schedule Flagging
If Admin defines target dates per layer (stretch / P0 goal), the summary shows `🔴 behind by X days` or `🟢 on track`. **Deferred** for demo — not all sites will have target dates loaded.

---

## 8. Sub-Module — Progress History

**Purpose:** Chronological audit list of all progress entries.

### 8.1 List View

| Column | Notes |
|---|---|
| Entry Code | `PRG/2026/0042` |
| Date | — |
| Chainage | km 1.2–1.6 |
| Side | LHS / RHS / Both / Lane / Median |
| Layer | badge with colour |
| Status | badge (Not Started / In Progress X% / Complete) |
| Photos | 📷 × 2 (count) |
| Entered By | User name |
| State | Draft / Submitted / Verified / Locked |

### 8.2 Filters & Sorting
- Date range
- Layer
- Side
- State
- Chainage range
- Entered by (for sites with multiple SEs)

### 8.3 Drill-Down
Click an entry → read-only detail view with full field set, photos, affected buckets list, audit log (who edited when).

---

## 9. Sub-Module — Consolidated Verification Queue

**Purpose:** Single screen where the SE reviews entries across all downstream modules that affect their assigned site(s).
**This is the SE's verification role** (PRD F-E03).

### 9.1 What Reaches This Queue

| Source Module | Routes to SE When |
|---|---|
| **Module 03 — On-Site Dumping** | SE's site = the dumping entry's site |
| **Module 02 — Camp Module** | A camp that serves the SE's site (inward / outward / RMC / labor). Optional — Admin configurable |
| **Module 04 — Plant Module** | A plant that supplies the SE's site (inward / production / outward). Optional — Admin configurable |

> **Design decision:** Dumping entries always route to SE (they happen on the site). Camp/Plant routing is **opt-in per site** via a System Threshold "SE verifies upstream entries for this site: yes/no". For the demo, enabled for Kalol Sanand.

### 9.2 Queue View

Tabs by source module. Each tab:

| Column | Notes |
|---|---|
| Entry Code | — |
| Type | Dumping / Camp Inward / Camp Outward / RMC / Labor / Plant Raw / Production / Plant Outward |
| Date | — |
| Description | Material, qty, chainage / camp / plant |
| Entered By | — |
| **Age** | "2h ago", "1d 4h" |
| Flags | ⚠️ variance / 📷 no photo / 🕒 stale |
| Actions | Verify / Request Correction |

### 9.3 Verification Actions

| Action | Effect |
|---|---|
| **Verify** | State → Verified. Verifier name + timestamp recorded. Removed from queue. |
| **Request Correction** | Remark sent to submitter. Entry stays in Submitted. Submitter sees badge on their entry and can edit + re-submit. |
| **Bulk Verify** | Select multiple → "Verify Selected". Useful for end-of-day review. |

### 9.4 Filters
- Tab (source module)
- Date range
- Flagged only (⚠️)
- Stale only (> 2 days, highlighted orange)
- Material / Layer / Chainage (for dumping)

### 9.5 Rules (Recap)
1. **Verification never blocks** — entries are already live in stock, reports, dashboard.
2. **Verifier cannot edit** — only verify or request correction.
3. **Audit trail:** verifier, timestamp, IP (via `mail.thread` + `auditlog`).
4. **Re-verification not required** after Supervisor re-submits a corrected entry — but the entry does re-appear in the queue.

---

## 10. Method A vs Method B (Design Note)

Per Demo Documentation §9.1 and PRD Epic E, two methods are on the table.

| Aspect | Method A (Status / %) — recommended | Method B (Qty-based) |
|---|---|---|
| Inputs | Status (Not Started / In Progress % / Complete) | Planned Qty (from Estimation) + Executed Qty |
| % calculation | Direct | `Executed / Planned × 100` |
| Prerequisite | Layer lookup only | Estimation module with planned quantities |
| Objectivity | SE's judgement | Quantity-based, more objective |
| Demo readiness | ✅ Ready now | ❌ Needs Estimation module loaded with planned values |

**Decision for demo:** Method A. Method B is a phase-2 upgrade path — the `progress.bucket` schema includes optional `planned_qty` and `executed_qty` fields so Method B can layer on without migration.

> **Pending from client:** confirm Method A is acceptable for demo. Ratify the phase-2 path to Method B.

---

## 11. Segment Model & Layer Hierarchy

### 11.1 Chainage Bucketing

| Site | Chainage Span | Bucket Size | # Buckets |
|---|---|---|---|
| Kalol Sanand (demo) | 0–27,000 m | 100 m | 270 |

Bucket size is configurable per site in Module 01 Site Master (default 100 m). Options: 50 m / 100 m / 250 m / 500 m. Smaller = more granular but heavier strip view.

### 11.2 Layer Hierarchy

Standard road construction sequence (bottom to top):

| Order | Layer | Full Name | Typical % of total cost |
|---|---|---|---|
| 1 | **GSB** | Granular Sub-Base | ~8% |
| 2 | **WMM** | Wet Mix Macadam | ~12% |
| 3 | **BM** | Bituminous Macadam | ~18% |
| 4 | **DBM** | Dense Bituminous Macadam | ~22% |
| 5 | **BC** | Bituminous Concrete | ~25% |
| 6 | **SDBC** | Semi-Dense Bituminous Concrete | ~10% |
| 7 | **CC** | Cement Concrete (culverts, drains, shoulders — often partial) | ~5% |

> **Note:** SDBC is either applied OR BC is — rarely both. Configurable per segment in Admin → Site Master → Layer Plan (phase 2). For demo, all layers are available on all segments.

### 11.3 Layer × Bucket × Side Matrix

For Kalol Sanand (27 km, 7 layers, LHS+RHS):
`270 buckets × 7 layers × 2 sides = 3,780 bucket rows`

Strip view renders one side at a time by default (LHS shown first). Combined view merges LHS+RHS (AND = both must be done; OR = either suffices — configurable).

---

## 12. State Machine

Progress entries follow the same state model as other modules, with one wrinkle: **SE can self-verify** since they are themselves the verifier. For cross-check, Admin can configure a secondary verifier (typically Owner or another SE).

```
┌─────────┐     Save/Submit     ┌───────────┐      Verify      ┌──────────┐     Period Close     ┌────────┐
│  Draft   │ ─────────────────→ │ Submitted │ ───────────────→ │ Verified │ ──────────────────→ │ Locked │
└─────────┘                     └───────────┘                   └──────────┘                     └────────┘
                                      │                               │
                                      │ Edit (by SE)                  │ Unlock (Admin only)
                                      ↓                               ↓
                                 Back to Submitted               Back to Submitted
```

### 12.1 Self-Verification

| Site Setting | Who verifies SE progress entries |
|---|---|
| "SE self-verifies" (default) | SE. Entries auto-advance to Verified on submit. |
| "Secondary verifier required" | Admin or designated user. Entries stay Submitted until secondary verifies. |

For **verification of downstream entries** (dumping, camp, plant), SE is always the verifier. Self-verification does not apply — those entries originate from other users.

### 12.2 Notes
- **Stock/quantity impact is not relevant here** — progress entries don't touch stock.
- **Report impact starts at Submitted** — Submitted entries already contribute to cumulative % in dashboards. Verification is accountability, not a gate.
- **Edits after verified** require Admin unlock, same pattern as other modules.

---

## 13. Access Control

### 13.1 Model-Level Access

| Model | SE (own sites) | Admin | Owner | Supervisor | Camp | Plant | Verifier |
|---|---|---|---|---|---|---|---|
| Progress Entry | CRUD (own sites) | CRUD | R | R (own site) | — | — | R |
| Progress Bucket | CRUD (via entry) | CRUD | R | R | — | — | R |
| Strip View | R (own sites) | R | R | R (own site) | — | — | R |
| Verification Queue | Verify (own sites) | Verify | — | — | — | — | Verify |

### 13.2 Record Rules
- SE sees only data for their assigned site(s) — enforced via `ir.rule` on `user.assignment_target` (polymorphic: may be multi-site).
- SE can verify entries routed to their sites (dumping direct; camp/plant via linkage + System Threshold).
- Admin sees all sites.
- Owner is read-only.

### 13.3 Multi-Site SE

Unlike Supervisor / Camp In-charge / Plant In-charge (strictly one-to-one), an SE may own multiple sites. Assignment Type on the user record supports `[Site(s)]` (Many2many) specifically for the SE role.

> **Demo:** Kiran Shah is assigned only to Kalol Sanand (single site). Multi-site is structurally supported but not demonstrated.

---

## 14. Pre-loaded Demo Data & Walkthrough

### 14.1 Demo Senior Engineer
**Kiran Shah** (`USR/005`) — role = Senior Engineer. Assigned site: Kalol Sanand. Preferences: metric km display, strip side = LHS by default.

### 14.2 Demo Site
**Kalol Sanand** — 27 km (0–27,000 m), single segment, LHS + RHS. Bucket size = 100 m → 270 buckets per side.

### 14.3 Pre-loaded Progress Entries (12 entries)

| # | Code | Date | Chainage | Side | Layer | Status | % | Photos | State |
|---|---|---|---|---|---|---|---|---|---|
| 1 | PRG/2026/0001 | 2026-04-10 | 0–1000 m | LHS | GSB | Complete | 100 | 2 | Verified |
| 2 | PRG/2026/0002 | 2026-04-11 | 1000–2000 m | LHS | GSB | Complete | 100 | 1 | Verified |
| 3 | PRG/2026/0003 | 2026-04-12 | 2000–2500 m | LHS | GSB | In Progress | 60 | 2 | Verified |
| 4 | PRG/2026/0004 | 2026-04-13 | 0–800 m | LHS | WMM | Complete | 100 | 1 | Verified |
| 5 | PRG/2026/0005 | 2026-04-14 | 800–1200 m | LHS | WMM | In Progress | 70 | 2 | Verified |
| 6 | PRG/2026/0006 | 2026-04-15 | 0–500 m | LHS | BM | Complete | 100 | 3 | Verified |
| 7 | PRG/2026/0007 | 2026-04-16 | 500–700 m | LHS | BM | In Progress | 40 | 1 | Verified |
| 8 | PRG/2026/0008 | 2026-04-17 | 0–300 m | LHS | DBM | In Progress | 50 | 1 | Submitted |
| 9 | PRG/2026/0009 | 2026-04-18 | 0–300 m | LHS | CC | Complete | 100 | 1 | Submitted |
| 10 | PRG/2026/0010 | 2026-04-19 | 1500–1700 m | RHS | GSB | Complete | 100 | 0 | Submitted |
| 11 | PRG/2026/0011 | 2026-04-20 | 0–200 m | LHS | BC | In Progress | 25 | 2 | Submitted |
| 12 | PRG/2026/0012 | 2026-04-21 | 200–400 m | LHS | DBM | In Progress | 30 | 1 | Draft |

### 14.4 Expected Layer-Wise Summary (LHS, after demo data)

| Layer | Buckets Complete | In Progress (avg) | Not Started | Overall % |
|---|---|---|---|---|
| GSB | 20 | 5 @ 60% | 245 | 8.5% |
| WMM | 8 | 4 @ 70% | 258 | 4.0% |
| BM | 5 | 2 @ 40% | 263 | 2.1% |
| DBM | 0 | 5 @ 40% | 265 | 0.7% |
| BC | 0 | 2 @ 25% | 268 | 0.2% |
| SDBC | 0 | 0 | 270 | 0.0% |
| CC | 3 | 0 | 267 | 1.1% |

**Cumulative LHS site progress (equal weights):** ~2.4%.

### 14.5 Verification Queue (at demo time)

| Tab | Count | Highlights |
|---|---|---|
| Dumping (Module 03) | 5 | 1 correction requested (pipe count); 1 stale (>24h) |
| Camp Entries | 4 | 1 stale (shuttering return >24h) |
| Plant Entries | 5 | 1 flagged ⚠️ variance (BC batch 10.7%); 1 stale |
| **Total** | **14** | |

### 14.6 Demo Walkthrough (Suggested)

1. **Log in as Kiran Shah (SE)** → Home dashboard shows cumulative ~2.4%, mini strip, 14 pending verifications.
2. **Open Chainage Strip View** → full LHS grid for Kalol Sanand. Hover over GSB @ km 0.2–0.3 → tooltip shows 100% complete, entered by Kiran on 2026-04-10.
3. **Drag across GSB cells km 2.5–3.0** → "Quick Update Status" appears → set 50% → save. Strip updates live.
4. **Open Progress Entry → Bulk Entry Grid** → show the grid input, drag & apply % for WMM km 1.2–1.5.
5. **Switch side to RHS tab** → show the RHS strip (mostly empty — only entry 10 covered 1500–1700 GSB).
6. **Open Layer-Wise Summary** → show per-layer %, drill into GSB → filtered progress history.
7. **Open Verification Queue → Dumping tab** → bulk-verify 3 entries, click "Request Correction" on the pipe-count entry → add remark → submit.
8. **Switch to Camp Entries tab** → verify the shuttering return entry.
9. **Switch to Plant Entries tab** → open the flagged BC batch, review variance, verify anyway with a remark.
10. **Export Strip View as PNG** → download → "share with Owner / WhatsApp".

---

## 15. Design Decisions & Assumptions

| # | Decision | Rationale |
|---|---|---|
| 1 | **Method A for demo** (status + %) | Simpler to launch; Method B needs Estimation loaded first |
| 2 | **100 m default bucket size** | Granular enough for accountability, coarse enough to render fast (≤1.5s) |
| 3 | **Bucket × Layer × Side as the storage grain** | Supports all visualisation and roll-up queries cleanly |
| 4 | **Strip is the hero UI** | Demo doc §9.3 and PRD F-E02 both emphasise visual progress; client owner will look here first |
| 5 | **Layer-order warning is non-blocking** | Real sites sometimes lay layers out of order (staging, weather); don't force sequence |
| 6 | **Monotonic percent warning is non-blocking** | Corrections should be allowed; warn but don't block |
| 7 | **SE can self-verify progress entries** | SE is the recording authority; blocking their own entries is circular. Secondary verifier is optional per site. |
| 8 | **SE verifies downstream entries via consolidated queue** | One screen, three tabs — reduces context switching for SE |
| 9 | **Camp/Plant routing is opt-in per site** | Some sites won't need SE sign-off on every camp/plant entry — Admin configures |
| 10 | **Bulk entry grid** | The #1 speed optimisation — 100 m × 7 layers = fiddly without bulk input |
| 11 | **Strip export to PNG/PDF** | Real outcome for the demo: client wants to share progress with owner via WhatsApp |
| 12 | **Equal layer weights by default** | Client hasn't provided weighted cost breakdown yet; defer |
| 13 | **Time-travel date cursor** | Retrospective analysis is a strong "wow" moment for management |
| 14 | **N/A layer option via hatched cell** | Some segments genuinely don't have certain layers (e.g., CC only on culverts) |
| 15 | **Multi-site SE supported, not demonstrated** | Structural support is free; demo keeps one SE = one site for clarity |
| 16 | **Backdating limit 3 days** | Consistent with Modules 02 / 03 / 04 |

---

## 16. Open Items / Notes for Developers

1. **Method B activation** — keep `planned_qty` and `executed_qty` nullable columns on `progress.bucket`. When Method B activates, migrate by populating `planned_qty` from the Estimation module; no schema change needed.
2. **Layer weights** — configurable in Admin → System Thresholds. Default = all equal. Plan a UI once client confirms weighted cumulative is desired.
3. **Behind-schedule flagging** — requires target dates per layer per segment. Defer until Admin → Schedule sub-module exists.
4. **Strip performance** — 270 buckets × 7 layers = 1,890 cells per side. Use CSS Grid with aria-labels; avoid DOM-per-cell for > 5,000 cells (zoom out to 500 m buckets or switch to canvas).
5. **Strip export** — use server-side headless Chromium (Odoo has `wkhtmltopdf` built-in, but canvas rendering may be cleaner via a dedicated endpoint).
6. **Consolidated Verification Queue SQL** — union of three source tables with computed `site_id` via site / camp→site / plant→site joins. Index on `(state, site_id)` recommended.
7. **Multi-site SE UI** — site selector pill bar at the top of all screens; persists in user preferences.
8. **Layer plan per segment (Phase 2)** — model: `mcway.site.layer_plan` with `(site, segment, layer, applicable, planned_start_date, planned_end_date)`. Drives N/A cells and schedule flags.
9. **Time-travel query** — ensure the `progress.bucket` table has `last_updated` date and a history log; time-travel queries `last_updated ≤ cursor_date` to show as-of state.
10. **Photo storage** — photos on progress entries feed into reports. Keep references in `ir.attachment` linked via `mail.thread`.
11. **LHS vs RHS vs Both — join semantics** — document clearly: "Both" status is AND (LHS OR RHS must be 100% to count) unless the site configures OR semantics. Default to AND for safety.
12. **Notification on correction request** — in-app notification to the submitter (Supervisor / Camp / Plant In-charge); email optional for demo.
13. **Bulk entry grid undo** — support "undo last bulk apply" for 30 seconds after save. Prevents destructive mass edits.
14. **Layer colour palette** — use colour-blind-safe ramp for status (ColorBrewer's RdYlGn works, but swap saturated red for lighter shade to avoid panic).
15. **Hover tooltip performance** — debounce hover to 200 ms to avoid flicker on fast mouse movement.
16. **Mobile view** — SE is primarily desktop, but a read-only mobile view of the strip for quick field check-ins is a phase-2 nice-to-have.
