# TS-06 — Workflow State Machines

**Version:** 0.1  
**Status:** Draft  
**Relates to:** ADR-008, DS-08 (Governance and Review Queues)  
**Pipeline stages covered:** Surface → Research → Solution → Prototype → Advise

---

## 1. Purpose

Every object in the Tech Range pipeline moves through a defined lifecycle. This spec formalizes those lifecycles as explicit state machines, defines the server-side service that enforces them, and specifies the storage layer that makes every transition auditable and recoverable.

The core design decision (ADR-008): state is always authoritative on the object's own row (`current_state`), and every change is appended to a single `state_transitions` log. No client can produce an illegal state change. All transition logic routes through `WorkflowStateService`.

This spec covers mechanics only. Review queue UI is in DS-08. CCE (Comparative Classification Engine) integration is referenced here for side-effect wiring but is not fully specified in this document.

---

## 2. WorkflowStateService

### 2.1 Responsibilities

- **Validate transitions.** For a given `(objectType, fromState, toState)` triple, confirm the transition exists in the defined machine for that object type. Reject anything not in the transition table.
- **Check preconditions.** Before executing a transition, evaluate all preconditions for that arc. If any precondition fails, return a `ValidationError` with a machine-readable code and human-readable message.
- **Authorize the actor.** Confirm the `actorId` holds the role required for the transition. System-initiated transitions carry `actorType = 'system'`; human-initiated transitions must match the required role.
- **Execute side effects.** After a successful write, trigger downstream consequences (CCE re-runs, queue entry creation, notifications, downstream object creation) in the same transaction where possible, or via outbox pattern where the side effect is asynchronous.
- **Write the audit log.** Append a row to `state_transitions` as part of the same database transaction as the `current_state` update.
- **Emit events.** After the transaction commits, publish a `workflow.transition` event to the internal event bus for any listener (analytics, notifications, external integrations).

### 2.2 Interface

```typescript
transition(
  objectId:   string,          // UUID of the object being transitioned
  objectType: ObjectType,      // 'signal' | 'corroboration_group' | 'cluster' | ...
  toState:    string,          // target state name
  actorId:    string | null,   // UUID of user or system process; null only for scheduler
  actorType:  ActorType,       // 'system' | 'analyst' | 'senior_analyst' | 'architect' | 'scheduler'
  reason:     string | null,   // free-text rationale; required for human-gated transitions
  metadata?:  Record<string, unknown>  // optional structured payload (e.g. review outcome, score)
): Promise<TransitionResult>
```

**`TransitionResult`**

```typescript
type TransitionResult =
  | { success: true;  transition: StateTransitionRecord }
  | { success: false; error: ValidationError }
```

**`ValidationError` codes**

| Code | Meaning |
|---|---|
| `ILLEGAL_TRANSITION` | `(fromState, toState)` arc does not exist for this object type |
| `PRECONDITION_FAILED` | One or more preconditions not met; `details` array lists which |
| `AUTHORIZATION_FAILED` | `actorType` / `actorId` not permitted for this transition |
| `OBJECT_NOT_FOUND` | No object with given `objectId` and `objectType` |
| `BOUNDARY_CASE_BLOCKING` | Object has unresolved boundary cases that must be resolved first |
| `CONCURRENT_TRANSITION` | Optimistic lock conflict; caller should retry |

### 2.3 Pre-condition Checking

Preconditions are evaluated in order. The first failure short-circuits and returns immediately.

**General preconditions (all objects)**

1. Object exists and its `current_state` matches the expected `fromState` (optimistic lock via `current_state` in the `WHERE` clause of the UPDATE).
2. No unresolved blocking boundary cases on this object (see Section 6 for which transitions are blocked).
3. Actor has the required role for this transition arc.

**Object-specific preconditions** are defined per-transition in Section 4.

Preconditions are implemented as composable predicate functions registered in a `PreconditionRegistry`. Each predicate receives `(objectId, objectType, toState, actorId, context)` and returns `{ passed: boolean, reason?: string }`.

### 2.4 Side Effect Execution

Side effects are categorized by execution timing:

**Synchronous (within transaction)**

- Update `current_state` on the object row.
- Insert row into `state_transitions`.
- Insert or update `review_queue` entry if the transition creates or closes a queue item.

**Asynchronous via outbox pattern**

