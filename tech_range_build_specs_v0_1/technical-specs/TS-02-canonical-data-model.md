# TS-02 — Canonical Data Model

**Version:** v0.1  
**Status:** Draft  
**Date:** 2026-07-06  
**Depends on:** ADR-002 (storage model), ADR-003 (evidence graph), ADR-005 (comparative classification), ADR-006 (availability sub-dimensions)

## Purpose

This document translates the Tech Range ontology into a concrete relational data model targeting PostgreSQL. It defines all entity tables, relationship tables, the match_set classification table, and temporal provenance fields.

This is a **logical and physical data model** — it defines what exists in the database and why. It is not an API contract, ORM schema, or migration script (those are implementation outputs derived from this spec).

## Design principles

1. Every classification is stored as a match_set record, not a column. See ADR-005.
2. Evidence records are never hard-deleted. Use `archived_at` to retire.
3. Match Set records are immutable once approved. Reclassification creates a new record and closes the old one.
4. Every relationship between objects is typed, timestamped, and carries confidence.
5. Temporal provenance is preserved with separate fields for event_time, captured_at, ingested_at, created_at, and valid_from per ADR-015.
6. Sensitive evidence references use a pointer pattern — the sensitive content lives in an internal-only store; only a reference ID is stored here.

## Naming conventions

- Table names: `snake_case`, plural
- Primary keys: `id` (UUID v4)
- Foreign keys: `{table_singular}_id`
- Timestamps: suffix `_at` (e.g. `created_at`, `captured_at`)
- Boolean flags: prefix `is_` (e.g. `is_boundary_case`, `is_archived`)
- Enum-like string fields: use PostgreSQL `TEXT` with a `CHECK` constraint or a reference table

---

## Core tables

### `sources`

A Source is a system, platform, publication, outlet, or data provider that produces information.

```sql
CREATE TABLE sources (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name              TEXT NOT NULL,
  source_type       TEXT NOT NULL CHECK (source_type IN (
                      'academic', 'news', 'social', 'repository', 'patent',
                      'funding', 'standards', 'vendor', 'government',
                      'conference', 'enrichment_provider', 'other'
                    )),
  base_url          TEXT,
  credibility_score NUMERIC(3,2) CHECK (credibility_score BETWEEN 0 AND 1),
  -- credibility_score: 0.0–1.0, influences Signal confidence upstream
  -- populated by source evaluation process (see ADR-014)

  -- ADR-014 credibility axis columns
  reliability               NUMERIC(3,2),          -- analyst override (NULL = use system value)
  domain_relevance          NUMERIC(3,2),          -- analyst override
  recency_weight            NUMERIC(3,2),          -- analyst override
  reliability_system        NUMERIC(3,2),          -- system-computed value (preserved on override)
  domain_relevance_system   NUMERIC(3,2),          -- system-computed value
  recency_weight_system     NUMERIC(3,2),          -- system-computed value
  last_signal_at            TIMESTAMPTZ,           -- most recent signal captured_at (used for recency decay)

  coverage_areas    TEXT[],
  -- free-text tags describing topical/geographic coverage
  refresh_cadence   TEXT,
  -- human-readable description: 'real-time', 'daily', 'weekly', etc.
  access_method     TEXT CHECK (access_method IN ('api', 'rss', 'scrape', 'manual', 'partner', 'internal')),
  known_limitations TEXT,
  is_active         BOOLEAN NOT NULL DEFAULT true,
  is_sensitive      BOOLEAN NOT NULL DEFAULT false,
  -- if true, source identity may not appear in public-safe Advisories
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  archived_at       TIMESTAMPTZ,

  -- Ingestion tracking
  last_fetched_at       TIMESTAMPTZ,
  fetch_failure_count   INTEGER NOT NULL DEFAULT 0,
  poll_interval_minutes INTEGER NOT NULL DEFAULT 1440,  -- default: once per day
  source_status         TEXT NOT NULL DEFAULT 'active'
                        CHECK (source_status IN ('active', 'paused', 'health_alert', 'retired'))
);
```

> **DECISION NEEDED — Source credibility scoring:** Who sets `credibility_score` and how? Options: (a) analyst-assigned manually, (b) computed from track record of source-level corroboration accuracy, (c) imported from a third-party source reputation service. *Recommendation: start with analyst-assigned; automate once enough history exists. See ADR-014.*

---

### `signals`

A Signal is the atomic unit of evidence — a timestamped piece of information from a Source.

