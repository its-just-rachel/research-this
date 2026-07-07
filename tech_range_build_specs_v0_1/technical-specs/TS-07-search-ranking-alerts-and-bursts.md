# TS-07 — Search, Ranking, Alerts, and Burst Detection

**Version:** 0.1
**Status:** Draft
**Last updated:** 2026-07-06

---

## 1. Purpose

This spec defines the search, alert, and burst detection subsystems for Tech Range. Analysts need to find relevant signals, profiles, and evidence quickly; leadership needs to be alerted when something significant happens.

Three concerns are addressed together because they share underlying query infrastructure:

1. **Search** — finding objects in the platform by keyword and structured filter
2. **Alerts and watchlists** — proactive notification when new signals, state changes, or bursts match an analyst's criteria
3. **Burst detection** — identifying when signal volume is accelerating for a Cluster or Neighborhood, which is a key early-warning mechanism

This spec depends on: TS-02 (Clusters), TS-03 (Technology Profiles), TS-04 (Signals), ADR-002 (vector search decision), ADR-011 (extension services), ADR-015 (signal archaeology / temporal queries).

---

## 2. Search

### 2.1 Full-text search

**What is searchable**

Full-text search covers the following content across the platform:

| Object | Columns indexed |
|---|---|
| Signal | `summary`, `observation_text` |
| Technology Profile | `name`, `description` |
| Hypothesis | `hypothesis_text` |
| Finding | `finding_text` |
| Advisory | `advisory_text`, `title` |

**PostgreSQL tsvector approach**

Each indexed table carries a generated `search_vector` column of type `tsvector`. Columns are weighted to reflect relevance hierarchy:

- `'A'` weight — object name / title (highest)
- `'B'` weight — short summary / description
- `'C'` weight — body text (observation, hypothesis, finding, advisory body)

Example for Technology Profiles:

```sql
ALTER TABLE tech_profiles
  ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'B')
  ) STORED;

CREATE INDEX idx_tech_profiles_search ON tech_profiles USING GIN(search_vector);
```

For tables where body text is long or updated frequently, the `search_vector` column is maintained by a trigger rather than a generated column, to avoid recomputing on every write.

> **DECISION NEEDED —** Full-text search index update strategy: trigger-based (immediate, but adds write latency on every signal insert/update) vs. background refresh job (slight staleness, no write latency penalty). *Recommendation: background job with < 5 minute lag; this is acceptable for intelligence workloads where near-real-time indexing is sufficient and write throughput matters more than instant search visibility.*

Language configuration: `english` dictionary for all tsvector columns in MVP. Multi-language support is deferred.

**Search query interface**

Analyst keyword input is translated to a `tsquery` expression before execution:

- Single term: `plainto_tsquery('english', 'hypersonics')` → simple AND of lexemes
- Multi-term: treated as AND by default (`plainto_tsquery`)
- Explicit OR: `websearch_to_tsquery('english', 'hypersonics OR directed energy')` supports OR and phrase syntax
- Phrase search: `phraseto_tsquery('english', 'counter-UAS')` matches exact phrase proximity

The API layer accepts a `search_mode` parameter (`plain`, `websearch`, `phrase`) and routes accordingly. `websearch` is the default for analyst-facing search.

**Ranking**

Results are ranked using `ts_rank_cd`, which accounts for document length and cover density (proximity of matching terms):

```sql
ts_rank_cd(search_vector, query, 32) AS rank
```

The normalization flag `32` divides rank by the number of unique words in the document, preventing very long documents from dominating. Final sort order blends `rank` with recency and state signals — see Section 2.4.

---

### 2.2 Structured filters

**Filterable dimensions**

All filterable dimensions are available alongside keyword search or independently:

