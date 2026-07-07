# ADR-014: Source Credibility and Trust Model

**Status:** Accepted
**Date:** 2026-07-06
**Deciders:** Tech Range Architecture
**Related:** ADR-012 (Sensitive Content Store), TS-02 (Data Model), TS-03 (Ingestion Pipeline), TS-04 (CCE)

---

## Context

Tech Range ingests signals from a wide range of sources: academic publications, industry news, vendor blogs, government reports, threat intelligence feeds, open-source repositories, conference proceedings, patent databases, and potentially sensitive or non-public sources. These signals feed the Comparative Classification Engine (CCE, TS-04), which uses source credibility as a scoring factor (initial weight: 0.10) when computing Neighborhood classification scores.

The `sources` table in TS-02 includes a `credibility_score NUMERIC(3,2)` column, but no prior spec defines how that score is computed or updated. TS-03 (the ingestion pipeline) explicitly defers to this ADR for source credibility methodology.

### Why this matters architecturally

A single opaque credibility score that feeds the CCE without explanation creates a hidden trust hierarchy. Analysts cannot challenge it, calibrate against it, or understand why signals from certain sources are weighted higher than others. The system will make decisions whose reasoning is partially invisible.

The critical failure mode: a source assigned high credibility turns out to be unreliable — vendor capture, editorial failure, coordinated disinformation, or simply domain drift. All signals attributed to that source carry inflated confidence in derived classifications, with no mechanism to retrospectively identify or correct the damage.

### Three trust questions the model must answer

1. **Reliability:** How consistently accurate and trustworthy has this source been over time?
2. **Domain relevance:** How relevant is this source to the specific Neighborhoods and Domain Areas Tech Range tracks?
3. **Recency:** Is this source still active and current, or has it gone dark or drifted?

A single composite score cannot surface meaningful answers to all three questions independently. Multi-dimensional scoring is required.

---

## Decision

Source credibility is represented as a **multi-dimensional score** with three named axes. Each axis is stored as a separate `NUMERIC(3,2)` column on the `sources` table, bounded 0.0–1.0. The composite `credibility_score` is derived from these three axes and is the value consumed by the CCE.

---

### Axis 1: Reliability (0.0–1.0)

How consistently accurate and trustworthy has this source been?

**Factors:**

- **Track record:** Proportion of signals from this source that were subsequently corroborated by independent evidence vs. contradicted. Updated incrementally as corroboration events and contradiction events are recorded.
- **Editorial standards:** Is the source peer-reviewed, editorially reviewed (with named editorial staff and correction policies), or unreviewed?
- **Independence:** Is the source independent of the technology vendors and sectors it covers, or does it have a financial or organizational relationship that may introduce bias?

**Initial default scores by source type:**

| Source Type | Default Reliability |
|---|---|
| Academic paper (peer-reviewed) | 0.80 |
| Government report (official) | 0.75 |
| Industry analysis (independent analyst firm) | 0.65 |
| News (established editorial publication) | 0.55 |
| Vendor blog / product announcement | 0.40 |
| Social media / forum post | 0.30 |
| Unknown / unclassified | 0.50 |

These defaults apply at the time a source is first registered. They are replaced by track-record-derived scores as corroboration and contradiction events accumulate.

---

### Axis 2: Domain Relevance (0.0–1.0)

How relevant is this source to Tech Range's specific Neighborhoods and Domain Areas?

**Factors:**

- **Coverage alignment:** Does this source regularly publish material that maps to the Neighborhoods and Domain Areas defined in the Tech Range taxonomy? A source that covers general technology news scores lower than one that specializes in, for example, AI safety research or satellite communications.
- **Technical depth:** Does this source provide original technical analysis, or does it aggregate and summarize coverage produced elsewhere? Original technical depth scores higher.

**Initial default scores by source type:**

| Source Type | Default Domain Relevance |
|---|---|
| Specialist academic publication (domain-matched) | 0.90 |
| Specialist industry publication (domain-matched) | 0.80 |
| Government technical report (domain-matched) | 0.75 |
| General technology news | 0.55 |
| Broad industry analysis | 0.50 |
| Vendor blog (domain-relevant product area) | 0.60 |
| Vendor blog (tangential or marketing-focused) | 0.30 |
| Social media / forum (specialist community) | 0.45 |
| Social media / forum (general) | 0.20 |
| Unknown / unclassified | 0.40 |

Domain relevance scores are set at source registration and revised when analysts add or update domain coverage tags on a source record.

---

