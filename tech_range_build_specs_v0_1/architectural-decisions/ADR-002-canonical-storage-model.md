# ADR-002: Canonical Storage Model

**Date:** 2026-07-06
**Status:** Accepted
**Deciders:** Tech Range Platform Team
**Supersedes:** —
**Superseded by:** —

---

## Context

Tech Range is a signal-first technology intelligence platform designed for a CTO/defense-sector function. Its data model (defined in TS-02) encompasses 17 entity types:

> Source, Signal, Enrichment, Corroboration Group, Cluster, Neighborhood, Emerging, Domain Area, Assignment/Match, Observation, Hypothesis, Technology Profile, Tech Evaluation, Solution Candidate, Prototype, Finding, Advisory

These entities do not exist in isolation. The platform's core analytical value comes from navigating and auditing the *relationships* between them — tracing how a raw signal corroborates an emerging technology cluster, how that cluster informs a Domain Area, and how a Hypothesis was ultimately elevated to a Finding. This requires persistence that is:

1. **Relational and typed.** Relationships between any two entities carry structured metadata: relationship type, confidence score (numeric, 3,2 precision, range 0.0–1.0), rationale, evidence basis, creator identity, timestamp, and review status. These are not loose associations; they are first-class data that must be queryable and auditable.

2. **Temporally provenance-aware.** Every entity and relationship record must carry a full temporal spine: `event_time` (when the real-world event occurred), `captured_at` (when the system ingested it), `valid_from` / `superseded_at` (bi-temporal validity). Signal archaeology — retrospective analysis of what was known, when, and on what basis — is a non-negotiable platform capability.

3. **Immutably historical.** Records are superseded, not deleted. The full state of any entity or relationship at any point in time must be reconstructable. This precludes storage patterns that overwrite in place.

4. **Sensitive-content safe.** Certain entities reference classified or sensitive content that must not be stored inline. These are represented via a `sensitive_ref_id` pointer pattern, keeping the reference in the canonical store while the content lives in a separate, access-controlled system.

5. **Analytically comparative.** Confidence axes, classification tags, technology maturity assessments, and corroboration scores must support range queries, aggregation, sorting, and cross-entity comparison — not just key-value lookup.

The question this ADR addresses: **what persistent storage technology should serve as the canonical data store for Tech Range entity and relationship data?**

---

## Decision

**PostgreSQL (relational, schema-enforced) is the canonical storage model for all Tech Range entity and relationship data.**

The entity schema is defined in PostgreSQL with typed tables per entity class. Cross-entity relationships are managed through a single generic `object_relationships` table — effectively a typed, attributed edge store implemented in relational terms. Confidence scores are stored as `NUMERIC(3,2)`. Temporal provenance columns (`event_time`, `captured_at`, `valid_from`, `superseded_at`) are mandatory on all entity and relationship tables.

---

## Rationale

### Why relational wins for this use case

**ACID guarantees protect analytical integrity.** Tech Range entities are not write-once logs; they are mutable analytical artifacts. An Observation is escalated to a Hypothesis; a Hypothesis is refined with new corroboration; a Signal is superseded when a better source arrives. These multi-step state transitions must be atomic — partial writes that leave a Hypothesis promoted but its predecessor not superseded would corrupt the analytical record. PostgreSQL's ACID transactions make this safe by default. Graph databases and document stores either lack or significantly complicate full ACID support across multi-node writes.

**The `object_relationships` table is a graph-in-relational pattern — and it is sufficient.** A dedicated graph database (e.g., Neo4j) would offer native traversal syntax and index-free adjacency. But the traversal depth in Tech Range is bounded: the longest meaningful path (Signal → Enrichment → Corroboration Group → Cluster → Emerging → Domain Area → Finding) is approximately six hops. At this depth, recursive CTEs (`WITH RECURSIVE`) in PostgreSQL perform competitively with graph engines, without introducing a second operational system. The relationship table carries full edge metadata (type, confidence, rationale, review status) as typed columns — something a property graph can do but requires careful schema discipline to enforce. In PostgreSQL, that discipline is structural.

**Generated columns and partitioning are available when needed.** Confidence aggregation, maturity score derivation, and temporal window queries can be accelerated with PostgreSQL generated columns, partial indexes, and declarative table partitioning — all without leaving the primary store. These are mature, well-documented features with no equivalent operational simplicity in graph or document alternatives.

**Sensitive data pointer pattern is cleanly expressible.** The `sensitive_ref_id` column pattern — a nullable foreign reference that exists in the canonical schema but points to an out-of-band secure store — is straightforward in relational DDL, easy to enforce via constraints, and trivial to audit via query. Document stores make this pattern workable but do not enforce referential integrity natively. Graph databases have no native concept of deferred external reference.

**Operational simplicity matters at this scale.** Tech Range is a platform built and operated by a small team with a defense-sector tempo. A single well-understood database engine — with mature backup, replication, access control, and monitoring tooling — reduces operational surface area. Introducing a graph database as a co-primary store would double the operational burden for a traversal benefit that can be achieved within PostgreSQL.

---

## Alternatives Considered

### Graph Database (e.g., Neo4j, Amazon Neptune)

**Strengths:** Native traversal syntax (Cypher / Gremlin) is expressive for multi-hop relationship queries. Index-free adjacency provides consistent traversal performance regardless of graph size. Property graphs natively model typed, attributed edges.

