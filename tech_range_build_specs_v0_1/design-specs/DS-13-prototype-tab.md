# DS-13 — Prototype Tab

**Version:** 0.1
**Status:** Draft
**Related specs:** DS-01 (Shell and Nav), DS-05 (Solution Candidate Workflow), DS-06 (Advisory and Briefing Workflow), TS-06 (State Machines), TS-08 (Roles and Permissions)

---

## 1. Purpose

The Prototype tab serves two functions within the platform:

1. **Request and kickoff** — any authorized user can submit a new prototype request directly from this tab. Requests that arrive via the Solution Candidate handoff (DS-05 §7) also surface here for confirmation and scoping before work begins.

2. **Prototype inventory** — a browsable, filterable list of all prototypes: in scope, in progress, complete, and abandoned. This is the program's single view of what is being built, what has been finished, and what was set aside.

The Prototype tab is the operational boundary between intelligence work (Surface → Research → Solution) and execution. Advisories on the Advise tab draw on prototype outcomes; the KPI progress strip in DS-06 §3 is sourced from the same prototype records managed here.

---

## 2. Tab Layout

The Prototype tab uses a two-panel layout:

```
┌─────────────────────────────────────────────────────────────────┐
│  [ + New Request ]          Filter / Sort controls              │
├─────────────────┬───────────────────────────────────────────────┤
│                 │                                               │
│  Prototype      │  Detail panel                                 │
│  list           │  (selected prototype or new-request form)     │
│  (35%)          │  (65%)                                        │
│                 │                                               │
└─────────────────┴───────────────────────────────────────────────┘
```

When no prototype is selected and no request is in progress, the right panel displays a summary: count of prototypes by state, plus any notifications awaiting the user's attention (e.g. a newly handed-off prototype awaiting their confirmation).

---

## 3. Prototype List (Left Panel)

### 3.1 List card

Each prototype is represented by a card showing:

| Field | Notes |
|---|---|
| Title | Prototype name |
| State badge | `scoped` / `in_progress` / `complete` / `abandoned` |
| Linked Solution Candidate | Name and link, if originated from a handoff; otherwise "Direct request" |
| Assigned team | Analyst name(s) or avatar(s) |
| Target completion | Date (or "No date set") |
| Last updated | Relative (e.g. "2 days ago") |
| Outcome indicator | For `complete` prototypes: brief outcome tag (e.g. "Proceeded to Advisory", "No further action") |

Cards are sorted by last updated descending by default.

### 3.2 Filters and sort

Controls displayed above the list:

- **State** — multiselect: `scoped`, `in_progress`, `complete`, `abandoned`
- **Origin** — `handoff from Solution` / `direct request` / `all`
- **Assigned to** — filter to a specific analyst (or "assigned to me")
- **Date range** — filter by created date or target completion date
- **Sort** — last updated (default), created date, target completion date

### 3.3 Role-conditional visibility

| Role | Visible states |
|---|---|
| `viewer` | `in_progress`, `complete` |
| `analyst+` | All states including `scoped` and `abandoned` |

---

## 4. Prototype Detail Panel (Right Panel)

Selecting a card opens its full detail in the right panel.

### 4.1 Header

- Title (editable by `analyst+` while in `scoped`)
- State badge with inline transition controls (role-gated; see §6)
- **Linked Solution Candidate** — if applicable, name rendered as a link navigating to DS-05 detail
- Assigned team
- Created date / Target completion date (editable by `analyst+`)

### 4.2 Hypothesis

The operational claim being tested by this prototype. Pre-populated from the Solution Candidate hypothesis if created via handoff; otherwise entered during request creation.

- Editable by `analyst+` while in `scoped` or `in_progress`
- Previous versions preserved; "View history" control available
- Displayed prominently at the top of the detail panel

### 4.3 Scope

A structured description of what the prototype will and will not do.

| Field | Type | Notes |
|---|---|---|
| Scope description | Rich text | What the prototype covers |
| Out of scope | Rich text | Explicit exclusions |
| Success criteria | Structured list | What "done" looks like; each criterion has a description and a `met` / `not met` / `pending` status |
| Constraints | Free text | Known limitations (budget, timeline, technology, access) |

- `analyst+` can edit scope while in `scoped` or `in_progress`
- `architect+` must confirm scope before state advances to `in_progress` (see §6)

### 4.4 Plans and Attachments

Attached documents, links, or references supporting the prototype.

- Pre-populated with plans from the linked Solution Candidate (if handoff origin)
- `analyst+` can add file attachments or external links
- `architect+` can remove attachments
- Displayed as a list: name, type (file / link), attached date, attached by

### 4.5 Progress Log

An append-only activity log visible to all authorized users.

- `analyst+` can post updates: free text with optional file attachment
- Entries display author, timestamp, and content
- Milestone entries (state transitions) are auto-inserted with the acting user's name

### 4.6 Outcomes

A structured record of what the prototype produced — populated as the work progresses and finalized when the prototype reaches `complete`.

| Field | Type | Notes |
|---|---|---|
| Outcome summary | Free text | What was learned or produced |
| Evaluation criteria results | Per-criterion | `met` / `not met` for each criterion from §4.3 |
| Recommendation | Select | `Proceed to Advisory` / `Return to research` / `No further action` / `Scale` |
| Supporting evidence | UUID links | Links to Observations, Findings, or Signals that back the outcome |
| Linked Advisory | Link | If an Advisory was created from this prototype's outcomes |