### Axis 3: Recency Weight (0.0–1.0)

How current and active is this source?

**Factors:**

- **Publication frequency:** How often does this source publish relevant material? A source that publishes daily scores higher than one that publishes quarterly, all else equal.
- **Last active date:** How recently has this source published material that was ingested by Tech Range?
- **Decay function:** A source that goes dark loses recency weight over time. The decay is applied on a rolling basis:

| Time since last ingested signal | Recency Weight multiplier |
|---|---|
| < 30 days | 1.0 (no decay) |
| 30–90 days | 0.85 |
| 91–180 days | 0.65 |
| 181–365 days | 0.40 |
| > 365 days | 0.20 |

The recency weight for a source is: `base_recency_score × decay_multiplier`, where `base_recency_score` reflects publication frequency (set at registration, range 0.4–1.0) and `decay_multiplier` is applied from the table above based on the delta between today and the source's `last_signal_at` timestamp.

---

### Composite credibility_score

The composite score that feeds the CCE is:

```
credibility_score = (reliability × 0.50) + (domain_relevance × 0.30) + (recency_weight × 0.20)
```

**Rationale for weights:** Reliability is the most consequential dimension — a source that is domain-relevant but consistently wrong is worse than useless. Domain relevance is weighted second because a highly reliable general source still contributes less signal value to a specialized platform than a reliable specialist source. Recency is a modifier that prevents over-reliance on historically strong but currently inactive sources.

This composite is the value stored in `sources.credibility_score` and is what the CCE reads.

---

### Analyst override

Analysts may manually set any axis score (reliability, domain_relevance, recency_weight) with a required rationale field. Manual overrides obey the following rules:

- The manually set value takes precedence over the system-computed value for that axis immediately upon save.
- The system-computed value for that axis is preserved in a separate `_system` column (e.g., `reliability_system`, `domain_relevance_system`) so the divergence is visible.
- Override history (who changed what, from what value, to what value, with what rationale, at what timestamp) is written to the source audit trail and is non-destructive.
- An analyst may revert to the system-computed score at any time; this also records an audit entry.

Manual overrides do not suppress automatic score update events (corroboration nudges, contradiction nudges, recency decay). The system continues to update the `_system` columns in the background. Analysts are notified when system-computed scores diverge significantly from their manual overrides (delta > 0.15 on any axis), so they can review whether the override remains justified.

---

## Enrichment Provider Trust Levels

Enrichment data (records in the TS-02 `enrichment` table) carries a trust level derived from the enrichment provider. This trust level propagates to any confidence score that incorporates that enrichment as evidence.

Three tiers are defined:

| Tier | Trust Weight | Definition |
|---|---|---|
| **Tier A** | 1.0 | Vetted providers with demonstrated accuracy in the relevant domain. Designation requires a documented evaluation, not automatic assignment. |
| **Tier B** | 0.70 | General-purpose providers with good track records in related domains. Default tier for known commercial enrichment providers at onboarding. |
| **Tier C** | 0.40 | Unvetted, new, or experimental providers. Enrichment data from Tier C providers is treated as provisional and does not anchor confident classifications. |

The enrichment trust tier is stored on the `enrichment_providers` record and is referenced when an enrichment record is written. When the CCE incorporates enrichment data into a confidence score, it multiplies the enrichment's evidentiary weight by the provider's trust weight before summation.

Provider tier assignment follows the same override and audit trail pattern as source axis scores: tier changes are logged with rationale, and the system preserves the previous tier.

---

## Score Update Triggers

Source credibility scores are updated under the following conditions:

| Trigger | Effect |
|---|---|
| New signal ingested from this source | `last_signal_at` updated; recency decay recalculated; `recency_weight` updated |
| Signal from this source corroborated by independent high-confidence evidence | `reliability` nudged up by +0.02 (system-computed column) |
| Signal from this source contradicted by independent high-confidence evidence | `reliability` nudged down by −0.04 (system-computed column; contradictions weighted more than corroborations to reflect asymmetry of error) |
| Analyst adds or revises domain coverage tags on source | `domain_relevance` recalculated from tag-to-taxonomy alignment |
| Analyst manually overrides any axis | Manual value recorded; system value preserved; audit entry written |
| Daily recency decay job | Recency weight recalculated for all sources based on `last_signal_at` |

Score update events do not immediately propagate to the CCE. The CCE reads `sources.credibility_score` at classification time. The composite score is recomputed and written to the `credibility_score` column as part of the score update job (daily batch, or on-demand when triggered by a contradiction or manual override event).

