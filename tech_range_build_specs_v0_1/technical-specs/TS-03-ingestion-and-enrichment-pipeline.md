# TS-03 — Ingestion and Enrichment Pipeline

**Status:** Draft  
**Version:** 0.1  
**Last updated:** 2026-07-06  
**Related:** ADR-009 (AI Tier Model), ADR-014 (Source Credibility Scoring), ADR-015 (Temporal Provenance), TS-02 (Sources), TS-04 (CCE), TS-07 (Burst Detection), TS-08 (Sensitive Content Store)

---

## 1. Purpose

This spec defines how raw information from external sources is transformed into structured Signals within the Tech Range platform. The ingestion and enrichment pipeline is the first point of contact between the outside world and the platform's internal data model. Its responsibilities are:

- Fetching, parsing, and extracting structured intelligence from raw content
- Enriching extracted signals with third-party and internal context
- Handing off to the Classification and Corroboration Engine (CCE) for initial triage
- Storing all artifacts with correct temporal provenance per ADR-015
- Routing sensitive content to the Sensitive Content Store (TS-08) without persisting it in the main database
- Notifying analysts of signals that match active watchlists or alert rules

The pipeline must be **async and non-blocking**: ingestion activity must never degrade the analyst workspace experience.

---

## 2. Pipeline Overview

The pipeline moves a piece of raw content through seven discrete stages before a Signal is fully ingested and available to analysts.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        INGESTION AND ENRICHMENT PIPELINE                     │
└──────────────────────────────────────────────────────────────────────────────┘

  Source ──► Fetch ──► Parse ──► Extract ──► Enrich ──► Classify ──► Store ──► Notify
              │                   (AI)                   (CCE)
              │
         [raw_content or
          sensitive_ref_id]

