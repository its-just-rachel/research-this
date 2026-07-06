# TS-04 — Comparative Classification Engine

**Version:** v0.1  
**Status:** Draft  
**Date:** 2026-07-06  
**Depends on:** ADR-005, TS-02, TS-06 (workflow state machines)

## Purpose

This document defines the behavior of the Comparative Classification Engine (CCE) — the system component responsible for assigning ranked match sets to every classifiable object in Tech Range.

The CCE is the intelligence core of the platform. It determines which Neighborhood, Domain Area, Technology Type, and Solution Candidate Type best matches each object, with what confidence, and how clearly that match differentiates from plausible alternatives.

The CCE must be inspectable (analysts can see why a classification was made), correctable (analysts can override any score), calibrated (system scores should improve as analyst corrections accumulate), and auditable (every classification and reclassification is preserved).

## Scope

The CCE classifies the following objects on the following axes:

| Object | Axes classified |
|---|---|
| Signal | Neighborhood, Domain Area (optional early) |
| Corroboration Group | Neighborhood, Domain Area (optional early) |
| Cluster | Neighborhood, Domain Area |
| Technology Profile | Neighborhood, Domain Area, Technology Type |
| Solution Candidate | Candidate Type, Domain Area |

The CCE does **not** classify:
- Sources (credibility scoring is a separate process — see ADR-014)
- Observations, Hypotheses, Findings (these inherit classification from linked profiles)
- Advisories (Advisories synthesize; they do not receive CCE-assigned classifications)

## Architecture overview

```
Input object
    |
    v
[Feature Extraction]
    |
    v
[Candidate Scoring]  <-- category definitions from neighborhoods/domain_areas tables
    |
    v
[Differentiation Calculation]
    |
    v
[Boundary Case Detection]
    |
    +-- boundary? --> [Analyst Review Queue]
    |
    v
[Match Set Write]  --> match_sets table (TS-02)
    |
    v
[Downstream Triggers]  --> profile promotion, state updates, burst detection
```

## 1. Feature extraction

Before scoring, the CCE extracts a **feature vector** from the input object. Features differ by object type.

### Signal features

- `signal_type` — categorical (paper, model_release, funding, etc.)
- `extracted_entities.technologies` — list of named technologies
- `extracted_entities.organizations` — list of organizations
- `extracted_entities.topics` — list of topic keywords
- `source.credibility_score` — float from sources table
- `source.coverage_areas` — domains the source covers
- `event_time` — recency signal
- `enrichments` — any available market context, company context, funding context

### Corroboration Group features

- Aggregate of member Signal features
- `corroboration_strength` — float
- `source_diversity_score` — float
- `member_count` — integer
- Time span of member signals (latest_seen_at − first_seen_at)

### Cluster features

- Aggregate of member Signal and Group features
- `coherence_score` — float
- `is_bursting` + `burst_score` — temporal acceleration
- `signal_count`, `group_count`

### Technology Profile features

- All of the above from linked evidence
- `knowledge_state` — current state
- `fidelity_level`, `availability_level`
- Linked Observations and Hypotheses text
- Existing Tech Evaluations

### Solution Candidate features

- Linked Technology Profile features
- `fidelity_level`, `availability_level` at creation time
- `access_path` text
- Strategic rationale text

## 2. Candidate scoring

For each axis, the CCE scores the object against **every active category** in the comparison set.

### Scoring approach

The CCE uses a **factor-weighted scoring model**. For each category, a score is computed as a weighted sum of factor scores:

```
category_score = Σ (factor_weight_i × factor_score_i)
```

Factor scores are floats in [0.0, 1.0]. Weights sum to 1.0. The result is clamped to [0.0, 1.0].

Factor weights are **configurable per axis and per category**. Initial weights are set by the platform team and updated during calibration.

### Neighborhood scoring factors (example initial weights)

| Factor | Initial weight | Derivation |
|---|---|---|
| Keyword/entity match to neighborhood definition | 0.35 | Semantic similarity between extracted entities and neighborhood keyword corpus |
| Source coverage area alignment | 0.15 | Does this source typically cover this neighborhood? |
| Cluster coherence with neighborhood clusters | 0.20 | Do existing clusters in this neighborhood have high similarity to this object? |
| Signal type relevance | 0.10 | Some signal types are more diagnostic (e.g., model_release for Agentic AI) |
| Source credibility | 0.10 | Higher-credibility sources contribute more confident scores |
| Recency | 0.10 | More recent signals score higher (exponential decay on event_time) |

> **DECISION NEEDED — Neighborhood keyword corpora:** Each Core Technology Neighborhood needs a curated keyword/entity corpus that the semantic similarity factor scores against. Who maintains this corpus, and what is the update process? *Recommendation: each Neighborhood has a designated analyst owner responsible for the corpus. Corpus updates require review and trigger a partial re-run of affected match_sets. Define ownership formally in ADR-001.*

