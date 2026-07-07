# DS-04 — Research Tab

**Status:** Draft
**Last updated:** 2026-07-07
**Relates to:** DS-07 (Radar and Portfolio Views), DS-03 (Comparative Classification UX), DS-08 (Queue and Workflow), DS-12 (Topics of Interest), TS-02 (Technology Profiles), TS-05 (Evidence Chain), TS-07 (Signal Alerts), TS-08 (Roles and Permissions), ADR-006 (Fidelity and Availability), ADR-012 (Pointer Pattern), ADR-016 (Market Reports)

---

## 1. Purpose

Research is the knowledge catalog: the authoritative store of what the platform knows about technologies. It hosts the full interactive radar, a searchable catalog of knowledge objects, and the Technology Profile workspace.

This is where durable knowledge objects live — Technology Profiles, Market Reports, Findings, Observations, and Signal Alerts — and where the tech radar sits in its primary, fully interactive form.

---

## 2. Tab layout

Two sections stacked vertically:

- **Upper section** (~45% of page height): full interactive radar (DS-07) with view switcher
- **Lower section** (~55%): knowledge catalog with search, filters, and entity list

The split is adjustable via a drag handle. Collapsing the radar gives a full-page catalog view. Collapsing the catalog gives a full-page radar view.

---

## 3. Full interactive radar (upper section)

This section embeds the full radar component from DS-07 directly. Refer to DS-07 for the complete interaction spec.

Key points in the Research tab context:

- **View switcher:** Radar | Matrix | Timeline | Neighborhood Map (see DS-07 §2)
- **Blip interaction:** clicking a blip opens the Technology Profile detail panel within the Research tab (§5 below) — it does not navigate to a separate page
- **Topics-of-interest highlights (DS-12):** blips matching the user's saved topics of interest receive a subtle highlight ring to surface relevant profiles at a glance

---

## 4. Knowledge catalog (lower section)

### 4.1 Search and filter bar

- Full-text search across all catalog entity types
- Entity type filter (multi-select): Technology Profiles / Market Reports / Findings / Observations / Signal Alerts
- Neighborhood filter (multi-select)
- Domain Area filter (multi-select)
- State filter (varies by entity type; default = published / active)
- Date range filter
- **"My topics" shortcut:** one-click filter to the user's topics of interest (DS-12)

### 4.2 Result list

Mixed-type results are shown in a unified list. Each item displays:

- Entity type badge (color-coded by type)
- Title / name
- Neighborhood and Domain Area tags
- State badge
- Last updated date
- Brief summary (1–2 lines)

Clicking any item opens its detail panel, which slides in from the right. If a detail panel is already open, it is replaced.

### 4.3 Technology Profile entry

Displays: name, `current_state` badge, Neighborhood, Domain Area, Fidelity level (0–5), Availability composite (0–5), signal count, last signal date, evidence completeness indicator.

"View full profile" opens the Technology Profile detail panel (§5).

### 4.4 Market Report entry

Displays: title, topic scope (Neighborhood or Domain Area), version number, state badge, prediction horizon date, prediction summary (N total / N pending / N confirmed / N refuted).

"View report" opens the Market Report detail panel (§6).

### 4.5 Finding entry

Displays: title, linked Technology Profile (if any), linked Prototype (if any), outcome summary, date. Read-only in the list. Click to open detail panel.

### 4.6 Observation entry

Displays: observation text (truncated to 2 lines), linked entity, analyst author, date.

Analyst+ can add a new Observation directly from the catalog list.

### 4.7 Signal Alerts entry

Displays: watchlist name, trigger type, triggered date, linked object.

Links through to the full alert record in DS-08 / TS-07.

---

## 5. Technology Profile detail panel

When a Technology Profile is opened — from the catalog, from a radar blip, or from any linked reference elsewhere in the platform — a full-width panel slides over the catalog section.

### 5.1 Header

Profile name, `current_state` badge, evidence completeness badge, Neighborhood, Domain Area, created date, last updated date, created by (or "System-generated" for auto-generated profiles).

### 5.2 Fidelity and Availability

Embedded scores per ADR-006.

**Fidelity:**
- Level (0–5)
- Confidence value (numeric)
- Brief contextual description of what the current Fidelity level means for this specific profile

