# Handoff — casehub-desiredstate

## Last Session

Promote DoublePreference/IntPreference to casehub-platform-api (#77) and add
examples aggregator POM (#75). Cross-repo: added both types to
`io.casehub.platform.api.preferences` in casehub-platform (branch
`issue-77-promote-preference-types`, commit `e74a86d`). Deleted local copies
from desiredstate api/, updated DesiredStatePreferenceKeys imports. Created
`examples/pom.xml` aggregator for casehub-examples subtree integration.
Landed as `47a673a` on main.

## Immediate Next Step

Pick next from What's Next table. #27 (managed pipeline mode) is paused on the
stack — resume with `/work` or start something new. Platform branch
`issue-77-promote-preference-types` needs merging to platform main.

## Cross-Module

**Blocked by:**
- `casehub-engine-flow` — CaseTransitionExecutor needs `CallableDispatchRegistry` SPI · S · Low
- `casehub-work` — WorkItem-backed handlers need `WorkItemCreator` SPI · S · Low

**Pending cross-repo:**
- `casehub-platform` — branch `issue-77-promote-preference-types` (`e74a86d`) needs merge to main
- `casehub-engine-api` — still has DoublePreference/IntPreference in `io.casehub.api.spi.routing` — can delete once platform-api publishes
- `casehub-ops` — FaultPolicy + SituationRecompiler signatures changed (tenancyId added); ops#52 filed · S · Low

## What's Left

- `neocortex#142` — Wire CbrOutcomeConsumer to platform CloudEvent routing · open
- `neocortex#141` — Map<String, Object> → Map<String, FeatureValue> migration · open
- `ops#52` — FaultPolicy/SituationRecompiler tenancyId migration · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused on stack |
| #54 | requiresHuman gating for deprovision | — | — | |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Spec: `docs/specs/2026-07-12-cbr-revise-outcome-feedback-design.md`
- Blog: `blog/2026-07-13-mdp01-when-the-graph-learns-from-its-mistakes.md`
