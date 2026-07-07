# TS-10 — Evaluation, Observability, and Quality Metrics

**Status:** Draft  
**Version:** 0.1  
**References:** ADR-005, ADR-014, TS-01, TS-04, TS-05, TS-11, TS-12

---

## 1. Purpose

Tech Range is a signal-first intelligence platform. Quality is not optional — it is the core product guarantee. This spec defines how the platform measures, tracks, and improves quality at two levels:

1. **System performance** — is the pipeline running correctly? are services available? are queues draining at expected rates?
2. **Intelligence quality** — are classifications trustworthy? are Advisories being produced? is analyst judgment feeding back into the system and improving it over time?

Multiple other specs defer to TS-10 for calibration storage and observability:

- **ADR-005** — references calibration event storage for CCE accuracy tracking
- **TS-04 §8** — CCE calibration loop; analyst corrections update factor weights over time
- **TS-12 §6** — classification improvement cycle; correction events accumulate per axis
- **ADR-014** — source credibility score updates driven by corroboration/contradiction events

This spec defines the metrics themselves, the schemas for storing calibration data, and the processes for acting on what the metrics reveal. Rendering and display are out of scope here (see DS-10 for the dashboard artifact); TS-11 covers operational alerting infrastructure.

---

## 2. Observability Layers

Observability in Tech Range is structured across three distinct layers. Each layer has different consumers, different update frequencies, and different response actions.

### Layer 1 — System Health

**Question answered:** Is the pipeline running?

Consumers: on-call engineer, platform architect.  
Update cadence: near real-time (polling or event-driven).  
Response on degradation: operational intervention (restart service, drain queue, scale resource).

### Layer 2 — Data Quality

**Question answered:** Are signals being classified correctly? Are evidence chains complete? Are boundary cases being resolved?

Consumers: senior analyst, data steward, architect.  
Update cadence: daily batch or event-triggered.  
Response on degradation: analyst workflow intervention (resolve backlog, trigger archaeology run, fix enrichment mapping).

### Layer 3 — Intelligence Quality

**Question answered:** Are classifications improving over time? Are Advisories being published? Are analysts engaging with the platform?

Consumers: senior analyst, product owner, architect.  
Update cadence: weekly and monthly reporting cycles, plus the calibration cycle (§6.3).  
Response on degradation: calibration weight review, analyst process adjustment, Advisory workflow review.

---

## 3. System Health Metrics

### 3.1 Pipeline Metrics

| Metric | Description | Unit |
|---|---|---|
| `ingestion_rate` | Signals ingested per hour, by source type | signals/hour |
| `enrichment_success_rate` | % of ingested signals that complete enrichment without error | % |
| `cce_classification_rate` | % of enriched signals that receive a CCE match_set within expected window | % |
| `dead_letter_queue_depth` | Number of signals in dead letter queue (failed processing, not retried) | count |
| `pipeline_stage_latency_p50` | Median end-to-end latency from ingestion to classified match_set | seconds |
| `pipeline_stage_latency_p95` | 95th percentile end-to-end pipeline latency | seconds |
| `pipeline_stage_latency_p99` | 99th percentile end-to-end pipeline latency | seconds |

Per-stage latency should also be tracked individually (ingestion → enrichment, enrichment → CCE, CCE → match_set write) to allow isolation of bottlenecks.

### 3.2 Service Uptime

Each service component defined in TS-01 should have an availability metric tracked as a percentage over rolling 24-hour and 7-day windows. At minimum:

- Ingestion service
- Enrichment service
- CCE classification service
- Review queue service
- Archaeology service
- API gateway
- PostgreSQL primary

### 3.3 Queue Depths

| Queue | Description |
|---|---|
| Review queue depth | Boundary cases awaiting analyst decision |
| Boundary case queue depth | Cases in `PENDING` state not yet assigned |
| Archaeology queue depth | Signals awaiting archaeology run |
| Dead letter queue depth | Failed signals awaiting triage |

Queue depths are leading indicators. A growing review queue may indicate analyst capacity issues before they affect Intelligence Quality metrics.

### 3.4 Database Health

| Metric | Description |
|---|---|
| Connection pool utilization | % of max connections in use (alert threshold: 80%) |
| Query latency p50 | Median query execution time |
| Query latency p95 | 95th percentile query execution time |
| Replication lag | Seconds of lag on read replicas, if applicable |

