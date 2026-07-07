# TS-08 — Security: Access Control and Audit

**Spec ID:** TS-08
**Status:** Draft
**Last updated:** 2026-07-06

---

## 1. Purpose

This spec defines the access control model, audit logging requirements, sensitive content handling rules, and data retention policy for the Tech Range platform. It establishes who can read, write, approve, and publish intelligence objects, how sensitive content is protected behind a pointer pattern, and what constitutes a complete and immutable audit trail.

This spec governs application-layer security. It assumes network-level security and infrastructure hardening are addressed in TS-11.

---

## 2. Threat model (brief)

### What we are protecting against

| Threat | Description |
|---|---|
| Unauthorized data access | An analyst reads records above their clearance or role boundary |
| Action without audit trail | A user modifies classification, state, or sensitive references with no record |
| Sensitive content leakage | The content behind a `sensitive_ref_id` pointer is exposed to an actor who is not authorized to follow the pointer |
| Privilege escalation | A user or service account gains role permissions beyond what was granted |
| Stale access | A departed analyst, rotated service account, or expired API key retains platform access |

### Out of scope for this spec

- Network-level security and perimeter controls (see TS-11)
- Infrastructure hardening: secrets management, key rotation, TLS configuration (see TS-11)
- Vulnerability scanning and penetration testing cadence

---

## 3. Role definitions

The platform defines six roles. Roles are mutually exclusive except where noted (a user holds exactly one role; service accounts are distinct from human roles).

| Role | Description | Pipeline stage access | Object type access | Sensitive content access |
|---|---|---|---|---|
| `viewer` | Read-only access to published outputs | Published Advisories only | Read Advisory (published state only) | None — cannot see `sensitive_ref_id` values or follow pointers |
| `analyst` | Ingestion review, triage, and classification | `pending_review`, `triaged`, `in_analysis` | Read/write Signal, TechEntity, Event, Relationship (own assignments); read Match Set | Cannot follow pointer; can see that a `sensitive_ref_id` exists on a record |
| `senior_analyst` | All analyst access plus approval authority | All stages except `published` | Read/write all intelligence objects; approve state transitions through `ready_for_advisory` | Can follow pointer to Sensitive Content Store for records in their program scope [Internal decision rule] |
| `architect` | Platform design and schema authority; final classification override | All stages | Read/write all object types; issue classification overrides; publish Advisories | Full access to Sensitive Content Store |
| `admin` | User and role management; audit log access; no intelligence write access | No pipeline access | Read all object types; no write except user/role records | Cannot follow pointer (admin is not an intelligence role) |
| `service_account` | Automated ingestion pipeline, CCE, scheduled jobs | Writes to `pending_review`; reads assigned pipeline stages per job scope | Read/write scoped to assigned job (e.g., ingestion service writes Signals only) | No access; service accounts cannot be granted sensitive content access |

**Notes:**
- An `admin` user who also functions as an analyst must hold two separate accounts. Admin and intelligence-write roles are not combinable. [Internal decision rule]
- `service_account` permissions are further constrained by API key scope at issuance time (see Section 4.4).

---

## 4. Access control model

### 4.1 Authentication

All human users authenticate through the platform's identity provider. SSO/SAML integration is expected for enterprise deployment. The specific IdP configuration, session token lifetime, and credential storage are deferred to TS-11.

Session tokens must be short-lived. Refresh token handling deferred to TS-11.

> **DECISION NEEDED —** Is MFA mandatory for all roles, or only for `architect` and `admin`? *Recommendation: Require MFA for all roles given the national security context of the platform. Downgrade to architect/admin-only only if analyst onboarding friction becomes a documented blocker.*

### 4.2 Authorization

Authorization is role-based and enforced exclusively at the API layer. The UI never makes security decisions.

- Every API request carries an authenticated session or API key.
- The API gateway validates authentication before routing any request.
- Application services (`WorkflowStateService`, `MatchSetService`, `AdvisoryService`, etc.) receive a resolved `actor` object containing `actor_id`, `actor_role`, and `actor_scope` before executing any business logic.
- Row-level security (RLS) policies in the database provide a defense-in-depth layer for sensitive-flagged records. RLS is not the primary enforcement point; it is a backstop.

### 4.3 Row-level security for sensitive records

Records with a non-null `sensitive_ref_id` are subject to additional row-level restrictions beyond standard role access:

- The `sensitive_ref_id` column is readable only by roles authorized to see that a sensitive reference exists (`analyst` and above).
- The content behind the pointer (the Sensitive Content Store record) is accessible only to `senior_analyst` (within program scope), `architect`, and roles explicitly granted per-record access. [Internal decision rule]
- `viewer` and `service_account` roles receive `null` in place of `sensitive_ref_id` in all API responses.

### 4.4 API keys for service accounts

Service accounts authenticate via API keys. API keys are:

- Issued per job type (e.g., one key for the ingestion pipeline, a separate key for the CCE scheduler).
- Scoped at issuance: each key encodes the permitted object types, pipeline stages, and HTTP methods. A key issued for ingestion cannot read Advisories.
- Non-expiring by default but must be rotatable without downtime (rotation procedure in TS-11).
- Stored as hashed values in the database; the plaintext key is shown once at issuance.
- Logged on every use in the audit log with `actor_type = 'service_account'`.

### 4.5 Per-object-type access control matrix

The role definitions in §3 describe access at a high level. This matrix specifies per-object permissions for entities not fully covered by the general descriptions above. DS specs (role summary tables) and TS-06 (state machine actors) are authoritative for UI-layer behavior; this matrix governs API-layer enforcement.

**Market Reports** (`market_reports`)

| Role | Read | Create / Edit draft | Submit for review | Publish / Withdraw | Add observations |
|---|---|---|---|---|---|
| `viewer` | Published only (`state = 'published'`, `is_active = true`) | No | No | No | No |
| `analyst` | All states | No | No | No | Yes |
| `senior_analyst` | All states | Yes (`draft`, `under_review`) | Yes | No | Yes |
| `architect` | All states | Yes | Yes | Yes | Yes |
| `admin` | All states | No (admin is not an intelligence role) | No | No | No |
| `service_account` | Scoped by key | No | No | No | No |

**Prototypes** (`prototypes`)

| Role | Read | Create (direct request) | Edit scope / criteria / plans | Log progress entries | Confirm scope (`scoped → in_progress`) | Mark complete | Abandon |
|---|---|---|---|---|---|---|---|
| `viewer` | `in_progress`, `complete` only | No | No | No | No | No | No |
| `analyst` | All states | Yes | Yes (while `scoped` or `in_progress`) | Yes | No | No | No |
| `senior_analyst` | All states | Yes | Yes | Yes | Yes | Yes | No |
| `architect` | All states | Yes | Yes | Yes | Yes | Yes | Yes |
| `admin` | All states | No | No | No | No | No | No |
| `service_account` | No | No | No | No | No | No | No |

> **Note on viewer access to prototypes:** `viewer` access to `in_progress` and `complete` prototypes is an intentional product decision to give leadership consumers visibility into the active prototype portfolio. This extends the default viewer scope (published Advisories only) and should be reviewed during security design review.

---

## 5. Sensitive content handling

### 5.1 The pointer pattern

Sensitive content is never stored in the main application database. The main schema stores only a `sensitive_ref_id` (UUID) on any intelligence object that has associated sensitive material. The actual content lives in a separate Sensitive Content Store.

This means:
- A backup or export of the main database contains no sensitive content — only UUIDs pointing to records that may or may not exist in the Sensitive Content Store.
- The Sensitive Content Store has its own access control layer, independent of main-schema role checks.
- Removing access to the Sensitive Content Store has no effect on the main schema record; it remains intact with a dangling pointer.

### 5.2 Sensitive Content Store access control

The Sensitive Content Store enforces its own authentication and authorization, separate from the main application's role system. Access is granted explicitly per principal (human user or system role), not derived automatically from main-schema role.

[Internal decision rule] — The mapping of main-schema roles to Sensitive Content Store access grants is an internal configuration decision. By default, `architect` has full access; `senior_analyst` access is scoped by program.

> **DECISION NEEDED —** Should the Sensitive Content Store be implemented as a separate database instance (strongest isolation), a schema within the main database protected by RLS and connection-level grants, or an external vault/secrets-manager pattern? *Recommendation: Separate database instance. This provides the cleanest isolation — a compromised main DB connection cannot reach sensitive content — and aligns with national security data-handling norms. Accept the operational overhead of managing a second instance.*

### 5.3 Creating and linking sensitive references

When an analyst or senior analyst identifies content that warrants sensitive handling:

1. A request is made to the Sensitive Content Store to create a new record. This requires the actor to hold Sensitive Content Store write access.
2. The Sensitive Content Store returns a `sensitive_ref_id` (UUID).
3. The actor writes the `sensitive_ref_id` to the appropriate field on the main-schema object.
4. Both operations are logged in the audit log: one entry for the Sensitive Content Store write, one for the main-schema field update.

Sensitive references may not be created by `analyst` role without explicit per-record grant from a `senior_analyst` or `architect`. [Internal decision rule]

### 5.4 Revocation and deletion

Sensitive content is never hard-deleted (see Section 7 on the soft delete pattern). If a sensitive reference must be revoked:

- The Sensitive Content Store record is moved to a `withdrawn` state with a revocation timestamp and actor.
- The `sensitive_ref_id` on the main-schema object is retained — the pointer remains, now pointing to a withdrawn record.
- Any subsequent attempt to follow the pointer returns a `WITHDRAWN` status, not a 404. This preserves the audit chain.
- The main-schema object is not automatically reclassified when a sensitive reference is revoked. A `senior_analyst` or `architect` must take explicit reclassification action.

### 5.5 Audit requirements for sensitive content access

Every interaction with the Sensitive Content Store must generate an audit log entry, including:

- Record creation and field writes
- Read access (following a pointer)
- State transitions (active → withdrawn, etc.)
- Access grant changes (who was given or removed from access)
- Failed access attempts (unauthorized pointer follow)

Sensitive content audit entries must include `actor_id`, `actor_type`, `sensitive_ref_id`, `action`, `ip_address`, and `created_at`. The `before_state` / `after_state` fields must not include the sensitive content itself — only metadata (record ID, state, classification level).

---

## 6. Audit log requirements

### 6.1 What must be logged

The following actions must generate an audit log entry. This list is exhaustive for initial implementation; additions require a spec update.

| Category | Actions |
|---|---|
| State transitions | Every `workflow_state` change on any intelligence object |
| Match set writes | Every `MatchSet` record created or updated by the CCE |
| Analyst corrections | Every human override of a CCE match or classification |
| Classification actions | Every initial classification, reclassification, and classification override |
| Advisory lifecycle | Draft created, approved, published, superseded, withdrawn |
| Market Report lifecycle | Draft created, submitted for review, published, withdrawn, superseded (new version created) |
| Prototype lifecycle | Request created, scope confirmed (scoped → in_progress), marked complete, abandoned |
| Sensitive content | Pointer created, content accessed (read), content withdrawn, access grant changed, failed access attempt |
| Authentication | Login (success and failure), logout, session expiry, API key used |
| Role and access changes | Role assignment, role change, user deactivation, API key issuance and revocation |

### 6.2 Audit log schema

```sql
CREATE TABLE audit_log (
  id            UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
  actor_id      UUID,                        -- NULL for unauthenticated/system events
  actor_type    TEXT         NOT NULL,       -- 'user', 'service_account', 'system'
  action        TEXT         NOT NULL,       -- e.g. 'state_transition', 'match_set_write', 'sensitive_content_read'
  object_id     UUID,                        -- ID of the affected record, if applicable
  object_type   TEXT,                        -- e.g. 'signal', 'tech_entity', 'advisory', 'match_set', 'market_report', 'prototype'
  before_state  JSONB,                       -- snapshot of relevant fields before action
  after_state   JSONB,                       -- snapshot of relevant fields after action
  ip_address    INET,
  session_id    UUID,
  created_at    TIMESTAMPTZ  NOT NULL DEFAULT now()
);
```

**Index recommendations:**
- `(actor_id, created_at)` — for user activity queries
- `(object_id, object_type, created_at)` — for object history queries
- `(action, created_at)` — for action-type reporting
- `created_at` with a partition strategy for long-term retention (partition by month or quarter)

### 6.3 Append-only enforcement

The audit log must be append-only. No `UPDATE` or `DELETE` statements may be issued against `audit_log` by any application role.

Enforcement mechanisms (implement at least two):

1. **Database-level:** The application database role used by the API service must not hold `UPDATE` or `DELETE` privileges on `audit_log`.
2. **Trigger-level:** A PostgreSQL trigger raises an exception on any attempt to update or delete an audit row.
3. **Separate write path:** The audit log writer is a separate service account with `INSERT`-only privileges. No application service writes directly to `audit_log` except through this path.

