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
- #187 — make history prune depth configurable via PreferenceProvider (hardcoded to 10, matches default)
- Per-node rollback deferred by design — functionally `UpdateNode` with historical spec lookup, no new runtime capability needed
- GraphSerializer doesn't serialize ordering constraints — pre-existing gap, not introduced by this work

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
