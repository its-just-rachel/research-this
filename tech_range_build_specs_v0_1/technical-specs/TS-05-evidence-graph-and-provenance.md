# TS-05 — Evidence Graph and Provenance

**Version:** 0.1
**Status:** Draft
**Last updated:** 2026-07-06
**Owner:** Platform Engineering
**Related docs:** TS-02 (Data Model), TS-06 (Workflow State Machine), ADR-003 (object_relationships table), ADR-015 (Temporal Provenance)

---

## 1. Purpose

Every output Tech Range produces — an Advisory, a Tech Evaluation, a Solution Candidate — carries an implicit claim: *this is what the evidence says.* That claim is only as trustworthy as the chain of evidence behind it. The purpose of this spec is to define how that chain is built, stored, traversed, enforced, and used for retrospective analysis.

This spec governs:

- The structure of the **evidence graph**: the typed, weighted, timestamped network of relationships between all platform entities
- The **provenance model**: what is tracked, at which layer, and how completeness is verified
- **Evidence chain traversal**: how to reconstruct the path from a published output back to its source signals
- **Disconfirmation handling**: how contradicting evidence is preserved and surfaced
- **Immutable history**: how the graph changes over time without losing the past
- **Signal archaeology**: how temporal provenance enables retrospective analysis
- **Enforcement rules**: which provenance requirements are hard gates vs. advisory

The intelligence value of every Tech Range output depends on traceable evidence chains. If we cannot answer "what is this based on and how did we get there?" — the output cannot be published.

---

## 2. What the Evidence Graph Is

### 2.1 Definition

The **evidence graph** is the full set of typed, weighted, timestamped relationships between all platform entities. It captures not just connections, but the nature, confidence, and history of those connections.

Every node in the graph is a platform entity: a Source, Signal, Corroboration Group, Cluster, Technology Profile, Hypothesis, Tech Evaluation, Solution Candidate, Advisory, Analyst, or Task. Every edge is a record in `object_relationships`.

The evidence graph is not a separate graph database. It is the `object_relationships` table traversed via SQL — specifically via recursive CTEs in PostgreSQL. This keeps the evidence graph co-located with the data it describes, eliminates synchronization problems, and makes point-in-time queries straightforward.

### 2.2 The `object_relationships` Table (from TS-02 / ADR-003)

Each row represents one typed, directional relationship between two entities:

| Column | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `source_id` | UUID | The "from" entity |
| `source_type` | entity_type enum | Type of the source entity |
| `target_id` | UUID | The "to" entity |
| `target_type` | entity_type enum | Type of the target entity |
| `relationship_type` | relationship_type enum | See §2.3 |
| `confidence` | NUMERIC(3,2) | 0.00–1.00 |
| `rationale` | TEXT | Human-readable reason; required for some types (see §8) |
| `evidence_basis` | UUID[] | IDs of supporting records; required for some types (see §8) |
| `valid_from` | TIMESTAMPTZ | When this relationship became active |
| `superseded_at` | TIMESTAMPTZ | When this relationship was replaced; NULL = currently active |
| `superseded_by` | UUID | FK to the replacement relationship record |
| `review_status` | review_status enum | e.g., `pending`, `accepted`, `retracted` |
| `created_at` | TIMESTAMPTZ | Record insertion time |

### 2.3 Relationship Types

The full set of typed relationships the platform recognizes:

| Type | Semantic meaning |
|---|---|
| `corroborates` | One entity supports or confirms another |
| `contradicts` | One entity challenges or disconfirms another |
| `clusters_into` | A signal or group belongs to a cluster |
| `classified_as` | An entity is assigned to a taxonomy node |
| `evidences` | An entity is used as evidence for a claim or output |
| `hypothesizes` | An entity proposes a hypothesis |
| `evaluated_by` | An entity is the subject of an evaluation |
| `precedes` | Temporal ordering |
| `supersedes` | Replacement relationship |
| `sourced_from` | An entity was derived from a source |
| `enriched_by` | An entity was modified by an enrichment process |
| `contains` | Structural containment |
| `related_to` | Generic associative relationship (use sparingly) |
| `contributes_to` | An entity is an input to another |
| `assigned_to` | Work assignment |
| `produces` | An entity generates another |
| `parent_of` / `child_of` | Hierarchical containment |
| `enables` | A dependency that unlocks another entity |
| `dependent_on` | An entity requires another to be valid |

