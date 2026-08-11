# Constructor Telescope, Eviction Listener, Protocol Update

**Branch:** issue-98-constructor-telescope-eviction
**Covers:** #98, #97, #99
**Date:** 2026-07-30

---

## 1. ReconciliationLoop Builder (#98)

### Problem

ReconciliationLoop has 7 public constructors telescoping from 5 to 10 parameters. The telescope
exists because Java lacks optional/named parameters — each overload adds one more optional param.
~31 test call sites spread across 10 test classes. A config record was considered but rejected:
the optional params (timing, collaborators, listeners) share no semantic relationship beyond
"being optional" — grouping by optionalness is artificial.

### Design

Add `static Builder builder(...)` taking the 5 mandatory collaborators. Fluent setters for optionals.

**Mandatory (builder constructor args):**

| Param | Type |
|-------|------|
| planner | TransitionPlanner |
| executor | TransitionExecutor |
| actualStateAdapterRouter | ActualStateAdapterRouter |
| faultPolicyEngine | FaultPolicyEngine |
| mergedEventSource | MergedEventSource |

**Optional (builder setters with defaults):**

| Setter | Type | Default |
|--------|------|---------|
| router | NodeProvisionerRouter | null |
| debounceWindow | Duration | DEFAULT_DEBOUNCE (1s) |
| resyncInterval | Duration | null (uses DEFAULT_RESYNC 5m) |
| cloudEventSink | Consumer\<CloudEvent\> | event -> {} |
| cbrTracker | CbrProposalTracker | new CbrProposalTracker() |
| globalListeners | List\<GlobalReconciliationListener\> | List.of() |

**Structural changes:**
- Delete all 7 public telescope constructors (lines 131–206)
- CDI `@Inject` constructor stays (production path — CDI requires distinct typed params)
- Private master constructor stays (called by CDI constructor and Builder.build())
- Builder is a static inner class with `build()` calling the master constructor

### Bug fix: CbrProposalTracker instance split

The CDI constructor passes `null` for cbrTracker. The master constructor creates `new CbrProposalTracker()`.
CbrFaultPolicy gets the CDI-managed `@ApplicationScoped` CbrProposalTracker injected. These are different
instances — proposals recorded by CbrFaultPolicy are invisible to the loop's outcome matching. CBR outcome
CloudEvents never emit in production.

**Fix:** Add `CbrProposalTracker` to the CDI constructor (8 → 9 injected params).

### Test migration

~31 call sites. Mechanical transformation. Example (most common pattern, ~20 sites):

```java
// Before
new ReconciliationLoop(planner, executor, adapter, engine, source, debounce, resync)
// After
ReconciliationLoop.builder(planner, executor, adapter, engine, source)
    .debounceWindow(debounce).resyncInterval(resync).build()
```

---

## 2. FaultCountEvictionListener (#97)

### Problem

`FaultCountStore.evict()` has zero callers. The method removes fault counts for nodes no longer in the
desired graph, but no consumer exists to call it after each reconciliation cycle.

### Key insight: no namespace registry needed

The issue description assumes a namespace registry: "each policy owns its namespace, so the listener
needs a registry of (namespace, FaultCountStore) pairs." First principles says otherwise: if a node
is gone from the desired graph, ALL its counts across ALL namespaces are stale — the node cannot fault
regardless of which policy tracked it. Cross-namespace eviction is semantically correct and eliminates
the registry.

### Design

**New method on FaultCountStore (api/):**

```java
void evictAcrossNamespaces(String tenancyId, Set<NodeId> retainedNodes);
```

Removes entries for non-retained nodes across all namespaces for the given tenant. Non-default
(pre-release — all implementations must provide it). Complements per-namespace `evict()` which
remains for policy-specific cleanup.

**InMemoryFaultCountStore implementation:**

```java
public void evictAcrossNamespaces(String tenancyId, Set<NodeId> retainedNodes) {
    counts.keySet().removeIf(key ->
        key.tenancyId().equals(tenancyId) && !retainedNodes.contains(key.nodeId()));
}
```

**JpaFaultCountStore implementation:**

```java
public void evictAcrossNamespaces(String tenancyId, Set<NodeId> retainedNodes) {
    // Empty retained → delete all for tenant
    // Non-empty → DELETE ... WHERE tenancy_id = :tid AND node_id NOT IN :retained
}
```

No Flyway migration — same table, different query pattern.

**FaultCountEvictionListener (runtime/):**

