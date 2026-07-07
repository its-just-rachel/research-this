# TS-11 — Deployment and Infrastructure

**Version:** 0.1
**Status:** Draft
**Last updated:** 2026-07-06
**Depends on:** TS-01 (System Architecture), TS-08 (Security), TS-10 (Observability)

---

## 1. Purpose

This spec defines the deployment topology, environment model, secrets management, reliability targets, backup and recovery procedures, CI/CD pipeline, and observability infrastructure for the Tech Range platform.

Deployment details are deferred from TS-01 and TS-08 to this document. This spec is the authoritative source for infrastructure decisions. It does not duplicate security policy (TS-08) or observability signal definitions (TS-10) — it defines the infrastructure those specs depend on.

The platform handles sensitive national security intelligence. Every infrastructure decision must be evaluated against that baseline: data in transit is encrypted, data at rest is encrypted, access is logged, and no component bypasses the access control model established in TS-08.

---

## 2. Environment Model

Three environments are defined. Each is logically isolated. No environment shares credentials, network segments, or data stores with another.

### 2.1 Development

- Purpose: local developer iteration and feature development.
- Composition: developer-local services + a shared development database. No real sensitive content permitted. Synthetic or fabricated test data only.
- Security controls: relaxed relative to production. SSO not required. Sensitive Content Store may be stubbed or replaced with a mock.
- Promotion gate: none — development is not a promotion gate. Code leaves development when a pull request is opened against the staging-validated branch.

### 2.2 Staging

- Purpose: integration testing, pre-release validation, and migration dry-runs.
- Composition: production-like topology. All services deployed and wired together. Uses sanitized or synthetic data only — no real sensitive content.
- Security controls: production-equivalent security controls enforced. SSO required. Sensitive Content Store deployed with real access controls, populated with synthetic data.
- Promotion gate: all automated test suites must pass in staging before production deploy is permitted. Schema migrations must be validated in staging before executing in production. No exceptions except emergency patches (see below).

### 2.3 Production

- Purpose: live platform serving analysts.
- Composition: full topology per Section 3. Real data including sensitive content. All security controls from TS-08 fully enforced.
- Promotion policy: code reaches production only by passing the staging promotion gate. Direct-to-production deploys are prohibited except for emergency security patches. Emergency patches require a post-hoc review within 24 hours, documented in the incident log.

### 2.4 Environment Promotion Summary

```
Development → (PR + review) → Staging → (automated gates pass) → Production
                                                                    ↑
                                            Emergency patch only ───┘
                                            (post-hoc review required within 24h)
```

---

## 3. Deployment Topology

### 3.1 Core Platform Services

All platform services are containerized using Docker. Container images are built from version-controlled Dockerfiles. No ad-hoc or manually constructed images in staging or production.

> **DECISION NEEDED —** Orchestration platform: Kubernetes is the recommended choice for operational consistency, replica management, rolling deploys, and ecosystem tooling. The specific Kubernetes distribution or managed service (e.g., self-hosted, government-managed, or commercial managed Kubernetes) must be determined based on the hosting model decision in Section 10. *Recommendation: Kubernetes. Specific platform: [Internal decision rule — pending hosting model decision in §10.]*

Service layout:

| Service | Deployment unit | Notes |
|---|---|---|
| API Gateway | Separate deployable service | Internet-facing; all external traffic enters here |
| WorkflowStateService | Separate deployable service | Internal only |
| Search/Alert Service | Separate deployable service | Internal only |
| CCE (Comparative Classification Engine) | Separate deployable service | Internal only; async worker |
| Ingestion Pipeline | Separate worker process | Pulls from ingest queue; not HTTP-facing |
| Enrichment Broker | Worker process co-located with Ingestion Pipeline | Calls external enrichment providers; [Internal decision rule] on co-location vs. separate container |
| Evidence Graph Service | Separate deployable service | Reads from and writes to Canonical Data Store; no external traffic |
| AI Assistance Layer | [Internal decision rule] | See note below |
| Analyst Workspace UI | Separate deployable service | Served via API Gateway |
| Radar View Service | Separate deployable service (read-only) | Reads from Canonical Data Store; no writes |
| Extension API | Separate deployable service behind API Gateway | GET-only access for extension services per ADR-011; separate rate limits from analyst traffic |

> **Data stores (Canonical Data Store, Sensitive Content Store)** are not listed here — see Section 3.2 for data store topology and isolation requirements.

**AI Assistance Layer placement:** co-location with CCE vs. independent service is pending ADR-009 (model hosting decision). Until ADR-009 is resolved, treat as a separate deployable service for planning purposes. ADR-009 must be resolved before staging environment is built.