### 2.4 What the Evidence Graph Enables

- **Audit trails:** Every claim in a published output can be traced back to its source signals.
- **Lineage reconstruction:** For any entity, the full derivation path can be recovered.
- **Point-in-time replay:** The state of the graph at any historical moment can be reconstructed.
- **Disconfirmation tracking:** Contradicting evidence is a first-class graph citizen, not a soft delete or comment.
- **Confidence propagation:** Confidence scores can be aggregated across the chain using net evidential weight calculations (see §5.3).

---

## 3. Provenance Model

Provenance answers: *where did this come from, how was it processed, and who touched it?* The Tech Range provenance model has three distinct layers.

### 3.1 Layer 1 — Source Provenance

**What it captures:** Where did the raw signal originate? How credible is that origin?

**Fields:**
- `sources.url` / `sources.feed_id` — the access path
- `sources.credibility_score` — platform-assigned credibility rating
- `sources.coverage_scope` — what domains / geographies / industries the source covers
- `signals.event_time` — when the underlying event occurred in the world (not when we saw it)
- `signals.captured_at` — when the source published or recorded it
- `signals.ingested_at` — when the platform ingested it
- `object_relationships` (type=`sourced_from`) — links a signal to its source record

**Provenance-complete at this layer means:** The signal has a non-null `event_time`, `captured_at`, and `ingested_at`; it has a `sourced_from` relationship to a Source record with a non-null `credibility_score`.

### 3.2 Layer 2 — Processing Provenance

**What it captures:** How was the raw signal transformed on its way to an analytical output?

**Fields:**
- `object_relationships` (type=`enriched_by`) — each enrichment step that touched an entity
- `object_relationships` (type=`clusters_into`) — grouping and clustering decisions
- `object_relationships` (type=`classified_as`) — taxonomy assignments, with `rationale` required
- `enrichment_runs.model_version`, `enrichment_runs.parameters` — which model/config produced the transformation
- `signals.created_at` — when the platform created the processed record

**Provenance-complete at this layer means:** Each enrichment step has a corresponding `enriched_by` relationship with a non-null `rationale`. Every `classified_as` relationship has a non-null `rationale` and non-empty `evidence_basis`.

### 3.3 Layer 3 — Analytical Provenance

**What it captures:** How did human judgment interact with the evidence?

**Fields:**
- `object_relationships` where `source_type = 'analyst'` — analyst-authored relationships
- `hypotheses.authored_by`, `hypotheses.created_at` — hypothesis formation authorship
- `analyst_corrections.original_value`, `analyst_corrections.corrected_value`, `analyst_corrections.rationale` — explicit overrides
- `object_relationships` (type=`evidences`) — analyst declaration that an entity is used as evidence in an output
- `object_relationships.review_status = 'accepted'` — analyst acceptance of system-generated relationships

**Provenance-complete at this layer means:** Every `evidences` relationship used in an Advisory has a non-null `rationale`, non-empty `evidence_basis`, and `review_status = 'accepted'` (i.e., analyst-confirmed, not just system-generated).

### 3.4 Summary — Provenance Completeness Requirements

A record is considered **provenance-complete** when all three layers are satisfied for its entity type:

| Entity type | Source layer complete | Processing layer complete | Analytical layer complete |
|---|---|---|---|
| Signal | event_time, captured_at, ingested_at, sourced_from relationship | — | — |
| Corroboration Group | All member signals provenance-complete | classified_as with rationale | — |
| Cluster | All member groups provenance-complete | All classified_as with rationale | — |
| Technology Profile | All member clusters provenance-complete | All classified_as with rationale | Analyst review_status = accepted |
| Hypothesis | Contributing signals provenance-complete | — | authored_by non-null, rationale non-null |
| Advisory | Full evidence chain provenance-complete (see §4.4) | — | All evidences relationships accepted by analyst |

---

## 4. Evidence Chain Traversal

### 4.1 Definition

An **evidence chain** is the path from a published Advisory or Finding back through its supporting entities to the raw signals and sources that underpin it. The canonical path is:

```
Source → Signal → Corroboration Group → Cluster → Technology Profile → Advisory
```

Each hop is an edge in `object_relationships`. Intermediate entities may appear at multiple hops (e.g., a Signal may be linked directly to a Hypothesis that feeds an Advisory). The traversal must capture all paths, not just the canonical one.

### 4.2 Traversal Algorithm — Recursive CTE

```sql
-- Traverse the evidence chain backwards from an Advisory.
-- Returns all entities in the chain with their hop depth.

WITH RECURSIVE evidence_chain AS (
    -- Base case: the Advisory itself
    SELECT
        r.source_id        AS entity_id,
        r.source_type      AS entity_type,
        r.target_id        AS parent_id,
        r.target_type      AS parent_type,
        r.relationship_type,
        r.confidence,
        r.rationale,
        r.evidence_basis,
        1                  AS depth
    FROM object_relationships r
    WHERE r.target_id = :advisory_id
      AND r.target_type = 'advisory'
      AND r.superseded_at IS NULL

    UNION ALL

    -- Recursive step: follow each entity's upstream relationships
    SELECT
        r.source_id,
        r.source_type,
        r.target_id,
        r.target_type,
        r.relationship_type,
        r.confidence,
        r.rationale,
        r.evidence_basis,
        ec.depth + 1
    FROM object_relationships r
    INNER JOIN evidence_chain ec
        ON r.target_id = ec.entity_id
       AND r.target_type = ec.entity_type
    WHERE r.superseded_at IS NULL
      AND ec.depth < :max_depth  -- prevent runaway traversal
)
SELECT *
FROM evidence_chain
ORDER BY depth, entity_type, entity_id;
```

For a point-in-time replay (see §6.3), replace the `superseded_at IS NULL` filters with:

```sql
WHERE r.valid_from <= :as_of
  AND (r.superseded_at IS NULL OR r.superseded_at > :as_of)
```

### 4.3 Maximum Traversal Depth

The canonical six-hop chain is:

| Hop | From | Relationship | To |
|---|---|---|---|
| 1 | Source | sourced_from | Signal |
| 2 | Signal | contributes_to | Corroboration Group |
| 3 | Corroboration Group | clusters_into | Cluster |
| 4 | Cluster | contributes_to | Technology Profile |
| 5 | Technology Profile | evidences | Advisory |
| 6 | Hypothesis | evidences | Advisory |

**Default max_depth: 8.** This provides two hops of headroom for non-canonical paths (e.g., a signal linked directly to a Hypothesis, or a Solution Candidate feeding an Advisory via an additional intermediate). Traversals exceeding depth 8 should log a warning and be flagged for manual review — they indicate either a data quality issue or an unanticipated structural pattern.

Do not set max_depth above 12 in production without explicit DBA sign-off; recursive CTE fan-out on a large graph can degrade query performance significantly.

### 4.4 Chain Completeness Check

A chain is **complete** when every node in the traversal satisfies its provenance-completeness requirements (§3.4) and no required relationship type is missing.

The completeness check function (to be implemented as a PostgreSQL function or application-layer service) verifies:

1. The Advisory has at least one `evidences` relationship with `review_status = 'accepted'`.
2. Every entity at depth 1–3 has a `sourced_from` relationship back to a Source record.
3. No entity in the chain has `current_state = 'draft'` (drafts cannot anchor a published Advisory).
4. Every `classified_as` relationship in the chain has a non-null `rationale`.
5. Every `evidences` relationship used in the Advisory has a non-empty `evidence_basis`.
6. The chain contains at least one Signal with a non-null `event_time`.

The completeness check returns a structured result: `{ complete: boolean, missing: [{ entity_id, entity_type, missing_requirement }] }`.

### 4.5 Publication Gate

**The evidence chain must be traversable and complete at publish time.** Advisory publication is gated by the WorkflowStateService (see TS-06); one of the preconditions for transitioning an Advisory to `published` state is a passing chain completeness check. See §8 for the full enforcement rules.

---

## 5. Disconfirmation Handling

Platform Principle 21 establishes disconfirmation as a first-class concept. The `contradicts` relationship type is not a soft flag or a comment — it is a typed edge in the evidence graph with the same rigor requirements as `corroborates`.

### 5.1 Storage of Contradicting Evidence

