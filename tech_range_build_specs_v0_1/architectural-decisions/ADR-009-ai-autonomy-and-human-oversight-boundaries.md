# ADR-009: AI Autonomy and Human Oversight Boundaries

**Date:** 2026-07-06
**Deciders:** Platform Architect, Senior Analyst Lead
**Technical Story:** TS-12 (AI Assistance and Human Oversight)
**Related ADRs:** ADR-001 (Platform Principles), ADR-008 (Workflow State Model)

---

## Status: Accepted

---

## Context

### Why this is an architectural decision, not a UI policy

The question of where the AI/human boundary sits is not a styling choice or a workflow preference. It determines which system components are permitted to write to which tables, which state transitions require a human actor_id, and how audit logs are structured. Getting this wrong at the architectural level means either building a system that silently acts on behalf of analysts in a national security context, or building one so constrained that AI adds no throughput benefit and analysts route around it. Both failure modes are unacceptable. The boundary must be decided explicitly, documented, and enforced at the data layer — not left to UI conventions that can erode over time.

### The national security context

Tech Range serves a CTO function operating in a defense and national security environment. The platform's outputs — Technology Profiles, Advisory sections, CCE classifications, match_set approvals — are consumed by decision-makers who may act on them. Errors in classification or advisory content do not simply produce a bad user experience; they can distort strategic assessments, misallocate attention, or generate false confidence in the absence of a real threat (or the inverse). The asymmetry between the cost of an AI error and the cost of an analyst's additional click means that default assumptions must lean toward human confirmation, not AI autonomy, wherever strategic outputs are involved.

### The velocity problem

Signal volume at the ingestion layer makes fully manual processing unworkable. The platform must ingest, enrich, and initially classify signals at a rate no analyst team can match by hand. If AI assistance is not embedded in the pipeline, the queue grows faster than analysts can clear it, and the platform fails its core function. AI is therefore not optional — the question is only where it is authorized to act on its own, and where it must stop and wait.

### The trust problem

AI confidence scores are not ground truth. In early deployment, models will be miscalibrated against the specific signal corpus and entity taxonomy of this platform. Calibration improves over time as analyst feedback accumulates, but it cannot be assumed at launch. Treating AI outputs as reliable before calibration is established would mean building a system that is wrong in ways that are not visible until a strategic output is examined and found to be based on an uncorrected AI error. The tier structure in this ADR is designed to ensure that actions with strategic consequences are gated behind human review during the calibration period, and that the path to expanding AI autonomy is defined, not assumed.

### Failure modes being designed against

Three specific failure modes drove this decision:

1. **Autonomous publication.** AI produces a draft Advisory, confidence is high, and the system publishes it without an analyst ever reviewing it. This must be architecturally impossible, not merely discouraged.

2. **Silent override of analyst judgment.** An analyst classifies a signal; a subsequent AI re-scoring pass silently changes that classification without generating a visible conflict. The analyst's judgment disappears into the audit trail without anyone noticing it was overridden.

3. **Unauditable AI action.** AI takes an action — enrichment lookup, entity extraction, rationale draft — and no record exists of which model version produced which output from which input. When a downstream error is traced back, there is no way to determine whether the AI or a human introduced it.

---

## Decision

AI operates in three tiers. Tier assignment is determined by the reversibility of the action, the strategic weight of its output, and the current calibration maturity of the underlying model.

### Tier 1 — Fully Autonomous

No human gate required. These actions execute automatically during ingestion or processing pipelines.

- Entity extraction from signal text
- Signal summarization for display (not for publication)
- Initial CCE classification scoring (producing a candidate score, not a committed classification)
- Ingestion-time enrichment lookups (entity resolution, taxonomy matching, external reference linking)
- Ambiguity note generation (flagging, not resolving)
- Evidence chain summarization for analyst review queues
- Surprise analysis flagging (surfacing for analyst attention, not escalating)