```java
@ApplicationScoped
public class FaultCountEvictionListener implements GlobalReconciliationListener {
    private final FaultCountStore store;

    @Inject
    public FaultCountEvictionListener(FaultCountStore store) {
        this.store = store;
    }

    @Override
    public void onReconciliationCycleCompleted(String tenancyId,
            DesiredStateGraph desired, ActualState actual) {
        store.evictAcrossNamespaces(tenancyId, desired.nodes().keySet());
    }
}
```

CDI-discovered via `Instance<GlobalReconciliationListener>` in ReconciliationLoop. Exception isolation
already handled by ReconciliationLoop's per-listener try/catch.

### Tenant stop cleanup

Add a lifecycle hook to `GlobalReconciliationListener`:

```java
public interface GlobalReconciliationListener {
    void onReconciliationCycleCompleted(String tenancyId, DesiredStateGraph desired, ActualState actual);
    default void onTenantStopped(String tenancyId) {}
}
```

`TenantLoop.stop()` calls `onTenantStopped(tenancyId)` on all global listeners with per-listener
try/catch exception isolation (same pattern as `fireListener()`). One failing listener must not
prevent others from being notified or block the stop sequence.

**Stop sequence ordering:** unsubscribe events → call `onTenantStopped` → cancel futures →
`cbrTracker.clearTenant()`. The `onTenantStopped` call fires after event unsubscription to prevent
a race where a new event arrives during listener cleanup, triggers `scheduleReconciliation()`, and
re-adds stale state after eviction. It fires before future cancellation because listeners may need
the scheduler context (e.g., for transactional cleanup).

The eviction listener implements it:

```java
@Override
public void onTenantStopped(String tenancyId) {
    store.evictAcrossNamespaces(tenancyId, Set.of());
}
```

Passing an empty retained set evicts all counts for the tenant. Without this, stale fault counts for
stopped tenants persist until the tenant is restarted and a full reconcile fires. For long-lived
processes with tenant churn, this is a slow storage leak.

### Design fix: remove listener firing from reconcileTypes()

`reconcileTypes()` currently calls `fireListener(fullDesired, actual)` at the end of each type-filtered
resync cycle. While it correctly passes the full desired graph (not the filtered one), listener firing
from this path is wrong for three reasons:

1. **Semantic incorrectness:** `reconcileTypes()` is a sub-operation (type-filtered resync driven by
   interval-grouped timers), not a full reconciliation cycle. Signaling "cycle completed" when only a
   subset of node types was reconciled is semantically wrong. Future listeners — not just eviction —
   may take once-per-cycle actions that should not fire for sub-operations.

2. **Partial actual state:** `readActual(filteredDesired, tenancyId)` only reads actual state for the
   filtered types. Listeners receive full `desired` but partial `actual` — a landmine for any listener
   that uses the `actual` parameter. The eviction listener happens not to use `actual`, but this is
   an implicit coupling that breaks silently when a new listener does.

3. **Redundant invocations:** With N interval groups, listeners fire N times per resync cycle with the
   same full desired graph. This is wasteful and potentially incorrect for listeners with side effects.

**Fix:** Remove listener firing from `reconcileTypes()` entirely.

**Delay analysis:** With this change, listeners fire only from full `reconcile()`, which is triggered by
events on `MergedEventSource` (debounced by debounce window). In production mode with interval groups,
there is no periodic full `reconcile()` — all resync goes through `reconcileTypes()`. This means
listeners fire only on event-driven reconciliation, not on periodic resync.

For eviction, this is correct: eviction is needed when nodes leave the desired graph. Node removal is
an external desired-state-change event, which triggers debounced `reconcile()` via `MergedEventSource`.
Eviction fires promptly (bounded by debounce window, default 1s) when it matters. Between events, stale
fault counts from unchanged resync cycles are harmless noise — they consume storage but don't affect
correctness because the nodes they track are still present in the desired graph.

Edge case: `faultFeedback()` mutates `desiredRef` via `casRetryMutations()` without triggering
`scheduleReconciliation()`. `GraphMutation.RemoveNode` is a first-class mutation variant, and
`CbrFaultPolicy` uses `GraphDiff.computeMutations()` which can produce any mutation type including
node removal (e.g., CBR suggests a simpler topology with fewer nodes). If a fault policy removes a node
from the graph, stale fault counts for that node persist until the next event-triggered reconcile. The
consequence is bounded: delayed eviction (storage waste), not incorrect behavior — the counts are inert
because the node is no longer in the desired graph and cannot fault.

