# ADR-015: Temporal Provenance Standard

**Status:** Accepted
**Date:** 2026-07-06
**Deciders:** Tech Range Platform Team
**Related:** ADR-005 (Immutable-but-Supersedable Pattern), TS-02 (Temporal Schema Spec)

---

## Context

Most database schemas track a single timestamp: `created_at`. This is sufficient when the only question is "when was this record written?" It is not sufficient when the record represents something that happened in the world at a specific moment, was noticed by a source at a different moment, reached the platform's pipeline at yet another moment, and was classified by an analyst at a fourth moment.

Tech Range is a signal-first technology intelligence platform. The platform's core value proposition is not just cataloguing signals — it is understanding the shape of emerging technology landscapes over time, including the shape of what *was not yet known* at any given point. That requires knowing not just that a signal exists, but when each actor in the knowledge chain encountered it.

Consider a concrete example: a research group publishes a paper in January. The paper describes work that began eighteen months earlier. A preprint server indexes it in February. Tech Range's ingestion pipeline captures it in April. An analyst classifies it in May. A customer views the resulting Advisory in June.

These are five distinct temporal facts. The gap between January (event_time) and April (ingested_at) is itself an intelligence finding: whatever technology direction that paper represents was advancing for months before any signal reached the platform. The gap between February (captured_at) and April (ingested_at) tells a different story — pipeline coverage or source latency. The gap between April and May tells yet another — analyst throughput or triage priority.

If the platform stores only `created_at` (when the DB record was written in May), all of this information is lost. The platform cannot answer "what did we know in February?" It cannot detect that this signal arrived late. It cannot support retrospective surprise analysis — the post-hoc question of why a technology direction wasn't flagged sooner.

Temporal provenance — the full chain of *when* at each stage — is therefore not a nice-to-have audit feature. It is architecturally load-bearing. It enables:

1. **Point-in-time reconstruction:** Reproduce the platform's knowledge state as of any past date.
2. **Signal archaeology:** Query across the gap between world-time and platform-time to surface systematic blind spots.
3. **Surprise analysis:** When a technology trend becomes undeniable, work backwards through the timestamp chain to understand when early signals existed vs. when they were captured vs. when they were classified.
4. **Source quality assessment:** Compare `captured_at` across sources covering the same domain to identify which sources surface signals earliest.
5. **Pipeline health monitoring:** Track `ingested_at - captured_at` distributions to detect crawl staleness or source coverage gaps.

The platform also includes classifiable records — match_sets, state_transitions, and key relationship records — whose meaning changes over time as new signals arrive and analysts revise classifications. These records require a separate bi-temporal validity mechanism (valid_from / superseded_at) that is independent of the ingestion timestamps. This pattern is defined in ADR-005 and TS-02; this ADR establishes how it integrates with the broader timestamp standard.

---

## Decision

All entity tables in Tech Range adopt a **five-timestamp standard**. Classifiable records additionally carry **bi-temporal validity columns**. No exceptions without explicit ADR amendment.

### The Five Timestamps

These five columns appear on every entity table. Their semantics are fixed and must not be reused for other purposes.

---

**`event_time TIMESTAMPTZ NULL`**

*World-time: when the thing actually happened.*

This is the timestamp of the underlying real-world event — the date a paper was submitted, a product was announced, a patent was filed, a regulatory decision was issued. It is the timestamp most directly relevant to intelligence analysis.

`event_time` is **nullable** because it is not always knowable. A crawled webpage may describe something without dating it. An inferred signal may have no clear event anchor. When `event_time` is unknown, the column is NULL and the record carries an `event_time_unknown` flag (see Implementation Requirements). When `event_time` is estimated rather than confirmed from source metadata, the record carries an `event_time_estimated` flag.

This column may precede the existence of the platform by years or decades. It must never be defaulted to the current timestamp. Setting it to `NOW()` when the actual event time is unknown is a data corruption — it destroys the ability to detect late signals.

---

