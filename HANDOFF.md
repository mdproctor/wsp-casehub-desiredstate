# Handoff — casehub-desiredstate

## Last Session

Landed #94, #95, #96 — fault infrastructure batch. GlobalReconciliationListener
SPI for application-scoped listeners (#96). JPA-backed FaultCountStore with CDI
priority ladder, Flyway migration, DefaultFaultCountStore fallback (#94).
ProvisionEscalationFaultPolicy migrated to FaultCountStore SPI with tenant
isolation and lazy eviction (#95). Design-reviewed (4 rounds, 14 issues, all
resolved, $15.10). Filed follow-ups: #97 (eviction listener), #98 (constructor
telescope), #99 (protocol update). Filed parent#400 (doc sync).

## Immediate Next Step

Pick next from What's Next table. #97 is the natural follow-on — eviction
consumer via GlobalReconciliationListener.

## What's Left

- casehub-ops — remove App* workaround clones now that #84 shipped · S · Low
- Hygiene: unrecovered blog/specs on closed branches issue-57, issue-85, issue-93 · S · Low
- Hygiene: unstamped branch issue-384-retire-reactive · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #97 | FaultCountEvictionListener — evict() consumer via GlobalReconciliationListener | M | Med | Follow-on from #94/#96. Namespace-aware design needed |
| #98 | ReconciliationLoop constructor telescope refactoring | S | Low | Pre-existing tech debt, now 8+ params |
| #99 | Protocol update — persistence-backend-cdi-priority @DefaultBean for functional fallbacks | XS | Low | Documentation only |
| #86 | Multi-tier escalation support in ThresholdFaultPolicy | M | Med | |
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | On pause stack |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |
| #74 | Logistics example with blocks summarisation feeding RAS | M | Med | Design question — integration scope |

## References

- Spec: `docs/specs/2026-07-29-fault-infra-design.md` (promoted to project)
- Design review: `~/adr/casehub-desiredstate/fault-infra-20260730-003825/`
