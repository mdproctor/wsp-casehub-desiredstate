# Handoff — casehub-desiredstate

## Last Session

Implemented #43 — `HumanNodeHandler` SPI in api/, `NoOpHumanNodeHandler` @DefaultBean in runtime/, `SimpleTransitionExecutor` delegates `requiresHuman` to handler. New `work-adapter/` module with `WorkItemHumanNodeHandler` creates WorkItems via `WorkItemCreator` SPI (casehub-work-api). Design went through two review rounds — caught a tier violation (work-adapter must depend on API only, not runtime) and an idempotency gap (`findActiveByCallerRef` not `findByCallerRef`). Both fixed. Upstream prerequisite (work#275) was already implemented in casehub-work.

## Immediate Next Step

Resume #27 (managed pipeline mode). Run `/work` — branch `issue-27-managed-pipeline-mode` is on the pause stack.

## Cross-Module

*No active cross-module dependencies.*

## What's Left

- PLATFORM.md desiredstate row committed to casehub-parent on `issue-293-channel-taxonomy` branch (893e5c65) — needs pushing when that branch closes · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused on stack; unblocked by #41 |
| #23 | CBR integration for desired-state evolution | M | High | Needs casehub-neural-text |
| #24 | State-vector abstraction for QuarkMind | L | High | Different graph model needed |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- ARC42STORIES.MD: `ARC42STORIES.MD` (on main)
- Blog: `2026-06-28-mdp01-the-spi-that-was-missing.md`
- Spec: `docs/superpowers/specs/2026-06-26-workitem-human-node-handler-design.md`
