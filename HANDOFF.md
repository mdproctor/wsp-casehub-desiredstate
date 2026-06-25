# Handoff — casehub-desiredstate

## Last Session

Closed #42 (ARC42STORIES.MD on main) and #41 (casehub-worker-api integration). Build is green again — engine-adapter now depends on casehub-worker-api for Worker/Capability types, with FlowWorkerFunction bridge from engine-flow. Discovered during resume that engine#543 and parent#288 had both closed, unblocking #41. Garden entry GE-20260625-d09c57 captures the three-module dependency split gotcha.

## Immediate Next Step

Resume #27 (managed pipeline mode) — now unblocked. Run `/work` to resume from the pause stack. The branch has stale #41 commits that should be dropped (start #27's worker-api integration fresh from main).

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- #40 cross-repo execution: steps 4-7 remain (claudony#157 + workers#14 + openclaw#37 + ops#8) · L · Med
- PLATFORM.md desiredstate row needs updating to mention ExecutionBackend · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused; unblocked by #41 |
| ops#7 | Deployment topology provisioning (drift self-healing) | M | Med | Unblocked by #38 |
| #23 | CBR integration for desired-state evolution | M | High | Needs casehub-neural-text |
| #24 | State-vector abstraction for QuarkMind | L | High | Different graph model needed |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- ARC42STORIES.MD: `ARC42STORIES.MD` (on main)
- Garden: `GE-20260625-d09c57` — Worker migration three-module split gotcha
- Extraction spec: `docs/superpowers/specs/2026-06-18-worker-foundation-extraction-design.md`