`contradicts` relationships are stored in `object_relationships` with:
- `relationship_type = 'contradicts'`
- Non-null `rationale` (required; see §8)
- Non-empty `evidence_basis` (required; see §8)
- `confidence` set to the strength of the contradiction

When a `contradicts` relationship is created, it is never silently discarded or merged into a lower confidence score on the corroborating side. Both the corroborating and contradicting evidence are preserved as independent graph edges.

### 5.2 Competing Hypotheses

When two Hypothesis entities have contradicting evidence — i.e., signals that `corroborate` one and `contradict` the other, or signals that directly `contradict` the hypothesis itself — both hypotheses are preserved in the graph. The platform does not resolve the contradiction by removing the weaker hypothesis. Analysts are responsible for rendering a judgment, which is stored as an `evidences` or `contradicts` relationship authored by the analyst.

### 5.3 Net Evidential Weight

For any entity E, its net evidential weight is calculated as:

```
net_weight(E) = normalize(
    Σ confidence(r) for all r where r.target_id = E.id AND r.relationship_type = 'corroborates' AND r.superseded_at IS NULL
  - Σ confidence(r) for all r where r.target_id = E.id AND r.relationship_type = 'contradicts' AND r.superseded_at IS NULL
)
```

Where `normalize` maps the result to [0.0, 1.0] using:

```
normalize(x) = (x - min_possible) / (max_possible - min_possible)
```

In practice, since all confidence values are in [0.0, 1.0], the raw difference falls in [-N, N] where N is the count of relationships. Normalize by dividing by N (or by the max possible sum), then clamp to [0.0, 1.0].

The net evidential weight is a derived value — it is not stored directly on the entity, but can be computed on demand or materialized as a view for performance.

### 5.4 Surfacing Disconfirming Evidence to Analysts

When a new `contradicts` relationship is added to an entity with `current_state = 'active'`:

1. A review queue entry is created in `analyst_review_queue` with:
   - `entity_id` and `entity_type` of the affected entity
   - `trigger_relationship_id` pointing to the new `contradicts` relationship
   - `queue_reason = 'new_contradiction'`
   - `priority` calculated from the confidence of the contradicting relationship and the publication status of downstream Advisories
2. The entity's `review_status` is set to `pending` if it was previously `accepted`.
3. Downstream Advisories that include the entity in their evidence chain are flagged for review (see §5.5).

Analysts are not automatically notified for every low-confidence contradiction. Notification thresholds are configurable per entity type.

### 5.5 Contradictions Added After Publication

If a `contradicts` relationship is added to any entity in an Advisory's evidence chain after the Advisory has been published:

1. The Advisory transitions from `published` to `under_review` state (per TS-06 workflow rules).
2. A review queue entry is created with `queue_reason = 'post_publication_contradiction'`.
3. The assigned analyst receives a notification.
4. The Advisory remains visible to consumers but is marked with a `review_flag` indicating it is under active review.

The Advisory does not automatically revert to `draft`. An analyst must make a judgment: retract the contradiction (with rationale), update the Advisory, or confirm the Advisory stands despite the new evidence.

---

## 6. Immutable History

### 6.1 No Deletes — Only Supersession

Relationships in `object_relationships` are never deleted. When a relationship is incorrect, outdated, or replaced, it is **superseded**:

1. Set `superseded_at = NOW()` and `superseded_by = <new_relationship_id>` on the old record.
2. Create a new record with the corrected values, `valid_from = NOW()`, and `superseded_at = NULL`.

The old record remains queryable. The new record is the active version.

This applies to:
- System-generated relationships (enrichment, classification, clustering)
- Analyst-authored relationships
- Automated confidence updates

The one exception is `review_status`: this field may be updated in place (e.g., `pending` → `accepted` or `retracted`) because it represents the current disposition of the relationship, not its content.

### 6.2 Querying the Current Graph

To query the current (active) state of the evidence graph, filter on:

```sql
WHERE superseded_at IS NULL
```

This returns only the most recent, non-superseded version of each relationship.

### 6.3 Point-in-Time Replay

To reconstruct the evidence graph as it existed at a historical point `:as_of`:

```sql
WHERE valid_from <= :as_of
  AND (superseded_at IS NULL OR superseded_at > :as_of)
```

