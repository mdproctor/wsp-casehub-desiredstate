# Handoff — casehub-desiredstate

## Last Session

Fixed CI (#78). Two failures: (1) DoublePreference/IntPreference imports
resolved by platform branch merge — no code change needed; (2)
SituationDetectionTest missing RasMetrics constructor parameter from
casehub-ras update — added the parameter. CI green at `08c8e50`.

## Immediate Next Step

Pick next from What's Next table. #27 (managed pipeline mode) is paused on the
stack — resume with `/work` or start something new.

## Cross-Module

**Blocked by:**
- `casehub-engine-flow` — CaseTransitionExecutor needs `CallableDispatchRegistry` SPI · S · Low
- `casehub-work` — WorkItem-backed handlers need `WorkItemCreator` SPI · S · Low

**Pending cross-repo:**
- `casehub-engine-api` — still has DoublePreference/IntPreference in `io.casehub.api.spi.routing` — can delete now that platform-api has published
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
