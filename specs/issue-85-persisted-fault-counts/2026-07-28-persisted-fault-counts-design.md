# Persisted Fault Counts for ThresholdFaultPolicy

**Issue:** casehubio/casehub-desiredstate#85
**Date:** 2026-07-28
**Status:** Approved

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
    int incrementAndGet(String tenancyId, NodeId nodeId);
    int getCount(String tenancyId, NodeId nodeId);
    void reset(String tenancyId, NodeId nodeId);
    void remove(String tenancyId, NodeId nodeId);
    void evict(String tenancyId, Set<NodeId> retainedNodes);
}
```

| Method | Purpose |
|--------|---------|
| `incrementAndGet` | Atomically increment and return the new count |
| `getCount` | Read without side effects; returns 0 if absent |
| `reset` | Set count to zero; `getCount` returns 0, next `incrementAndGet` returns 1 |
| `remove` | Delete entry entirely |
| `evict` | Bulk removal: delete all entries for tenancyId whose nodeId is NOT in retainedNodes |

## InMemoryFaultCountStore

In `api/` (`io.casehub.desiredstate.api`). Default implementation.

- `ConcurrentHashMap` with private `record Key(String tenancyId, NodeId nodeId)` composite key
- Thread-safe, matches current ThresholdFaultPolicy behavior
- `evict` iterates stored keys for the given tenancyId, removes those not in the retained set

## ThresholdFaultPolicy Changes

Four changes:

### 1. Builder accepts FaultCountStore

```java
public Builder faultCountStore(FaultCountStore store) { ... }
```

Defaults to `new InMemoryFaultCountStore()` if not set. Existing callers (casehub-ops policies)
continue working unchanged.

### 2. onFault uses the store

Replaces `faultCounts.merge(event.node(), 1, Integer::sum)` with
`store.incrementAndGet(tenancyId, event.node())`.

Fixes tenant isolation — counts now keyed by `(tenancyId, nodeId)`.

### 3. Lazy eviction

When `onFault` finds the node absent from the graph (the `node == null` early return),
it calls `store.remove(tenancyId, event.node())` before returning. Counts for removed nodes
are cleaned up on next fault event.

### 4. Public resetCount method

```java
public void resetCount(String tenancyId, NodeId nodeId) {
    store.reset(tenancyId, nodeId);
}
```

Wrapping policies that want reset-on-recovery call this when they detect recovery.
ThresholdFaultPolicy itself never calls it.

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
| custom store via builder | Injected store receives incrementAndGet calls |
| lazy eviction | Fault for node not in graph calls store.remove |
| resetCount | Resets count; next faults start from 1 |
| default store | Existing tests pass without `.faultCountStore()` in builder |

## Cross-Repo Impact

**casehub-ops** (Kubernetes, Deployment, IoT, Infra fault policies):
- No change needed — builder defaults to InMemoryFaultCountStore
- Persistence opt-in: inject a persistent FaultCountStore via CDI and pass to builder (future work)
- Reset-on-recovery opt-in: implement ReconciliationListener and call `delegate.resetCount(...)` (separate concern)

**No other repos affected.** The API change is additive.

## CLAUDE.md Updates

- Add `FaultCountStore` to Core SPIs table
- Add `InMemoryFaultCountStore` to Core Runtime Types table
- Update `ThresholdFaultPolicy` description to mention store delegation and `resetCount`