This returns every relationship that was active at `:as_of` — created before or at that time and not yet superseded as of that time.

Combined with the temporal fields on entities themselves (see ADR-015 and §3.1), this enables full point-in-time reconstruction of the platform's state.

### 6.4 Applications

**Surprise analysis:** Given a new signal arriving today, how much of its content was already present in the graph (via earlier signals)? Query the graph at the earlier signal's `event_time` and compare.

**"What did we know on date X?":** Reconstruct the full evidence chain for an Advisory as it stood on a given date — which signals were ingested, which contradictions were known, what confidence scores were in effect.

**Audit and compliance:** Demonstrate to an external party exactly what evidence supported a claim at the time it was published.

---

## 7. Signal Archaeology Integration

**Signal archaeology** is retrospective analysis using temporal provenance — using the gap between `event_time`, `captured_at`, and `ingested_at` to understand how knowledge evolved, where intelligence was delayed, and what was knowable at a given moment.

The evidence graph is the primary substrate for archaeological queries. Because every relationship carries `valid_from` and `superseded_at`, and every signal carries `event_time` / `captured_at` / `ingested_at`, the graph can be queried across time.

### 7.1 Archaeological Query Pattern 1 — Late-Arriving Signal Detection

Detects signals where the gap between when the event occurred (`event_time`) and when the platform ingested it (`ingested_at`) is anomalously large. These are candidates for "we should have known sooner" analysis.

```sql
-- Find signals with event_time > 7 days before ingested_at,
-- linked to a specific Advisory's evidence chain.

WITH advisory_signals AS (
    -- Get all signals in the Advisory's evidence chain
    SELECT ec.entity_id AS signal_id
    FROM evidence_chain_for(:advisory_id) ec   -- uses the recursive CTE from §4.2
    WHERE ec.entity_type = 'signal'
)
SELECT
    s.id,
    s.title,
    s.event_time,
    s.captured_at,
    s.ingested_at,
    EXTRACT(EPOCH FROM (s.ingested_at - s.event_time)) / 86400.0 AS event_to_ingest_days,
    EXTRACT(EPOCH FROM (s.captured_at - s.event_time)) / 86400.0 AS event_to_capture_days
FROM signals s
INNER JOIN advisory_signals acs ON s.id = acs.signal_id
WHERE s.ingested_at - s.event_time > INTERVAL '7 days'
ORDER BY event_to_ingest_days DESC;
```

### 7.2 Archaeological Query Pattern 2 — Evidence Chain Reconstruction at a Historical Date

Reconstructs the exact evidence chain that existed for an Advisory on a given date, regardless of subsequent updates.

```sql
-- Evidence chain for Advisory :advisory_id as it existed on :as_of_date.

WITH RECURSIVE historical_chain AS (
    SELECT
        r.source_id AS entity_id,
        r.source_type AS entity_type,
        r.relationship_type,
        r.confidence,
        1 AS depth
    FROM object_relationships r
    WHERE r.target_id = :advisory_id
      AND r.target_type = 'advisory'
      AND r.valid_from <= :as_of_date
      AND (r.superseded_at IS NULL OR r.superseded_at > :as_of_date)

    UNION ALL

    SELECT
        r.source_id,
        r.source_type,
        r.relationship_type,
        r.confidence,
        hc.depth + 1
    FROM object_relationships r
    INNER JOIN historical_chain hc
        ON r.target_id = hc.entity_id
       AND r.target_type = hc.entity_type
    WHERE r.valid_from <= :as_of_date
      AND (r.superseded_at IS NULL OR r.superseded_at > :as_of_date)
      AND hc.depth < 8
)
SELECT * FROM historical_chain ORDER BY depth, entity_type, entity_id;
```

### 7.3 Archaeological Query Pattern 3 — Disconfirmation Timeline

For a given entity, shows when corroborating and contradicting evidence arrived, and when the entity was promoted relative to when contradictions were known.