### Technology Type scoring factors

| Factor | Initial weight | Derivation |
|---|---|---|
| Linguistic markers in title/summary | 0.40 | "framework", "library", "model", "technique", "platform" etc. |
| Entity type of primary subject | 0.30 | Is the subject a product, a method, a system? |
| Source type signal | 0.20 | Papers → technique/framework; product launches → tool/platform |
| Analyst prior | 0.10 | If analyst has previously typed similar objects, weight toward that type |

### Domain Area scoring factors

| Factor | Initial weight | Derivation |
|---|---|---|
| Explicit domain mention in content | 0.40 | Direct reference to national security, defense, space, etc. |
| Organization type match | 0.25 | Is the originating org a defense contractor, government agency, etc.? |
| Source domain coverage | 0.20 | Does the source typically cover this domain? |
| Neighborhood × Domain correlation | 0.15 | Some Neighborhoods are strongly correlated with specific Domain Areas |

### Solution Candidate Type scoring factors

| Factor | Initial weight | Derivation |
|---|---|---|
| Fidelity level | 0.30 | High fidelity → prototype/pilot candidates score higher |
| Availability level | 0.30 | High availability → prototype/pilot; low → partner access/advisory |
| Access path evidence | 0.20 | Known access path → partner access or procurement scores higher |
| Strategic urgency signals | 0.20 | Burst signals, leadership priority flags → advisory or deep dive |

## 3. Differentiation calculation

After scoring all categories, the CCE computes:

```python
primary_match = argmax(scores)
primary_score = max(scores)
secondary_matches = sorted(scores, descending)[1:]  # all except primary
diff_strength = primary_score - max(secondary_scores)
```

The full ranked list and all scores are stored in `match_sets.secondary_matches` as JSONB.

## 4. Boundary case detection

```python
is_boundary_case = (diff_strength < 0.15)
```

This is also computed as a generated column in PostgreSQL (see TS-02), so no application logic is required to set it. The CCE writes `diff_strength` and the database enforces `is_boundary_case`.

### Boundary case queue routing

When `is_boundary_case = true`, the CCE must:

1. Write the match_set with `review_status = 'system_assigned'`
2. Insert a queue entry for analyst review (see TS-06 for queue model)
3. Include in the queue entry:
   - Object type and ID
   - Axis
   - Primary match and score
   - All secondary matches and scores
   - diff_strength
   - Evidence basis IDs
   - A pre-populated `ambiguity_notes` draft (AI-generated explanation of why the scores are close)

Boundary cases must be resolved before the affected object can be promoted downstream:

| Object | Blocked action |
|---|---|
| Signal | Grouping into a Corroboration Group (if boundary on Neighborhood) |
| Cluster | Technology Profile creation (if boundary on Neighborhood) |
| Technology Profile | Solution Candidate creation (if boundary on Neighborhood or Domain Area) |
| Solution Candidate | Approval and Advisory linkage (if boundary on Candidate Type) |

> **DECISION NEEDED — Blocking strictness:** Should boundary cases hard-block downstream actions, or generate a warning that analysts can override? *Recommendation: hard-block for Neighborhood and Domain Area assignments on Technology Profiles and Solution Candidates (these directly affect Advisory quality). Warning-only for Signal and Group-level boundary cases (lower stakes, higher volume). Define in TS-06.*

## 5. Match Set write

After scoring and boundary detection, the CCE writes a new `match_set` record. It will:

1. Check if an active match_set already exists for this object/axis combination
2. If yes: compare new primary match and diff_strength against existing
   - If `new_primary != old_primary OR new_diff_strength < 0.15 when old was not`: trigger reclassification (close old, write new)
   - If no material change: skip write (idempotent)
3. If no: write new match_set with `valid_from = now()`

"Material change" is defined as:
- Primary match category changes
- Primary score changes by > 0.10
- diff_strength crosses the 0.15 boundary in either direction
- Evidence basis changes by 2+ objects

> **DECISION NEEDED — Idempotency window:** Should re-running the CCE on an already-classified object always be idempotent (skip if no material change), or should there be a minimum time gap before a re-run can generate a new match_set? *Recommendation: idempotent by default; no time gate. This keeps the model simple and makes batch re-runs safe.*

## 6. Reclassification triggers

The CCE must re-run on an object when any of the following occur:

### Automatic triggers

| Event | Scope |
|---|---|
| New Signal added to a Corroboration Group | Re-run the Group |
| New Group or Signal added to a Cluster | Re-run the Cluster |
| New evidence linked to a Technology Profile | Re-run the Profile on all axes |
| Secondary score rises above primary score | Immediate reclassification (system-detected during any CCE run) |
| Analyst marks a match_set as `disputed` | Re-run queued with analyst notes as input |

