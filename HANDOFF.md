# Handoff — casehub-desiredstate

## Last Session

Implemented #57 (spatial/vector POC) — new `examples/spatial/` module with 10x10 terrain grid, fog of war, three evaluation scenarios (defense posture, attack waypoints, force distribution). 43 tests, all passing. Key finding: graph model handles spatial topology (layers 1-2) but cannot reason about aggregate outcomes or strategic alternatives (layers 3-4). Evidence for constraint/evaluation model documented as test assertions. Landed as `ead4861` on main. Design-reviewed (5 rounds, 21 issues, $20.80). Garden entry GE-20260703-b2073a (TransitionPlanner UnknownSpec gotcha).

## Immediate Next Step

#59 and #60 are conditional on #57's findings — now unblocked. #59 (constraint/evaluation model design) is the natural follow-on: design the SPI that addresses the three failure modes documented in ForceDistributionTest (fault policy information gap, planner/policy conflict, no aggregate evaluation). Run `/work` to begin.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #59 | Design constraint/evaluation model for aggregate subgraph reasoning | M | High | Unblocked by #57 findings — conditional deliverable |
| #60 | Assess runtime factoring cost for pluggable state representations | M | Med | Unblocked by #57 findings |
| #58 | POC: desired state lifecycle transitions | S | Med | Prerequisite for #56 |
| #56 | POC: planner-backed GoalCompiler | M | High | Highest architectural value |
| #51 | ActualStateAdapter routing for multi-domain | M | Med | Natural follow-on to #18 |
| #52 | Routing for remaining CDI-ambiguous SPIs | M | Med | GoalCompiler, EventSource, FaultPolicy |
| #46 | Pipeline example — PendingApproval gate | S | Low | Unblocked |
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused on stack |
| #23 | CBR integration for desired-state evolution | M | High | Needs casehub-neural-text |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Spec: `docs/superpowers/plans/2026-07-02-spatial-vector-poc.md` (project)
- Blog: `blog/2026-07-03-mdp01-where-the-graph-runs-out.md`
- Garden: GE-20260703-b2073a (TransitionPlanner UnknownSpec gotcha)
- Design review: `~/adr/casehub-desiredstate/spatial-vector-poc-20260702-150847/`
