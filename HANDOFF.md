# HANDOFF — casehub-desiredstate

## Last Session

Closed #134 (rollback — revert to previous desired-state graph). Landed as `5725aff..4cd6880` on main (5 commits).

**What shipped:**
- **DesiredStateHistoryStore SPI** — append-only version log per tenant with `record()`, `load()`, `listVersions()`, `prune()`, `remove()`. InMemoryDesiredStateHistoryStore in api/, DefaultDesiredStateHistoryStore in runtime-core/
- **ReconciliationLoop.rollback(tenancyId, version, initiatedBy)** — loads historical graph, cancels pending approval and active lifecycle, emits ROLLBACK_INITIATED CloudEvent, swaps desired state. Rollback skips pruning (rollback-of-rollback recovery). `rollbackInProgress` volatile flag blocks concurrent reconcile()
- **PlanContext sealed interface** — `Normal | Rollback(fromVersion, toVersion, reason, rollbackId)`. Breaking change to PlanApprovalPolicy.evaluate() — third parameter, no backward compat default
- **LifecycleCanceller SPI** — breaks ReconciliationLoop→LifecycleManager circular dependency. LifecycleManager implements it
- **History capture** — recorded at `updateDesired()` and `start()` only. Not at `compareAndSetDesired()` (lifecycle phase transitions) or `casRetryMutations()` (fault policy). HISTORY_MAX_DEPTH preference key (default 10)
- **PlanApprovalGate.cancelPending(tenancyId)** — cancels stale pending plans during rollback
- **JPA persistence** — DesiredStateHistoryEntity with composite PK, Flyway V3 migration, JpaDesiredStateHistoryStore (Quarkus), SpringJpaDesiredStateHistoryStore
- **Spring parity** — DesiredStateRuntimeAutoConfiguration and DesiredStatePersistenceAutoConfiguration updated
- **MockDesiredStateHistoryStore** in testing/ module
- **Code review fix** — history store cleanup on tenant stop (TenantLoop.stop() calls historyStore.remove())

**What's deferred:**
- #187 — make history prune depth configurable via PreferenceProvider (XS/Low)
- #188 — per-node rollback convenience API (S/Med)
- #189 — GraphSerializer ordering constraints serialization (XS/Low, bug)

## Next Session

Start with `work start 187` — all three issues are follow-ons from #134 rollback work.

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|-------|-------|------------|------------|-------|
| 189 | fix: GraphSerializer ordering constraints | XS | Low | — | Bug fix, do first — unblocks correct history round-trips |
| 187 | feat: PreferenceProvider for history prune depth | XS | Low | — | Mechanical wiring, same session as 189 |
| 188 | feat: per-node rollback | S | Med | — | Builds on history store from #134, needs design for dependency semantics |

**Recommended order:** 189 → 187 → 188. The two XS issues are quick wins on the same code paths. #188 is the meatiest and benefits from a fresh context window.

**Blog entries:**
- Diary: workspace blog/2026-10-26-mdp01-rollback-desired-state.md
- Article: project docs/blog/2026-10-26-mdp01-rollback-in-reconciliation-loops.md

## Cross-Module

#161 still pending — Spring auto-config codegen requires work in `casehubio/parent` repo.

## References

| Artifact | Location |
|----------|----------|
| Issue | casehubio/casehub-desiredstate#134 |
| Commits | 5725aff..4cd6880 on main |
| Design spec | specs/issue-134-rollback-desired-state-graph/2026-10-26-rollback-design.md |
| Decisions | specs/issue-134-rollback-desired-state-graph/decisions.md (12 decisions, reviewed) |
| Implementation plan | plans/2026-10-26-rollback-desired-state-graph.md |
| Follow-up | casehubio/casehub-desiredstate#187 (preference-driven prune depth) |
| Follow-up | casehubio/casehub-desiredstate#188 (per-node rollback) |
| Follow-up | casehubio/casehub-desiredstate#189 (GraphSerializer ordering constraints) |
