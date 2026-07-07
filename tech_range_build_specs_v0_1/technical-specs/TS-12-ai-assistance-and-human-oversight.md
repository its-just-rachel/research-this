# TS-12 — AI Assistance and Human Oversight

**Spec ID:** TS-12  
**Status:** Draft  
**Last updated:** 2026-07-06  
**Related specs:** TS-03 (Ingestion Pipeline), TS-04 (CCE), TS-10 (Observability), TS-11 (Advisory Workflow)  
**Informs:** ADR-009 (AI Autonomy and Human Oversight Boundaries — see ADR-009 (Accepted))

---

## 1. Purpose

Tech Range uses AI assistance at multiple points in the signal ingestion, classification, and analyst workflow pipeline. This spec defines where AI assistance is applied, what form that assistance takes, what human gates are required before AI outputs become ground truth, and how the system maintains inspectability and improves over time through analyst calibration signals.

The platform serves a CTO function in a defense and national security context. Errors in technology classification or advisory content carry real strategic consequences — they can misdirect investment, mischaracterize competitive position, or produce flawed assessments that inform sensitive decisions. AI assistance in this environment must be held to a higher standard of transparency, controllability, and recoverability than typical commercial applications.

This spec does not define the AI models used (model selection is flagged as a DECISION NEEDED item where applicable). It defines the behavioral contract that any AI system integrated into Tech Range must satisfy.

---

## 2. Design Principles for AI in Tech Range

These principles govern all AI integration decisions. They are not aspirational — they are constraints that individual feature implementations must satisfy.

### 2.1 AI suggests, humans decide

No AI output becomes ground truth without a human gate. AI can pre-populate fields, generate candidate text, and produce classification scores, but publication, approval, and record finalization are always triggered by an analyst or authorized user. There is no autonomous publication path.

### 2.2 Every AI output must be inspectable

An analyst must always be able to see *why* the AI produced a given output — not just the result. For classification outputs, this means factor-by-factor score breakdowns. For text outputs, this means a clear disclosure label and, where applicable, a summary of the evidence or source material used to generate the text. Opaque outputs that cannot be explained to an analyst are not acceptable.

### 2.3 AI confidence ≠ ground truth

Classification confidence scores and AI-generated suggestions are inputs to analyst judgment, not conclusions. A high confidence score does not exempt a record from analyst review. A low confidence score is a signal to queue the record for closer attention, not to discard the AI output entirely. Confidence scores are displayed to analysts as context, with explicit UI framing that distinguishes them from analyst-verified classifications.

### 2.4 Analyst corrections are calibration signals

Every time an analyst adjusts an AI-generated classification or substantially revises AI-drafted text, that action is a signal about the quality of the AI's current weights and behavior. The system must capture these corrections and feed them into a calibration process. The accuracy of AI assistance is expected to improve as analysts engage with the system — this requires that the feedback loop is complete, not advisory.

### 2.5 No AI action on sensitive content without explicit human initiation

The platform may handle signals or content that are operationally sensitive. AI components must not autonomously access, summarize, or process content that an analyst has not explicitly initiated. This applies to advisory drafting, evidence chain summarization, and any AI text generation triggered by content in restricted or sensitive workflow states.

---

## 3. AI Assistance Inventory

The table below enumerates every location in the platform where AI assistance is currently applied or planned. Each row specifies what the AI does, the form of its output, whether a human gate is required before the output takes effect, and whether analyst interaction with the output feeds calibration.