### 3.5 Alerting Thresholds

The following conditions trigger a system health alert. Alert routing is defined in TS-11.

> **DECISION NEEDED —** Who receives system health alerts and at what escalation path? *Recommendation: operational alerts (dead letter queue, latency, uptime) go to on-call engineer per TS-11; intelligence quality degradation alerts (first-time-right rate drop, Advisory publication gap) go to senior analyst.*

| Condition | Threshold | Severity |
|---|---|---|
| Dead letter queue depth | > 50 items | High |
| Pipeline stage latency p95 | > 10 seconds | High |
| Pipeline stage latency p99 | > 30 seconds | Critical |
| Enrichment success rate | < 90% over 1 hour | High |
| CCE classification rate | < 85% over 1 hour | High |
| Service availability | Any service < 99% over 1 hour | Critical |
| Database connection pool | > 80% utilization | Medium |
| Replication lag | > 30 seconds | High |

---

## 4. Data Quality Metrics

### 4.1 Classification Coverage

**Definition:** % of active Signals that have at least one approved `match_set` (i.e., analyst-confirmed classification on at least one axis).

**Target:** > 90% of Signals older than 7 days should have at least one approved match_set. Signals less than 7 days old are in the normal classification window and excluded.

**Degradation signal:** A drop below 85% indicates either CCE throughput issues, analyst review backlog, or a new signal type that CCE is not handling.

### 4.2 Boundary Case Resolution Rate

**Definition:** % of boundary cases resolved within SLA.

> **DECISION NEEDED —** What is the target time from boundary_case creation to analyst resolution? *Recommendation: 3 business days for standard cases; 1 business day for cases blocking a Technology Profile state transition (e.g., transition from `emerging` to `active` requires classification lock).*

Track separately:
- Standard case resolution rate (target: > 80% within SLA)
- Blocking case resolution rate (target: > 95% within SLA)

Cases older than 2× SLA should escalate to senior analyst automatically.

### 4.3 Evidence Chain Completeness

**Definition:** % of active Technology Profiles with a complete evidence chain, as defined by the TS-05 completeness check.

A complete evidence chain requires:
- At least one corroborated Signal per axis dimension with an approved match_set
- At least one Corroboration Group linking signals from independent sources
- No unresolved contradiction on the primary classification

**Target:** > 75% of Technology Profiles in `active` state should have a complete evidence chain. Profiles in `emerging` state are exempt (incomplete chains are expected during emergence).

### 4.4 Orphaned Object Rate

**Definition:** % of Signals that are not linked to any Corroboration Group after N days from ingestion.

An orphaned signal is one that has been classified (has an approved match_set) but has never been corroborated or placed into contradiction with another signal. Orphaned signals represent intelligence dead ends — they were classified but never integrated into the evidence graph.

**Target:** < 15% of signals older than 30 days should be orphaned.  
**Action trigger:** Orphan rate > 25% triggers a review of the Corroboration Group assignment logic and may trigger an archaeology run.

### 4.5 Temporal Provenance Completeness

**Definition:** % of signals where `event_time` (the time the technology event occurred) is known vs. unknown or estimated.

A signal with `event_time = NULL` or `event_time_confidence = 'estimated'` degrades the temporal accuracy of Technology Profile timelines and Advisory narratives.

**Target:** > 70% of signals should have a known `event_time` (not estimated, not null).

---

## 5. Intelligence Quality Metrics

### 5.1 CCE Accuracy (Per Axis)

**Definition:** % of CCE-assigned classifications later approved without change by an analyst — the "first-time right" rate.

Tracked per axis: `neighborhood`, `domain_area`, `tech_type`, `candidate_type`.

A match_set is counted as "first-time right" if:
- An analyst reviewed it and approved without modification
- No `swap`, `adjustment`, or `dispute` calibration event was recorded

**Target:** > 70% first-time right rate per axis. Below 60% on any axis triggers a calibration review cycle (§6.3).

Track trend: month-over-month change in first-time right rate per axis is the primary signal that calibration weight updates are working.

### 5.2 Analyst Correction Rate

**Definition:** % of match_sets that receive an analyst correction (swap, adjustment, or dispute).

