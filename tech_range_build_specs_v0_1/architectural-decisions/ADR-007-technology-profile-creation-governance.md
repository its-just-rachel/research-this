# ADR-007: Technology Profile Creation and Governance

**Status:** Accepted
**Date:** 2026-07-06
**Deciders:** Tech Range Platform Team

---

## Context

Technology Profiles are the primary organizational unit of the Tech Range platform — durable knowledge objects that aggregate signals, carry lifecycle state, attract analyst attention, and anchor radar placement. Because of this centrality, profile creation is not a neutral act. Each profile that exists has downstream costs:

- It competes for analyst attention during triage and evaluation
- It occupies a position in the radar taxonomy, whether or not that position is well-earned
- It generates downstream evaluations, relationship edges, and cluster associations that accumulate over time
- It creates the expectation of maintenance — a profile that goes stale is worse than no profile

Ungoverned profile creation produces **profile sprawl**: a long tail of weak, underspecified, or premature profiles that dilute attention away from the signals that actually matter. At the same time, governance that is too restrictive produces **late recognition**: technologies that are moving fast accumulate signal without getting a profile home, delaying the platform's ability to surface them to analysts.

The platform currently supports two distinct creation pathways: **emergent recognition**, where the system detects a signal cluster with sufficient density and proposes a profile; and **intentional injection**, where an analyst creates a profile directly for a technology they believe the platform should track. These pathways have different triggering conditions, different evidence bases, and different risk profiles. They require different governance.

Profiles are also not static once created. They evolve through a defined Knowledge State machine:

```
Candidate → Emerging → Active → Under Review → Archived
```

Governance must cover the full lifecycle — not just creation, but also merging, splitting, archiving, and revival.

---

## Decision

Tech Range will implement a **two-track creation model** for Technology Profiles, with distinct governance rules for each track, plus lifecycle governance covering merge, split, archive, and revival operations.

### Track 1: Emergent Recognition (System-Suggested)

The system monitors signal clusters continuously. When a cluster crosses defined thresholds, the system generates a **Profile Suggestion** and routes it to the analyst queue for review. A profile is not created until an analyst approves the suggestion. Suggestions that are not acted upon within the review window are auto-archived.

### Track 2: Intentional Injection (Analyst-Initiated)

An analyst with appropriate authorization initiates a profile directly, providing a minimum evidence package. The draft profile enters a peer review step before transitioning out of Candidate state. This track does not require a signal cluster to exist first.

Both tracks produce a profile in **Candidate** state. Candidate profiles have limited visibility and do not appear on the public radar until they transition to Emerging or Active.

---

## Rationale

### Why not fully automated profile creation?

Automated creation without review produces noise at scale. The signal ingestion pipeline surfaces hundreds of candidate signals across dozens of domains. A cluster crossing a size threshold is a necessary but not sufficient condition for a meaningful Technology Profile — the cluster may represent a transient news cycle, a vendor marketing push, a rebranding of an existing tracked technology, or a topic that is genuinely outside the platform's current classification scope. No threshold combination eliminates these false positives reliably. An analyst approval gate costs very little when the suggestion is good, and prevents real harm when it is not.

### Why not fully manual profile creation?

Requiring an analyst to manually initiate every profile creates a bottleneck that the platform is explicitly designed to avoid. The volume problem is real: at platform scale, the number of emerging signals that warrant a profile exceeds what any analyst team can monitor directly. Emergent recognition is the platform's core value proposition — the system should surface what the analysts would not have thought to look for. A fully manual model collapses that advantage.

### Why a two-track model rather than a single unified track?

The two tracks exist because the evidence base and the initiating condition are fundamentally different. Emergent track profiles are hypothesis-driven: the system has observed something and is asking "should this be a profile?" Intentional track profiles are knowledge-driven: an analyst knows something is real and is asking "why isn't this tracked yet?" Collapsing them into a single flow would either under-govern intentional injections (which can bypass the cluster requirement) or over-govern emergent suggestions (which already carry system-generated evidence). Keeping them separate allows each track to be tuned independently.

### Anti-criteria

Profiles should NOT be created — under either track — for:

- **Transient signals without a cluster:** a single high-volume signal spike that has not produced a persistent cluster
- **Duplicate technologies:** a technology already tracked under an existing Active or Emerging profile, even if the terminology differs
- **Out-of-scope technologies:** technologies outside the platform's current classification taxonomy that would require taxonomy extension to place correctly (taxonomy extension is a separate governance process)
- **Pre-competitive noise:** vendor-generated activity, conference keynote mentions, or analyst report citations that have not yet produced independent signal accumulation

---

## Emergent Track: Detail

### Trigger Conditions

A Profile Suggestion is generated when a signal cluster satisfies all of the following conditions simultaneously:

> **DECISION NEEDED —** What is the minimum cluster signal count required to trigger a suggestion? *Recommendation: Start with a threshold in the range of 15–25 distinct signals from at least 3 independent sources, calibrated against known false-positive clusters in the ingestion backlog.*

> **DECISION NEEDED —** What is the minimum coherence score required? The coherence score measures intra-cluster semantic similarity and cross-signal source diversity. *Recommendation: Require a coherence score above a mid-range threshold (e.g., 0.65 on a 0–1 scale) to filter keyword-collision clusters from genuinely coherent topic clusters.*

- **Neighborhood assignment confidence:** The cluster must be assignable to a taxonomy neighborhood with sufficient confidence, meaning the system can propose a classification location with high confidence. Clusters that cannot be placed are surfaced as taxonomy anomalies rather than profile suggestions.

### What the Suggestion Contains

A system-generated Profile Suggestion includes:

- Proposed profile name and short description (system-generated, editable)
- Proposed taxonomy placement (neighborhood + tier)
- Cluster summary: signal count, source diversity, time window, coherence score
- Top signals by weight (representative examples, not exhaustive)
- Potential duplicates: existing profiles with high name or cluster overlap, flagged for analyst review
- Anti-criteria check: automated scan for known anti-criteria patterns (e.g., vendor-source concentration, short signal window)

### Approval Flow

1. Suggestion enters the **Analyst Review Queue** with status `Pending Approval`
2. Any authorized analyst can claim the suggestion for review
3. Reviewer assesses the suggestion, optionally edits the proposed name, description, and placement
4. Reviewer approves (profile created in Candidate state) or rejects (suggestion archived with reason code)
5. Approved profiles in Candidate state are visible to the reviewer's team and begin accumulating additional signal before transitioning to Emerging

### Review Window and Expiry

> **DECISION NEEDED —** How long should a system-generated suggestion remain open before expiry? *Recommendation: A 14-day review window balances urgency (fast-moving signals decay in relevance) against analyst availability. Auto-archive at expiry rather than auto-escalate, to avoid queue bloat; allow re-suggestion if the cluster re-qualifies.*

If no analyst acts within the review window, the suggestion is **auto-archived** with reason code `EXPIRED_NO_ACTION`. The underlying cluster remains active and will re-trigger a suggestion if it continues to grow and re-crosses the thresholds. This prevents suggestions from accumulating indefinitely and forces organic re-evaluation.

---

## Intentional Track: Detail

### Who Can Initiate

Intentional injection is available to analysts with **profile creation authorization**. This is a role-level permission, not a per-profile approval. The intent is to allow experienced analysts to act on knowledge they hold that the signal pipeline has not yet surfaced, while preventing casual or exploratory profile creation by users without domain context.

> **DECISION NEEDED —** What role level should hold profile creation authorization? *Recommendation: Grant to Senior Analyst level and above by default; allow team leads to grant it to specific analysts on a per-domain basis.*

### Minimum Evidence Package

An intentional profile submission must include:

- Technology name and at least one alternative name or alias
- Short description (minimum 50 words) explaining what the technology is and why it warrants tracking
- At least two external source references (published, linkable) confirming the technology exists and is active
- Proposed taxonomy placement (neighborhood + tier)
- A brief statement of why the technology is not already covered by an existing profile

The system performs an automated duplicate check at submission time and surfaces any high-similarity existing profiles. The analyst must explicitly acknowledge or dismiss each flagged potential duplicate before the submission proceeds.

### Peer Review

Intentional track submissions enter a **peer review** step before transitioning out of Candidate state. The submitting analyst nominates a reviewer, or the system assigns one from the same domain team if none is nominated.

The peer reviewer checks:
- Evidence quality and source credibility
- Accuracy of the taxonomy placement
- Duplicate check acknowledgement was not dismissed inappropriately
- Anti-criteria compliance

The peer reviewer approves or requests revision. Revisions re-enter the review queue. There is no automatic expiry on intentional track submissions — they remain in review until resolved.

### How Intentional Differs from Emergent

| Dimension | Emergent Track | Intentional Track |
|---|---|---|
| Initiating condition | System-detected cluster crossing thresholds | Analyst knowledge |
| Evidence at creation | System-generated cluster summary | Analyst-provided source package |
| Requires existing cluster | Yes | No |
| Review type | Analyst approval of system suggestion | Peer review of analyst submission |
| Expiry | Auto-archives if no action in review window | No automatic expiry |
| Starting state | Candidate | Candidate |

---

## Profile Lifecycle Governance

### Merge

Two profiles should be merged when they are determined to represent the same technology under different names or framings. Merge candidates are surfaced by:

- High cross-profile signal overlap detected by the system
- Analyst flagging via the "potential duplicate" report mechanism
- Taxonomy review identifying co-located profiles with overlapping scope

Merge requires:

- Identification of a primary profile (the one that survives) and a secondary profile (the one absorbed)
- Redirect of all signals, evaluations, and cluster associations from secondary to primary
- A merge rationale note attached to the primary profile's history
- Notification to any analysts who have the secondary profile in active watch lists

> **DECISION NEEDED —** Who has authority to approve a merge? *Recommendation: Require approval from the team lead responsible for the taxonomy neighborhood where both profiles reside. For cross-neighborhood merges, require sign-off from both leads.*

### Split

A profile should be split when it is determined to have been too broad — tracking what are actually two distinct technologies under a single profile. Split candidates are identified by:

- Analyst observation that signals in the profile fall into two incoherent groups
- Cluster analysis showing bimodal signal distribution within the profile
- Taxonomy review identifying a profile that spans two distinct neighborhoods

Split requires:

- Definition of two new profiles (or one new + the existing profile scoped down)
- Re-assignment of all signals to the appropriate child profile
- A split rationale note on both resulting profiles
- Retention of the original profile's history on the primary child profile

> **DECISION NEEDED —** Who has authority to approve a split? *Recommendation: Same authority model as merge — neighborhood team lead approval, with cross-neighborhood requiring both leads.*

### Archive

A profile transitions to **Archived** state when the technology it tracks is no longer active or relevant to the platform's tracking mission. Archive triggers include:

- No new signals in the profile for an extended period
- Technology confirmed discontinued, acquired and absorbed, or superseded
- Analyst-initiated archive request with rationale
- Taxonomy restructuring that relocates the technology's scope to another profile

> **DECISION NEEDED —** What is the signal inactivity period that triggers an automatic Under Review → archive recommendation? *Recommendation: Flag profiles with zero new signals for 90 days as candidates for archiving; require analyst confirmation before archiving, rather than auto-archiving, to avoid silently losing profiles for technologies in genuine quiet periods.*

Archived profiles are not deleted. They retain their full signal and evaluation history, remain searchable, and can be revived.

### Revival (Archived → Active)

An archived profile can be revived if the technology resurfaces. Revival requires:

- A new signal cluster forming around the technology, or an analyst-initiated revival request
- Minimum evidence package equivalent to the intentional track submission (to confirm the technology is genuinely re-emerging, not a one-time mention)
- Peer review of the revival rationale
- Profile re-enters at **Candidate** state and follows the standard progression

Revival is not automatic. The archive state should not be treated as a soft pause — revival requires active justification.

---

## Consequences

**Positive:**

- Profile creation is governed end-to-end, reducing sprawl and attention dilution
- The two-track model preserves emergent discovery (the platform's core value) while maintaining quality control
- Lifecycle governance (merge, split, archive, revival) prevents the profile graph from accumulating structural debt over time
- System-generated suggestions carry evidence packages that reduce analyst review burden
- Anti-criteria are explicit and checkable, reducing subjective gatekeeping

**Negative / Trade-offs:**

- The approval gate on emergent suggestions introduces latency between signal detection and profile creation; fast-moving technologies may be under-tracked during the gap
- Peer review on intentional track adds friction for analysts who have high-confidence knowledge; this friction is intentional but may generate resistance
- Merge and split operations are operationally complex and require tooling support to re-associate signals correctly; this is a build dependency
- Review window expiry (auto-archive on inaction) means suggestions can be lost if the analyst queue is backlogged; queue health monitoring is a dependency

**Dependencies:**

- Cluster coherence scoring must be implemented and calibrated before the emergent track threshold system can be activated
- Profile suggestion queue UI must be built before emergent track governance can operate in practice
- Signal re-association logic (for merge/split) must be implemented before those operations can be performed safely
- Role-level authorization for profile creation (intentional track) must be integrated with the platform's identity and permissions system

---

## Open Decisions Summary

The following items require internal resolution before the governance model can be fully operationalized:

> **DECISION NEEDED —** Minimum cluster signal count threshold for system suggestion generation. *Recommendation: 15–25 distinct signals from at least 3 independent sources.*

> **DECISION NEEDED —** Minimum coherence score threshold for system suggestion generation. *Recommendation: ~0.65 on a normalized 0–1 scale, calibrated against backlog false positives.*

> **DECISION NEEDED —** Review window duration for emergent track suggestions before auto-archive. *Recommendation: 14 days.*

> **DECISION NEEDED —** Role level required for intentional track profile creation authorization. *Recommendation: Senior Analyst and above by default; team lead override for specific analysts.*

> **DECISION NEEDED —** Merge and split approval authority — who can authorize. *Recommendation: Taxonomy neighborhood team lead; cross-neighborhood requires both leads.*

> **DECISION NEEDED —** Signal inactivity period that triggers an archive recommendation. *Recommendation: 90 days of zero new signals flags the profile for analyst-confirmed archiving.*
