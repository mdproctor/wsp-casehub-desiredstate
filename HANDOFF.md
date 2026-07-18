# Handoff — casehub-desiredstate

## Last Session

Closed #81 and #82 — WorkItemPendingApprovalHandler in work-adapter,
casehub-ops migration already done. Stateless delegator bridging
PendingApprovalHandler SPI to WorkItemCreator. Design-reviewed (3 rounds,
15 issues, 12 verified). Key design: no self-cleaning on COMPLETED
(idempotent across retry cycles), callerRef `desiredstate-approval:<tenancyId>:<nodeId>:<action>`,
rejection cleanup via `obsoleteByCallerRef`. #82 closed without code changes —
all 5 ops GoalCompilers already migrated. Landed as `b42635f` on main.

## Immediate Next Step

Pick next from What's Next table. #27 (managed pipeline mode) has a branch
`issue-27-managed-pipeline-mode` on the pause stack — resume with `/work`.

## Cross-Module

**Blocked by:**
- `casehub-engine-flow` — CaseTransitionExecutor needs `CallableDispatchRegistry` SPI · S · Low
- `casehub-work` — WorkItem-backed handlers need `WorkItemCreator` SPI · S · Low

## What's Left

- `neocortex#142` — Wire CbrOutcomeConsumer to platform CloudEvent routing · open
- `neocortex#141` — Map<String, Object> → Map<String, FeatureValue> migration · open

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | On pause stack |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Spec: `docs/specs/2026-07-17-workitem-approval-handler-design.md`
- Plan: `docs/plans/2026-07-18-workitem-approval-handler.md`
- Blog: `blog/2026-07-18-mdp01-the-approval-handler-that-almost-cleaned-up.md`