| Dimension | Field | Type |
|---|---|---|
| Neighborhood | `neighborhood_id` | UUID, multi-select |
| Domain Area | `domain_area` | enum, multi-select |
| Signal type | `signal_type` | enum, multi-select |
| Object state | `signal_state` / `current_state` | enum, multi-select — signals use `signal_state`; tech profiles, clusters, and other objects use `current_state` |
| Fidelity level | `fidelity_level` | integer range (1–5) |
| Availability level | `availability_level` | integer range (1–5) |
| Event time | `event_time` | date range |
| Ingested at | `ingested_at` | date range |
| Created at | `created_at` | date range |
| Source credibility | `source_credibility` | float range (0.0–1.0) |
| Confidence | `confidence` | float range (0.0–1.0) |
| Ingestion mode | `ingestion_mode` | enum (`manual`, `automated`, `import`) |

**Filter combination logic**

- Filters across dimensions are **ANDed** (a signal must match all active filters)
- Multiple values selected within a single dimension are **ORed** (a signal matching any selected Neighborhood satisfies the Neighborhood filter)
- Keyword search and structured filters are ANDed together

**Filter persistence — watchlists**

Analysts can save a named filter set as a watchlist. Saved watchlists persist across sessions and can optionally carry alert conditions (see Section 3). The filter criteria are stored as JSONB and re-executed on demand.

---

### 2.3 Semantic similarity search (deferred)

Vector similarity search using `pgvector` is deferred to Phase 2 per ADR-002.

When implemented, embeddings will be stored on `signals` and `tech_profiles` (a `embedding vector(1536)` column or equivalent). Similarity search will complement — not replace — keyword search, surfacing conceptually related signals even when exact terms don't match.

> **DECISION NEEDED —** ADR-002 has not yet resolved the pgvector extension decision. Phase 2 search design is blocked on that decision. *Recommendation: resolve ADR-002 before Phase 2 search design begins; the keyword + filter approach defined here is the complete MVP search capability and does not require vector search to ship.*

For MVP: keyword search + structured filters as defined in Sections 2.1 and 2.2 are the complete search capability.

---

### 2.4 Search result types and ranking

**Result types**

Unified search returns results across four object types, distinguished in the response payload:

- `signal` — individual signals
- `tech_profile` — Technology Profiles
- `cluster` — Clusters (with burst state surfaced)
- `advisory` — published Advisories

**Ranking factors**

Results within a type are ranked by a composite score:

1. `ts_rank_cd` score from full-text match (primary)
2. Object recency — `created_at` or `ingested_at` decays rank for older objects (logarithmic decay, configurable half-life)
3. Object state — `active` and `published` states ranked above `candidate`, `archived`, or `deprecated`
4. Source credibility — for Signals, the average credibility of contributing sources boosts rank

The composite score formula (in SQL pseudocode):

```sql
(ts_rank_cd(search_vector, query, 32) * 0.6)
+ (recency_score(created_at) * 0.2)
+ (state_boost(signal_state /* or current_state — see note */) * 0.1)
+ (avg_source_credibility * 0.1)
AS composite_rank
```

> **Note on `state_boost` field:** for `signal` result types, the state boost reads `signal_state`; for `tech_profile`, `cluster`, and `advisory` result types it reads `current_state`. The `state_boost` function must branch on result type when constructing the composite score.

Weights are configurable via platform settings; the values above are defaults.

**Pagination**

Search results use cursor-based pagination, not offset-based. As new signals arrive continuously, offset pagination produces unstable result sets (results shift between pages). Cursors are encoded from the last-seen `composite_rank` + `id` tuple, ensuring stable pages even as new data arrives.

Page size default: 25 results. Maximum: 100.

---

## 3. Alerts and watchlists

### 3.1 Watchlist model

A watchlist is a named saved search with optional alert conditions. Analysts create watchlists to monitor an area of interest over time.

**Schema**

