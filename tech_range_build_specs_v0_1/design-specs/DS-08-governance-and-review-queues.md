# DS-08 — Governance and Review Queues

## 1. Purpose

A unified governance and review interface accessible from the utility nav notification bell (quick view) and as a dedicated queue page. All items requiring human judgment, approval, or action flow through here — boundary cases, disputed classifications, reclassification requests, emerging technology review, profile creation approvals, advisory review, and needs-internal-answer items. One queue, one interface, role-gated filters.

---

## 2. Access Points

Two entry points into the queue:

1. **Notification bell** (utility nav, DS-01): shows unread item count as a badge; dropdown shows the top 5 items with quick-action buttons (Approve, Assign to me). Notification count reflects only items visible to the logged-in user's role.
2. **Full queue page**: accessible via notification bell "View all" (for `analyst+`) or `/queues` direct route listed in the utility nav for `analyst+` (see DS-01 §2.2 for utility nav definition).

> **DECISION NEEDED —** Should the queue page be a dedicated top-level route (e.g. `/queues`) or always accessed via the notification bell dropdown? *Recommendation: both — notification bell for quick triage, dedicated `/queues` route for full queue management. Analysts working the queue need a full-page experience.*

---

## 3. Queue Page Layout

| Zone | Contents |
|---|---|
| Left sidebar | Queue type filters (see §4); active filter highlighted; item counts per type |
| Main panel | Filtered item list with sort controls; default view is My Items |
| Right detail panel | Selected item detail + role-appropriate action buttons |
| Page header | Total unread count for current filter + SLA status indicator (are items aging past SLA?) |

The layout is a persistent three-panel design. Selecting an item in the main panel populates the right detail panel without a page navigation. On narrow viewports, the detail panel replaces the list (back arrow returns to list).

---

## 4. Queue Types and Filters

| Queue type | Who sees it | SLA target |
|---|---|---|
| My Items | analyst+ | varies |
| Boundary Cases | analyst+ | 3 business days |
| Disputed Classifications | senior_analyst+ | 2 business days |
| Reclassification Requests | architect+ | 5 business days |
| Emerging Technology Review | senior_analyst+ | 5 business days |
| Profile Creation Requests | architect+ | 5 business days |
| Advisory Review | senior_analyst+ | 3 business days |
| Needs Internal Answer | architect+ | [Internal decision rule] |

**My Items** is the default view on load — shows only items assigned to the logged-in user, across all queue types the user's role permits. Switching to any named queue type shows all items of that type visible to the user's role (assigned and unassigned).

**Role visibility summary:**
- `viewer`: no queue access
- `analyst`: My Items + Boundary Cases (scoped to watched neighborhoods)
- `senior_analyst`: all analyst items + Advisory Review + Emerging Technology Review + Disputed Classifications
- `architect`: all senior_analyst items + Reclassification Requests + Profile Creation Requests + Needs Internal Answer
- `admin`: all queues, all items

**Sort options** (applied within any filter view): by age oldest-first (default), by priority high-first, by SLA deadline, by object type.

---

## 5. Queue Item Card

Each card in the main panel list displays:

- **Queue type badge** — color-coded by type (e.g., amber for Boundary Case, blue for Emerging Tech)
- **Object name and type** — e.g., "Agentic RAG Pipelines — Technology Profile"
- **Brief reason queued** — one line, e.g., "diff_strength 0.11 between Layer 4 and Layer 5" or "Disputed by analyst J. Torres"
- **Assigned to** — analyst name, or "Unassigned"
- **Age** — time since created_at, e.g., "2d 4h"
- **SLA status** — color-coded pill: On track (green) / At risk (amber, ≥80% of SLA elapsed) / Overdue (red)
- **Quick action buttons** — contextual; most common: Approve, Resolve, Assign to me

Cards are sorted per the active sort control. Overdue items surface to the top of the default age sort regardless.

---

## 6. Item Detail Panel

### 6.1 Boundary Case Detail

**Canonical queue_type:** `boundary_case_review`

Triggered when a classification's `diff_strength < 0.15` — the two top matches are close enough to warrant analyst judgment.

- Displays the object (signal, cluster, or technology profile) with its full classification panel (DS-03, embedded, full interaction)
- The ambiguous axis is highlighted: primary match and next-best match shown side by side with their scores
- Analyst actions:
  - **Confirm primary** — approves the existing classification, removes from queue
  - **Swap to secondary** — overrides to next-best match, records the decision
  - **Escalate** — pushes to senior_analyst queue
  - **Add note** — attaches a reasoning note to the item without resolving
- Resolution records the decision, the deciding analyst, and timestamp; removes item from queue

### 6.2 Disputed Classification Detail

**Canonical queue_type:** `disputed_classification`

Triggered when an analyst flags a classification as incorrect.

- Shows the original classification, the dispute reason, and the disputing analyst's identity and timestamp
- Senior analyst actions:
  - **Accept dispute** — initiates reclassification (may create a Reclassification Request for architect if scope warrants)
  - **Reject dispute** — dismisses the dispute; requires a written reason logged against the item
  - **Escalate to architect** — pushes to architect queue with context preserved
- All decisions logged with actor, timestamp, and rationale.

