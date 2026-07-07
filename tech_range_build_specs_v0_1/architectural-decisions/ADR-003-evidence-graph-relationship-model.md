# ADR-003: Evidence Graph Relationship Model

**Status:** Accepted
**Date:** 2026-07-06
**Deciders:** Platform Architecture
**Supersedes:** —
**Superseded by:** —

---

## Context

Tech Range's intelligence value is not stored in entities — it is stored in the *chains of evidence that connect them*. A Signal that cannot be traced to its corroborating observations, the analyst rationale that clustered it, and the review that accepted or challenged that cluster is not intelligence; it is an assertion. Assertions decay. Evidence chains don't.

The platform currently defines 17 entity types: Source, Signal, Enrichment, Corroboration Group, Cluster, Neighborhood, Emerging, Domain Area, Assignment/Match, Observation, Hypothesis, Technology Profile, Tech Evaluation, Solution Candidate, Prototype, Finding, and Advisory. These entities have rich, semantically meaningful relationships with each other — a Signal corroborates a Cluster, a Cluster is classified as a Technology Profile, a Technology Profile evidences a Solution Candidate, a Hypothesis contradicts an earlier classification. Each of those relationships carries distinct intelligence weight and must be independently auditable.

Three forces drive the need for a disciplined relationship model:

1. **Auditability requirements.** Defense-sector and intelligence-adjacent workflows require that every inference can be walked back to its source evidence. A relationship with no provenance is inadmissible for operational use.

2. **Confidence as a first-class attribute.** The platform does not produce binary judgments. Every relationship has a confidence score (0.0–1.0, stored as `NUMERIC(3,2)`) that reflects the strength of the underlying evidence. That confidence must be stored on the relationship itself, not inferred from entity state.

3. **Disconfirmation as signal, not noise.** Contradicting evidence is as analytically valuable as confirming evidence. A model that can only represent "things that are true" cannot represent the real state of intelligence analysis, where competing hypotheses coexist until evidence resolves them. Negative evidence must be first-class.

The relationship model described in this ADR was selected to satisfy these forces without sacrificing query practicality or operational performance.

---

## Decision

All relationships between platform entities are stored in a single generic `object_relationships` table in PostgreSQL. Every row in this table represents one typed, evidenced, confidence-scored relationship between two entities. No relationship exists in the platform that is not represented here (with the narrow exception of match_set membership, described below).

### Table Definition

```sql
CREATE TABLE object_relationships (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id         UUID NOT NULL,
    source_type       TEXT NOT NULL,
    target_id         UUID NOT NULL,
    target_type       TEXT NOT NULL,
    relationship_type TEXT NOT NULL,
    confidence        NUMERIC(3,2) NOT NULL CHECK (confidence >= 0.0 AND confidence <= 1.0),
    rationale         TEXT,
    evidence_basis    UUID[] NOT NULL DEFAULT '{}',
    created_by        UUID NOT NULL,        -- references users or processes
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_from        TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at     TIMESTAMPTZ,          -- NULL = currently active
    review_status     TEXT NOT NULL DEFAULT 'pending'
                          CHECK (review_status IN ('pending', 'accepted', 'challenged', 'retracted'))
);
```

### Indexing Strategy

```sql
-- Primary traversal patterns
CREATE INDEX idx_or_source ON object_relationships (source_id, source_type) WHERE superseded_at IS NULL;
CREATE INDEX idx_or_target ON object_relationships (target_id, target_type) WHERE superseded_at IS NULL;
CREATE INDEX idx_or_type   ON object_relationships (relationship_type)       WHERE superseded_at IS NULL;

-- Confidence-filtered queries (e.g., "show only high-confidence links")
CREATE INDEX idx_or_confidence ON object_relationships (confidence DESC)     WHERE superseded_at IS NULL;

-- Evidence lookup (who else cites this evidence UUID?)
CREATE INDEX idx_or_evidence ON object_relationships USING GIN (evidence_basis);

-- Temporal history (full audit walk including superseded rows)
CREATE INDEX idx_or_source_history ON object_relationships (source_id, valid_from);
```

