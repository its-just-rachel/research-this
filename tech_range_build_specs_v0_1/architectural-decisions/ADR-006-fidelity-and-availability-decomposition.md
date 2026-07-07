# ADR-006: Fidelity and Availability Decomposition

**Status:** Accepted
**Date:** 2026-07-06
**Deciders:** Tech Range Platform Team
**Related:** TS-02 (tech_profiles schema), ADR-005, Platform Principle P19, 06-fidelity-availability-model.md

---

## Context

### Current model

Tech Range assesses each tracked technology on two core dimensions:

- **Fidelity** (0–5): How mature and proven is this technology? Scores reflect the technology's state in the broader market — from early research (0–1) through production-proven at scale (4–5). This is a market-wide property; it does not depend on who is assessing it.
- **Availability** (0–5): How accessible is this technology to this organization? Scores reflect how readily the organization can obtain, deploy, and operationalize the technology — from completely inaccessible (0–1) through fully in-house and operational (4–5). This is a firm-specific property.

Both scores are time-bound judgments, reassessed on a cadence appropriate to the technology's signal velocity. Together they form the Fidelity×Availability matrix that drives Engagement Posture recommendations and Solution Candidate classification.

### Why the decomposition question matters

A single `availability_level` integer conceals structurally different situations that require different organizational responses:

| Situation | Market Avail. | Firm Access | Deployment Ready | Single score |
|---|---|---|---|---|
| Widely available, we can get it, can deploy it | 5 | 5 | 4 | 5 — accurate |
| Widely available, but we have no vendor relationship | 5 | 1 | — | 3 — misleading |
| Available to large enterprises only, and we qualify | 3 | 4 | 2 | 3 — hides deployment gap |
| Classified/restricted, we have clearance | 1 | 4 | 3 | 2 — penalizes firm-specific advantage |

The bottleneck that limits a technology's strategic usability is rarely "all dimensions are middling." It is almost always a specific blocker — we can't get access, or we can't run it, or contracting is blocked. A single score hides which lever to pull.

### Strategic value of decomposition

Different sub-dimension gaps trigger different organizational actions:

- **Low Market Availability** → monitor the market; no internal action accelerates this
- **Low Firm Access** (despite high market availability) → pursue vendor relationship, partnership, or licensing; escalate procurement
- **Low Deployment Readiness** → invest in prototype infrastructure, skills, or platform capability; assigned to engineering
- **Low Procurement Readiness** → contracting action; compliance review; acquisition pathway planning; assigned to legal/acquisition

A single composite score produces the recommendation "do something about availability." Decomposed scores produce "contact the vendor, assign to procurement, estimated 6-week timeline." The decomposition is what makes Availability actionable.

### Cost of decomposition

- More fields to assess per technology profile (analyst burden increases)
- Schema complexity grows: four sub-dimension columns instead of one
- UI surface must support multi-field assessment without creating fatigue
- Composite formula must be defined, documented, and consistently enforced
- Analysts must understand which sub-dimensions are applicable and when

These costs are real. The decision below addresses them through progressive disclosure and careful scoping of what is MVP.

---

## Decision

**Availability IS decomposed into four sub-dimensions, implemented via a progressive disclosure model. Fidelity is NOT decomposed.**

### Availability sub-dimensions

| Sub-dimension | Scale | Definition | Assessed when |
|---|---|---|---|
| **Market Availability** | 0–5 | How broadly available is this technology in the open market? Is it commercially available, in limited release, research-only, or proprietary/restricted? | Always — this is the entry-point assessment |
| **Firm Access** | 0–5 | Can this specific organization obtain or access the technology? Considers vendor relationships, licensing, classification/clearance requirements, and geographic/regulatory restrictions. | When Market Availability ≥ 2 |
| **Deployment Readiness** | 0–5 | Can this organization realistically deploy and operate this technology given its current infrastructure, skills, and platform capabilities? | When Firm Access ≥ 2 |
| **Procurement Readiness** | 0–5 | Are contractual, compliance, and acquisition pathways in place or achievable? Considers contract vehicle availability, compliance posture, and acquisition authority. | When Firm Access ≥ 2, context-dependent |

