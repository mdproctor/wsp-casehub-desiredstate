# Handoff — casehub-desiredstate

## Last Session

Landed #98, #97, #99 — constructor telescope, eviction listener, protocol update.
ReconciliationLoop.Builder replaces 7 telescope constructors. CbrProposalTracker
CDI injection bug fixed (was creating separate instance, breaking CBR outcome events).
FaultCountEvictionListener with cross-namespace eviction — no namespace registry needed.
reconcileTypes() listener firing removed (semantic fix). onTenantStopped lifecycle hook.
CDI priority protocol split Tier 1 into 1a (functional fallback) / 1b (no-op).
Design-reviewed (3 rounds, 11 issues, all verified, $12.32).

## Immediate Next Step

Pick next from What's Next table. #86 (multi-tier escalation) is the natural
follow-on — builds on the eviction and fault count infrastructure.

## What's Left

- casehub-ops — remove App* workaround clones now that #84 shipped · S · Low
- Hygiene: unrecovered blog/specs on closed branches issue-57, issue-85, issue-93 · S · Low
- Hygiene: unstamped branch issue-384-retire-reactive · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #86 | Multi-tier escalation support in ThresholdFaultPolicy | M | Med | Builds on eviction infrastructure |
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | On pause stack |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |
| #74 | Logistics example with blocks summarisation feeding RAS | M | Med | Design question — integration scope |

## References

- Spec: `docs/specs/2026-07-30-constructor-eviction-protocol-design.md` (posted to #98)
- Design review: `~/adr/casehub-desiredstate/constructor-eviction-protocol-20260730-052418/`
