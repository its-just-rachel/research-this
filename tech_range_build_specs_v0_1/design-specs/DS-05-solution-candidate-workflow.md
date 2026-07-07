# DS-05 — Solution Candidate Workflow

**Version:** 0.1
**Status:** Draft
**Related specs:** DS-03 (Comparative Classification UX), TS-06 (State Machine), TS-08 (Roles and Permissions)

---

## 1. Purpose

This spec defines the Solution tab and its workflow: how proposed engagement paths are created, evaluated, approved, and handed off to the Prototype phase.

The Solution tab is the bridge between intelligence (Research) and execution (Prototype). Technology Profiles from the Research tab are handed off here as **Solution Candidates** — structured records that connect a technology (or set of technologies) to a specific operational need or opportunity. Each Solution Candidate carries a hypothesis, evaluation criteria, attached plans, logged outcomes, and a lifecycle state that governs who can act on it and when.

---

## 2. Tab Layout

The Solution tab uses a two-column layout:

| Column | Width | Content |
|---|---|---|
| Left | 35% | Filterable list of Solution Candidate cards |
| Right | 65% | Detail panel for the selected candidate |

### Filter and sort controls

Displayed above the card list. Controls:

- **State** — filter by one or more candidate states (`draft`, `under_evaluation`, `approved`, `handed_off`, `archived`)
- **Neighborhood** — filter by the Neighborhood of the linked Technology Profile
- **Domain Area** — filter by Domain Area
- **Assigned analyst** — filter to a specific analyst
- **Date** — sort by last updated or created date (ascending / descending)

### Role-conditional visibility

- **viewer:** sees only `approved` and `handed_off` candidates
- **analyst and above:** sees all states including `draft`, `under_evaluation`, and `archived`

---

## 3. Solution Candidate Card (List View)

Each card in the left-column list displays:

- **Title** — candidate name
- **Linked Technology Profile** — name and current profile state (badge)
- **Candidate state** — badge reflecting current state in the state machine
- **Assigned analyst** — name or avatar
- **Last updated** — relative date (e.g., "3 days ago")
- **Hypothesis summary** — one-line excerpt of the hypothesis field
- **Boundary indicator** — if the linked Technology Profile has any unresolved boundary classifications, a warning indicator is shown on the card

---

## 4. Solution Candidate Detail Panel

Selecting a card opens its detail panel in the right column.

### 4.1 Header

- Title (editable by analyst+)
- State badge
- Linked Technology Profile — name, rendered as a link that navigates to the profile detail in the Research tab
- Assigned analyst(s)
- Created date
- Last updated date

### 4.2 Hypothesis

The central claim this candidate is testing: what operational outcome is expected from applying this technology.

- Editable by analyst+
- Version history is preserved — previous hypotheses are accessible via a "View history" control
- The active hypothesis is always displayed prominently at the top of the detail panel

### 4.3 Evaluation Criteria

A structured list of criteria against which the candidate will be evaluated before handoff.

Each criterion record contains:

| Field | Values |
|---|---|
| Description | Free text |
| Weight | `high` / `medium` / `low` |
| Status | `not assessed` / `in progress` / `met` / `not met` |

- **analyst+** can add and edit criteria
- **senior_analyst** marks final status on each criterion
- Criteria can be reordered; order is preserved

> **DECISION NEEDED —** Should evaluation criteria be free-form or drawn from a standardized taxonomy? *Recommendation: start with free-form; introduce optional taxonomy tags in Phase 2 once patterns emerge from analyst usage.*

### 4.4 Plans

Attached plans describe the proposed engagement approach.

- Plans can be uploaded as files or linked externally (URL)
- Displayed as a list with name, type (file / link), and attached date
- **analyst** can attach plans
- **architect** can remove plans
- No other roles can delete attachments

### 4.5 Outcomes

A log of what has been observed so far in evaluation. Outcomes feed into the Prototype and Advise phases.

- **analyst+** can log outcomes as free-text observations
- Each outcome can be linked to one or more evaluation criteria from section 4.3
- Outcomes are displayed in reverse-chronological order with author and timestamp
- Outcomes are append-only (no deletion); corrections are logged as new entries with a reference to the original