### Composite availability_level

The `availability_level` field in `tech_profiles` (TS-02) is computed as:

```
availability_level = min(market_availability, firm_access)
```

This is the **bottleneck metric**: a technology with high market availability but zero firm access is effectively unavailable for strategic purposes. The bottleneck is the binding constraint.

Deployment Readiness and Procurement Readiness are **supplemental fields**. They inform specific action recommendations and Engagement Posture nuance but do not factor into the composite. A technology can be "Available = 4" (we can get it, market is open) while Deployment Readiness = 1 — that combination recommends Evaluate or Prototype posture, not Deploy.

### Fidelity remains single-dimension

Fidelity (0–5) is not decomposed. The question "how proven is this technology in the market?" is sufficiently unidimensional for strategic purposes. Sub-dimensions like "academic maturity vs. commercial maturity" introduce precision that is not needed at the signal-tracking layer. If finer-grained technology maturity assessment is needed, that belongs in a deeper Due Diligence object, not in the core profile.

---

## Rationale

### Why decompose Availability but not Fidelity

Fidelity is a property of the technology itself — it is the same number regardless of which organization is assessing it. Two analysts at different firms looking at the same technology should converge on similar Fidelity scores. Availability sub-dimensions are about *this firm's relationship to the technology* — Firm Access depends on our vendor agreements; Deployment Readiness depends on our infrastructure; Procurement Readiness depends on our contracting posture. These are inherently firm-specific and therefore warrant decomposition at the firm layer.

### Why four sub-dimensions

The four sub-dimensions cover the three most common blocker types encountered in technology adoption:

1. **Can't get it** (Firm Access bottleneck) — the technology exists and is mature, but we have no pathway to obtain it
2. **Can't run it** (Deployment Readiness bottleneck) — we can license it but lack the infrastructure, skills, or platform integration to operate it
3. **Can't contract for it** (Procurement Readiness bottleneck) — we can technically access and deploy it but have no compliant acquisition pathway

Market Availability is the prerequisite assessment that contextualizes all three.

### Why progressive disclosure

Assessing all four sub-dimensions on a newly detected signal is premature and wasteful. A technology at Fidelity = 1 (early research signal) does not warrant a procurement readiness assessment. Progressive disclosure ensures that assessment depth matches object maturity: start with Market Availability, add Firm Access when the market signal is credible, add Deployment and Procurement assessments when firm access is confirmed. This keeps analyst burden proportional to strategic relevance.

### Why min() for composite

Three alternatives were considered for the composite formula:

- **Weighted average** (e.g., 60% market + 40% firm): masks zero-access blockers. A technology with Market = 5, Firm Access = 0 would produce composite = 3, suggesting "moderate availability" when the reality is "completely inaccessible to us."
- **Analyst-set composite**: removes formula consistency; analysts may set it optimistically; loses the bottleneck signal.
- **min()**: correctly surfaces the binding constraint. If either market or firm access is the bottleneck, the composite reflects it. This is conservative but strategically honest.

The min() function may feel pessimistic in edge cases (e.g., Market = 5, Firm Access = 4 → composite = 4, not 5), but the slight downward pressure is appropriate: composite availability should reflect the *harder* of the two conditions, not the easier.

### Alternatives considered and rejected

| Alternative | Reason rejected |
|---|---|
| Keep single Availability score | Loses actionable detail; different blocker types produce identical scores; no way to route to the right organizational action |
| Weighted average composite | Masks zero-access blockers; produces false "moderate availability" signals |
| Full decomposition with equal composite weight | Adds analyst burden without strategic benefit; deployment and procurement readiness do not belong in the core availability composite |
| Three sub-dimensions (drop Procurement Readiness) | Considered seriously; procurement is deferred to Phase 2 (see DECISION NEEDED below), but the schema slot is reserved |

