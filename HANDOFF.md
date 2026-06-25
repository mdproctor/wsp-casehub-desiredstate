# Handoff — casehub-desiredstate

## Last Session

Closed #38 (TransitionPlanner DRIFTED re-provisioning). Restructured `plan()` with exhaustive switch expressions on `NodeStatus` — DRIFTED nodes in desired are now re-provisioned, DRIFTED orphans deprovisioned. Adding a fifth `NodeStatus` fails to compile. Also created ARC42STORIES.MD (#42) on the #27 branch. #27 remains paused (blocked by Worker Foundation Extraction #40).

## Immediate Next Step

Pick the next issue. Top candidates: #27 (managed pipeline, blocked by #40 extraction chain) or casehub-ops#7 (now unblocked by #38 — drift self-healing works). Engine#543 (worker migration) is the critical path for unblocking #27.

## Cross-Module

**We're blocking** (these repos need the extraction to land):
- `casehub-engine` — engine#543 depends on parent#288 (BOM, now closed) · L · Med
- `casehub-desiredstate` — #41 depends on engine#543 · S · Low

## What's Left

- #40 cross-repo execution: engine#543 → #41 + claudony#157 + workers#14 + openclaw#37 + ops#8 · XL · Med
- PLATFORM.md desiredstate row needs updating to mention ExecutionBackend · XS · Low
- ARC42STORIES.MD (#42) is on the paused #27 branch, not yet on main · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused; blocked by #40 |
| ops#7 | Deployment topology provisioning (drift self-healing) | M | Med | Unblocked by #38 |
| #23 | CBR integration for desired-state evolution | M | High | Needs casehub-neural-text |
| #24 | State-vector abstraction for QuarkMind | L | High | Different graph model needed |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Spec (#38): `docs/superpowers/specs/2026-06-23-drifted-reprovision-design.md`
- Blog: `blog/2026-06-25-mdp01-the-one-line-fix-that-wasnt.md` (workspace)
- Extraction spec: `docs/superpowers/specs/2026-06-18-worker-foundation-extraction-design.md`
