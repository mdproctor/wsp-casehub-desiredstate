# Persisted Fault Counts for ThresholdFaultPolicy

**Issue:** casehubio/casehub-desiredstate#85
**Date:** 2026-07-28
**Status:** Approved

## Scope

This spec delivers the `FaultCountStore` SPI, an in-memory default implementation, and the
`ThresholdFaultPolicy` refactoring to use it. This is the architectural prerequisite for
persistence — the SPI enables pluggable persistent backends (JPA, Redis, etc.) without
further changes to the policy or runtime. The persistent backend itself is a follow-up
(casehubio/casehub-desiredstate#86). Issue #85 remains open until a persistent
implementation lands.

## Problem

ThresholdFaultPolicy uses an in-memory `ConcurrentHashMap<NodeId, Integer>` for fault counts.
Three problems:

1. **Volatile** — counts reset on process restart. A node that faulted twice before restart
   starts fresh, defeating threshold-based escalation.
2. **Leaking** — counts for nodes removed from the desired-state graph are never evicted.
   The map grows without bound over the process lifetime.
3. **No tenant isolation** — `onFault` receives `tenancyId` but ignores it. Node "n1" in
   tenant A and tenant B share a single fault count.

## Design Decisions

**Store SPI inside ThresholdFaultPolicy.** The counting concern belongs to the policy, not
the runtime. ThresholdFaultPolicy delegates counting to a pluggable `FaultCountStore` passed
via the builder. Default: in-memory (current behavior). Persistent implementations are
separate modules.

**Accumulate through recovery (default).** A recovered node retains its fault count. A
flapping node that faults 3 times across 5 recoveries still hits threshold. This is the
conservative default — repeated instability triggers intervention regardless of intermittent
recovery. Reset-on-recovery is available via `resetCount()` for callers that detect recovery
externally.

## FaultCountStore SPI

New interface in `api/` (`io.casehub.desiredstate.api`):

```java
public interface FaultCountStore {
    int incrementAndGet(String namespace, String tenancyId, NodeId nodeId);
    int getCount(String namespace, String tenancyId, NodeId nodeId);
    void reset(String namespace, String tenancyId, NodeId nodeId);
    void remove(String namespace, String tenancyId, NodeId nodeId);
    void evict(String namespace, String tenancyId, Set<NodeId> retainedNodes);
}
```

| Method | Purpose |
|--------|---------|
| `incrementAndGet` | Atomically increment and return the new count |
| `getCount` | Read without side effects; returns 0 if absent |
| `reset` | Set count to zero; `getCount` returns 0, next `incrementAndGet` returns 1 |
| `remove` | Delete entry entirely |
| `evict` | Bulk removal: delete all entries for (namespace, tenancyId) whose nodeId is NOT in retainedNodes |

**Key dimensions:** `(namespace, tenancyId, nodeId)`. The `namespace` scopes counts to a
single policy instance, ensuring two `ThresholdFaultPolicy` instances sharing the same
persistent `FaultCountStore` (e.g., a CDI-managed bean) never collide. This satisfies
Quality Goal 4 (fault composability) regardless of deployment topology. The `namespace`
is set via the builder and passed through to all store calls.

## InMemoryFaultCountStore

In `api/` (`io.casehub.desiredstate.api`). Default implementation.

- `ConcurrentHashMap` with private `record Key(String namespace, String tenancyId, NodeId nodeId)` composite key
- Thread-safe, matches current ThresholdFaultPolicy behavior
- `evict` iterates stored keys for the given (namespace, tenancyId), removes those not in the retained set

**Placement justification:** InMemoryFaultCountStore is in `api/` because ThresholdFaultPolicy's
builder requires it for the default (no-arg) path, and ThresholdFaultPolicy must remain in `api/`
as a reusable building block. Unlike `@DefaultBean` implementations (e.g., SimpleTransitionExecutor
in `runtime/`), InMemoryFaultCountStore is not CDI-managed — it is manually instantiated in the
builder. Moving it to `runtime/` would create an `api/ → runtime/` dependency cycle.

## ThresholdFaultPolicy Changes

Four changes:

### 1. Builder accepts FaultCountStore and namespace

```java
public Builder faultCountStore(FaultCountStore store) { ... }
public Builder namespace(String namespace) { ... }
```

Defaults to `new InMemoryFaultCountStore()` if store not set. The `namespace` is required
when a custom store is provided (builder validation); when using the default in-memory store,
namespace defaults to a deterministic value derived from the policy's `faultTypes` set.
Existing callers (casehub-ops policies) continue working unchanged — the default in-memory
store with auto-derived namespace preserves current behavior.

### 2. onFault uses the store

Replaces `faultCounts.merge(event.node(), 1, Integer::sum)` with
`store.incrementAndGet(tenancyId, event.node())`.

Fixes tenant isolation — counts now keyed by `(tenancyId, nodeId)`.

### 3. Lazy eviction

The `node == null` check (node absent from desired graph) is moved **above** the `faultTypes`
check. When `onFault` finds the node absent from the graph, it calls
`store.remove(namespace, tenancyId, event.node())` before returning — regardless of the fault
type. This ensures lazy eviction fires for any fault on a removed node, not just faults
matching this policy's type filter. Revised guard order:

1. Look up node in desired graph
2. If node exists and is an ignored type → return
3. **If node is null → `store.remove(...)` and return** (moved up from position 4)
4. If fault type doesn't match → return
5. If node type doesn't match → return
6. Increment and check threshold

### 4. Bulk eviction via ReconciliationListener

The `evict(namespace, tenancyId, retainedNodes)` method on `FaultCountStore` supports bulk
cleanup of entries for nodes no longer in the desired graph. The natural integration point
is a `ReconciliationListener` — after each reconciliation cycle, the listener receives the
current `DesiredStateGraph` and can call `store.evict(namespace, tenancyId, desired.nodes().keySet())`.

Lazy eviction (§3) is the primary cleanup mechanism — it fires on any fault for a removed node.
Bulk eviction is an optimization for deployers who want proactive cleanup of stale entries
in persistent stores without waiting for a fault event. The integration pattern is external
to ThresholdFaultPolicy, consistent with the SPI-driven architecture.

### 5. Public resetCount method

```java
public void resetCount(String tenancyId, NodeId nodeId) {
    store.reset(namespace, tenancyId, nodeId);
}
```

Wrapping policies that want reset-on-recovery call this when they detect recovery.
ThresholdFaultPolicy itself never calls it — recovery detection is a runtime concern
(ReconciliationLoop tracks `activeProblems` and emits `NodeRecoveredData` CloudEvents),
not a policy concern. The recommended integration pattern: a wrapping policy implements
`ReconciliationListener`, observes `ActualState` for recovered nodes, and calls
`delegate.resetCount(tenancyId, nodeId)` on transition to `PRESENT`.

### Field removal

`ConcurrentHashMap<NodeId, Integer> faultCounts` is removed entirely.

## Test Coverage

### InMemoryFaultCountStoreTest (new)

| Test | Asserts |
|------|---------|
| incrementAndGet returns sequential counts | 1, 2, 3 for successive calls |
| getCount returns 0 for absent entries | No side effects, no entry created |
| reset sets count to zero | Next increment returns 1 |
| remove deletes entry | Next increment returns 1; getCount returns 0 |
| evict removes entries not in retained set | Retained entries untouched, others gone |
| tenant isolation | Same nodeId in different tenancies counted independently |

### ThresholdFaultPolicyTest (updated)

| Test | Asserts |
|------|---------|
| tenant isolation | Same node faulting in two tenancies tracked independently |
| custom store via builder | Injected store receives incrementAndGet calls with correct namespace |
| lazy eviction — matching fault type | Fault for node not in graph calls store.remove |
| lazy eviction — non-matching fault type | Fault for node not in graph still calls store.remove (guard reordering) |
| resetCount | Resets count; next faults start from 1 |
| default store | Existing tests pass without `.faultCountStore()` in builder |
| namespace required for custom store | Builder throws if custom store provided without namespace |

## Cross-Repo Impact

**casehub-ops** (Kubernetes, Deployment, IoT, Infra fault policies):
- No change needed — builder defaults to InMemoryFaultCountStore with auto-derived namespace
- Persistence opt-in: inject a persistent FaultCountStore via CDI, pass to builder with explicit namespace (future work, casehubio/casehub-desiredstate#86)
- Reset-on-recovery opt-in: implement ReconciliationListener and call `delegate.resetCount(...)` (separate concern)

**examples/pipeline/ — ProvisionEscalationFaultPolicy:**
`ProvisionEscalationFaultPolicy` uses the same `ConcurrentHashMap<NodeId, Integer>` pattern
with the same three bugs (no tenant isolation, no persistence, no eviction). It has different
semantics (three-tier escalation with AI/human review node creation) that don't map to
ThresholdFaultPolicy composition, but it could use `FaultCountStore` directly for its counting.
Follow-up: casehubio/casehub-desiredstate#87.

**No other repos affected.** The API change is additive.

## CLAUDE.md Updates

- Add `FaultCountStore` to Core SPIs table
- Add `InMemoryFaultCountStore` to Core Runtime Types table
- Update `ThresholdFaultPolicy` description to mention store delegation and `resetCount`
