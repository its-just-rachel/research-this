# ADR-008: Workflow State Model

**Status:** Accepted
**Date:** 2026-07-06
**Deciders:** Platform Architecture Team

---

## Context

Tech Range is a five-stage intelligence pipeline — Surface → Research → Solution → Prototype → Advise — through which objects of several types travel before reaching analysts and clients. At any given moment, multiple analysts may be working in parallel on the same or adjacent objects. Without a formal state model, the system devolves into "whatever the last person clicked": signals get silently dropped, clusters get promoted before corroboration is complete, findings get published without peer review, and there is no audit trail showing who moved what and why.

The coordination failures this creates are concrete:

- **Objects get stuck.** A signal sits in `triaged` for three weeks because no analyst picked it up and nothing escalated it.
- **Gates get bypassed.** A finding moves from `draft` to `published` because a UI check was the only enforcement and the API was called directly.
- **History is lost.** A technology profile was marked `archived` six months ago; no one recorded why, who did it, or whether it is safe to revive.
- **Parallel edits corrupt state.** Two analysts each approve different transitions on the same object; the last write wins and the other is silently discarded.

The platform must therefore enforce a formal, server-side state machine for every major object type. The state model must:

1. Define all valid states and the transitions between them.
2. Declare for each transition who or what may trigger it and what preconditions must be satisfied.
3. Enforce these rules at the API layer — not just in the UI — so that no client, script, or direct API call can produce an illegal state transition.
4. Log every transition with full actor and timestamp context in an append-only audit trail.
5. Require human approval at gates where analyst judgment or sign-off is mandatory.

---

## Decision

We will implement **explicit, per-object-type state machines** with the following architecture:

1. **Each object table** carries a `current_state` column (enum or varchar with a check constraint) that reflects the object's live state.
2. **A `state_transitions` log table** records every transition in append-only fashion (see schema below).
3. **A `WorkflowStateService`** (server-side, not client-side) holds the authoritative transition graph for each object type. All state changes are routed through this service. Direct updates to `current_state` outside of the service are not permitted.
4. **API endpoints** that change object state call `WorkflowStateService.transition(objectType, objectId, toState, actor)`, which validates preconditions, writes the new state, and writes the log record atomically in a single database transaction.
5. **The UI** may show or hide transition controls based on state, but it is not the enforcement layer. The API will reject illegal transitions regardless of what the UI presents.

---

## State Machine Definitions

### Signal

| State | Description |
|-------|-------------|
| `new` | Ingested from a source feed; not yet reviewed |
| `triaged` | Reviewed by an analyst; classified and tagged |
| `grouped` | Associated with a Corroboration Group |
| `archived` | Retained for record; no further action |
| `rejected` | Determined to be noise or out of scope |

**Transitions:**

| From | To | Trigger | Preconditions | Side Effects |
|------|----|---------|---------------|--------------|
| `new` | `triaged` | analyst | Signal has been opened and reviewed | Assigns analyst; sets `triaged_at` |
| `new` | `rejected` | analyst | — | Sets `rejected_at`, `rejected_by` |
| `triaged` | `grouped` | system or analyst | Corroboration Group exists and accepts signal | Writes `corroboration_group_id` |
| `triaged` | `archived` | analyst | — | Sets `archived_at` |
| `triaged` | `rejected` | analyst | — | Sets `rejected_at`, `rejected_by` |
| `grouped` | `triaged` | analyst | Group is dissolved or signal removed from group | Clears `corroboration_group_id` |
| `archived` | `triaged` | analyst | See DECISION NEEDED #3 | Sets `reopened_at` |

**Human gate required:** `new → rejected` (prevents inadvertent bulk rejection by automation).

---

### Corroboration Group

| State | Description |
|-------|-------------|
| `forming` | Group created; signals being added |
| `active` | Group has sufficient signals to be analytically meaningful |
| `closed` | Analysis complete; group locked |
| `dissolved` | Group determined to be invalid or redundant |

**Transitions:**

| From | To | Trigger | Preconditions | Side Effects |
|------|----|---------|---------------|--------------|
| `forming` | `active` | system or analyst | Signal count ≥ configured threshold | Emits `group.activated` event |
| `forming` | `dissolved` | analyst | — | Returns member signals to `triaged` |
| `active` | `closed` | analyst | At least one Cluster references this group | Sets `closed_at` |
| `active` | `dissolved` | approver | All member signals have alternative assignment or are archived | Returns signals to `triaged` |
| `closed` | `active` | analyst | See DECISION NEEDED #3 | Reopens for additional signals |

