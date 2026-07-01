# Handoff — casehub-desiredstate

## Last Session

Closed #18 (multi-provisioner dispatch), #19 (per-type reconciliation scheduling), #45 (PLATFORM.md SPI list), #49 (SituationSource — already done), #50 (Worker capability migration — already done). Made the runtime type-aware: `NodeProvisionerRouter` dispatches to provisioners by `NodeType`, interval-grouped timers replace the fixed 5-minute resync, CAS merge-and-retry fixes the mutation race (GE-20260616-3d2605). Preferences integration for per-NodeType resync overrides via `DurationPreference`. `ReactiveNodeProvisioner` deleted (zero implementations). Cross-repo: `DurationPreference` in casehub-platform-api, ops#33 provisioner migrations.

## Immediate Next Step

Ship cross-repo prerequisites: work#281 (WorkItemRef.payload) and work#282 (WorkItemCreator.obsoleteByCallerRef). Then implement WorkItemPendingApprovalHandler in work-adapter/.

## Cross-Module

**Blocked by:**
- `casehub-work` — work#281 + work#282 gate WorkItemPendingApprovalHandler · S · Low

## What's Left

- PLATFORM.md desiredstate row on casehub-parent `issue-293-channel-taxonomy` branch — needs pushing when that branch closes · XS · Low
- WorkItemPendingApprovalHandler in work-adapter/ — BLOCKED on work#281/282 · M · Med
- SituationSource types — moved to casehub-ras-api on branch `issue-22-move-situation-types` (b89d997) · needs merging

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #51 | ActualStateAdapter routing for multi-domain | M | Med | Natural follow-on to #18 |
| #52 | Routing for remaining CDI-ambiguous SPIs | M | Med | GoalCompiler, EventSource, FaultPolicy |
| #46 | Pipeline example — PendingApproval gate | S | Low | Unblocked |
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused on stack |
| #23 | CBR integration for desired-state evolution | M | High | Needs casehub-neural-text |
| #24 | State-vector abstraction for QuarkMind | L | High | Different graph model |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Spec: `docs/superpowers/specs/2026-07-01-type-aware-runtime-design.md`
- Plan: `docs/superpowers/plans/2026-07-01-type-aware-runtime.md`
- Garden: GE-20260701-82909e (ScheduledFuture map replacement race)
