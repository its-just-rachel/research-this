# TS-09 — API and Integration Contracts

**Status:** Draft  
**Version:** 0.1  
**Last updated:** 2026-07-06

---

## 1. Purpose

This spec defines the API and integration contracts for the Tech Range platform. It covers:

- The external-facing REST API exposed through the API Gateway (TS-01) for analyst clients and extension services (ADR-011)
- Internal service-to-database contracts for the Ingestion Pipeline, CCE, and WorkflowStateService
- Enrichment provider integration contracts
- Error response standards, versioning policy, and deprecation timelines

All external traffic routes through the API Gateway. Internal services (Ingestion Pipeline, CCE, WorkflowStateService) write directly to the PostgreSQL database under explicit, scoped service account permissions. Extension services are strictly read-only consumers of the published API — they cannot write to core tables.

---

## 2. API Design Principles

### 2.1 RESTful over CRUD

Resources map to platform entities (signals, tech profiles, advisories, match sets, watchlists, alerts). HTTP verbs map to retrieval and creation, not to state management. State transitions are first-class operations with explicit endpoints.

### 2.2 Explicit State Transitions

State changes are performed via dedicated action endpoints, not via PATCH on a status field. This ensures that transition preconditions, audit logging, and workflow enforcement (WorkflowStateService) are applied consistently.

**Correct:** `POST /v1/signals/{id}/triage`  
**Incorrect:** `PATCH /v1/signals/{id}` with `{ "status": "triaged" }`

This pattern applies to all stateful resources: signals, technology profiles, advisories, and match sets.

### 2.3 Cursor-Based Pagination

All list endpoints use cursor-based pagination. Offset pagination is not supported. Responses include a `next_cursor` field; clients pass `cursor=<value>` on subsequent requests. Page size is controlled via `limit` (default: 25, max: 100).

```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6IjEyMyJ9",
    "has_more": true,
    "limit": 25
  }
}
```

### 2.4 Structured Filtering

List endpoints accept structured query parameters aligned with the filter dimensions defined in TS-07. Filter parameters use snake_case and support comma-delimited multi-value selection where appropriate (e.g., `status=triaged,grouped`).

### 2.5 URI Versioning

API versions are expressed in the URI path: `/v1/`, `/v2/`. The current version is `v1`. Breaking changes require a new version. Non-breaking additive changes (new optional fields in responses, new optional query parameters) are deployed within the current version without a version bump.

### 2.6 Sensitive Content Handling

Sensitive content is never returned inline in standard API responses. When a resource contains sensitive content, the response includes a `sensitive_ref_id` field. Retrieval of the actual sensitive content requires a separate authenticated call to the Sensitive Content Store API using that reference ID, subject to the caller's role-based access permissions (TS-08).

---

## 3. Core Resource Endpoints

Auth is required on all endpoints. Role requirements are noted per endpoint per TS-08 (SSO/SAML for analyst sessions; API key for service accounts).

---

### 3.1 Signals

| Method | Path | Description | Auth | Notes |
|--------|------|-------------|------|-------|
| GET | `/v1/signals` | List signals | Analyst, Read-only | Filterable by status, source, classification axis, date range, boundary case flag, corroboration group. Cursor-paginated. Sensitive content: `content_preview` is truncated; full content via `sensitive_ref_id` if flagged. |
| GET | `/v1/signals/{id}` | Get a single signal | Analyst, Read-only | Returns full signal record. If `is_sensitive = true`, body fields are replaced with `sensitive_ref_id`; call Sensitive Content Store to retrieve. |
| POST | `/v1/signals` | Submit a new signal | Analyst | Used for analyst-submitted signals and intentional injection workflows. Required fields per TS-02 schema. Returns created signal with `id`. |
| POST | `/v1/signals/{id}/triage` | Mark signal as triaged | Analyst | Triggers WorkflowStateService transition. Precondition: signal must be in `new` state. Returns updated state. 422 if precondition fails. |
| POST | `/v1/signals/{id}/group` | Add signal to a corroboration group | Analyst | Body: `{ "group_id": "uuid" }`. Creates or joins a corroboration group. Precondition: signal must be in `triaged` state. |
| POST | `/v1/signals/{id}/archive` | Archive a signal | Analyst | Triggers state transition to `archived`. Valid from: `triaged`, `grouped`. |
| POST | `/v1/signals/{id}/reject` | Reject a signal | Analyst | Triggers state transition to `rejected`. Valid from: `new`, `triaged`. Body: `{ "reason": "string" }`. |

**Sensitive content note:** When `is_sensitive = true` on a signal, the following fields are omitted from the response and a `sensitive_ref_id` is returned in their place: `raw_content`, `normalized_content`, `source_url`. All other metadata fields are returned normally.

---

### 3.2 Technology Profiles

