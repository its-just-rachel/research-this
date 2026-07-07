# DS-07 — Radar and Portfolio Views

**Status:** Draft  
**Version:** 0.1  
**Related:** ADR-010, DS-01, DS-03, DS-12, TS-09

---

## 1. Purpose

This spec covers the full interactive tech radar and portfolio views on the Research tab. These are multiple views of the same Technology Profile data, designed to serve different analyst and senior_analyst or above needs — from high-level portfolio orientation to detailed trend and maturity analysis.

The radar is a derived, read-only view. It reflects active Technology Profiles and cannot be used to edit or reclassify them. All editing happens on the profile itself (ADR-010). The radar serves three distinct cognitive purposes:

- **Portfolio overview** — what technologies are we tracking, and at what maturity?
- **Trend detection** — what's moving, what's stagnating?
- **Decision support** — what's ready for action vs. still emerging?

The animated radar on the Overview page (DS-01) is a teaser only. This spec covers the full interactive version on the Research tab.

---

## 2. View Switcher

The Research tab header includes a persistent view switcher with four options:

```
Radar | Matrix | Timeline | Neighborhood Map
```

- Default view on first load: **Radar**
- The user's last-selected view is persisted in their profile (DS-12)
- Switching views preserves active filter state where applicable

---

## 3. Radar View (Full Interactive)

### 3.1 Layout

- Full-width visualization occupying approximately 70% of page height
- Four quadrants, each labeled by its Neighborhood grouping (six Neighborhoods mapped into four quadrants per ADR-010)
- Filter and control bar positioned below the radar
- Detail panel slides in from the right when a blip is selected; radar resizes to accommodate

### 3.2 Blip Properties

Each Technology Profile (and optionally each Cluster) is rendered as a blip:

| Property | Source | Details |
|---|---|---|
| X position | `availability_composite` | 0–5 scale mapped to horizontal axis |
| Y position | `fidelity_level` | 0–5 scale mapped to vertical axis |
| Size | Signal count | Proportional; min 8px diameter, max 24px diameter |
| Color | Neighborhood | Six-color accessible palette; defined in design tokens |
| Shape | Record type | Circle = Technology Profile; Diamond = Cluster (emerging tech not yet profiled) |
| Burst indicator | `is_bursting` on linked cluster | Pulsing ring rendered on qualifying blips |

The axes are labeled:
- **X-axis (horizontal):** Availability — how accessible is this technology?
- **Y-axis (vertical):** Fidelity — how well understood is this technology?

### 3.3 Hover Interaction

Hovering over any blip surfaces a mini card containing:

- Technology name
- Neighborhood
- Domain Area
- Fidelity level
- Availability composite
- Signal count
- Current state

The mini card includes a "View profile →" link that navigates to the full Technology Profile in the Research catalog.

### 3.4 Click / Select

Clicking a blip:

- Opens the blip detail panel (§3.6) sliding in from the right
- Highlights the selected blip
- Dims all other blips to reduce visual noise

Clicking elsewhere on the radar canvas deselects and closes the detail panel.

### 3.5 Filter and Control Bar

Displayed below the radar. Controls apply in real time and update blip rendering immediately.

**Filters:**

- **Neighborhood** — multi-select; all selected by default
- **Domain Area** — multi-select; all selected by default
- **Fidelity range** — dual-handle slider, 0–5
- **Availability range** — dual-handle slider, 0–5
- **State** — multi-select; default = `active` only

**Toggles:**

- Show Clusters (diamonds) alongside Profiles — off by default
- Show burst indicators — on by default

**Search:**

- Jump-to field: type a Technology Profile name to pan and highlight the matching blip

### 3.6 Blip Detail Panel

Slides in from the right when a blip is selected. Content:

**Header**
- Technology Profile name
- State badge (e.g., Active, Archived, Emerging)
- Neighborhood
- Domain Area

**Scores**
- Fidelity level with sub-dimension breakdown (per ADR-006)
- Availability composite with sub-dimension breakdown (per ADR-006)

**Evidence summary**
- Total signal count
- Corroboration group count
- Latest signal date

**Classification panel**
- Embedded DS-03 classification panel
- Read-only for Viewer role; full interaction for Analyst and above

**Actions**
- "View full profile →" — navigates to the Technology Profile detail page in the Research catalog
- "Add to watchlist" — available to all authenticated roles
- "Request reclassification" — Analyst and above only

---

## 4. Matrix View

Tabular alternative to the radar for analysts who prefer to sort and compare.

**Columns:** Name, Neighborhood, Domain Area, Fidelity, Availability, Signal Count, State, Last Updated

- All columns are sortable
- Filtering uses the same filter bar as the radar (§3.5)
- Each row links to the full Technology Profile

**Role visibility:**
- Viewer: active profiles only
- Analyst and above: all states

---

## 5. Timeline View

A horizontal timeline showing when Technology Profiles entered each state over time.

- **X-axis:** Time
- **Y-axis:** Grouped by Neighborhood
- **Events plotted:** profile created, state transitions, major reclassifications, burst events

Intended use cases: signal archaeology, trend analysis, understanding the pace of maturation across Neighborhoods. Read-only. No editing actions available from this view.

Implementation reference: KronoGraph-style layout.

---

## 6. Neighborhood Map View

A simplified treemap or bubble chart showing Neighborhoods as containers with Technology Profiles as nested items.

- Size of each container reflects total signal volume for that Neighborhood
- Intended for portfolio balance assessment at a glance
- Read-only

> **DECISION NEEDED —** Should the Neighborhood Map view be included in MVP or deferred? *Recommendation: defer to Phase 2 — the radar and matrix views cover the core need; Neighborhood Map is a nice-to-have for portfolio balance discussions.*

---

## 7. Export

| Export | Format | Roles |
|---|---|---|
| Current radar screenshot | PNG | All roles |
| Profile list (matrix view) | CSV | Analyst and above |

Raw confidence scores and evidence chain data are not exportable to CSV from the UI. Access to that data requires the API (TS-09).

---

## 8. Open Decisions

> **DECISION NEEDED —** Should the radar show Technology Profiles in all states (with visual differentiation), or only active profiles by default with other states as an opt-in filter? *Recommendation: default to active only (per ADR-010); other states available via filter. Prevents clutter and reflects the radar's purpose as a current-portfolio view.*

> **DECISION NEEDED —** Should the Neighborhood Map view be included in MVP or deferred? *Recommendation: defer to Phase 2 — the radar and matrix views cover the core need; Neighborhood Map is a nice-to-have for portfolio balance discussions.*