- Trigger CCE re-run (e.g., when a Signal moves to `triaged`, CCE should re-evaluate grouping candidates).
- Create downstream objects (e.g., promoting a Cluster to `promoted` triggers creation of a Technology Profile candidate row).
- Send notifications to assigned analysts or queue owners.
- Publish `workflow.transition` event to event bus.

The outbox table (`workflow_side_effects`) stores pending async work. A background worker polls and executes it, with retry and dead-letter handling. This means the caller gets a committed transition even if, say, the CCE job queue is temporarily unavailable.

### 2.5 Atomicity Guarantee

The following operations execute in a single database transaction:

1. `UPDATE <object_table> SET current_state = $toState WHERE id = $objectId AND current_state = $fromState`
2. `INSERT INTO state_transitions (...)`
3. `INSERT INTO review_queue (...)` — if transition creates a queue entry
4. `UPDATE review_queue SET resolved_at = now() ...` — if transition resolves a queue entry
5. `INSERT INTO workflow_side_effects (...)` — outbox rows for async work

If the `UPDATE` in step 1 affects zero rows (because `current_state` changed under us), the transaction is rolled back and `CONCURRENT_TRANSITION` is returned.

---

## 3. Storage

### 3.1 `current_state` on Object Tables

Each object table carries a `current_state` column with a `CHECK` constraint listing valid states for that type. Example for `signals`:

```sql
ALTER TABLE signals
  ADD COLUMN current_state TEXT NOT NULL DEFAULT 'new'
  CHECK (current_state IN ('new', 'triaged', 'grouped', 'archived', 'rejected'));
```

The set of valid states is defined by the state machine for each object type (Section 4). CHECK constraints are the database-level guard against corrupt state values; `WorkflowStateService` is the application-level guard against illegal transitions.

### 3.2 `state_transitions` Log Table

```sql
CREATE TABLE state_transitions (
  id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  object_id    UUID        NOT NULL,
  object_type  TEXT        NOT NULL,
  from_state   TEXT,                   -- NULL for initial/creation transition
  to_state     TEXT        NOT NULL,
  actor_id     UUID,                   -- NULL for system-initiated
  actor_type   TEXT        NOT NULL
    CHECK (actor_type IN ('system', 'analyst', 'senior_analyst', 'architect', 'scheduler')),
  actor_role   TEXT,                   -- denormalized role at time of transition
  reason       TEXT,
  metadata     JSONB       NOT NULL DEFAULT '{}',
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Column notes**

- `from_state` is NULL only for the initial creation transition (when an object first enters the pipeline with no prior state).
- `actor_id` is NULL when `actor_type = 'system'` or `'scheduler'`.
- `actor_role` is denormalized from the user record at transition time so the audit log is self-contained and survives role changes.
- `metadata` carries structured payload — CCE scores, review outcomes, boundary case IDs, etc.

This table is append-only. No rows are ever updated or deleted. Deletion or correction must be modeled as a compensating transition.

### 3.3 Indexes

```sql
-- Primary audit query: all transitions for a given object
CREATE INDEX idx_state_transitions_object
  ON state_transitions (object_id, object_type, created_at DESC);

-- Queue / dashboard queries: all transitions of a given type within a time range
CREATE INDEX idx_state_transitions_object_type_created
  ON state_transitions (object_type, created_at DESC);

-- Actor audit: all transitions performed by a given user
CREATE INDEX idx_state_transitions_actor
  ON state_transitions (actor_id, created_at DESC)
  WHERE actor_id IS NOT NULL;

-- To-state queries (e.g. "find all objects recently promoted")
CREATE INDEX idx_state_transitions_to_state
  ON state_transitions (object_type, to_state, created_at DESC);
```

---

## 4. State Machines — Full Definitions

### 4.1 Signal

A Signal is a raw incoming intelligence item from any ingestion source.

**State diagram**

```
              ┌─────────┐
     create   │         │
  ──────────► │   new   │
              │         │
              └────┬────┘
                   │ CCE run complete,
                   │ review_status set
                   ▼
              ┌─────────┐
              │ triaged  │
              └────┬────┘
                   │
          ┌────────┴────────┐
          │                 │
          │ assigned to      │ no viable group,
          │ group            │ low confidence
          ▼                 ▼
     ┌─────────┐      ┌──────────┐
     │ grouped  │      │ archived │
     └──────────┘      └──────────┘
          │
          │ analyst manual
          │ rejection
          ▼
     ┌──────────┐
     │ rejected │
     └──────────┘