**Rationale for Tier 1 eligibility:** These actions are reversible, produce inputs to human review rather than final outputs, and do not generate content that is surfaced externally or used directly in strategic documents. An error in Tier 1 is visible to the analyst who reviews the queue and can be corrected before it influences anything downstream.

### Tier 2 — Suggest with Easy Approval

Human can approve with a single action. The default state is "accept" — meaning the suggestion is surfaced prominently and the analyst confirms or dismisses. This tier is appropriate for high-volume actions where analyst attention is the scarce resource and friction on routine approvals degrades platform utility.

- CCE classification results for non-boundary cases (confidence ≥ 0.75, not within 0.05 of a tier boundary)
- Rationale text drafts for analyst editing and confirmation
- Ambiguity notes that the analyst is resolving (not generating)

**Tier 2 does not apply to boundary cases.** Any classification within the boundary confidence band (defined in TS-12) falls to Tier 3 regardless of overall confidence score.

### Tier 3 — Suggest Requiring Explicit Human Action

Default state is "reject." The analyst must affirmatively act to accept the suggestion. Silence is not approval. These actions produce or directly influence strategic outputs, and the cost of an uncorrected AI error exceeds the cost of requiring the analyst's deliberate engagement.

- Boundary case classification resolution (confidence within boundary band or confidence < 0.75 with flags)
- Any classification with confidence < 0.50
- Technology Profile creation (AI may pre-populate fields; a human must commit the record)
- Advisory section drafts (AI may produce a draft; a human must approve and publish)
- Match_set approval (review_status = 'approved' requires actor_id)
- Any re-classification that would change a previously analyst-approved classification

### Actions AI Must Never Take

These actions are architecturally prohibited regardless of confidence score, model version, or tier assignment. They are enforced at the data layer, not the UI layer — the system must reject these writes if the actor is not a human user.

- Publish an Advisory
- Set review_status = 'approved' on any match_set
- Create a committed Technology Profile record
- Override or silently replace an analyst-approved classification
- Access or initiate processing of sensitive content without analyst initiation
- Escalate or de-escalate an object's workflow state (e.g., moving a signal from 'review' to 'escalated')

These are not defaults that can be overridden by configuration. They represent the hard boundary of AI authority in this system.

---

## Rationale

### Why Tier 1 is safe to automate

Tier 1 actions share a key property: they produce inputs to human review, not outputs that reach decision-makers. Entity extraction gives an analyst a starting point; it does not make a claim. Signal summarization surfaces content for review; it does not classify it. Initial CCE scoring produces a candidate; it does not commit a classification. Errors in Tier 1 are visible to the analyst at review time and correctable before they propagate. The volume of these actions makes human-in-the-loop gating impractical, and the reversibility of the actions makes gating unnecessary.

### Why Tier 2 uses easy approval

Classification of non-boundary cases is the highest-volume action requiring any judgment. If every CCE classification required the same deliberate engagement as a boundary case, analyst attention would be exhausted on routine confirmations, leaving less capacity for the genuinely ambiguous signals where human judgment matters most. Easy approval preserves the analyst's role as the decision-maker while acknowledging that for clear cases, the decision is low-cost and high-frequency. The key design constraint is that easy approval must remain visibly an approval — the analyst sees the suggestion and acts, rather than having the suggestion silently committed.

### Why Tier 3 requires explicit action

Advisory sections, Technology Profiles, and boundary-case classifications are the platform's strategic deliverables. They are the outputs that get read, acted on, and cited. A wrong AI draft that an analyst never reviewed is an undetected error in a strategic document. Making the default "reject" ensures that if an analyst is overwhelmed, distracted, or absent, the system does not silently populate strategic outputs with unreviewed AI content. The cost to the analyst of one deliberate click is low. The cost of an unreviewed AI output in a national security advisory is not.

### Why the "never" list is absolute