**`captured_at TIMESTAMPTZ NOT NULL`**

*Source publication time: when the source published, indexed, or released the signal.*

This is the timestamp at which the source that Tech Range ingests from made the signal available — the publication date on an arXiv paper, the timestamp of a news article, the date a patent was granted, the commit timestamp in a public repository.

`captured_at` is **NOT NULL** because the platform only ingests from sources, and every source either carries a publication timestamp or can be assigned one at crawl time as a conservative fallback (the crawl timestamp, which represents the latest possible captured_at). Ingestion pipelines are responsible for extracting this value from source metadata.

The gap `captured_at - event_time` measures **source latency**: how long after the world event the source noticed it. This is a property of the source, not of the platform.

---

**`ingested_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`**

*Pipeline receipt time: when Tech Range's ingestion pipeline first received and stored the raw signal.*

This timestamp is set by the ingestion pipeline at the moment the raw record is written to the staging layer. It is a DB-side default and must never be supplied by application code or set to any value other than the actual time of ingestion.

The gap `ingested_at - captured_at` measures **pipeline latency**: how current the platform's crawl coverage is for a given source. A persistent large gap indicates a source that is crawled infrequently or a feed that is slow to index.

The gap `ingested_at - event_time` is the **total signal latency**: how long after the world event the platform received any trace of it. This is the primary input to late-signal detection.

---

**`created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`**

*DB record creation time: when the record was written to the production database.*

This is the standard audit timestamp present in most systems. In Tech Range it has a specific, narrow meaning: when did the platform's processing pipeline write this particular row. It will typically be close to `ingested_at` for raw signals, but may diverge for derived records (entity extractions, match_set members) that are created during post-ingestion processing.

`created_at` is always DB-side and must never be supplied by client code.

---

**`valid_from TIMESTAMPTZ NOT NULL DEFAULT NOW()`**

*Classification validity start: when this version of the record's classification or relationship became authoritative.*

For classifiable records (see Supersedable Records below), `valid_from` marks the beginning of the window during which this record's classification is considered the current best understanding. When an analyst revises a classification, the existing record is not updated — it is superseded. A new record is inserted with a new `valid_from` and the old record receives a `superseded_at` timestamp. This preserves full history.

For non-classifiable records (raw signals, source metadata), `valid_from` is set at insert time and is effectively equal to `created_at`. These records are never superseded.

---

### Bi-Temporal Validity Columns (Classifiable Records Only)

In addition to the five base timestamps, classifiable records carry:

**`superseded_at TIMESTAMPTZ NULL`**

NULL means this record is the current authoritative version. A non-NULL value means this record has been replaced by a newer version. Queries for current state always filter `WHERE superseded_at IS NULL`. Historical queries use the `valid_from / superseded_at` window to reconstruct past states.

The combination of `valid_from` and `superseded_at` enables bi-temporal querying: for any point in time T, the authoritative record is the one where `valid_from <= T AND (superseded_at IS NULL OR superseded_at > T)`.

---

## Supersedable Records

The following record types carry `valid_from` + `superseded_at` and follow the immutable-but-supersedable pattern:

| Record Type | Rationale |
|---|---|
| `match_sets` | A match_set represents the platform's current best grouping of signals around a technology concept. As new signals arrive and analyst understanding evolves, the composition of the match_set changes. Immutable history of past groupings is required for surprise analysis. |
| `state_transitions` | Analyst-assigned state transitions (e.g., "Emerging → Accelerating") carry analytical judgement that may be revised. The history of when transitions were assigned is intelligence-relevant. |
| `entity_relationships` | Key relationships between entities (technology → actor, technology → domain) that are inferred or analyst-confirmed and may be revised as understanding improves. |
| `signal_classifications` | The assignment of a raw signal to a technology concept or theme. Reclassification must not destroy the prior classification history. |

The following record types do **not** carry `superseded_at` and are treated as append-only immutable:

| Record Type | Rationale |
|---|---|
| `raw_signals` | Raw ingested content is an immutable fact of receipt. It is never revised. |
| `sources` | Source metadata is updated in-place with standard `updated_at`; source identity does not require versioned history. |
| `advisories` | Published advisories carry `published_at` and revision tracking via a separate `advisory_versions` table; they do not use the supersedable pattern. |
| `entities` | Base entity records (technologies, actors, domains) are updated in-place; their relationship records (above) carry the versioned history. |

---

## Signal Archaeology

The timestamp combination enables a class of retrospective queries that are not possible in systems with only `created_at`. These queries are collectively called **signal archaeology**.

### Point-in-Time Reconstruction

Any query can be scoped to a past knowledge state by filtering on `ingested_at`:

```sql
-- What signals had the platform received as of 2025-01-01?
SELECT * FROM raw_signals
WHERE ingested_at < '2025-01-01';

-- What was the authoritative match_set state on 2025-01-01?
SELECT * FROM match_sets
WHERE valid_from <= '2025-01-01'
  AND (superseded_at IS NULL OR superseded_at > '2025-01-01');
```

### Late-Signal Detection

Signals where the gap between `event_time` and `ingested_at` exceeds the configured threshold are flagged as late-arriving. These are surfaced in the signal archaeology queue for analyst review.

```sql
-- Signals that arrived more than 30 days after the world event
SELECT
  id,
  event_time,
  captured_at,
  ingested_at,
  ingested_at - event_time AS total_latency,
  ingested_at - captured_at AS pipeline_latency
FROM raw_signals
WHERE event_time IS NOT NULL
  AND ingested_at - event_time > INTERVAL '30 days'
ORDER BY total_latency DESC;
```

The `ingested_at - captured_at` component of this gap is attributable to the platform's pipeline. The `captured_at - event_time` component is attributable to the source. Both are separately useful for different monitoring purposes.

### Surprise Analysis

When a technology trajectory becomes undeniable (e.g., a major announcement, a regulatory trigger, a competitive launch), the platform can run retrospective analysis to understand when signals existed:

```sql
-- For a given technology concept, show the distribution of signal arrival
-- relative to event time — how long were signals "out there" before being ingested?
SELECT
  date_trunc('month', event_time) AS event_month,
  date_trunc('month', ingested_at) AS ingested_month,
  COUNT(*) AS signal_count,
  AVG(EXTRACT(EPOCH FROM (ingested_at - event_time)) / 86400) AS avg_latency_days
FROM raw_signals rs
JOIN signal_classifications sc ON rs.id = sc.signal_id
WHERE sc.technology_id = $1
  AND event_time IS NOT NULL
GROUP BY 1, 2
ORDER BY 1, 2;
```

This query reveals whether signals were trickling in for months before classification (a triage/prioritization problem), or whether the signals themselves only surfaced late (a source coverage problem), or whether the gap occurred at the `captured_at → ingested_at` stage (a pipeline latency problem). Each root cause has a different remediation.

### Source Quality Benchmarking

```sql
-- Which sources surface signals earliest (smallest captured_at - event_time)?
SELECT
  source_id,
  AVG(EXTRACT(EPOCH FROM (captured_at - event_time)) / 86400) AS avg_source_latency_days,
  COUNT(*) AS signal_count
FROM raw_signals
WHERE event_time IS NOT NULL
GROUP BY source_id
HAVING COUNT(*) > 50
ORDER BY avg_source_latency_days ASC;
```

---

## Late-Signal Detection

A signal is classified as **late-arriving** when the gap between `event_time` and `ingested_at` exceeds a configured threshold. Late-arriving signals are intelligence-significant: they indicate the platform (or its sources) missed an event for a meaningful period.

When a late signal is detected:

1. The record receives a `late_signal` boolean flag set to `true` at ingestion time.
2. The record is added to the **signal archaeology queue** — a review-oriented view surfaced in the analyst interface.
3. The record's source latency component (`captured_at - event_time`) and pipeline latency component (`ingested_at - captured_at`) are logged separately to the source quality metrics table.
4. If the signal belongs to a technology concept currently in an active match_set, the match_set is flagged for re-evaluation: the late signal may alter the trajectory assessment.