Each service runs with the minimum OS and package footprint required. Base images are pinned to specific digests, not floating tags. Images are scanned for known vulnerabilities as part of the CI pipeline (see Section 7).

### 3.2 Data Stores

**Primary PostgreSQL instance:** hosts all core platform data — signals, canonical entities, workflow state, graph edges, audit log, user and permission records. This is the Canonical Data Store referenced in TS-01.

**Sensitive Content Store:** a physically separate PostgreSQL instance. Separate host or managed instance, separate credentials, separate backup schedule, separate access logs. No service that does not have an explicit need-to-know (per TS-08) may connect to this instance.

**Isolation requirements — no exceptions:**

- The primary DB and Sensitive Content Store must not share connection pools.
- They must not share credentials or credential stores.
- They must not share backup destinations.
- They must not be reachable from the same network segment unless an explicit firewall rule permits traffic from a named service with documented justification.

### 3.3 Network Topology

```
Internet
    │
    ▼
┌──────────────┐
│  API Gateway │  ← only internet-facing component
└──────┬───────┘
       │ private network
       ▼
┌──────────────────────────────────────────────────────────┐
│  Internal service mesh                                   │
│                                                          │
│  WorkflowStateService   Search/Alert   CCE   Ingestion   │
│  AI Assistance Layer    Analyst UI                       │
│                                                          │
│           │                          │                   │
│           ▼                          ▼                   │
│    Primary PostgreSQL        [restricted segment]        │
│                               Sensitive Content Store    │
└──────────────────────────────────────────────────────────┘
```

Rules:

- The API Gateway is the only component with an internet-facing network interface.
- All internal services communicate on a private network not reachable from the internet.
- The Sensitive Content Store sits on a more restricted network segment. Only services with documented need-to-know (WorkflowStateService for retrieval, Ingestion Pipeline for write-time storage, AI Assistance Layer if approved in ADR-009) may have firewall rules permitting access.
- Direct database access from the internet is prohibited under all circumstances. There is no SSH tunnel, no bastion host with DB access, and no jump host arrangement that bypasses this rule in production.
- Network policy is enforced at the infrastructure level (firewall rules or Kubernetes NetworkPolicy), not only at the application level.

---

## 4. Secrets Management

No secrets — database credentials, API keys, tokens, or encryption keys — may appear in source code, committed configuration files, or environment variables checked into version control.

**Secrets store:**

> **DECISION NEEDED —** Secrets management platform. Options include HashiCorp Vault (self-hosted or HCP), a government cloud-native secrets service, or equivalent. Must support dynamic credential generation, audit logging of secret access, and rotation. *Recommendation: HashiCorp Vault or cloud-native equivalent with audit logging. [Internal decision rule — pending hosting model decision in §10.]*

**Service account credentials:**

Each service has its own database credential scoped to the minimum required permissions for that service, per TS-08 §8. No service uses a shared credential. No service uses a credential with broader permissions than required.

| Service | Primary DB access | Sensitive Content Store access |
|---|---|---|
| API Gateway | None (routes only) | None |
| WorkflowStateService | Read/write own tables | Read (if authorized) |
| CCE | Read signals, write classifications | None |
| Ingestion Pipeline | Write signals | Write (if signal is sensitive) |
| Search/Alert Service | Read | None |
| AI Assistance Layer | Read (if applicable) | [Pending ADR-009] |

**Secret rotation:**

> **DECISION NEEDED —** DB credential rotation schedule. *Recommendation: rotate on a [Internal decision rule] schedule (e.g., 30 or 90 days); use dynamic secrets via Vault where the platform supports it to minimize rotation overhead.*

> **DECISION NEEDED —** API key expiry policy. *Recommendation: API keys expire after [Internal decision rule] (e.g., 90 days) unless explicitly renewed; short-lived tokens preferred for service-to-service calls.*

**Audit:** all secret access (read, write, rotation) is logged with timestamp, service identity, and secret name. Logs are forwarded to the central log store defined in Section 8.

---

## 5. Reliability Targets

Targets apply to the production environment only.

| Component | Uptime target | Notes |
|---|---|---|
| API Gateway | 99.9% (~8.7 hours downtime/year) | Hard target; analyst access depends on this |
| Primary PostgreSQL | 99.9% | Platform non-functional without it |
| Sensitive Content Store | 99.9% | Analysts cannot retrieve sensitive content if unavailable |
| WorkflowStateService | 99.9% | |
| Search/Alert Service | 99.5% | Degraded gracefully if unavailable |
| Ingestion Pipeline | Best-effort | Must have dead letter queue; downtime delays ingestion but does not lose signals |
| CCE | Best-effort async | Downtime delays classification; does not lose data; signals queue and reprocess on recovery |
| AI Assistance Layer | Best-effort | Analyst workflows degrade gracefully if unavailable |