Publication, approval, and profile creation share a characteristic that distinguishes them from Tier 3 suggestions: they are either irreversible or near-irreversible in practice. A published Advisory has been read. An approved match_set has downstream effects on workflow state. A committed Technology Profile is the ground-truth record that other analyses reference. The national security context means that the cost of an autonomous AI error in these actions — a mistaken publication, a false approval — is categorically different from a correctable suggestion. No calibration improvement or confidence threshold makes AI autonomy appropriate here. The "never" list is not a placeholder for a future Tier 3 reclassification. It is a permanent architectural boundary.

Platform Principles P19 (dimensional confidence) and P21 (disconfirmation as first-class evidence) both require human judgment at resolution. AI can flag an anomaly or surface a disconfirming signal, but it cannot determine that a disconfirming signal outweighs confirming evidence in a specific geopolitical context. That determination belongs to the analyst.

---

## Audit Requirements

Every AI action must produce a log entry regardless of tier. The audit log is the mechanism by which the "never" list is verifiable, by which tier reclassifications are assessed, and by which analyst oversight is demonstrated.

### All tiers — required fields

| Field | Description |
|---|---|
| `action_id` | Unique identifier for this AI action instance |
| `action_type` | Canonical name of the action (e.g., `entity_extraction`, `cce_score`, `rationale_draft`) |
| `tier` | 1, 2, or 3 |
| `model_version` | Full version identifier of the model that produced the output |
| `input_hash` | Hash of the input content (enables replay and audit without storing raw content twice) |
| `output` | The AI-generated content or score |
| `confidence_score` | Where applicable |
| `timestamp` | UTC, millisecond precision |
| `object_id` | The signal, profile, or advisory the action applies to |

### Tier 1 — No analyst acknowledgment required

Tier 1 audit entries are written automatically. No analyst interaction is required to complete the log entry. These entries are retained and queryable for calibration and incident analysis.

### Tier 2 — Actor ID required at approval

When an analyst approves a Tier 2 suggestion, the approval event must append `actor_id` and `approval_timestamp` to the audit entry. If the suggestion is dismissed, `dismissed_by` and `dismissed_at` are recorded. A Tier 2 entry without an actor_id indicates a suggestion that has not yet been acted on — not an auto-approved action.

### Tier 3 — Actor ID and rationale entry required

Tier 3 approval events require `actor_id`, `approval_timestamp`, and a non-empty `analyst_rationale` field. The rationale may be brief, but it must be present. This ensures that the analyst's explicit engagement with the AI suggestion is documented, not just the fact that a button was clicked.

### Edit history for AI-generated content

AI-generated content that is subsequently edited by a human — rationale drafts, Advisory section drafts, ambiguity notes — must retain the original AI draft as an immutable record. The audit log must contain: the original AI output, each subsequent edit with actor_id and timestamp, and the final published version. It must be possible to reconstruct the full edit history of any AI-originated content.

---

## Calibration and Improvement

### Tier assignment is not permanent for Tier 2 and Tier 3

The tier assignments in this ADR reflect the platform's state at initial deployment, when AI models are not yet calibrated against the platform's specific corpus and analyst feedback has not yet accumulated. As calibration matures, some Tier 3 action types may become appropriate for Tier 2. Tier 1 assignments are stable by nature — the actions in Tier 1 are there because of their reversibility, not because of model confidence, and that property does not change with calibration.

The calibration model defined in TS-12 governs how confidence scores are evaluated against analyst-approved ground truth, what precision and recall thresholds must be met before a tier reclassification is proposed, and how A/B comparison between model versions is conducted.

### Proposing a tier reclassification

A proposal to move an action type from Tier 3 to Tier 2 requires:

1. A senior analyst submitting a written proposal referencing TS-12 calibration data showing the action type has met the precision/recall thresholds defined there
2. Platform architect review and sign-off
3. A documented update to this ADR (or a superseding ADR) recording the reclassification, its evidentiary basis, and its effective date

