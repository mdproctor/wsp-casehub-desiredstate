# Handoff — casehub-desiredstate

## Last Session

Landed #85 — FaultCountStore SPI, InMemoryFaultCountStore, ThresholdFaultPolicy
refactored to delegate counting. Namespace-scoped keys for composability, tenant
isolation, lazy eviction via guard reordering, resetCount for recovery integration.
Design-reviewed (4 rounds, 10 issues, all resolved). Fixed CI (#92 spatial test)
en route. Filed follow-ups: #94 (persistent backend), #95 (ProvisionEscalationFaultPolicy
migration), #96 (composite ReconciliationListener).

## Immediate Next Step

Pick next from What's Next table. #94 is the natural follow-on from #85.

## What's Left

- casehub-ops — remove App* workaround clones now that #84 shipped · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #94 | Persistent FaultCountStore implementation | M | Med | Follow-on from #85 |
| #95 | Migrate ProvisionEscalationFaultPolicy to FaultCountStore | S | Low | Same counting bugs as pre-#85 ThresholdFaultPolicy |
| #96 | Composite ReconciliationListener (multi-listener) | S | Med | Unblocks bulk eviction integration pattern |
| #86 | Multi-tier escalation support in ThresholdFaultPolicy | M | Med | |
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | On pause stack |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |
| #74 | Logistics example with blocks summarisation feeding RAS | M | Med | Design question — integration scope |

## References

- ADR: `docs/adr/0001-desired-state-as-planning-paradigm.md`
- Spec: `docs/specs/2026-07-28-persisted-fault-counts-design.md`
