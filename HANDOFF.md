# Handoff — casehub-desiredstate

## Last Session

Closed #47 (CTE PendingApproval integration) and #48 (ARC42STORIES.MD placement). CTE's dispatch mechanism was non-functional — `desiredstate:dispatch` had no handler. Fixed via engine#590 (CallableDispatchRegistry — extensible workflow call dispatch). DesiredStateDispatch handles the full PendingApproval lifecycle within workflow steps. CTE pre-filters approval-gated nodes before case creation. Also placed PendingApproval as Chapter 8 in ARC42STORIES.MD with new L7 (Work Adapter) layer.

## Immediate Next Step

Ship cross-repo prerequisites: work#281 (WorkItemRef.payload) and work#282 (WorkItemCreator.obsoleteByCallerRef). Then implement WorkItemPendingApprovalHandler in work-adapter/.

## Cross-Module

**Blocked by:**
- `casehub-work` — work#281 + work#282 gate WorkItemPendingApprovalHandler · S · Low

## What's Left

- PLATFORM.md desiredstate row on casehub-parent `issue-293-channel-taxonomy` branch — needs pushing when that branch closes · XS · Low
- WorkItemPendingApprovalHandler in work-adapter/ — BLOCKED on work#281/282 · M · Med
- SituationSource types (#49) **moved to casehub-ras-api** — branch `issue-22-move-situation-types` pushed (b89d997). Deleted ActiveSituation, SituationSource, SituationChangeEvent from api/, DefaultSituationSource from runtime/. Added ras-api dependency.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #45 | PLATFORM.md — add PendingApprovalHandler to SPI list | XS | Low | Unblocked |
| #46 | Pipeline example — PendingApproval gate in three-tier escalation | S | Low | Unblocked |
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused on stack |
| #23 | CBR integration for desired-state evolution | M | High | Needs casehub-neural-text |
| #24 | State-vector abstraction for QuarkMind | L | High | Different graph model |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Spec: `docs/superpowers/specs/2026-06-29-cte-pending-approval-design.md`
- Plan: `docs/superpowers/plans/2026-06-30-cte-pending-approval.md`
- Garden: GE-20260629-45f4be (callerRef blocking), GE-20260629-db82b4 (reject reason audit-only)