Audit log records must not contain sensitive content (see Section 5.5).

### 6.4 Retention

Minimum retention for audit logs: **7 years**, recommended given the national security and defense context of the platform.

Audit logs older than the retention window must not be automatically deleted; deletion requires a documented review process and explicit admin action. [Internal decision rule]

Storage tiering (e.g., move records older than 2 years to cold storage) is acceptable provided the records remain queryable within a reasonable SLA. [Internal decision rule]

---

## 7. Data retention policy

Tech Range operates on a **no-hard-delete** principle for all intelligence objects. The rationale: a record that was believed true, acted upon, or contributed to a published Advisory retains intelligence and legal value even after it is superseded or withdrawn.

| Object type | Retention rule |
|---|---|
| Active records (Signals, TechEntities, Events, Relationships) | Retained indefinitely |
| Superseded `match_set` versions | Retained indefinitely; prior versions remain queryable with full history |
| Superseded `state_transition` records | Retained indefinitely; full transition history is preserved |
| Archived intelligence objects | Retained with complete field history; state = `archived`, not deleted |
| Withdrawn Advisories | Retained with `withdrawn` state; the published content and the withdrawal reason are both preserved |
| Sensitive content (Sensitive Content Store) | Retained with `withdrawn` state on revocation; no hard delete (see Section 5.4) |
| Audit logs | Minimum 7-year retention; see Section 6.4 |
| User records (departed analysts) | Deactivated, not deleted; `actor_id` references in audit logs must remain resolvable |

**Soft delete pattern:** State transitions to `archived` or `withdrawn` are the only permitted way to retire an intelligence object. Direct `DELETE` statements against intelligence tables are not permitted from any application role.

---

## 8. Access control enforcement

Security logic is enforced server-side, in the following layers in order:

1. **API gateway** — validates authentication token or API key on every inbound request before routing. Unauthenticated requests are rejected at the gateway with no application processing.

2. **Application service layer** — each service (`WorkflowStateService`, `MatchSetService`, `AdvisoryService`, `SensitiveRefService`) receives a resolved `actor` object and checks `actor.role` against the permitted roles for the requested action before executing any business logic. This is the primary authorization enforcement point.

3. **Row-level security (PostgreSQL RLS)** — RLS policies on sensitive-flagged record columns provide a database-layer backstop. RLS is not the primary enforcement point; it exists to limit blast radius if application-layer authorization is bypassed or misconfigured.

4. **Sensitive Content Store access layer** — a separate access control check, independent of main-schema authorization, governs all reads and writes to the Sensitive Content Store. Holding a `senior_analyst` role in the main schema does not automatically confer Sensitive Content Store access; access must be explicitly granted.

**The UI layer enforces no security logic.** Role-based UI rendering (hiding buttons, filtering menus) is for user experience only. Any action that should be unauthorized must be rejected by the API regardless of whether the UI exposed the option.

**WorkflowStateService:** Before executing any state transition, checks:
- `actor.role` is in the set of roles permitted to execute the target transition
- The transition is valid from the current state (state machine check)
- If the object has a `sensitive_ref_id`, checks that the actor has the minimum role required to write to sensitive-flagged objects [Internal decision rule]

**CCE (Classification and Confidence Engine):** Before writing any `match_set` record, checks that the service account's API key scope permits writes to the target object type and pipeline stage. The CCE must not write to objects in `published` or `withdrawn` states.

---

## 9. Open decisions

> **DECISION NEEDED —** Sensitive Content Store implementation: separate database instance vs. RLS-protected schema within the main database vs. external vault (e.g., HashiCorp Vault, AWS Secrets Manager). *Recommendation: Separate database instance. Provides strongest isolation; sensitive content is unreachable from a compromised main DB connection. Accept the operational overhead of a second managed database.*

> **DECISION NEEDED —** Should `leadership` users (if this role is added, or mapped to `viewer` with expanded scope) receive read-only access to all active intelligence records, or only to published Advisories? *Recommendation: Published Advisories only as the default. Grant active-record read access only as an explicit, logged role extension. This limits exposure of in-progress analysis and reduces sensitive content leakage risk.*

> **DECISION NEEDED —** Is MFA mandatory for all human roles, or only for `architect` and `admin`? *Recommendation: Mandatory for all roles. The platform handles potentially sensitive national security intelligence; the friction cost of MFA for analysts is low relative to the risk of a compromised analyst account.*