```

**Transition table**

| From State | To State | Trigger | Actor Required | Preconditions | Side Effects |
|---|---|---|---|---|---|
| *(none)* | `new` | Ingestion pipeline creates signal | `system` | None | Enqueue for CCE run |
| `new` | `triaged` | CCE run completes | `system` | CCE run record exists; `review_status` set on signal row | If boundary case flagged, create `review_queue` entry |
| `triaged` | `grouped` | CCE assigns signal to a Corroboration Group | `system` | Target Corroboration Group exists and is in `forming` or `active` state; signal `review_status != 'blocked'` | Update group member count; if group reaches minimum, trigger group → `active` transition |
| `triaged` | `archived` | Confidence below retention threshold (automated) or analyst decision | `system` or `analyst` | Signal age > retention window OR analyst provides reason | No downstream objects remain linked to this signal |
| `triaged` | `rejected` | Analyst marks as noise/duplicate | `analyst` | `reason` must be provided | Log rejection reason; excluded from future CCE grouping |
| `grouped` | `archived` | Parent group dissolved; signal not reassigned | `system` | Parent group is in `dissolved` state; no replacement group assigned | See Section 7 orphan handling |

**"Triaged" definition:** A Signal is triaged when CCE has processed it, a `review_status` of `pending`, `clear`, or `boundary` has been set, and the signal is ready for grouping decisions. CCE may flag a signal as a boundary case (diff_strength < 0.15), which creates a queue entry but does not block the `triaged` state itself — it blocks the subsequent `grouped` transition.

---

### 4.2 Corroboration Group

A Corroboration Group is a set of signals that collectively corroborate a shared technology observation.

**State diagram**

```
              ┌─────────┐
    create    │         │
  ──────────► │ forming  │
              │         │
              └────┬────┘
                   │ member count
                   │ ≥ minimum
                   ▼
              ┌─────────┐
              │  active  │◄──────────────────┐
              └────┬────┘                   │
                   │                        │ new signals
          ┌────────┴────────┐               │ added
          │                 │               │
     analyst          all members           │
     closes            archived /           │
     group             rejected             │
          │                 │               │
          ▼                 ▼               │
     ┌────────┐       ┌──────────┐          │
     │ closed │       │ dissolved │──────────┘
     └────────┘       └──────────┘
                       (remaining signals become orphaned — see §7)
```

**Transition table**

| From State | To State | Trigger | Actor Required | Preconditions | Side Effects |
|---|---|---|---|---|---|
| *(none)* | `forming` | CCE creates group from first signal match | `system` | At least one signal assigned | Assign initial member signals |
| `forming` | `active` | Member count reaches minimum threshold | `system` | Member count ≥ 3 (see DECISION NEEDED §7.1) | Trigger Cluster coherence re-evaluation; notify assigned analyst |
| `forming` | `dissolved` | No additional signals arrive within formation window | `scheduler` | Formation window elapsed; member count still below minimum | Orphan any assigned signals (see §7) |
| `active` | `closed` | Analyst manually closes group (evidence incorporated) | `analyst` | Linked Cluster or Technology Profile exists capturing this group's evidence | Create `state_transitions` entry with reason; remove from active queue |
| `active` | `dissolved` | All member signals archived or rejected | `system` | Zero signals in `grouped` state remain in this group | Orphan handling per §7 |

**Minimum member count:** 3 signals required before a group transitions from `forming` to `active`. This is a system constant but see DECISION NEEDED §7.1 for whether this should be configurable per signal source type.

**What dissolves a group:** Either the scheduler fires after the formation window expires with insufficient members, or all member signals exit the `grouped` state via archive/reject. In both cases, the service runs orphan handling before marking the group `dissolved`.

---

### 4.3 Cluster

A Cluster is a coherence grouping of Corroboration Groups pointing at a common technology signal space.

**State diagram**

```
              ┌───────────┐
    create    │           │
  ──────────► │ candidate │
              │           │
              └─────┬─────┘
                    │ coherence score
                    │ ≥ threshold
                    ▼
              ┌───────────┐
              │   active  │
              └─────┬─────┘
                    │
          ┌─────────┴─────────┐
          │                   │
     coherence            analyst
     score drops           promotes
     below floor           cluster
          │                   │
          ▼                   ▼
     ┌──────────┐       ┌──────────┐
     │ dissolved │       │ promoted │
     └──────────┘       └──────────┘
                         (Technology Profile
                          candidate created)