---

## Schema Implications

The `tech_profiles` table in TS-02 currently stores `availability_level INTEGER CHECK (fidelity_level >= 0 AND fidelity_level <= 5)`. The following extension is required.

### New columns on tech_profiles

```sql
-- Availability sub-dimensions (nullable; assessed progressively)
market_availability       INTEGER CHECK (market_availability BETWEEN 0 AND 5),
firm_access               INTEGER CHECK (firm_access BETWEEN 0 AND 5),
deployment_readiness      INTEGER CHECK (deployment_readiness BETWEEN 0 AND 5),
procurement_readiness     INTEGER CHECK (procurement_readiness BETWEEN 0 AND 5),

-- availability_level remains: computed composite (see DECISION NEEDED #1 below)
-- availability_level INTEGER CHECK (availability_level BETWEEN 0 AND 5),

-- Dimensional confidence (per Platform Principle P19)
fidelity_confidence       NUMERIC(3,2) CHECK (fidelity_confidence BETWEEN 0 AND 1),
availability_confidence   NUMERIC(3,2) CHECK (availability_confidence BETWEEN 0 AND 1),

-- Reassessment tracking
fidelity_assessed_at          TIMESTAMPTZ,
availability_assessed_at      TIMESTAMPTZ,
previous_fidelity_level       INTEGER CHECK (previous_fidelity_level BETWEEN 0 AND 5),
previous_availability_level   INTEGER CHECK (previous_availability_level BETWEEN 0 AND 5),
```

### Backwards compatibility

`availability_level` is retained as the composite field and continues to serve as the input to the Fidelity×Availability matrix. Existing queries and matrix logic do not change. The sub-dimension columns are additive.

### Null handling for the composite

When sub-dimensions have not yet been assessed, null values require careful handling:

- If `market_availability` is null: `availability_level` should be null (no basis for composite)
- If `market_availability` is set but `firm_access` is null: composite = `market_availability` with a confidence penalty applied (firm access unknown, assume bottleneck may exist)
- If both are set: composite = `LEAST(market_availability, firm_access)`

> **DECISION NEEDED —** Should `availability_level` be auto-computed as a generated column, or remain analyst-set? *Recommendation: implement as a generated column: `availability_level INTEGER GENERATED ALWAYS AS (LEAST(COALESCE(market_availability, 0), COALESCE(firm_access, 5))) STORED`. Note: COALESCE behavior for null sub-dimensions needs explicit review — `COALESCE(market_availability, 0)` treats unassessed market availability as 0 (conservative); `COALESCE(firm_access, 5)` treats unassessed firm access as 5 (optimistic, meaning market_availability drives the composite when firm_access is unknown). These defaults should be validated against analyst expectations before finalizing.*

---

## Engagement Posture Integration

### Fidelity×Availability matrix → Engagement Posture

The composite `availability_level` (0–5) and `fidelity_level` (0–5) map to Engagement Posture as follows:

| | **Availability 0–1** | **Availability 2–3** | **Availability 4–5** |
|---|---|---|---|
| **Fidelity 0–1** | Monitor | Monitor | Monitor |
| **Fidelity 2–3** | Monitor / Evaluate | Evaluate | Prototype |
| **Fidelity 4–5** | Evaluate | Prototype / Acquire | Deploy |

This matrix is unchanged by decomposition — it continues to operate on the composite `availability_level`.

### Sub-dimension scores → specific action recommendations

Sub-dimension scores layer onto posture recommendations to specify *which* actions are appropriate:

| Fidelity | Market Avail. | Firm Access | Deployment Ready | Recommended action |
|---|---|---|---|---|
| High (4–5) | High (4–5) | Low (0–1) | — | Partnership pursuit / licensing negotiation; escalate to procurement |
| High (4–5) | High (4–5) | Medium (2–3) | Low (0–1) | Prototype investment; infrastructure/skills gap assessment |
| High (4–5) | High (4–5) | High (4–5) | High (4–5) | Acquire / Deploy; Procurement Readiness becomes gating factor |
| High (4–5) | Low (0–1) | — | — | Monitor market; no internal action accelerates access |
| Medium (2–3) | High (4–5) | High (4–5) | Medium (2–3) | Evaluate; controlled pilot; document deployment constraints |
| Low (0–1) | Any | Any | — | Monitor; signal only; no action recommended |

This action routing logic is computed at the application layer and surfaces in the Engagement Posture recommendation object. The sub-dimension scores are inputs; the routing rules are not encoded in the database schema.

---

## Consequences

### Positive

- **Availability bottleneck is visible.** The composite score surfaces the binding constraint; sub-dimension scores explain why it is what it is.
- **Strategic actions are differentiated by sub-dimension.** "Low availability" no longer produces a generic recommendation — it produces a routed action assigned to the right team (procurement, engineering, legal).
- **Progressive disclosure keeps analyst burden manageable.** New signals require only Market Availability to be assessed; deeper sub-dimensions are unlocked as the object matures.
- **Backwards compatibility preserved.** The Fidelity×Availability matrix and all downstream Engagement Posture logic continue to operate on `availability_level` without change.
- **Schema reserves Procurement Readiness.** Even if deferred to Phase 2, the column exists and is nullable — no migration required to activate it.

### Negative

- **Schema extension required in TS-02.** Migration needed to add sub-dimension columns; existing `availability_level` column behavior changes if implemented as a generated column.
- **UI must support progressive assessment.** The assessment form cannot simply render all four sub-dimension fields at once; it must show or hide fields based on the progressive disclosure rules. This adds UI complexity.
- **Composite formula must be enforced consistently.** If `availability_level` is a generated column, the COALESCE defaults are load-bearing and must be documented. If analyst-set, consistency must be enforced through validation logic.
- **Analysts must understand the model.** The distinction between Market Availability and Firm Access is intuitive once explained, but requires onboarding. A technology can score 5 on market availability and 0 on firm access — analysts who conflate the two will under-assess firm-specific blockers.

---

## Open Decisions

> **DECISION NEEDED —** Should `availability_level` be auto-computed as a generated column or remain analyst-set? *Recommendation: generated column using `GENERATED ALWAYS AS (LEAST(COALESCE(market_availability, 0), COALESCE(firm_access, 5))) STORED`; but the COALESCE defaults for null sub-dimensions must be explicitly validated with the analyst team before the migration is finalized, as they encode a policy choice about how to handle partially-assessed profiles.*

> **DECISION NEEDED —** Is Procurement Readiness in scope for MVP, or deferred to Phase 2? *Recommendation: defer to Phase 2. Procurement Readiness requires integration with contracting, acquisition authority, and compliance data that is unlikely to be available at launch. The schema column is reserved and nullable; the UI assessment flow omits it in Phase 1. Re-evaluate when contracting/compliance data integration is scoped.*

> **DECISION NEEDED —** Who has authority to assess Firm Access and Procurement Readiness sub-dimensions? *Recommendation: Senior Analyst or above for Firm Access (requires knowledge of vendor relationships and organizational access); Program Lead or designated acquisition authority for Procurement Readiness (requires access to sensitive partner and contract information). [Internal decision rule — access control policy must be confirmed with program leadership before implementation.]*

---

## References

- `06-fidelity-availability-model.md` — ground-truth definitions of Fidelity and Availability scales
- `TS-02` — tech_profiles schema specification
- `SPEC-REGISTRY` — flags the decomposition question as an open architectural decision
- Platform Principle P19 — dimensional confidence tracking
- ADR-005 — (related schema decision, if applicable)
