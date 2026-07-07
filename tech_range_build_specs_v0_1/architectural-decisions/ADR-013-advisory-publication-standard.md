# ADR-013: Advisory Publication Standard

**Status:** Accepted
**Date:** 2026-07-06
**Deciders:** Platform Architect, Lead Analyst
**Related:** TS-05 (Evidence Chain Traversal), TS-06 (Advisory State Machine), ADR-009 (AI Draft Authority), ADR-012 (Sensitive Evidence Handling), Platform Principle P19 (Confidence Transparency)

---

## Context

The Advisory is Tech Range's primary intelligence output — the artifact that leadership consumes to make strategic decisions. Because those decisions may carry national security implications, an Advisory that makes a recommendation without documenting its evidence or acknowledging its uncertainty is not just a quality problem: it is a potential source of strategic harm.

Without a publication standard, the platform drifts toward inconsistency. Different analysts apply different thresholds. Some Advisories are dense with evidence; others are effectively editorials. Leadership cannot calibrate how much weight to give a recommendation if they cannot assess how well-grounded it is. The analytical chain — Signal → Finding/Tech Evaluation → Technology Profile → Advisory — loses its value if it is not preserved in the published artifact.

The failure mode being prevented is specific: an Advisory transitions from `review` to `approved` with a compelling Executive Summary, a vivid Strategic Implication, and no substantiated evidence chain. The recommendation sounds authoritative. The evidence chain is broken or absent. A decision gets made on it.

The tension this ADR must resolve is real. Standards that are too strict slow the pipeline — a time-sensitive intelligence product that takes two weeks to clear publication gates has failed the mission. Standards that are too loose produce unreliable outputs that erode leadership trust over time. This ADR accepts some publication latency in exchange for defensibility of every published artifact.

---

## Decision

An Advisory may not transition from `review` to `approved` unless all requirements in this ADR are satisfied. These requirements are enforced by a combination of automated gate checks (run when an analyst submits for review) and human judgment exercised by the approving authority.

---

### Structural Requirements

All of the following sections must be present and non-empty in the Advisory body:

| Section | Purpose |
|---|---|
| Executive Summary | Plain-language statement of the finding and its significance |
| Strategic Implication | What this means for strategic posture or decision-making |
| Evidence Summary | Synthesis of the supporting evidence chain |
| Confidence Assessment | Overall confidence score plus named dimensions with rationale |
| Recommended Actions | Specific, attributable actions linked to Strategic Implications |
| Analytical Limitations | Honest accounting of what the analysis cannot resolve |

In addition:

- At least one linked Technology Profile must be in `current_state = 'active'`
- At least one linked Finding or Tech Evaluation must be present

Rationale: sections that are present but empty (whitespace, placeholder text, or boilerplate) do not satisfy this requirement. The automated gate checks for content length above a minimum threshold; a human reviewer is responsible for assessing whether the content is substantive.

---

### Evidence Chain Requirements

Per TS-05 chain completeness check:

- The complete evidence chain must be traversable from the Advisory back to at least one Signal
- No broken links may exist in the chain — all `evidence_basis` UUIDs must resolve to existing records
- `overall_confidence` must be provided as a NUMERIC(3,2) value (0.00–1.00)
- At least one named confidence dimension with rationale must be documented, per Platform Principle P19

The evidence chain traversal is deterministic and fully automatable. A broken link is treated as a blocking error, not a warning.

---

### Analytical Chain Rule

The following traceability requirements are enforced as structured field requirements, not prose conventions:

- Every **Recommended Action** must be linked to at least one **Strategic Implication** via structured reference
- Every **Strategic Implication** must be linked to at least one piece of **Evidence** (a Finding, Tech Evaluation, or Evidence record)

This means an analyst cannot write a Recommended Action that floats free of the analytical chain. The recommendation must be traceable to a stated implication, and the implication must be traceable to evidence. Enforcement is structural: the Advisory schema requires these linkages to be populated before the state transition is permitted.

> **DECISION NEEDED —** What is the minimum number of independent sources required in the evidence chain to support any claim appearing in a Strategic Implication? The platform can enforce a numeric floor at the evidence chain level. *Recommendation: at least 2 distinct sources for any Strategic Implication claim, defined as sources with different `source_id` values. Flag as internal decision rule; exact threshold to be confirmed by platform architect and senior analyst.*

---

### Approval Authority

| Advisory type | Required approver |
|---|---|
| Standard Advisory | `senior_analyst` role |
| Advisory containing sensitive evidence references (per ADR-012) | `architect` role |
| Advisory recommending immediate strategic action | [Internal decision rule — leadership sign-off threshold] |

The sensitive evidence escalation is automated: if ADR-012 flagging is present anywhere in the evidence chain, the required approval tier is automatically elevated to `architect`. The analyst does not need to manually identify this condition.

The threshold for what constitutes "immediate strategic action" — and who in leadership must sign off — is a policy decision outside the scope of this ADR.

---

### TL/DR Briefs vs. Advisories

A **TL/DR brief** is a short leadership summary artifact, typically 1–3 paragraphs, designed for rapid consumption. It is distinct from an Advisory in the following ways:

| Dimension | Advisory | TL/DR Brief |
|---|---|---|
| Evidence chain | Owns and contains its evidence chain | Inherits by reference from a linked published Advisory |
| Required sections | All six sections required | No section requirements; must link to a published Advisory |
| Publication gate | Full gate check (this ADR) | Lightweight check (formatting, accuracy, linked Advisory status) |
| Approval authority | Per approval authority table above | Authoring analyst |

A TL/DR brief **cannot be published** without a linked Advisory in `published` state. The brief does not reproduce the evidence chain; it inherits it. A reader who needs to verify the evidence traces to the linked Advisory.

A TL/DR brief is not a substitute for an Advisory. It is a consumption artifact that depends on an Advisory's existence.

> **DECISION NEEDED —** Do TL/DR briefs require their own approval workflow, or do they inherit approval from the linked Advisory? *Recommendation: inherit — if the linked Advisory is in `approved` or `published` state, the TL/DR brief requires only a formatting and accuracy check by the authoring analyst before publication. No separate approval chain. Rationale: the analytical work has already been approved; the brief is a presentation layer, not a new analytical artifact.*

---

## What the Publication Gate Checks (Automated)

The following checks run automatically when an analyst submits an Advisory for the `review` state transition. A failed check blocks the transition and returns a structured error to the analyst.

| Check | Failure condition | Blocking? |
|---|---|---|
| Section completeness | Any required section absent or below minimum content threshold | Yes |
| Evidence chain traversal | Chain does not reach at least one Signal | Yes |
| Broken link check | Any `evidence_basis` UUID does not resolve | Yes |
| `overall_confidence` not null | Field is null or out of range | Yes |
| Linked Technology Profile active | No linked profile in `current_state = 'active'` | Yes |
| Linked Finding or Tech Evaluation | No linked Finding or Tech Evaluation record | Yes |
| Analytical chain traceability | Any Recommended Action has no linked Strategic Implication, or any Strategic Implication has no linked Evidence | Yes |
| Sensitive content flag | ADR-012 flagging present anywhere in evidence chain | Non-blocking; escalates approval tier to `architect` |

All blocking checks must pass before the Advisory enters `review`. The sensitive content flag does not block submission — it modifies the required approval authority.

---

## What Requires Human Judgment (Not Automatable)

The automated gate establishes a floor, not a ceiling. The following dimensions are the responsibility of the approving authority and cannot be reduced to a check:

- **Quality of the Executive Summary:** Is it clear, accurate, and appropriately scoped? Does it avoid overstating certainty?
- **Whether the Strategic Implication is well-reasoned:** Does it follow from the evidence, or does it reach beyond what the evidence supports?
- **Whether the Analytical Limitations are honest and complete:** Does the analyst acknowledge what the analysis cannot resolve, or are the limitations cosmetic?
- **Whether the Recommended Actions are proportionate:** Are they calibrated to the confidence level and the nature of the evidence, or do they overstate the urgency?

An Advisory that passes all automated checks and fails on these dimensions should not be approved. The approving authority is responsible for this judgment.

---

## Review Timing

> **DECISION NEEDED —** How long may an Advisory sit in `review` state before escalation or expiration is triggered? *Recommendation: escalate to the next approval tier after 5 business days of inactivity in `review`; auto-withdraw with notification after 30 calendar days. Rationale: time-sensitive intelligence loses value; a 30-day cap forces a decision and prevents Advisory queue accumulation. Auto-withdrawal moves the Advisory back to `assembling` state with a system-generated note; the analyst may resubmit.*

---

## Consequences

### Positive

- Every published Advisory has a defensible, traversable evidence chain — leadership can trace any recommendation back to its source signals
- The platform's intelligence value compounds over time: a growing body of linked, high-quality Advisories creates a durable analytical record
- Analysts are protected from publishing under-supported analysis under time pressure — the gate provides a structural backstop
- The distinction between TL/DR briefs and Advisories clarifies the consumption model for leadership: briefs are for rapid orientation; Advisories are for decision-making

### Negative

- Publication is slower than a simple "publish" button; analysts working against deadlines will feel the friction
- The Analytical Chain Rule (structured linkage of Recommended Actions → Strategic Implications → Evidence) requires discipline in Advisory authoring; this is a non-trivial workflow change
- The sensitive evidence escalation to `architect` approval may create a bottleneck if a significant proportion of Advisories touch sensitive evidence chains
- Automated checks can create a false sense of completeness — an Advisory that passes all gate checks is not necessarily a good Advisory; that message must be communicated to analysts

---

## Related Decisions

- **TS-05** — Evidence chain traversal logic and chain completeness check implementation
- **TS-06** — Advisory state machine; defines legal state transitions and who may trigger them
- **ADR-009** — AI may draft Advisory sections but may not trigger `review` submission or any state transition at or beyond `review`
- **ADR-012** — Sensitive evidence handling; defines flagging conditions that trigger approval tier escalation in this ADR
- **Platform Principle P19** — Confidence transparency; requires named confidence dimensions with rationale, enforced by this ADR's structural requirements
- **02-ontology** — Ground-truth definition of required Advisory sections and Advisory data model
