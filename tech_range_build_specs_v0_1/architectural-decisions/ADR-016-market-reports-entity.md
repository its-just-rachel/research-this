# ADR-016: Market Reports as a Canonical Entity Type

**Status:** Proposed
**Date:** 2026-07-07
**Deciders:** architect, senior_analyst
**Related:** TS-02 (Entity Schema), TS-06 (State Machines), TS-07 (Watchlist System), TS-08 (Access Control), TS-09 (API Endpoints), ADR-005 (Immutable-but-Supersedable Pattern), ADR-012 (Sensitive Evidence Handling), ADR-013 (Advisory Publication Standard), ADR-015 (Temporal Provenance Standard), DS-12 (Notification Preferences)

---

## Context

TS-02 establishes 17 canonical entity types. Design spec work for the Research tab identified a gap: the platform has no entity for sustained, topic-scoped research that evolves over time and makes forward-looking predictions.

The existing output artifacts do not fill this gap. Advisories (ADR-013) are closed artifacts — once published, they are not updated; a revision requires a new Advisory with a new evidence chain. Technology Profiles are durable records tied to a specific technology, updated in-place as understanding evolves, but they do not carry predictions or function as standalone research documents. Neither artifact is scoped to a topic area (a Neighborhood or Domain Area) rather than a specific technology or actionable recommendation.

The gap is real and consequential. Analysts conducting sustained research on a technology landscape — tracking a domain over months, building forward views, and refining those views as signals accumulate — have no platform artifact that reflects that work. The output either gets fragmented into a series of Advisories (losing the living, cumulative quality of the research) or lives outside the platform in documents that are not linked to the evidence graph.

The Market Report entity is the missing artifact: a living, versioned research document scoped to a Neighborhood or Domain Area, explicitly designed to carry predictions that can be validated as signals accumulate, and integrated into the watchlist and notification system so analysts can subscribe to updates.

Market Reports belong to the Research tab of the platform. They are a primary catalog entry in that tab's interface, alongside Technology Profiles and other research-oriented entity types.

---

## Decision

Introduce `market_reports` as the 18th canonical entity type, and `market_report_predictions` as a supporting table. A Market Report is a living, versioned, prediction-bearing research document scoped to a Neighborhood or Domain Area. It is distinct from an Advisory (closed, recommendation-focused, full evidence chain required at publication) and from a Technology Profile (durable but not a standalone research document, tied to a specific technology, no predictions).

This decision adds two tables to TS-02, a new state machine to TS-06, a new row to the TS-08 access control matrix, and new endpoints to TS-09. It does not alter any existing entity type.

---

## Key Distinctions from Related Entities

| Dimension | Market Report | Advisory | Technology Profile |
|---|---|---|---|
| Lifecycle | Living document — updated over time, versioned | Closed artifact — published once; revision requires a new Advisory | Durable record — updated in-place as understanding evolves |
| Scope | A topic area: a Neighborhood or Domain Area | A specific recommendation for a specific strategic context | A specific technology |
| Predictions | Yes — explicit, dated, tracked to resolution | No | No |
| Versioning | Explicit version numbers; old versions preserved per ADR-005 | New Advisory supersedes old; both preserved | In-place updates with `updated_at`; no version history |
| Evidence chain | Referenced via `evidence_basis` UUID array; not required at draft or publish | Full traversable evidence chain required before publication (ADR-013) | Evidence accumulated over time; no publication gate |
| Approval to publish | `senior_analyst` role | `senior_analyst` role minimum; `architect` if sensitive evidence present (ADR-013) | Governed by ADR-007 |
| Watches and subscriptions | Yes — integrated with TS-07 watchlist | No dedicated watch model | No dedicated watch model |
| Primary tab | Research | Advisories / Intelligence Outputs | Technology Profiles / Research |

---

## Schema

