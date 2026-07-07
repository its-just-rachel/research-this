# ADR-012: Sensitive Evidence Handling — Pointer Pattern

**Status:** Accepted
**References:** TS-02, TS-03, TS-08, ADR-002, ADR-009

---

## Context

Tech Range operates in a defense and national security context. Analysts regularly work with signals, sources, and evidence that contain sensitive content. Categories of content treated as sensitive include:

- **Classification-marked content** — content carrying any classification or handling marking
- **Restricted program references** — references to programs with controlled access
- **Partner access-path information** — information that would reveal how or through whom data was obtained
- **Non-public source identities** — identities of sources that must not be disclosed
- **Controlled technical specifications** — technical details subject to export control, program restriction, or partner agreement
- **Client mission context** — information about a client's operational mission or priorities that is not for general circulation

### Why not column-level encryption?

Encryption protects content at rest, but it does not prevent sensitive data from entering the main database. Once content is in the main database — even encrypted — it can leak through:

- Application and infrastructure logs
- Database backups and snapshots
- Query result sets (before the application layer decrypts)
- AI processing pipelines, which ingest database content for extraction and enrichment

Column-level encryption solves the storage problem; it does not solve the exposure problem. The pointer pattern ensures sensitive content never enters those flows at all.

### Why not exclude sensitive evidence entirely?

Some of the most analytically valuable intelligence comes from sensitive sources. Excluding it would make the platform materially less useful for its primary purpose. The platform must be able to support analysis in high-sensitivity contexts, not just open-source contexts.

### The gap the pointer pattern fills

Analysts must be able to reference sensitive evidence in their analytical work — in Hypotheses, Tech Evaluations, Findings, and Advisories — without that evidence being exposed outside the controlled access path. The pointer pattern provides this: the main database carries only an opaque reference; the actual content lives in a separate, access-controlled system and is retrieved only by authorized users through a logged access path.

---

## Decision

All sensitive content is stored in the **Sensitive Content Store** — a separate, access-controlled system. The main platform database stores only three fields:

| Field | Type | Purpose |
|---|---|---|
| `sensitive_ref_id` | `UUID` | Opaque reference to the content in the Sensitive Content Store |
| `sensitivity_level` | `TEXT` | Categorical marker (e.g. `'restricted'`, `'confidential'`, `'internal'`) |
| `sensitivity_reason` | `TEXT` | Brief, non-revealing description of why this content is sensitive (e.g. `"partner access path"`, `"restricted program reference"`, `"non-public source identity"`) |

The `sensitivity_reason` field must not itself reveal sensitive content — it describes the category of sensitivity, not the content.

---

## Where the Pointer Pattern Applies

The following entity types have fields that may carry a `sensitive_ref_id`:

| Entity | Fields That May Carry `sensitive_ref_id` |
|---|---|
| **Signal** | `raw_content_ref` (the raw signal content); `summary` (if the summary itself reveals sensitive content) |
| **Source** | Source identity (when the source is non-public or must not be disclosed) |
| **Enrichment** | Enrichment content |
| **Observation** | Observation text |
| **Hypothesis** | Hypothesis text |
| **Tech Evaluation** | Evaluation details; recommendation |
| **Finding** | Finding text |
| **Advisory** | Specific sections (flagged individually) |

When a field carries a `sensitive_ref_id`, the corresponding plain-text field for that slot is left null or replaced with a non-revealing placeholder. The entity record is still fully functional in the main database; the content is simply not present there.

---

## What Must NOT Happen

Sensitive content must never:

- **Be logged** in `audit_log.before_state` or `audit_log.after_state` — audit log entries for entities with sensitive references must record the reference UUID only, not the content
- **Be passed to AI processing pipelines** — Tier 1 autonomous AI (TS-03 signal extraction and enrichment) must not process sensitive content; signals flagged with a `sensitive_ref_id` are excluded from autonomous AI processing paths
- **Be included in API responses** to clients that have not been explicitly granted access to the specific reference
- **Be included in radar or portfolio views** without an access control check at render time
- **Be stored in any cache layer** without the same access controls as the Sensitive Content Store itself — caching sensitive content in a lower-control cache (CDN, application cache, read replica) is prohibited