```

**Transition table**

| From State | To State | Trigger | Actor Required | Preconditions | Side Effects |
|---|---|---|---|---|---|
| *(none)* | `candidate` | CCE identifies coherent group set | `system` | ≥ 2 Corroboration Groups with shared coherence signal; initial coherence score computed | None |
| `candidate` | `active` | CCE coherence score exceeds activation threshold | `system` | Coherence score ≥ 0.65 (configurable); ≥ 2 active member groups | Notify analyst; add to analyst review queue |
| `candidate` | `dissolved` | Coherence score never reached threshold; member groups dissolved | `system` or `scheduler` | All member groups in `closed` or `dissolved` state OR formation window elapsed | No Technology Profile created |
| `active` | `promoted` | Analyst promotes cluster to Technology Profile suggestion | `analyst` | Coherence score still above floor (≥ 0.40); analyst provides reason; no existing Technology Profile with overlapping domain (or explicit override) | **Create Technology Profile in `candidate` state** with cluster metadata; link cluster to profile |
| `active` | `dissolved` | Coherence score drops below floor or member groups all close | `system` | Coherence score < 0.40 OR all member groups in terminal states | If a linked Technology Profile exists in `candidate` state, notify analyst |

**Coherence threshold:** Activation at 0.65, dissolution floor at 0.40. Values are system constants stored in configuration, not hardcoded in the state machine definition.

---

### 4.4 Technology Profile

A Technology Profile is the persistent record of a tracked technology, assembled from cluster evidence.

**State diagram**

```
              ┌───────────┐
    create    │           │
  ──────────► │ candidate │◄──────────────────────┐
              │           │                       │ revival with
              └─────┬─────┘                       │ new evidence
                    │ sufficient evidence           │
                    │ + analyst approval            │
                    ▼                              │
              ┌───────────┐                       │
              │ emerging  │                       │
              └─────┬─────┘                       │
                    │ sustained signal             │
                    │ + senior analyst approval    │
                    ▼                              │
              ┌───────────┐                       │
              │  active   │◄──────────────────┐   │
              └─────┬─────┘                   │   │
                    │                         │   │
              periodic review                 │   │
              or analyst trigger              │   │
                    │                         │   │
                    ▼                         │   │
              ┌─────────────┐                 │   │
              │ under_review │                │   │
              └──────┬──────┘                 │   │
                     │                         │   │
              ┌──────┴──────┐                  │   │
              │             │                  │   │
         review          review               │   │
         passes          fails                │   │
              │             │                  │   │
              └──────►  active ◄───────────────┘   │
                            │                      │
                       review fails                │
                       irrecoverably               │
                            ▼                      │
                       ┌──────────┐                │
                       │ archived │────────────────┘
                       └──────────┘
```

**Transition table**

| From State | To State | Trigger | Actor Required | Preconditions | Side Effects |
|---|---|---|---|---|---|
| *(none)* | `candidate` | Cluster promotion creates profile | `system` | Originating cluster in `promoted` state | Link to source cluster; initialize Knowledge State |
| `candidate` | `emerging` | Analyst confirms sufficient initial evidence | `analyst` | ≥ 1 active Corroboration Group linked; Knowledge State initialized; reason provided | Notify senior analyst; add to research queue |
| `emerging` | `active` | Senior analyst confirms sustained signal | `senior_analyst` | ≥ 2 active Corroboration Groups; coherence score ≥ 0.60; Knowledge State `≥ partial`; reason provided | Mark as active tracked technology; enable Solution Candidate creation |
| `active` | `under_review` | Periodic review timer fires OR analyst flags for review | `system` or `analyst` | None (any active profile can enter review) | Create review queue entry; assign to senior analyst |
| `under_review` | `active` | Review passes — evidence still supports active tracking | `senior_analyst` | Review completed; reason provided | Close queue entry; reset review timer |
| `under_review` | `archived` | Review fails — evidence no longer supports tracking | `senior_analyst` or `architect` | Review completed; reason provided | Archive linked Solution Candidates (if `draft`); notify stakeholders |
| `active` | `archived` | Manual archival by architect (with justification) | `architect` | Reason provided; no linked Advisories in `approved` or `published` state | Same as above |
| `archived` | `candidate` | Revival — new cluster evidence surfaces for this technology | `analyst` | New Cluster in `promoted` state references same technology domain; analyst provides revival reason | Re-link to new cluster; reset Knowledge State for re-evaluation |

**Knowledge State alignment:** The Knowledge State field (`unknown` → `partial` → `known` → `superseded`) tracks evidence completeness independently of workflow state. The `emerging → active` transition requires Knowledge State ≥ `partial`. No other transitions have a hard Knowledge State requirement, but the review queue will flag profiles whose workflow state has outpaced their Knowledge State.

**Back-transitions:** `under_review → active` (review passes) and `archived → candidate` (revival) are explicitly permitted. All other back-transitions are illegal.

---

### 4.5 Solution Candidate

A Solution Candidate is a proposed technical response to a signal captured in a Technology Profile.

**State diagram**

```
              ┌────────┐
    create    │        │
  ──────────► │ draft  │
              │        │
              └───┬────┘
                  │ analyst
                  │ submits for review
                  ▼
              ┌────────────┐
              │ under_review│
              └──────┬─────┘
                     │
              ┌──────┴──────┐
              │             │
         approved        rejected
              │             │
              ▼             ▼
          ┌────────┐   ┌──────────┐
          │approved│   │ rejected │
          └───┬────┘   └──────────┘
              │
              │ work complete
              ▼
          ┌───────────┐
          │ handed_off │
          └───────────┘
