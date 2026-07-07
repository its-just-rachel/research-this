# DS-09 — Signal Archaeology Workflow

**Status:** Draft
**Last updated:** 2026-07-07
**Related:** ADR-015 (bi-temporal data model), TS-03 (Ingestion and Enrichment Pipeline), TS-05 (temporal query patterns), TS-08 (roles and audit policy), DS-08 (Governance Queues)

---

## 1. Purpose

Signal archaeology provides retrospective analysis tools for the Tech Range platform. It enables analysts to reconstruct past platform states, detect late-arriving signals, run structured surprise reviews, and inject historical signals into the evidence record.

The capability rests on the five timestamps captured per record — `event_time`, `captured_at`, `ingested_at`, `created_at`, `valid_from` — and the `valid_from` / `superseded_at` pair that enables bi-temporal queries (per ADR-015 and TS-05). Together these make it possible to ask:

- "What did the platform know as of [date]?"
- "Did we see this coming?"
- "This signal arrived late — does it change the picture retroactively?"
- "If we inject this known historical event, how would the platform have classified it?"

Primary users are senior analysts conducting investigation and architects calibrating platform behavior. Analysts (non-senior) have read-only access to archaeology results.

Signal archaeology tools are surfaced in the **Surface tab** (advanced panel) and as a dedicated view for senior analysts. The late-signal detection queue is displayed within this workflow but is governed by DS-08 (Governance Queues).

---

## 2. Access

| Role | Access level |
|---|---|
| viewer | No access |
| analyst | Read-only: can view archaeology query results; cannot run queries or inject signals |
| senior_analyst | Full query access; can run surprise reviews; can inject signals with architect approval |
| architect | Full access; can inject signals without additional approval |

**Surface tab placement:** Archaeology tools appear in an expandable "Signal Archaeology" panel within the Surface tab. This panel is collapsed by default — it is not part of the default Surface view. Users expand it explicitly.

**Direct route:** accessible via a dedicated route (`/archaeology`) or via a direct link from Surface or Research. `senior_analyst+` only. Not a top-level utility nav item — see DS-01 §2.2. This view provides the full workflow without the Surface tab context.

---

## 3. Point-in-time reconstruction

### 3.1 Query interface

A date/time picker and scope selector let the analyst specify what to reconstruct:

- **As of:** date/time picker (required)
- **Scope:** All Profiles / Specific Neighborhood (search/select) / Specific Profile (search/select)
- **Run query** → results panel

The query executes a bi-temporal lookup using `valid_from` / `superseded_at` to return the state of Technology Profiles and their `match_sets` as they existed at the selected moment (per TS-05).

### 3.2 Results display

Results render the Technology Profiles and their classification state as of the selected timestamp. The view is visually distinguished from the live view by a persistent banner:

> Viewing historical state — [date/time]. This is a read-only reconstruction.

All edit, classify, and flag actions are disabled in historical mode.

**Comparison mode:** Senior analyst+ can enable a side-by-side view comparing the historical reconstruction against the current live state. The comparison highlights:

- Profiles that did not yet exist at the historical date
- State transitions (e.g., `emerging` → `active`) that occurred in the interval
- Reclassifications: profiles whose `match_sets` or Domain Area assignments changed
- Signals that arrived after the historical date but relate to the same profiles (late signals)

This comparison is the primary tool for "did we see this coming?" analysis.

### 3.3 Export

Senior analyst+ can export a point-in-time reconstruction as a PDF snapshot. The export includes the query parameters, the resulting profile states, and any comparison delta if comparison mode was active. Exports are used for retrospective briefings and postmortems.

---

## 4. Late-signal detection queue

### 4.1 What it shows

Signals where `ingested_at - event_time` exceeds a configured threshold. These are signals whose underlying events occurred significantly earlier than when the platform received them. A late-arriving signal may retroactively change what the platform "knew" at a past date, making it relevant to archaeology.

> **DECISION NEEDED —** What is the late-signal detection threshold (`ingested_at - event_time`)? *Recommendation: flag signals where lag > 30 days as noteworthy; flag signals where lag > 90 days as significant; both thresholds configurable by architect.*

### 4.2 Queue interface

A filterable list of late-arriving signals. Columns:

| Column | Description |
|---|---|
| Signal summary | Short description of the signal |
| event_time | When the underlying event occurred |
| ingested_at | When the platform received the signal |
| Lag | Computed duration (`ingested_at - event_time`) |
| Lag tier | Noteworthy / Significant (based on thresholds) |
| Affected Profile / Cluster | Linked Technology Profile or Cluster, if classified |
| Status | Unreviewed / Flagged / Dismissed |

Filters: lag tier, affected neighborhood, date range, status.

### 4.3 Analyst actions

- **View signal detail:** open the full signal record
- **Trigger archaeology comparison:** run a point-in-time reconstruction for `[event_time]` and compare against current state — surfaces what the platform looked like when the event occurred vs. now
- **Flag for review:** route to a senior analyst to assess whether this late signal changes any classifications; becomes an item in the DS-08 governance queue
- **Dismiss:** mark as reviewed with no action required; requires a brief dismissal note

