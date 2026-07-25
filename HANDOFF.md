# Handoff — casehub-desiredstate

## Last Session

Landed #90 ThresholdFaultPolicy — reusable count-based fault escalation with
FaultPolicy SPI composition. Design review eliminated EscalationAction (duplicate
type), moved addReviewNode to FaultPolicy as static factory. Null validation added
to FaultEvent.detail and StepOutcome reason fields. Pre-existing engine-adapter
build failure on main (CaseTransitionExecutor Mutiny type inference — from #384
blocking RAS SPI migration).

## Immediate Next Step

Fix the pre-existing engine-adapter build failure (`CaseTransitionExecutor.java:115`
— Mutiny `completionStage` type inference after #384 blocking RAS SPI migration).

## Cross-Module

**Enabled** (we delivered, downstream not yet done):
- `casehub-ras` — `SituationDefinitionRegistry.forTesting()` factory shipped (`f196209`) · XS · Low

## What's Left

- `neocortex#142` — Wire CbrOutcomeConsumer to platform CloudEvent routing · open
- casehub-ops — remove App* workaround clones now that #84 shipped · S · Low
- engine-adapter build failure — CaseTransitionExecutor.java:115 Mutiny type inference · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #85 | Persisted fault counts for ThresholdFaultPolicy | M | Med | Unblocked by #90 |
| #86 | Multi-tier escalation support in ThresholdFaultPolicy | M | Med | Unblocked by #90 |
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | On pause stack, unblocked (#40 closed) |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |
| #74 | Logistics example with blocks summarisation feeding RAS | M | Med | Design question — integration scope |

## References

- ADR: `docs/adr/0001-desired-state-as-planning-paradigm.md`