| Method | Path | Description | Auth | Notes |
|--------|------|-------------|------|-------|
| GET | `/v1/tech-profiles` | List technology profiles | Analyst, Read-only | Filterable by lifecycle stage, classification axis, maturity tier, associated signal count. Cursor-paginated. |
| GET | `/v1/tech-profiles/{id}` | Get a single technology profile | Analyst, Read-only | Returns full profile including current lifecycle state, associated match sets, and signal count. |
| POST | `/v1/tech-profiles` | Create a technology profile | Analyst | Used for intentional profile creation (not pipeline-generated). Required fields per TS-02 schema. |
| GET | `/v1/tech-profiles/{id}/evidence-chain` | Get the full evidence chain for a profile | Analyst, Read-only | Returns ordered list of corroborating signals and their match sets. This endpoint may return sensitive signal references; `sensitive_ref_id` applies to individual signal entries. |
| POST | `/v1/tech-profiles/{id}/transitions/{to_state}` | Trigger a lifecycle state transition | Analyst | Path param `to_state` is the target state (e.g., `emerging`, `active`, `archived`). Preconditions enforced by WorkflowStateService; returns 422 with detail if evidence chain is incomplete. |
| GET | `/v1/tech-profiles/{id}/match-sets` | List match sets associated with this profile | Analyst, Read-only | Returns all CCE-generated match sets linked to the profile. Filterable by axis, boundary case flag. |

---

### 3.3 Advisories

| Method | Path | Description | Auth | Notes |
|--------|------|-------------|------|-------|
| GET | `/v1/advisories` | List advisories | Analyst, Read-only | Filterable by status, classification axis, tech profile, date range. Cursor-paginated. |
| GET | `/v1/advisories/{id}` | Get a single advisory | Analyst, Read-only | Returns full advisory content. If advisory references sensitive signals, those are returned as `sensitive_ref_id` references only. |
| POST | `/v1/advisories` | Create a draft advisory | Analyst | Creates advisory in `draft` state. Body includes title, classification references, linked tech profile ID, content. |
| POST | `/v1/advisories/{id}/submit-for-review` | Submit advisory for editorial review | Analyst | Transitions from `draft` to `in_review`. Precondition: at least one linked tech profile required. |
| POST | `/v1/advisories/{id}/approve` | Approve advisory | Editor, Admin | Transitions from `in_review` to `approved`. Records approver identity. |
| POST | `/v1/advisories/{id}/publish` | Publish advisory | Editor, Admin | Transitions from `approved` to `published`. Sets `published_at` timestamp. Triggers downstream notifications if configured. |
| POST | `/v1/advisories/{id}/withdraw` | Withdraw a published advisory | Admin | Transitions from `published` to `withdrawn`. Body: `{ "reason": "string" }`. Reason is required and stored in the state transition log. |

---

### 3.4 Match Sets (Classification)

Match sets are generated by the CCE and represent classification decisions. Analysts can approve, dispute, or override; they cannot create match sets directly.

| Method | Path | Description | Auth | Notes |
|--------|------|-------------|------|-------|
| GET | `/v1/match-sets` | List match sets | Analyst, Read-only | Filterable by object ID, classification axis, boundary case flag, status (pending, approved, disputed, overridden). Cursor-paginated. |
| GET | `/v1/match-sets/{id}` | Get a single match set | Analyst, Read-only | Returns full match set including primary/secondary positions, confidence scores, and axis. |
| POST | `/v1/match-sets/{id}/approve` | Approve a match set classification | Analyst | Records analyst approval. Transitions status to `approved`. Precondition: match set must be in `pending` state. |
| POST | `/v1/match-sets/{id}/dispute` | Flag a match set as disputed | Analyst | Transitions status to `disputed`. Body: `{ "reason": "string" }`. Routes to queue for senior analyst review. |
| POST | `/v1/match-sets/{id}/override` | Override primary/secondary classification | Analyst | Swaps the CCE-assigned primary and secondary positions. Body: `{ "new_primary_id": "uuid", "new_secondary_id": "uuid", "justification": "string" }`. Records override in state transition log with actor identity. Transitions status to `overridden`. |

---

### 3.5 Search and Alerts

| Method | Path | Description | Auth | Notes |
|--------|------|-------------|------|-------|
| GET | `/v1/search` | Unified search across signals, tech profiles, and advisories | Analyst, Read-only | Query param: `q` (text). Optional: `type` (signal, tech_profile, advisory), `status`, `axis`, `date_from`, `date_to`. Returns mixed-type result list with entity type indicated per item. Cursor-paginated. Sensitive signals appear with `is_sensitive: true`; content is replaced with `sensitive_ref_id`. |
| GET | `/v1/watchlists` | List all watchlists for the authenticated user | Analyst | Returns user's saved watchlists with filter configurations. |
| POST | `/v1/watchlists` | Create a new watchlist | Analyst | Body: `{ "name": "string", "filters": { ... } }`. Filter shape matches TS-07 dimensions. |
| GET | `/v1/alerts` | List alerts triggered against watchlists | Analyst | Returns unread and read alerts. Filterable by `read` status, `watchlist_id`, date range. Cursor-paginated. |
| PATCH | `/v1/alerts/{id}/read` | Mark an alert as read | Analyst | Updates `read_at` timestamp. Idempotent: marking an already-read alert returns 200 without error. |

