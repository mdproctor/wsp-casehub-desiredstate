# Handoff — casehub-desiredstate

## Last Session

CBR integration (#23). Three SPIs (ConfigurationRetriever, ConfigurationAdapter,
CbrConfiguration) in api/. CbrFaultPolicy (tactical — per-node mutations) and
CbrSituationRecompiler (strategic — whole-graph replacement via SituationRecompilerEngine
chain-of-responsibility) in runtime/. SituationRecompiler SPI evolved with ActualState
parameter and priority() method. GraphDiff converts adapted graph fragments to mutations
scoped by NodeType. Design-reviewed (3 rounds, $12.88, 16 issues, all resolved).
Landed as `12d4f4b` on main.

## Immediate Next Step

Pick next from What's Next table. #27 (managed pipeline mode) is paused on the
stack — resume with `/work` or start something new.

## Cross-Module

**Blocked by:**
- `casehub-engine-flow` — CaseTransitionExecutor needs `CallableDispatchRegistry` SPI · S · Low
- `casehub-work` — WorkItem-backed handlers need `WorkItemCreator` SPI · S · Low

**Notify:**
- `casehub-ops` — SituationRecompiler SPI signature changed (ActualState param + priority()); 4 implementations need updating · S · Low

## What's Left

- `#76` — CBR Revise step: outcome feedback loop to case store · M · High
- `#77` — Promote DoublePreference/IntPreference to casehub-platform-api · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused on stack |
| #23 follow-on | Wire CBR into pipeline or expansion example | S | Med | Needs domain ConfigurationRetriever impl |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Spec: `docs/specs/2026-07-10-cbr-integration-design.md`
- Plan: `docs/plans/2026-07-12-cbr-integration.md`
- Design review: `~/adr/casehub-desiredstate/cbr-integration-20260711-191746/`
- Blog: `blog/2026-07-12-mdp01-when-the-graph-remembers.md`