### Match Sets

Match sets (the comparative classification model from ADR-005 and TS-04) express relationships through their own membership and scoring tables. Where a match set membership implies a canonical `classified_as` or `evaluated_by` relationship, that relationship is mirrored into `object_relationships` by the scoring pipeline so that the evidence graph remains complete. Match set data is the authoritative source for comparative scoring; `object_relationships` is the authoritative source for the evidence graph.

### Immutability and Temporal Provenance

Rows in `object_relationships` are never updated to change their semantic content. When a relationship is revised or superseded:

1. The existing row receives `superseded_at = now()` and `review_status = 'retracted'`.
2. A new row is inserted with the revised content and a fresh `valid_from`.

This preserves the complete history of how the platform's understanding evolved. Queries that need the current state of the graph filter on `superseded_at IS NULL`. Queries that need the full provenance history do not apply that filter.

---

## Rationale

### Why not per-entity foreign keys?

The natural-key approach — adding a `cluster_id FK` to the signals table, a `profile_id FK` to the clusters table, etc. — is simple to implement for straightforward hierarchies but fails in practice for this platform:

- It cannot represent many-to-many relationships without junction tables that proliferate with every new entity type.
- It cannot store confidence, rationale, or evidence_basis on the relationship itself without extending every junction table with those columns.
- Adding a new relationship type between existing entities requires a schema migration.
- Negative relationships (`contradicts`) have no natural home — they cannot be expressed as a foreign key.

Per-entity foreign keys are appropriate for strict 1:N ownership relationships (e.g., a Signal belongs to exactly one Corroboration Group). They are used in the platform for those cases. They are not appropriate for the intelligence graph.

### Why not a dedicated graph database (e.g., Neo4j)?

A graph database would provide native traversal performance and a natural fit for relationship-first data. It was seriously considered. The reasons for not adopting one for the v0.1 build are:

- **Operational complexity.** Running a second database engine alongside PostgreSQL adds infrastructure burden, a separate backup/recovery surface, and a new operational skill requirement at a stage when the team needs to move fast.
- **Transactional integrity.** Keeping entity state (in PostgreSQL) and relationship state (in a graph database) consistent across writes requires distributed transactions or eventual consistency patterns. For an intelligence platform where consistency is a correctness requirement, not a preference, this cost is high.
- **Query surface.** The platform's analytical queries combine entity attribute filtering with relationship traversal (e.g., "find all Signals with source credibility > 0.7 that contradict a Cluster whose review_status is 'accepted'"). These queries are natural SQL joins. Expressing them across two query languages is awkward.
- **Migration path.** If traversal depth and volume grow to the point where a graph database is warranted, the `object_relationships` table is a clean extraction target. The decision is reversible.

### Why not JSONB blobs?

Storing relationships as JSONB embedded in entity records or in a generic key-value store was considered and rejected:

- JSONB structures are not queryable with the same reliability as typed columns. Filtering on confidence, indexing evidence_basis arrays, and joining on relationship_type all require the values to be first-class columns.
- Schema enforcement is weak. The platform requires that every relationship row has a non-null confidence, a valid review_status, and a non-null created_by. These constraints cannot be enforced on JSONB without check constraints that replicate a schema in a fragile way.
- JSONB blobs make it easy to add fields without governance. The relationship model needs to be governed; unstructured storage works against that.

---

## Relationship Type Taxonomy

The following relationship types are defined in v0.1. They are enforced by a `CHECK` constraint on `relationship_type` (or, if governed externally, by an `ENUM` or a reference table — see DECISION NEEDED below).