---

## 5. Surprise review workflow

### 5.1 Purpose

A structured workflow for the question: "Given that [event] occurred on [date], what signals did the platform hold at that time, and should we have anticipated it?"

Surprise reviews produce documented findings that can seed Advisories or be linked to existing ones. They are the primary mechanism for postmortem analysis and platform calibration.

> **DECISION NEEDED —** Should surprise reviews be visible to all analyst+ roles, or only to the creating analyst and senior analysts? *Recommendation: visible to all senior_analyst+ roles; analysts can see only their own reviews. Surprise reviews may contain sensitive analytical judgments about past platform performance.*

### 5.2 Workflow

1. **Create review:** Analyst describes the event, sets the event date, and optionally links to a relevant Domain Area or Neighborhood. A title and brief description are required.

2. **System reconstruction:** The platform automatically runs a point-in-time reconstruction covering `[event_date − 90 days]` to `[event_date]`. This establishes the signal corpus and profile states the platform held in the window leading up to the event.

3. **AI-assisted scan (Tier 2):** The AI scans the signal corpus for the reconstruction window and highlights signals that, in retrospect, may have been early indicators of the event. Highlighted signals are presented with a confidence note; analysts treat these as suggestions, not conclusions.

4. **Analyst review:** The analyst works through the highlighted signals:
   - Were they classified correctly at the time?
   - Were any missed or incorrectly dismissed?
   - Were corroboration groups formed appropriately?
   - Were there late signals (from the late-signal queue) that, if arrived earlier, would have changed the picture?

5. **Document findings:** The analyst writes up their conclusions as one or more Observations or Findings (canonical entity types). Findings are attached to the surprise review record.

6. **Link to Advisory:** Findings can seed a new Advisory or be linked to an existing one. This closes the loop between retrospective analysis and forward-facing intelligence products.

### 5.3 Surprise review list

Senior analyst+ can access a list of all surprise reviews they are permitted to see:

| Column | Description |
|---|---|
| Event description | Short title of the event under review |
| Event date | The date the event occurred |
| Author | Creating analyst |
| Status | In progress / Complete |
| Linked findings | Count of Observations / Findings attached |
| Linked Advisory | Advisory seeded or linked from this review, if any |

---

## 6. Intentional signal injection

### 6.1 Purpose

Intentional injection inserts a signal with a past `event_time` into the platform's evidence record. Use cases include:

- Adding a known historical technology milestone as a signal to test how the platform would have classified it
- Filling a known gap in the evidence record (e.g., a source that was unavailable at the time)
- Calibrating classification behavior against documented historical events

Injected signals become part of the permanent evidence graph and are permanently marked as injected.

### 6.2 Governance

- **senior_analyst:** can inject with approval from an architect; injection is held in a pending state until approved
- **architect:** can inject directly without additional approval
- All injected signals are marked `ingestion_mode = 'intentional_injection'` (per TS-03)
- Injected signals are visually distinguished in the evidence graph with a permanent indicator: "Retroactively injected on [injection date] by [analyst]"
- Injection triggers reclassification consideration: the Comparative Classification Engine (CCE) is notified to re-evaluate any affected Technology Profiles or Clusters
- Injected signals cannot be deleted — they may only be archived (per TS-08 no-hard-delete policy)

### 6.3 Injection form

| Field | Required | Notes |
|---|---|---|
| Signal summary | Yes | Short description of the signal content |
| Source | Yes | Must be an existing source record, or a new source with written justification |
| event_time | Yes | The historical date/time the event occurred |
| signal_type | Yes | Standard signal type taxonomy |
| Neighborhood hint | No | Optional — helps with initial routing |
| Domain Area hint | No | Optional |
| Injection rationale | Yes | Why this signal is being injected; what gap it fills or what test it supports |
| Approver | Conditional | Required if submitter is senior_analyst; routes to architect approval queue |

On submission by a senior_analyst, the injection enters a pending state and appears in the architect's approval queue (DS-08). On approval (or on direct submission by an architect), the signal is written to the evidence store with `ingestion_mode = 'intentional_injection'` and the CCE is notified.

### 6.4 Audit trail

Every injected signal generates a permanent audit log entry recording:

- Injecting analyst
- Approving architect (if applicable)
- Injection timestamp (`created_at`)
- The stated `event_time`
- Injection rationale (full text)
- CCE notification status

The audit log entry is immutable and is linked from the signal record. Injected signals are flagged in any point-in-time reconstruction that includes them, so analysts running archaeology queries can distinguish organic signals from injected ones.

---

## 7. Summary of open decisions

| # | Question | Recommendation |
|---|---|---|
| 1 | Late-signal detection threshold | Flag lag > 30 days as noteworthy; lag > 90 days as significant; thresholds configurable by architect |
| 2 | Surprise review visibility | Visible to all senior_analyst+ roles; analysts see only their own reviews |
