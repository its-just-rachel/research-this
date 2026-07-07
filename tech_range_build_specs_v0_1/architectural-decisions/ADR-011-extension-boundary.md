# ADR-011: Extension Boundary — Core Platform vs. Adjacent Capabilities

**Status:** Accepted
**Date:** 2026-07-06
**Authors:** Platform Architecture
**Related:** TS-01 (Component Inventory), TS-09 (API Contracts), DS-10 (Extension Boundary UX)

---

## Context

### The Extensions

Tech Range's core intelligence pipeline covers five stages: **Surface → Research → Solution → Prototype → Advise**. Adjacent to this pipeline are five capabilities that consume its outputs but are not part of it:

| Extension | What It Does |
|---|---|
| **World Models** | Structured representations of how a technology domain works — actors, dependencies, dynamics, feedback loops |
| **Wargaming** | Adversarial simulation using technology intelligence; tests strategic positions against opponent moves |
| **Red Teaming** | Adversarial challenge of platform conclusions — systematically attacks published Advisories and Solution Candidates to find weaknesses |
| **Forecasting** | Quantitative prediction of technology trajectories; assigns probability distributions to future states |
| **Simulation** | Capability modeling — simulates how a technology or capability set performs under defined conditions |

These extensions are strategically valuable. They make the platform more powerful. Leadership frequently asks for them. That is precisely why the boundary must be explicit.

### Why Premature Integration Is Harmful

Each extension has its own data model requirements that can distort the core schema if integrated early:

- **World Models** want to annotate Technology Profiles with domain-graph metadata (node weights, edge types, causal links). Adding these columns to core tables before the World Model data model is stable creates schema debt.
- **Wargaming** wants to create versioned "scenario snapshots" of the intelligence state. This requires point-in-time copies or versioning logic that the core platform does not need.
- **Red Teaming** wants to attach adversarial findings directly to Advisories. If implemented as direct writes, it bypasses the evidence graph and creates a parallel, unaudited amendment path.
- **Forecasting** wants to store probability distributions against Technology Profiles and Clusters. These are nullable, sparsely populated, and only meaningful in a forecasting context — they do not belong in the core schema.
- **Simulation** requires parameterized capability definitions that have no equivalent in the core intelligence model.

Building extensions before the core is stable creates two compounding problems: (1) the core schema gets columns it doesn't own, driven by extension needs, and (2) when extension requirements change — and they will — the core schema changes with them. This produces rework at the layer that is most expensive to change.

Extensions also require different expertise and tooling. Simulation and Wargaming require subject-matter expertise in adversarial analysis. Forecasting requires probabilistic modeling. Attempting to staff and build these concurrently with the core intelligence pipeline divides team focus during the period when focus matters most.

**TS-01** already designates all five extensions as Phase 3 deferred. This ADR formalizes the architectural reasoning behind that deferral and establishes the rules that govern how extensions interact with the core platform — both now and when they are eventually built.

---

## Decision

Extensions are kept behind a formal boundary. The boundary is enforced by the following rules:

### The Boundary Rules

1. **Extensions may read from the core platform** — Technology Profiles, Advisories, Solution Candidates, Clusters, Neighborhoods — via published, versioned API contracts defined in TS-09. No other access path is permitted.

2. **Extensions may not write to core platform tables directly.** No extension service holds a write connection to core platform database tables.

3. **Extensions may not add columns to core platform tables to serve their own needs.** If an extension requires additional data attached to a core entity, that data lives in the extension's own data store, keyed by the core entity's stable identifier.

4. **Extensions live in separate repos, separate services, and separate deployment units.** They are not subfolders of the core platform monorepo. They deploy independently and fail independently.

5. **An extension is promoted to core only by a formal ADR.** The boundary does not dissolve informally. No feature flag, no "temporary" coupling, no shared database schema that will "be split later."

---

## What Crosses the Boundary

### Outbound: Core → Extension (read-only, via API)

The following core platform artifacts are exposed to extensions via read-only API contracts:

- **Technology Profiles** — full structured profile including metadata, evidence links, maturity assessments
- **Advisory content** — published Advisories and their evidence basis
- **Solution Candidate assessments** — evaluation outcomes, scoring, rationale
- **Cluster and Neighborhood data** — technology groupings, relationships, similarity signals

API contracts are versioned. Extensions pin to a contract version. Core platform deprecates contract versions on a defined schedule, not on demand.

### Inbound: Extension → Core (via ingestion pipeline, not direct write)

Extension outputs that are strategically relevant to the core platform do not write back directly. They enter the core as **Signals or Findings** through the standard ingestion and evidence graph pipeline.

Examples:

- A Red Team finding that contradicts a published Advisory is submitted as a Signal with source type `RED_TEAM_FINDING`. It enters the evidence graph, triggers a review flag on the Advisory, and is evaluated by the research pipeline like any other signal. It does not overwrite the Advisory directly.
- A Forecasting model that produces a materially different maturity trajectory than the current Technology Profile assessment submits that assessment as a Signal with source type `FORECAST_INPUT`. The research pipeline decides whether and how to incorporate it.
- A Wargaming scenario that surfaces a previously untracked technology actor submits that actor as a candidate Signal for Surface-stage evaluation.

This keeps the core evidence graph as the single source of truth. Extension outputs are evidence, not authority.

---

## Promotion Criteria

An extension can be proposed for promotion to core when all of the following are true:

1. **Usage threshold:** It has been used in at least [Internal threshold — to be defined] production advisory cycles with documented outcomes.
2. **Data model stability:** Its data model requirements are stable. The proposed promotion does not require changes to existing core platform tables — only additive schema changes are acceptable.
3. **Formal ADR:** A promotion ADR is written that documents the extension's data model, its integration points with the core pipeline, its operational requirements, and the rationale for promotion over continued boundary operation.
4. **Approval:** The promotion ADR is approved per the governance process defined below.
5. **Team capacity:** The extension team has capacity and commitment to maintain the promoted capability within the core platform governance model — including schema migration ownership, API contract versioning, and on-call coverage.

---

## Consequences

### Positive

- The core platform stays focused and shippable. The MVP is not blocked by Simulation tooling or Wargaming data models.
- Extensions can evolve independently. A forecasting model change does not require a core platform release.
- Core data model integrity is preserved. The Technology Profile schema reflects what the intelligence pipeline needs, not what every downstream consumer wants to hang off it.
- The evidence graph remains the single authoritative path for incorporating new intelligence. Red Team findings, forecasts, and simulation outputs are all held to the same evidentiary standard as any other signal.
- When extensions are eventually built, they have a stable, versioned API to build against — not a moving target.

### Negative

- Extensions cannot share UI components by directly embedding in the core platform UI. They can consume from a shared component library, but they render in their own surface. This creates some UX seam at the extension boundary (addressed in DS-10).
- Some capabilities that feel strategically core — particularly Red Teaming and Forecasting, which are tightly coupled to Advisory quality — must wait for promotion. This may create pressure to violate the boundary informally. That pressure must be resisted.
- The inbound Signal path (extension → core via ingestion pipeline) adds latency compared to a direct write. A Red Team finding does not immediately amend an Advisory; it enters a review queue. This is a feature, not a bug, but it will feel like friction to users who expect immediate reconciliation.

---

## Decision Needed

> **DECISION NEEDED —** Who has authority to approve an extension promotion ADR? *Recommendation: Platform architect + CTO sign-off required, given that promotions carry strategic and resourcing implications beyond a single team's scope. A promotion ADR that is approved by the platform architect alone is insufficient.*

> **DECISION NEEDED —** Should extensions share the core PostgreSQL database (in a separate schema) or run completely separate data stores? *Recommendation: Separate data stores. Extensions that need core data get it via the published API, never via direct database access — even read-only. Shared-database-separate-schema arrangements are routinely violated over time as engineers take shortcuts; physical separation enforces the boundary without relying on discipline alone. This recommendation applies even where it creates operational overhead.*

---

## References

- **TS-01** — Component Inventory (Phase 3 deferred extensions)
- **TS-09** — API Contract definitions (outbound extension API)
- **DS-10** — Extension Boundary UX (how extensions surface in the platform UI)