**RPO (Recovery Point Objective):** less than 1 hour. In the event of catastrophic failure, no more than 1 hour of committed data may be lost.

**RTO (Recovery Time Objective):** less than 4 hours. Full service must be restorable within 4 hours of a catastrophic failure.

These targets must be validated annually via a documented DR exercise (see Section 6.5).

---

## 6. Backup and Recovery

### 6.1 Primary PostgreSQL Backup

- Daily full backup.
- Continuous WAL (Write-Ahead Log) archiving to enable point-in-time recovery.
- Retention: 90 days minimum.
- Backup storage: [Internal decision rule — encrypted, access-controlled storage separate from the primary instance].

### 6.2 Sensitive Content Store Backup

- Daily full backup.
- Separate storage destination from the primary DB backup. No shared bucket, volume, or storage account.
- Separate access controls: backup storage for the Sensitive Content Store is accessible only to the backup service account and the designated recovery role. Not accessible to the same accounts that access primary DB backups.
- Retention: 90 days minimum.

### 6.3 Backup Integrity and Encryption

- All backups encrypted at rest using a key managed separately from the data being backed up.
- Backup encryption keys stored in the secrets manager (Section 4), not co-located with the backup files.

### 6.4 Restore Testing

- Monthly restore drill to the staging environment.
- Drill covers: restore from daily backup, point-in-time recovery to a specific timestamp, and verification that restored data is consistent (row counts, schema version, sample query checks).
- Drill results documented. Failures trigger an incident and a remediation ticket before the next production deploy.

### 6.5 DR Exercise

- Annual full DR exercise simulating catastrophic failure of the production environment.
- Exercise validates RTO and RPO targets from Section 5.
- Results documented. If targets are not met, a remediation plan is required before the next exercise.

---

## 7. CI/CD Pipeline

### 7.1 Required Checks Before Merge

All pull requests must pass the following before merge to the main branch:

- Unit tests (full suite).
- Integration tests against a real test database (not mocked).
- Schema migration validation: if the PR includes a migration file, the migration must apply cleanly to the current staging schema and must be reversible or have a documented forward-only justification.
- Container image vulnerability scan: no known critical CVEs in the image being built.
- Secrets scan: no credentials or tokens detected in committed files.

### 7.2 Schema Migration Policy

> **DECISION NEEDED —** Migration tool. Options include Flyway, Alembic, Liquibase, or equivalent. Must support versioned migrations, checksums, and rollback tracking. *Recommendation: [Internal decision rule — select based on primary language stack; Alembic if Python-primary, Flyway if JVM or language-agnostic.]* 

No manual schema changes in production. All schema changes are applied via the automated migration tool as part of the deploy pipeline. Hotfix schema changes follow the same process — there is no bypass.

Migrations that require table locks (e.g., adding a non-nullable column without a default, rebuilding an index non-concurrently) must:

1. Be identified in the PR review as lock-requiring.
2. Be scheduled during a defined low-traffic maintenance window.
3. Trigger an analyst notification at least 24 hours in advance stating expected downtime or degradation.

### 7.3 Deploy Strategy

Blue/green or rolling deploy strategy for all production releases. The strategy must ensure:

- No analyst-visible downtime for routine releases.
- The previous version remains available for rapid rollback (within 15 minutes) if the new version fails health checks.
- Schema migrations are applied before new code is deployed (expand-contract pattern for breaking schema changes).

### 7.4 Promotion Gate (Staging → Production)

Automated gate. Production deploy is blocked unless:

- All CI checks pass on the commit being promoted.
- Integration test suite passes against the staging environment.
- Schema migration (if any) has executed cleanly in staging.
- No open severity-1 incidents in production.

Manual override of the promotion gate requires documented approval and triggers a post-hoc review.

---

## 8. Observability Infrastructure

Observability signal definitions (what each service emits) are specified in TS-10. This section defines the infrastructure that collects, stores, and routes those signals.

### 8.1 Structured Logging

All services emit logs in JSON format to stdout. Each log line includes at minimum:

```json
{
  "timestamp": "<ISO 8601>",
  "level": "<debug|info|warn|error>",
  "service_name": "<service>",
  "request_id": "<uuid>",
  "message": "<string>"
}
```

Additional fields (user_id, entity_id, operation, duration_ms) are added where applicable per TS-10.

A log aggregation agent (e.g., Fluentd, Fluent Bit, or equivalent — [Internal decision rule]) collects logs from all containers and forwards to a central log store.

**Log retention:**