---

## 4. Internal Service Contracts

Internal services write directly to the PostgreSQL database for performance. Each service operates under a scoped service account with the minimum permissions required for its function. These contracts are enforced at the database and application level — they are not enforced via the API Gateway.

### 4.1 Ingestion Pipeline → Core DB

**Purpose:** Write inbound signals, source records, and enrichment results from external feeds.

**Access pattern:** Direct DB write via service account.

**Permissions:**
- `INSERT` on: `signals`, `sources`, `enrichments`, `object_relationships`
- No `UPDATE`, `DELETE`, or `SELECT` access on any table

**Contract:**
- The pipeline must write all required fields per the TS-02 signal schema before committing a record. Partial writes are not permitted.
- If required fields are missing or validation fails, the record is written to the dead-letter queue with the failure reason attached. Silent partial writes are a contract violation.
- The pipeline sets `signal_state = 'new'` on all new signal records. It does not set any downstream state.
- Enrichment records written by the pipeline must include: `provider_id`, `provider_version`, `confidence_score`, `retrieved_at`. See Section 5.

### 4.2 CCE → Core DB

**Purpose:** Write match set and boundary case classification results.

**Access pattern:** Direct DB write via service account.

**Permissions:**
- `INSERT` on: `match_sets`, `boundary_cases`
- `SELECT` on: all entity tables (signals, tech_profiles, sources, enrichments, object_relationships)
- No `UPDATE` or `DELETE` access

**Contract:**
- CCE must write all required match set fields per TS-02 before committing. `is_boundary_case` is a generated column — CCE does not set it directly; it is computed from the written classification values.
- CCE does not update signal or tech profile state. Downstream state transitions triggered by classification events are handled by WorkflowStateService, not by CCE writing directly to state columns.
- CCE reads signal and entity data for classification purposes only. It does not cache entity data externally.

### 4.3 WorkflowStateService → Core DB

**Purpose:** Manage all state transitions for signals, technology profiles, advisories, and match sets.

**Access pattern:** Direct DB write via service account.

**Permissions:**
- `UPDATE` on: `current_state`, `signal_state`, `group_state`, `cluster_state` columns on their respective tables
- `INSERT` on: `state_transitions` log table
- `SELECT` on: all stateful entity tables (to read current state before transition)

**Contract:**
- Every state transition is atomic: the `current_state` update and the `state_transitions` log INSERT occur in a single transaction. A transition that cannot be logged is not committed.
- WorkflowStateService enforces all preconditions before writing (e.g., evidence chain completeness for tech profile promotions, role requirements for advisory approval). If a precondition fails, it returns a structured error to the caller — it does not partially write state.
- WorkflowStateService is the only service permitted to write to state columns. The pipeline, CCE, and extension services must not write to these columns directly.

### 4.4 Extension Services → Core API (Read-Only)

**Purpose:** Allow external extension services (ADR-011) to consume platform data without write access to core tables.

**Access pattern:** API key authenticated calls to the public API Gateway.

**Permissions:**
- Scoped to `GET` endpoints only across all resource types
- No `POST`, `PATCH`, or `DELETE` access
- Cannot access the Sensitive Content Store API regardless of role

**Contract:**
- Extension services authenticate using API keys (TS-08). Keys are issued per service, not per user.
- Extension service traffic is rate-limited separately from analyst session traffic. Rate limit configuration is per API key, configurable by role (see Section 8, decision item).
- Extensions pin to a specific API version. They are not automatically migrated to new versions. Breaking changes follow the 180-day deprecation window defined in Section 6.
- Extensions may not cache or re-serve platform data outside their own system boundary without explicit data-sharing agreement.

---

## 5. Enrichment Provider Integration

### 5.1 Contract

All enrichment providers are called exclusively by the Ingestion Pipeline. The CCE, WorkflowStateService, and extension services do not call enrichment providers directly.

Enrichment results are stored as `Enrichment` records per the TS-02 schema and linked to their parent signals via `object_relationships`.

Every enrichment provider response written to the database must include the following fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `provider_id` | string | Yes | Stable identifier for the enrichment provider |
| `provider_version` | string | Yes | Version string of the provider at time of call |
| `confidence_score` | float (0.0–1.0) | Yes | Provider-assigned confidence in the enrichment result |
| `retrieved_at` | timestamp (UTC) | Yes | Timestamp when the provider response was received |

