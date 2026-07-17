# Handoff — casehub-desiredstate

## Last Session

Closed #72, #79, #80 — HumanGating per-action flags, work-adapter boundary,
case-lifecycle coordination. `boolean requiresHuman` replaced with `HumanGating`
enum (NONE, PROVISION_ONLY, DEPROVISION_ONLY, ALL). `WorkItemHumanNodeHandler`
removed — human nodes require CTE case-backed orchestration. CTE cancels
previous cases before starting new ones. GraphDiff latent bug fixed (UpdateNode
now carries full adapted DesiredNode). Pipeline example demonstrates Gold-tier
DEPROVISION_ONLY. Design-reviewed (4 rounds, 12 issues, all resolved). Landed
as `67af3d1` on main.

## Immediate Next Step

Pick next from What's Next table. #27 (managed pipeline mode) has a branch
`issue-27-managed-pipeline-mode` — resume with `/work` or start something new.

## Cross-Module

**Blocked by:**
- `casehub-engine-flow` — CaseTransitionExecutor needs `CallableDispatchRegistry` SPI · S · Low
- `casehub-work` — WorkItem-backed handlers need `WorkItemCreator` SPI · S · Low

**Pending cross-repo:**
- `casehub-ops` — DesiredNode constructor change (`boolean` → `HumanGating`); #82 filed · S · Low

## What's Left

- `neocortex#142` — Wire CbrOutcomeConsumer to platform CloudEvent routing · open
- `neocortex#141` — Map<String, Object> → Map<String, FeatureValue> migration · open
- `ops#52` — FaultPolicy/SituationRecompiler tenancyId migration · S · Low
- `#81` — WorkItemPendingApprovalHandler implementation in work-adapter · S · Med
- `#82` — casehub-ops DesiredNode constructor migration · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Branch exists |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |
| #81 | WorkItemPendingApprovalHandler in work-adapter | S | Med | Blocked on work#281/282 |
| #82 | casehub-ops DesiredNode constructor migration | S | Low | Filed from this branch |

## References

- Spec: `docs/specs/2026-07-17-humangating-work-adapter-design.md`
- Plan: `docs/plans/2026-07-17-humangating-work-adapter.md`