| Where AI Assists | What It Does | Output Type | Human Gate Required? | Calibration Source? |
|---|---|---|---|---|
| **Signal summarization** (ingestion pipeline, TS-03) | Generates a short summary of a raw ingested signal for analyst review queue display | Text — read-only summary | Yes — analyst can edit or discard before the summary is stored | No — summarization quality is not currently tracked for calibration |
| **Entity extraction** (ingestion pipeline, TS-03) | Identifies and tags named entities (organizations, technologies, people, geographies) from raw signal content | Structured tags — candidate entities with confidence scores | Yes — analyst confirms, corrects, or removes entities before record is committed | Weak — corrections to entity assignments are logged but not yet used for model updates |
| **Neighborhood classification** (CCE, TS-04) | Scores a candidate object against all active Neighborhoods using factor-weighted CCE logic | Ranked list of Neighborhood matches with per-factor scores | Yes — analyst must approve or swap the AI-selected primary Neighborhood | Yes — primary/secondary swaps and score adjustments >0.20 delta feed calibration |
| **Domain Area classification** (CCE, TS-04) | Scores a candidate object against Domain Area taxonomy using factor-weighted CCE logic | Ranked list with per-factor scores | Yes — analyst must approve or adjust | Yes — same calibration signal logic as Neighborhood |
| **Technology Type classification** (CCE, TS-04) | Applies Technology Type taxonomy scoring via CCE | Ranked list with per-factor scores | Yes — analyst must approve or adjust | Yes |
| **Candidate Type classification** (CCE, TS-04) | Assigns Candidate Type (e.g., prototype, advisory, partner_access, venture_scout) based on CCE scoring and signal recency | Single assignment with confidence score | Yes — analyst must approve; Candidate Type drives workflow routing | Yes — adjustments feed calibration for this axis independently |
| **Boundary case ambiguity notes** (CCE, TS-04) | When top two classification matches are within a defined score proximity, generates an ambiguity note explaining the competing assignments | Text — displayed inline with classification output | No gate required for display; analyst decides whether to act on the note | No — ambiguity notes are not currently calibration sources |
| **Rationale text drafting** (analyst workspace) | Drafts candidate rationale text for a classification decision based on the factor scores and signal evidence | Text — editable draft field, labeled as AI-generated | Yes — analyst must review and save; AI draft is not persisted until analyst action | Weak — substantial analyst edits to rationale text are logged |
| **Observation text drafting** (analyst workspace) | Drafts a candidate Observation entry based on the associated signal cluster and current classification state | Text — editable draft field, labeled as AI-generated | Yes — analyst must review, edit, and explicitly save | Weak — logged but not currently fed to calibration model |
| **Evidence chain summarization** (profile workspace) | Summarizes the signal evidence chain supporting a Technology Profile's classification, for display in the profile view | Text — read-only summary with source signal references | Yes — analyst must initiate this action explicitly; output is not auto-generated | No |
| **Advisory section drafting** (advisory workflow, TS-11) | Drafts one or more sections of an Advisory document based on associated Technology Profiles and analyst-approved classifications | Text — editable draft sections, labeled as AI-generated | Yes — full Advisory publication requires human approval; AI draft does not trigger any downstream state change | No — advisory drafting quality is not currently a calibration source |
| **Surprise analysis flagging** (signal archaeology) | Identifies signals that diverge significantly from expected classification patterns for an entity or Neighborhood, flagging them as potential surprises | Structured flag with explanation text | Yes — analyst must acknowledge or dismiss; flags do not auto-escalate | Weak — analyst dismissals and acknowledgments are logged |

> **Tier assignments (ADR-009):** See ADR-009 for full Tier 1/2/3 assignments. Summary: signal ingestion, deduplication, and initial cluster formation are Tier 1 (fully autonomous). Non-boundary CCE classification results with confidence ≥ 0.75 are Tier 2 (suggest + lightweight approval). Boundary case classification, Advisory section pre-population (§3.7 above), profile field pre-population, and reclassification suggestions are Tier 3 (suggest + explicit human action required).

---

## 4. Human Gates — Where AI Cannot Act Autonomously

The following prohibitions are hard constraints. They are not configurable by role, feature flag, or operational context. Any implementation that would allow an AI component to take the following actions autonomously is out of scope for Tech Range.

**AI must NOT:**

- **Publish any Advisory.** Advisory publication is a human action. AI may draft Advisory sections and surface them for analyst review, but the publish action must be triggered by an authorized user. This applies regardless of Advisory classification level or urgency.

- **Approve any match_set.** AI may set `review_status = 'system_assigned'` to indicate that a classification has been produced and is ready for analyst review. AI may not set `review_status = 'approved'`. Approval is an analyst action.

- **Create a committed Technology Profile record (pre-population of fields in a draft context is Tier 3 permitted, per ADR-009).** AI may suggest that a Profile should be created based on signal clustering or classification patterns, and may pre-populate fields in a draft context. It may not create the committed Profile record. Profile creation requires analyst initiation.

- **Access or summarize sensitive content without analyst initiation.** Any AI text generation triggered by content in a sensitive or restricted workflow state must be explicitly initiated by the analyst working the record. There is no background processing of sensitive content.

- **Override an analyst-approved classification.** Once an analyst has approved a classification on a given axis, AI may not change that classification in a subsequent processing pass, re-ingestion event, or automated workflow step. AI may surface a note indicating that new signals may be relevant to the classification, but the analyst must act on it.

- **Escalate or de-escalate an object's workflow state without a human trigger.** Workflow state transitions (e.g., moving a candidate from Watch to Tracked, or flagging a Profile for executive review) are human-initiated. AI may surface recommendations, but state changes require an explicit human action.

---

## 5. Inspectability Requirements

### 5.1 Classification outputs

For every AI-generated classification on any CCE axis (Neighborhood, Domain Area, Technology Type, Candidate Type), the analyst-facing UI must expose:

- The ranked list of top candidate matches, not just the top result
- For each candidate match: the per-factor score contribution, the factor weight applied, and the factor name in plain language
- The top competing category (second-ranked match) with its score, so analysts can evaluate how close the decision was
- The overall confidence score for the top match, with a clear label that this is a confidence score, not an analyst-verified classification

