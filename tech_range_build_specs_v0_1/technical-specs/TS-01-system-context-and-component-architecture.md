# TS-01 — System Context and Component Architecture

**Version:** 0.1  
**Status:** Draft  
**Last updated:** 2026-07-06  
**Spec series:** Technical Specifications

---

## 1. Purpose

This spec establishes the authoritative system context and component architecture for Tech Range. It defines what Tech Range is at the system boundary level, names and describes every major component, maps data flow from raw signal ingestion to advisory publication, and sets build phase assignments. All other technical specs (TS-02 through TS-12) and design specs (DS-01 through DS-09) describe subsystems within the context established here.

Tech Range is a signal-first technology intelligence platform built for a CTO function operating in defense and national security. It is an active intelligence pipeline — not a dashboard or document repository. It moves technology signals through a five-stage workflow (Surface → Research → Solution → Prototype → Advise) with human-in-the-loop review gates at each transition, producing structured, provenance-backed intelligence that supports technology acquisition decisions.

---

## 2. System Context Diagram

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                          EXTERNAL SIGNAL SOURCES                             ║
║  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ ║
║  │ Academic &   │  │  Patent      │  │  News, RSS,  │  │  Gov / Defense   │ ║
║  │ Preprint     │  │  Databases   │  │  Newsletters │  │  Contract DBs    │ ║
║  │ Repositories │  │  (USPTO etc) │  │  Podcasts    │  │  (SAM.gov etc)   │ ║
║  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘ ║
╚═════════╪═════════════════╪═════════════════╪════════════════════╪═══════════╝
          │                 │                 │                    │
          └─────────────────┴────────┬────────┴────────────────────┘
                                     │ raw signals (pull/push)
                          ╔══════════╪═══════════════════════════════╗
                          ║   TECH RANGE PLATFORM BOUNDARY           ║
                          ║                                          ║
                          ║  ┌───────────────────────────────────┐   ║
                          ║  │  Ingestion & Enrichment Pipeline  │   ║
                          ║  │  (TS-03)                          │   ║
                          ║  └────────────────┬──────────────────┘   ║
                          ║                   │ normalized signals    ║
                          ║  ┌────────────────▼──────────────────┐   ║
                          ║  │  Canonical Data Store             │   ║
                          ║  │  (TS-02) — PostgreSQL             │◄──╫──── Enrichment Providers
                          ║  │  17 entity types                  │   ║     (Diffbot, Crunchbase,
                          ║  └──┬────────────┬────────┬──────────┘   ║      Clearbit, etc.)
                          ║     │            │        │              ║
                          ║  ┌──▼───┐  ┌────▼──┐  ┌─▼────────────┐ ║
                          ║  │ CCE  │  │Evidence│  │  Workflow    │ ║
                          ║  │(TS-04│  │ Graph  │  │  State       │ ║
                          ║  │ )    │  │(TS-05) │  │  Machine     │ ║
                          ║  └──┬───┘  └────┬───┘  │  (TS-06)    │ ║
                          ║     │           │       └─────┬────────┘ ║
                          ║     └─────┬─────┘             │          ║
                          ║           │         ┌──────────▼───────┐ ║
                          ║  ┌────────▼───────┐ │  Search, Ranking │ ║
                          ║  │  AI Assistance │ │  & Alert Service │ ║
                          ║  │  Layer (TS-12) │ │  (TS-07)         │ ║
                          ║  └────────┬───────┘ └──────────┬───────┘ ║
                          ║           │                     │         ║
                          ║  ┌────────▼─────────────────────▼──────┐ ║
                          ║  │  API & Integration Gateway (TS-09)  │ ║
                          ║  └──────┬──────────────────────┬───────┘ ║
                          ║         │                      │         ║
                          ║  ┌──────▼──────────┐   ┌──────▼───────┐ ║
                          ║  │ Analyst         │   │ Security,    │ ║
                          ║  │ Workspace UI    │   │ Access &     │ ║
                          ║  │ (DS-01–DS-09)   │   │ Audit        │ ║
                          ║  └────────┬────────┘   │ (TS-08)      │ ║
                          ║           │             └──────────────┘ ║
                          ╚═══════════╪══════════════════════════════╝
                                      │
               ┌──────────────────────┼──────────────────────┐
               │                      │                       │
        ┌──────▼──────┐    ┌──────────▼──────┐    ┌─────────▼──────────┐
        │  Analysts   │    │  Leadership /   │    │  Sensitive Content │
        │  (internal  │    │  Consumers      │    │  Store             │
        │  users)     │    │  (read-only     │    │  (internal-only,   │
        │             │    │  advisories)    │    │  ref pointer in    │
        └─────────────┘    └─────────────────┘    │  main DB only)     │
                                                   └────────────────────┘

  ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
  │  EXTENSION BOUNDARY (future, not in MVP scope)                         │
  │                                                                         │
  │  ┌─────────────────┐   ┌──────────────────┐   ┌──────────────────────┐ │
  │  │  World Models   │   │  Wargaming       │   │  Red Team /          │ │
  │  │  (structured    │   │  (scenario +     │   │  Adversarial         │ │
  │  │  tech futures)  │   │  war game engine)│   │  Assessment Engine   │ │
  │  └─────────────────┘   └──────────────────┘   └──────────────────────┘ │
  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