### 4.6 Linked Findings and Observations

A read-only panel listing the canonical Findings and Observations entities associated with this candidate.

- Each item shows: entity type badge, title, source Technology Profile, and date
- Each item links to its full record
- This panel is populated automatically based on associations set on the Finding or Observation records; it cannot be edited from the Solution tab

### 4.7 Classification Panel

An embedded instance of the DS-03 Comparative Classification UX component, scoped to the `candidate_type` axis for this Solution Candidate.

- Displays current classification on the candidate_type axis
- Follows DS-03 edit permissions (analyst+ can propose; senior_analyst+ can finalize)
- Boundary classifications surface the same warning indicators shown on the card in list view

---

## 5. State Transitions and Actions

Solution Candidate states follow TS-06. Valid transitions:

| From state | Action | Who | Notes |
|---|---|---|---|
| `draft` | Submit for evaluation | analyst+ | Routes to senior_analyst review queue |
| `under_evaluation` | Approve | senior_analyst+ | Candidate is ready to proceed |
| `under_evaluation` | Return to draft | senior_analyst+ | A feedback note is required |
| `approved` | Hand off to Prototype | senior_analyst+ | Creates a linked Prototype record; navigates user to Prototype tab |
| `approved` | Archive | architect+ | A reason is required |
| `handed_off` | Archive | architect+ | After prototype completes; a reason is required |

All state transitions are logged to the `state_transitions` table (TS-06) with actor, timestamp, from-state, to-state, and any required note or reason.

---

## 6. Create New Solution Candidate

A **New Candidate** button is visible to analyst+ in the top bar of the Solution tab.

Clicking it opens a slide-in panel form with the following fields:

| Field | Type | Required |
|---|---|---|
| Title | Text | Yes |
| Linked Technology Profile | Search/select | Yes |
| Hypothesis | Free text | Yes |
| Initial notes | Free text | No |

On submit:

- The candidate is created in `draft` state
- The creating analyst is set as the assigned analyst
- The candidate appears at the top of the list, filtered to show drafts

> **DECISION NEEDED —** Should a Solution Candidate be able to link to multiple Technology Profiles (e.g. a multi-technology engagement path), or is it always 1:1 with a Technology Profile? *Recommendation: allow multiple linked profiles — real solutions often combine technologies; enforce at least one required.*

---

## 7. Handoff to Prototype

When a senior_analyst+ triggers the **Hand off to Prototype** action from an `approved` candidate:

1. A Prototype record is automatically created, pre-populated with:
   - The candidate's hypothesis
   - The full list of evaluation criteria (carried over with their weights; statuses reset to `not assessed` for the Prototype phase)
   - All attached plans
2. The creating analyst is notified to review and confirm the Prototype record on the Prototype tab
3. The Solution Candidate's state transitions to `handed_off`
4. The candidate remains visible in the Solution tab but becomes **read-only** — no further edits to hypothesis, criteria, plans, or outcomes
5. The UI navigates the acting user to the newly created Prototype record on the Prototype tab

The link between the Solution Candidate and its Prototype record is bidirectional and displayed in the header of both records.

---

## 8. Role Summary

| Capability | viewer | analyst | senior_analyst | architect |
|---|---|---|---|---|
| See approved / handed_off candidates | Yes | Yes | Yes | Yes |
| See draft / under_evaluation candidates | No | Yes | Yes | Yes |
| Create new candidate | No | Yes | Yes | Yes |
| Edit hypothesis, criteria, plans | No | Yes | Yes | Yes |
| Log outcomes | No | Yes | Yes | Yes |
| Submit for evaluation | No | Yes | Yes | Yes |
| Approve / return to draft | No | No | Yes | Yes |
| Hand off to Prototype | No | No | Yes | Yes |
| Archive candidate | No | No | No | Yes |
| Remove attached plans | No | No | No | Yes |
| Govern scope and platform alignment | No | No | No | Yes |