No action type may be moved from the "never" list to any tier under any circumstances without a full architectural review and explicit sign-off from the platform owner.

### New model versions

A new model version may not assume an existing tier assignment. Before a new model version is deployed to any tier, it must be evaluated against analyst-approved ground truth using the A/B testing framework defined in TS-12. If the new version's performance on the existing tier's action types meets or exceeds the current version, the tier assignment carries forward. If it does not, the new version may only be deployed to a more restricted tier until calibration data supports promotion.

---

## Consequences

### Positive

- Analysts retain full judgment authority over all strategic outputs. No Advisory is published, no match_set approved, and no Technology Profile committed without a human decision.
- AI adds throughput to the ingestion and review pipeline without replacing the oversight function. Analysts spend their time on the decisions that require human judgment, not on manually processing a queue that could be pre-processed automatically.
- The audit trail is complete. Every AI action is logged with enough information to reconstruct what the model did, with what input, at what confidence, and whether a human reviewed it. This audit trail is sufficient to support both internal quality review and external oversight requirements.
- The tier structure provides a defined path for expanding AI autonomy as calibration matures, without requiring an architectural redesign.

### Negative

- Tier 2 and Tier 3 actions require dedicated UI surfaces for presenting suggestions and capturing approvals. This is non-trivial front-end work and must be scoped into the implementation plan.
- Analyst fatigue from the approval queue is a real risk. If the volume of Tier 2 and Tier 3 suggestions is high relative to the analyst team's capacity, the approval queue becomes a burden rather than a tool. Queue management, batching, and prioritization must be designed into the workflow.
- The full benefit of AI assistance is delayed until calibration matures. In early deployment, analysts will be approving AI suggestions that are wrong more often than they should be. This is expected and by design — the calibration process depends on it — but it means the efficiency gains of easy approval are not realized until the models are performing well on the platform's specific corpus.
- The "never" list and Tier 3 defaults require enforcement at the data layer. This means additional complexity in the write path for advisory publication, match_set approval, and profile creation. UI-only enforcement is insufficient and will not be accepted as an implementation of this ADR.

---

## Open Decisions

> **DECISION NEEDED —** Tier 2 "easy approval" timeout: if an analyst does not act on a Tier 2 suggestion within N days, should it auto-approve, auto-reject, or escalate to a queue manager? The choice has direct implications for audit completeness — an auto-approved suggestion without an actor_id violates the audit requirements in this ADR. *Recommendation: auto-reject with an escalation flag surfaced to the queue manager; never auto-approve. A Tier 2 suggestion that ages out without action is a signal that either the suggestion volume is too high or the suggestion quality is too low — both warrant attention, not silent approval.*

> **DECISION NEEDED —** Who can authorize a tier reclassification (moving an action type from Tier 3 to Tier 2)? The Calibration and Improvement section above proposes senior analyst proposal plus platform architect approval plus ADR update, but the specific roles and approval chain have not been formally assigned. *Recommendation: formalize this as a two-signature requirement — the lead analyst (or designated senior analyst) and the platform architect — with the ADR update serving as the documented record. No verbal or Slack-based approvals. The ADR update is the authorization.*

> **DECISION NEEDED —** How is AI model versioning exposed to analysts during the approval workflow? Analysts approving Tier 2 and Tier 3 suggestions should know which model version produced the content they are reviewing, particularly during periods when multiple versions are in A/B comparison. *Recommendation: display a visible model_version badge on any AI-generated content surface in the UI, including in the approval workflow. The badge should be persistent (not tooltip-only) and should link to the model's current calibration metrics where available. This supports informed analyst judgment and enables analysts to flag systematic errors tied to a specific model version.*

---

*This ADR is implemented by TS-12 (AI Assistance and Human Oversight). Changes to the tier assignments, the "never" list, or the audit requirements defined here must be reflected in both documents.*