```sql
-- Disconfirmation timeline for entity :entity_id.
-- Shows the sequence: when was it promoted, when did corroborating evidence arrive,
-- when did contradicting evidence arrive?

WITH entity_relationships AS (
    SELECT
        r.relationship_type,
        r.confidence,
        r.valid_from           AS relationship_created_at,
        r.rationale,
        s.event_time           AS signal_event_time,
        s.ingested_at          AS signal_ingested_at
    FROM object_relationships r
    LEFT JOIN signals s
        ON r.source_id = s.id AND r.source_type = 'signal'
    WHERE r.target_id = :entity_id
      AND r.relationship_type IN ('corroborates', 'contradicts')
),
promotion_event AS (
    SELECT
        wsh.transitioned_at    AS promoted_at,
        wsh.to_state           AS promotion_state
    FROM workflow_state_history wsh
    WHERE wsh.entity_id = :entity_id
      AND wsh.to_state IN ('active', 'published', 'under_review')
    ORDER BY wsh.transitioned_at
    LIMIT 1
)
SELECT
    er.relationship_type,
    er.confidence,
    er.relationship_created_at,
    er.signal_event_time,
    er.signal_ingested_at,
    pe.promoted_at,
    CASE
        WHEN er.relationship_type = 'contradicts'
         AND er.relationship_created_at < pe.promoted_at
        THEN TRUE ELSE FALSE
    END AS contradiction_preceded_promotion
FROM entity_relationships er
CROSS JOIN promotion_event pe
ORDER BY er.relationship_created_at;
```

This query enables the question: *"Did we know this was contested before we promoted it?"*

---

## 8. Provenance Enforcement Rules

### 8.1 Mandatory `rationale` Requirements

The following relationship types require a non-null, non-empty `rationale` field. The application layer (and a database CHECK constraint) must enforce this:

| Relationship type | Required on | Enforcement |
|---|---|---|
| `contradicts` | All instances | DB constraint + app validation |
| `classified_as` | All instances | DB constraint + app validation |
| `evidences` | All instances used in an Advisory | App validation at Advisory state transition |
| `supersedes` | All instances | DB constraint |

### 8.2 Mandatory `evidence_basis` Requirements

The following relationship types require a non-empty `evidence_basis` array:

| Relationship type | Required on | Enforcement |
|---|---|---|
| `contradicts` | All instances | DB constraint + app validation |
| `classified_as` | All instances used in an Advisory evidence chain | App validation at Advisory state transition |
| `evidences` | All instances used in an Advisory | App validation at Advisory state transition |

### 8.3 WorkflowStateService Integration

The WorkflowStateService (TS-06) runs provenance checks as preconditions on state transitions. The following checks are required:

**Advisory: `draft` → `review`**
- Evidence chain traversal returns at least one result (non-empty chain)
- All `evidences` relationships in the chain have non-null `rationale`
- At least one Signal in the chain has a non-null `event_time`
- No entity in the chain is in `draft` state

**Advisory: `review` → `published`**
- Full chain completeness check passes (§4.4) — all five completeness criteria met
- All `evidences` relationships have `review_status = 'accepted'`
- All `contradicts` relationships in the chain have been reviewed (i.e., `review_status` is not `pending`)
- Net evidential weight (§5.3) is above the configured publication threshold (default: 0.6)

**Hypothesis: `draft` → `active`**
- At least one `corroborates` relationship with `confidence >= 0.5`
- `authored_by` is non-null
- `rationale` on the hypothesis record is non-null

**Signal: `raw` → `processed`**
- `event_time` is non-null
- `sourced_from` relationship exists and points to a Source with non-null `credibility_score`

Precondition failures return a structured error response identifying which check failed and why, so the UI can surface actionable feedback to analysts.

---

## 9. Performance Considerations

### 9.1 Indexing Strategy for `object_relationships`

The following indexes are required. Implement before any evidence chain traversal runs in production:

```sql
-- Primary traversal pattern: find all relationships targeting a given entity
CREATE INDEX idx_obj_rel_target
    ON object_relationships (target_id, target_type)
    WHERE superseded_at IS NULL;

-- Reverse traversal: find all relationships from a given entity
CREATE INDEX idx_obj_rel_source
    ON object_relationships (source_id, source_type)
    WHERE superseded_at IS NULL;

-- Relationship type filtering (common in provenance queries)
CREATE INDEX idx_obj_rel_type
    ON object_relationships (relationship_type)
    WHERE superseded_at IS NULL;

-- Point-in-time queries: valid_from + superseded_at
CREATE INDEX idx_obj_rel_temporal
    ON object_relationships (valid_from, superseded_at);

-- evidence_basis array containment queries (e.g., "which relationships cite this signal?")
CREATE INDEX idx_obj_rel_evidence_basis
    ON object_relationships USING GIN (evidence_basis);

-- Composite for the common active-relationship lookup
CREATE INDEX idx_obj_rel_source_target_active
    ON object_relationships (source_id, source_type, target_id, target_type)
    WHERE superseded_at IS NULL;
```