```sql
CREATE TABLE market_reports (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title                 TEXT NOT NULL,
  topic_scope_type      TEXT NOT NULL CHECK (topic_scope_type IN ('neighborhood', 'domain_area')),
  topic_scope_ref       TEXT NOT NULL,             -- the neighborhood or domain_area name
  current_version       INTEGER NOT NULL DEFAULT 1,
  summary               TEXT NOT NULL,
  body                  TEXT,                      -- full report body; may be a sensitive_ref_id pointer per ADR-012
  prediction_horizon    DATE,                      -- target date by which predictions are expected to resolve
  state                 TEXT NOT NULL DEFAULT 'draft'
                        CHECK (state IN ('draft', 'under_review', 'published', 'archived')),
  evidence_basis        UUID[] NOT NULL DEFAULT '{}',
  watch_count           INTEGER NOT NULL DEFAULT 0,
  created_by            UUID NOT NULL,
  published_at          TIMESTAMPTZ,
  last_updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  superseded_at         TIMESTAMPTZ,
  superseded_by         UUID REFERENCES market_reports(id),
  is_active             BOOLEAN NOT NULL GENERATED ALWAYS AS (superseded_at IS NULL) STORED,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE market_report_predictions (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  market_report_id      UUID NOT NULL REFERENCES market_reports(id),
  claim                 TEXT NOT NULL,
  prediction_date       DATE NOT NULL,
  resolution_date       DATE,
  resolution_status     TEXT NOT NULL DEFAULT 'pending'
                        CHECK (resolution_status IN (
                          'pending', 'confirmed', 'refuted', 'partially_confirmed'
                        )),
  resolution_evidence   UUID[],                   -- signal/finding UUIDs that resolved this prediction
  resolution_note       TEXT,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Five-Timestamp Compliance (ADR-015)

`market_reports` is an analyst-authored entity type, not an ingested signal. As such, the temporal semantics differ from raw signal tables:

- `event_time` does not apply — Market Reports do not represent a single real-world event. This column is **omitted** from `market_reports` and `market_report_predictions`.
- `captured_at` and `ingested_at` do not apply for the same reason. These columns are omitted.
- `created_at` is present and DB-side as standard.
- `valid_from` is represented by `last_updated_at` for the living-document update model, and by the standard `created_at` for `market_report_predictions`.
- `superseded_at` and `superseded_by` implement the ADR-005 immutable-but-supersedable pattern at the version level.

> **DECISION NEEDED —** Should `market_reports` carry the full five-timestamp columns (with `event_time`, `captured_at`, `ingested_at` nullable or omitted) for schema uniformity, or should analyst-authored entities be explicitly carved out of the ADR-015 five-timestamp standard? *Recommendation: carve out analyst-authored entities explicitly. Define a narrower two-column standard (created_at, last_updated_at) for entities that originate from analyst authorship rather than signal ingestion, and document this carve-out as an amendment to ADR-015. Applying ingestion provenance columns to authored content creates semantic confusion and adds columns that will always be NULL.*

### Indexing

```sql
CREATE INDEX ON market_reports (topic_scope_type, topic_scope_ref);
CREATE INDEX ON market_reports (state);
CREATE INDEX ON market_reports (superseded_at) WHERE superseded_at IS NULL;
CREATE INDEX ON market_reports (created_by);
CREATE INDEX ON market_report_predictions (market_report_id);
CREATE INDEX ON market_report_predictions (resolution_status);
CREATE INDEX ON market_report_predictions (prediction_date);
```

---

## Versioning Model

Market Reports use the immutable-but-supersedable pattern from ADR-005. When a Market Report is updated substantially:

1. A new record is created with `current_version = old_version + 1`.
2. The old record's `superseded_at` is set to `now()` and `superseded_by` is set to the new record's id.
3. The `is_active` generated column on the old record transitions to `false` automatically.
4. Predictions are **not** automatically carried forward. The analyst explicitly decides which predictions from the prior version remain in the new version. Predictions linked to a superseded `market_report_id` are preserved as historical record; new or continued predictions are created against the new record's id.
5. Minor updates (see DECISION NEEDED below) may be applied in-place without a version increment.

Superseded versions are preserved indefinitely. Readers may traverse `superseded_by` to reach the current version of any report. The current version is always the record where `is_active = true` (i.e., `superseded_at IS NULL`).

> **DECISION NEEDED —** What constitutes a version-increment vs. an in-place edit? *Recommendation: version increment required for any change to `summary`, `body`, `predictions`, or `prediction_horizon`; in-place edit allowed for metadata-only changes (`title` typos, `topic_scope_ref` corrections, `watch_count` updates). Rationale: changes to analytical substance should be versioned so prior readers who subscribed to the report can see that the analysis has changed; metadata corrections do not alter the analytical record. Enforcement: the API layer checks the changed field set before deciding which update path to invoke.*

---

## State Machine

```
draft → under_review → published → archived
                  ↑
          (return to draft)