```sql
CREATE TABLE signals (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source_id           UUID NOT NULL REFERENCES sources(id),
  source_url          TEXT,
  source_ref          TEXT,
  -- stable identifier within the source system (e.g. arXiv ID, GitHub SHA, DOI)

  -- Temporal provenance (five-timestamp standard per ADR-015 — do not collapse)
  event_time          TIMESTAMPTZ,
  -- world-time: when the underlying event occurred (publication date, commit time, etc.)
  -- NULL if unknown; see event_time_unknown / event_time_estimated flags below
  captured_at         TIMESTAMPTZ NOT NULL,
  -- source publication time: when the source published/released/indexed this signal
  -- extracted from source metadata by the ingestion pipeline; never defaulted
  ingested_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  -- pipeline receipt time: when Tech Range's ingestion pipeline first received this signal
  -- (this is what earlier drafts called 'captured_at' — renamed to correct the semantic)
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  -- when the DB record was created
  valid_from          TIMESTAMPTZ NOT NULL DEFAULT now(),
  -- classification/relationship validity start

  -- event_time annotation flags (required when event_time is NULL or estimated)
  event_time_unknown  BOOLEAN NOT NULL DEFAULT false,
  -- true when event_time cannot be determined from any source
  event_time_estimated BOOLEAN NOT NULL DEFAULT false,
  -- true when event_time is inferred/estimated rather than confirmed from source metadata

  CONSTRAINT event_time_flag_mutex CHECK (
    NOT (event_time_unknown AND event_time_estimated)
  ),
  CONSTRAINT event_time_unknown_requires_null CHECK (
    NOT (event_time_unknown AND event_time IS NOT NULL)
  ),

  -- Workflow state
  signal_state        TEXT NOT NULL DEFAULT 'new'
                      CHECK (signal_state IN ('new', 'triaged', 'grouped', 'archived', 'rejected')),

  title               TEXT,
  summary             TEXT,

  -- Ingestion metadata
  ingestion_mode        TEXT NOT NULL DEFAULT 'automated_pull'
                        CHECK (ingestion_mode IN ('automated_pull', 'analyst_submitted', 'intentional_injection')),
  raw_content_ref       TEXT,                    -- pointer/key to raw content in external storage (S3, blob); NOT inline
  content_hash          TEXT,                    -- SHA-256 of raw content for deduplication
  model_version         TEXT,                    -- AI model version used for extraction (e.g. 'extraction:v1.2')
  ai_generated          BOOLEAN NOT NULL DEFAULT false,  -- true if summary/entities were AI-generated
  created_by            TEXT NOT NULL,           -- analyst user ID or 'system' for automated ingestion

  -- Extraction metadata
  extracted_entities  JSONB,
  -- {organizations: [], technologies: [], people: [], locations: [], topics: []}
  initial_tags        TEXT[],
  language            TEXT DEFAULT 'en',
  extraction_confidence NUMERIC(3,2) CHECK (extraction_confidence BETWEEN 0 AND 1),
  -- confidence in the accuracy of entity extraction and summary

  signal_type         TEXT NOT NULL CHECK (signal_type IN (
                        'paper',              -- academic or technical paper
                        'model_release',      -- AI/ML model release
                        'product_launch',     -- product or service launch
                        'code_commit',        -- significant open-source commit or release
                        'benchmark',          -- performance benchmark or evaluation result
                        'funding',            -- investment, grant, or acquisition funding
                        'acquisition',        -- company acquisition or merger
                        'partnership',        -- strategic partnership or collaboration
                        'regulation',         -- regulatory action, policy, or standard
                        'threat_report',      -- threat intelligence or adversarial activity report
                        'patent',             -- patent filing or grant
                        'job_posting',        -- hiring signal (e.g. specialized roles)
                        'conference_talk',    -- conference presentation or keynote
                        'vendor_blog',        -- vendor announcement or technical blog
                        'news',               -- general news article
                        'star_spike',         -- GitHub or similar repository activity spike
                        'standards_update',   -- standards body update (ISO, NIST, IEEE, etc.)
                        'intentional_injection', -- analyst-created signal without source URL
                        'other'               -- does not fit any above category
                      )),

  is_sensitive        BOOLEAN NOT NULL DEFAULT false,
  -- if true, content or source is restricted; handled by pointer pattern
  sensitive_ref_id    TEXT,
  -- pointer to internally-stored sensitive content; null if not sensitive

  is_archived         BOOLEAN NOT NULL DEFAULT false,
  archived_at         TIMESTAMPTZ,
  archive_reason      TEXT,

  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX signals_source_id_idx ON signals(source_id);
CREATE INDEX signals_event_time_idx ON signals(event_time) WHERE event_time IS NOT NULL;
CREATE INDEX signals_captured_at_idx ON signals(captured_at);
CREATE INDEX signals_ingested_at_idx ON signals(ingested_at);
CREATE INDEX signals_valid_from_idx ON signals(valid_from);
CREATE INDEX signals_signal_type_idx ON signals(signal_type);
CREATE INDEX signals_signal_state_idx ON signals(signal_state);
```

---

### `enrichments`

Enrichment is additional context added to any platform object by an external tool, analyst, or process.

```sql
CREATE TABLE enrichments (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  object_id       UUID NOT NULL,
  -- ID of the enriched object (signal, group, cluster, profile, etc.)
  object_type     TEXT NOT NULL CHECK (object_type IN (
                    'signal', 'corroboration_group', 'cluster',
                    'tech_profile', 'solution_candidate', 'advisory'
                  )),
  enrichment_type TEXT NOT NULL CHECK (enrichment_type IN (
                    'entity_extraction', 'source_credibility',
                    'market_context', 'company_context', 'funding_context',
                    'patent_linkage', 'analyst_note', 'ai_summary',
                    'third_party_tool', 'standards_linkage', 'other'
                  )),
  provider        TEXT NOT NULL,
  -- 'harmonic', 'alphasense', 'analyst:{user_id}', 'system:{process_name}'
  content         JSONB NOT NULL,
  -- typed payload; schema varies by enrichment_type
  is_ai_generated BOOLEAN NOT NULL DEFAULT false,
  -- AI-generated enrichments must be distinguishable from source facts
  confidence      NUMERIC(3,2) CHECK (confidence BETWEEN 0 AND 1),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
  is_retracted    BOOLEAN NOT NULL DEFAULT false,
  retracted_at    TIMESTAMPTZ,
  retraction_note TEXT
);

CREATE INDEX enrichments_object_idx ON enrichments(object_type, object_id);
```

