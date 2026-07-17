# Handoff — casehub-desiredstate

## Last Session

Closed #54 — requiresHuman gating for deprovision. `HumanNodeHandler` gains
default `onDeprovision`, `SimpleTransitionExecutor` checks `requiresHuman` for
both provision and deprovision, `WorkItemHumanNodeHandler` creates deprovision
WorkItems with action-qualified callerRef, `CaseTransitionExecutor` separates
human removals into `humanTask` bindings. Design review changed the approach
from abstract to default method. Landed as `c4669e4` on main.

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
- `#79` — per-action requiresHuman flags · S · Med
- `#80` — WorkItem lifecycle coordination (provision/deprovision) · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused on stack |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |
| #79 | Per-action requiresHuman flags | S | Med | Filed from #54 |
| #80 | WorkItem lifecycle coordination | S | Med | Filed from #54 |

## References

- Spec: `docs/specs/2026-07-17-requireshuman-deprovision-design.md`
- Blog: `blog/2026-07-17-mdp01-when-deprovision-meets-the-human-gate.md`
