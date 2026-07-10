# Handoff — casehub-desiredstate

## Last Session

Multi-domain SPI routing (#51, #52). ActualStateAdapterRouter dispatches
readActual() by NodeType with orphan detection. MergedEventSource composes
domain EventSource streams with per-stream error isolation. Scope analysis
confirmed FaultPolicy and GoalCompiler need no routing. DesiredStateGraph
gains filterByTypes() default method. Design-reviewed (3 rounds, $12.62).
Landed as `adfaf5f` on main.

## Immediate Next Step

Pick next from What's Next table. #27 (managed pipeline mode) is the
highest-value remaining work. Run `/work` to start a branch.

## Cross-Module

**We're blocking:**
- `casehub-engine-flow` — CaseTransitionExecutor depends on `CallableDispatchRegistry` SPI · S · Low
- `casehub-work` — WorkItem-backed handlers depend on `WorkItemCreator` SPI · S · Low
- `casehub-ops` — GoalCompiler/SituationRecompiler migration (4 implementations) · S · Low

**Blocked by:** nothing

## What's Left

Nothing trailing.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused on stack |
| #23 | CBR integration for desired-state evolution | M | High | Slots into RAS Ganglia now |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Spec: `docs/specs/2026-07-08-multi-domain-spi-routing-design.md`
- Plan: `docs/plans/2026-07-08-multi-domain-spi-routing.md`
- Design review: `~/adr/casehub-desiredstate/multi-domain-spi-routing-20260709-140534/`
