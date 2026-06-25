# Handoff — casehub-desiredstate

## Last Session

Closed #42 (ARC42STORIES.MD) — extracted the two docs commits from the paused #27 branch and landed them on main independently. During resume cross-check, discovered that engine#543 and parent#288 are both CLOSED, meaning #41 (integrate casehub-worker-api) is now unblocked. Build is currently broken on main — engine-adapter can't find `Worker`/`Capability` types that were extracted to casehub-worker. #41 fixes this.

## Immediate Next Step

Start #41 — integrate casehub-worker-api into desiredstate. This fixes the broken build and unblocks #27 (managed pipeline mode). Start fresh from main, not from the stale #41 commit on the #27 branch — the worker-api dependency coordinates may have changed since engine#543 landed.

## Cross-Module

**We're blocking:**
- `casehub-desiredstate` — #27 depends on #41 · M · High

**No longer blocked by:**
- `casehub-engine` — engine#543 CLOSED (Worker migration done)
- `casehub-parent` — parent#288 CLOSED (BOM updated)

## What's Left

- #40 cross-repo execution: steps 3-7 remain (desiredstate#41 + claudony#157 + workers#14 + openclaw#37 + ops#8) · L · Med
- PLATFORM.md desiredstate row needs updating to mention ExecutionBackend · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #41 | Integrate casehub-worker-api into desiredstate | S | Low | Unblocked — fixes broken build |
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused; blocked by #41 |
| ops#7 | Deployment topology provisioning (drift self-healing) | M | Med | Unblocked by #38 |
| #23 | CBR integration for desired-state evolution | M | High | Needs casehub-neural-text |
| #24 | State-vector abstraction for QuarkMind | L | High | Different graph model needed |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- ARC42STORIES.MD: `ARC42STORIES.MD` (now on main)
- Extraction spec: `docs/superpowers/specs/2026-06-18-worker-foundation-extraction-design.md`
- Blog: `blog/2026-06-25-mdp01-the-one-line-fix-that-wasnt.md` (workspace)