### Manual triggers

| Event | Scope |
|---|---|
| Analyst explicitly requests reclassification | Single object, all axes |
| Neighborhood or Domain Area definition updated | Batch re-run all active match_sets for that category on the affected axis |
| New Neighborhood created | Batch re-run all Emerging-flagged match_sets |
| Keyword corpus updated for a Neighborhood | Batch re-run all active match_sets on Neighborhood axis |

### Batch re-run management

Batch reclassification jobs should:
1. Run in a background queue, not inline
2. Produce a preview report (objects affected, estimated changes) before committing
3. Preserve the old match_sets via the supersede pattern (TS-02)
4. Post results to an audit log (TS-08)
5. Notify the requesting analyst when complete

## 7. Analyst correction flow

Analysts interact with CCE output through the review interface (DS-03). The correction flow:

```
Analyst opens review queue item
    |
    v
Views: object, primary match, secondary matches, diff_strength, evidence basis, rationale
    |
    v
Analyst may:
  (a) Approve as-is → review_status = 'approved'
  (b) Swap primary/secondary → creates new match_set superseding current; review_status = 'approved'
  (c) Adjust scores manually → creates new match_set with review_status = 'analyst_reviewed'
  (d) Add/remove evidence basis → triggers CCE re-run with updated inputs
  (e) Flag as needs_internal_answer → review_status = 'needs_internal_answer'; escalated
  (f) Flag as disputed → review_status = 'disputed'; escalated to senior analyst or architect
```

All analyst actions are logged with user ID, timestamp, and free-text rationale.

When an analyst swaps primary and secondary:
- The new match_set gets `created_by = analyst_user_id`
- `primary_score` and `secondary_matches` are updated to reflect the swap
- `diff_strength` is recalculated
- If the result still has `diff_strength < 0.15`, `is_boundary_case` remains `true` and the record stays in review

## 8. Confidence calibration

The CCE's factor weights are tunable. Calibration is the process of improving weights based on observed analyst corrections.

### Calibration signal

A calibration signal is generated whenever an analyst:
- Swaps primary and secondary matches (the system was wrong; reduce weight on the factors that favored the rejected category)
- Adjusts a score significantly (> 0.20 delta)
- Approves a score without change (weak positive signal)

### Calibration process

1. Collect correction events over a rolling window (see TS-10 for the recommended window)
2. Compute per-factor accuracy: for each factor, how often did high factor scores predict analyst-approved matches?
3. Adjust factor weights toward better-predicting factors
4. Re-run a sample of recent match_sets with new weights; compare against analyst-approved ground truth
5. Deploy updated weights if accuracy improves; roll back if it degrades

> **DECISION NEEDED — Calibration cadence and authority:** How frequently should factor weights be updated, and who must approve a weight change? *Recommendation: monthly review cycle. Weight changes require sign-off from a senior analyst or the platform architect. Changes are versioned and logged. See TS-10.*

Calibration events and weight versions should be stored for audit purposes. This is deferred to TS-10 for full specification.

## 9. Emerging detection

The CCE plays a role in Emerging detection:

1. When no category scores above 0.40 on the Neighborhood axis, the CCE assigns `Emerging` as primary match
2. When `Emerging` is assigned, the CCE creates an `emerging_records` entry (TS-02)
3. When multiple objects accumulate Emerging assignments and share similar feature vectors, the CCE flags a potential Emerging Cluster Candidate (triggers analyst review)

The Emerging detection thresholds are configurable:

| Parameter | Recommended initial value |
|---|---|
| Max score to assign Emerging primary | 0.40 |
| Min similar-object count to flag Cluster Candidate | 5 |
| Min time window for clustering similar Emerging signals | 30 days |

> **DECISION NEEDED — Emerging promotion authority:** Who can approve an Emerging Cluster Candidate as a true new Cluster? Who can approve an Emerging Neighborhood Candidate as a new Core Technology Neighborhood? *Recommendation: Cluster promotion = senior analyst. Neighborhood promotion = platform architect + leadership sign-off + ADR required. Define in ADR-001.*

## 10. Performance and scaling considerations

- CCE runs should be async by default (background queue); never block signal ingestion
- Single-object classification target latency: < 2 seconds
- Batch re-run throughput target: 10,000 match_sets per hour on baseline infrastructure
- Match_sets table will be the highest-write table in the system; partition by `valid_from` once row count exceeds 1M
- Factor scoring for Neighborhood axis is the most compute-intensive; consider caching semantic similarity scores per (object, neighborhood) pair with a TTL

## 11. What this spec does not cover

- UI for analyst review (DS-03)
- Full state machine for object lifecycle (TS-06)
- Source credibility scoring (ADR-014)
- AI assistance for rationale generation (TS-12)
- Search and alert infrastructure (TS-07)
- Observability and quality metrics (TS-10)
