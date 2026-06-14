# Handoff — casehub-desiredstate

## Last Session

Epic 1 complete — generic desired-state runtime built, reviewed, squashed, merged, and pushed. 5 modules, 99 tests, Nefarious Dungeons example with 2D tile visualizer. Key architectural insight: desired-state and casehub-engine are two types of Goal (invariant vs achievement), with composite goals as the unifying concept.

## Immediate Next Step

Pick the next issue. Candidates: casehub-ops domain implementations (deployment, infra, compliance, iot) building on the generic runtime, or one of the platform-level design issues (#22–#25, parent#233, parent#234).

## What's Left

- PLATFORM.md needs updating with desiredstate entries (build order, dependency map, capability ownership) · S · Low
- `issue-1-minimal-api-types` project branch to delete (orphaned, all work merged from `issue-1-generic-runtime`) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #2 | Epic 2: Toy domain — infrastructure provisioning (validates generic runtime) | L | Med | Blocked by Epic 1 (done) — now unblocked |
| #22 | Unified tracing across casehub-engine and quarkus-flow | S | Med | Stretch goal |
| #23 | CBR integration for desired-state evolution | M | High | Future — needs casehub-neural-text |
| #24 | State-vector abstraction for QuarkMind | L | High | Future — different graph model needed |
| #25 | Desired-state as alternative case planning model | L | High | Platform-level — depends on parent#233 |
| parent#233 | Goal as first-class platform concept | XL | High | Invariant/achievement/composite goals |
| parent#234 | Reactive case container — long-lived root cases | L | High | Event-driven process manager pattern |

## References

- Spec: `docs/superpowers/specs/2026-06-12-generic-runtime-design.md`
- Plan: `docs/superpowers/plans/2026-06-13-generic-runtime-plan.md`
- Research: `docs/superpowers/research/2026-06-07-desired-state-management-research.md`
- Blog: `blog/2026-06-14-mdp01-desired-state-as-goal.md` (workspace)
