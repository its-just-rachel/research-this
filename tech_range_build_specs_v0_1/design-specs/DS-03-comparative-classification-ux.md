# DS-03: Comparative Classification UX

**Component:** Comparative Classification Panel
**Related systems:** TS-04 (Comparative Classification Engine), TS-08 (Role-Based Access Control), ADR-005 (Immutable Records), ADR-009 (AI Tiers)
**Type:** Reusable embedded component — not a standalone page
**Surfaces:** Signal triage (Surface tab), tech profile detail (Research tab), solution candidate evaluation (Solution tab)
**Status:** Draft

---

## 1. Purpose

Defines how Comparative Classification Engine (CCE) results are presented to users and how analysts interact with them. The CCE produces `match_set` records for signals, clusters, tech profiles, and solution candidates. Each `match_set` classifies an object along up to four axes and exposes primary/secondary matches, confidence scores, and boundary case flags.

This spec covers the visual structure, role-differentiated display, and analyst action surface for the classification panel wherever it appears. It does not define the CCE engine itself (see TS-04) or the governance review queue (see DS-08).

---

## 2. Classification display

### 2.1 Axis panel layout

Each classified object has up to four axes: **Neighborhood**, **Domain Area**, **Technology Type**, and **Candidate Type**. Each axis renders as a self-contained panel section within the classification component.

**Per-axis panel structure:**

- **Axis label** — displayed as a section header (e.g., "Neighborhood", "Technology Type")
- **Boundary badge** — if `is_boundary_case = true` for this axis, display an amber "Boundary" badge in the panel header, left-aligned alongside the axis label
- **Primary match** — classification name rendered with a labeled progress bar on a 0.0–1.0 scale
  - Bar color by confidence tier:
    - Green: `primary_score >= 0.70`
    - Amber: `0.40 <= primary_score < 0.70`
    - Red: `primary_score < 0.40`
- **diff_strength indicator** — small inline label below or beside the primary match bar, showing the gap to the next-best option. Two phrasings:
  - `diff_strength >= 0.15`: "Gap: 0.22 — clear"
  - `diff_strength < 0.15`: "Gap: 0.08 — ambiguous"
- **Secondary matches** — collapsed by default; expandable via a chevron toggle. Shows the top 2–3 alternative classifications, each with its score and a brief rationale snippet sourced from the `match_set` JSONB payload. Secondary matches are only visible to analyst+ roles (see §2.2).

**Axis panel amber left border** is applied when `is_boundary_case = true` on that axis, reinforcing the badge and tooltip treatment described in §2.3.

### 2.2 Confidence visualization by role

Raw decimal scores must not be the primary display for viewer-role users. Role-differentiated rendering:

| Element | viewer | analyst / senior_analyst / architect / admin |
|---|---|---|
| Primary classification label | Visible | Visible |
| Confidence tier label | Visible ("High confidence", "Moderate confidence", "Low confidence") | Visible |
| Numeric `primary_score` | Hidden; visible on hover | Always visible |
| `diff_strength` value | Hidden | Visible |
| Secondary matches | Hidden (see Decision Needed §7) | Visible, collapsed by default |
| Evidence basis | "Based on N evidence items" only | Full detail (§4) |
| Classification history | Hidden | Visible (§5) |

**Confidence tier label mapping:**

- `primary_score >= 0.70` → "High confidence"
- `0.40 <= primary_score < 0.70` → "Moderate confidence"
- `primary_score < 0.40` → "Low confidence"

**Example viewer display:** "Agentic / Vertical AI — High confidence"

### 2.3 Boundary case visual treatment

When `is_boundary_case = true` on any axis:

- The axis panel receives an **amber left border** (4px)
- An amber **"Boundary" badge** appears in the panel header
- A **tooltip** on the badge reads: "This classification is ambiguous — the top two options scored within 0.15 of each other. Analyst review recommended."
- The object appears in the DS-08 Governance Queue under the "Boundary Cases" section until the axis is approved or overridden by a senior_analyst or above

Boundary cases do not block the object from appearing in normal workflows. They surface as a visual call-to-action for analysts without interrupting viewer access.

---

## 3. Analyst actions

All analyst actions are gated by role as specified. The action surface (buttons, secondary match expansion, dispute panel) is not rendered for viewer-role users.

### 3.1 Approve

- **Control:** "Approve" button per axis panel; "Approve all axes" button at the component level for bulk approval
- **Available to:** senior_analyst, architect, admin
- **Effect:**
  - Sets `review_status = 'approved'` on the `match_set` record
  - Removes the object from the DS-08 boundary case queue if applicable
  - Updates the axis panel: amber border and badge clear; panel header shows a green checkmark and "Approved" status
- **Confirmation:** None required for individual axis approval. "Approve all axes" requires a single confirmation step ("Approve classifications on all 4 axes for this object?") before executing.

### 3.2 Override (swap primary/secondary)

Analysts can promote a secondary match to the primary classification for an axis.