Complement of the first-time-right rate, but counted at the match_set level rather than the axis level. A single match_set may have corrections on multiple axes.

**Breakdown by correction type:**
- Swap rate: % of match_sets with a primary/secondary swap
- Adjustment rate: % of match_sets with a score adjustment > 0.20 delta
- Dispute rate: % of match_sets that are disputed and referred back to CCE

### 5.3 Mean Correction Magnitude

**Definition:** Average absolute delta in `primary_score` across all analyst corrections (swap and adjustment events only).

Formula: `mean(|after_score - before_score|)` across all calibration events where `event_type IN ('swap', 'adjustment')`.

**Interpretation:** A high mean correction magnitude indicates CCE confidence scores are not well-calibrated. A low rate with high correction count indicates many small adjustments — CCE is directionally correct but imprecise.

### 5.4 Reclassification Rate

**Definition:** % of Technology Profiles that have had a Neighborhood or Domain Area reclassification in the last 90 days.

A reclassification is a change to the Technology Profile's primary `neighborhood` or `domain_area` classification — not a score adjustment, but an actual change in classification label.

**Interpretation:** Some reclassification is expected and healthy (technologies evolve). A high rate (> 20% of profiles reclassified in 90 days) may indicate either a genuinely turbulent technology landscape or instability in the classification schema itself.

### 5.5 Advisory Publication Rate

**Definition:** Number of Advisories published per calendar month.

Track by Advisory type if Advisory types are defined. Track trend: month-over-month and rolling 3-month average.

**Target:** To be set by the product owner based on analyst capacity. A publication rate of zero for > 45 days is an engagement health alert.

### 5.6 Advisory Evidence Chain Depth

**Definition:** Average number of hops from a published Advisory to the earliest contributing Signal, traversing the evidence chain.

A shallow chain (average < 2 hops) may indicate Advisories are being written without sufficient evidence integration. A very deep chain (average > 8 hops) may indicate the evidence graph has become too complex to maintain coherently.

**Healthy range:** 3–6 hops on average.

---

## 6. AI Calibration Tracking

### 6.1 Calibration Event Schema

Every analyst interaction with a match_set that modifies or evaluates a classification is recorded as a calibration event. These events are the ground truth for CCE weight improvement.

```sql
CREATE TABLE calibration_events (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  axis              TEXT NOT NULL CHECK (axis IN (
                      'neighborhood',
                      'domain_area',
                      'tech_type',
                      'candidate_type'
                    )),
  event_type        TEXT NOT NULL CHECK (event_type IN (
                      'swap',           -- analyst swapped primary/secondary classification
                      'adjustment',     -- analyst adjusted score by > 0.20 delta
                      'approval',       -- analyst approved without change (weak positive signal)
                      'dispute'         -- analyst disputed the classification entirely
                    )),
  match_set_id      UUID NOT NULL REFERENCES match_sets(id),
  analyst_id        UUID NOT NULL,
  before_primary    TEXT,              -- classification label before analyst action
  after_primary     TEXT,              -- classification label after analyst action (NULL for approval/dispute)
  before_score      NUMERIC(3,2),      -- CCE confidence score before analyst action
  after_score       NUMERIC(3,2),      -- analyst-set score (NULL for approval/dispute)
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX calibration_events_axis_created ON calibration_events (axis, created_at);
CREATE INDEX calibration_events_match_set ON calibration_events (match_set_id);
CREATE INDEX calibration_events_analyst ON calibration_events (analyst_id, created_at);
```

**Notes:**

- `approval` events are recorded even when no change is made. They are weak positive signals: CCE got it right. Volume of approval events relative to correction events is the first-time-right rate.
- `dispute` events do not set `after_primary` or `after_score` — they flag the match_set for re-evaluation. A disputed match_set re-enters the CCE queue with the original signal evidence and the dispute note as additional context.
- This table is part of the audit trail and is retained indefinitely (§9).

### 6.2 Weight Version Tracking

CCE uses per-axis factor weights to compute classification scores. Each approved weight configuration is stored as a version record.