Partial indexes (`WHERE superseded_at IS NULL`) keep the active-graph indexes small and fast, since the vast majority of queries operate on current state.

### 9.2 Recursive CTE Depth Limits

PostgreSQL does not enforce a hard depth limit on recursive CTEs by default. The `:max_depth` parameter in the traversal CTE (§4.2) is the application-layer guard. Default value: 8.

For production deployments with large graphs (> 1M relationship rows), set `statement_timeout` on evidence chain traversal queries (suggested: 30 seconds). Log and alert on queries exceeding 10 seconds — this indicates either a deep or fan-heavy chain that warrants investigation.

Avoid unbounded recursive traversal (omitting `max_depth`) in any application code path. All traversal functions must accept and pass through the depth limit parameter.

### 9.3 Materialized View Candidates

The following traversal paths are candidates for materialization to improve query performance for high-frequency reads. **Defer implementation to the ops team until query performance baselines are established in production** — premature materialization adds maintenance overhead without validated benefit.

| Candidate view | Description | Refresh strategy |
|---|---|---|
| `mv_advisory_evidence_chains` | Pre-computed evidence chains for all published Advisories | Refresh on Advisory publication or evidence chain update |
| `mv_entity_net_weight` | Net evidential weight for all active entities | Refresh on new corroborates/contradicts relationship |
| `mv_provenance_completeness` | Completeness status for all entities | Refresh nightly or on entity state change |

Note: materialized views of evidence chains must be invalidated when any supersession occurs within the chain. Implement using PostgreSQL NOTIFY/LISTEN or a trigger-based invalidation queue, not time-based refresh alone.

---

## 10. DECISION NEEDED

> **DECISION NEEDED —** Should evidence chains for published Advisories be pre-computed and stored as a snapshot at publication time, or should they always be traversed live from `object_relationships`? Pre-computation improves read performance and protects against retroactive graph changes affecting audit reads, but adds a separate snapshot artifact to maintain and introduces the risk of snapshot-vs-live divergence. Live traversal is always accurate but slower and subject to performance degradation as the graph grows. *Recommendation: Implement live traversal first; add snapshot caching as a materialized view (§9.3) once query performance baselines show a need. Snapshots should supplement, not replace, live traversal — the live graph remains the source of truth.*

> **DECISION NEEDED —** Who has authority to mark a `contradicts` relationship as `retracted`, and what process is required? Options: (a) any analyst with write access to the entity; (b) the analyst who created the relationship; (c) a senior analyst or team lead only; (d) any analyst, but retractions require a written rationale and a second-analyst confirmation. Retraction of a contradiction on a published Advisory's chain is a materially different action than retraction of a contradiction on a draft entity. *Recommendation: Distinguish by entity status. For entities in `draft` or `active` state: the creating analyst may retract with a written rationale. For entities in `published` or `under_review` state: retractions require written rationale plus confirmation from a designated reviewer. All retractions are stored as supersessions (never deleted) and generate an audit log entry.*

> **DECISION NEEDED —** Should evidence chain completeness (§4.4) be enforced as a hard gate or a soft warning at Advisory publication time? A hard gate (blocking) ensures no Advisory is published without a complete chain, but may create friction when a chain is structurally complete but missing a non-critical field (e.g., a signal's `captured_at` is null because the source doesn't publish timestamps). A soft warning (non-blocking) allows publication with documented gaps, but risks normalizing incomplete provenance. *Recommendation: Hard gate for the five core completeness criteria defined in §4.4. Introduce a documented exception process: an analyst may override the gate with a written exception rationale, which is stored on the Advisory record and surfaced in the published output. Exception overrides require a second-analyst sign-off. This preserves rigor while allowing edge-case publication.*