---

### `corroboration_groups`

A Corroboration Group contains Signals describing the same underlying event, announcement, release, or storyline.

```sql
CREATE TABLE corroboration_groups (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_summary         TEXT NOT NULL,
  -- human-readable description of the shared event or storyline
  corroboration_strength NUMERIC(3,2) CHECK (corroboration_strength BETWEEN 0 AND 1),
  source_diversity_score NUMERIC(3,2) CHECK (source_diversity_score BETWEEN 0 AND 1),
  -- proportion of member signals from distinct sources
  first_seen_at         TIMESTAMPTZ,
  -- event_time of the earliest member signal
  latest_seen_at        TIMESTAMPTZ,
  -- event_time of the most recent member signal
  member_count          INTEGER NOT NULL DEFAULT 0,
  -- denormalized count; updated by trigger

  -- Workflow state (aligned with TS-06/ADR-008)
  group_state           TEXT NOT NULL DEFAULT 'forming'
                        CHECK (group_state IN ('forming', 'active', 'closed', 'dissolved')),

  review_status         TEXT NOT NULL DEFAULT 'system_created'
                        CHECK (review_status IN ('system_created', 'analyst_reviewed', 'approved', 'disputed')),
  created_by            TEXT NOT NULL,
  -- 'system:{process}' or user ID
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_from            TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  is_archived           BOOLEAN NOT NULL DEFAULT false,
  archived_at           TIMESTAMPTZ
);

CREATE TABLE corroboration_group_members (
  group_id    UUID NOT NULL REFERENCES corroboration_groups(id),
  signal_id   UUID NOT NULL REFERENCES signals(id),
  is_primary  BOOLEAN NOT NULL DEFAULT false,
  -- one signal may be designated the canonical representative
  added_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  added_by    TEXT NOT NULL,
  PRIMARY KEY (group_id, signal_id)
);
```

---

### `clusters`

A Cluster contains related Signals and/or Corroboration Groups that collectively indicate a broader pattern or theme.

```sql
CREATE TABLE clusters (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  theme           TEXT NOT NULL,
  summary         TEXT,
  coherence_score NUMERIC(3,2) CHECK (coherence_score BETWEEN 0 AND 1),
  -- how tightly the member objects relate to the stated theme
  first_signal_at TIMESTAMPTZ,
  latest_signal_at TIMESTAMPTZ,
  signal_count    INTEGER NOT NULL DEFAULT 0,
  group_count     INTEGER NOT NULL DEFAULT 0,
  -- burst indicators
  is_bursting     BOOLEAN NOT NULL DEFAULT false,
  burst_detected_at TIMESTAMPTZ,
  burst_score     NUMERIC(3,2),
  -- relative acceleration in signal volume

  -- Workflow state (aligned with TS-06/ADR-008)
  cluster_state   TEXT NOT NULL DEFAULT 'candidate'
                  CHECK (cluster_state IN ('candidate', 'active', 'promoted', 'dissolved')),

  review_status   TEXT NOT NULL DEFAULT 'system_created'
                  CHECK (review_status IN ('system_created', 'analyst_reviewed', 'approved', 'disputed')),
  created_by      TEXT NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  is_archived     BOOLEAN NOT NULL DEFAULT false,
  archived_at     TIMESTAMPTZ
);

CREATE TABLE cluster_members (
  cluster_id    UUID NOT NULL REFERENCES clusters(id),
  member_id     UUID NOT NULL,
  member_type   TEXT NOT NULL CHECK (member_type IN ('signal', 'corroboration_group')),
  match_score   NUMERIC(3,2) CHECK (match_score BETWEEN 0 AND 1),
  -- how strongly this member contributes to the cluster theme
  added_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  added_by      TEXT NOT NULL,
  PRIMARY KEY (cluster_id, member_id, member_type)
);
```

---

### `neighborhoods`

Neighborhoods are strategic technology areas. They are reference data — a small, managed set.

```sql
CREATE TABLE neighborhoods (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT NOT NULL UNIQUE,
  slug        TEXT NOT NULL UNIQUE,
  -- machine-readable identifier: 'agentic_vertical_ai', 'edge_ai_ran', etc.
  description TEXT,
  is_emerging BOOLEAN NOT NULL DEFAULT false,
  -- true only for the 'Emerging' holding area
  is_active   BOOLEAN NOT NULL DEFAULT true,
  sort_order  INTEGER,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  retired_at  TIMESTAMPTZ
);

-- Seed data (managed migration, not runtime inserts):
-- Agentic / Vertical AI
-- Edge / AI-RAN
-- Physical AI
-- Quantum
-- Software-Defined Everything (SDx)
-- Cloud Platforms
-- Emerging (is_emerging = true)
```

