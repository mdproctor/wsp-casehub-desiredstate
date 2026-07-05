# Handoff — casehub-desiredstate

## Last Session

Designed and implemented #59 (aggregate fault detection via RAS integration). Epic #61.
FaultPolicy SPI gained ActualState (#62). ReconciliationLoop emits CloudEvents (#63).
New ras-adapter module with Ganglia, situation definitions (#64). SituationRecompiler SPI
+ desiredstate:replan dispatch (#68). Spatial POC proof (#67). End-to-end SituationDetectionTest
(#69). All cross-repo deps landed and consumed: ras#25 (Streak), ras#26 (Rate), ras#27
(SPI promotion), engine#652 (CaseDefinition labels). Design-reviewed (4 rounds, $15.96).
Code-reviewed (6 warnings fixed). Landed as `9b0eb46` on main.

## Immediate Next Step

Pick next from What's Next table. #58 (lifecycle transitions) is the natural follow-on —
prerequisite for #56 (planner-backed GoalCompiler).

## Cross-Module

**We're blocking:**
- `casehub-engine-flow` — CaseTransitionExecutor depends on `CallableDispatchRegistry` SPI · S · Low
- `casehub-work` — WorkItem-backed handlers depend on `WorkItemCreator` SPI · S · Low

**Blocked by:** nothing — work#281/282 resolved.

## What's Left

- work-adapter PendingApprovalHandler (#46) — unblocked now that work#281/282 closed · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #58 | POC: desired state lifecycle transitions | S | Med | Prerequisite for #56 |
| #56 | POC: planner-backed GoalCompiler | M | High | Highest architectural value |
| #60 | Assess runtime factoring cost for alternative state representations | M | Med | Informed by #59 findings |
| #51 | ActualStateAdapter routing for multi-domain | M | Med | Follow-on to #18 |
| #52 | Routing for remaining CDI-ambiguous SPIs | M | Med | GoalCompiler, EventSource, FaultPolicy |
| #46 | Pipeline example — PendingApproval gate | S | Low | Unblocked |
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused on stack |
| #23 | CBR integration for desired-state evolution | M | High | Slots into RAS Ganglia now |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Spec: `docs/specs/2026-07-04-constraint-evaluation-model-design.md`
- Plan: `docs/plans/2026-07-05-aggregate-fault-detection.md`
- Design review: `~/adr/casehub-desiredstate/constraint-evaluation-model-20260705-002208/`