```

- `draft`: initial state on creation. Editable by the creating analyst. Not visible to viewers.
- `under_review`: submitted for `senior_analyst` review. In-place edits blocked pending review outcome.
- `published`: visible to all roles with read access. Triggers watchlist notifications per TS-07 and DS-12.
- `archived`: no longer active; visible as historical record but excluded from default Research catalog views.

Any state may be **superseded** by a new version. Supersession is not a state transition on the original record — it is the creation of a new record that claims the `superseded_by` pointer. The original record's state is frozen at whatever state it held when superseded.

### Governance

| Action | Minimum role |
|---|---|
| Create draft | `analyst` |
| Edit draft | `analyst` (own drafts); `senior_analyst` (any draft) |
| Submit for review (`draft` → `under_review`) | `analyst` |
| Approve and publish (`under_review` → `published`) | `senior_analyst` |
| Return to draft (`under_review` → `draft`) | `senior_analyst` |
| Archive (`published` → `archived`) | `senior_analyst` |
| Create new version (supersede current) | `analyst` (initiates draft); `senior_analyst` (publishes) |
| Govern entity type definition | `architect` |

---

## Watch and Subscription Model

Market Reports integrate with the TS-07 watchlist system. An analyst or viewer with at least `viewer` role may watch a Market Report. The `watch_count` column on `market_reports` is maintained by the watchlist system and must not be modified directly by application code.

When a new version of a watched Market Report transitions to `published`, the watchlist system delivers an alert to each subscriber per their notification preferences (DS-12). The alert includes:

- Report title and new version number
- Summary of changes (if the publishing analyst provides a change note — see DECISION NEEDED below)
- Link to the new version

> **DECISION NEEDED —** Should the `market_reports` schema include a `change_note TEXT` field to allow the publishing analyst to describe what changed in this version? *Recommendation: yes — add `change_note TEXT` as an optional field on `market_reports`, populated at publish time. The change note is included in watchlist alert payloads and displayed in the version history UI. An empty change note is permitted but discouraged. The API should prompt for a change note when the `under_review → published` transition is triggered.*

---

## Prediction Resolution

Predictions in `market_report_predictions` are created when the analyst authors or updates a Market Report. Each prediction carries:

- A `claim` (free text statement of what the report predicts)
- A `prediction_date` (when the prediction was made)
- A `resolution_date` (when the prediction was formally assessed — set at resolution time)
- A `resolution_status` (`pending`, `confirmed`, `refuted`, `partially_confirmed`)
- `resolution_evidence` (UUIDs of the signals or findings that supported the resolution judgment)
- A `resolution_note` (free text explanation of the resolution)

Resolution of predictions is a distinct action from updating the Market Report. An `analyst` may mark predictions as resolved without triggering a version increment on the parent report. When a prediction's `prediction_horizon` date passes and the prediction remains `pending`, the system surfaces it in the analyst's attention queue.

> **DECISION NEEDED —** Should prediction resolution require `senior_analyst` approval, or may the authoring `analyst` resolve their own predictions? *Recommendation: the authoring analyst may resolve their own predictions with no additional approval, but the resolution is flagged for `senior_analyst` visibility in the report's audit trail. Rationale: predictions are analytical judgments that should be owned by the analyst who made them; requiring separate approval for every resolution would create friction that discourages the use of predictions altogether. Senior analyst review of the resolution trail during the next report version cycle is sufficient oversight.*

---

## Sensitive Content

The `body` field may reference sensitive sources even when the `summary` does not. The ADR-012 pointer pattern applies to the `body` field: the field may contain either inline text or a `sensitive_ref_id` pointer to content stored in the sensitive evidence store.

The `summary` field is not eligible for the pointer pattern — it must contain inline text. If a report's summary cannot be written without sensitive content, the report itself is too sensitive for the standard publication pathway and requires `architect` review.

> **DECISION NEEDED —** Should Market Reports that contain a sensitive `body` pointer require `architect` approval to publish, consistent with the sensitive evidence escalation in ADR-013? *Recommendation: yes — if `body` contains a `sensitive_ref_id` pointer, the required approval tier for the `under_review → published` transition escalates to `architect`, matching the behavior established in ADR-013 for Advisories. This escalation is automated: the API detects the pointer pattern at submission time and adjusts the required approval role accordingly.*

---

## Impact on Existing Specs

The following specifications require updates as a consequence of this decision:

- **TS-02** — Add `market_reports` and `market_report_predictions` to the canonical entity type registry and schema definition. Document the five-timestamp carve-out for analyst-authored entities (pending DECISION NEEDED resolution above).
- **TS-06** — Add the Market Report state machine (`draft → under_review → published → archived`) with transition rules and governance roles.
- **TS-07** — Register `market_reports` as a watchable entity type. Define alert payload for new-version notifications: when a market report transitions to `published` (is_active = true, superseded_by IS NULL), trigger `market_report_version_published` alert to all watchlist subscribers with payload `{ object_type: 'market_report', object_id, title, version: current_version, topic_scope_ref }`.
- **TS-08** — Add `market_reports` and `market_report_predictions` to the access control matrix with the role assignments defined in the State Machine section above.
- **TS-09** — Add the following endpoint group: `POST /v1/market-reports`, `GET /v1/market-reports`, `GET /v1/market-reports/:id`, `PATCH /v1/market-reports/:id`, `POST /v1/market-reports/:id/versions`, `GET /v1/market-reports/:id/predictions`, `PATCH /v1/market-report-predictions/:id/resolve`.
- **DS-12** — Add Market Report new-version alerts to the notification preference schema. Users should be able to opt into or out of these alerts independently of other alert types.
- **DS-04 (Research tab design spec)** — Market Reports are a primary entity in the Research catalog. The Research tab must display a Market Reports section with filtering by `topic_scope_type`, `topic_scope_ref`, and `state`. The current version of each report is the default view; version history is accessible via a version history panel.

---

## Consequences

### Positive

- Analysts have a platform-native artifact for sustained, living research — the analytical work stays inside the platform and inside the evidence graph rather than living in external documents.
- Predictions are made explicit and trackable. The platform accumulates a record of forward-looking claims and their resolution, which becomes a durable measure of analytical quality over time.
- Watchlist integration means stakeholders can follow specific research areas without polling. New versions surface automatically through existing notification infrastructure.
- The versioning model preserves the full history of a living research document, enabling point-in-time retrieval of prior analytical views on a topic area.

### Negative

- Adding an 18th canonical entity type increases schema surface area and the number of tables, state machines, and API endpoints the platform must maintain.
- The living-document model introduces a class of user expectation that differs from the published-and-closed Advisory model. Communication to analysts about the difference between the two artifact types will be necessary to avoid confusion.
- Prediction resolution requires ongoing analyst attention as `prediction_horizon` dates arrive. Without discipline, predictions will accumulate in `pending` state indefinitely and lose their value as an accountability mechanism.
- The sensitive body pointer pattern adds API complexity at the publish transition: the API must inspect the `body` field for pointer syntax to determine the required approval tier.

---

## Related Decisions

- **ADR-002** — Canonical storage model; PostgreSQL relational persistence used for both new tables.
- **ADR-005** — Immutable-but-supersedable pattern; Market Report versioning follows this pattern directly.
- **ADR-012** — Sensitive evidence pointer pattern; applies to the `body` field of `market_reports`.
- **ADR-013** — Advisory publication standard; Market Reports are explicitly distinct from Advisories in lifecycle and evidence chain requirements.
- **ADR-015** — Five-timestamp standard; analyst-authored entities may require a documented carve-out (see DECISION NEEDED above).

---

*This ADR proposes the introduction of `market_reports` as the 18th canonical entity type. It becomes authoritative upon acceptance. Downstream specs (TS-02, TS-06, TS-07, TS-08, TS-09, DS-12) must be updated to reflect this decision before implementation begins.*