| Type | Direction | Description |
|---|---|---|
| `corroborates` | Signal → Cluster, Signal → Signal | This source or signal strengthens the evidential basis for the target. Positive evidence. |
| `contradicts` | Signal → Cluster, Signal → Signal, Hypothesis → Relationship | This source or signal weakens or directly opposes the target. Negative evidence. First-class disconfirmation. |
| `clusters_into` | Signal → Corroboration Group, Corroboration Group → Cluster | This entity was grouped or clustered into the target by an analytical process. |
| `classified_as` | Cluster → Technology Profile, Observation → Category | This entity was classified into the target category or profile type. |
| `evidences` | Signal → any, Source → any | This entity provides direct evidential support for the target's existence or properties. |
| `hypothesizes` | Hypothesis → Cluster, Process → Relationship | This entity proposes the target as a hypothesis, pending evidential resolution. Confidence typically low at creation. |
| `evaluated_by` | Solution Candidate → Advisory, Cluster → Tech Evaluation | This entity was formally evaluated by the target process or actor. |
| `precedes` | Signal → Signal, Signal → Signal | Temporal ordering relationship. This entity precedes the target in time and may be causally relevant. |
| `supersedes` | any → any | This entity or relationship replaces the target, which should be treated as outdated. Pairs with `superseded_at` on the target row. |
| `sourced_from` | Signal → Source, Enrichment → Source | This entity was derived from or collected via the target source. |
| `enriched_by` | Technology Profile → Enrichment | This entity's properties were augmented by the target enrichment process. |

New relationship types must go through the governance process described in the DECISION NEEDED section below. Undocumented types added directly to production data will fail validation.

---

## Disconfirmation

The `contradicts` relationship type is a first-class citizen of the evidence graph, not an edge case to be handled by soft-deleting conflicting records or by notes fields on entities.

### How `contradicts` affects confidence

When a `contradicts` relationship is created pointing at an existing relationship or entity, the platform does not automatically revise confidence scores. Confidence is calculated explicitly, not inferred. However:

- The scoring pipeline reads `contradicts` relationships when computing the `evidence_basis` weight for a Cluster or Technology Profile.
- A high-confidence `contradicts` relationship from a high-credibility source decreases the net evidential weight for the target.
- The formula for net evidential weight is: `Σ(confidence_i × source_credibility_i)` for corroborating relationships minus `Σ(confidence_j × source_credibility_j)` for contradicting relationships, normalized to [0.0, 1.0].
- The resulting net weight is surfaced in the UI as the entity's `evidence_strength` attribute, which is distinct from any individual relationship's confidence.

### How `contradicts` relationships are created

