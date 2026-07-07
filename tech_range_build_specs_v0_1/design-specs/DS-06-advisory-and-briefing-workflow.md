# DS-06 Advisory and Briefing Workflow

**Version:** 0.1
**Status:** Draft
**Related:** ADR-013, TS-06, TS-08

---

## 1. Purpose

This spec defines the Advise tab and its end-to-end workflow: how evidence-backed Advisories and TL/DR briefs are created, reviewed, approved, and published. It also defines how the tab surfaces Tech Evaluation outcomes and prototype KPI progress.

The Advise tab is the platform's primary output layer. It is where analytical work becomes a formal, attributable recommendation or brief — consumed by leadership, stakeholders, and anyone who needs signal without needing to trace through the underlying intelligence.

---

## 2. Tab Layout

Three-panel layout:

- **Top strip** — Prototype KPI progress bar. Always visible across all three panels.
- **Left 30%** — List of Advisories and TL/DR briefs. Filterable.
- **Right 70%** — Detail view for the selected Advisory or TL/DR brief.

---

## 3. Prototype KPI Progress Strip

A persistent strip pinned to the top of the Advise tab. Visible to all roles. Read-only.

Displays:

- **Prototypes underway** — count of active Prototype records.
- **Prototypes completed this quarter** — count of Prototype records with a completed outcome in the current quarter.
- **KPI target vs. actual** — visual progress bar or gauge showing progress toward the quarterly prototype KPI target.
- **Link** — "View all prototypes →" navigates to the Prototype tab.

Data is drawn from Prototype records. The strip does not require a specific Advisory to be selected; it reflects platform-wide prototype status at all times.

---

## 4. Advisory and TL/DR List (Left Panel)

### 4.1 Filter Controls

- **Type** — Advisory / TL/DR / All
- **State** — draft / under_review / approved / published / withdrawn / All
- **Neighborhood** — multi-select
- **Domain Area** — multi-select
- **Author** — text search or dropdown
- **Date range** — published date or last-updated date

### 4.2 List Item Display

Each item shows:

- Title
- Type badge — `Advisory` or `TL/DR`
- State badge — `draft`, `under_review`, `approved`, `published`, or `withdrawn`
- Author
- Published date (if published) or last-updated date
- One-line summary

### 4.3 Role-Conditional Visibility

- **viewer** — sees only `published` items.
- **analyst and above** — sees all items in states accessible to their role, including drafts they own or are assigned to review.

---

## 5. Advisory Detail View

### 5.1 Header

Displays: title, type (`Advisory`), state badge, author, approval authority (standard or strategic), published date (if published), and withdrawal reason (if withdrawn).

### 5.2 Sections (per ADR-013 Structure)

Each section renders as a collapsible panel. Sections:

1. **Executive Summary**
2. **Context**
3. **Analytical Chain**
4. **Evidence Chain**
5. **Recommendation**
6. **Limitations and Caveats**

**Evidence chain panel** shows the linked signals, corroboration groups, and findings that support this Advisory. A completeness indicator is displayed:

> Evidence chain: 5 signals, 2 corroboration groups, 0 unresolved contradictions — Complete.

Evidence chain completeness is determined by the six-criteria check defined in TS-05 §4.4. All six criteria must pass before the Submit for Review action is enabled. The indicator shows the specific gap when criteria are not met: e.g., "Evidence chain incomplete — 2 signals, 1 unresolved contradiction."

**Analytical chain panel** shows the reasoning steps linking evidence to the recommendation. Editable by analyst and above when the Advisory is in `draft` or `under_review` state.

### 5.3 Tech Evaluation Outcomes Panel

Positioned below the main ADR-013 sections. Read-only for all roles.

Shows linked Tech Evaluations and their outcomes. Each evaluation entry displays:

- Technology name
- Evaluation result — `met`, `partially met`, or `not met`
- Link to the full evaluation record

This panel is present on all Advisories where Tech Evaluations have been linked. It is empty (hidden or collapsed) when no evaluations are linked.

### 5.4 AI Assistance Indicators

Sections drafted or pre-populated with AI assistance display a subtle badge:

> AI-assisted — analyst reviewed

Per ADR-009: AI may draft sections, but cannot publish. All AI-drafted content requires explicit analyst review and confirmation before the Advisory can be submitted for review. The badge distinguishes AI-drafted content that has been reviewed from content that has not yet been confirmed by an analyst.

---

## 6. TL/DR Brief View

Simpler layout than the full Advisory detail. Displays:

- **Title**
- **Summary paragraph** — the core message in plain language
- **Key points** — bullet list, typically 3–5 items
- **Source count** — number of signals or findings cited
- **"View full Advisory →" link** — present only if a parent Advisory exists; omitted for standalone TL/DRs

TL/DRs generated on-demand from the Surface tab are archived here and appear in the left panel list alongside Advisories. They are filterable by topic, date, and source.

---

## 7. Advisory Workflow Actions

| State | Action | Who | Notes |
|---|---|---|---|
| `draft` | Submit for review | analyst+ | Evidence chain completeness check runs before submission; action is blocked if the chain is incomplete |
| `under_review` | Approve (standard) | senior_analyst+ | Advisory moves to `approved` |
| `under_review` | Approve (strategic) | architect+ | Required for Advisories flagged as strategic scope |
| `under_review` | Return to draft | senior_analyst+ | Requires a feedback note; note is visible to the author |
| `approved` | Publish | senior_analyst+ | Makes Advisory visible to viewer role; triggers watchlist alerts and email digest if configured |
| `published` | Withdraw | architect+ | Rare; requires a stated reason; Advisory remains in archive with `withdrawn` state |

### 7.1 Publish Side Effects

When an Advisory is published:

- Watchlist subscribers receive alerts for matched Neighborhoods, Domain Areas, or technologies.
- Email digest is triggered if the platform is configured for digest delivery.
- The Advisory becomes visible to viewer-role users immediately.

### 7.2 Withdrawal

Withdrawal does not delete the Advisory. The record remains in the archive with state `withdrawn` and the stated reason attached. This preserves the audit trail.

---

## 8. Create New Advisory

The **New Advisory** button is visible to senior_analyst and above.

Clicking it opens a side panel with fields:

- **Title**
- **Type** — Advisory or TL/DR
- **Scope** — standard or strategic (determines approval authority per ADR-013)
- **Linked Neighborhood** — one or more
- **Linked Domain Area** — one or more
- **Initial executive summary** — free text

On creation, the Advisory starts in `draft` state. If evidence (signals, corroboration groups, findings) is linked at creation time, AI may pre-populate the analytical sections (Tier 3 assistance per ADR-009). (Tier 3: AI may produce a draft section; a human must review before submission.) Pre-populated sections are marked as AI-assisted and require analyst review before submission.

---

## 9. Decisions Needed

> **DECISION NEEDED —** Should the Advise tab show Tech Evaluations as a separate list section alongside Advisories, or only when linked from within an Advisory? *Recommendation: show as a separate filterable list section — Tech Evaluations have independent value as outcome records even when not part of a published Advisory.*

> **DECISION NEEDED —** When an Advisory is superseded (a new version published), should the old version remain visible in the list with a "Superseded" badge, or be hidden by default? *Recommendation: remain visible with badge and a "Superseded by →" link; preserves audit trail and allows comparison.*