Late-signal detection runs at ingestion time. It requires `event_time` to be non-NULL and requires the threshold to be configured per signal category (see DECISION NEEDED items below).

Signals with NULL `event_time` cannot be evaluated for lateness and are excluded from archaeology queue routing. This is an intentional design choice: unknown event time is not the same as on-time arrival, and conflating the two would corrupt the archaeology queue signal.

---

## Implementation Requirements

### NOT NULL Constraints

| Column | Constraint | Reason |
|---|---|---|
| `captured_at` | NOT NULL | Every ingested signal has a source publication time; fallback to crawl time if not in metadata |
| `ingested_at` | NOT NULL | Set by pipeline at insert time |
| `created_at` | NOT NULL | Standard DB audit column |
| `valid_from` | NOT NULL | Set at insert time |
| `superseded_at` | NULL allowed | NULL = current record; non-NULL = superseded |
| `event_time` | NULL allowed | Not always knowable; see annotation requirements |

### Nullable `event_time` Annotation

When `event_time` is NULL, the reason must be recorded. Every table with `event_time` carries two companion columns:

- `event_time_unknown BOOLEAN NOT NULL DEFAULT FALSE` — set to TRUE when the event time cannot be determined from any source
- `event_time_estimated BOOLEAN NOT NULL DEFAULT FALSE` — set to TRUE when the event time is inferred/estimated rather than confirmed from source metadata

These flags are mutually exclusive: a value cannot be both unknown and estimated. Constraints enforce this.

```sql
CONSTRAINT event_time_flag_mutex CHECK (
  NOT (event_time_unknown AND event_time_estimated)
),
CONSTRAINT event_time_unknown_requires_null CHECK (
  NOT event_time_unknown OR event_time IS NULL
),
CONSTRAINT event_time_estimated_requires_value CHECK (
  NOT event_time_estimated OR event_time IS NOT NULL
)
```

### No Client-Side Timestamp Generation

All timestamps with `DEFAULT NOW()` are set by the database. Application code and ingestion pipelines must never pass these values explicitly unless overriding for a documented reason (e.g., bulk historical backfill). Migrations that alter timestamp columns must include explicit review of default behavior.

The one exception: `captured_at` and `event_time` are sourced from external metadata and are always provided by the pipeline, never defaulted by the DB. Pipelines are responsible for extracting these from source metadata accurately.

### TIMESTAMPTZ Everywhere

All timestamp columns use `TIMESTAMPTZ` (timestamp with time zone). `TIMESTAMP` without timezone is never used for any column in this standard. This is non-negotiable: timezone-naive timestamps cannot be safely combined across data sources and break point-in-time reconstruction queries when sources span time zones.

All TIMESTAMPTZ values are stored and compared in UTC. Source metadata in non-UTC timezones is converted at ingestion time.

### Indexing

Every table in the five-timestamp standard carries the following baseline indexes:

```sql
CREATE INDEX ON table_name (ingested_at);
CREATE INDEX ON table_name (event_time) WHERE event_time IS NOT NULL;
CREATE INDEX ON table_name (valid_from);
-- For supersedable records only:
CREATE INDEX ON table_name (superseded_at) WHERE superseded_at IS NULL;
```

The partial index on `superseded_at IS NULL` is critical for current-state queries on high-volume supersedable tables.

### Reviewed and Published Timestamps

Two additional domain-specific timestamps appear on specific record types but are not part of the five-timestamp standard:

- `reviewed_at TIMESTAMPTZ NULL` — on classifiable records, set when an analyst reviews/approves. NULL = not yet reviewed.
- `published_at TIMESTAMPTZ NULL` — on Advisories, set when the advisory is published externally. NULL = draft.

These are domain columns, not provenance columns. They do not replace any of the five standard timestamps.

---

## Consequences

### Positive

