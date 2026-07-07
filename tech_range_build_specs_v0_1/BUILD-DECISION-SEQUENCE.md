# Tech Range Build Decision Sequence

**Version:** v0.1  
**Status:** Build-planning draft

## Purpose

This document sequences the major decisions needed before Tech Range can be built responsibly.

The key risk is building UI or ingestion before resolving the data model, comparative classification model, governance model, and provenance model. That would recreate the current documentation problem in software form.

## Phase 0 — Confirm build boundary

Decisions:

1. What is the minimum viable platform boundary?
2. Is Tech Range one product, multiple services, or a platform plus engines?
3. Which existing repo becomes the product shell?
4. Which repo owns ingestion and enrichment?
5. Which repo owns deployment and infrastructure?
6. Which material remains design reference or extension?

Outputs:

- ADR-001
- TS-01
- Repo migration / folder plan

## Phase 1 — Lock the data and relationship model

Decisions:

1. What persistence model supports Signals, Groups, Clusters, Profiles, Assignments, and Advisories?
2. Are relationships stored relationally, graph-style, or hybrid?
3. How are match confidence, evidence basis, and rationale stored?
4. How are event time, capture time, grouping time, classification time, evaluation time, and advisory publication time preserved?
5. How are negative evidence and disconfirmation represented?

Outputs:

- ADR-002
- ADR-003
- ADR-004
- ADR-015
- TS-02
- TS-05

## Phase 2 — Define comparative classification

Decisions:

1. What objects can receive ranked assignments?
2. Which axes support primary and secondary matches?
3. How is match strength calculated?
4. How is differentiation from close alternatives calculated?
5. When does a close secondary match trigger analyst review?
6. When can classifications reshuffle automatically?
7. How are old assignments preserved historically?

Outputs:

- ADR-005
- TS-04
- DS-03

## Phase 3 — Define ingestion and enrichment boundaries

Decisions:

1. Which sources are in MVP?
2. What is a primary Signal vs enrichment?
3. How are third-party enrichments distinguished from source facts and analyst judgment?
4. What source credibility model is needed?
5. What data may be stored, masked, excluded, or referenced only by pointer?

Outputs:

- ADR-012
- ADR-014
- TS-03
- TS-08

## Phase 4 — Define object governance and state machines

Decisions:

1. When does a Signal become review-worthy?
2. When is a Corroboration Group created?
3. When is a Cluster created?
4. When is a Technology Profile suggested, created, merged, split, archived, or reverted?
5. When does silence count as analyst acquiescence?
6. Which state transitions require human approval?

Outputs:

- ADR-007
- ADR-008
- TS-06
- DS-08

## Phase 5 — Define user workflows

Decisions:

1. What are the primary analyst workflows?
2. What is the default daily/weekly workspace?
3. How does an analyst move from Signal to Observation to Hypothesis to Profile?
4. How does an analyst create and evaluate Solution Candidates?
5. What gets surfaced to leadership?

Outputs:

- DS-01
- DS-02
- DS-04
- DS-05
- DS-06

## Phase 6 — Define strategic outputs and visualization

Decisions:

1. What is required for an Advisory?
2. What is a TL/DR brief versus an Advisory?
3. What does the radar show?
4. How do Fidelity, Availability, Engagement Posture, and Domain Area appear in portfolio views?
5. What cannot be inferred from a visualization?

Outputs:

- ADR-010
- ADR-013
- DS-06
- DS-07

## Phase 7 — Define operational architecture

Decisions:

1. Where does the system run?
2. What are security boundaries?
3. How are jobs scheduled and monitored?
4. How are AI operations observed and evaluated?
5. How are integrations versioned and governed?

Outputs:

- TS-09
- TS-10
- TS-11
- TS-12

## Recommended first build decision workshop

Use the first workshop to decide:

1. MVP platform boundary.
2. Core data store approach.
3. Comparative assignment model.
4. Technology Profile creation governance.
5. Sensitive evidence handling.

Do not decide screen layouts before these are resolved.