- `analyst+` can populate outcomes while `in_progress` or `complete`
- `senior_analyst+` sets the final Recommendation on completion
- Outcomes feed the KPI progress strip on the Advise tab (DS-06 §3)

---

## 5. Request and Kickoff Workflow

There are two paths by which a prototype enters the system. Both result in a Prototype record in `scoped` state.

### 5.1 Path A — Handoff from Solution Candidate (DS-05 §7)

When a `senior_analyst+` executes the **Hand off to Prototype** action on an approved Solution Candidate:

1. A Prototype record is created automatically in `scoped` state, pre-populated with hypothesis, evaluation criteria, and plans from the Solution Candidate.
2. The creating analyst receives an in-platform notification.
3. The UI navigates the acting user directly to the new Prototype record on the Prototype tab.
4. A banner at the top of the detail panel reads: **"This prototype was created from a Solution Candidate handoff. Review the pre-populated scope and confirm when ready."**
5. The analyst reviews, edits if needed, then transitions state to `in_progress` when scope is confirmed (see §6).

### 5.2 Path B — Direct Request (Prototype tab)

A **+ New Request** button is visible to `analyst+` in the top bar of the Prototype tab.

Clicking it opens the right panel as a creation form:

| Field | Type | Required | Notes |
|---|---|---|---|
| Title | Text | Yes | |
| Hypothesis | Free text | Yes | What this prototype is testing |
| Linked Solution Candidate | Search/select | No | Optional — for tracking lineage without a formal handoff |
| Scope description | Rich text | No | Can be refined after creation |
| Success criteria | Structured list | No | At least one recommended; can be added post-creation |
| Target completion date | Date picker | No | |
| Initial notes | Free text | No | |

On submit:
- The prototype is created in `scoped` state
- The submitting analyst is set as the primary assigned analyst
- The record appears at the top of the list, filtered to `scoped`
- A `profile_creation_request` queue entry is **not** created — prototype creation does not require governance approval, unlike Technology Profile creation

> **DECISION NEEDED —** Should direct prototype requests require `senior_analyst+` approval before advancing to `in_progress`, or can the submitting `analyst` advance it themselves? *Recommendation: require `senior_analyst+` to confirm scope before `in_progress`, mirroring the scoping confirmation in the handoff path. This keeps accountability consistent.*

---

## 6. State Transitions and Actions

Prototype states follow TS-06 §4.6. Valid transitions:

| From | To | Action | Who | Notes |
|---|---|---|---|---|
| `scoped` | `in_progress` | Confirm scope and begin | `senior_analyst+` | Scope must have at least one success criterion; hypothesis must be present |
| `scoped` | `abandoned` | Abandon request | `architect+` | Reason required |
| `in_progress` | `complete` | Mark complete | `senior_analyst+` | Outcome summary and Recommendation must be set; all success criteria must have a result |
| `in_progress` | `abandoned` | Abandon | `architect+` | Reason required; the Progress Log records the final entry |
| `complete` | — | (terminal) | — | Read-only after completion |
| `abandoned` | — | (terminal) | — | Read-only after abandonment |

All transitions are logged to `state_transitions` (TS-06) with actor, timestamp, from-state, to-state, and any required reason.

---

## 7. Role Summary

| Capability | viewer | analyst | senior_analyst | architect | admin |
|---|---|---|---|---|---|
| View `in_progress` and `complete` | Yes | Yes | Yes | Yes | Yes |
| View `scoped` and `abandoned` | No | Yes | Yes | Yes | Yes |
| Submit direct prototype request | No | Yes | Yes | Yes | No |
| Edit hypothesis, scope, criteria, plans | No | Yes | Yes | Yes | No |
| Post progress log entries | No | Yes | Yes | Yes | No |
| Confirm scope / begin (`scoped → in_progress`) | No | No | Yes | Yes | No |
| Mark complete (`in_progress → complete`) | No | No | Yes | Yes | No |
| Set final Recommendation on completion | No | No | Yes | Yes | No |
| Abandon (`scoped` or `in_progress → abandoned`) | No | No | No | Yes | No |
| Edit any field (override) | No | No | No | No | Yes |

---

## 8. Integration Points

| Upstream / downstream | Description |
|---|---|
| **DS-05 Solution Candidate** | Handoff (§5.1) creates Prototype records; bidirectional link between Solution Candidate and Prototype is displayed in both records |
| **DS-06 Advise tab** | KPI progress strip (DS-06 §3) counts prototypes by state and outcome recommendation; links to this tab for full inventory |
| **DS-08 Governance Queues** | No queue entry for prototype creation; `abandoned` records may surface in the escalation queue if a reason flags a systemic issue |
| **TS-06 §4.6** | Canonical state machine for prototypes |
| **TS-08** | Role permissions authority |

---

## 9. Open Decisions

| # | Question | Recommendation |
|---|---|---|
| 9.1 | Should `analyst` be able to advance `scoped → in_progress` for direct requests, or always require `senior_analyst+`? | Require `senior_analyst+` for consistency with handoff path |
| 9.2 | Should prototype records have a dedicated "team" assignment (multiple analysts) or just a single owner? | Support multiple assignees — prototypes are typically multi-person efforts |
| 9.3 | Should prototype records appear in the Surface node graph as a node type, or remain a workflow-only construct? | Phase 2 consideration — not needed at v1.0 |
