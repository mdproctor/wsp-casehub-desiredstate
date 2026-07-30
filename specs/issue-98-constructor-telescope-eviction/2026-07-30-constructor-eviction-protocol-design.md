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
void evictAll(String tenancyId, Set<NodeId> retainedNodes);
```

Removes entries for non-retained nodes across all namespaces for the given tenant. Non-default
(pre-release — all implementations must provide it). Complements per-namespace `evict()` which
remains for policy-specific cleanup.

**InMemoryFaultCountStore implementation:**

```java
public void evictAll(String tenancyId, Set<NodeId> retainedNodes) {
    counts.keySet().removeIf(key ->
        key.tenancyId().equals(tenancyId) && !retainedNodes.contains(key.nodeId()));
}
```

**JpaFaultCountStore implementation:**

```java
public void evictAll(String tenancyId, Set<NodeId> retainedNodes) {
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
        store.evictAll(tenancyId, desired.nodes().keySet());
    }
}
```

CDI-discovered via `Instance<GlobalReconciliationListener>` in ReconciliationLoop. Exception isolation
already handled by ReconciliationLoop's per-listener try/catch.

### Design fix: remove listener firing from reconcileTypes()

`reconcileTypes()` calls `fireListener(filtered, actual)` with a type-filtered graph. If the eviction
listener receives a filtered graph, it treats nodes of other types as "not retained" and incorrectly
evicts their counts. This affects any GlobalReconciliationListener that uses the desired graph as truth.

**Fix:** Remove listener firing from `reconcileTypes()` entirely. Type-filtered resync is a sub-operation,
not a full cycle. If resync discovers drift or executes transitions, it emits CloudEvents → triggers
debounced full `reconcile()` → listeners fire with the correct full graph. Delay bounded by debounce
window (default 1s).

Split `fireListener()` into `fireGlobalListeners()` + `firePerTenantListener()` for clarity.
`reconcile()` calls both. `reconcileTypes()` calls neither.

---

## 3. Protocol Update (#99)

### Problem

The `persistence-backend-cdi-priority` protocol (casehub/garden) narrows `@DefaultBean` to "no-op or mock."
`DefaultFaultCountStore` is a functional `@DefaultBean` — wraps `InMemoryFaultCountStore`, counting works
out of the box without persistence. Same pattern as `SimpleTransitionExecutor`.

### Change

Update the protocol to document the CDI priority tier:

| Tier | Pattern | Example |
|------|---------|---------|
| Full implementation | `@ApplicationScoped` (displaces @DefaultBean) | JpaFaultCountStore |
| Functional fallback | `@DefaultBean @ApplicationScoped` | DefaultFaultCountStore |
| No-op / misconfiguration signal | `@DefaultBean @ApplicationScoped` | NoOpPendingApprovalHandler |

`@DefaultBean` is valid for functional fallbacks when the SPI has meaningful non-persistent semantics.
The distinction: a functional fallback works correctly within its constraints (in-memory counting works,
just not durable); a no-op signals misconfiguration.

Cross-repo: casehub/garden. Protocols do not live in this repo.

---

## Execution Order

1. **#98** — Builder + CbrProposalTracker bug fix. Foundation for subsequent test patterns.
2. **#97** — evictAll() on FaultCountStore, eviction listener, reconcileTypes() listener fix.
3. **#99** — Protocol update in casehub/garden.

---

## Files Changed

### #98
- `runtime/.../ReconciliationLoop.java` — add Builder, delete telescope, fix CDI constructor
- `runtime/src/test/.../*.java` — ~31 call sites migrated (10 test classes)
- `engine-adapter/src/test/.../DesiredStateReplanDispatchTest.java` — 1 call site
- `examples/expansion/src/test/.../ExpansionLifecycleTest.java` — 1 call site

### #97
- `api/.../FaultCountStore.java` — add evictAll()
- `api/.../InMemoryFaultCountStore.java` — implement evictAll()
- `persistence-jpa/.../JpaFaultCountStore.java` — implement evictAll()
- `runtime/.../FaultCountEvictionListener.java` — new file
- `runtime/.../ReconciliationLoop.java` — split fireListener(), remove from reconcileTypes()
- `api/src/test/.../InMemoryFaultCountStoreTest.java` — evictAll() tests
- `persistence-jpa/src/test/.../JpaFaultCountStoreTest.java` — evictAll() tests
- `runtime/src/test/.../FaultCountEvictionListenerTest.java` — new file
- `runtime/src/test/.../ReconciliationLoop*Test.java` — adjust for listener firing change

### #99
- `casehub/garden/docs/protocols/.../persistence-backend-cdi-priority.md` — update
