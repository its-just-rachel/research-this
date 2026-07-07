# ADR-010: Radar Visualization Model

## Status: Accepted

---

## Context

### The radar-as-master-model failure mode

Technology intelligence platforms that treat the radar as their primary data structure consistently fall into the same trap: quadrant IDs become foreign keys, quadrant moves become the primary reclassification event, and the data model ends up shaped around what a radar chart can express. The symptoms are predictable:

- Technologies that straddle two neighborhoods cannot be represented — so they get forced into one arbitrarily
- Reclassification becomes a visual act (drag the dot) rather than a substantive one (update Fidelity, Availability, or Neighborhood based on new evidence)
- Analysts learn to think in quadrants rather than in the underlying intelligence dimensions
- Leadership receives a radar that is authoritative by convention rather than by derivation — and it drifts from the actual data over time

Tech Range's canonical data model (TS-02) stores Technology Profiles with five classification dimensions: **Fidelity**, **Availability**, **Neighborhood**, **Domain Area**, and **Engagement Posture**. The radar is one way to visualize a slice of that model. It is not the model itself.

### The strategic value of a radar

Done correctly, a radar communicates posture at a glance to leadership audiences who do not have time to review individual profiles. It earns its place in the platform only if it faithfully reflects the underlying data. A radar that diverges from the data model — because it was edited directly, cached indefinitely, or driven by a separate data structure — actively misleads the people it is meant to serve.

### Three constraints the radar must not violate

1. **The radar is a read-only derived view.** It computes from `tech_profiles` on demand or on a short cache TTL. It is never stored as an independent data structure that can diverge from the profiles that generated it.

2. **Analysts must not classify technologies by placing them on the radar.** All classifications — Fidelity, Availability, Neighborhood, Domain Area, Engagement Posture — are set in the data model via the Change/Classification Event (CCE) workflow. The radar renders the result of those classifications; it does not produce them.

3. **The radar must not be the only way to see technology posture.** Portfolio views, domain filter views, and pipeline views (all defined in DS-07) provide alternative angles on the same underlying data. No view is privileged. If the radar is unavailable or unsuitable for a given audience, posture can still be communicated.

---

## Decision

The radar is a **derived read-only view** computed from Technology Profile data. Specifically:

- Reads from `tech_profiles` WHERE `current_state = 'active'`
- Positions profiles by **Fidelity** (ring) and **Availability** (ring or sub-position within ring)
- Groups profiles by **Neighborhood** (sector/segment): Agentic/Vertical AI, Edge/AI-RAN, Physical AI, Quantum, Software-Defined Everything (SDx), Cloud Platforms, and Emerging (holding area)
- Color-codes or labels profiles by **Domain Area** or **Engagement Posture** depending on the view mode
- Is re-computed on demand or on a short cache TTL — **never stored as a separate data structure**
- **Cannot be edited directly** — any change to a profile's position on the radar requires editing the profile's Fidelity, Availability, or classification via the CCE, not dragging a dot on the radar chart

---

## What the radar shows

| Element | Source |
|---|---|
| Active Technology Profiles | `tech_profiles` WHERE `current_state = 'active'` |
| Position (ring) | Fidelity × Availability (0–5 scales) |
| Sector grouping | Neighborhood classification |
| Engagement Posture indicator | Posture field: Monitor / Evaluate / Prototype / Acquire / Deploy |
| Emerging signals cluster | `emerging_records` WHERE `resolved = false` (rendered as distinct zone — see DECISION NEEDED below) |
| Recent movement indicator | Profiles whose Fidelity or Availability changed within the last N days (visual delta marker) |

---

## What the radar cannot show (and why)

| Excluded element | Reason |
|---|---|
| Signals, Corroboration Groups, or Clusters | Too granular; the radar's audience is leadership, not analysts working raw signals |
| Confidence scores or `diff_strength` values | Analyst-layer tools; inappropriate for a leadership briefing surface |
| Sensitive content or sensitive Domain Area applicability | The radar may be shared in briefings; sensitive classifications must not leak into an uncontrolled view |
| Profiles in `candidate`, `emerging`, `under_review`, or `archived` states | The radar shows the actionable landscape, not the intake pipeline or historical record |
| Sub-integer precision on Fidelity or Availability | The radar is inherently coarse-grained; the 0–5 scales define its resolution ceiling |

---

## Radar vs. portfolio views

The radar is one of several views defined in DS-07. Others — the portfolio matrix, domain filter view, and pipeline view — render the same underlying `tech_profiles` data from different angles for different audiences. No view is privileged over another in the data model. The data model supports all of them equally; none of the views imposes constraints on what the data model can store.

This means a technology that is difficult to represent on a radar (e.g., one that spans two Neighborhoods, or one with a nuanced Engagement Posture that doesn't map cleanly to a ring) is still fully represented in the model and accessible through other views.

---

## Consequences

### Positive

- The radar accurately reflects the intelligence model at all times — it cannot drift because it has no independent state
- The radar can be regenerated at any time from current profile data; there is no "stale radar" problem beyond the cache TTL
- Leadership receives a view that is trustworthy by construction, not by convention
- Analysts are not tempted to use the radar as a classification tool, which would corrupt the underlying model

### Negative

- The radar cannot drive workflow (intentional constraint): users who want to reclassify a technology cannot do so by interacting with the radar; they must navigate to the profile
- Some UX patterns users may expect from radar-style tools (drag to reclassify, right-click to promote) are explicitly not supported — this requires clear UX communication in DS-07
- Cache TTL introduces a window during which the radar may not reflect the most recent profile changes — acceptable if TTL is appropriately short and a manual refresh is available

---

## DECISION NEEDED

> **DECISION NEEDED —** What is the acceptable cache TTL for radar computation — i.e., how stale can the radar be before it risks misleading leadership? *Recommendation: 4-hour TTL with a manual "Refresh radar" button available to analysts and administrators; the radar must never be more than one business day stale under any circumstances.*

> **DECISION NEEDED —** Should Emerging cluster candidates be rendered as a distinct visual zone on the radar (low opacity, clearly labeled "Unconfirmed"), or omitted from the radar entirely until they reach `active` state? *Recommendation: render as a distinct "Emerging" sector with low opacity and an explicit "Unconfirmed signals" label, so leadership has visibility into the intake horizon without confusing pipeline items with active profiles.*
