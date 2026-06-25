*Updated: ops#7 closed — removed from backlog.*

# Handoff — casehub-desiredstate

## Last Session

Cleared the S/XS backlog: #34 (already fixed — closed), #44 (PLATFORM.md cross-repo update), #26 (PendingApproval on both ProvisionResult and DeprovisionResult), #36 (tenancyId through ActualStateAdapter and TransitionExecutor SPIs), #39 (pipeline wired to CaseTransitionExecutor). Also closed #40 — the Worker extraction epic is fully complete (all 7 cross-repo steps done; openclaw#37 and ops#8 were already resolved).

## Immediate Next Step

Resume #27 (managed pipeline mode). Run `/work` — the branch `issue-27-managed-pipeline-mode` exists in both repos but has stale #41 commits. Start #27's worker-api integration fresh from main.

## Cross-Module

*No active cross-module dependencies.*

## What's Left

- PLATFORM.md desiredstate row committed to casehub-parent on `issue-293-channel-taxonomy` branch (893e5c65) — needs pushing when that branch closes · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Branch exists; unblocked by #41 |
| #43 | SimpleTransitionExecutor — create WorkItem for requiresHuman nodes | S | Med | Needs casehub-work dep |
| #23 | CBR integration for desired-state evolution | M | High | Needs casehub-neural-text |
| #24 | State-vector abstraction for QuarkMind | L | High | Different graph model needed |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- ARC42STORIES.MD: `ARC42STORIES.MD` (on main)
- Blog: `2026-06-25-mdp02-clearing-the-backlog.md`