> **DECISION NEEDED — Neighborhood change governance:** Adding, renaming, or retiring a Neighborhood must trigger batch reclassification of all match_set records for that neighborhood. Who approves this change, and is there a preview/dry-run step before committing? *Recommendation: require a documented ADR or change record, a dry-run report showing affected object counts, and a designated approver (likely a platform architect or senior analyst). Define formally in ADR-001.*

---

### `domain_areas`

Domain Areas describe where a technology may matter for the firm, clients, missions, or markets.

```sql
CREATE TABLE domain_areas (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT NOT NULL UNIQUE,
  slug        TEXT NOT NULL UNIQUE,
  description TEXT,
  is_active   BOOLEAN NOT NULL DEFAULT true,
  sort_order  INTEGER,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  retired_at  TIMESTAMPTZ
);

-- Seed data:
-- National Security & Cyber
-- Defense Tech
-- Global Defense
-- Civil
-- Space Systems
-- Vertical AI / Data Platforms
```

---

### `match_sets`

The central classification table. Every assignment across every axis for every object is stored here. See ADR-005 for the full model.

```sql
CREATE TABLE match_sets (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  object_id         UUID NOT NULL,
  object_type       TEXT NOT NULL CHECK (object_type IN (
                      'signal', 'corroboration_group', 'cluster',
                      'tech_profile', 'solution_candidate'
                    )),
  axis              TEXT NOT NULL CHECK (axis IN (
                      'neighborhood', 'domain_area', 'tech_type', 'candidate_type'
                    )),

  -- Primary match
  primary_match     TEXT NOT NULL,
  -- the category name/slug of the best-fit match
  primary_score     NUMERIC(3,2) NOT NULL CHECK (primary_score BETWEEN 0 AND 1),

  -- Secondary matches (ordered array of {category, score} objects)
  secondary_matches JSONB NOT NULL DEFAULT '[]',
  -- [{"category": "physical_ai", "score": 0.61}, ...]

  -- Differentiation
  diff_strength     NUMERIC(3,2) NOT NULL CHECK (diff_strength BETWEEN 0 AND 1),
  -- primary_score minus highest secondary score
  is_boundary_case  BOOLEAN NOT NULL GENERATED ALWAYS AS (diff_strength < 0.15) STORED,

  comparison_set    TEXT[] NOT NULL,
  -- all category slugs considered during this classification run

  evidence_basis    UUID[] NOT NULL DEFAULT '{}',
  -- IDs of signals/groups/clusters/observations supporting this match
  rationale         TEXT,
  ambiguity_notes   TEXT,
  -- required when is_boundary_case = true

  review_status     TEXT NOT NULL DEFAULT 'system_assigned'
                    CHECK (review_status IN (
                      'system_assigned', 'analyst_reviewed', 'approved',
                      'disputed', 'needs_internal_answer'
                    )),
  created_by        TEXT NOT NULL,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  reviewed_by       UUID,
  -- FK to users table (defined by auth layer)
  reviewed_at       TIMESTAMPTZ,

  -- Temporal validity (immutable-but-supersedable pattern per ADR-005 / ADR-015)
  valid_from        TIMESTAMPTZ NOT NULL DEFAULT now(),
  superseded_at     TIMESTAMPTZ,
  superseded_by     UUID REFERENCES match_sets(id),
  -- null if this is the current active assignment

  is_active         BOOLEAN NOT NULL GENERATED ALWAYS AS (superseded_at IS NULL) STORED
);

CREATE INDEX match_sets_object_idx ON match_sets(object_type, object_id);
CREATE INDEX match_sets_axis_idx ON match_sets(axis);
CREATE INDEX match_sets_primary_match_idx ON match_sets(primary_match);
CREATE INDEX match_sets_boundary_idx ON match_sets(is_boundary_case) WHERE is_boundary_case = true;
CREATE INDEX match_sets_review_status_idx ON match_sets(review_status);
CREATE INDEX match_sets_active_idx ON match_sets(object_type, object_id, axis) WHERE is_active = true;
-- Use this index for "current classification" lookups
```

---

### `boundary_cases`

Tracks objects flagged as boundary cases during classification — i.e., where the differentiation between the primary match and competing matches is below the threshold. Referenced by TS-04 (Comparative Classification Engine).