The factor score breakdown must be accessible from the classification display — not buried in a settings panel or debug view. Analysts working a record should be able to drill into the factor breakdown in one click or tap.

### 5.2 AI-generated text

Every text field that was pre-populated by AI assistance must carry a visible disclosure label: **"AI draft — review before saving."** This label must persist until the analyst takes an explicit save action, at which point the record is owned by the analyst. The provenance of the original draft (AI-generated vs. analyst-authored) is subject to the storage decision flagged in Section 9.

### 5.3 Factor score visualization

The factor score visualization is a first-class UI requirement, not a debug feature. Implementations that surface only a final confidence score without factor-level detail do not satisfy this spec. The visualization must:

- Show each contributing factor as a labeled item
- Show the factor's weight (either as a raw weight value or as a percentage of total weight)
- Show the factor's score contribution for this specific record
- Be rendered in the primary analyst workflow view, not in a separate detail panel

---

## 6. Calibration Model

The calibration model describes how analyst behavior feeds back into AI classification accuracy over time. Calibration is the mechanism by which Tech Range AI assistance improves — it requires active engagement from analysts and a disciplined review process to take effect.

### 6.1 What generates a calibration signal

The following analyst actions generate calibration signals:

| Action | Signal Type | Weight |
|---|---|---|
| Analyst swaps primary classification to a different category (any CCE axis) | Strong negative signal for original assignment; strong positive signal for new assignment | High |
| Analyst adjusts a factor score with a delta > 0.20 on a 0–1 scale | Directional signal for that factor's weight | Medium |
| Analyst approves a classification without making any changes | Weak positive signal for the AI's current assignment | Low |
| Analyst approves a classification after minor adjustment (delta ≤ 0.20) | Neutral-to-weak positive signal | Low |

Analyst corrections to AI-drafted text are logged but do not currently feed the calibration model for text generation. This may be revisited as text generation quality metrics are established.

### 6.2 Where calibration data is stored

Calibration event storage and the data structures supporting it are deferred to TS-10 (Observability and Data Quality). At minimum, each calibration event must carry: the record ID, the CCE axis, the original AI assignment, the analyst's final assignment, the delta (if applicable), the analyst user ID, and a timestamp. The model_version in effect at the time of the AI assignment must also be recorded (see Section 7.2).

### 6.3 Calibration cycle

Weight updates are not applied in real time. The calibration cycle is:

1. **Monthly accumulation:** Calibration signals are accumulated over a rolling calendar month.
2. **Weight review:** At the end of each month, a designated reviewer (analyst lead or platform owner) reviews the accumulated calibration signals and proposed weight adjustments.
3. **Sign-off requirement:** Weight adjustments must be explicitly approved before they take effect. No automated weight update occurs without human sign-off. [Internal decision rule — sign-off authority and quorum requirements to be defined in operational runbook]
4. **Deployment:** Approved weight changes are applied to the CCE configuration and logged with the reviewer's identity and timestamp.

### 6.4 Calibration scope

Factor weights are adjusted per-axis independently. A calibration signal on the Neighborhood axis does not affect factor weights on the Domain Area, Technology Type, or Candidate Type axes. This prevents cross-axis interference and makes calibration failures easier to diagnose.

---

## 7. AI Model Governance

### 7.1 Which capabilities require a defined model version

The following AI capabilities require an explicitly tracked model version at deployment time:

- **CCE classification scoring** — the factor-weighting logic and any ML components contributing to score calculation
- **Text generation** — any AI component used for rationale drafting, observation drafting, advisory section drafting, signal summarization, and evidence chain summarization

Capabilities that are implemented as deterministic rule-based logic (e.g., entity extraction based on a fixed taxonomy lookup) do not require model versioning under this spec, but must carry a configuration version that is similarly tracked.

### 7.2 Model version tracking

Every record that carries an AI-generated output must include a `model_version` field. This field must be set at the time the AI output is generated and must not be overwritten when the analyst edits the output. The `model_version` field must be:

- Populated for all records generated after TS-12 implementation (backfill of historical records is out of scope)
- Included in calibration event records (Section 6.2)
- Queryable for post-hoc analysis of classification accuracy by model version

`model_version` format: `[capability]:[version_string]` — for example, `cce-neighborhood:1.2.0` or `text-gen:2024-11`. Exact versioning scheme is [Internal decision rule — to be defined before first model deployment].

### 7.3 Model update process

Model updates follow a staged rollout process:

1. **Baseline establishment:** Before a new model version is considered, a baseline accuracy score is computed against the analyst-approved ground truth corpus for the relevant CCE axis or text generation task.
2. **Staging deployment:** The new model version is deployed to a staging environment and run against the same ground truth corpus. Accuracy comparison is produced.
3. **A/B comparison:** For a defined evaluation period, the new model version is run in shadow mode alongside the current version on live records. Analyst approval rates and correction rates are compared between model versions.
4. **Sign-off:** A designated reviewer approves the new model version for production deployment based on the A/B comparison results. [Internal decision rule — approval authority to be defined in operational runbook]
5. **Production deployment:** New model version is set as active; `model_version` field for new records reflects the updated version.

### 7.4 Rollback

If a deployed model version produces a measurable degradation in analyst approval rates, correction frequency, or classification accuracy against ground truth, the previous model version can be restored. Rollback must:

- Be triggerable by a designated platform owner without a full deployment cycle
- Restore the previous `model_version` value for new records
- Not retroactively change `model_version` on already-generated records
- Be logged with the reason, the triggering reviewer identity, and a timestamp

---

## 8. Failure Modes and Fallbacks

AI components will fail. This section defines the expected behavior when they do, so that failures degrade gracefully and do not block analyst workflows.

### 8.1 Classification confidence below threshold

If the CCE produces a top-match confidence score below a defined minimum threshold, the system must:

- Assign the record a Candidate Type of **Emerging** by default
- Create an entry in the analyst review queue, flagged as low-confidence
- Display the low-confidence flag to the analyst in the queue view
- Not block ingestion — the record proceeds through the pipeline in a pending-review state

[Internal decision rule — minimum confidence threshold value is a tunable configuration parameter, not hardcoded in this spec]

### 8.2 AI text generation failure

If an AI text generation call fails (timeout, model unavailability, API error):

- The affected text field is surfaced to the analyst as a blank field with the label: **"AI assistance unavailable — please enter manually."**
- The analyst workflow is not blocked. The field remains editable.
- The failure is logged for observability (deferred to TS-10 for full failure logging spec).
- No retry is automatically attempted in the analyst-facing session. Retry logic, if implemented, is a background process that does not surface to the analyst mid-workflow.

### 8.3 Insufficient calibration data

If the accumulated calibration signal volume for a given CCE axis does not meet the minimum threshold for a weight update in a given monthly cycle:

- The current factor weights are held unchanged.
- The monthly weight review proceeds but concludes with no update.
- The accumulated signals are carried forward to the next cycle (signals are not discarded).
- The reviewer is notified that data volume was insufficient and the update was skipped.

> **DECISION NEEDED —** What is the minimum calibration signal volume required before a weight update is permitted for a given CCE axis? *Recommendation: Define a minimum of 30–50 analyst correction events per axis per cycle before allowing weight updates, to prevent noise from small sample sets driving spurious changes. Exact number should be validated against expected analyst throughput once the platform is in active use.*

---

## 9. DECISION NEEDED Items

The following decisions are required before TS-12 can be considered implementation-ready. They are flagged here because they affect system architecture, vendor selection, or data model choices that will be difficult to reverse.

> **DECISION NEEDED —** Which LLM or model powers AI text generation in Tech Range (rationale drafting, observation drafting, advisory section drafting, signal summarization, evidence chain summarization) — is this an internally hosted model, a third-party API (e.g., Anthropic, OpenAI), or a hybrid? *Recommendation: Given the defense/national security context and sensitivity of advisory content, a strong default is an internally hosted or air-gapped model for any text generation that operates on classified or operationally sensitive content. External API usage may be acceptable for lower-sensitivity summarization tasks (e.g., open-source signal summarization), but this requires an explicit data classification boundary decision. This decision blocks vendor contracting and infrastructure design.*

> **DECISION NEEDED —** Is AI-generated text stored with a separate provenance flag in the data model, or is it stored inline as analyst-owned text once the analyst saves the field? *Recommendation: Store a provenance flag (`ai_drafted: true/false`) alongside the text field, persisted even after analyst edits. This preserves the ability to audit AI contribution to published content, supports future text generation quality analysis, and is consistent with the inspectability principle in Section 2.2. The alternative (treating AI draft as analyst-owned on save) loses provenance permanently and limits retrospective analysis. Implementation note: the flag records the origin of the initial draft, not whether the analyst changed it — a separate `analyst_edited` flag or diff mechanism would be needed to track the degree of modification.*

> **DECISION NEEDED —** What is the minimum calibration data volume before a weight update is permitted for a given CCE axis in a monthly cycle? *(Also flagged in Section 8.3.) Recommendation: 30–50 confirmed analyst correction events per axis. This threshold should be treated as a configurable parameter in the CCE configuration layer, not hardcoded, so it can be adjusted as analyst throughput data becomes available.*

---

*TS-12 Draft — for internal review. This spec informs ADR-009 (AI Autonomy and Human Oversight Boundaries — see ADR-009 (Accepted)). DECISION NEEDED items must be resolved before implementation begins.*