**Weaknesses:**
- ACID guarantees across complex write patterns (especially multi-entity state transitions) are weaker or require careful configuration in most graph engines.
- Operational complexity is higher: separate backup strategy, separate access control model, separate monitoring, separate query tooling.
- Temporal bi-temporality (valid_from / superseded_at) is not a native concept in any major graph engine; it must be layered on manually, reducing the ergonomic advantage.
- At six-hop maximum traversal depth with typed edge metadata, the traversal performance advantage over PostgreSQL recursive CTEs is marginal for this workload.
- **Not selected.**

### Document Database (e.g., MongoDB, CouchDB)

**Strengths:** Flexible schema accommodates heterogeneous entity subtypes without migrations. Horizontal scaling is straightforward. JSON-native representation aligns with API response shapes.

**Weaknesses:**
- Referential integrity between documents is not enforced by the engine. Cross-entity relationships, confidence scores, and temporal provenance are all application-layer concerns with no database-level guarantee — a significant risk for an analytical platform where data correctness is the product.
- Range queries on NUMERIC confidence scores and temporal window queries are possible but not the document model's strength.
- Immutable history and bi-temporal provenance require explicit application-level discipline that relational schemas encode structurally.
- **Not selected.**

### Hybrid: Graph + Relational (write to PostgreSQL, project graph into Neo4j for traversal)

**Strengths:** Preserves ACID on writes; enables native graph traversal for complex queries.

**Weaknesses:**
- Introduces a synchronization problem: the graph projection must stay consistent with the canonical relational store, adding a failure mode and operational complexity.
- At current traversal depth (≤6 hops), the query performance benefit does not justify the synchronization overhead.
- Schema changes in PostgreSQL must propagate to the graph projection — a coordination tax on every migration.
- This option remains viable if traversal workloads grow significantly in a future phase. See "What's Deferred" below.
- **Not selected for this phase.**

---

## Consequences

### What this enables

- Full bi-temporal provenance queries over any entity or relationship using standard SQL window functions and CTEs.
- Signal archaeology: reconstruct the state of any analytical conclusion at any past point in time.
- Confidence score range queries, aggregations, and cross-entity comparisons using typed `NUMERIC(3,2)` columns with standard SQL.
- Referential integrity enforcement across all 17 entity types and the `object_relationships` edge table via foreign key constraints.
- Atomic multi-entity state transitions (e.g., promoting a Hypothesis to a Finding while superseding prior versions) within a single transaction.
- Sensitive data pointer pattern enforced structurally via nullable FK columns and query-level access control.
- Single operational system for backup, replication, monitoring, and access control.

### What this constrains

- Deep graph traversal (>10 hops) will require recursive CTEs or application-level traversal logic rather than a native graph query language. If the entity graph grows significantly denser, this may become a performance bottleneck.
- Schema changes require migrations. The flexibility advantage of a document store (add a field without a migration) does not apply here. All entity type changes are DDL events.
- Full-text and vector similarity search (e.g., embedding-based Signal retrieval) are possible via PostgreSQL extensions (`pg_trgm`, `pgvector`) but are not native relational operations. These use cases require extension management.

---

## What's Deferred

- **Graph query optimization.** If traversal query latency becomes a bottleneck as the entity graph grows, options include: adding a graph extension to PostgreSQL (Apache AGE), introducing a read-side graph projection (Neo4j / Neptune), or tuning recursive CTE indexes. This decision is deferred until query profiling reveals an actual bottleneck.
- **Read replica for analytics.** High-volume analytical queries (e.g., bulk confidence score aggregation, cross-cluster comparison, retrospective provenance scans) may eventually contend with OLTP write traffic. A read replica strategy — either PostgreSQL streaming replication or a dedicated analytical store (e.g., Redshift, DuckDB over exported snapshots) — is deferred until traffic patterns justify it.
- **Partitioning strategy.** The `object_relationships` table and high-volume entity tables (Signal, Enrichment) are candidates for range partitioning on `captured_at` or `valid_from`. The partitioning DDL is deferred until table size warrants it, but the schema should be designed to accommodate future partitioning without breaking changes.

---

## Decisions Needed

> **DECISION NEEDED —** Should PostgreSQL be extended with Apache AGE or pgvector to support graph traversal and/or vector similarity queries within the canonical store, or should these remain out-of-scope for v0.1? *Recommendation: Defer both extensions to v0.2. Apache AGE adds traversal ergonomics but the current six-hop maximum does not justify the operational overhead of managing a Postgres extension with its own release cadence. pgvector is more compelling near-term if Signal similarity retrieval is a v0.1 feature — evaluate against the Signal ingestion roadmap before locking the v0.1 schema.*

> **DECISION NEEDED —** What is the partitioning trigger threshold for the `object_relationships` and `signals` tables — should a row count target (e.g., 10M rows) or a calendar schedule drive the first partitioning migration? *Recommendation: Use a row count trigger (10M rows in either table) rather than a calendar schedule. This keeps partitioning a data-driven decision and avoids premature schema complexity. Instrument row counts in the operational dashboard from day one so the threshold is visible.*

> **DECISION NEEDED —** Should a read replica be provisioned at launch to isolate analytical query load, or added reactively when OLTP contention is observed? *Recommendation: Reactive addition. At v0.1 scale, a read replica adds cost and synchronization operational overhead without a demonstrated benefit. Provision the primary with sufficient IOPS and connection pooling (PgBouncer) to handle mixed workloads, and instrument slow query logs from launch so the signal for adding a replica is unambiguous.*