```

**Transition table**

| From State | To State | Trigger | Actor Required | Preconditions | Side Effects |
|---|---|---|---|---|---|
| *(none)* | `draft` | Analyst creates candidate against active Technology Profile | `analyst` | Parent Technology Profile in `active` state | None |
| `draft` | `under_review` | Analyst submits candidate | `analyst` | Candidate has title, description, and at least one linked Technology Profile; reason optional | Create review queue entry for senior analyst |
| `under_review` | `approved` | Senior analyst approves | `senior_analyst` | Review completed; reason provided | Notify originating analyst; enable Prototype creation |
| `under_review` | `rejected` | Senior analyst rejects | `senior_analyst` | Reason required | Notify originating analyst; reason stored on candidate row |
| `approved` | `handed_off` | Analyst marks work as delivered | `analyst` | Linked Prototype(s) exist in `complete` state OR explicit "no prototype" override with reason | Link to completion Prototype(s); create Finding if applicable |
| `approved` | `under_review` | Analyst requests re-review (scope change) | `analyst` | Reason provided | Reopen review queue entry |

**"Handed off" definition:** The Solution Candidate has been converted into a deliverable — either a completed Prototype, a published Advisory, or an explicit decision that no further work is required. This state marks the end of the solution lifecycle for this candidate.

**Rejection handling:** Rejected candidates are not deleted. The `rejected` state is terminal within the current cycle. Revival requires creating a new candidate (the rejected candidate may be referenced in the new one's metadata for context). See DECISION NEEDED §7.2.

---

### 4.6 Prototype

A Prototype is an implementation artifact produced in response to an approved Solution Candidate.

**State diagram**

```
              ┌────────┐
    create    │        │
  ──────────► │ scoped │
              │        │
              └───┬────┘
                  │ analyst
                  │ begins work
                  ▼
              ┌─────────────┐
              │ in_progress  │
              └──────┬──────┘
                     │
              ┌──────┴──────┐
              │             │
           complete      abandoned
              │             │
              ▼             ▼
          ┌──────────┐  ┌───────────┐
          │ complete │  │ abandoned │
          └──────────┘  └───────────┘