```sql
CREATE TABLE boundary_cases (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  match_set_id      UUID NOT NULL REFERENCES match_sets(id),
  object_id         UUID NOT NULL,
  object_type       TEXT NOT NULL,
  axis              TEXT NOT NULL,
  diff_strength     NUMERIC(3,2) NOT NULL,
  primary_match     TEXT NOT NULL,
  primary_score     NUMERIC(3,2) NOT NULL,
  competing_matches JSONB NOT NULL DEFAULT '[]',
  cce_reasoning     TEXT,
  resolved          BOOLEAN NOT NULL DEFAULT false,
  resolved_by       UUID,
  resolved_at       TIMESTAMPTZ,
  resolution_notes  TEXT,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

### `observations`

An Observation synthesizes what evidence indicates without speculation.

```sql
CREATE TABLE observations (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tech_profile_id  UUID REFERENCES tech_profiles(id),
  -- null if observation is not yet linked to a profile
  summary          TEXT NOT NULL,
  detail           TEXT,
  evidence_ids     UUID[] NOT NULL DEFAULT '{}',
  -- signals, groups, clusters supporting this observation
  confidence       NUMERIC(3,2) CHECK (confidence BETWEEN 0 AND 1),
  created_by       TEXT NOT NULL,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_from       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  is_retracted     BOOLEAN NOT NULL DEFAULT false,
  retracted_at     TIMESTAMPTZ,
  retraction_note  TEXT
);
```

---

### `hypotheses`

A Hypothesis is a testable explanation or prediction based on Observations.

```sql
CREATE TABLE hypotheses (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tech_profile_id  UUID REFERENCES tech_profiles(id),
  statement        TEXT NOT NULL,
  implication      TEXT,
  time_horizon     TEXT,
  -- human-readable: '6 months', '1–2 years', etc.
  test_path        TEXT,
  -- how this hypothesis would be tested
  status           TEXT NOT NULL DEFAULT 'draft'
                   CHECK (status IN ('draft', 'active', 'supported', 'weakened', 'rejected', 'unresolved')),
  confidence       NUMERIC(3,2) CHECK (confidence BETWEEN 0 AND 1),
  supporting_observation_ids UUID[] NOT NULL DEFAULT '{}',
  disconfirming_evidence_ids UUID[] NOT NULL DEFAULT '{}',
  -- explicitly track contrary evidence
  created_by       TEXT NOT NULL,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_from       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  resolved_at      TIMESTAMPTZ,
  resolution_note  TEXT
);
```

---

### `tech_profiles`

A Technology Profile is the organization's evolving understanding of a technology.

```sql
CREATE TABLE tech_profiles (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name                  TEXT NOT NULL,
  working_name          TEXT,
  aliases               TEXT[],
  description           TEXT,

  -- Classification (actual values stored in match_sets; these are denormalized for query convenience)
  tech_type             TEXT CHECK (tech_type IN ('tool', 'platform', 'technique', 'framework_model', 'unknown')),
  -- denormalized from active match_set on axis='tech_type'

  -- Creation
  creation_mode         TEXT NOT NULL CHECK (creation_mode IN ('emergent', 'intentional')),
  requester_note        TEXT,
  -- for intentional creation: why/who prompted this

  -- Workflow state (enforced by WorkflowStateService; aligned with TS-06/ADR-008)
  current_state         TEXT NOT NULL DEFAULT 'candidate'
                        CHECK (current_state IN (
                          'candidate', 'emerging', 'active', 'under_review', 'archived'
                        )),

  -- Evidence completeness (separate from workflow state — how well-understood the technology is)
  evidence_completeness TEXT NOT NULL DEFAULT 'unknown'
                        CHECK (evidence_completeness IN (
                          'unknown', 'partial', 'known', 'superseded'
                        )),

  -- External technology state
  fidelity_level        INTEGER CHECK (fidelity_level BETWEEN 0 AND 5),
  fidelity_rationale    TEXT,
  fidelity_trend        TEXT CHECK (fidelity_trend IN ('rising', 'stable', 'declining', 'unknown')),

  availability_level    INTEGER CHECK (availability_level BETWEEN 0 AND 5),
  -- composite availability score: min(market_availability, firm_access) per ADR-006
  availability_rationale TEXT,
  availability_trend    TEXT CHECK (availability_trend IN ('rising', 'stable', 'declining', 'unknown')),

  -- Availability sub-dimensions (ADR-006)
  market_availability      INTEGER CHECK (market_availability BETWEEN 0 AND 5),
  firm_access              INTEGER CHECK (firm_access BETWEEN 0 AND 5),
  deployment_readiness     INTEGER CHECK (deployment_readiness BETWEEN 0 AND 5),
  procurement_readiness    INTEGER CHECK (procurement_readiness BETWEEN 0 AND 5),

  -- Per-dimension confidence (Platform Principle P19)
  fidelity_confidence      NUMERIC(3,2) CHECK (fidelity_confidence BETWEEN 0 AND 1),
  availability_confidence  NUMERIC(3,2) CHECK (availability_confidence BETWEEN 0 AND 1),

  -- Reassessment tracking
  fidelity_assessed_at     TIMESTAMPTZ,
  availability_assessed_at TIMESTAMPTZ,
  previous_fidelity_level  INTEGER CHECK (previous_fidelity_level BETWEEN 0 AND 5),
  previous_availability_level INTEGER CHECK (previous_availability_level BETWEEN 0 AND 5),

  -- Signal archaeology
  archaeology_status    TEXT DEFAULT 'not_started'
                        CHECK (archaeology_status IN ('not_started', 'in_progress', 'complete', 'not_applicable')),
  archaeology_notes     TEXT,

  -- Profile confidence
  profile_confidence    NUMERIC(3,2) CHECK (profile_confidence BETWEEN 0 AND 1),
  -- overall confidence in this profile's current state of knowledge
  confidence_rationale  TEXT,

  -- Relationships
  parent_profile_id     UUID REFERENCES tech_profiles(id),
  -- max one level of nesting

  -- Governance
  review_status         TEXT NOT NULL DEFAULT 'candidate'
                        CHECK (review_status IN ('candidate', 'active', 'under_review', 'archived')),
  created_by            TEXT NOT NULL,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_from            TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  archived_at           TIMESTAMPTZ,
  archive_reason        TEXT,

  -- Recommended next action (updated by Tech Evaluation process)
  recommended_next_step TEXT
);

