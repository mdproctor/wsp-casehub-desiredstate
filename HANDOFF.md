# Handoff — casehub-desiredstate

*Updated: parent#271 closed — removed from backlog.*

## Last Session

Closed #28 (per-node execution toolbox). Refactored `PipelineProvisioner` from a monolithic if-chain to hybrid dispatch: metadata/review handled directly, processing stages delegated to pluggable `ExecutionBackend` implementations. `DefaultExecutionBackend` extracts existing logic. 32 tests (23 existing + 9 new). Brainstorming surfaced Worker data coordination patterns (DataExchange, DataChannel) — filed as engine#528 (WorkerFunction pluggability) and engine#532 (Worker data patterns).

## Immediate Next Step

Pick the next issue. Top candidates: pipeline evolution (#27 for managed mode) or platform design (#23–#25). engine#528 and engine#532 are prerequisites for the Exchange/DataChannel vision.

## What's Left

- PLATFORM.md desiredstate row needs updating to mention ExecutionBackend · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow orchestration per node with audit trail | M | Med | Evolves pipeline from example to runtime |
| #23 | CBR integration for desired-state evolution | M | High | Needs casehub-neural-text |
| #24 | State-vector abstraction for QuarkMind | L | High | Different graph model needed |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |
| engine#528 | WorkerFunction pluggability — extract Flow to optional module | M | Med | Prerequisite for DataExchange/DataChannel |
| engine#532 | Worker data coordination patterns — DataExchange, DataChannel | L | High | Design post engine#528 |

## References

- Spec: `docs/superpowers/specs/2026-06-17-pipeline-runtime-design.md`
- Blog: `blog/2026-06-18-mdp01-the-toolbox-that-wasnt.md` (workspace)