**Human gate required:** `active → dissolved` (data loss risk; requires approver role).

---

### Cluster

| State | Description |
|-------|-------------|
| `candidate` | Auto-detected by clustering algorithm; awaiting analyst review |
| `active` | Confirmed by analyst; being developed into a Technology Profile |
| `promoted` | A Technology Profile has been created from this cluster |
| `dissolved` | Cluster rejected or merged into another |

**Transitions:**

| From | To | Trigger | Preconditions | Side Effects |
|------|----|---------|---------------|--------------|
| `candidate` | `active` | analyst | Analyst has reviewed constituent signals | Sets `confirmed_by`, `confirmed_at` |
| `candidate` | `dissolved` | analyst | — | Sets `dissolved_reason` |
| `active` | `promoted` | system | Technology Profile created and links this cluster | Writes `technology_profile_id` |
| `active` | `dissolved` | analyst | No Technology Profile yet created | Sets `dissolved_reason` |
| `promoted` | `active` | approver | Technology Profile is reverted to `candidate` | Clears `technology_profile_id`; See DECISION NEEDED #3 |

**Human gate required:** `candidate → active` (confirms algorithmic output before downstream work begins).

---

### Technology Profile

| State | Description |
|-------|-------------|
| `candidate` | Stub created from a promoted Cluster; not yet researched |
| `emerging` | Research underway; knowledge base being populated |
| `active` | Sufficiently documented; available for Solution matching |
| `under_review` | Flagged for review due to new signals or staleness |
| `archived` | No longer actively tracked |

**Transitions:**

| From | To | Trigger | Preconditions | Side Effects |
|------|----|---------|---------------|--------------|
| `candidate` | `emerging` | analyst | Profile has been assigned to a researcher | Sets `researcher_id` |
| `emerging` | `active` | analyst | Knowledge completeness score ≥ threshold | Emits `profile.activated` event |
| `active` | `under_review` | system or analyst | Staleness timer exceeded or analyst flags it | Sets `review_triggered_at`, `review_reason` |
| `under_review` | `active` | analyst | Review complete; profile confirmed current | Sets `reviewed_at`, `reviewed_by` |
| `under_review` | `emerging` | analyst | Significant gaps found; needs additional research | — |
| `active` | `archived` | approver | No active clusters or solution candidates depend on it | Sets `archived_at` |
| `archived` | `candidate` | approver | See DECISION NEEDED #3 | Resets completeness scores; emits revival event |

**Allowed back-transitions:** `under_review → emerging`, `archived → candidate` (with approver role and documented reason).

**Human gate required:** `active → archived` (prevents accidental loss of active intelligence assets).

---

### Solution Candidate

| State | Description |
|-------|-------------|
| `draft` | Being authored; not yet submitted for review |
| `under_review` | Submitted; awaiting approver sign-off |
| `approved` | Cleared for handoff to Prototype stage |
| `handed_off` | Accepted by Prototype team |
| `rejected` | Not approved; may be revised and resubmitted |

**Transitions:**

| From | To | Trigger | Preconditions | Side Effects |
|------|----|---------|---------------|--------------|
| `draft` | `under_review` | analyst | All required fields populated; linked Technology Profile is `active` | Notifies approver queue |
| `under_review` | `approved` | approver | — | Sets `approved_by`, `approved_at` |
| `under_review` | `rejected` | approver | Rejection reason recorded | Notifies submitting analyst |
| `under_review` | `draft` | analyst | Analyst withdraws submission for revision | — |
| `approved` | `handed_off` | system or analyst | Prototype record created and linked | Writes `prototype_id` |
| `rejected` | `draft` | analyst | See DECISION NEEDED #3 | Clears `rejected_at`; increments revision counter |

**Human gate required:** `under_review → approved` and `under_review → rejected` (both require approver role).

---

### Prototype

| State | Description |
|-------|-------------|
| `scoped` | Prototype spec written; not yet started |
| `in_progress` | Active development or experimentation underway |
| `complete` | Prototype finished and outcomes documented |
| `abandoned` | Work stopped before completion |

**Transitions:**

