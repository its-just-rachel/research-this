# Tech Range Spec Registry

**Version:** v0.1  
**Status:** Build-planning draft

## Purpose

This registry lists the design specs, technical specs, and architectural decision records needed to move Tech Range from ground-truth doctrine to buildable product architecture.

## Design specs

| ID | Spec | Purpose | Build priority |
|---|---|---|---|
| DS-01 | Platform Shell and Overview | Defines the six-tab primary nav, utility nav, conditional display framework (role capability matrix), Overview tab (snapshot modal, topics highlight strip, animated radar, KPI dashboard). | High |
| DS-02 | Surface | Defines the Surface tab: ReGraph node graph (semantic zoom, signal burst detection), KronoGraph timebar, signal microblog list, TL/DR bottom drawer (archive + paste-a-link + configure-generate modes). | High |
| DS-03 | Comparative Classification UX | Defines how primary/secondary matches, confidence, differentiation, and reshuffling are shown to users. Reusable component embedded in Surface, Research, and Solution. | High |
| DS-04 | Research | Defines the Research tab: full interactive radar (DS-07 embed), knowledge catalog (Tech Profiles, Market Reports, Findings, Observations, Signal Alerts), Technology Profile detail panel, Market Report detail panel. | High |
| DS-05 | Solution Candidate Workflow | Defines how proposed engagement paths are created, reviewed, approved, and handed off to Prototype. | High |
| DS-06 | Advisory and Briefing Workflow | Defines how evidence-backed Advisories and recurring briefs are generated, reviewed, and published. | Medium |
| DS-07 | Radar and Portfolio Views | Defines the full interactive radar (Radar/Matrix/Timeline/Neighborhood Map views) embedded in DS-04 Research. | Medium |
| DS-08 | Governance and Review Queues | Defines queues for analyst review, exceptions, Emerging review, profile creation, reclassification, and approvals. | High |
| DS-09 | Signal Archaeology Workflow | Defines retrospective analysis workflows for intentional injection and surprise reviews. | Medium |
| DS-10 | Extension Boundary UX | Defines how World Models, Wargaming, Red Teaming, Forecasting, and Simulation appear as adjacent extensions. | Low / Later |
| DS-11 | User Management | Defines the admin-only utility nav page: user list, invite flow, role management, deactivation, audit trail, service accounts. | Medium |
| DS-12 | Profile and Preferences | Defines the all-user utility nav page: personal info, topics of interest (highlight strip driver), specialties, notification preferences, watchlist summary. | Medium |
| DS-13 | Prototype | Defines the Prototype tab: prototype request/kickoff workflow, active and completed prototype list, status tracking, role-differentiated actions. **Not yet written.** | High |

## Technical specs

| ID | Spec | Purpose | Build priority |
|---|---|---|---|
| TS-01 | System Context and Component Architecture | Defines major components, boundaries, and responsibilities. | High |
| TS-02 | Canonical Data Model | Translates ontology into implementation entities, tables, fields, and relationships. | High |
| TS-03 | Ingestion and Enrichment Pipeline | Defines source ingestion, provenance, extraction, enrichment, and trust boundaries. | High |
| TS-04 | Comparative Classification Engine | Defines ranked match sets, confidence, differentiation, reassignment, and review triggers. | High |
| TS-05 | Evidence Graph and Provenance | Defines lineage, relationship discipline, auditability, and temporal provenance. | High |
| TS-06 | Workflow State Machines | Defines states and transitions for Signals, Groups, Clusters, Profiles, Candidates, Prototypes, Findings, and Advisories. | High |
| TS-07 | Search, Ranking, Alerts, and Bursts | Defines discovery, retrieval, ranking, temporal shifts, and notification logic. | Medium |
| TS-08 | Security, Access Control, and Audit | Defines access boundaries, sensitive-source handling, audit logs, and data retention. | High |
| TS-09 | API and Integration Contracts | Defines service boundaries and contracts with RepoMan, Research Tool UI, enrichment providers, and future extensions. | Medium |
| TS-10 | Evaluation, Observability, and Quality Metrics | Defines how system performance, classification quality, and analyst trust are measured. | Medium |
| TS-11 | Deployment and Infrastructure | Defines environments, deployment topology, secrets, reliability, and operational runbooks. | Medium |
| TS-12 | AI Assistance and Human Oversight | Defines where AI can suggest, summarize, classify, draft, and where human review is required. | High |

## Architectural decisions

| ADR | Decision | Why it matters | Priority |
|---|---|---|---|
| ADR-001 | Repository and documentation boundary | Prevents conceptual drift and repo sprawl. | High |
| ADR-002 | Canonical storage model | Determines relational, document, graph, or hybrid persistence. | High |
| ADR-003 | Evidence graph relationship model | Determines how relationships, match confidence, and provenance are stored. | High |
| ADR-004 | Corroboration Group vs implementation alias | Resolves terminology/schema conflict with `corroboration_set`. | High |
| ADR-005 | Comparative classification and match confidence model | Determines primary/secondary matches, differentiation, and reshuffling. | High |
| ADR-006 | Fidelity and Availability decomposition | Decides whether availability splits into market, firm access, deployment readiness, and procurement readiness. | High |
| ADR-007 | Technology Profile creation governance | Decides system-created vs analyst-approved profiles and review windows. | High |
| ADR-008 | Workflow state model | Prevents object-state confusion across pipeline activities. | High |
| ADR-009 | AI autonomy and human oversight boundaries | Controls trust, review, and auditability. | High |
| ADR-010 | Radar visualization model | Keeps radar as a view, not master architecture. | Medium |
| ADR-011 | Extension boundary | Keeps World Models, Wargaming, and Red Teaming separate until promoted. | Medium |
| ADR-012 | Sensitive evidence handling | Defines how restricted evidence is referenced, masked, excluded, or linked. | High |
| ADR-013 | Advisory publication standard | Defines evidence requirements and approval flow for strategic outputs. | Medium |
| ADR-014 | Source credibility and trust model | Determines source scoring, credibility, and enrichment trust levels. | Medium |
| ADR-015 | Event-time and temporal provenance standard | Enables signal archaeology and surprise analysis. | High |
| ADR-016 | Market Reports entity | Introduces market_reports as an 18th canonical entity — living, versioned, topic-scoped research documents with forward-looking predictions tracked over time. | High |

## Immediate recommendation

Do not start with UI screens alone. Start with these three build artifacts:

1. `TS-02-canonical-data-model.md`
2. `TS-04-comparative-classification-engine.md`
3. `ADR-005-comparative-classification-and-match-confidence.md`

Those three determine whether the rest of the platform can preserve the intelligence model instead of becoming a static taxonomy dashboard.