```sql
CREATE TABLE cce_weight_versions (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  axis              TEXT NOT NULL CHECK (axis IN (
                      'neighborhood',
                      'domain_area',
                      'tech_type',
                      'candidate_type'
                    )),
  weights           JSONB NOT NULL,       -- {factor_name: weight_value, ...}
  version_note      TEXT,                 -- human-readable description of what changed and why
  approved_by       UUID NOT NULL,        -- references analyst/user who approved this version
  approved_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  superseded_at     TIMESTAMPTZ,          -- set when a newer version becomes active
  is_active         BOOLEAN NOT NULL GENERATED ALWAYS AS (superseded_at IS NULL) STORED
);

CREATE INDEX cce_weight_versions_axis_active ON cce_weight_versions (axis, is_active);
```

**Notes:**

- Only one weight version per axis can be active at a time (`is_active = TRUE`). When a new version is approved, `superseded_at` is set on the prior version.
- The `weights` JSONB field stores factor names and weights as defined in TS-04. Example structure: `{"taxonomy_match": 0.35, "source_credibility": 0.25, "temporal_recency": 0.20, "corroboration_count": 0.20}`.
- `version_note` should document what the calibration event analysis showed, why weights were adjusted, and who reviewed. This is the institutional memory of CCE improvement.
- Weight versions are retained indefinitely (§9).

### 6.3 Calibration Cycle

The calibration cycle runs monthly. It is a supervised process — weight changes require human approval and are never applied automatically.

**Step 1 — Event collection (monthly)**  
Collect all `calibration_events` from the past 30 days, grouped by axis.

> **DECISION NEEDED —** What is the minimum number of correction events per axis per cycle before a weight update is permitted? *Recommendation: 30 events minimum per axis (any mix of swap, adjustment, dispute). If fewer than 30 events exist for an axis, hold current weights and accumulate — do not update weights on thin data. Document the hold in the weight version log.*

**Step 2 — Per-factor accuracy analysis**  
For each factor in each axis, compute the correlation between high factor scores (top quartile) and analyst-approved outcomes (approval events). Factors that strongly predict analyst agreement should receive higher weights; factors that do not discriminate should receive lower weights.

This analysis is produced as a structured report, not as automatic weight changes.

**Step 3 — Weight adjustment proposal**  
An automated process proposes weight adjustments based on the per-factor analysis. The proposal includes:
- Current weights
- Proposed weights
- Calibration event volume per axis
- Projected first-time-right rate improvement (based on backtest against the 30-day event set)

The proposal is surfaced to the senior analyst and architect for review. It is not applied without approval.

**Step 4 — Review and approval**  
Senior analyst and architect review the proposal together. They may:
- Accept as proposed
- Modify the proposal (adjust one or more factor weights manually)
- Reject and defer (if event volume is insufficient or the proposed change is too large)

If accepted, a new `cce_weight_versions` record is created with `version_note` documenting the decision rationale.

**Step 5 — Activation and monitoring**  
CCE switches to the new weight version immediately upon record creation.

A 30-day monitoring period follows: compare the first-time-right rate (§5.1) in the 30 days before activation vs. the 30 days after. If the post-activation rate is lower than pre-activation by > 5 percentage points on any axis, the prior weight version is reinstated and the calibration cycle is re-examined.

---

## 7. Source Credibility Quality Tracking

Source credibility scores (managed per ADR-014) are themselves subject to quality tracking.

### 7.1 Per-Source Track Record

For each active source, track over a rolling 90-day window:
- Number of signals ingested
- Number corroborated (signal matched and reinforced by signals from independent sources)
- Number contradicted (signal contradicted by analyst judgment or independent signals)
- Corroboration ratio: `corroborated / (corroborated + contradicted)`

This per-source track record is the empirical basis for credibility score updates.

### 7.2 Score Drift Alerts

When any source's `credibility_score` changes by > 0.15 within a 30-day window, a score drift alert is generated and sent to the senior analyst.

The alert should include:
- Source identifier and name
- Previous score and new score
- The corroboration/contradiction events that drove the change
- Any Signals from this source currently used as primary evidence in active Technology Profiles

Score drift alerts are informational — they do not automatically lock or unlock the source. Analyst judgment determines whether the score change reflects a genuine change in source reliability or a data artifact.

### 7.3 Enrichment Provider Accuracy

Where enrichment data is provided by external providers (taxonomy mappings, entity resolution, etc.):

