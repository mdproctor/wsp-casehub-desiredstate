# Handoff — casehub-desiredstate

## Last Session

CBR Revise step (#76). CbrProposalTracker mediates between CBR components and
ReconciliationLoop — records proposals at CBR decision time, matches per-node
outcomes against pending proposals after execution, emits `io.casehub.cbr.outcome`
CloudEvents. Breaking SPI change: FaultPolicy.onFault() and SituationRecompiler.recompile()
gained tenancyId as first parameter (13+2 implementations in casehub-ops need updating).
Design-reviewed (3 rounds, $13.08, 14 issues, all resolved). Landed as `796d416` on main.

## Immediate Next Step

Pick next from What's Next table. #27 (managed pipeline mode) is paused on the
stack — resume with `/work` or start something new.

## Cross-Module

**Blocked by:**
- `casehub-engine-flow` — CaseTransitionExecutor needs `CallableDispatchRegistry` SPI · S · Low
- `casehub-work` — WorkItem-backed handlers need `WorkItemCreator` SPI · S · Low

**Notify:**
- `casehub-ops` — FaultPolicy + SituationRecompiler signatures changed (tenancyId added); ops#52 filed · S · Low
- `casehub-neocortex` — CbrCaseMemoryStore.recordOutcome() needed to consume CBR outcome events; neocortex#140 filed · M · Med

## What's Left

- `#77` — Promote DoublePreference/IntPreference to casehub-platform-api · S · Low
- `neocortex#140` — CBR Revise SPI in neocortex (recordOutcome, confidence adjustment) · M · Med
- `ops#52` — FaultPolicy/SituationRecompiler tenancyId migration · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused on stack |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Spec: `docs/specs/2026-07-12-cbr-revise-outcome-feedback-design.md`
- Plan: `docs/plans/2026-07-13-cbr-revise-outcome-feedback.md`
- Design review: `~/adr/casehub-desiredstate/cbr-revise-outcome-feedback-20260712-235748/`
- Blog: `blog/2026-07-13-mdp01-when-the-graph-learns-from-its-mistakes.md`