```

**Transition table**

| From State | To State | Trigger | Actor Required | Preconditions | Side Effects |
|---|---|---|---|---|---|
| *(none)* | `scoped` | Analyst creates prototype against approved Solution Candidate | `analyst` | Parent Solution Candidate in `approved` state | None |
| `scoped` | `in_progress` | Analyst begins work | `analyst` | Prototype has scope description | None |
| `in_progress` | `complete` | Analyst marks complete | `analyst` | Completion notes provided | Trigger Solution Candidate `handed_off` eligibility check; notify senior analyst |
| `in_progress` | `abandoned` | Analyst or senior analyst abandons | `analyst` or `senior_analyst` | Reason required | Notify originating analyst if senior analyst abandons; if no other Prototypes remain in progress against parent Solution Candidate, flag parent for review |
| `scoped` | `abandoned` | Scope determined to be invalid before work begins | `analyst` or `senior_analyst` | Reason required | Same as above |

**Link back to Solution Candidate:** On transition to `complete`, the service checks whether all Prototypes linked to the parent Solution Candidate are in terminal states (`complete` or `abandoned`), and if at least one is `complete`, adds the Solution Candidate to the `handed_off` eligibility queue for the analyst to confirm.

---

### 4.7 Finding

A Finding is a discrete insight or conclusion drawn from research, publishable independently or as part of an Advisory.

**State diagram**

```
              ┌────────┐
    create    │        │
  ──────────► │ draft  │
              │        │
              └───┬────┘
                  │ submit
                  │ for review
                  ▼
              ┌─────────────┐
              │ peer_review  │
              └──────┬──────┘
                     │
              ┌──────┴──────┐
              │             │
          published      withdrawn
              │             │
              ▼             ▼
          ┌───────────┐  ┌───────────┐
          │ published │  │ withdrawn │
          └───────────┘  └───────────┘
```

**Transition table**

| From State | To State | Trigger | Actor Required | Preconditions | Side Effects |
|---|---|---|---|---|---|
| *(none)* | `draft` | Analyst creates finding | `analyst` | None | None |
| `draft` | `peer_review` | Analyst submits for peer review | `analyst` | Finding has title, body, and at least one supporting evidence reference | Create review queue entry |
| `peer_review` | `published` | Peer analyst approves | `analyst` (different from author) | Reviewer is not the original author; reason optional | Close queue entry; notify author; make finding available for Advisory inclusion |
| `peer_review` | `draft` | Reviewer returns for revision | `analyst` | Reason required | Close queue entry; notify author with feedback |
| `peer_review` | `withdrawn` | Author or senior analyst withdraws | `analyst` or `senior_analyst` | Reason required | Close queue entry; if Finding is linked to an Advisory in `assembling` state, remove link |
| `published` | `withdrawn` | Senior analyst withdraws published finding | `senior_analyst` | Reason required; if linked to a published Advisory, Advisory must first be withdrawn | Notify stakeholders; flag any Advisories that included this Finding |

---

### 4.8 Advisory

An Advisory is a curated, evidence-backed intelligence product delivered to stakeholders.

**State diagram**

```
              ┌────────────┐
    create    │            │
  ──────────► │ assembling │
              │            │
              └─────┬──────┘
                    │ evidence chain
                    │ complete; submit
                    ▼
              ┌────────┐
              │ review  │
              └────┬───┘
                   │
              ┌────┴────┐
              │         │
          approved   withdrawn (pre-approval)
              │         │
              ▼         ▼
          ┌────────┐  ┌───────────┐
          │approved│  │ withdrawn │
          └───┬────┘  └───────────┘
              │
              │ published
              │ by architect/leadership
              ▼
          ┌───────────┐
          │ published │
          └─────┬─────┘
                │
           withdrawn
           post-publication
                │
                ▼
          ┌───────────┐
          │ withdrawn │
          └───────────┘
