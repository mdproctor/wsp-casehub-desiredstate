# Handoff — casehub-desiredstate

*Updated: #60 closed — removed from backlog.*

## Last Session

Designed, reviewed, and implemented #46 (PendingApproval gate), #58 (lifecycle
transitions), #56 (planner-backed GoalCompiler). GoalCompiler SPI returns
`CompilationResult` (sealed: `SingleGraph` | `Lifecycle`). New `LifecycleManager`
in runtime orchestrates CAS-based phase transitions via `ReconciliationListener`.
Pipeline example gains per-stage approval gate. New `examples/expansion/` module
with HTN-backed GoalCompiler, build→defend lifecycle, SituationRecompiler-driven
replanning. Design-reviewed (4 rounds, $15.73). 9 implementation tasks via SDD.
Landed as `66d2113` on main.

## Immediate Next Step

Pick next from What's Next table. #51 (ActualStateAdapter routing) continues the
multi-domain thread.

## Cross-Module

**We're blocking:**
- `casehub-engine-flow` — CaseTransitionExecutor depends on `CallableDispatchRegistry` SPI · S · Low
- `casehub-work` — WorkItem-backed handlers depend on `WorkItemCreator` SPI · S · Low
- `casehub-ops` — GoalCompiler/SituationRecompiler migration (4 implementations) · S · Low

**Blocked by:** nothing

## What's Left

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #51 | ActualStateAdapter routing for multi-domain | M | Med | Follow-on to #18 |
| #52 | Routing for remaining CDI-ambiguous SPIs | M | Med | GoalCompiler, EventSource, FaultPolicy |
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused on stack |
| #23 | CBR integration for desired-state evolution | M | High | Slots into RAS Ganglia now |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Spec: `docs/specs/2026-07-06-pending-approval-lifecycle-planner-design.md`
- Plan: `docs/plans/2026-07-06-pending-approval-lifecycle-planner.md`
- Design review: `~/adr/casehub-desiredstate/pending-approval-lifecycle-planner-20260706-172800/`
