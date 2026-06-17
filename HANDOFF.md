# Handoff — casehub-desiredstate

## Last Session

Closed #22 (OTel tracing), #29 (AgentProvider for AI_REVIEW), #30 (WorkItem for HUMAN_REVIEW), and branch cleanup (deleted `issue-1-minimal-api-types`). Three-tier fault escalation now wired to real platform SPIs. 17 new tests (139 total). Garden entry submitted: `GE-20260617-abe516` (static tracer field gotcha). parent#271 filed for PLATFORM.md cross-repo dependency sync.

## Immediate Next Step

Pick the next issue. Top candidates: pipeline evolution (#27, #28 for managed mode and per-node toolbox) or platform design (#23–#25).

## What's Left

- parent#271 — PLATFORM.md sync for desiredstate pipeline AgentProvider dependency · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow orchestration per node with audit trail | M | Med | Evolves pipeline from example to runtime |
| #28 | Per-node execution toolbox — Camel, Drools, Stream backends | L | High | AiFusion showcase |
| #23 | CBR integration for desired-state evolution | M | High | Needs casehub-neural-text |
| #24 | State-vector abstraction for QuarkMind | L | High | Different graph model needed |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Spec (OTel): `docs/superpowers/specs/2026-06-17-otel-tracing-design.md`
- Spec (agent): `docs/superpowers/specs/2026-06-17-agent-review-integration-design.md`
- Spec (WorkItem): `docs/superpowers/specs/2026-06-17-workitem-integration-design.md`
- Blog: `blog/2026-06-17-mdp02-wiring-the-three-tiers.md` (workspace)
- Garden: `GE-20260617-abe516` (OTel static tracer gotcha)