**Interaction flow:**

1. Analyst expands the secondary matches section on an axis
2. Clicking a secondary match option surfaces a "Set as primary" button inline with that option
3. Clicking "Set as primary" opens an inline rationale field (mandatory; cannot submit without text)
4. On submit, the swap is applied according to role:
   - **analyst:** creates a new `match_set` record with the swap applied; the new record's `review_status` is set to `analyst_reviewed` and is routed to DS-08 for senior_analyst confirmation. The original record is preserved per ADR-005 (immutable-but-supersedable).
   - **senior_analyst, architect, admin:** swap takes effect immediately; new `match_set` record is created with `review_status = 'approved'`; original record is preserved

**Immutability:** Override always creates a new `match_set` record. The prior record's `superseded_at` and `superseded_by` fields are updated to reflect the chain. No `match_set` record is mutated or deleted.

**AI calibration:** The swap is surfaced to the CCE as a calibration signal per TS-04 §8. The rationale text is included in the signal payload.

### 3.3 Dispute

- **Control:** "Dispute" button on each axis panel
- **Available to:** analyst, senior_analyst, architect, admin
- **Interaction:** Opens a side panel with:
  - A free-text description field: "Describe the issue" (required)
  - An optional field: "Suggested classification" (free text or pick from axis taxonomy)
- **Effect:**
  - Sets `review_status = 'disputed'` on the `match_set` record
  - Routes the object to DS-08 review queue under "Disputed Classifications"
  - Notifies senior_analysts via DS-08 queue notification (notification mechanism defined in DS-08)
- **Visibility:** Active disputes are visible to all analyst+ roles on the object's classification panel as an inline callout beneath the relevant axis.

### 3.4 Flag as needs internal answer

- **Control:** "Flag as needs answer" option, accessible from the axis action menu (ellipsis or secondary action surface alongside "Dispute")
- **Available to:** analyst, senior_analyst, architect, admin
- **Use case:** The correct classification depends on an unresolved internal decision (e.g., whether a given technology falls within the Quantum neighborhood)
- **Effect:**
  - Sets `review_status = 'needs_internal_answer'` on the `match_set` record
  - Routes to DS-08 queue under "Pending Internal Decisions"
  - Axis panel displays a "Pending decision" badge (neutral/blue) until resolved

### 3.5 Request reclassification

- **Available to:** analyst, senior_analyst, architect, admin
- **Applicable when:** The object already has an `approved` classification and the analyst believes it should be reconsidered
- **Interaction:** Opens a lightweight modal with:
  - "Reason for reclassification request" (required)
  - "Suggested new classification" (optional; free text or taxonomy picker)
- **Effect:**
  - Creates a reclassification request record linked to the `match_set`
  - Routes to DS-08 queue; requires senior_analyst or architect to action the request
  - The existing approved classification remains in effect and displayed until the reclassification is executed

---

## 4. Evidence basis display

A collapsed "Evidence basis" section appears below the axis panels, visible to analyst+ roles only.

**Contents:**
- List of signal and cluster UUIDs the CCE used when generating this `match_set`
- Each item is a link that navigates to that object's detail view
- Displayed as: `[object type] [UUID short form] — [object label if available]`

**Viewer display:** "Based on N evidence items." (no detail, no links)

---

## 5. Classification history

An expandable "Classification history" panel at the bottom of the component, visible to analyst+ roles only.

**Contents:**
- Chronological timeline of all `match_set` versions for this object and axis
- Each entry shows: timestamp, actor (human username or "CCE"), action (initial classification, override, approval, dispute), and a summary of what changed
- The chain is reconstructed from the `superseded_at` / `superseded_by` fields on `match_set` records per ADR-005
- The current active record is marked "Current"

Viewer role does not see this panel.

---

## 6. Empty and loading states

| State | Display |
|---|---|
| No classification yet | "Pending classification" placeholder per axis. If an estimated completion time is available from the CCE job queue, display it: "Expected within ~10 minutes." |
| CCE classification in progress | Spinner per axis panel; axis label remains visible |
| All axes approved | Green "Classification confirmed" status banner at the top of the component, replacing any amber boundary indicators |
| Partial approval | Individual axes show their status; no top-level banner until all axes reach `approved` |
| CCE error / classification failed | Axis shows an error state with a "Retry" option available to senior_analyst+ |

---

## 7. Decisions needed

> **DECISION NEEDED —** Should analysts be able to add a free-text annotation to an approved classification (e.g., "Approved; note that this may shift to SDx as the product matures")? *Recommendation: yes — a notes field on the match_set review record is low-cost and high-value for audit trail quality; it does not affect the classification itself and is visible to all analyst+ roles on the history panel.*

> **DECISION NEEDED —** For the viewer role, should secondary matches be entirely hidden, or visible read-only? *Recommendation: hidden for viewer — secondary matches imply analytical uncertainty that may be confusing without context; show only the primary classification label and confidence tier.*
