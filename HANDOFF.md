# Handoff — casehub-desiredstate

## Last Session

Implemented #14 — PendingApprovalHandler SPI. New approval workflow for the ReconciliationLoop: when a provisioner returns PendingApproval, the handler creates a WorkItem, tracks approval status across cycles, and re-calls the provisioner with PlanApproval context on approval. Rejection fires APPROVAL_REJECTED through the fault pipeline via new StepOutcome.Rejected sealed variant. 6-round adversarial design review caught 28 issues (acknowledgeRejection lifecycle, faultFeedback guard, exhaustive terminal status handling, 3 cross-repo prerequisites). Work-adapter implementation (Task 5) is BLOCKED on work#281/282. Also slimmed pom.xml dependencyManagement (#319).

## Immediate Next Step

Ship cross-repo prerequisites: work#281 (WorkItemRef.payload field) and work#282 (WorkItemCreator.obsoleteByCallerRef). Then implement desiredstate Task 5 — WorkItemPendingApprovalHandler in work-adapter/.

## Cross-Module

**We're blocking** (other modules waiting on us):
- None

**Blocked by** (can't proceed until):
- `casehub-work` — work#281 (WorkItemRef.payload) + work#282 (obsoleteByCallerRef) gate Task 5 · S · Low

## What's Left

- PLATFORM.md desiredstate row committed to casehub-parent on `issue-293-channel-taxonomy` branch (893e5c65) — needs pushing when that branch closes · XS · Low
- Task 5: WorkItemPendingApprovalHandler in work-adapter/ — BLOCKED on work#281/282 · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #45 | PLATFORM.md update — add PendingApprovalHandler to SPI list | XS | Low | After #14 ships |
| #46 | Pipeline example — PendingApproval gate in three-tier escalation | S | Low | Depends on #14 |
| #47 | CaseTransitionExecutor PendingApproval integration | M | High | Different execution model |
| #48 | ARC42STORIES.MD placement for PendingApprovalHandler | XS | Low | |
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused on stack |
| #23 | CBR integration for desired-state evolution | M | High | Needs casehub-neural-text |
| #24 | State-vector abstraction for QuarkMind | L | High | Different graph model needed |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Spec: `docs/superpowers/specs/2026-06-28-pending-approval-handler-design.md`
- Plan: `docs/superpowers/plans/2026-06-29-pending-approval-handler.md`
- ADR workspace: `~/adr/casehub-desiredstate/pending-approval-handler-20260629-003709/`
- Garden: GE-20260629-45f4be (callerRef blocking), GE-20260629-db82b4 (reject reason audit-only)
