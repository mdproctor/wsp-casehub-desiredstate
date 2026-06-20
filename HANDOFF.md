# Handoff — casehub-desiredstate

## Last Session

Pivoted from #27 (managed pipeline) to Worker Foundation Extraction (#40). Created `casehub-worker` repo (api, runtime, testing) and `casehub-platform-governance` submodule (PolicyEnforcer). Filed detailed cross-repo migration issues. #27 brainstorming paused — provisioning vs runtime processing distinction unresolved.

## Immediate Next Step

Execute the Worker extraction across repos. Start with parent#288 (BOM + PLATFORM.md), then engine#543 (60+ file migration). See #40 for the full execution order. #27 resumes after all migration issues land.

## Cross-Module

**We're blocking** (these repos need the extraction to land):
- `casehub-engine` — engine#543 depends on parent#288 (BOM) · L · Med
- `casehub-desiredstate` — #41 depends on engine#543 · S · Low

**Platform changes on branch** (not yet merged):
- `casehub-platform` branch `issue-104-governance-types` — governance types + PolicyEnforcer. Needs merge to main + publish before parent#288.

## What's Left

- #40 cross-repo execution: parent#288 → engine#543 → #41 + claudony#157 + workers#14 + openclaw#37 + ops#8 · XL · Med
- PLATFORM.md desiredstate row needs updating to mention ExecutionBackend · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Blocked by #40; issue has full brainstorming context |
| #23 | CBR integration for desired-state evolution | M | High | Needs casehub-neural-text |
| #24 | State-vector abstraction for QuarkMind | L | High | Different graph model needed |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Extraction spec: `docs/superpowers/specs/2026-06-18-worker-foundation-extraction-design.md`
- Extraction plan: `docs/superpowers/plans/2026-06-18-worker-foundation-extraction.md`
- Pipeline spec (#28): `docs/superpowers/specs/2026-06-17-pipeline-runtime-design.md`
- Blog: `blog/2026-06-19-mdp01-worker-breaks-free.md` (workspace)