---

## Access Control for Sensitive Content

**Who may follow a `sensitive_ref_id` to retrieve actual content:**

- Role-based minimum: `senior_analyst` role and above
- For the most sensitive content: explicit per-reference access grants, in addition to role

**Every retrieval must be logged** in the audit trail (see TS-08). The log entry records: the user, the reference UUID, the timestamp, and the access path (UI, API, or export). The content itself is not logged.

**Access grants are time-limited and revocable.** Expiry and revocation do not automatically invalidate analytical work that cited the reference — see Dangling Pointer Handling below.

---

## Evidence Chain Integrity

### Valid but not fully traversable

When a `sensitive_ref_id` is used as `evidence_basis` in an `object_relationships` record, the evidence chain is structurally valid. The relationship exists and is recorded. However, the content at the end of that chain is not traversable by unauthorized users — they will see the reference UUID and the `sensitivity_reason`, but not the content.

### Advisory publication gate

If an Advisory's evidence chain contains one or more sensitive references, the Advisory must:

1. Carry a `sensitivity_flag` at the Advisory level
2. Go through an **elevated approval process** — standard publication approval is not sufficient
3. Have the sensitivity flag visible to any downstream consumer of the Advisory

### Dangling pointer handling

A sensitive reference becomes a **dangling pointer** when the content in the Sensitive Content Store is deleted or access is fully withdrawn. When this occurs:

- The reference UUID remains in the main database
- The entity carrying the reference must surface a **warning to analysts** (e.g., "Evidence reference [UUID] is no longer accessible — content has been removed or access withdrawn")
- The Advisory or Finding that cited the reference is **not automatically invalidated** — it is the analyst's responsibility to review and determine whether the work remains supportable
- The dangling state is recorded in the audit trail

---

## Consequences

### Positive

- Sensitive content never enters non-controlled flows — logs, AI pipelines, backups, and API responses are clean
- Analysts can work with sensitive evidence and include it in formal analytical products
- The platform is usable in high-sensitivity contexts without requiring a separate deployment or shadow workflow
- The audit trail for sensitive content access is complete and independent of the main audit log

### Negative

- Evidence chains citing sensitive content are not fully auditable by all analysts — reviewers without access see an opaque reference
- Dangling pointer handling adds operational and UI complexity
- The Sensitive Content Store is a second system to operate, with its own schema, access control, backup, and availability requirements
- Tier 1 AI processing cannot operate on sensitive signals — some signals will require manual analyst processing

---

## DECISION NEEDED Items

> **DECISION NEEDED —** What is the actual sensitivity level vocabulary (`'restricted'`, `'confidential'`, `'internal'`, and any others)? This spec uses placeholder values. The real vocabulary must align with the enterprise's classification and handling framework.
> *Recommendation: [Internal decision rule] — this vocabulary must be defined in the enterprise-controlled environment and must not appear in this public-safe spec. A separate internal-only appendix should define the mapping.*

> **DECISION NEEDED —** Who operates the Sensitive Content Store — is it a separate team, a separate tool, or built as part of Tech Range?
> *Recommendation: Built as part of Tech Range Phase 1, operated by the same team, with strict schema and access separation enforced at the database and API layer. A separate team or tool adds operational overhead that is not justified at this stage; the separation is enforced by architecture, not by organizational boundary.*

> **DECISION NEEDED —** What happens to an Advisory that cited sensitive evidence if that evidence is later fully declassified — does it retain its sensitivity flag, or can the flag be removed?
> *Recommendation: The sensitivity flag is manually removable by a `senior_analyst` or above, after confirming declassification. The audit trail records the removal, the user who made it, and the stated basis for declassification. The flag is not removed automatically — a human must make the call.*