CREATE INDEX tech_profiles_current_state_idx ON tech_profiles(current_state);
CREATE INDEX tech_profiles_evidence_completeness_idx ON tech_profiles(evidence_completeness);
CREATE INDEX tech_profiles_review_status_idx ON tech_profiles(review_status);
CREATE INDEX tech_profiles_fidelity_availability_idx ON tech_profiles(fidelity_level, availability_level);
```

> **DECISION NEEDED — Profile auto-creation threshold:** How many corroborating signals in a Cluster, with what minimum coherence_score, should trigger automatic candidate profile creation? *Recommendation: 3+ signals from 2+ distinct sources in a Cluster with coherence_score >= 0.60. This is tunable. Define formally in ADR-007.*

---

### `tech_profile_evidence`

Links evidence objects (Signals, Groups, Clusters) to a Technology Profile.

```sql
CREATE TABLE tech_profile_evidence (
  profile_id    UUID NOT NULL REFERENCES tech_profiles(id),
  evidence_id   UUID NOT NULL,
  evidence_type TEXT NOT NULL CHECK (evidence_type IN ('signal', 'corroboration_group', 'cluster')),
  contribution_type TEXT CHECK (contribution_type IN ('supporting', 'disconfirming', 'contextual')),
  added_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  added_by      TEXT NOT NULL,
  PRIMARY KEY (profile_id, evidence_id, evidence_type)
);
```

---

### `tech_evaluations`

A Tech Evaluation is a formal assessment of a Technology Profile.

```sql
CREATE TABLE tech_evaluations (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tech_profile_id     UUID NOT NULL REFERENCES tech_profiles(id),
  evaluator           TEXT NOT NULL,
  -- user ID or process name
  evaluation_date     TIMESTAMPTZ NOT NULL DEFAULT now(),

  -- Inputs summary
  evidence_summary    TEXT,
  fidelity_at_eval    INTEGER CHECK (fidelity_at_eval BETWEEN 0 AND 5),
  availability_at_eval INTEGER CHECK (availability_at_eval BETWEEN 0 AND 5),
  confidence_at_eval  NUMERIC(3,2) CHECK (confidence_at_eval BETWEEN 0 AND 1),

  -- Outputs
  recommendation      TEXT NOT NULL CHECK (recommendation IN (
                        'continue_monitoring', 'deepen_research', 'create_solution_candidate',
                        'revise_solution_candidate', 'prepare_advisory', 'pursue_partner_access',
                        'recommend_prototype', 'venture_investment_scout', 'archive_hold'
                      )),
  recommendation_rationale TEXT NOT NULL,
  next_review_date    DATE,

  status              TEXT NOT NULL DEFAULT 'draft'
                      CHECK (status IN ('draft', 'submitted', 'approved', 'superseded')),
  approved_by         TEXT,
  approved_at         TIMESTAMPTZ,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_from          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

### `solution_candidates`

A Solution Candidate is a proposed way for the firm to engage with a Technology Profile.

```sql
CREATE TABLE solution_candidates (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name                  TEXT NOT NULL,
  tech_profile_id       UUID NOT NULL REFERENCES tech_profiles(id),

  -- Candidate type (actual values stored in match_sets axis='candidate_type')
  candidate_type        TEXT,
  -- denormalized from active match_set for convenience

  hypothesis            TEXT,
  strategic_rationale   TEXT NOT NULL,
  access_path           TEXT,

  -- State at time of candidate creation (snapshot for auditability)
  fidelity_at_creation  INTEGER CHECK (fidelity_at_creation BETWEEN 0 AND 5),
  availability_at_creation INTEGER CHECK (availability_at_creation BETWEEN 0 AND 5),
  knowledge_confidence_at_creation NUMERIC(3,2),

  -- Success criteria
  proposed_okrs         JSONB,
  -- [{objective, key_results: []}]
  proposed_kpis         JSONB,
  test_plan             TEXT,

  -- Dependencies and risks
  dependencies          TEXT,
  risks                 TEXT,

  -- Decision
  decision_owner        TEXT,
  recommended_next_step TEXT,

  -- Lifecycle state (aligned with TS-06/ADR-008)
  status                TEXT NOT NULL DEFAULT 'draft'
                        CHECK (status IN (
                          'draft', 'under_review', 'approved', 'handed_off', 'rejected'
                        )),
  converted_to          TEXT CHECK (converted_to IN (
                          'prototype', 'advisory', 'partner_access', 'venture_scout', null
                        )),
  created_by            TEXT NOT NULL,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_from            TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  archived_at           TIMESTAMPTZ,
  archive_reason        TEXT
);

CREATE INDEX solution_candidates_profile_idx ON solution_candidates(tech_profile_id);
CREATE INDEX solution_candidates_status_idx ON solution_candidates(status);
```

---

### `prototypes`

A Prototype is a hands-on validation activity for a selected Solution Candidate.

```sql
CREATE TABLE prototypes (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  solution_candidate_id UUID NOT NULL REFERENCES solution_candidates(id),
  tech_profile_id       UUID NOT NULL REFERENCES tech_profiles(id),
  name                  TEXT NOT NULL,
  description           TEXT,
  hypothesis_being_tested TEXT NOT NULL,
  approach              TEXT,

  -- Timeline
  started_at            TIMESTAMPTZ,
  completed_at          TIMESTAMPTZ,
  target_completion     DATE,

  -- Results
  outcome               TEXT CHECK (outcome IN ('validated', 'invalidated', 'inconclusive', 'abandoned')),
  findings_summary      TEXT,

  -- Lifecycle state (aligned with TS-06/ADR-008)
  status                TEXT NOT NULL DEFAULT 'scoped'
                        CHECK (status IN ('scoped', 'in_progress', 'complete', 'abandoned')),
  created_by            TEXT NOT NULL,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_from            TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

### `findings`

A Finding is validated learning from research, evaluation, or prototyping.

```sql
CREATE TABLE findings (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source_type           TEXT NOT NULL CHECK (source_type IN ('prototype', 'tech_evaluation', 'research', 'external')),
  source_id             UUID,
  -- ID of the prototype, tech_evaluation, etc. that produced this finding
  tech_profile_id       UUID REFERENCES tech_profiles(id),
  solution_candidate_id UUID REFERENCES solution_candidates(id),

  statement             TEXT NOT NULL,
  detail                TEXT,
  evidence_ids          UUID[] NOT NULL DEFAULT '{}',
  finding_type          TEXT CHECK (finding_type IN ('validation', 'invalidation', 'learning', 'risk', 'opportunity')),
  confidence            NUMERIC(3,2) CHECK (confidence BETWEEN 0 AND 1),

  -- Lifecycle state (aligned with TS-06/ADR-008)
  status                TEXT NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft', 'peer_review', 'published', 'withdrawn')),

  created_by            TEXT NOT NULL,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_from            TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

### `advisories`

An Advisory is the primary strategic output of Tech Range.

```sql
CREATE TABLE advisories (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title                 TEXT NOT NULL,
  decision_question     TEXT NOT NULL,
  executive_summary     TEXT NOT NULL,

  -- Linked objects
  tech_profile_ids      UUID[] NOT NULL DEFAULT '{}',
  solution_candidate_ids UUID[] NOT NULL DEFAULT '{}',
  finding_ids           UUID[] NOT NULL DEFAULT '{}',

  -- Content sections (stored as structured text or markdown)
  key_observations      TEXT,
  hypotheses_interpretations TEXT,
  fidelity_assessment   TEXT,
  availability_assessment TEXT,
  domain_area_relevance TEXT,
  strategic_implications TEXT,
  recommended_action    TEXT NOT NULL,
  alternatives_considered TEXT,
  risks_and_uncertainties TEXT,
  evidence_traceability TEXT,
  -- narrative or structured trace from recommendations back to evidence

  -- Confidence
  overall_confidence    NUMERIC(3,2) CHECK (overall_confidence BETWEEN 0 AND 1),
  confidence_rationale  TEXT NOT NULL,

  -- Lifecycle state (aligned with TS-06/ADR-008)
  status                TEXT NOT NULL DEFAULT 'assembling'
                        CHECK (status IN ('assembling', 'review', 'approved', 'published', 'withdrawn')),
  created_by            TEXT NOT NULL,
  reviewed_by           TEXT,
  approved_by           TEXT,
  published_by          TEXT,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_from            TIMESTAMPTZ NOT NULL DEFAULT now(),
  reviewed_at           TIMESTAMPTZ,
  approved_at           TIMESTAMPTZ,
  published_at          TIMESTAMPTZ,
  superseded_at         TIMESTAMPTZ,
  superseded_by         UUID REFERENCES advisories(id)
);
```

> **DECISION NEEDED — Advisory publication requirements:** What minimum fields must be populated before an Advisory can move to `approved` status? *Recommendation: decision_question, executive_summary, recommended_action, confidence_rationale, at least one linked tech_profile_id, at least one finding_id or key_observations entry, and overall_confidence >= 0.50. Codify as a pre-approval check in TS-06 and ADR-013.*

---

## Relationship tables

### `object_relationships`

Generic typed relationship table. Supports all durable relationships between platform objects. Uses a unified relationship type vocabulary merging structural/hierarchical and evidence graph types.

```sql
CREATE TABLE object_relationships (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source_id       UUID NOT NULL,
  source_type     TEXT NOT NULL,
  target_id       UUID NOT NULL,
  target_type     TEXT NOT NULL,
  relationship_type TEXT NOT NULL CHECK (relationship_type IN (
                    'corroborates', 'contradicts', 'clusters_into', 'classified_as',
                    'evidences', 'hypothesizes', 'evaluated_by', 'precedes', 'supersedes',
                    'sourced_from', 'enriched_by',
                    'contains', 'related_to', 'contributes_to', 'assigned_to',
                    'produces', 'parent_of', 'child_of', 'enables', 'dependent_on'
                  )),
  confidence      NUMERIC(3,2) CHECK (confidence BETWEEN 0 AND 1),
  evidence_basis  UUID[] NOT NULL DEFAULT '{}',
  valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
  superseded_at   TIMESTAMPTZ,
  superseded_by   UUID REFERENCES object_relationships(id),
  review_status   TEXT NOT NULL DEFAULT 'pending'
                  CHECK (review_status IN ('pending', 'accepted', 'challenged', 'retracted')),
  rationale       TEXT,
  created_by      TEXT NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX obj_rel_source_idx ON object_relationships(source_type, source_id);
CREATE INDEX obj_rel_target_idx ON object_relationships(target_type, target_id);
CREATE INDEX obj_rel_type_idx ON object_relationships(relationship_type);
```

---

## Emerging tracking

### `emerging_records`

Tracks objects assigned to the Emerging neighborhood and their progression.

```sql
CREATE TABLE emerging_records (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  object_id       UUID NOT NULL,
  object_type     TEXT NOT NULL CHECK (object_type IN ('signal', 'corroboration_group', 'cluster')),
  emerging_state  TEXT NOT NULL DEFAULT 'emerging_signal'
                  CHECK (emerging_state IN (
                    'emerging_signal', 'emerging_pattern', 'emerging_cluster_candidate',
                    'emerging_neighborhood_candidate', 'promoted', 'reclassified', 'archived'
                  )),
  review_notes    TEXT,
  promoted_to     TEXT,
  -- neighborhood slug if promoted
  reviewed_by     TEXT,
  reviewed_at     TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

> **DECISION NEEDED — Emerging review thresholds:** At what signal volume and source diversity does an Emerging object automatically enter the analyst review queue? *Recommendation: 5+ signals from 3+ distinct sources within a 30-day window, or coherence_score >= 0.70 on a Cluster with Emerging primary assignment. Tune after first operational quarter.*

---

## Temporal provenance summary

All major tables preserve the following timestamps per the five-timestamp standard (ADR-015):

| Timestamp field | Meaning |
|---|---|
| `event_time` | When the underlying real-world event occurred (world-time; nullable) |
| `captured_at` | When the source published/released/indexed the signal (source publication time) |
| `ingested_at` | When Tech Range's ingestion pipeline first received the raw signal (pipeline receipt time) |
| `created_at` | When the database record was created |
| `valid_from` | Classification/relationship validity start (immutable-but-supersedable records) |
| `updated_at` | Last modification timestamp |
| `reviewed_at` | When an analyst reviewed the object |
| `approved_at` | When the object was formally approved |
| `archived_at` | When the object was retired |
| `superseded_at` | When a supersedable record was replaced by a newer version |

**Availability sub-dimensions (ADR-006):** `availability_level` is a composite score defined as `min(market_availability, firm_access)`. It is further decomposed into four sub-dimensions stored on `tech_profiles`: `market_availability`, `firm_access`, `deployment_readiness`, and `procurement_readiness` (each 0–5). Per-dimension confidence scores (`fidelity_confidence`, `availability_confidence`) and reassessment tracking fields (`fidelity_assessed_at`, `availability_assessed_at`, `previous_fidelity_level`, `previous_availability_level`) are also stored on the profile per Platform Principle P19.

The gap `captured_at - event_time` measures **source latency** (how long after the world event the source noticed it). The gap `ingested_at - captured_at` measures **pipeline latency** (how current the platform's crawl coverage is). The gap `ingested_at - event_time` is the **total signal latency**. These distinct timestamps support signal archaeology, surprise analysis, and confidence review. See ADR-015 for the authoritative definition.

---

## Indexes and query patterns

Key query patterns this model must support efficiently:

| Query | Index strategy |
|---|---|
| Current classification for an object on an axis | `match_sets_active_idx` on `(object_type, object_id, axis) WHERE is_active` |
| All boundary cases awaiting review | `match_sets_boundary_idx` on `is_boundary_case WHERE true` |
| Signals by Neighborhood (via match_sets) | Join signals to match_sets on axis='neighborhood' WHERE is_active |
| Technology Profile by fidelity/availability | `tech_profiles_fidelity_availability_idx` |
| Signal burst detection | Aggregate on `signals.event_time` by `cluster_id`, windowed by time |
| Classification history for an object | `match_sets_object_idx` + order by `valid_from` |
| Advisories in draft pending approval | `status` filter on advisories |
| Late-signal detection | Filter `signals` on `ingested_at - event_time > INTERVAL '30 days'` WHERE event_time IS NOT NULL |
| Point-in-time knowledge reconstruction | Filter `signals` on `ingested_at < $timestamp`; filter `match_sets` on `valid_from <= $timestamp AND (superseded_at IS NULL OR superseded_at > $timestamp)` |

Full-text search on signal summaries and profile descriptions should use PostgreSQL `tsvector` / `tsquery` or an external search index (Elasticsearch, pg_vector for semantic search). This is deferred to TS-07.

---

## What this spec does not cover

- ORM layer / migration scripts — derived from this spec by engineering
- Auth and user management — referenced as `created_by TEXT` (user ID); full auth model in TS-08
- API contracts — in TS-09
- Search and ranking — in TS-07
- AI assistance layer — in TS-12
- Sensitive content storage — the `sensitive_ref_id` pattern points to an internal-only store; its schema is defined separately in the enterprise-controlled environment