```sql
CREATE TABLE watchlists (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name                TEXT NOT NULL,
  owner_id            UUID NOT NULL REFERENCES analysts(id),
  filter_criteria     JSONB NOT NULL DEFAULT '{}',
  alert_conditions    JSONB NOT NULL DEFAULT '{}',
  notification_channel TEXT NOT NULL DEFAULT 'in_platform'
                        CHECK (notification_channel IN ('in_platform', 'email', 'both')),
  email_digest_freq   TEXT DEFAULT 'daily'
                        CHECK (email_digest_freq IN ('immediate', 'hourly', 'daily')),
  is_active           BOOLEAN NOT NULL DEFAULT true,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**`filter_criteria` JSONB structure**

```json
{
  "neighborhoods": ["uuid-1", "uuid-2"],
  "domain_areas": ["directed_energy", "hypersonics"],
  "signal_types": ["academic_publication", "patent"],
  "fidelity_range": { "min": 3, "max": 5 },
  "confidence_range": { "min": 0.6, "max": 1.0 },
  "date_range": { "field": "event_time", "from": "2025-01-01", "to": null }
}
```

**`alert_conditions` JSONB structure**

```json
{
  "new_match": true,
  "state_change": { "enabled": true, "watched_ids": ["uuid-a", "uuid-b"] },
  "confidence_change": { "enabled": true, "threshold": 0.15 },
  "burst_detected": { "enabled": true, "neighborhoods": ["uuid-1"] }
}
```

**Alert condition types**

| Condition | Trigger |
|---|---|
| `new_match` | Any new object (Signal, Profile, Cluster) matches the watchlist filter criteria |
| `state_change` | A watched object transitions state (e.g., `candidate` → `active`, `active` → `deprecated`) |
| `confidence_change` | A watched object's confidence score changes by more than the configured threshold |
| `burst_detected` | A Neighborhood or Cluster in the watchlist scope enters `is_bursting = true` |
| `boundary_case_created` | A new boundary case is created on a watched Technology Profile |
| `advisory_published` | A new Advisory is published that overlaps with watchlist scope |

---

### 3.2 Alert delivery

**In-platform notifications (always on)**

Every triggered alert writes to the `alerts` table (Section 3.3) and surfaces in the analyst's in-platform notification queue. This channel cannot be disabled.

**Email digest (configurable)**

Analysts can configure email delivery at three frequencies:

- `immediate` — email sent within 5 minutes of alert creation
- `hourly` — digest of all alerts in the last hour, sent at the top of the hour
- `daily` — digest of all unread alerts, sent at a configurable time (default: 08:00 analyst local time)

Email delivery is implemented via a background job that queries `alerts WHERE read = false AND created_at > last_digest_sent_at` for each analyst with email enabled.

**No external webhooks in MVP**

Webhook delivery to external services is deferred to Phase 2 (TS-09).

> **DECISION NEEDED —** Whether to expose the watchlist and alert API to ADR-011 extension services in Phase 2 (extensions may want to subscribe to alerts programmatically). *Recommendation: yes, expose via TS-09 API in Phase 2; extension services should be able to create watchlists and receive alerts via webhook, which is consistent with the ADR-011 extension model.*

---

### 3.3 Alert schema

```sql
CREATE TABLE alerts (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  watchlist_id      UUID NOT NULL REFERENCES watchlists(id),
  analyst_id        UUID NOT NULL,
  alert_type        TEXT NOT NULL CHECK (alert_type IN (
                      'new_match', 'state_change', 'confidence_change',
                      'burst_detected', 'boundary_case_created',
                      'advisory_published'
                    )),
  object_id         UUID,
  object_type       TEXT,
  message           TEXT NOT NULL,
  read              BOOLEAN NOT NULL DEFAULT false,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_analyst_unread
  ON alerts (analyst_id, read, created_at DESC)
  WHERE read = false;

CREATE INDEX idx_alerts_watchlist
  ON alerts (watchlist_id, created_at DESC);
```

`object_id` and `object_type` together identify the specific object that triggered the alert (e.g., a Signal UUID and `'signal'`). They may be null for alerts that are not tied to a single object (e.g., a `burst_detected` alert for a whole Neighborhood).

---

## 4. Burst detection

### 4.1 What a burst is

A burst is a statistically significant **acceleration** in signal volume for a Cluster or Neighborhood within a time window, relative to a historical baseline.

Key distinction: burst ≠ high volume. A high-volume Neighborhood that has been consistently active is not bursting. A previously quiet Neighborhood that suddenly receives a surge of new signals **is** bursting. The burst signal is about the rate of change, not the absolute count.

Burst detection is a primary early-warning mechanism for the platform. When a Neighborhood starts bursting, it may indicate an emerging technology development, a geopolitical event, or a coordinated publication campaign — all of which are analytically significant.

---

### 4.2 Burst detection algorithm

**Rolling window comparison**

For each active Cluster and Neighborhood, the algorithm compares:

- **Recent window:** signal count in the last N days (default: 14)
- **Baseline window:** signal count in the N days immediately prior (days N+1 through 2N)

**Burst score formula**

```
burst_score = (recent_count - baseline_count) / max(baseline_count, 1)
```

This produces a normalized acceleration ratio. Examples:

| Recent | Baseline | Burst score |
|---|---|---|
| 10 | 4 | 1.50 (bursting) |
| 10 | 10 | 0.00 (stable) |
| 4 | 10 | −0.60 (decelerating) |
| 5 | 0 | 5.00 (new activity, treated as bursting) |

**Burst threshold**

`burst_score >= 0.50` marks a Cluster or Neighborhood as `is_bursting = true`.

This threshold means the recent window has at least 50% more signals than the baseline window. The threshold is configurable at the platform level.

**Minimum signal count guard**

To prevent false positives from very low-volume clusters, a Cluster or Neighborhood must have **at least 3 signals in the recent window** to qualify for burst detection. A cluster with 2 signals in the recent window and 0 in the baseline has a burst score of 2.0 but does not trigger a burst.

**Window size**

The default window size is 14 days (recent) vs. 14 days (prior). Window size is configurable at the platform level. Per-Neighborhood configurability is deferred to Phase 2.

> **DECISION NEEDED —** Should analysts be able to set custom burst window sizes per watchlist, or is the platform-level default sufficient for MVP? *Recommendation: platform-level default (14/14 days) for MVP; per-Neighborhood and per-watchlist window configurability is a Phase 2 feature once analysts have used burst detection enough to have calibrated preferences.*

---

### 4.3 Burst detection job

The burst detection job runs as a scheduled background process.

**Schedule:** every 6 hours (configurable)

**Job steps:**

1. For each active Neighborhood, compute `burst_score` using the rolling window formula across all signals with `neighborhood_id = X`
2. For each active Cluster, compute `burst_score` using all signals assigned to that Cluster
3. Update `burst_score` and `is_bursting` on the `clusters` and `neighborhoods` tables
4. For any Cluster or Neighborhood where `is_bursting` changed from `false` to `true`: create alerts for all analysts with matching active watchlists
5. Write a record to `burst_history` for every Cluster/Neighborhood processed (Section 4.4)
6. Surface burst events to the platform's burst alert queue (DS-08)

**Idempotency:** the job is idempotent. Re-running it within the same window produces the same scores and does not create duplicate alerts (alerts are deduplicated by `watchlist_id + object_id + alert_type + date(created_at)`).

---

### 4.4 Burst decay and history

**Burst decay**

On each job run, if a Cluster's or Neighborhood's `burst_score` has dropped below the threshold, `is_bursting` is set back to `false`. There is no hysteresis in MVP — the score crossing the threshold in either direction is the single state transition point.

**Burst history table**

Every burst detection job run writes a record per evaluated object, preserving the full history of burst scores for signal archaeology:

```sql
CREATE TABLE burst_history (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  object_id           UUID NOT NULL,
  object_type         TEXT NOT NULL CHECK (object_type IN ('cluster', 'neighborhood')),
  burst_score         NUMERIC(6, 4) NOT NULL,
  recent_count        INTEGER NOT NULL,
  baseline_count      INTEGER NOT NULL,
  window_days         INTEGER NOT NULL,
  is_bursting         BOOLEAN NOT NULL,
  evaluated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_burst_history_object
  ON burst_history (object_id, object_type, evaluated_at DESC);

CREATE INDEX idx_burst_history_time
  ON burst_history (evaluated_at DESC);
```

This table supports burst archaeology queries (Section 5.3) without requiring point-in-time reconstruction of the underlying signal table.

---

## 5. Signal archaeology queries

Per ADR-015, the platform maintains temporal metadata on signals and classifications to support point-in-time queries. This section defines the query patterns this spec introduces or depends on.

### 5.1 Point-in-time reconstruction

**Query pattern**

"Show me all active Technology Profiles and their classifications as of [date]."

**Implementation**

All temporal objects carry `valid_from` and `superseded_at` columns (per ADR-015). A point-in-time query filters:

```sql
WHERE valid_from <= :as_of_date
  AND (superseded_at IS NULL OR superseded_at > :as_of_date)
```

Supported on: `match_sets`, `object_relationships`, `tech_profiles`, `signals`.

The search API accepts an optional `as_of` parameter. When provided, it applies the temporal filter above to all queries, restricting results to objects that existed and were active at that point in time.

---

### 5.2 Late-signal detection queue

Per ADR-015, signals where `ingested_at - event_time > threshold` (default: 30 days) are surfaced in the late-signal queue.

The search and archaeology UI exposes a late-signal review mode where analysts can:

1. See signals that arrived significantly after their event time
2. Assess whether those signals would have changed existing classifications at their `event_time`
3. Flag signals for retroactive re-classification review (which creates a task for the analyst, not an automatic re-classification)

The threshold is configurable at the platform level.

---

### 5.3 Burst archaeology

**Query pattern**

"Show me what was bursting in [Neighborhood] between [date range]."

**Implementation**

Uses the `burst_history` table (Section 4.4) directly:

```sql
SELECT *
FROM burst_history
WHERE object_id = :neighborhood_id
  AND object_type = 'neighborhood'
  AND evaluated_at BETWEEN :from_date AND :to_date
ORDER BY evaluated_at ASC;
```

This allows analysts to see the full burst score time series for a Neighborhood, identify when burst events started and ended, and correlate them with specific signals that arrived during those windows.

---

## 6. Performance requirements

| Requirement | Target |
|---|---|
| Search response time | < 500ms for keyword + filter queries on up to 100,000 signals |
| Alert delivery latency | < 5 minutes from triggering event to in-platform notification |
| Burst detection job duration | < 10 minutes for a full run at baseline signal volume |
| Full-text index staleness | < 5 minutes from signal insert to searchable (if background-job approach is chosen) |
| Cursor pagination page load | < 200ms for subsequent pages (index-backed cursor) |

**Index strategy summary**

- GIN indexes on all `search_vector` columns
- B-tree indexes on all foreign keys, `created_at`, `ingested_at`, `event_time` (for date range filters)
- Composite index on `alerts (analyst_id, read, created_at DESC)` for unread notification queue
- Composite index on `burst_history (object_id, object_type, evaluated_at DESC)` for archaeology queries

---

## 7. DECISION NEEDED items

Three open decisions from this spec:

> **DECISION NEEDED —** Full-text search index update strategy: trigger-based (immediate consistency, write latency penalty on every signal insert) vs. background refresh job (< 5 minute staleness, no write latency). *Recommendation: background refresh job with < 5 minute lag; this is acceptable for intelligence workloads where analysts are not expecting sub-second search visibility for newly ingested signals.*

> **DECISION NEEDED —** Whether to expose the watchlist and alert API to ADR-011 extension services in Phase 2, allowing extensions to create watchlists and receive alerts via webhook (TS-09). *Recommendation: yes, expose via TS-09 in Phase 2; extension services subscribing to alerts is consistent with the ADR-011 extension model and does not require MVP changes.*

> **DECISION NEEDED —** Burst window size configurability: should analysts be able to set custom window sizes per watchlist or per Neighborhood, or is the platform-level default (14-day recent / 14-day baseline) sufficient for MVP? *Recommendation: platform-level default only for MVP; per-Neighborhood and per-watchlist configurability is a Phase 2 feature once operational experience with burst detection has been established.*

---

*Depends on: TS-02, TS-03, TS-04, ADR-002, ADR-011, ADR-015*
*Referenced by: DS-08 (burst alert queue), TS-09 (extension API)*