> **DECISION NEEDED —** Log retention tiers and storage platform. *Recommendation: minimum 90 days hot (queryable), 2 years archived (restorable on request). [Internal decision rule — storage platform pending hosting decision in §10.]*

### 8.2 Metrics

Each service exposes a Prometheus-compatible `/metrics` endpoint. A metrics scraper collects from all services on a defined interval. Metrics are stored in a time-series database compatible with Prometheus query syntax.

TS-10 defines the specific metrics each service must emit. Infrastructure-level metrics (CPU, memory, disk, network) are collected from the container orchestration layer.

### 8.3 Alerting

Two alert routing paths, per TS-10 §10:

- **Operational alerts** (service down, error rate spike, DB unreachable, latency threshold exceeded): route to on-call engineer. Paging via [Internal decision rule — PagerDuty or equivalent].
- **Intelligence quality alerts** (CCE throughput degraded, ingestion backlog exceeding threshold, classification error rate elevated): route to senior analyst on duty.

Alert definitions live in version-controlled configuration. Changes to alert thresholds go through code review.

### 8.4 Distributed Tracing

Every request entering the API Gateway is assigned a `request_id` (UUID). This ID is:

- Returned in the API response headers.
- Propagated in all internal service calls triggered by the request (as an HTTP header or message queue attribute).
- Included in all log lines emitted while processing that request.
- Included in all audit log entries triggered by that request (per TS-08).

This provides end-to-end traceability: given a `request_id`, every log line, audit entry, and metric event associated with that request can be correlated.

Span-level distributed tracing (OpenTelemetry or equivalent) is a future enhancement. The `request_id` propagation defined here is the baseline requirement for v0.1.

---

## 9. Operational Runbooks (Stubs)

The following runbooks must exist and be validated before go-live. Full content is deferred to operational documentation. Each runbook must be tested in staging before it is considered valid.

| Runbook | Trigger | Owner |
|---|---|---|
| **Service down** | API Gateway, WorkflowStateService, or Search/Alert Service is unreachable | On-call engineer |
| **DB unreachable** | Primary PostgreSQL health check fails | On-call engineer |
| **Sensitive Content Store unreachable** | Sensitive Content Store health check fails | On-call engineer + designated security contact |
| **Restore from backup** | Data loss or corruption event; step-by-step restore from daily backup | On-call engineer |
| **Point-in-time recovery** | Targeted data loss; restore to a specific timestamp using WAL archive | On-call engineer |
| **Secret rotation — DB credentials** | Scheduled rotation or credential compromise; rotate without service downtime | On-call engineer |
| **Schema migration with table lock** | Deploying a migration that requires a lock; pre-notification, scheduling, execution | Deploy engineer |
| **Dead letter queue processing** | Ingestion Pipeline DLQ backlog exceeds threshold; triage, replay, or discard | On-call engineer + senior analyst |
| **Scale-up: CCE or Ingestion Pipeline** | Burst load causing processing backlog; add capacity | On-call engineer |
| **Emergency production patch** | Critical security fix bypassing normal promotion gate; deploy + post-hoc review | Engineering lead + on-call engineer |

---

## 10. Decisions Required

The following decisions must be made before infrastructure provisioning begins. Each is flagged as [Internal decision rule] throughout this document because they involve sensitive strategic, compliance, or vendor information not appropriate for a public-safe spec.

> **DECISION NEEDED —** Cloud provider and hosting model. Options include on-premises, government cloud (e.g., FedRAMP-authorized, IL4/IL5/IL6 compliant), commercial cloud, or hybrid. This decision drives nearly every other infrastructure choice in this spec and must be resolved first. It is a compliance and strategic decision, not only a technical one. *Recommendation: evaluate against data classification requirements, compliance obligations (e.g., CMMC, FedRAMP, IL level), and operational support capacity before selecting. [Internal decision rule.]*

> **DECISION NEEDED —** Container orchestration platform. Kubernetes is the recommended orchestration layer for operational consistency, rolling deploy support, network policy enforcement, and ecosystem tooling. The specific distribution (self-hosted upstream Kubernetes, a government-managed distribution, or a commercial managed Kubernetes service) depends on the hosting model decision above. *Recommendation: Kubernetes. Specific platform: [Internal decision rule — pending hosting model decision.]*

> **DECISION NEEDED —** Disaster recovery site. Does the platform have a secondary deployment environment for DR purposes? If yes, where is it located, what is its relationship to the primary site (active-passive, active-active, warm standby), and what is the failover procedure? This is a strategic and compliance decision with cost, data residency, and classification implications. *Recommendation: define a warm standby DR site capable of meeting the RTO/RPO targets in §5. Location and architecture: [Internal decision rule — pending hosting model decision.]*
