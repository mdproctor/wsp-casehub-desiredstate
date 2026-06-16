# Handoff — casehub-desiredstate

## Last Session

Epic 2 complete — data pipeline example domain with medallion architecture, three-tier fault escalation, and 12 tests. Three runtime bugs fixed (DRIFTED detection, CAS race, deprovision fault typing). Domain pivot from K8s to data pipeline for AiFusion positioning. Spec went through 7 review iterations that exposed the runtime gaps.

## Immediate Next Step

Pick the next issue. Top candidates: casehub-ops domain implementations building on the now-proven runtime, or platform-level design issues (#22–#25, parent#233, parent#234). The data pipeline example is designed to evolve into a real runtime — issues #27-31 track that evolution (managed mode, toolbox, LLM/WorkItem integration, medallion enforcement).

## What's Left

- PLATFORM.md needs updating with pipeline example entry (parent#256 filed) · S · Low
- `issue-1-minimal-api-types` project branch to delete (orphaned, all work merged from `issue-1-generic-runtime`) · XS · Low
- Minor polish on pipeline example (#35 — layerOf null return, spec wording, EventSource emitter) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow orchestration per node with audit trail | M | Med | Evolves pipeline from example to runtime |
| #28 | Per-node execution toolbox — Camel, Drools, Stream backends | L | High | AiFusion showcase |
| #29 | Real LLM integration for AI_REVIEW fault nodes via agent-api | S | Med | Connects to casehub-platform-agent-api |
| #30 | Real WorkItem integration for HUMAN_REVIEW nodes via casehub-work | S | Med | Connects to casehub-work |
| #31 | Medallion layer enforcement as a planner constraint | S | Low | Currently emergent, could be enforced |
| #22 | Unified tracing across casehub-engine and quarkus-flow | S | Med | Stretch goal |
| #23 | CBR integration for desired-state evolution | M | High | Future — needs casehub-neural-text |
| #24 | State-vector abstraction for QuarkMind | L | High | Different graph model needed |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |
| parent#233 | Goal as first-class platform concept | XL | High | Invariant/achievement/composite goals |

## References

- Spec: `docs/superpowers/specs/2026-06-15-data-pipeline-example-design.md`
- Plan: `docs/superpowers/plans/2026-06-16-data-pipeline-plan.md`
- Research: `docs/superpowers/research/2026-06-07-desired-state-management-research.md`
- Blog: `blog/2026-06-16-mdp01-the-pipeline-that-broke-the-runtime.md` (workspace)