**Availability:**
- Composite score (minimum of `market_availability` and `firm_access`)
- Four sub-dimension breakdown: `market_availability`, `firm_access`, `deployment_readiness`, `procurement_readiness` (all 0–5 integer scale, nullable)

Role visibility: analyst+ can see confidence values. Viewers see level labels only.

### 5.3 Classification panel

Embeds the DS-03 Comparative Classification UX scoped to this profile's `neighborhood`, `domain_area`, and `tech_type` axes. Full interaction is available for analyst+.

### 5.4 Evidence chain

A collapsible panel showing the evidence graph for this profile: signals → corroboration groups → clusters → profile. Uses the evidence chain traversal defined in TS-05.

- Evidence completeness indicator shown at the top of the panel
- Each node in the chain links to the individual signal, corroboration group, or cluster record
- Analyst+ sees the full traversable chain
- Viewers see a summary count only (e.g., "32 signals across 7 clusters")

### 5.5 Observations and Findings

Chronological list of Observations and Findings linked to this profile.

Analyst+ can add a new Observation inline: text field with a submit button, no separate navigation required.

### 5.6 Market Reports linked to this profile

Read-only list of Market Reports that cover the same Neighborhood or Domain Area as this profile. Each entry links to the Market Report detail panel (§6).

### 5.7 Lifecycle and governance

- State machine visualization: current state highlighted, available transitions shown with labels indicating who can execute each transition
- Reassessment tracking: when was Fidelity / Availability last reassessed? Is reassessment overdue?

This section is visible to analyst+ only.

### 5.8 Actions

| Action | Minimum role |
|---|---|
| Add observation | analyst |
| Request reclassification → DS-08 queue | analyst |
| Advance state (with confirmation dialog) | senior_analyst |
| Add to watchlist | viewer |
| View in radar (scrolls to blip above) | viewer |

---

## 6. Market Report detail panel

When a Market Report is opened from the catalog:

### 6.1 Header

Title, version (e.g., v3), state badge, topic scope (Neighborhood or Domain Area name), published date (if published), author.

### 6.2 Report body

Full text of the report. Sensitive body content may be masked per the pointer pattern (ADR-012, ADR-016). Viewer role sees only published versions.

### 6.3 Predictions panel

A structured list of predictions associated with this report. Each prediction shows:

- Claim text
- Prediction date
- Resolution date (if set)
- Resolution status badge: **Pending** / **Confirmed** / **Refuted** / **Partially Confirmed**
- Resolution evidence: linked signals or Findings that resolved this prediction

Analyst+ can mark a prediction's resolution status when evidence arrives. Senior_analyst+ approves the resolution.

### 6.4 Version history

Collapsible section listing all versions of this report in the superseded chain. Each version entry links to a read-only snapshot. The current version is highlighted.

### 6.5 Subscription

"Watch this report" button adds the report to the user's watchlist. The user receives an alert when a new version is published.

### 6.6 Actions

| Action | Minimum role | Condition |
|---|---|---|
| Watch | viewer | — |
| Add observation | analyst | — |
| Submit for review | analyst | report is in `draft` state |
| Publish | senior_analyst | report is in `approved` state |
| Create new version (creates draft, supersedes current) | senior_analyst | report is in `published` state |

---

## 7. Roles summary (TS-08)

| Role | Technology Profiles | Market Reports |
|---|---|---|
| viewer | Read published / active only | Read published only |
| analyst | Read all states; add observations; request profile creation | Read all states; add observations; submit draft for review |
| senior_analyst | Create profiles; publish market reports; approve observations | Publish; approve resolutions; create new versions |
| architect | Govern profile lifecycle; approve intentional profile creation | — |

---

## 8. Open decisions

> **DECISION NEEDED —** Should the Research catalog include Corroboration Groups as a searchable entity type, or only the higher-level objects (Profiles, Reports, Findings)? *Recommendation: include as a filterable type — Corroboration Groups are meaningful knowledge objects that analysts may want to find directly, especially when building an evidence chain.*

> **Resolved:** Market Reports are writable from the Research tab for authorized roles. `senior_analyst+` may edit draft/under_review reports; `architect+` may publish or withdraw. See DS-04 §6.6 for the action table.