| From | To | Trigger | Preconditions | Side Effects |
|------|----|---------|---------------|--------------|
| `scoped` | `in_progress` | analyst | Assigned engineer or researcher confirmed | Sets `started_at` |
| `scoped` | `abandoned` | approver | — | Sets `abandoned_reason`, `abandoned_at` |
| `in_progress` | `complete` | analyst | Outcomes documented; linked Finding exists or is created | Sets `completed_at` |
| `in_progress` | `abandoned` | approver | — | Sets `abandoned_reason`, `abandoned_at` |
| `complete` | `in_progress` | analyst | See DECISION NEEDED #3; additional validation required | Increments iteration counter |

**Human gate required:** `in_progress → abandoned` (prevents silent project termination without approver sign-off).

---

### Finding

| State | Description |
|-------|-------------|
| `draft` | Being written; not yet submitted |
| `peer_review` | Submitted for peer review |
| `published` | Live and client-accessible |
| `withdrawn` | Removed from circulation after publication |

**Transitions:**

| From | To | Trigger | Preconditions | Side Effects |
|------|----|---------|---------------|--------------|
| `draft` | `peer_review` | analyst | Finding linked to a `complete` Prototype or `active` Technology Profile | Notifies reviewer queue |
| `peer_review` | `published` | approver | Reviewer sign-off recorded | Sets `published_at`; emits `finding.published` event |
| `peer_review` | `draft` | analyst | Reviewer returns for revision | Records reviewer comments |
| `published` | `withdrawn` | approver | Withdrawal reason documented | Sets `withdrawn_at`, `withdrawn_reason`; suppresses from client views |
| `withdrawn` | `draft` | approver | See DECISION NEEDED #3 | Creates revision record |

**Human gate required:** `peer_review → published` and `published → withdrawn` (both require approver role; publication and withdrawal are high-visibility actions).

---

### Advisory

| State | Description |
|-------|-------------|
| `assembling` | Being constructed from Findings; not yet submitted |
| `review` | Submitted for editorial and compliance review |
| `approved` | Cleared for publication |
| `published` | Live and client-accessible |
| `withdrawn` | Removed from circulation |

**Transitions:**

| From | To | Trigger | Preconditions | Side Effects |
|------|----|---------|---------------|--------------|
| `assembling` | `review` | analyst | At least one `published` Finding attached | Notifies approver queue |
| `review` | `approved` | approver | All attached Findings are `published` | Sets `approved_by`, `approved_at` |
| `review` | `assembling` | analyst | Analyst recalls for revision | Records recall reason |
| `approved` | `published` | system or analyst | Scheduled publish date reached or manual trigger | Sets `published_at`; emits `advisory.published` event |
| `published` | `withdrawn` | approver | Withdrawal reason documented | Sets `withdrawn_at`; suppresses from client distribution |
| `withdrawn` | `assembling` | approver | See DECISION NEEDED #3 | Creates new revision; links original as predecessor |

**Human gate required:** `review → approved`, `published → withdrawn`, `withdrawn → assembling` (all require approver role).

---

## Storage Schema

### `current_state` on object tables

Each object table carries a `current_state` column. Example for `signals`:

```sql
ALTER TABLE signals
  ADD COLUMN current_state VARCHAR(32) NOT NULL DEFAULT 'new',
  ADD CONSTRAINT signals_current_state_check
    CHECK (current_state IN ('new', 'triaged', 'grouped', 'archived', 'rejected'));
```

The check constraint is a last-resort guard only. The `WorkflowStateService` is the primary enforcement layer and will reject invalid transitions before they reach the database.

### `state_transitions` log table

```sql
CREATE TABLE state_transitions (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  object_id        UUID        NOT NULL,
  object_type      VARCHAR(64) NOT NULL,  -- e.g. 'signal', 'cluster', 'technology_profile'
  from_state       VARCHAR(32),           -- NULL on object creation
  to_state         VARCHAR(32) NOT NULL,
  actor_id         UUID,                  -- NULL for system-triggered transitions
  actor_type       VARCHAR(16) NOT NULL,  -- 'system' | 'user'
  actor_role       VARCHAR(32),           -- 'analyst' | 'approver' | 'system' etc.
  reason           TEXT,                  -- optional; required for certain transitions
  metadata         JSONB,                 -- additional context (e.g. precondition snapshot)
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX state_transitions_object_idx
  ON state_transitions (object_type, object_id, created_at DESC);
```

This table is append-only. No `UPDATE` or `DELETE` is permitted on it. Row-level security should prevent even service-role deletes except via an explicit `purge_audit_log` admin procedure.

### Atomicity guarantee

Every state transition executes in a single database transaction:

```
BEGIN
  1. Read current_state of object (with SELECT FOR UPDATE)
  2. Validate transition via WorkflowStateService
  3. UPDATE object SET current_state = to_state
  4. INSERT INTO state_transitions (...)
COMMIT
```

If any step fails, the transaction rolls back and the object state is unchanged.

---

## Human Gates vs. System-Automatable Transitions

| Transition | Gate Type | Notes |
|-----------|-----------|-------|
| `new → rejected` (Signal) | Human (analyst) | Prevents bulk auto-rejection |
| `candidate → active` (Cluster) | Human (analyst) | Confirms algorithmic output |
| `active → dissolved` (Corroboration Group) | Human (approver) | Data loss risk |
| `under_review → approved` (Solution Candidate) | Human (approver) | Quality gate |
| `under_review → rejected` (Solution Candidate) | Human (approver) | Quality gate |
| `in_progress → abandoned` (Prototype) | Human (approver) | Project governance |
| `peer_review → published` (Finding) | Human (approver) | Client-visible; high stakes |
| `published → withdrawn` (Finding) | Human (approver) | Client-visible; high stakes |
| `review → approved` (Advisory) | Human (approver) | Final publication gate |
| `published → withdrawn` (Advisory) | Human (approver) | Client distribution impact |
| `forming → active` (Corroboration Group) | System (threshold-based) | Auto-promotes when signal count met |
| `active → promoted` (Cluster) | System | Triggered by Technology Profile creation |
| `approved → published` (Advisory) | System or analyst | Can be scheduled or manual |
| `active → under_review` (Technology Profile) | System (staleness timer) | Can also be analyst-triggered |

---

## Consequences

**Positive:**

- State integrity is guaranteed at the API layer; no client can produce an illegal transition regardless of how it calls the API.
- Every state change is auditable: who, what, when, and why.
- Human gates are formally declared and enforced, not informally relied upon.
- The append-only log enables full timeline reconstruction for any object.
- Stuck-object escalation (see DECISION NEEDED #2) becomes tractable because state entry timestamps are always recorded.

**Negative / Trade-offs:**

- All state-changing code paths must be routed through `WorkflowStateService`, adding a layer of indirection that developers must learn to use consistently.
- Adding new states or transitions requires a code change (if transitions are hardcoded) or a migration (if configurable), not a simple config edit.
- The `state_transitions` table will grow continuously and requires a retention or archival strategy for long-running deployments.
- Reverting an incorrect transition requires an explicit compensating transition (not a simple undo), which must itself be logged — this is the correct behavior but adds operational overhead when mistakes occur.

---

## Decision Needed

> **DECISION NEEDED —** Should valid transition rules be hardcoded in the `WorkflowStateService` source code, or stored in a configurable `workflow_transition_rules` table that can be updated without a code deployment? *Recommendation: Hardcode in service code for the initial release. Transition rules encode business logic and change infrequently; keeping them in code means they are version-controlled, reviewable in PRs, and testable. Introduce a configurable layer only if operational experience shows a genuine need for runtime rule changes (e.g., multi-tenant deployments where each tenant has different approval workflows). Premature configurability adds schema complexity and a new attack surface for misconfiguration.*

> **DECISION NEEDED —** What is the escalation policy for objects that remain in a given state beyond an acceptable time threshold (e.g., a Signal stuck in `triaged` for 14 days, a Solution Candidate stuck in `under_review` for 7 days)? Should the system auto-escalate to a team lead, auto-transition to an escalated state, or simply emit an alert? *Recommendation: Emit a timed alert (Slack notification + in-app banner) to the assigned analyst and their team lead after a configurable threshold, without auto-transitioning state. Auto-transition risks moving objects past human gates without review; alerts preserve human control while ensuring visibility. Define per-object-type thresholds in configuration (not hardcoded) so they can be tuned without a deployment.*

> **DECISION NEEDED —** Should rejected or archived objects be revivable, and if so, under what conditions and by whom? Several state machines include back-transitions from terminal states (e.g., `archived → candidate` for Technology Profile, `rejected → draft` for Solution Candidate). Should these be universally allowed for users with the `approver` role, or should revival require an explicit written justification that is reviewed asynchronously? *Recommendation: Allow revival for users with the `approver` role, but require a non-empty `reason` field in the transition call — enforced by the `WorkflowStateService` as a precondition. Do not require an asynchronous review step for revival; the audit log provides accountability. Restrict revival of `published → withdrawn` Advisories and Findings to `admin` role only, since those objects may have been shared with clients and revival has external-facing implications.*
