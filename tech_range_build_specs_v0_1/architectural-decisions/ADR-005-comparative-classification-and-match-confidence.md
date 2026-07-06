# ADR-005 — Comparative Classification and Match Confidence Model

**Status:** Draft
**Date:** 2026-07-06
**Deciders:** Tech Range architecture team
**Supersedes:** None
**Superseded by:** None

## Context

Tech Range classifies every evidence object — Signals, Corroboration Groups, Clusters, and Technology Profiles — against multiple axes simultaneously: Technology Type, Core Technology Neighborhood, Domain Area, and Engagement Posture / Solution Candidate Type.

A naive classification model would assign each object a single static label per axis. This fails for Tech Range for three reasons:

1. **Evidence is weak and changing.** Signals arrive before technologies are well-understood. Early evidence may support multiple Neighborhoods equally. As more evidence arrives, the best match may shift.

2. **Category boundaries are not crisp.** Technologies routinely span Neighborhoods (a Physical AI system may require SDx for deployment). Forcing single-label assignment loses important boundary information.

3. **Classification errors compound downstream.** If a Signal is assigned to the wrong Neighborhood, the Technology Profile inherits that error. If the profile is wrong, the Solution Candidate is misaligned. If the Solution Candidate is wrong, leadership receives a bad recommendation.

Comparative classification — where every assignment is a ranked match set with confidence, differentiation, and rationale — addresses all three problems. It forces the classifier to say not just "this is X" but "this is X rather than Y, with confidence Z, because of evidence W."

This ADR defines the data model, confidence scoring approach, boundary case rules, and reclassification triggers for Tech Range's comparative classification model.

## Decision

Tech Range will implement **ranked comparative classification** for every assignment across all classification axes.

### Core model

Every assignment is represented as a **Match Set**: a ranked list of candidate category matches with the following fields per match:

```
match_set:
  object_id         -- the classified object
  object_type       -- Signal | Group | Cluster | TechProfile | SolutionCandidate
  axis              -- TechType | Neighborhood | DomainArea | CandidateType
  primary_match     -- the top-ranked category
  primary_score     -- float 0.0–1.0
  secondary_matches -- array of { category, score } ordered by score desc
  diff_strength     -- primary_score minus highest secondary score (0.0–1.0)
  comparison_set    -- array of all categories considered
  evidence_basis    -- array of Signal/Group/Cluster/Observation IDs supporting the match
  rationale         -- free text explanation from classifier or analyst
  ambiguity_notes   -- free text, required when diff_strength < 0.15
  is_boundary_case  -- boolean, true when diff_strength < 0.15
  review_status     -- system_assigned | analyst_reviewed | approved | disputed | needs_internal_answer
  created_by        -- user ID or system process name
  created_at        -- timestamp
  reviewed_by       -- user ID, null if not yet reviewed
  reviewed_at       -- timestamp, null if not yet reviewed
  valid_from        -- timestamp (when this assignment became active)
  superseded_at     -- timestamp (null if still active)
  superseded_by     -- FK to newer match_set record, null if still active
```

### Confidence scoring

Confidence is expressed as a **float 0.0–1.0** for each match. A score of 1.0 means the classifier has maximum confidence that the object belongs to this category given current evidence. A score of 0.0 means no evidence supports this match.

Confidence is **not** a single generic score. For each axis:

| Axis | Confidence factors |
|---|---|
| Technology Type | Clarity of technical description, source count, analyst review status |
| Core Technology Neighborhood | Signal density in this area, source diversity, Cluster coherence, Neighborhood definition fit |
| Domain Area | Explicit applicability signals, mission/market context, analyst judgment |
| Solution Candidate Type | Fidelity level, Availability level, access path evidence, strategic priority signals |

The system computes an initial confidence score from extractable signals. Analyst review can raise or lower the score, and that adjustment is recorded with rationale.

### Differentiation strength

`diff_strength = primary_score − max(secondary_scores)`

This measures how clearly the primary match beats the alternatives.

| diff_strength | Interpretation |
|---|---|
| ≥ 0.40 | Strong differentiation. High confidence in primary assignment. |
| 0.15–0.39 | Moderate differentiation. Primary is plausible but alternatives are meaningful. |
| < 0.15 | Boundary case. Primary and secondary are nearly tied. Analyst review required. |
| 0.00 | Tie. System cannot distinguish. Escalate immediately. |

### Boundary case handling

When `diff_strength < 0.15`, the system must:

1. Set `is_boundary_case = true`
2. Set `review_status = system_assigned` (not `approved`)
3. Surface the object in the analyst review queue with the boundary case flag
4. Require `ambiguity_notes` to be populated before approval
5. Block downstream promotion (e.g., Profile → Solution Candidate) until `review_status = approved` or `analyst_reviewed`

Analysts reviewing a boundary case may:
- Approve the primary match with written rationale
- Swap primary and secondary matches
- Add a note that the taxonomy needs refinement
- Flag the object as `needs_internal_answer` if the decision requires sensitive context