**Contract change for per-tenant `ReconciliationListener`:** The same three reasons apply to the
per-tenant listener set via `start(tenancyId, desired, listener)`. Callers passing a
`ReconciliationListener` currently receive notifications on every type-filtered resync — that contract
changes. The method name `onReconciliationCycleCompleted` implies a full cycle; type-filtered resync
is not one. Callers that need per-type-group notifications should use a different mechanism (e.g.,
subscribe to CloudEvents).

Split `fireListener()` into `fireGlobalListeners()` + `firePerTenantListener()` for clarity.
`reconcile()` calls both. `reconcileTypes()` calls neither.

---

## 3. Protocol Update (#99)

### Problem

The `persistence-backend-cdi-priority` protocol (casehub/garden) narrows `@DefaultBean` to "no-op or mock."
`DefaultFaultCountStore` is a functional `@DefaultBean` — wraps `InMemoryFaultCountStore`, counting works
out of the box without persistence. Same pattern as `SimpleTransitionExecutor`.

### Change

Update the protocol to split Tier 1 into two semantic categories. The full tier table becomes:

| Tier | Annotation | Semantics | Example |
|------|-----------|-----------|---------|
| 1a | `@DefaultBean @ApplicationScoped` | Functional fallback — works correctly within constraints (e.g., in-memory counting works, just not durable) | DefaultFaultCountStore |
| 1b | `@DefaultBean @ApplicationScoped` | No-op / misconfiguration signal — signals missing backend | NoOpPendingApprovalHandler |
| 2 | `@ApplicationScoped` | Primary backend (JPA) — displaces @DefaultBean | JpaFaultCountStore |
| 3 | `@Alternative @Priority(1)` | Secondary backend (NoSQL) — beats @ApplicationScoped | — |
| 4 | `@Alternative @Priority(100)` | In-memory test override — beats everything | — |

**What changes:** Tier 1 splits from "mock/no-op" into two categories. `@DefaultBean` is valid for
functional fallbacks when the SPI has meaningful non-persistent semantics. Tiers 2–4 are unchanged.

**What doesn't change:** The "Don't use `@DefaultBean` on real (non-mock) implementations" guidance
narrows to: "Don't use `@DefaultBean` on implementations that require external infrastructure (DB,
message broker) to function." A functional fallback that works self-contained (in-memory) is not
a "real implementation" in this sense — it's a degraded-but-functional default.

Cross-repo: casehub/garden. Protocols do not live in this repo.

---

## Execution Order

1. **#98** — Builder + CbrProposalTracker bug fix. Foundation for subsequent test patterns.
2. **#97** — evictAcrossNamespaces() on FaultCountStore, eviction listener, reconcileTypes() listener fix.
3. **#99** — Protocol update in casehub/garden.

---

## Files Changed

### #98
- `runtime/.../ReconciliationLoop.java` — add Builder, delete telescope, fix CDI constructor
- `runtime/src/test/.../*.java` — ~31 call sites migrated (10 test classes)
- `engine-adapter/src/test/.../DesiredStateReplanDispatchTest.java` — 1 call site
- `examples/expansion/src/test/.../ExpansionLifecycleTest.java` — 1 call site

### #97
- `api/.../FaultCountStore.java` — add evictAcrossNamespaces()
- `api/.../InMemoryFaultCountStore.java` — implement evictAcrossNamespaces()
- `api/.../GlobalReconciliationListener.java` — add default onTenantStopped() lifecycle hook
- `persistence-jpa/.../JpaFaultCountStore.java` — implement evictAcrossNamespaces()
- `runtime/.../FaultCountEvictionListener.java` — new file (implements both hooks)
- `runtime/.../ReconciliationLoop.java` — split fireListener(), remove from reconcileTypes(), call onTenantStopped in stop()
- `api/src/test/.../InMemoryFaultCountStoreTest.java` — evictAcrossNamespaces() tests
- `persistence-jpa/src/test/.../JpaFaultCountStoreTest.java` — evictAcrossNamespaces() tests
- `runtime/src/test/.../FaultCountEvictionListenerTest.java` — new file
- `runtime/src/test/.../ReconciliationLoop*Test.java` — adjust for listener firing change + stop cleanup

### #99
- `casehub/garden/docs/protocols/.../persistence-backend-cdi-priority.md` — update