```

---

## 3. Component Inventory

| Component | Responsibility | Spec Reference | Build Phase |
|---|---|---|---|
| Ingestion & Enrichment Pipeline | Pulls, normalizes, and enriches raw signals from all external sources; produces canonical Signal records | TS-03 | Phase 1 (MVP) |
| Comparative Classification Engine (CCE) | Assigns ranked classification sets across four axes — Neighborhood, Domain Area, Technology Type, and Candidate Type / Engagement Posture — to every signal and knowledge artifact | TS-04 | Phase 1 (MVP) |
| Canonical Data Store | Single PostgreSQL store for all 17 entity types; source of truth for everything except sensitive-flagged content bodies | TS-02 | Phase 1 (MVP) |
| Evidence Graph & Provenance Service | Tracks and serves the relationship lineage between evidence, knowledge, and action artifacts; powers provenance views | TS-05 | Phase 1 (MVP) |
| Workflow State Machine Service | Enforces valid state transitions across the five pipeline stages; gates human-in-the-loop review steps | TS-06, ADR-008 | Phase 1 (MVP) |
| Analyst Workspace (UI layer) | Provides the full analyst experience: inbox, signal review, research workspace, advisory authoring, admin | DS-01 – DS-09 | Phase 1 (MVP) |
| Search, Ranking & Alert Service | Full-text and semantic search across all entities; relevance ranking; alert rules engine; digest generation | TS-07 | Phase 1 (MVP) |
| AI Assistance Layer | Provides LLM-backed extraction, summarization, gap detection, and draft generation across the pipeline | TS-12 | Phase 1 (MVP, constrained) |
| API & Integration Gateway | Single authenticated entry point for all UI, CLI, and external integration traffic; enforces authorization | TS-09 | Phase 1 (MVP) |
| Security, Access Control & Audit Service | Role and clearance-level enforcement; immutable audit log; session management | TS-08 | Phase 1 (MVP) |
| Sensitive Content Store | Separate store for content bodies marked sensitive; main DB holds only a `sensitive_ref_id` pointer | TS-08 (section TBD) | Phase 1 (MVP) |
| Extension Boundary — World Models | Structured forward-looking technology scenario models; consumes advisory outputs | TS-TBD | Phase 3 (deferred) |
| Extension Boundary — Wargaming | Scenario and wargame engine consuming Tech Range intelligence | TS-TBD | Phase 3 (deferred) |
| Extension Boundary — Red Team | Adversarial assessment engine; stress-tests advisory conclusions | TS-TBD | Phase 3 (deferred) |

---

## 4. Component Descriptions

### 4.1 Ingestion & Enrichment Pipeline

The Ingestion & Enrichment Pipeline is the entry point for all external information into Tech Range. It is responsible for pulling or receiving raw content from external signal sources (academic preprints, patent databases, news feeds, government contract databases, and manually submitted documents), normalizing that content into a canonical Signal record format, and driving the enrichment process through configured enrichment providers (metadata, entity extraction, translation where applicable). The pipeline owns the raw-to-normalized transformation logic, connector configurations, deduplication, and the enrichment job queue.

The pipeline does NOT own classification or ranking — those are delegated to the CCE after a signal is persisted to the Canonical Data Store. It does NOT own workflow state — a signal entering the system begins in the `SURFACE` state and the Workflow State Machine owns all transitions from that point. Key interfaces: publishes normalized Signal records to the data store via an internal event; exposes a connector management API consumed by the admin UI; emits enrichment status events consumed by the alert service.

### 4.2 Comparative Classification Engine (CCE)

The Comparative Classification Engine assigns ranked classification sets to every signal, research record, solution record, and advisory that passes through the pipeline. For each artifact it produces classifications across four axes: Neighborhood (which Core Technology Neighborhood — Agentic/Vertical AI, Edge/AI-RAN, Physical AI, Quantum, SDx, Cloud Platforms, or Emerging), Domain Area (National Security & Cyber, Defense Tech, Global Defense, Civil, Space Systems, Vertical AI/Data Platforms), Technology Type (framework, platform, model, technique, etc.), and Candidate Type / Engagement Posture (for Solution Candidates only). Classification is comparative — the CCE does not use fixed keyword rules alone; it compares the artifact against the existing corpus using a combination of embedding similarity and rule-based heuristics. The CCE owns classification models, the classification result schema, and re-classification job logic.

The CCE does NOT own the underlying entity taxonomy (Neighborhoods, Domain Areas, Technology Types) — those are master data records in the Canonical Data Store. It does NOT own the evidence relationship graph — results from the CCE are inputs that the Evidence Graph service links into provenance chains. Key interfaces: consumes normalized Signal records and artifact mutation events from the data store; writes classification results back as structured metadata on those records; exposes a re-classification trigger endpoint for analyst-initiated overrides.

### 4.3 Canonical Data Store

The Canonical Data Store is the single source of truth for all structured data in Tech Range. It is a PostgreSQL database that holds all 17 entity types defined in TS-02, including Signals, Technology Areas, Threat Vectors, Organizations, People, Solutions, Prototypes, Advisories, and their join and metadata tables. All services that need to persist or query structured data do so through this store. The data store owns the schema, migrations, and referential integrity rules.

The data store does NOT hold the body content of sensitive-flagged artifacts — it holds only a `sensitive_ref_id` pointer to the Sensitive Content Store. It does NOT own application-level business logic — queries and mutations arrive via the API Gateway or internal service calls; the data store executes them but does not enforce workflow rules. Key interfaces: exposes a connection pool to internal services; schema migrations are version-controlled and applied by the deployment pipeline; the Evidence Graph service maintains a separate but linked graph layer over the relational records.

### 4.4 Evidence Graph & Provenance Service

The Evidence Graph & Provenance Service tracks the full lineage of every knowledge and action artifact in the system — which signals supported which research conclusions, which research records informed which solutions, which solutions were cited in which advisories. It models these relationships as a directed acyclic graph where nodes are entity records in the Canonical Data Store and edges are typed provenance relationships (SUPPORTS, REFUTES, CITED_BY, DERIVED_FROM, etc.). The service owns the graph data model, the provenance edge write path, and the graph traversal API.

The Evidence Graph Service does NOT own the entity records themselves — those live in the Canonical Data Store. It does NOT enforce workflow rules — it is a read/write substrate for provenance relationships, not a state machine. Key interfaces: listens to artifact mutation events to auto-create provenance edges; exposes a graph traversal query API consumed by the Analyst Workspace to render provenance views; exposes a bulk provenance export endpoint consumed by the advisory authoring flow.

### 4.5 Workflow State Machine Service

The Workflow State Machine Service enforces valid state transitions for every artifact moving through the five pipeline stages: Surface → Research → Solution → Prototype → Advise. It owns the transition rules, the human-in-the-loop review gate logic, and the state history log. It is the authoritative service for answering "what state is this artifact in?" and "what transitions are currently permitted for this user?". It emits transition events that drive alert notifications and UI state updates.

The service does NOT own the work performed within each stage — that belongs to the analysts and to services like the CCE and AI Assistance Layer. It does NOT own access control decisions beyond transition authorization — role and clearance enforcement is delegated to the Security service, which the state machine calls before committing any transition. Key interfaces: exposes a transition request endpoint (artifact ID + target state + actor); subscribes to artifact creation events to initialize state records; publishes transition events to the event bus consumed by the Alert Service and Analyst Workspace.

### 4.6 Analyst Workspace (UI Layer)

The Analyst Workspace is the primary interface through which analysts interact with every stage of the intelligence pipeline. It provides: an inbox and triage view for surfaced signals (DS-01 through DS-03), a research workspace for annotating and linking evidence (DS-04, DS-05), a solution and prototype review surface (DS-06, DS-07), an advisory authoring and publication interface (DS-08), and administrative surfaces for taxonomy and connector management (DS-09). The UI layer owns presentation logic, form state, and local filtering/sorting behavior.

The Analyst Workspace does NOT own any business logic — it calls the API Gateway for all data reads and writes. It does NOT enforce access control rules — it renders what the API returns and shows affordances based on permissions returned in the API response, but the API Gateway and Security service enforce those rules server-side. Key interfaces: consumes the REST/GraphQL API exposed by the API Gateway; subscribes to a real-time notification channel (WebSocket or SSE) pushed by the Alert Service; uses the Evidence Graph traversal API for provenance visualization components.

### 4.7 Search, Ranking & Alert Service

The Search, Ranking & Alert Service provides three related capabilities: full-text and semantic search across all entity types in the Canonical Data Store; relevance ranking that surfaces the most pertinent signals and artifacts for a given analyst context; and an alert rules engine that evaluates new content against analyst-defined or system-defined alert criteria and dispatches notifications. It owns search indexes (maintained in sync with the data store), ranking model configuration, alert rule definitions, and notification delivery.

The service does NOT own the underlying entity data — it indexes from the Canonical Data Store and treats that store as the record of truth. It does NOT own authentication or delivery channel credentials — notification dispatch routes through the API Gateway and external delivery integrations. Key interfaces: subscribes to artifact creation and mutation events to update indexes; exposes a search query API consumed by the Analyst Workspace and API Gateway; exposes an alert rule management API; dispatches notification events to a delivery queue.

### 4.8 AI Assistance Layer

The AI Assistance Layer provides LLM-backed capabilities that augment but do not replace analyst judgment at each pipeline stage. In Phase 1 (MVP, constrained scope) this includes: entity and claim extraction from ingested documents, auto-generated signal summaries, classification confidence explanations, and research gap flagging. In Phase 2 this expands to draft section generation for advisories and semantic similarity suggestions. The layer owns prompt templates, LLM provider configuration, output parsing and validation logic, and the feedback capture mechanism that lets analysts accept, reject, or edit AI-generated content.

The AI Assistance Layer does NOT make autonomous workflow decisions — every AI-generated output is a suggestion that requires analyst review before it enters the provenance chain as accepted evidence. It does NOT own the entity taxonomy or classification rules — it consumes those from the Canonical Data Store via the CCE. Key interfaces: called by the Ingestion Pipeline post-normalization for extraction and summarization; called by the Analyst Workspace on demand for contextual assistance; writes accepted AI outputs back via the API Gateway with a provenance edge indicating AI origin and analyst acceptance.

> **DECISION NEEDED —** Whether the AI Assistance Layer is a separate service or capabilities embedded within the CCE and the Analyst Workspace API. *Recommendation: implement as a separate internal service in Phase 1. Embedding it creates tight coupling between classification logic and LLM prompt management, and a separate service allows independent scaling, model swaps, and cleaner prompt version control. The CCE and Analyst Workspace call it via an internal API.*

### 4.9 API & Integration Gateway

The API & Integration Gateway is the single authenticated entry point for all traffic into Tech Range — from the Analyst Workspace UI, from CLI tooling, and from any future external integrations. It owns API versioning, request authentication (token validation, session verification), authorization delegation to the Security service, rate limiting, and the API schema (REST and/or GraphQL). It is the only component that external clients are permitted to call directly.

The Gateway does NOT own authorization logic — it calls the Security service for permission checks before forwarding requests to internal services. It does NOT own business logic — it routes, validates, and enforces transport-level concerns, then delegates to the appropriate downstream service. Key interfaces: ingress from all external callers; internal fan-out to Canonical Data Store, Workflow State Machine, Search Service, Evidence Graph, and AI Assistance Layer; authentication token validation via the Security service.

### 4.10 Security, Access Control & Audit Service

The Security, Access Control & Audit Service enforces role-based and clearance-level access control across all Tech Range resources, maintains an immutable audit log of every read and write action, and manages analyst session lifecycle. It owns the permission model (roles, clearance levels, resource-level ACLs), the audit log store, and the session token lifecycle. Every state transition, data mutation, and sensitive-content access that occurs anywhere in the platform is recorded here.

The Security service does NOT own authentication credentials — those are managed by the identity provider (IdP) integration configured for the deployment. It does NOT own data encryption at rest — that is a deployment-topology concern addressed in TS-11. Key interfaces: exposes an authorization check endpoint consumed by the API Gateway (and directly by the Workflow State Machine for transition authorization); receives audit events from all services via an internal event bus; exposes an audit log query API consumed by admin surfaces in the Analyst Workspace.

### 4.11 Sensitive Content Store

The Sensitive Content Store is a physically separate store for the body content of artifacts that have been flagged as sensitive (classification-restricted, source-protected, or otherwise restricted from the standard access tier). When an analyst or ingestion process flags an artifact as sensitive, the content body is written to this store and the main Canonical Data Store record retains only a `sensitive_ref_id` pointer. The sensitive store owns its own access log, its own encryption configuration, and its own access authorization tier.

The Sensitive Content Store does NOT store metadata about sensitive artifacts — structured metadata (entity relationships, classification results, workflow state) remains in the Canonical Data Store and is governed by standard role and clearance-level rules. Access to content via the `sensitive_ref_id` is gated by the Security service at a higher clearance level than standard records. Key interfaces: write endpoint called by the Ingestion Pipeline and Analyst Workspace when sensitivity flag is set; read endpoint called by the API Gateway only after the Security service confirms clearance-level authorization.

> **DECISION NEEDED —** Whether the Sensitive Content Store is implemented as a physically separate database instance or as a separate schema with row-level security (RLS) within the main PostgreSQL cluster. *Recommendation: separate database instance for MVP. RLS is operationally simpler but introduces risk of misconfiguration exposing sensitive rows through the same connection pool. A separate instance with separate credentials provides a hard boundary and is easier to audit, restrict, and selectively air-gap in future deployment topologies.*

### 4.12 Extension Boundary (Future: World Models, Wargaming, Red Team)

The Extension Boundary represents three future capabilities that consume Tech Range intelligence outputs but are not part of the core intelligence pipeline: World Models (structured forward-looking technology scenario models built from advisory outputs), Wargaming (a scenario and wargame engine that stress-tests technology acquisition decisions), and Red Team (an adversarial assessment engine that challenges advisory conclusions). These are architecturally deferred but the API Gateway and data export formats defined in Phase 1 must not foreclose them.

The Extension Boundary components do NOT mutate records in the Canonical Data Store in Phase 1 — they are consumers of published advisories and exported knowledge graphs, not contributors to the evidence provenance chain. In Phase 3, a bi-directional interface may be defined to allow wargame findings to feed back as evidence into new research cycles. Key interface requirement for Phase 1: the API Gateway must expose a stable advisory export endpoint and a structured knowledge graph export format that future extension services can consume without requiring changes to core internals.

---

## 5. Data Flow — Primary Path

The following traces a single Signal from external ingestion through to Advisory publication.

**Trigger:** The Ingestion Pipeline's scheduler fires a pull job for a configured source connector (e.g., arXiv RSS feed for a monitored technology area).

1. **Pull & normalize.** The Ingestion Pipeline fetches new documents from the source, deduplicates against known Signal fingerprints in the Canonical Data Store, and normalizes each new document into a canonical Signal record with source metadata, raw text body, and ingestion timestamp. If the document is flagged as sensitive by connector configuration or content rules, the body is written to the Sensitive Content Store and the Signal record receives a `sensitive_ref_id` in place of the body.

2. **Enrich.** The pipeline dispatches enrichment jobs — entity extraction (organizations, people, technologies), metadata augmentation from configured enrichment providers, and language detection. Enriched fields are written back to the Signal record. The AI Assistance Layer is called to generate a structured summary and extract key claims.

3. **Classify.** The Canonical Data Store mutation triggers the CCE. The CCE evaluates the normalized and enriched Signal record and writes ranked classification results across four axes (Neighborhood, Domain Area, Technology Type, and Candidate Type / Engagement Posture where applicable) onto the Signal record.

4. **Initialize workflow state.** The Workflow State Machine Service detects the new Signal record and initializes its state as `SURFACE`. The Signal is now visible in the analyst inbox.

5. **Analyst review — Surface stage.** An analyst opens the Signal in the Analyst Workspace inbox view. They review the summary, classification, and source. They may accept the CCE classification, override it, or add annotations. When satisfied, they trigger a state transition to `RESEARCH`. The Workflow State Machine validates the transition and the analyst's authorization, records the transition in the state history, and publishes a transition event.

6. **Research stage.** In the Research workspace, the analyst links the Signal to existing Knowledge records, adds supporting or refuting evidence from related Signals, and authors or refines Research records. Each link written creates a provenance edge in the Evidence Graph Service. The AI Assistance Layer may be invoked to flag gaps in the evidence base. When the analyst is satisfied, they transition the artifact to `SOLUTION`.

7. **Solution and Prototype stages.** The analyst or team evaluates solution options and may commission or link prototype assessments. Each solution and prototype record is classified by the CCE and linked into the evidence graph. State transitions through `SOLUTION` and `PROTOTYPE` follow the same Workflow State Machine pattern.

8. **Advisory authoring.** When the artifact enters the `ADVISE` stage, the analyst opens the advisory authoring interface. The Evidence Graph Service provides a full provenance trace of all evidence supporting the advisory. The AI Assistance Layer may generate draft sections. The analyst reviews, edits, and finalizes the advisory.

9. **Publication.** The analyst triggers advisory publication. The Workflow State Machine transitions the advisory to `PUBLISHED`. The Search & Alert Service indexes the advisory and evaluates it against alert rules — dispatching notifications to relevant subscribers. Leadership consumers access the published advisory via the API Gateway (read-only, no workflow mutation capability).

10. **Audit trail.** At every step where a data mutation or state transition occurs, the Security & Audit Service records an immutable audit event with actor, action, resource, and timestamp.

---

## 6. Key Architectural Boundaries

### 6.1 API Gateway boundary (external-facing)

Everything external to the platform — the Analyst Workspace UI, CLI tooling, leadership read-only consumers, and future extension services — communicates exclusively through the API Gateway. No service inside the platform exposes a direct external endpoint. The API Gateway enforces authentication and delegates authorization to the Security service before any request reaches an internal service.

### 6.2 Internal service boundary (service-to-service)

Internal services communicate via direct internal API calls (REST or gRPC, TBD in TS-09) and via an internal event bus for asynchronous mutations. Internal service traffic does not traverse the API Gateway. Services trust the internal network for transport but still validate event payloads and call the Security service for authorization checks on sensitive-content access.

### 6.3 Sensitive Content boundary

The Sensitive Content Store sits behind an additional authorization layer. Even within the internal service network, no service reads sensitive content bodies without first receiving a positive clearance-level authorization response from the Security service. The `sensitive_ref_id` in the main data store is the only cross-boundary pointer; the full content never transits through services that do not hold explicit need-to-know authorization.

### 6.4 Extension boundary (future)

The Extension Boundary components (World Models, Wargaming, Red Team) are consumers of published advisory data and structured knowledge graph exports. In Phase 1, these are output-only integrations: Tech Range publishes; extension services consume. Extension services do NOT write into the Canonical Data Store or influence workflow state in Phase 1. A bi-directional interface design is deferred to Phase 3.

---

## 7. Build Phase Mapping

| Component | Phase | Notes |
|---|---|---|
| Canonical Data Store | Phase 1 — MVP | Full 17-entity schema; sensitive_ref_id pointer from day one |
| Ingestion & Enrichment Pipeline | Phase 1 — MVP | Initial connector set TBD in TS-03; manual submission supported from day one |
| Comparative Classification Engine (CCE) | Phase 1 — MVP | Initial taxonomy seeded from master data; heuristic + embedding models |
| Evidence Graph & Provenance Service | Phase 1 — MVP | Core provenance edges; advanced graph traversal queries may be Phase 2 |
| Workflow State Machine Service | Phase 1 — MVP | All five stages; transition history |
| Analyst Workspace UI | Phase 1 — MVP | Core flows per DS-01 – DS-09; advanced visualizations may be Phase 2 |
| Search, Ranking & Alert Service | Phase 1 — MVP | Full-text search and basic alert rules; semantic ranking in Phase 2 |
| AI Assistance Layer | Phase 1 — MVP (constrained) | Extraction, summarization, gap flagging only; draft generation in Phase 2 |
| API & Integration Gateway | Phase 1 — MVP | REST API; GraphQL optional in Phase 2 |
| Security, Access Control & Audit Service | Phase 1 — MVP | Role + clearance model; immutable audit log |
| Sensitive Content Store | Phase 1 — MVP | Separate instance; basic clearance-level gating |
| World Models | Phase 3 — Deferred | Depends on mature advisory output format and corpus volume |
| Wargaming Engine | Phase 3 — Deferred | Depends on World Models and advisory corpus |
| Red Team Engine | Phase 3 — Deferred | Depends on advisory corpus and wargaming outputs |

> **DECISION NEEDED —** Confirm the MVP boundary for the AI Assistance Layer in Phase 1. The recommendation above scopes it to extraction, summarization, and gap flagging only. If draft generation is required for MVP to demonstrate value to leadership, this scope must expand and the associated LLM cost, latency, and review workflow burden must be accounted for in the Phase 1 build estimate. *Recommendation: hold the constrained scope for MVP; add draft generation as the first Phase 2 increment so it can be validated on a live advisory corpus rather than synthetic data.*

---

## 8. What This Spec Does Not Cover

The following topics are intentionally out of scope for TS-01 and are addressed in their designated specifications:

- **Deployment topology** — how services are packaged, containerized, orchestrated, and deployed across environments (dev, staging, production). See TS-11.
- **Secrets management** — how credentials, API keys, and encryption keys are stored, rotated, and injected into services at runtime. See TS-11.
- **Detailed database schema** — the full 17-entity type schema with field definitions, indexes, and migration strategy. See TS-02.
- **Connector catalog and enrichment provider contracts** — the specific external sources integrated in Phase 1 and their adapter interfaces. See TS-03.
- **Classification model details** — embedding model selection, taxonomy seeding, confidence scoring methodology. See TS-04.
- **API schema and versioning policy** — endpoint definitions, request/response contracts, versioning. See TS-09.

---

## 9. Decision Items

> **DECISION NEEDED —** MVP boundary: which components are fully in scope for v1 vs. deferred to Phase 2 or Phase 3? The component inventory and build phase mapping in this spec represent the current recommended baseline. Before Phase 1 build planning begins, the team must confirm which components are gating for the initial analyst pilot and which can ship as stubs (e.g., the AI Assistance Layer, semantic search ranking). *Recommendation: convene a scoping session with the build lead and product owner to ratify the Phase 1 list in Section 7 as the formal MVP boundary. Any component not ratified should have an explicit stub or mock strategy defined so Phase 1 does not produce dependency blockers for Phase 2.*

> **DECISION NEEDED —** Whether the AI Assistance Layer is a separate internal service or capabilities embedded within the CCE and the Analyst Workspace API. See Section 4.8 for full context. *Recommendation: separate internal service. Embedding creates tight coupling, complicates independent scaling, and mixes prompt management with classification and UI concerns. A separate service with a clean internal API is more maintainable and testable.*

> **DECISION NEEDED —** Whether the Sensitive Content Store is a physically separate database instance or a separate schema with row-level security (RLS) within the main PostgreSQL cluster. See Section 4.11 for full context. *Recommendation: separate database instance for Phase 1. Provides a hard technical boundary, independent credentials, and a simpler audit story. RLS can be reconsidered in Phase 2 if operational overhead of maintaining two database instances proves excessive relative to the threat model.*

---

*End of TS-01*
