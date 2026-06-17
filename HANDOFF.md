# Handoff — casehub-desiredstate

## Last Session

Closed #35 (pipeline polish) and #31 (medallion layer enforcement) on a single branch. `layerOf()` now returns `Optional<PipelineLayer>` — fault nodes (AI_REVIEW, HUMAN_REVIEW) return empty. `MedallionLayerConstraint` validates no backward or layer-skipping dependencies; wired into `PipelineGoalCompiler.compile()`. Spec wording synced. 7 new tests (13→20 total in pipeline module).

## Immediate Next Step

Pick the next issue. Top candidates: pipeline evolution (#27–#30 for managed mode, toolbox, LLM/WorkItem integration) or platform design (#22–#25). The orphan `issue-1-minimal-api-types` branch is still pending deletion.

## What's Left

- PLATFORM.md needs updating with pipeline example entry (parent#256 filed) · S · Low
- `issue-1-minimal-api-types` project branch to delete (orphaned, all work merged from `issue-1-generic-runtime`) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow orchestration per node with audit trail | M | Med | Evolves pipeline from example to runtime |
| #28 | Per-node execution toolbox — Camel, Drools, Stream backends | L | High | AiFusion showcase |
| #29 | Real LLM integration for AI_REVIEW fault nodes via agent-api | S | Med | Connects to casehub-platform-agent-api |
| #30 | Real WorkItem integration for HUMAN_REVIEW nodes via casehub-work | S | Med | Connects to casehub-work |
| #22 | Unified tracing across casehub-engine and quarkus-flow | S | Med | Stretch goal |
| #23 | CBR integration for desired-state evolution | M | High | Future — needs casehub-neural-text |
| #24 | State-vector abstraction for QuarkMind | L | High | Different graph model needed |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Spec: `docs/superpowers/specs/2026-06-15-data-pipeline-example-design.md`
- Research: `docs/superpowers/research/2026-06-07-desired-state-management-research.md`
- Blog: `blog/2026-06-17-mdp01-enforcing-what-was-already-true.md` (workspace)
