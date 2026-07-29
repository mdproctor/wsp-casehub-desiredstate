# Persisted Fault Counts Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #85 — feat: persisted fault counts for ThresholdFaultPolicy
**Issue group:** #85

**Goal:** Extract ThresholdFaultPolicy's in-memory fault counting into a pluggable
FaultCountStore SPI with namespace-scoped, tenant-isolated keys, lazy eviction,
and a public reset mechanism.

**Architecture:** FaultCountStore SPI and InMemoryFaultCountStore live in `api/`
(pure Java, no CDI). ThresholdFaultPolicy delegates all counting to the store via
a builder-injected instance. Namespace isolates per-policy counts within a shared
store. Lazy eviction cleans up on fault events for removed nodes. resetCount enables
external recovery-reset integration.

**Tech Stack:** Java 21, JUnit 5, AssertJ

## Global Constraints

- ThresholdFaultPolicy and all new types remain in `api/` (pure Java, no CDI)
- Builder backward compatibility: existing callers without `.faultCountStore()` work unchanged
- FaultCountStore keys: `(namespace, tenancyId, nodeId)` — all three dimensions required
- Thread-safe: InMemoryFaultCountStore must be safe for concurrent access

---

### Task 1: FaultCountStore SPI + InMemoryFaultCountStore

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/FaultCountStore.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/InMemoryFaultCountStore.java`
- Create: `api/src/test/java/io/casehub/desiredstate/api/InMemoryFaultCountStoreTest.java`

**Interfaces:**
- Consumes: `NodeId` from `io.casehub.desiredstate.api`
- Produces:
  - `FaultCountStore` interface — `incrementAndGet(String namespace, String tenancyId, NodeId nodeId) → int`, `getCount(String namespace, String tenancyId, NodeId nodeId) → int`, `reset(String namespace, String tenancyId, NodeId nodeId) → void`, `remove(String namespace, String tenancyId, NodeId nodeId) → void`, `evict(String namespace, String tenancyId, Set<NodeId> retainedNodes) → void`
  - `InMemoryFaultCountStore` class — default implementation of `FaultCountStore`

- [ ] **Step 1: Write InMemoryFaultCountStoreTest — incrementAndGet**

```java
package io.casehub.desiredstate.api;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class InMemoryFaultCountStoreTest {

    private InMemoryFaultCountStore store;

    @BeforeEach
    void setUp() {
        store = new InMemoryFaultCountStore();
    }

    @Test
    void incrementAndGet_returnsSequentialCounts() {
        assertThat(store.incrementAndGet("ns", "t1", NodeId.of("n1"))).isEqualTo(1);
        assertThat(store.incrementAndGet("ns", "t1", NodeId.of("n1"))).isEqualTo(2);
        assertThat(store.incrementAndGet("ns", "t1", NodeId.of("n1"))).isEqualTo(3);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn test -f api/pom.xml -Dtest=InMemoryFaultCountStoreTest#incrementAndGet_returnsSequentialCounts -pl api`
Expected: FAIL — class does not exist

- [ ] **Step 3: Create FaultCountStore interface**

Use `ide_create_file` to create `api/src/main/java/io/casehub/desiredstate/api/FaultCountStore.java`:

```java
package io.casehub.desiredstate.api;

import java.util.Set;

public interface FaultCountStore {
    int incrementAndGet(String namespace, String tenancyId, NodeId nodeId);
    int getCount(String namespace, String tenancyId, NodeId nodeId);
    void reset(String namespace, String tenancyId, NodeId nodeId);
    void remove(String namespace, String tenancyId, NodeId nodeId);
    void evict(String namespace, String tenancyId, Set<NodeId> retainedNodes);
}
```

- [ ] **Step 4: Create InMemoryFaultCountStore**

Use `ide_create_file` to create `api/src/main/java/io/casehub/desiredstate/api/InMemoryFaultCountStore.java`:

```java
package io.casehub.desiredstate.api;

import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

public class InMemoryFaultCountStore implements FaultCountStore {

    private record Key(String namespace, String tenancyId, NodeId nodeId) {}

    private final ConcurrentHashMap<Key, Integer> counts = new ConcurrentHashMap<>();

    @Override
    public int incrementAndGet(String namespace, String tenancyId, NodeId nodeId) {
        return counts.merge(new Key(namespace, tenancyId, nodeId), 1, Integer::sum);
    }

    @Override
    public int getCount(String namespace, String tenancyId, NodeId nodeId) {
        return counts.getOrDefault(new Key(namespace, tenancyId, nodeId), 0);
    }

    @Override
    public void reset(String namespace, String tenancyId, NodeId nodeId) {
        counts.put(new Key(namespace, tenancyId, nodeId), 0);
    }

    @Override
    public void remove(String namespace, String tenancyId, NodeId nodeId) {
        counts.remove(new Key(namespace, tenancyId, nodeId));
    }

    @Override
    public void evict(String namespace, String tenancyId, Set<NodeId> retainedNodes) {
        counts.keySet().removeIf(key ->
                key.namespace().equals(namespace)
                        && key.tenancyId().equals(tenancyId)
                        && !retainedNodes.contains(key.nodeId()));
    }
}
```

- [ ] **Step 5: Run incrementAndGet test to verify it passes**

Run: `/opt/homebrew/bin/mvn test -f api/pom.xml -Dtest=InMemoryFaultCountStoreTest#incrementAndGet_returnsSequentialCounts`
Expected: PASS

- [ ] **Step 6: Add remaining tests**

Add these tests to `InMemoryFaultCountStoreTest`:

```java
@Test
void getCount_returnsZeroForAbsent() {
    assertThat(store.getCount("ns", "t1", NodeId.of("n1"))).isEqualTo(0);
}

@Test
void reset_setsCountToZero() {
    store.incrementAndGet("ns", "t1", NodeId.of("n1"));
    store.incrementAndGet("ns", "t1", NodeId.of("n1"));
    store.reset("ns", "t1", NodeId.of("n1"));
    assertThat(store.getCount("ns", "t1", NodeId.of("n1"))).isEqualTo(0);
    assertThat(store.incrementAndGet("ns", "t1", NodeId.of("n1"))).isEqualTo(1);
}

@Test
void remove_deletesEntry() {
    store.incrementAndGet("ns", "t1", NodeId.of("n1"));
    store.incrementAndGet("ns", "t1", NodeId.of("n1"));
    store.remove("ns", "t1", NodeId.of("n1"));
    assertThat(store.getCount("ns", "t1", NodeId.of("n1"))).isEqualTo(0);
    assertThat(store.incrementAndGet("ns", "t1", NodeId.of("n1"))).isEqualTo(1);
}

@Test
void evict_removesEntriesNotInRetainedSet() {
    store.incrementAndGet("ns", "t1", NodeId.of("a"));
    store.incrementAndGet("ns", "t1", NodeId.of("b"));
    store.incrementAndGet("ns", "t1", NodeId.of("c"));

    store.evict("ns", "t1", Set.of(NodeId.of("a"), NodeId.of("c")));

    assertThat(store.getCount("ns", "t1", NodeId.of("a"))).isEqualTo(1);
    assertThat(store.getCount("ns", "t1", NodeId.of("b"))).isEqualTo(0);
    assertThat(store.getCount("ns", "t1", NodeId.of("c"))).isEqualTo(1);
}

@Test
void tenantIsolation() {
    store.incrementAndGet("ns", "tenant-a", NodeId.of("n1"));
    store.incrementAndGet("ns", "tenant-a", NodeId.of("n1"));
    store.incrementAndGet("ns", "tenant-b", NodeId.of("n1"));

    assertThat(store.getCount("ns", "tenant-a", NodeId.of("n1"))).isEqualTo(2);
    assertThat(store.getCount("ns", "tenant-b", NodeId.of("n1"))).isEqualTo(1);
}

@Test
void namespaceIsolation() {
    store.incrementAndGet("policy-a", "t1", NodeId.of("n1"));
    store.incrementAndGet("policy-a", "t1", NodeId.of("n1"));
    store.incrementAndGet("policy-b", "t1", NodeId.of("n1"));

    assertThat(store.getCount("policy-a", "t1", NodeId.of("n1"))).isEqualTo(2);
    assertThat(store.getCount("policy-b", "t1", NodeId.of("n1"))).isEqualTo(1);
}

@Test
void evict_scopedToNamespaceAndTenancy() {
    store.incrementAndGet("ns-a", "t1", NodeId.of("n1"));
    store.incrementAndGet("ns-b", "t1", NodeId.of("n1"));
    store.incrementAndGet("ns-a", "t2", NodeId.of("n1"));

    store.evict("ns-a", "t1", Set.of());

    assertThat(store.getCount("ns-a", "t1", NodeId.of("n1"))).isEqualTo(0);
    assertThat(store.getCount("ns-b", "t1", NodeId.of("n1"))).isEqualTo(1);
    assertThat(store.getCount("ns-a", "t2", NodeId.of("n1"))).isEqualTo(1);
}
```

- [ ] **Step 7: Run all InMemoryFaultCountStoreTest tests**

Run: `/opt/homebrew/bin/mvn test -f api/pom.xml -Dtest=InMemoryFaultCountStoreTest`
Expected: ALL PASS

- [ ] **Step 8: Verify with ide_diagnostics**

Run `ide_diagnostics` on both new files to confirm no compilation errors.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/desiredstate add api/src/main/java/io/casehub/desiredstate/api/FaultCountStore.java api/src/main/java/io/casehub/desiredstate/api/InMemoryFaultCountStore.java api/src/test/java/io/casehub/desiredstate/api/InMemoryFaultCountStoreTest.java
git -C /Users/mdproctor/claude/casehub/desiredstate commit -m "feat(#85): FaultCountStore SPI and InMemoryFaultCountStore"
```

---

### Task 2: ThresholdFaultPolicy refactoring

**Files:**
- Modify: `api/src/main/java/io/casehub/desiredstate/api/ThresholdFaultPolicy.java`
- Modify: `runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java`

**Interfaces:**
- Consumes: `FaultCountStore`, `InMemoryFaultCountStore` from Task 1
- Produces:
  - `ThresholdFaultPolicy.Builder.faultCountStore(FaultCountStore) → Builder`
  - `ThresholdFaultPolicy.Builder.namespace(String) → Builder`
  - `ThresholdFaultPolicy.resetCount(String tenancyId, NodeId nodeId) → void`

- [ ] **Step 1: Write failing test — tenant isolation**

Add to `ThresholdFaultPolicyTest`:

```java
@Test
void tenantIsolation_sameFaultCountedIndependently() {
    var policy = policyWithThreshold(2);
    var graph = graphWith("n1", TARGET);
    var event = new FaultEvent(NodeId.of("n1"), FaultType.PROVISION_FAILED, "fail");

    policy.onFault("tenant-a", event, graph, EMPTY_ACTUAL);
    assertThat(policy.onFault("tenant-b", event, graph, EMPTY_ACTUAL)).isEmpty();
    assertThat(policy.onFault("tenant-a", event, graph, EMPTY_ACTUAL)).hasSize(1);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn test -f runtime/pom.xml -Dtest=ThresholdFaultPolicyTest#tenantIsolation_sameFaultCountedIndependently`
Expected: FAIL — tenant-a count is 2, but tenant-b incremented the shared count to 2 as well, so tenant-a at call 2 is count=3 and triggers the action for tenant-b too.

Actually wait — the current code ignores tenancyId entirely. The ConcurrentHashMap keys by NodeId only. So:
- `onFault("tenant-a", ...)` → count for n1 = 1
- `onFault("tenant-b", ...)` → count for n1 = 2 (threshold=2, triggers!) — test expects empty but gets size 1

The test should fail with: assertion on `onFault("tenant-b", ...)` returns size 1 instead of empty.

- [ ] **Step 3: Write failing test — custom store via builder**

```java
@Test
void customStore_receivesIncrementCalls() {
    var store = new InMemoryFaultCountStore();
    var policy = ThresholdFaultPolicy.builder()
            .faultTypes(Set.of(FaultType.PROVISION_FAILED))
            .threshold(2)
            .action(FaultPolicy.addReviewNode(REVIEW,
                    (event, current) -> new TestReviewSpec(event.node(), event.detail())))
            .faultCountStore(store)
            .namespace("test-policy")
            .build();
    var graph = graphWith("n1", TARGET);
    var event = new FaultEvent(NodeId.of("n1"), FaultType.PROVISION_FAILED, "fail");

    policy.onFault("t1", event, graph, EMPTY_ACTUAL);

    assertThat(store.getCount("test-policy", "t1", NodeId.of("n1"))).isEqualTo(1);
}
```

- [ ] **Step 4: Write failing test — lazy eviction for matching fault type**

```java
@Test
void lazyEviction_matchingFaultType_removesCount() {
    var store = new InMemoryFaultCountStore();
    var policy = ThresholdFaultPolicy.builder()
            .faultTypes(Set.of(FaultType.PROVISION_FAILED))
            .threshold(3)
            .action((t, e, g, a) -> List.of())
            .faultCountStore(store)
            .namespace("test")
            .build();
    var graph = graphWith("n1", TARGET);
    var event = new FaultEvent(NodeId.of("n1"), FaultType.PROVISION_FAILED, "fail");

    policy.onFault("t1", event, graph, EMPTY_ACTUAL);
    assertThat(store.getCount("test", "t1", NodeId.of("n1"))).isEqualTo(1);

    var emptyGraph = graphFactory.of(List.of(), List.of());
    policy.onFault("t1", event, emptyGraph, EMPTY_ACTUAL);

    assertThat(store.getCount("test", "t1", NodeId.of("n1"))).isEqualTo(0);
}
```

- [ ] **Step 5: Write failing test — lazy eviction for non-matching fault type**

```java
@Test
void lazyEviction_nonMatchingFaultType_stillRemovesCount() {
    var store = new InMemoryFaultCountStore();
    var policy = ThresholdFaultPolicy.builder()
            .faultTypes(Set.of(FaultType.PROVISION_FAILED))
            .threshold(3)
            .action((t, e, g, a) -> List.of())
            .faultCountStore(store)
            .namespace("test")
            .build();
    var graph = graphWith("n1", TARGET);
    var provisionEvent = new FaultEvent(NodeId.of("n1"), FaultType.PROVISION_FAILED, "fail");

    policy.onFault("t1", provisionEvent, graph, EMPTY_ACTUAL);
    assertThat(store.getCount("test", "t1", NodeId.of("n1"))).isEqualTo(1);

    var emptyGraph = graphFactory.of(List.of(), List.of());
    var degradedEvent = new FaultEvent(NodeId.of("n1"), FaultType.NODE_DEGRADED, "drift");
    policy.onFault("t1", degradedEvent, emptyGraph, EMPTY_ACTUAL);

    assertThat(store.getCount("test", "t1", NodeId.of("n1"))).isEqualTo(0);
}
```

- [ ] **Step 6: Write failing test — resetCount**

```java
@Test
void resetCount_resetsAndNextFaultsStartFromOne() {
    var policy = policyWithThreshold(3);
    var graph = graphWith("n1", TARGET);
    var event = new FaultEvent(NodeId.of("n1"), FaultType.PROVISION_FAILED, "fail");

    policy.onFault("t1", event, graph, EMPTY_ACTUAL);
    policy.onFault("t1", event, graph, EMPTY_ACTUAL);
    policy.resetCount("t1", NodeId.of("n1"));

    policy.onFault("t1", event, graph, EMPTY_ACTUAL);
    policy.onFault("t1", event, graph, EMPTY_ACTUAL);
    assertThat(policy.onFault("t1", event, graph, EMPTY_ACTUAL))
            .as("Third fault after reset triggers action")
            .hasSize(1);
}
```

- [ ] **Step 7: Write failing test — namespace required for custom store**

```java
@Test
void builder_requiresNamespaceForCustomStore() {
    assertThatThrownBy(() -> ThresholdFaultPolicy.builder()
            .faultTypes(Set.of(FaultType.PROVISION_FAILED))
            .threshold(1)
            .action((t, e, g, a) -> List.of())
            .faultCountStore(new InMemoryFaultCountStore())
            .build())
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("namespace");
}
```

- [ ] **Step 8: Run all new tests to verify they fail**

Run: `/opt/homebrew/bin/mvn test -f runtime/pom.xml -Dtest=ThresholdFaultPolicyTest`
Expected: New tests FAIL (no `faultCountStore`/`namespace`/`resetCount` on builder/policy), existing tests PASS.

- [ ] **Step 9: Implement ThresholdFaultPolicy changes**

Use `ide_edit_member` to replace the full class. Changes:
1. Replace `ConcurrentHashMap<NodeId, Integer> faultCounts` field with `FaultCountStore store` and `String namespace`
2. Constructor reads `store` and `namespace` from builder (with defaults)
3. Add `deriveNamespace(Set<FaultType>)` private static method
4. Reorder guards in `onFault`: move `node == null` above `faultTypes` check, call `store.remove(...)` before return
5. Replace `faultCounts.merge(...)` with `store.incrementAndGet(namespace, tenancyId, ...)`
6. Add `resetCount(String tenancyId, NodeId nodeId)` public method
7. Add `faultCountStore(FaultCountStore)` and `namespace(String)` to Builder
8. Add builder validation: namespace required when custom store provided

Full implementation for `ThresholdFaultPolicy`:

```java
package io.casehub.desiredstate.api;

import java.util.List;
import java.util.Objects;
import java.util.Set;
import java.util.stream.Collectors;

public class ThresholdFaultPolicy implements FaultPolicy {

    private final Set<FaultType> faultTypes;
    private final Set<NodeType> nodeTypes;
    private final Set<NodeType> ignoreTypes;
    private final int threshold;
    private final FaultPolicy action;
    private final FaultCountStore store;
    private final String namespace;

    private ThresholdFaultPolicy(Builder builder) {
        this.faultTypes = Set.copyOf(builder.faultTypes);
        this.nodeTypes = builder.nodeTypes == null ? Set.of() : Set.copyOf(builder.nodeTypes);
        this.ignoreTypes = builder.ignoreTypes == null ? Set.of() : Set.copyOf(builder.ignoreTypes);
        this.threshold = builder.threshold;
        this.action = builder.action;
        this.store = builder.store != null ? builder.store : new InMemoryFaultCountStore();
        this.namespace = builder.namespace != null ? builder.namespace : deriveNamespace(this.faultTypes);
    }

    private static String deriveNamespace(Set<FaultType> faultTypes) {
        return faultTypes.stream()
                .map(FaultType::name)
                .sorted()
                .collect(Collectors.joining(","));
    }

    public static Builder builder() {
        return new Builder();
    }

    @Override
    public List<GraphMutation> onFault(String tenancyId, FaultEvent event,
                                        DesiredStateGraph current, ActualState actual) {
        DesiredNode node = current.nodes().get(event.node());

        if (node != null && ignoreTypes.contains(node.type())) {
            return List.of();
        }

        if (node == null) {
            store.remove(namespace, tenancyId, event.node());
            return List.of();
        }

        if (!faultTypes.contains(event.type())) {
            return List.of();
        }

        if (!nodeTypes.isEmpty() && !nodeTypes.contains(node.type())) {
            return List.of();
        }

        int count = store.incrementAndGet(namespace, tenancyId, event.node());
        if (count < threshold) {
            return List.of();
        }

        return action.onFault(tenancyId, event, current, actual);
    }

    public void resetCount(String tenancyId, NodeId nodeId) {
        store.reset(namespace, tenancyId, nodeId);
    }

    public static class Builder {
        private Set<FaultType> faultTypes;
        private Set<NodeType> nodeTypes;
        private Set<NodeType> ignoreTypes;
        private int threshold = 3;
        private FaultPolicy action;
        private FaultCountStore store;
        private String namespace;

        public Builder faultTypes(Set<FaultType> faultTypes) { this.faultTypes = faultTypes; return this; }
        public Builder nodeTypes(Set<NodeType> nodeTypes) { this.nodeTypes = nodeTypes; return this; }
        public Builder ignoreTypes(Set<NodeType> ignoreTypes) { this.ignoreTypes = ignoreTypes; return this; }
        public Builder threshold(int threshold) { this.threshold = threshold; return this; }
        public Builder action(FaultPolicy action) { this.action = action; return this; }
        public Builder faultCountStore(FaultCountStore store) { this.store = store; return this; }
        public Builder namespace(String namespace) { this.namespace = namespace; return this; }

        public ThresholdFaultPolicy build() {
            Objects.requireNonNull(faultTypes, "faultTypes is required");
            Objects.requireNonNull(action, "action is required");
            if (threshold < 1) {
                throw new IllegalArgumentException("threshold must be >= 1, got " + threshold);
            }
            if (store != null && namespace == null) {
                throw new IllegalArgumentException("namespace is required when a custom faultCountStore is provided");
            }
            return new ThresholdFaultPolicy(this);
        }
    }
}
```

- [ ] **Step 10: Run all ThresholdFaultPolicyTest tests**

Run: `/opt/homebrew/bin/mvn test -f runtime/pom.xml -Dtest=ThresholdFaultPolicyTest`
Expected: ALL PASS (new tests + existing tests)

- [ ] **Step 11: Verify with ide_diagnostics**

Run `ide_diagnostics` on `ThresholdFaultPolicy.java` to confirm no compilation errors.

- [ ] **Step 12: Run full api + runtime build**

Run: `/opt/homebrew/bin/mvn test -f api/pom.xml && /opt/homebrew/bin/mvn test -f runtime/pom.xml`
Expected: ALL PASS

- [ ] **Step 13: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/desiredstate add api/src/main/java/io/casehub/desiredstate/api/ThresholdFaultPolicy.java runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java
git -C /Users/mdproctor/claude/casehub/desiredstate commit -m "feat(#85): refactor ThresholdFaultPolicy to use FaultCountStore SPI"
```

---

### Task 3: Full build verification + CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: All types from Tasks 1 and 2
- Produces: Updated CLAUDE.md documentation

- [ ] **Step 1: Run full project build**

Run: `/opt/homebrew/bin/mvn --batch-mode install`
Expected: ALL modules PASS (api, runtime, testing, engine-adapter, work-adapter, ras-adapter, all examples)

- [ ] **Step 2: Update CLAUDE.md — Core SPIs table**

Add row to the Core SPIs table:

```
| `FaultCountStore` | `incrementAndGet(namespace, tenancyId, nodeId) → int`, `getCount(...)`, `reset(...)`, `remove(...)`, `evict(namespace, tenancyId, retainedNodes)` | Pluggable fault count storage — namespace-scoped, tenant-isolated |
```

- [ ] **Step 3: Update CLAUDE.md — Core Runtime Types table**

Add row:

```
| `InMemoryFaultCountStore` | Default `FaultCountStore` — `ConcurrentHashMap` with `(namespace, tenancyId, nodeId)` composite key. Thread-safe. In `api/` (builder default, not CDI-managed) |
```

- [ ] **Step 4: Update CLAUDE.md — ThresholdFaultPolicy description**

Update the existing `ThresholdFaultPolicy` entry in Core Runtime Types:

```
| `ThresholdFaultPolicy` | Reusable `FaultPolicy` (api module) — counts faults per node via pluggable `FaultCountStore` SPI, delegates to configured `FaultPolicy` at threshold. Builder: faultTypes, nodeTypes, ignoreTypes, threshold, action, faultCountStore, namespace. `resetCount(tenancyId, nodeId)` for external recovery-reset. In-memory `InMemoryFaultCountStore` default. Lazy eviction on fault for removed nodes (#85 tracks persistence) |
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/desiredstate add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/desiredstate commit -m "docs(#85): update CLAUDE.md — FaultCountStore SPI, InMemoryFaultCountStore, ThresholdFaultPolicy changes"
```