- Manually by analysts during review (the primary path).
- Automatically by enrichment jobs that detect direct contradictions with existing signals (e.g., a vendor's claimed capability contradicts a filed patent record).
- By the comparison pipeline when a match set score falls below a threshold that implies the classification hypothesis is not supported.

Contradicting relationships require a non-null `rationale`. A `contradicts` row with an empty rationale is rejected at write time.

### Competing hypotheses

It is valid and expected for the graph to contain two `classified_as` relationships from the same entity to two different Technology Profiles, one with `review_status = 'accepted'` and one with `review_status = 'challenged'`, representing an unresolved classification dispute. The platform does not force premature resolution. The dispute is itself an intelligence signal.

---

## Consequences

### What this enables

- **Complete audit trails.** Any inference the platform produces can be walked back to its evidence chain via the `evidence_basis` UUID array and the chain of relationships that led to it.
- **Confidence-aware queries.** Analysts can filter and rank relationships by confidence, source credibility, and review status using standard SQL.
- **Temporal replay.** The full history of how any entity's relationship graph evolved is preserved and queryable by filtering on `valid_from` and `superseded_at`.
- **Disconfirmation as analysis.** Contradicting evidence is stored, surfaced, and factored into confidence calculations rather than discarded.
- **Schema flexibility without schema migrations.** Adding a new relationship type between existing entity types requires only a governance review and a reference table update, not a DDL change.
- **Evidence provenance for every relationship.** The `evidence_basis` UUID array links every relationship to the raw evidence that supports it, enabling evidence-chain traversal.

### What this constrains

- **Deep graph traversal performance.** PostgreSQL is not a native graph database. Traversals more than 2–3 hops deep (e.g., Signal → Corroboration Group → Cluster → Technology Profile → Solution Candidate) require recursive CTEs (`WITH RECURSIVE`) or multiple joins, which do not scale to arbitrary depth without careful query design and materialized intermediate views. Queries that need to walk the full evidence chain for a complex inference may be slow without optimization.
- **Write discipline.** Every relationship creation requires a confidence score, a created_by, and (for `contradicts`) a rationale. Bulk imports from enrichment providers must be adapted to supply these values or be rejected. There is no "lightweight" path to add a relationship.
- **Governance overhead.** The relationship type taxonomy must be maintained. Adding a new type is a conscious act, not a free-form string. This is a feature, but it creates a process requirement.
- **Index maintenance cost.** The GIN index on `evidence_basis` and the partial indexes on `superseded_at IS NULL` add write overhead. At high ingestion volumes, index maintenance will become a tuning concern.

---

## Decision Log

### Competing hypotheses and forced resolution

The decision to allow competing unresolved relationships (e.g., two `classified_as` relationships for the same entity pointing at different Technology Profiles) was debated. The alternative — requiring resolution before a second classification can be added — was rejected because it would suppress valid analytical uncertainty and create artificial pressure to close disputes before they are genuinely resolved.

### evidence_basis as UUID[] vs. a junction table

Storing evidence UUIDs as a PostgreSQL array rather than a normalized junction table was chosen for query simplicity and write atomicity. The trade-off is that referential integrity on individual evidence UUIDs is not enforced by the database. Application-level validation is required. If referential integrity becomes a correctness problem in practice, migration to a junction table is straightforward.

---

## DECISION NEEDED Items

> **DECISION NEEDED —** Who has authority to add new relationship types to the taxonomy, and what is the process? Options: (a) any engineer via PR with a brief written justification, (b) platform architect sign-off required, (c) a formal governance committee with analyst representation. *Recommendation: Option (b) for v0.1 — single platform architect sign-off via PR review, with a lightweight template (type name, direction, description, example use case). Escalate to option (c) if the platform is adopted by multiple teams with divergent analysis workflows.*

> **DECISION NEEDED —** Should the platform maintain one or more materialized views for common multi-hop traversal paths (e.g., Signal → Cluster → Profile, or the full evidence chain for a Solution Candidate)? A materialized view would dramatically improve read performance for the dashboard and advisory generation pipelines but adds refresh complexity and a staleness window. *Recommendation: Defer materialized views until query performance on real data is measured. Instrument traversal query latency from day one. If p95 latency for a 3-hop traversal exceeds 500ms at the expected data volume, create a materialized view with a 5-minute refresh cycle as the first optimization step.*

> **DECISION NEEDED —** How should bulk relationship imports from enrichment providers be handled when the provider cannot supply confidence scores or rationale? Options: (a) reject the import and require the provider to conform, (b) assign a default low confidence (e.g., 0.3) and a synthetic rationale ("bulk import from [provider] on [date]"), (c) import into a staging table for analyst review before promotion to `object_relationships`. *Recommendation: Option (c) for sources where data quality is unknown, option (b) for established providers with a track record. Never option (a) as a hard gate — it creates an integration barrier that will result in valuable signals being discarded. The staging table approach gives analysts visibility into what is waiting for review without blocking ingestion.*

---

## Related Documents

- `ADR-005-match-set-comparative-classification.md` — defines how match sets produce `classified_as` and `evaluated_by` relationships that are mirrored into this table
- `TS-02-canonical-data-model.md` — the full entity and table definitions, of which `object_relationships` is one table
- `TS-04-scoring-pipeline.md` — how confidence scores on relationships feed into entity-level evidence_strength calculations
- Platform Principles P21 (disconfirmation as first-class signal) and P07 (immutable provenance)
