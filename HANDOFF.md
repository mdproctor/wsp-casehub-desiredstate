# Handoff — casehub-desiredstate

## Last Session

Wrote ADR 0001 — combined findings from epic #24's three POCs. Decision: graph
model + separate RAS reasoning layer (Option B). Epic #24 closed. Fixed CI: two
root causes — JUnit class discovery leaking `scope:provided` deps via package-access
hack (cross-repo fix to casehub-ras: `forTesting()` factory), and jackson-jq 1.6.0/1.6.2
convergence. ARC42 stale scan updated Chapter 7 (#40, #41 closed) and expansion markers.

## Immediate Next Step

Pick next from What's Next table. #27 (managed pipeline mode) is unblocked (#40
shipped) and has a branch on the pause stack — resume with `/work`.

## Cross-Module

**Enabled** (we delivered, downstream not yet done):
- `casehub-ras` — `SituationDefinitionRegistry.forTesting()` factory shipped (`f196209`) · XS · Low

## What's Left

- `neocortex#142` — Wire CbrOutcomeConsumer to platform CloudEvent routing · open
- casehub-ops — remove App* workaround clones now that #84 shipped · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | On pause stack, **unblocked** (#40 closed) |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |
| #74 | Logistics example with blocks summarisation feeding RAS | M | Med | Design question — integration scope |

## References

- ADR: `docs/adr/0001-desired-state-as-planning-paradigm.md`
- Blog: `blog/2026-07-23-mdp01-the-class-that-shouldnt-have-been-there.md`
- Garden: `GE-20260723-5d8f51` — JUnit discovery + scope:provided gotcha