### 6.3 Reclassification Request Detail

**Canonical queue_type:** `reclassification_request`

Triggered when someone requests a profile's classification be changed.

- Shows current classification, requested new classification, and requester's rationale
- Architect actions:
  - **Approve reclassification** — creates new `match_set`, supersedes the prior match; triggers downstream impact check (are any advisories or signals pointing at the old classification?)
  - **Reject** — dismisses the request; requires a written reason
- Downstream impact warning displayed if the reclassification would affect published advisories or active signals.

### 6.4 Emerging Technology Review Detail

**Canonical queue_type:** `profile_review`

Triggered when the system creates a Technology Profile in `candidate` state via emergent detection.

- Shows the candidate Technology Profile with all auto-populated fields
- **Evidence basis section**: the signals that triggered emergent detection, with their strength scores and source references
- Analyst+ actions:
  - **Confirm as Emerging** — advances the profile state from `candidate` to `emerging`; profile becomes visible in the technology graph
  - **Merge with existing profile** — if the candidate appears to be a duplicate of an existing profile; opens a merge target selector
  - **Archive** — removes the candidate from the queue and marks it as a false positive; reason required
- Confirmation prompts for Merge and Archive (destructive or hard to reverse).

### 6.5 Advisory Review Detail

**Canonical queue_type:** `advisory_review`

Triggered when an advisory is submitted for approval.

- Shows the submitted advisory in full — all sections, evidence chain, and an **evidence completeness indicator** (are all cited signals resolvable? are all cited profiles published?)
- Incomplete evidence chain is flagged inline with specific gaps listed
- Senior analyst / architect actions:
  - **Approve** — Transitions Advisory to `approved` state. Advisory is not yet published.
  - **Publish** — `architect+` only. Transitions `approved → published`; sets `published_at`; notifies subscribers.
  - **Return to draft** — sends back to the submitting analyst with required feedback (free-text reason field, non-optional)
  - **Escalate** — moves to architect queue with context

### 6.6 Profile Creation Request Detail

**Canonical queue_type:** `profile_creation_request`

Triggered when an intentional profile creation request is submitted and awaiting architect sign-off.

- Shows the proposed profile fields, the requesting analyst, and the stated rationale
- Architect actions:
  - **Approve creation** — advances the profile to `draft` state for the analyst to complete
  - **Reject** — dismisses the request; reason required

### 6.7 Needs Internal Answer Detail

**Canonical queue_type:** `needs_internal_answer`

Triggered when a classification question is blocked on an unresolved internal decision (e.g., taxonomy ambiguity, conflicting framework definitions).

- Shows the blocking question, the originating item, and any prior discussion thread
- Architect actions:
  - **Record decision** — documents the internal resolution; unblocks downstream items waiting on the same decision
  - **Escalate** — marks as needing cross-team input; flags for admin visibility
- Items in this queue can block multiple downstream items; the detail panel shows a count of dependent blocked items.

---

## 7. SLA and Escalation

- **At risk** (amber): flagged when ≥80% of the item's SLA window has elapsed without resolution
- **Overdue** (red): flagged when SLA has elapsed; item auto-escalates to the next role tier
  - Boundary Case overdue → escalates to senior_analyst
  - Disputed Classification overdue → escalates to architect
  - Reclassification Request / Profile Creation / Emerging Tech overdue → flagged to admin
- Escalation notifications are sent via the notification bell to the receiving role tier
- TS-10 tracks mean time to resolve per queue type as a quality metric; SLA breach rate is a monitored health signal

SLA windows use business days (weekends and configured holidays excluded).

---

## 8. Bulk Actions

Available in Boundary Case and Emerging Technology Review queues only.

- Analyst can select multiple items via checkbox and trigger **Approve all**
- Bulk approval requires confirmation dialog: *"You are approving [N] boundary cases. This cannot be undone in bulk. Confirm?"*
- Bulk approve is only available when the analyst has viewed each item in the current selection (read state tracked per item; unviewed items are excluded from bulk selection by default with an override opt-in)
- Bulk actions are logged as a batch with individual item records; the batch ID is recorded against each resolved item

---

## 9. Data Source

Queue items are stored in the `review_queue` table (TS-06) with the following fields used by this interface:

| Field | Usage |
|---|---|
| `queue_type` | Filter routing |
| `object_id` + `object_type` | Loading the correct detail panel (§6.x) |
| `assigned_to` | My Items filter; card display |
| `priority` | Sort by priority |
| `created_at` | Age calculation; sort by age |
| `due_at` | SLA status calculation |
| `resolved_at` | Removing items from active queue view |

---

## 10. Open Decisions

> **DECISION NEEDED —** Should the queue page be a dedicated top-level route (e.g. `/queues`) or always accessed via the notification bell dropdown? *Recommendation: both — notification bell for quick triage, dedicated `/queues` route for full queue management. Analysts working the queue need a full-page experience.*

> **DECISION NEEDED —** Should queue items be auto-assigned to analysts based on Neighborhood specialty (DS-12 profile), or always manually assigned? *Recommendation: auto-assign based on specialty tags as a default, with manual override; reduces coordination overhead for senior analysts managing queue routing.*