- The platform can reconstruct its knowledge state at any past point in time, enabling credible retrospective analysis.
- Late-arriving signals are detected automatically and surfaced for analyst attention rather than silently entering the corpus.
- Source quality can be measured objectively and continuously, enabling data-driven decisions about source prioritization and crawl frequency.
- Surprise analysis becomes a structured, queryable capability rather than an ad-hoc forensic exercise.
- The five-timestamp standard creates a consistent mental model across the entire schema; any engineer can reason about provenance without table-specific knowledge.

### Negative

- Five timestamp columns per table increases storage overhead, though at typical signal volumes this is negligible relative to content storage.
- Ingestion pipelines must reliably extract `captured_at` and `event_time` from source metadata. Sources with poor metadata quality degrade the archaeology capability. This requires ongoing source quality investment.
- The `event_time_unknown` / `event_time_estimated` annotation requirement adds pipeline complexity. Pipelines must make an explicit determination for every signal rather than silently leaving event_time NULL.
- Developers unfamiliar with the standard may be confused by having five timestamps to reason about. The standard must be clearly documented in onboarding materials.

### Risks

- If `event_time` is systematically populated with `ingested_at` as a fallback (a tempting shortcut), late-signal detection becomes blind. This must be enforced at code review and caught by data quality monitoring.
- The bi-temporal pattern on supersedable records adds query complexity. All queries against supersedable tables must explicitly handle the `superseded_at IS NULL` filter or they will return duplicate/stale records. ORM layers and query builders must be configured to enforce this by default.

---

## DECISION NEEDED Items

> **DECISION NEEDED —** What constitutes a "late" signal, and should the threshold vary by signal category? The decision affects how noisy the archaeology queue is and how sensitive the system is to pipeline gaps. A single global threshold (e.g., 30 days) is simple but may be too coarse: a 30-day-old academic preprint is mildly late, while a 30-day-old regulatory filing in a fast-moving domain may be critically late. *Recommendation: Define per-category thresholds with defaults: 30 days for academic/research signals, 14 days for regulatory/policy signals, 7 days for news/industry signals, 3 days for competitive intelligence signals. Store thresholds in a configuration table so they can be tuned without a schema migration. Implement a single-threshold MVP (30 days) and refine based on archaeology queue volume in the first 60 days of operation.*

> **DECISION NEEDED —** When `event_time` is unknown, should the platform attempt to estimate it (e.g., from contextual clues in the signal content, from the publication date of a citing source, from domain heuristics) and store the estimate with the `event_time_estimated` flag, or should it leave `event_time` NULL and accept that those signals are excluded from archaeology queries? Estimation improves coverage but risks introducing systematic bias if estimation heuristics are inaccurate. *Recommendation: Permit estimation only when a documented heuristic exists and the confidence can be bounded (e.g., "event occurred within the 12 months prior to captured_at based on tense analysis"). Require that estimated event_times carry an `event_time_estimate_method` VARCHAR column recording which heuristic was used, enabling retrospective audit and correction. Do not estimate by default; estimation must be an explicit pipeline capability opt-in, not a fallback.*

> **DECISION NEEDED —** What is the retention policy for superseded records? Superseded records are immutable history required for point-in-time reconstruction and surprise analysis, but they accumulate indefinitely for high-churn match_sets. Options include: (a) retain all superseded records indefinitely, (b) retain superseded records for a rolling N-year window, (c) retain all superseded records but archive them to cold storage after M months. *Recommendation: Retain all superseded records indefinitely in hot storage for the first two years of platform operation to preserve the full archaeology capability while the platform's usage patterns are being established. Revisit with actual volume data before implementing any archival policy. Do not implement time-based deletion of superseded records: once a historical state is lost, surprise analysis for that period becomes impossible, and the intelligence value of that capability is likely to be discovered only after the data has been deleted.*

---

*This ADR is the authoritative definition of the temporal provenance standard. Any deviation in schema design requires an explicit amendment or a new ADR that supersedes this one.*