**Provider unavailability:** If a provider is unavailable or returns an error during ingestion, the signal is ingested without that enrichment result. A retry job is queued with the signal ID, provider ID, and failure reason. Ingestion does not block on enrichment availability.

### 5.2 Provider Abstraction

Each enrichment provider is wrapped in a provider adapter that implements a standard internal interface. Adapters are responsible for:

- **Auth:** Provider-specific authentication (API key, OAuth, mTLS — per provider)
- **Rate limiting:** Per-provider rate limit enforcement, independent of the pipeline's own throughput
- **Response normalization:** Mapping provider-specific response formats to the internal `Enrichment` schema
- **Error handling:** Classifying failures as retryable vs. non-retryable; surfacing structured errors to the retry queue

Adding a new enrichment provider requires only a new adapter implementation. No changes to core pipeline logic are required.

---

## 6. API Versioning and Deprecation

**Current version:** v1

### 6.1 When a New Version Is Required

A new API version (`/v2/`, etc.) is required when:

- The shape of any response object changes (field removed, field renamed, type changed)
- The semantics of an existing field change (e.g., a status value changes meaning)
- A required request parameter is added to an existing endpoint
- An existing state transition endpoint changes its precondition logic in a breaking way

Additive, non-breaking changes (new optional response fields, new optional query parameters, new endpoints) are deployed within the current version without a version bump.

### 6.2 Deprecation Timeline

| Consumer type | Minimum notice period | Old version remains functional |
|--------------|----------------------|-------------------------------|
| Analyst clients | 90 days | 90 days after new version ships |
| Extension services (ADR-011) | 180 days | 180 days after new version ships |

Deprecation notices are communicated via the API response header `Deprecation: <date>` on affected endpoints and via documented changelog.

---

## 7. Error Response Standard

All API errors return a consistent JSON envelope. HTTP status codes are not supplemented by application-level success/failure flags — the HTTP status code is authoritative.

**Error response shape:**

```json
{
  "error": {
    "code": "string",
    "message": "string",
    "field": "string | null",
    "request_id": "uuid"
  }
}
```

| Field | Description |
|-------|-------------|
| `code` | Machine-readable error code (e.g., `VALIDATION_ERROR`, `STATE_CONFLICT`, `PRECONDITION_FAILED`) |
| `message` | Human-readable description of the error |
| `field` | The specific field that caused the error, if applicable; `null` otherwise |
| `request_id` | UUID for this request, used for log correlation |

**HTTP status code usage:**

| Code | Meaning |
|------|---------|
| 400 | Validation error — request body or query parameter is malformed or missing required fields |
| 401 | Unauthenticated — no valid session or API key provided |
| 403 | Unauthorized — authenticated but caller lacks permission for this action or resource |
| 404 | Not found — resource does not exist or caller lacks visibility |
| 409 | State conflict — requested transition is invalid from the resource's current state |
| 422 | Precondition failed — request is structurally valid but a domain precondition is unmet (e.g., evidence chain incomplete for a tech profile promotion) |
| 429 | Rate limited — caller has exceeded their rate limit; `Retry-After` header included |
| 500 | Internal server error — platform fault; `request_id` in response for log correlation |

**Note on 404 vs. 403:** Resources the caller cannot access due to role-based restrictions return 404, not 403, to avoid leaking the existence of restricted records. 403 is reserved for action-level authorization failures on resources the caller can see.

---

## 8. Open Decisions

> **DECISION NEEDED —** Should the platform expose a GraphQL API alongside REST for complex evidence chain traversal queries? Extension services navigating multi-hop evidence chains (signal → match set → tech profile → advisory) may find GraphQL's query flexibility preferable to multiple sequential REST calls. *Recommendation: REST only for MVP. Evaluate GraphQL for Phase 2 based on demonstrated extension service needs and query complexity patterns observed in production.*

> **DECISION NEEDED —** What is the rate limiting strategy for API traffic? Options: per-user-session, per-service-account (API key), per-IP, or a combination. Per-IP is unreliable for analyst clients behind proxies and insufficient for distinguishing service accounts. *Recommendation: Per-service-account rate limiting for API key traffic; per-user-session rate limiting for analyst SSO sessions. Limits are configurable per role (TS-08). IP-based limiting applies only as a blunt backstop for unauthenticated traffic at the gateway level.*

> **DECISION NEEDED —** Should the platform support webhooks for real-time event delivery to extension services (ADR-011), allowing extensions to receive push notifications rather than polling? Webhooks add infrastructure complexity (delivery guarantees, retries, secret rotation) but reduce polling load. *Recommendation: Defer to Phase 2. MVP extension services poll against `GET /v1/signals` and `GET /v1/alerts` with cursor-based pagination. Revisit webhook support when extension service count and polling load justify the infrastructure investment.*