---

## Retroactive Impact on Active Classifications

When a source's `credibility_score` changes by more than the trigger threshold (see DECISION NEEDED item below), the system must flag active `match_sets` in the CCE that used signals from that source for analyst review.

This is implemented as a **background job**, not a synchronous re-run. The job:

1. Queries for all `match_set_signals` records referencing signals from the affected source.
2. Identifies the parent `match_sets` that are in an active or accepted state.
3. Writes a review flag on each affected `match_set` with a note indicating the source credibility change event and the delta.
4. Does not automatically change the match_set classification or confidence score — that requires analyst review and explicit re-run.

This design is intentional: automatic silent re-scoring of accepted classifications without analyst awareness would undermine trust in the platform.

---

## Sensitive Source Identity

Sources that are themselves sensitive (non-public feeds, restricted-access intelligence sources, partner-provided data under NDA) present an identity masking problem: the source must exist in the database to support credibility scoring and signal attribution, but its name and URL must not be visible in the general sources table.

The pointer pattern from ADR-012 (Sensitive Content Store) applies to source identity when the source itself is sensitive:

- The `sources` table record contains only a non-identifying reference ID and a sensitivity flag.
- The source name, URL, access credentials, and provenance metadata are stored in the Sensitive Content Store.
- Credibility scoring operates on the reference ID; the axis scores and composite score are stored in the main `sources` table (non-sensitive).
- Analysts with appropriate access level can resolve the reference to the full source identity via the Sensitive Content Store lookup.

> **DECISION NEEDED —** Should source identity masking (the pointer pattern from ADR-012) be applied to all sensitive/non-public sources from initial deployment, or only implemented when a specific sensitive source is onboarded? *Recommendation: implement the structural support (sensitivity flag + pointer column on `sources`) from day one, but defer population of the Sensitive Content Store to the first actual onboarding of a sensitive source. The schema should be ready; the operational workflow can wait.*

---

## DECISION NEEDED Items

> **DECISION NEEDED —** Initial default scores by source type are rough estimates based on general reasoning, not empirical calibration against Tech Range's actual signal corpus. Who reviews and calibrates these defaults after the first 90 days of operation, and what constitutes a calibration record? *Recommendation: senior analyst review at the 90-day mark, comparing default-initialized scores against track-record-derived scores for sources that have accumulated at least 20 corroboration/contradiction events. Results documented in a calibration record attached to this ADR as an appendix. Review should produce updated default values for each source type.*

> **DECISION NEEDED —** What delta in `credibility_score` (composite) triggers the background re-run job that flags affected `match_sets` for review? Too low a threshold generates noise; too high a threshold misses meaningful degradation in source reliability. *Recommendation: delta > 0.20 on the composite score. Additionally, any manual override that moves a source's composite score by more than 0.10 in a single edit should trigger the job regardless of the absolute delta threshold, since manual overrides reflect deliberate analyst judgment about source quality.*

---

## Consequences

### Positive

- The trust hierarchy is transparent: any analyst can inspect the three axis scores, see when they were last updated, and understand why the composite is what it is.
- Score changes are challengeable: analysts can override any axis with a rationale, and the system preserves the divergence between analyst judgment and system-computed values.
- Source degradation propagates to downstream classifications: when a source is found to be unreliable, active match_sets using its signals are flagged, preventing silent persistence of tainted classifications.
- Enrichment quality is tracked separately from source quality, preventing enrichment from a low-trust provider from laundering into high-confidence classifications.

### Negative

- Maintaining per-source corroboration and contradiction track records adds operational overhead. The ingestion pipeline (TS-03) must emit corroboration and contradiction events, and those events must be reliably attributed to their originating sources.
- The retroactive match_set flagging job is compute-intensive at scale. If a high-volume source experiences a large credibility change, the number of affected match_sets could be significant. Job must be designed with batching and rate limiting from the start.
- Multi-dimensional scoring increases the surface area for analyst confusion if the three axes and their weights are not clearly surfaced in the UI. The analyst-facing source detail view must display all three axes, not just the composite.
- Initial default scores are opinionated estimates. Until sufficient track record accumulates, the system is operating on priors, not evidence.

---

## References

- TS-02: Data Model — `sources` table schema, `enrichment` table schema
- TS-03: Ingestion Pipeline — defers to this ADR for credibility methodology
- TS-04: CCE — source credibility factor (weight 0.10) in Neighborhood classification scoring
- ADR-012: Sensitive Content Store — pointer pattern for sensitive data