```

**Transition table**

| From State | To State | Trigger | Actor Required | Preconditions | Side Effects |
|---|---|---|---|---|---|
| *(none)* | `assembling` | Senior analyst creates advisory | `senior_analyst` | None | None |
| `assembling` | `review` | Senior analyst submits for review | `senior_analyst` | Evidence chain complete (see below); evidence chain completeness check passes (per TS-05 §4.4); ≥ 1 published Finding linked; all linked Technology Profiles in `active` state; reason optional | Create review queue entry for architect |
| `review` | `approved` | Architect approves | `architect` | Review completed; reason provided | Notify senior analyst; add to publication queue for leadership |
| `review` | `assembling` | Architect returns for revision | `architect` | Reason required | Close queue entry; notify senior analyst with feedback |
| `review` | `withdrawn` | Architect or leadership withdraws during review | `architect` or `leadership` | Reason required | Close queue entry; notify senior analyst |
| `approved` | `published` | Leadership authorizes publication | `architect` or `leadership` | Advisory in `approved` state; publication target specified | Publish to delivery channel; notify subscribers; create publication record |
| `approved` | `withdrawn` | Withdrawn before publication | `architect` or `leadership` | Reason required | Notify senior analyst |
| `published` | `withdrawn` | Post-publication withdrawal | `leadership` | Reason required; withdrawal notice drafted | Notify subscribers; record withdrawal with timestamp and reason |

**Evidence chain requirement:** Before an Advisory can transition to `review`, the service validates:
1. At least one published Finding is linked.
2. Every Technology Profile referenced in the Advisory is in `active` state.
3. Every Solution Candidate referenced (if any) is in `approved` or `handed_off` state.
4. No linked Findings are in `withdrawn` state.

**Approval authority:** Architect can approve and publish. Leadership can authorize publication or withdraw post-publication. Senior analysts cannot self-approve — they require architect review.

---

### 4.9 Market Reports

A Market Report is a living, versioned research document scoped to a Neighborhood or Domain Area (introduced in ADR-016).

**States:** `draft` → `under_review` → `published`

Additional transitions:
- `published` → `withdrawn` (manual withdrawal by `architect+`)
- `published` → `superseded` (system-set when a new version is created via `POST /v1/market-reports/{id}/new-version`)

**Transition table:**

| From | To | Actor | Preconditions | System action |
|---|---|---|---|---|
| `draft` | `under_review` | `senior_analyst+` | Body and topic_scope_ref present | Creates review_queue entry (queue_type: `peer_review`) |
| `under_review` | `published` | `architect+` | Peer review complete | Sets published_at; notifies watchers |
| `under_review` | `draft` | `architect+` | Reviewer requests revision | Notifies author |
| `published` | `withdrawn` | `architect+` | — | Sets withdrawn_at; marks is_active = false |
| `published` | `superseded` | system | New version created | Sets superseded_at, superseded_by on old record; new record starts in draft |

---

## 5. Review Queues

### 5.1 Events That Create Queue Entries

| Event | Object Type | Queue Type | Assigned To |
|---|---|---|---|
| Signal transition to `triaged` with boundary case flagged | Signal | `boundary_case_review` | `analyst` |
| Corroboration Group transitions to `active` | Corroboration Group | `group_review` | `analyst` |
| Cluster transitions to `active` | Cluster | `cluster_review` | `analyst` |
| Technology Profile transitions to `emerging` | Technology Profile | `profile_review` | `senior_analyst` |
| Technology Profile transitions to `under_review` | Technology Profile | `profile_review` | `senior_analyst` |
| Solution Candidate transitions to `under_review` | Solution Candidate | `solution_review` | `senior_analyst` |
| Finding transitions to `peer_review` | Finding | `peer_review` | `analyst` (not author) |
| Advisory transitions to `review` | Advisory | `advisory_review` | `architect` |
| Object stuck in any non-terminal state past escalation threshold | Any | `escalation` | role above current reviewer |

### 5.2 Queue Entry Schema

The full queue UI is defined in DS-08. The underlying schema:

```sql
CREATE TABLE review_queue (
  id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  object_id       UUID        NOT NULL,
  object_type     TEXT        NOT NULL,
  queue_type      TEXT        NOT NULL CHECK (queue_type IN (
                    'boundary_case_review',
                    'disputed_classification',
                    'reclassification_request',
                    'group_review',
                    'cluster_review',
                    'profile_review',
                    'profile_creation_request',
                    'solution_review',
                    'peer_review',
                    'advisory_review',
                    'needs_internal_answer',
                    'escalation'
                  )),
  assigned_to     UUID,                   -- user ID; NULL if unassigned
  assigned_role   TEXT        NOT NULL,   -- role responsible for resolution
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  due_at          TIMESTAMPTZ,            -- escalation deadline
  escalated_at    TIMESTAMPTZ,
  resolved_at     TIMESTAMPTZ,
  resolved_by     UUID,
  resolution      TEXT,                   -- 'approved' | 'rejected' | 'returned' | 'withdrawn'
  resolution_note TEXT,
  metadata        JSONB       NOT NULL DEFAULT '{}'
);

CREATE INDEX idx_review_queue_open
  ON review_queue (assigned_role, created_at)
  WHERE resolved_at IS NULL;

CREATE INDEX idx_review_queue_object
  ON review_queue (object_id, object_type, resolved_at);