### Reclassification rules

Match Sets are **immutable once approved**, but can be **superseded** by a newer Match Set.

Reclassification is triggered when:

- A new Signal, Group, or Cluster changes the evidence basis materially
- A secondary match score rises above the primary score (automatic trigger)
- The diff_strength of an approved assignment falls below 0.15 due to new evidence (automatic trigger)
- A category definition changes (batch re-run required across affected objects)
- A new Core Technology Neighborhood is created (batch re-run across Emerging-flagged objects)
- An analyst explicitly requests reclassification

On reclassification:
- The existing Match Set record is closed (`superseded_at` and `superseded_by` populated)
- A new Match Set record is created with `valid_from = now()`
- The history is preserved — never deleted

### Assignment history

Because Match Sets are never deleted, the platform maintains a complete reclassification history for every object on every axis. This supports:

- Signal archaeology (what did we think this technology was 6 months ago?)
- Audit trails for Advisory recommendations
- Surprise analysis (did we miss a reclassification trigger?)
- Taxonomy improvement (which categories generate the most boundary cases?)

## Consequences

### Positive

- Every classification is falsifiable and traceable to evidence
- Boundary cases are surfaced before they cause downstream errors
- Reclassification history enables signal archaeology and surprise analysis
- Analysts can see *why* a classification was made, not just *what* it is
- Taxonomy gaps become visible when the same secondary match recurs across many objects

### Negative

- More data to store per object than a simple label
- Analysts must review boundary cases, which creates queue load
- Initial system confidence scores may be poorly calibrated and require tuning
- Reclassification batch jobs on category changes can be expensive at scale

### Mitigations

- Boundary case queue should be prioritized over full review queue
- Confidence calibration should be measured and reported (see TS-10)
- Category changes should be previewed before committing to estimate reclassification volume
- History table can be partitioned by `valid_from` for query performance

## Alternatives considered

### 1. Single-label assignment with no confidence

Simplest implementation. Each object has one label per axis.

Rejected because: provides no information about how confident the assignment is, obscures boundary cases, makes reclassification invisible, and gives analysts no basis for disagreeing.

### 2. Multiple tags with no ranking

Each object can have multiple Neighborhood tags, Domain Area tags, etc., but no ranking or confidence.

Rejected because: loses the primary/secondary distinction that is essential for Advisory generation and for understanding which Neighborhood "owns" an object.

### 3. Ordinal confidence (Low / Medium / High)

Simpler to display and reason about by hand.

Rejected because: ordinal values cannot support the `diff_strength` calculation that drives boundary case detection. The system needs a numeric gap to determine when a boundary case exists. Ordinal values can be derived from numeric scores for display purposes.

### 4. Per-dimension confidence objects

Store each confidence dimension (source credibility, corroboration strength, etc.) as a separate field rather than rolling up to a single score per match.

Not rejected — this is additive. The rolled-up score is required for `diff_strength` and boundary case logic. Dimensional breakdowns may be stored as supplementary fields once the core model is stable. This is flagged as a future enhancement.

## Open questions requiring internal decisions

> **DECISION NEEDED — Profile promotion gate:** Should `is_boundary_case = true` on a Technology Profile's Neighborhood assignment block Solution Candidate creation, or only generate a warning? *Recommendation: block until reviewed, to prevent misaligned Solution Candidates from reaching leadership.*

> **DECISION NEEDED — Auto-reclassification authority:** Should the system automatically create a new Match Set when a secondary score exceeds the primary, or should it only flag and require analyst confirmation? *Recommendation: auto-create the new Match Set but set `review_status = system_assigned`, not `approved`. This keeps the history moving without requiring a human to initiate reclassification.*

> **DECISION NEEDED — Confidence calibration cadence:** How frequently should confidence scoring be evaluated against analyst overrides to retune the model? *Recommendation: monthly review, tracked in TS-10.*

> **DECISION NEEDED — Taxonomy change governance:** Who has authority to add, rename, or retire a Core Technology Neighborhood or Domain Area, and what is the process for batch reclassification? *Recommendation: define in ADR-001 (repository and documentation boundary) and TS-06 (workflow state machines).*

## Related decisions

- **ADR-002** — Canonical storage model (defines PostgreSQL as persistence layer)
- **ADR-003** — Evidence graph relationship model (defines how relationships are stored alongside match sets)
- **ADR-007** — Technology Profile creation governance (defines how boundary cases on profile-level assignments affect profile lifecycle)
- **ADR-008** — Workflow state model (defines how `review_status` transitions interact with object lifecycle)
- **TS-02** — Canonical Data Model (implements the match_set table and related entities)
- **TS-04** — Comparative Classification Engine (implements the scoring, diff_strength, and reclassification logic)
- **TS-10** — Evaluation, Observability, and Quality Metrics (defines confidence calibration measurement)