SYNC / ASYNC BOUNDARY:
┌───────────────────────────────────────────┬──────────────────────────────────┐
│  SYNCHRONOUS (request-scoped)             │  ASYNCHRONOUS (background)       │
│                                           │                                  │
│  Fetch → Parse → Extract → Store (stub)   │  Enrich → Classify → Store       │
│                                           │  (full) → Notify                 │
└───────────────────────────────────────────┴──────────────────────────────────┘
```

**Sync stages** create a stub Signal record (with `signal_state = 'new'`) as quickly as possible so the analyst can see that a signal has entered the system. **Async stages** complete enrichment, CCE classification, and notification in the background without blocking the analyst workspace.

For **analyst-submitted** and **intentional injection** modes, the sync stages complete within the request lifecycle and return the stub Signal ID to the analyst immediately.

---

## 3. Three Ingestion Modes

### 3.1 Automated Pull

The system periodically polls a configured list of monitored sources and ingests new content discovered since the last fetch.

**Scheduling**

- Each source has a configurable `poll_interval_minutes` (default: 60 minutes). High-priority sources may be set to shorter intervals.
- A scheduler service manages the poll queue and dispatches fetch jobs to pipeline workers.
- Each fetch job records `last_fetched_at` on the Source record upon completion (success or failure).

**Source list management**

- Any analyst may propose a new source; a senior analyst (or admin) must approve it before it enters the active poll list.
- Sources can be paused (temporarily removed from polling) or archived (permanently removed and historical signals retained).
- See Section 7 for the full Sources schema reference.

**Deduplication**

- On each poll cycle, the pipeline computes a `content_hash` (SHA-256 of normalized content) for each fetched item.
- If a `content_hash` already exists in the database, the item is skipped immediately — no Signal is created, no processing resources are consumed.
- Near-duplicate detection (same source, same event time window, high semantic similarity) runs as a background job after ingestion. See Section 5.

**Rate limiting and backoff**

- Outbound requests to any single source domain are subject to a per-domain rate limit (configurable; default: 1 request / 10 seconds).
- On HTTP 429 or 503, the pipeline applies exponential backoff with jitter (base: 30s, max: 4 hours).
- After 5 consecutive fetch failures on a single source, the source is flagged for health review and an alert is raised. See Section 7.

---

### 3.2 Analyst-Submitted

An analyst submits a URL, document upload (PDF, DOCX, plain text), or a pasted text snippet directly from the analyst workspace.

**Trigger**

- The analyst workspace exposes a "Submit Signal" interface that accepts: a URL, a file upload, or a text paste.
- On submission, an ingestion job is enqueued with `ingestion_mode = 'analyst_submitted'` and `created_by = <analyst_id>`.

**Pipeline stages**

- The same fetch → parse → extract → enrich → classify → store → notify stages run as for automated pull.
- For direct text pastes, the Fetch stage is skipped; the submitted text is treated as the raw content.
- The analyst sees a stub Signal record immediately (sync stages); enrichment and classification complete in the background.

**Attribution**

- `created_by` on the Signal record is set to the submitting analyst's ID.
- This is distinct from `source` (the publication or outlet the content came from). Both fields are populated independently.
- Analyst-submitted signals are visually distinguished in the UI with a "Submitted by [analyst]" badge.

---

### 3.3 Intentional Injection

An analyst creates a Signal directly in the platform for a technology that the platform should track, even though no organic signal has yet arrived from an external source. This mode is used to seed the evidence graph with known entities before they begin generating inbound signals.

**Use cases**

- A technology the analyst knows is emerging but has not yet been publicly written about.
- A known acquisition or partnership announced verbally that has no public source URL yet.
- A deliberate placeholder to anchor a Neighborhood or relationship graph ahead of expected signals.

**Required fields at creation**

| Field | Description |
|---|---|
| `signal_type` | Must be `intentional_injection` |
| `technology_name` | Name of the technology being tracked |
| `description` | Analyst-supplied description |
| `analyst_rationale` | Why this signal is being injected; what the analyst expects it to anchor |
| `created_by` | Analyst ID |
| `event_time` | Analyst's best estimate of world-time relevance (or current time if unknown) |

**Pipeline behavior**

- Fetch and Parse stages are skipped (no external content to retrieve).
- Extract stage runs on the analyst-supplied description using the same AI extraction pipeline (Tier 1, fully autonomous per ADR-009).
- Enrich, Classify, Store, and Notify stages run normally.
- `ingestion_mode = 'intentional_injection'` is recorded on the Signal.

**UI and graph treatment**

- Intentional injections are flagged with a distinct visual indicator in the analyst workspace (e.g., an "Injected" badge).
- In the evidence graph, intentional injection nodes are rendered with a different visual style than organically sourced signals to make provenance visible at a glance.
- When an organic signal later arrives that corroborates an intentional injection, the CCE links them; the injected signal's status updates to reflect corroboration.

---

## 4. Pipeline Stages

### 4.1 Fetch

The Fetch stage retrieves raw content from an external URL or accepts content directly (for analyst-submitted text or intentional injections).

**HTTP fetch**

- Requests are made with a descriptive `User-Agent` header identifying the Tech Range crawler.
- Timeout: 30 seconds for initial connection, 60 seconds total.
- Retry logic: up to 3 retries on transient errors (5xx, network timeout) with exponential backoff. 4xx errors are not retried (except 429, which triggers rate-limit backoff).
- Redirects: followed up to 5 hops; circular redirects are treated as fetch failure.

**Content type handling**

| Content type | Handling |
|---|---|
| HTML | Parsed to extract main body text, stripping navigation, ads, and boilerplate |
| PDF | Text extracted via PDF parser; if PDF is image-only (scanned), OCR is applied |
| Plain text | Accepted as-is |
| DOCX / other office formats | Converted to plain text |
| Unsupported types | Fetch failure; logged with `error_code = 'unsupported_content_type'` |

**Storage**

- `raw_content_ref`: a reference key pointing to the full extracted text in external storage (S3 or equivalent). Raw content is never stored inline in the main DB; the DB holds only this reference key.
- If the content is flagged as sensitive (see Section 6): `raw_content_ref` is **not** stored. Instead, a `sensitive_ref_id` pointer is written to the Signal record; the content goes to the Sensitive Content Store (TS-08).
- `content_hash` (SHA-256 of normalized raw content): stored for deduplication.

**Failure handling**

- Fetch failures are logged with structured fields: `stage = 'fetch'`, `source_id`, `url`, `error_code`, `http_status`, `duration_ms`.
- No Signal record is created on fetch failure.
- Failed jobs are placed on a retry queue. After 3 retries, the job moves to the dead letter queue and an alert is emitted.

---

### 4.2 Parse

The Parse stage extracts structured metadata from the raw fetched content.

**Extracted fields**

| Field | Description |
|---|---|
| `title` | Document or article title |
| `author` | Author name(s) if present |
| `source_url` | Canonical URL of the content |
| `captured_at` | Source publication date if available (see note below) |
| `content_type` | `html`, `pdf`, `text`, `docx`, etc. |
| `language` | Detected language code (ISO 639-1) |
| `encoding` | Character encoding |

**`captured_at` determination**

- `captured_at` is set to the **source publication date** if reliably extractable from the document (HTML `<meta>` tags, structured data, byline dates, PDF metadata).
- If no publication date is available, `captured_at` defaults to `ingested_at` (the timestamp when the pipeline received the content).
- `captured_at` is never inferred or estimated by AI; it is always either extracted from document metadata or set to `ingested_at`.

**Relationship to ADR-015 timestamps**

All five ADR-015 temporal fields are set as follows at the end of the Parse stage:

| Field | Value |
|---|---|
| `event_time` | Set in the Extract stage (AI-inferred world-time of the described event) |
| `captured_at` | Source publication date, or `ingested_at` if unavailable |
| `ingested_at` | Timestamp when the pipeline received the content |
| `created_at` | DB record creation timestamp (set at Store stage) |
| `valid_from` | Set at Store stage; defaults to `ingested_at` |

---

### 4.3 Extract (AI Tier 1)

The Extract stage uses AI (operating at Tier 1 — fully autonomous, per ADR-009) to derive structured intelligence from the parsed content.

**Named entity extraction**

The AI model extracts the following entity types from the content:

- **Technologies**: named technologies, models, systems, frameworks, protocols
- **Organizations**: companies, research labs, government bodies, standards bodies
- **Topics**: high-level thematic categories (e.g., "large language models", "semiconductor supply chain")
- **Locations**: countries, regions, cities relevant to the signal's subject matter
- **People**: named individuals where relevant to the signal (e.g., named researchers, executives)

Each extracted entity is stored with a confidence score.

**Signal type classification**

The AI assigns one of the following `signal_type` values:

> **Note:** The canonical `signal_type` vocabulary is defined in TS-02. This list is reproduced here for reference — TS-02 is the source of truth.

| Value | Description |
|---|---|
| `paper` | Academic or research publication |
| `model_release` | Release of an AI model or major model update |
| `product_launch` | Commercial product or service launch |
| `code_commit` | Notable code commit or repository activity |
| `benchmark` | Benchmark result or evaluation |
| `funding` | Funding round, grant, or investment |
| `acquisition` | Corporate acquisition or merger |
| `partnership` | Formal partnership or joint venture |
| `regulation` | Regulatory filing, rule, law, or government action |
| `threat_report` | Cybersecurity threat intelligence or vulnerability disclosure |
| `patent` | Patent filing or grant |
| `job_posting` | Job posting indicating strategic hiring direction |
| `conference_talk` | Conference presentation or talk |
| `vendor_blog` | Vendor-authored blog post or announcement |
| `news` | General news article |
| `star_spike` | Spike in repository stars or similar popularity signal |
| `standards_update` | Update to a technical standard or specification |
| `intentional_injection` | Analyst-created signal (no external source) |
| `other` | Does not fit a defined type |

**Summary generation**

The AI generates a 2–3 sentence plain-language summary of the signal's content. The summary is intended for analyst triage, not as authoritative interpretation.

**`event_time` inference**

The AI infers the real-world time of the described event (which may differ from `captured_at` — for example, a paper published today describing research conducted last year). `event_time` is recorded as the AI's best estimate; analysts can correct it.

**Model version recording**

- The `model_version` field on the Signal record captures the AI model identifier used for extraction.
- This supports auditability and allows re-extraction if a model is retrained or replaced.

**Provisional status of all extraction outputs**

All fields produced by the Extract stage are marked as `ai_generated = true` and are **provisional**. Analysts may correct any field (entity, signal type, summary, event_time) at any time. Corrections are tracked in the Signal's edit history.

---

### 4.4 Enrich

The Enrich stage augments the Signal with context from third-party providers and internal data sources.

**Enrichment types**

| Enrichment type | Description |
|---|---|
| Company context | For organizations extracted in the Signal: funding history, employee count, sector, notable investors |
| Technology context | For technologies extracted: related technologies, standards bodies, known use cases |
| Threat intelligence | For signals of type `threat_report`: CVE references, MITRE ATT&CK mappings, known actor attribution |
| Patent data | Patent filings related to extracted technologies (Phase 2) |

> **DECISION NEEDED —** Which enrichment providers are in scope for MVP (company context, tech context, threat intel, patent data)? *Recommendation: start with one company context provider and one tech context provider for MVP; add threat intel and patent data providers in Phase 2. Specific provider names are an [Internal decision rule] and are not recorded in this spec.*

**Provider trust boundaries**

- Each enrichment result is tagged with its `provider_id`, `provider_version`, and a `confidence` score.
- Enrichment data is not treated as authoritative; it is advisory context for analysts.
- Conflicting enrichment from multiple providers is surfaced to analysts without auto-resolution.

**Enrichment data model**

- Enrichment results are stored as separate `Enrichment` records, not merged into the Signal record.
- Each `Enrichment` record is linked to the Signal via an `object_relationships` entry with relationship type `enriched_by`.
- This separation ensures that enrichment data can be updated independently as provider data changes, and that the Signal record remains a clean representation of the source content.

**Sensitive enrichment**

- If an enrichment result contains sensitive content (see Section 6), the sensitive content pointer pattern is applied: the sensitive portion is routed to the Sensitive Content Store (TS-08), and a `sensitive_ref_id` is stored on the `Enrichment` record in place of the content.

**Enrichment failure handling**

- Enrichment failures are non-blocking: if an enrichment provider is unavailable, the Signal proceeds through the pipeline without that enrichment. The missing enrichment is queued for retry.
- After N retries (configurable; default: 3), the enrichment attempt is abandoned and logged. The Signal is not blocked.

---

### 4.5 Classify (CCE Handoff)

After ingestion and enrichment, the Signal is handed off to the Classification and Corroboration Engine (CCE, TS-04) for initial triage and classification.

**Handoff**

- The pipeline publishes a `signal.ingested` event to the internal event bus with the Signal ID.
- The CCE consumes this event asynchronously.
- The Signal is stored with `signal_state = 'new'` before CCE processing begins.

**CCE processing**

The CCE (detailed in TS-04) performs:

- Technology neighborhood classification: which Neighborhoods does this signal belong to?
- Match set creation: which existing signals or entities does this signal relate to?
- Corroboration candidate identification: does this signal corroborate a previously ingested signal from a different source?

**State transitions**

| State | Condition |
|---|---|
| `new` | Signal has been stored; CCE has not yet completed |
| `triaged` | CCE has completed classification; match sets written |

- After CCE completes, the Signal transitions from `new` to `triaged`.
- If the CCE produces a boundary case (low-confidence classification, ambiguous neighborhood assignment, or conflicting match sets), the Signal remains in `signal_state = 'triaged'` and a separate entry is created in the `boundary_cases` table for analyst attention. There is no `review_required` state in the canonical state machine.

**Async guarantee**

- CCE classification **never blocks** the analyst workspace. Analysts may interact with a `new`-state Signal before classification completes.
- The UI clearly indicates `signal_state` so analysts understand when classification is still pending.

---

### 4.6 Store

The Store stage writes the final Signal record and all associated artifacts to the database.

**Signal record**

All five ADR-015 temporal fields are written at this stage:

- `event_time`: AI-inferred world-time of the described event
- `captured_at`: source publication date, or `ingested_at` if unavailable
- `ingested_at`: when the pipeline received the content
- `created_at`: DB record creation timestamp (this stage)
- `valid_from`: defaults to `ingested_at` unless overridden

Additional fields written at Store:

- `signal_state = 'new'`
- `content_hash`: for deduplication
- `raw_content_ref` or `sensitive_ref_id` (not both); `raw_content_ref` is a reference key to external storage (S3 or equivalent) — raw content is never stored inline in the main DB
- `ai_generated = true` on all AI-produced fields
- `model_version`: AI model identifier used in Extract
- `ingestion_mode`: `automated_pull`, `analyst_submitted`, or `intentional_injection`
- `created_by`: analyst ID (for analyst-submitted and intentional injection modes)

**Source record**

- If a Source record already exists for the content's origin, it is updated (`last_fetched_at`, fetch statistics).
- If no Source record exists (first-time ingestion from this source), a new Source record is created. The source enters the system in a pending credibility state; credibility scoring is deferred to the ADR-014 system.

**Enrichment records**

- Each `Enrichment` record is written with its `provider_id`, `provider_version`, `confidence`, and content (or `sensitive_ref_id` if sensitive).
- `object_relationships` entries are created linking each Enrichment to the Signal with relationship type `enriched_by`.

---

### 4.7 Notify

The Notify stage determines whether the newly stored Signal should generate alerts for analysts.

**Watchlist and alert rule matching**

- The platform maintains analyst-configured watchlists (tracked technologies, organizations, topics) and alert rules (custom signal type or entity filters).
- After a Signal is stored, the notification engine evaluates all active watchlists and alert rules against the Signal's extracted entities and signal type.
- Matching rules trigger notifications delivered to the relevant analysts via their configured notification channels.

**Burst alert surfacing**

- If the Signal's primary Neighborhood has exceeded the burst detection threshold (`burst_score` above configured limit), the Signal is surfaced to the burst alert queue.
- Burst detection logic determines whether a cluster of signals constitutes a genuine emerging trend or incidental volume. Full specification is in TS-07.
- Burst alerts are surfaced in the analyst workspace as a distinct notification type.

**Notification delivery**

- Notifications are created as records in the `notifications` table, associated with both the Signal and the relevant analyst(s).
- Delivery channels (in-app, email, webhook) are configured per analyst.
- Notification delivery failures do not affect Signal ingestion; they are retried independently.

---

## 5. Deduplication

The pipeline uses a three-level deduplication strategy to handle exact duplicates, near-duplicates, and cross-source corroboration distinctly.

### Level 1: Exact duplicate (content hash)

- On Fetch, a `content_hash` (SHA-256 of normalized content) is computed.
- If an existing Signal already has this `content_hash`, the incoming content is discarded immediately. No Signal is created. The skip event is logged.
- Normalization before hashing: whitespace canonicalization, URL normalization, removal of dynamic timestamps or ad identifiers that vary across fetches of the same content.

### Level 2: Near-duplicate detection

Near-duplicate detection identifies signals that represent the same real-world event reported slightly differently by the same source (e.g., two versions of the same article, or a press release re-published with minor edits).

**Detection criteria (all three must be met):**

1. Same source (`source_id` matches)
2. `event_time` within a configurable window (default: ±24 hours)
3. Semantic similarity score > 0.85 (computed via embedding similarity)

**Handling:**

- Near-duplicates are **not** auto-merged. Auto-merge risks silently collapsing signals that an analyst may have already annotated.
- Instead, a merge review task is queued for analyst attention. The analyst can confirm the merge, reject it, or mark the signals as related-but-distinct.

> **DECISION NEEDED —** Should near-duplicate detection (semantic similarity computation) run synchronously within the pipeline or as a background job after ingestion? *Recommendation: run as a background job; do not block ingestion on similarity computation, as embedding calls add latency and the cost of a brief window where a near-duplicate is visible as a separate signal is low.*

### Level 3: Cross-source corroboration

When the same real-world event is reported by multiple independent sources, the platform treats this as **corroboration**, not duplication.

- Cross-source signals with matching `event_time` windows and high semantic similarity (same threshold: 0.85) are **not** merged or deduplicated.
- Instead, the CCE identifies them as corroboration candidates and creates a Corroboration Group (or adds to an existing one).
- Corroboration is a first-class signal quality indicator: an event reported by five independent sources has higher evidential weight than one reported by a single source.
- Corroboration Group management is specified in TS-04 (CCE).

---

## 6. Sensitive Content Handling

The platform applies a sensitive content pointer pattern to ensure that sensitive material never resides in the main database, while preserving the ability to reference it in Signal records and the evidence graph.

### Detection

Sensitivity is assessed at two points:

1. **During Fetch / Parse**: the content scanner checks for known sensitivity markers, including:
   - Classification markings (e.g., document headers indicating restricted distribution)
   - Restricted program names or codenames on the platform's internal sensitive-term list
   - Partner-confidential flags embedded in document metadata or content

2. **During Enrich**: if an enrichment result from a third-party provider contains sensitive content (e.g., a threat intel lookup returns restricted attribution data), the same detection logic applies to the enrichment result.

### Pointer creation

- When sensitive content is detected, the content is **routed directly to the Sensitive Content Store (TS-08)** and is not written to the main database.
- A `sensitive_ref_id` (an opaque pointer) is written to the Signal record (or Enrichment record) in place of the content.
- The `raw_content_ref` field on the Signal is left null; the `sensitive_ref_id` field is populated.

### Access logging

- Every access to a sensitive reference (any lookup using a `sensitive_ref_id`) must be logged with: accessor identity, access timestamp, Signal or Enrichment ID, and the reason for access if provided.
- Access logging is enforced at the Sensitive Content Store layer (TS-08); the ingestion pipeline does not need to implement it independently.

### Content retention

> **DECISION NEEDED —** What is the content retention policy for content referenced by `raw_content_ref` stored at ingestion? *Recommendation: retain raw content in external storage for 90 days to allow re-processing (e.g., if the extraction model is updated); after 90 days, retain only the extracted structured data and the reference key may be invalidated. Sensitive content retention policy is governed by TS-08 independently.*

---

## 7. Source Management

Sources represent the publications, outlets, feeds, or data providers from which the platform ingests content. Source management is shared between the ingestion pipeline and the Sources system (TS-02).

### Sources table (reference)

Key fields relevant to the ingestion pipeline:

| Field | Description |
|---|---|
| `source_id` | Internal identifier |
| `url` | Base URL or feed URL |
| `name` | Human-readable source name |
| `source_type` | `rss`, `web`, `api`, `upload`, `manual` |
| `credibility_score` | Managed by ADR-014 system; read-only from pipeline perspective |
| `coverage_areas` | Topic or technology areas this source covers |
| `last_fetched_at` | Timestamp of last successful fetch |
| `fetch_failure_count` | Running count of consecutive fetch failures |
| `poll_interval_minutes` | Polling frequency in minutes |
| `status` | `active`, `paused`, `archived` |

### Adding a new source

1. Any analyst may propose a new source via the source management interface.
2. The proposal includes: URL, name, type, and coverage areas.
3. A senior analyst (or admin) reviews and approves or rejects the proposal.
4. Approved sources enter the active poll list on the next scheduler cycle.

### Source credibility scoring

- Credibility scoring is managed by the ADR-014 system, which is out of scope for this spec.
- The ingestion pipeline's responsibility is limited to: recording accurate source metadata and fetch statistics, and making that data available to the ADR-014 system.
- The pipeline does not compute or modify `credibility_score` directly.

### Source health monitoring

- The pipeline tracks `fetch_failure_count` (consecutive fetch failures) per source.
- After 5 consecutive failures, the source is flagged with `status = 'health_alert'` and an alert is raised to the platform operations team.
- If a source that was previously healthy produces zero new signals for a configurable period (default: 7 days), a "source gone dark" alert is raised. This may indicate the source URL has changed, content has moved behind a paywall, or the source has ceased publication.

---

## 8. Error Handling and Observability

### Structured logging

Every pipeline stage emits structured log entries with the following fields:

| Field | Description |
|---|---|
| `stage` | Pipeline stage name: `fetch`, `parse`, `extract`, `enrich`, `classify`, `store`, `notify` |
| `signal_id` | Signal ID if the Signal record has been created; null for pre-store failures |
| `source_id` | Source identifier |
| `ingestion_mode` | `automated_pull`, `analyst_submitted`, `intentional_injection` |
| `duration_ms` | Time spent in this stage |
| `status` | `success`, `failure`, `skipped` |
| `error_code` | Structured error code on failure (see error code registry) |
| `error_detail` | Human-readable error description |
| `retry_count` | Number of retries attempted |

### Dead letter queue

- Signals that fail enrichment or classification after N retries (configurable; default: 3 per stage) are moved to the dead letter queue (DLQ).
- DLQ entries include the full Signal record at the time of failure, the failing stage, the error code, and the retry history.
- DLQ items are reviewed by the platform operations team. Remediation options: manual retry, skip the failing stage, or discard.
- DLQ depth is monitored; alerts fire if DLQ depth exceeds a threshold (default: 50 items).

### Pipeline metrics

The following metrics are tracked and surfaced on the platform operations dashboard:

| Metric | Description |
|---|---|
| `ingestion_rate` | Signals ingested per hour, by ingestion mode |
| `fetch_success_rate` | Proportion of fetch attempts that succeed |
| `fetch_failure_rate` | Proportion of fetch attempts that fail, by error code |
| `enrichment_success_rate` | Proportion of enrichment calls that return valid results |
| `cce_classification_rate` | Proportion of signals that complete CCE classification |
| `boundary_case_rate` | Proportion of signals that result in a CCE boundary case requiring analyst review |
| `deduplication_skip_rate` | Proportion of fetched items skipped as exact duplicates |
| `near_duplicate_flag_rate` | Rate at which near-duplicate review tasks are created |
| `dlq_depth` | Current dead letter queue depth |
| `sensitive_content_rate` | Proportion of signals routed to the Sensitive Content Store |

### Alerting thresholds (defaults)

| Alert | Threshold |
|---|---|
| Source fetch failure | 5 consecutive failures for a single source |
| DLQ depth | > 50 items |
| Overall fetch failure rate | > 20% in a 1-hour window |
| CCE classification rate drop | < 80% in a 1-hour window |

---

## 9. Decision Needed Items

> **DECISION NEEDED —** Which enrichment providers are in scope for MVP (company context, tech context, threat intel, patent data)? *Recommendation: start with one company context provider and one tech context provider for MVP; add threat intel and patent data providers in Phase 2. Specific provider names are an [Internal decision rule] and are not recorded in this spec.*

> **DECISION NEEDED —** Should the pipeline support real-time webhooks from sources (push model) in addition to scheduled pull? *Recommendation: pull-only for MVP; add webhook support in Phase 2 for high-priority sources that offer push delivery. Webhook support requires additional infrastructure for endpoint registration, secret verification, and replay handling.*

> **DECISION NEEDED —** What is the content retention policy for content referenced by `raw_content_ref` stored at ingestion? *Recommendation: retain raw content in external storage for 90 days to allow re-processing (e.g., if the AI extraction model is updated or replaced); after 90 days, retain only the extracted structured Signal data and the reference key may be invalidated. Sensitive content retention is governed separately by TS-08.*

> **DECISION NEEDED —** Should near-duplicate detection (semantic similarity computation) run synchronously within the pipeline or as a background job after ingestion? *Recommendation: run as a background job after ingestion; do not block the ingestion pipeline on embedding computation. The cost of a brief window during which a near-duplicate appears as a separate signal before background detection runs is acceptable; blocking ingestion on a potentially slow embedding call is not.*

---

*End of TS-03*