```

### 5.3 Escalation

When a queue entry's `due_at` passes without resolution, the scheduler triggers an escalation transition:

1. `due_at` is computed at queue entry creation time based on object type and queue type (see DECISION NEEDED §7.1).
2. At `due_at`, the scheduler sets `escalated_at = now()` and reassigns to the next role in the escalation chain.
3. Escalation chain: `analyst` → `senior_analyst` → `architect` → `leadership`.
4. A `workflow.escalation` event is emitted. Notification is sent to both the original assignee and the new assignee.
5. A new `state_transitions` row is written for the object with `actor_type = 'scheduler'` and `reason = 'escalation: queue item aged past threshold'`.
6. If the item reaches `leadership` and still ages out, the item is flagged as `critical_escalation` and enters a separate manual triage list.

---

## 6. Boundary Case Integration

### 6.1 How CCE Boundary Cases Create Queue Entries

When CCE scores a Signal with `diff_strength < 0.15`, it:

1. Sets `review_status = 'boundary'` on the Signal row.
2. Writes a `boundary_cases` record with the signal ID, diff_strength score, competing group candidates, and CCE reasoning.
3. Calls `WorkflowStateService.transition(signalId, 'signal', 'triaged', 'system', 'system', 'CCE run complete — boundary case flagged')` with `metadata.boundary_case_id` set.
4. The transition side effect creates a `review_queue` entry of type `boundary_case_review` assigned to an `analyst`.

The Signal transitions to `triaged` normally — the boundary case does not block the triaged state. It blocks the *next* transition.

### 6.2 Blocking Rules

The following transitions are blocked while a Signal has an unresolved boundary case:

| Object | Blocked Transition | Blocking Condition |
|---|---|---|
| Signal | `triaged → grouped` | Signal has unresolved `boundary_case_review` queue entry |
| Corroboration Group | `forming → active` | Any member Signal has unresolved boundary case |
| Cluster | `candidate → active` | Any member Group has a Signal with unresolved boundary case, AND that Signal's contribution would change the coherence score by > 0.05 |

The `WorkflowStateService` precondition check for each of these transitions queries for open boundary case queue entries on the object and its dependencies. If found, it returns `BOUNDARY_CASE_BLOCKING` with the list of blocking `boundary_case_id` values.

Resolution: An analyst reviewing the boundary case queue entry selects one of:
- **Assign to group** — resolves the block, signals proceeds to `grouped`.
- **Exclude from group** — signals proceeds to `archived` with reason.
- **Flag for senior analyst** — escalates the queue entry without resolving the block.

---

## 7. DECISION NEEDED Items

> **DECISION NEEDED —** What are the escalation time thresholds per object and queue type? The current design computes `due_at` at queue creation time but no thresholds have been set. Proposed defaults: `boundary_case_review` → 24 hours; `group_review` → 48 hours; `cluster_review` → 48 hours; `profile_review` → 72 hours; `solution_review` → 48 hours; `peer_review` → 72 hours; `advisory_review` → 96 hours. Should these be configurable per team, or fixed system constants? *Recommendation: define as configurable system settings stored in a `workflow_config` table, with the proposed defaults as seed data. This allows tuning without a code deploy.*

> **DECISION NEEDED —** Can rejected objects be revived, and who has the authority to do so? Currently, `rejected` Signals and `rejected` Solution Candidates are terminal — revival means creating a new object. `archived` Technology Profiles explicitly support revival via `archived → candidate`. The question is whether this should be extended: can a `rejected` Signal be un-rejected (back to `triaged`) if an analyst made an error, and if so, should that require senior analyst approval? *Recommendation: allow `rejected → triaged` for Signals within a 7-day window, requiring senior analyst approval and a mandatory reason. After 7 days, rejection is permanent and a new Signal must be submitted. Do not allow Solution Candidate revival — the new-candidate approach preserves cleaner history.*

> **DECISION NEEDED —** How should orphaned objects be handled when a parent is dissolved? The current spec notes that Signals become orphaned when their Corroboration Group is dissolved, but does not define the resolution path. Three options: (A) Auto-archive orphaned Signals immediately; (B) Return orphaned Signals to `triaged` state for analyst reassignment; (C) Hold orphaned Signals in a new `orphaned` queue state, blocking further transitions until an analyst resolves them. *Recommendation: Option B — return orphaned Signals to `triaged` with a system-generated reason noting the dissolved group, and create a `group_review` queue entry for analyst reassignment. This preserves the Signal data and gives analysts a chance to reassign to a surviving group. Option C adds a state that every consumer of the state machine must handle; Option A loses potentially valid data.*
