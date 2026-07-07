# Tech Range Build Specification Pack

**Version:** v0.1  
**Created:** 2026-07-06  
**Status:** Build-planning draft

## Purpose

This pack translates the Tech Range ground-truth model into the next set of design specs, technical specs, and architectural decisions needed before implementation.

The ground-truth docs define what Tech Range is. This build pack defines what must be specified, designed, decided, and validated to build it.

## Inputs

This pack assumes the current Tech Range ground-truth model:

- Surface -> Research -> Solution -> Prototype -> Advise
- Evidence -> Knowledge -> Action
- Signal-first, knowledge-centered architecture
- Technology Profiles as durable knowledge objects
- Solution Candidates as engagement paths
- Comparative classification with primary and secondary matches
- Fidelity x Availability as external technology-state logic
- Advisories as primary strategic outputs
- World Models, Wargaming, and Red Teaming as extensions unless explicitly promoted

## Directory map

```text
tech_range_build_specs_v0_1/
  README.md
  SPEC-REGISTRY.md
  BUILD-DECISION-SEQUENCE.md
  design-specs/
  technical-specs/
  architectural-decisions/
```

## Recommended use

1. Review `SPEC-REGISTRY.md` first.
2. Use `BUILD-DECISION-SEQUENCE.md` to decide what must be resolved before engineering starts.
3. Complete architectural decisions before finalizing technical specs.
4. Keep sensitive internal details in enterprise-controlled copies only.
5. Treat these as build specs, not replacements for ground-truth doctrine.

## Safety note

Do not add sensitive internal strategy, partner details, client context, restricted examples, controlled technical data, or non-public thresholds in public-safe versions. Use placeholders such as:

- `[Internal decision rule]`
- `[Sensitive source]`
- `[Restricted access path]`
- `[Client mission context]`
- `[Internal threshold]`