- Track % of enrichment data later confirmed by analyst judgment
- Track % of enrichment data later refuted or corrected by analyst judgment
- Provider accuracy is factored into decisions about enrichment provider selection (TS-11 territory for vendor management; this spec captures the metric)

---

## 8. Analyst Engagement Metrics

Analyst engagement metrics are a proxy for platform health from the human side. A technically functional platform that analysts are not using is not producing intelligence.

### 8.1 Review Queue Throughput

**Definition:** Number of boundary cases resolved per analyst per week.

Track as individual analyst metrics and as team aggregate. A sustained drop in throughput is a capacity or process signal.

### 8.2 Mean Time to Resolve

**Definition:** Average time from `boundary_case` creation to analyst resolution (any terminal state: approved, swapped, disputed-and-closed).

Track separately by case type (standard vs. blocking) to validate against the boundary case SLA defined in §4.2.

### 8.3 Watchlist Coverage

**Definition:** % of active Technology Profiles watched by at least one analyst.

An unwatched Technology Profile will not receive timely review when new signals arrive. Low watchlist coverage is a signal that the analyst team's attention is concentrated on a subset of the tracked technology landscape.

**Target:** > 80% of Technology Profiles in `active` state should be on at least one analyst's watchlist.

### 8.4 Advisory Contribution

**Definition:** Number of distinct analysts who contributed to at least one published Advisory in the last 90 days.

Contribution includes: authorship, evidence chain review, match_set approval used as primary evidence in the Advisory, or formal peer review of the Advisory before publication.

A single-analyst Advisory pipeline is a single point of failure for intelligence production.

---

## 9. Metrics Storage and Access

### 9.1 System Health Metrics

System health metrics (§3) are emitted as:
- **Structured logs** from each service component — JSON-formatted, including service name, metric name, value, and timestamp
- **Prometheus-compatible counters and gauges** — for pipeline rates, queue depths, and latency histograms

Implementation details for metric collection, scraping, and alerting are deferred to TS-11. This spec defines what metrics exist; TS-11 defines how they are collected and routed.

### 9.2 Intelligence and Calibration Metrics

Data quality and intelligence quality metrics (§4, §5, §6, §7) are computed from PostgreSQL as analytical queries against:
- `calibration_events` — the primary calibration data store
- `cce_weight_versions` — weight history
- `match_sets` — classification results
- `signals` — raw signal inventory
- `technology_profiles` — profile state and evidence chain data
- `boundary_cases` — resolution tracking

These metrics are computed on demand or via scheduled batch queries. They are not cached in a separate metrics store unless query performance requires it (deferrable architectural decision).

### 9.3 Dashboard

A metrics dashboard is DS-10 territory (or a future artifact). This spec defines what data is available and how it is stored. Visualization, layout, and access controls for the dashboard are out of scope here.

### 9.4 Retention Policy

| Data | Retention |
|---|---|
| `calibration_events` | Indefinite — part of the audit trail and calibration history |
| `cce_weight_versions` | Indefinite — institutional memory of CCE improvement |
| System health logs | 90 days (operational; older data has low marginal value) |
| Prometheus metrics | 30 days in time-series storage; monthly summaries retained indefinitely |
| Analyst engagement metrics | Computed from source tables; no separate retention policy needed |

---

## 10. DECISION NEEDED Items

> **DECISION NEEDED —** What is the target time from `boundary_case` creation to analyst resolution? *Recommendation: 3 business days for standard cases; 1 business day for cases blocking a Technology Profile state transition (e.g., a profile cannot transition from `emerging` to `active` while a blocking boundary case is unresolved).*

> **DECISION NEEDED —** What is the minimum number of calibration events per axis per cycle before a weight update is permitted? *Recommendation: 30 events minimum per axis (any mix of swap, adjustment, dispute calibration events). If fewer than 30 events exist for a given axis in the 30-day window, hold current weights and accumulate — do not update weights on thin data. Document the hold decision in the weight version log with the event count that was available.*

> **DECISION NEEDED —** Who receives system health alerts and at what escalation path? *Recommendation: operational alerts (dead letter queue depth, pipeline latency, service availability, database health) go to the on-call engineer per TS-11; intelligence quality degradation alerts (first-time-right rate drop below 60%, Advisory publication gap > 45 days, watchlist coverage below 70%) go to the senior analyst.*
