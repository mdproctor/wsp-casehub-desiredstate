# Drift Policy Layer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #160 — feat: drift policy layer — permitted drift exemptions with revert modes
**Issue group:** #160

**Goal:** Add a DriftPolicy SPI that lets domains declare permitted drift exemptions, preventing the reconciliation loop from re-provisioning exempt nodes while emitting observable events.

**Architecture:** New `DriftPolicy` SPI in `api/` evaluated inside `detectDrift()` before FaultEvent creation. `DriftPolicyEngine` in `runtime-core/` aggregates policies with first-EXEMPT-wins semantics. `ExemptionStore` SPI with `InMemoryExemptionStore` default provides pluggable exemption storage. Exempt nodes produce `NodeDriftExemptedData` CloudEvents instead of `NodeDriftedData`, and are excluded from the transition plan via a `Set<NodeId> exemptNodes` threaded through `TenantLoop.plan()` to `TransitionPlanner.plan()`.

**Tech Stack:** Java 21, Quarkus CDI, Spring Boot auto-configuration, JUnit 5, Awaitility

## Global Constraints

- Pre-release platform — breaking changes are acceptable
- All types in `api/` must be pure Java (no CDI, no Spring, no Quarkus dependencies)
- CDI wiring follows `RuntimeBeans` `@Produces` pattern (no bridge classes)
- Spring wiring goes in existing `DesiredStateRuntimeAutoConfiguration` (single-class convention)
- `@Priority` required on `DriftPolicy` implementations for deterministic ordering
- `ReconciliationCompletedData` backward-compatible constructors must be maintained
- `NodeDriftedData` semantic change is a documented breaking change

---

## Batch 1: API Types

### Task 1: RevertMode, RevertCondition, ExemptionSpec

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/RevertMode.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/RevertCondition.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/ExemptionSpec.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/api/RevertConditionTest.java`

**Interfaces:**
- Consumes: `NodeStatus` (existing enum in api/)
- Produces: `RevertMode` enum (`DURATION`, `SCHEDULE`, `STATUS_CHANGE`, `NEVER`), `RevertCondition` sealed interface with `mode()` default method and variants (`OnDuration`, `OnSchedule`, `OnStatusChange`, `Never`), `ExemptionSpec` record (`revertCondition`, `metadata`)

- [ ] **Step 1: Write failing tests for RevertCondition**

```java
package io.casehub.desiredstate.api;

import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.Set;
import static org.junit.jupiter.api.Assertions.*;

class RevertConditionTest {

    @Test
    void onDuration_modeIsDuration() {
        var condition = new RevertCondition.OnDuration(Duration.ofMinutes(30));
        assertEquals(RevertMode.DURATION, condition.mode());
        assertEquals(Duration.ofMinutes(30), condition.duration());
    }

    @Test
    void onDuration_rejectsNull() {
        assertThrows(NullPointerException.class,
            () -> new RevertCondition.OnDuration(null));
    }

    @Test
    void onSchedule_modeIsSchedule() {
        var condition = new RevertCondition.OnSchedule("0 9 * * 1-5");
        assertEquals(RevertMode.SCHEDULE, condition.mode());
        assertEquals("0 9 * * 1-5", condition.cronExpression());
    }

    @Test
    void onStatusChange_modeIsStatusChange() {
        var condition = new RevertCondition.OnStatusChange(
            Set.of(NodeStatus.PRESENT));
        assertEquals(RevertMode.STATUS_CHANGE, condition.mode());
        assertTrue(condition.triggerStatuses().contains(NodeStatus.PRESENT));
    }

    @Test
    void onStatusChange_defensiveCopy() {
        var mutable = new java.util.HashSet<>(Set.of(NodeStatus.PRESENT));
        var condition = new RevertCondition.OnStatusChange(mutable);
        mutable.add(NodeStatus.ABSENT);
        assertEquals(1, condition.triggerStatuses().size());
    }

    @Test
    void never_modeIsNever() {
        var condition = new RevertCondition.Never();
        assertEquals(RevertMode.NEVER, condition.mode());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=RevertConditionTest`
Expected: compilation failure — classes don't exist yet

- [ ] **Step 3: Implement RevertMode enum**

```java
package io.casehub.desiredstate.api;

public enum RevertMode { DURATION, SCHEDULE, STATUS_CHANGE, NEVER }
```

- [ ] **Step 4: Implement RevertCondition sealed interface**

```java
package io.casehub.desiredstate.api;

import java.time.Duration;
import java.util.Objects;
import java.util.Set;

public sealed interface RevertCondition {

    default RevertMode mode() {
        return switch (this) {
            case OnDuration d -> RevertMode.DURATION;
            case OnSchedule s -> RevertMode.SCHEDULE;
            case OnStatusChange e -> RevertMode.STATUS_CHANGE;
            case Never n -> RevertMode.NEVER;
        };
    }

    record OnDuration(Duration duration) implements RevertCondition {
        public OnDuration { Objects.requireNonNull(duration, "duration"); }
    }

    record OnSchedule(String cronExpression) implements RevertCondition {
        public OnSchedule { Objects.requireNonNull(cronExpression, "cronExpression"); }
    }

    record OnStatusChange(Set<NodeStatus> triggerStatuses) implements RevertCondition {
        public OnStatusChange {
            Objects.requireNonNull(triggerStatuses, "triggerStatuses");
            triggerStatuses = Set.copyOf(triggerStatuses);
        }
    }

    record Never() implements RevertCondition {}
}
```

- [ ] **Step 5: Implement ExemptionSpec record**

```java
package io.casehub.desiredstate.api;

import java.util.Map;
import java.util.Objects;

public record ExemptionSpec(RevertCondition revertCondition,
                            Map<String, String> metadata) {
    public ExemptionSpec {
        Objects.requireNonNull(revertCondition, "revertCondition");
        metadata = metadata != null ? Map.copyOf(metadata) : Map.of();
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=RevertConditionTest`
Expected: all 6 tests PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/RevertMode.java api/src/main/java/io/casehub/desiredstate/api/RevertCondition.java api/src/main/java/io/casehub/desiredstate/api/ExemptionSpec.java runtime/src/test/java/io/casehub/desiredstate/api/RevertConditionTest.java
git commit -m "feat(#160): RevertMode, RevertCondition, ExemptionSpec — drift exemption value types"
```

### Task 2: DriftPolicy SPI, DriftDecision, DriftContext, Exemption

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/DriftPolicy.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/DriftDecision.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/DriftContext.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/Exemption.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/api/ExemptionTest.java`

**Interfaces:**
- Consumes: `NodeId`, `NodeStatus`, `DesiredNode`, `DesiredStateGraph`, `ActualState`, `ExemptionSpec`, `RevertCondition` (all from api/)
- Produces: `DriftPolicy` interface, `DriftDecision` sealed interface, `DriftContext` record, `Exemption` record with `shouldRevert(Instant, NodeStatus)`

- [ ] **Step 1: Write failing tests for Exemption.shouldRevert()**

```java
package io.casehub.desiredstate.api;

import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.time.Instant;
import java.util.Map;
import java.util.Set;
import static org.junit.jupiter.api.Assertions.*;

class ExemptionTest {

    private static final NodeId NODE = NodeId.of("n1");
    private static final Instant NOW = Instant.parse("2026-10-01T12:00:00Z");

    @Test
    void shouldRevert_onDuration_notExpired() {
        var spec = new ExemptionSpec(
            new RevertCondition.OnDuration(Duration.ofMinutes(30)), Map.of());
        var exemption = new Exemption(NODE, spec, NOW, NOW.plus(Duration.ofMinutes(30)));
        assertFalse(exemption.shouldRevert(NOW.plusSeconds(60), NodeStatus.DRIFTED));
    }

    @Test
    void shouldRevert_onDuration_expired() {
        var spec = new ExemptionSpec(
            new RevertCondition.OnDuration(Duration.ofMinutes(30)), Map.of());
        var exemption = new Exemption(NODE, spec, NOW, NOW.plus(Duration.ofMinutes(30)));
        assertTrue(exemption.shouldRevert(NOW.plus(Duration.ofMinutes(31)), NodeStatus.DRIFTED));
    }

    @Test
    void shouldRevert_onStatusChange_statusMatches() {
        var spec = new ExemptionSpec(
            new RevertCondition.OnStatusChange(Set.of(NodeStatus.PRESENT)), Map.of());
        var exemption = new Exemption(NODE, spec, NOW, null);
        assertTrue(exemption.shouldRevert(NOW, NodeStatus.PRESENT));
    }

    @Test
    void shouldRevert_onStatusChange_statusDoesNotMatch() {
        var spec = new ExemptionSpec(
            new RevertCondition.OnStatusChange(Set.of(NodeStatus.PRESENT)), Map.of());
        var exemption = new Exemption(NODE, spec, NOW, null);
        assertFalse(exemption.shouldRevert(NOW, NodeStatus.DRIFTED));
    }

    @Test
    void shouldRevert_never() {
        var spec = new ExemptionSpec(new RevertCondition.Never(), Map.of());
        var exemption = new Exemption(NODE, spec, NOW, null);
        assertFalse(exemption.shouldRevert(NOW.plus(Duration.ofDays(365)), NodeStatus.DRIFTED));
    }

    @Test
    void shouldRevert_expiresAt_overridesDuration() {
        var spec = new ExemptionSpec(
            new RevertCondition.OnDuration(Duration.ofHours(1)), Map.of());
        var exemption = new Exemption(NODE, spec, NOW, NOW.plusSeconds(10));
        assertTrue(exemption.shouldRevert(NOW.plusSeconds(11), NodeStatus.DRIFTED));
    }

    @Test
    void rejectsNullNodeId() {
        assertThrows(NullPointerException.class,
            () -> new Exemption(null, new ExemptionSpec(new RevertCondition.Never(), Map.of()), NOW, null));
    }

    @Test
    void rejectsNullSpec() {
        assertThrows(NullPointerException.class,
            () -> new Exemption(NODE, null, NOW, null));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=ExemptionTest`
Expected: compilation failure — classes don't exist yet

- [ ] **Step 3: Implement DriftDecision**

```java
package io.casehub.desiredstate.api;

import java.util.Objects;

public sealed interface DriftDecision {
    record Reconcile() implements DriftDecision {}
    record Exempt(ExemptionSpec spec) implements DriftDecision {
        public Exempt { Objects.requireNonNull(spec, "spec"); }
    }

    static DriftDecision reconcile() { return new Reconcile(); }
    static DriftDecision exempt(ExemptionSpec spec) { return new Exempt(spec); }
}
```

- [ ] **Step 4: Implement DriftContext**

```java
package io.casehub.desiredstate.api;

import java.util.Objects;

public record DriftContext(String tenancyId, DesiredStateGraph graph,
                           ActualState actual) {
    public DriftContext {
        Objects.requireNonNull(tenancyId, "tenancyId");
        Objects.requireNonNull(graph, "graph");
        Objects.requireNonNull(actual, "actual");
    }
}
```

- [ ] **Step 5: Implement DriftPolicy**

```java
package io.casehub.desiredstate.api;

public interface DriftPolicy {
    DriftDecision evaluate(NodeId nodeId, NodeStatus status,
                           DesiredNode node, DriftContext context);
}
```

- [ ] **Step 6: Implement Exemption**

```java
package io.casehub.desiredstate.api;

import java.time.Duration;
import java.time.Instant;
import java.util.Objects;

public record Exemption(NodeId nodeId, ExemptionSpec spec,
                        Instant grantedAt, Instant expiresAt) {
    public Exemption {
        Objects.requireNonNull(nodeId, "nodeId");
        Objects.requireNonNull(spec, "spec");
        Objects.requireNonNull(grantedAt, "grantedAt");
    }

    public boolean shouldRevert(Instant now, NodeStatus currentStatus) {
        if (expiresAt != null && now.isAfter(expiresAt)) {
            return true;
        }
        return switch (spec.revertCondition()) {
            case RevertCondition.OnDuration d ->
                now.isAfter(grantedAt.plus(d.duration()));
            case RevertCondition.OnSchedule s -> false;
            case RevertCondition.OnStatusChange sc ->
                sc.triggerStatuses().contains(currentStatus);
            case RevertCondition.Never n -> false;
        };
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=ExemptionTest`
Expected: all 8 tests PASS

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/DriftPolicy.java api/src/main/java/io/casehub/desiredstate/api/DriftDecision.java api/src/main/java/io/casehub/desiredstate/api/DriftContext.java api/src/main/java/io/casehub/desiredstate/api/Exemption.java runtime/src/test/java/io/casehub/desiredstate/api/ExemptionTest.java
git commit -m "feat(#160): DriftPolicy SPI, DriftDecision, DriftContext, Exemption — core drift policy types"
```

### Task 3: ExemptionStore SPI, InMemoryExemptionStore, NodeDriftExemptedData, event type constant

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/ExemptionStore.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/InMemoryExemptionStore.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/NodeDriftExemptedData.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/DesiredStateEventTypes.java` — add `NODE_DRIFT_EXEMPTED`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/ReconciliationCompletedData.java` — add `exemptedCount`
- Test: `runtime/src/test/java/io/casehub/desiredstate/api/InMemoryExemptionStoreTest.java`

**Interfaces:**
- Consumes: `NodeId`, `Exemption` (from Task 2)
- Produces: `ExemptionStore` interface, `InMemoryExemptionStore` class, `NodeDriftExemptedData` record, `NODE_DRIFT_EXEMPTED` constant, `ReconciliationCompletedData.exemptedCount` field

- [ ] **Step 1: Write failing tests for InMemoryExemptionStore**

```java
package io.casehub.desiredstate.api;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.time.Instant;
import java.util.Map;
import java.util.Set;
import static org.junit.jupiter.api.Assertions.*;

class InMemoryExemptionStoreTest {

    private InMemoryExemptionStore store;
    private static final Instant NOW = Instant.parse("2026-10-01T12:00:00Z");

    @BeforeEach
    void setUp() { store = new InMemoryExemptionStore(); }

    private Exemption exemption(String nodeId) {
        return new Exemption(NodeId.of(nodeId),
            new ExemptionSpec(new RevertCondition.Never(), Map.of()),
            NOW, null);
    }

    @Test
    void grant_and_get() {
        store.grant("t1", NodeId.of("a"), exemption("a"));
        assertTrue(store.get("t1", NodeId.of("a")).isPresent());
    }

    @Test
    void get_absent() {
        assertTrue(store.get("t1", NodeId.of("a")).isEmpty());
    }

    @Test
    void revoke() {
        store.grant("t1", NodeId.of("a"), exemption("a"));
        store.revoke("t1", NodeId.of("a"));
        assertTrue(store.get("t1", NodeId.of("a")).isEmpty());
    }

    @Test
    void grant_overwrites() {
        var e1 = new Exemption(NodeId.of("a"),
            new ExemptionSpec(new RevertCondition.Never(), Map.of()), NOW, null);
        var e2 = new Exemption(NodeId.of("a"),
            new ExemptionSpec(new RevertCondition.Never(), Map.of("k", "v")), NOW, null);
        store.grant("t1", NodeId.of("a"), e1);
        store.grant("t1", NodeId.of("a"), e2);
        assertEquals("v", store.get("t1", NodeId.of("a")).orElseThrow().spec().metadata().get("k"));
    }

    @Test
    void getActive_returns_all() {
        store.grant("t1", NodeId.of("a"), exemption("a"));
        store.grant("t1", NodeId.of("b"), exemption("b"));
        assertEquals(2, store.getActive("t1").size());
    }

    @Test
    void getActive_isolates_tenants() {
        store.grant("t1", NodeId.of("a"), exemption("a"));
        store.grant("t2", NodeId.of("b"), exemption("b"));
        assertEquals(1, store.getActive("t1").size());
    }

    @Test
    void evict_removes_non_retained() {
        store.grant("t1", NodeId.of("a"), exemption("a"));
        store.grant("t1", NodeId.of("b"), exemption("b"));
        store.evict("t1", Set.of(NodeId.of("a")));
        assertTrue(store.get("t1", NodeId.of("a")).isPresent());
        assertTrue(store.get("t1", NodeId.of("b")).isEmpty());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=InMemoryExemptionStoreTest`
Expected: compilation failure

- [ ] **Step 3: Implement ExemptionStore interface**

```java
package io.casehub.desiredstate.api;

import java.util.Map;
import java.util.Optional;
import java.util.Set;

public interface ExemptionStore {
    void grant(String tenancyId, NodeId nodeId, Exemption exemption);
    void revoke(String tenancyId, NodeId nodeId);
    Optional<Exemption> get(String tenancyId, NodeId nodeId);
    Map<NodeId, Exemption> getActive(String tenancyId);
    void evict(String tenancyId, Set<NodeId> retainedNodes);
}
```

- [ ] **Step 4: Implement InMemoryExemptionStore**

```java
package io.casehub.desiredstate.api;

import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

public class InMemoryExemptionStore implements ExemptionStore {

    private record Key(String tenancyId, NodeId nodeId) {}

    private final ConcurrentHashMap<Key, Exemption> exemptions = new ConcurrentHashMap<>();

    @Override
    public void grant(String tenancyId, NodeId nodeId, Exemption exemption) {
        exemptions.put(new Key(tenancyId, nodeId), exemption);
    }

    @Override
    public void revoke(String tenancyId, NodeId nodeId) {
        exemptions.remove(new Key(tenancyId, nodeId));
    }

    @Override
    public Optional<Exemption> get(String tenancyId, NodeId nodeId) {
        return Optional.ofNullable(exemptions.get(new Key(tenancyId, nodeId)));
    }

    @Override
    public Map<NodeId, Exemption> getActive(String tenancyId) {
        return exemptions.entrySet().stream()
            .filter(e -> e.getKey().tenancyId().equals(tenancyId))
            .collect(Collectors.toMap(e -> e.getKey().nodeId(), Map.Entry::getValue));
    }

    @Override
    public void evict(String tenancyId, Set<NodeId> retainedNodes) {
        exemptions.keySet().removeIf(key ->
            key.tenancyId().equals(tenancyId) && !retainedNodes.contains(key.nodeId()));
    }
}
```

- [ ] **Step 5: Implement NodeDriftExemptedData**

```java
package io.casehub.desiredstate.api;

import java.util.Objects;

public record NodeDriftExemptedData(
        String tenancyId, String nodeId, String nodeType,
        String revertMode, String revertCondition,
        long graphVersion, String parentNodeId) {
    public NodeDriftExemptedData {
        Objects.requireNonNull(tenancyId, "tenancyId");
        Objects.requireNonNull(nodeId, "nodeId");
        Objects.requireNonNull(nodeType, "nodeType");
        Objects.requireNonNull(revertMode, "revertMode");
    }
}
```

- [ ] **Step 6: Add NODE_DRIFT_EXEMPTED to DesiredStateEventTypes**

Add after the existing `NODE_DRIFTED` constant:

```java
public static final String NODE_DRIFT_EXEMPTED =
    "io.casehub.desiredstate.node.drift.exempted";
```

- [ ] **Step 7: Add exemptedCount to ReconciliationCompletedData**

Add `int exemptedCount` as a new field to the canonical constructor. Update backward-compatible constructors to pass `0` for `exemptedCount`. Add validation `exemptedCount >= 0`.

- [ ] **Step 8: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=InMemoryExemptionStoreTest`
Expected: all 7 tests PASS

- [ ] **Step 9: Run full build to verify backward compatibility**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — backward-compatible constructors preserve existing callers

- [ ] **Step 10: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/ExemptionStore.java api/src/main/java/io/casehub/desiredstate/api/InMemoryExemptionStore.java api/src/main/java/io/casehub/desiredstate/api/NodeDriftExemptedData.java api/src/main/java/io/casehub/desiredstate/api/DesiredStateEventTypes.java api/src/main/java/io/casehub/desiredstate/api/ReconciliationCompletedData.java runtime/src/test/java/io/casehub/desiredstate/api/InMemoryExemptionStoreTest.java
git commit -m "feat(#160): ExemptionStore SPI, InMemoryExemptionStore, NodeDriftExemptedData, event types"
```

## Batch 2: Runtime Engine + Reconciliation Loop Integration

### Task 4: DriftPolicyEngine

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DriftPolicyEngine.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/DriftPolicyEngineTest.java`

**Interfaces:**
- Consumes: `DriftPolicy`, `DriftDecision`, `DriftContext`, `NodeId`, `NodeStatus`, `DesiredNode` (from api/)
- Produces: `DriftPolicyEngine` class with `evaluate()` method (first-EXEMPT-wins)

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class DriftPolicyEngineTest {

    private static final NodeId NODE = NodeId.of("n1");
    private static final NodeStatus STATUS = NodeStatus.DRIFTED;
    private static final DesiredNode DESIRED_NODE =
        new DesiredNode(NODE, new TestSpec(), HumanGating.NONE);
    private static final DriftContext CTX = new DriftContext(
        "tenant", new ImmutableDesiredStateGraph(Map.of(NODE, DESIRED_NODE), List.of()),
        new ActualState(Map.of(NODE, NodeStatus.DRIFTED)));

    @Test
    void emptyPolicies_returnsReconcile() {
        var engine = new DriftPolicyEngine(List.of());
        assertInstanceOf(DriftDecision.Reconcile.class,
            engine.evaluate(NODE, STATUS, DESIRED_NODE, CTX));
    }

    @Test
    void singlePolicy_exempt() {
        DriftPolicy alwaysExempt = (id, s, n, c) ->
            DriftDecision.exempt(new ExemptionSpec(new RevertCondition.Never(), Map.of()));
        var engine = new DriftPolicyEngine(List.of(alwaysExempt));
        assertInstanceOf(DriftDecision.Exempt.class,
            engine.evaluate(NODE, STATUS, DESIRED_NODE, CTX));
    }

    @Test
    void singlePolicy_reconcile() {
        DriftPolicy alwaysReconcile = (id, s, n, c) -> DriftDecision.reconcile();
        var engine = new DriftPolicyEngine(List.of(alwaysReconcile));
        assertInstanceOf(DriftDecision.Reconcile.class,
            engine.evaluate(NODE, STATUS, DESIRED_NODE, CTX));
    }

    @Test
    void firstExemptWins() {
        DriftPolicy reconcile = (id, s, n, c) -> DriftDecision.reconcile();
        DriftPolicy exempt = (id, s, n, c) ->
            DriftDecision.exempt(new ExemptionSpec(new RevertCondition.Never(), Map.of()));
        DriftPolicy shouldNotBeCalled = (id, s, n, c) -> {
            fail("Should not be called after EXEMPT");
            return DriftDecision.reconcile();
        };
        var engine = new DriftPolicyEngine(List.of(reconcile, exempt, shouldNotBeCalled));
        assertInstanceOf(DriftDecision.Exempt.class,
            engine.evaluate(NODE, STATUS, DESIRED_NODE, CTX));
    }

    @Test
    void allReconcile_returnsReconcile() {
        DriftPolicy r1 = (id, s, n, c) -> DriftDecision.reconcile();
        DriftPolicy r2 = (id, s, n, c) -> DriftDecision.reconcile();
        var engine = new DriftPolicyEngine(List.of(r1, r2));
        assertInstanceOf(DriftDecision.Reconcile.class,
            engine.evaluate(NODE, STATUS, DESIRED_NODE, CTX));
    }

    private static class TestSpec implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("test"); }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=DriftPolicyEngineTest`
Expected: compilation failure

- [ ] **Step 3: Implement DriftPolicyEngine**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.*;
import java.util.List;

public class DriftPolicyEngine {

    private final List<DriftPolicy> policies;

    public DriftPolicyEngine(List<DriftPolicy> policies) {
        this.policies = List.copyOf(policies);
    }

    public DriftDecision evaluate(NodeId nodeId, NodeStatus status,
                                  DesiredNode node, DriftContext context) {
        for (DriftPolicy policy : policies) {
            DriftDecision decision = policy.evaluate(nodeId, status, node, context);
            if (decision instanceof DriftDecision.Exempt) {
                return decision;
            }
        }
        return DriftDecision.reconcile();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=DriftPolicyEngineTest`
Expected: all 5 tests PASS

- [ ] **Step 5: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/desiredstate/runtime/DriftPolicyEngine.java runtime/src/test/java/io/casehub/desiredstate/runtime/DriftPolicyEngineTest.java
git commit -m "feat(#160): DriftPolicyEngine — first-EXEMPT-wins policy chain"
```

### Task 5: ReconciliationLoop + TransitionPlanner integration

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java` — add `driftPolicyEngine` and `exemptionStore` fields, modify `detectDrift()`, modify `reconcile()` and `reconcileTypes()`, modify `emitCycleEvents()`, update constructor, update Builder
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/TransitionPlanner.java` — add `Set<NodeId> exemptNodes` parameter, add exemption check in caller loop
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationEventEmitter.java` — add `nodeDriftExempted()` method
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ExemptionEvictionListener.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopDriftPolicyTest.java`

**Interfaces:**
- Consumes: `DriftPolicyEngine` (Task 4), `ExemptionStore`, `Exemption`, `DriftContext`, `DriftDecision`, `NodeDriftExemptedData`, `ExemptionSpec`, `RevertCondition` (all from api/)
- Produces: Modified `ReconciliationLoop` with drift policy support, modified `TransitionPlanner` with exempt nodes, `ExemptionEvictionListener`, `ReconciliationEventEmitter.nodeDriftExempted()`

- [ ] **Step 1: Write failing integration tests**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.testing.MockActualStateAdapter;
import io.casehub.desiredstate.testing.MockTransitionExecutor;
import io.casehub.desiredstate.testing.CannedEventSource;
import io.cloudevents.CloudEvent;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;

import static org.awaitility.Awaitility.await;
import static org.junit.jupiter.api.Assertions.*;

class ReconciliationLoopDriftPolicyTest {

    private DefaultDesiredStateGraphFactory factory;
    private MockActualStateAdapter actualAdapter;
    private MockTransitionExecutor testExecutor;
    private TransitionPlanner planner;
    private FaultPolicyEngine faultEngine;
    private CannedEventSource testEventSource;
    private InMemoryExemptionStore exemptionStore;
    private List<CloudEvent> emittedEvents;
    private ReconciliationLoop loop;

    private static final Duration TEST_DEBOUNCE = Duration.ofMillis(50);
    private static final Duration TEST_RESYNC = Duration.ofHours(1);
    private static final Duration AWAIT = Duration.ofSeconds(5);

    @BeforeEach
    void setUp() {
        factory = new DefaultDesiredStateGraphFactory();
        actualAdapter = new MockActualStateAdapter();
        actualAdapter.setHandledTypes(Set.of(NodeType.of("test")));
        testExecutor = new MockTransitionExecutor();
        planner = new TransitionPlanner();
        faultEngine = new FaultPolicyEngine(List.of());
        testEventSource = new CannedEventSource();
        exemptionStore = new InMemoryExemptionStore();
        emittedEvents = new CopyOnWriteArrayList<>();
    }

    @AfterEach
    void tearDown() {
        if (loop != null) loop.shutdown();
    }

    private ReconciliationLoop buildLoop(DriftPolicy... policies) {
        var driftEngine = new DriftPolicyEngine(List.of(policies));
        var adapterRouter = new DefaultActualStateAdapterRouter(List.of(actualAdapter));
        return ReconciliationLoop.builder(planner, testExecutor, adapterRouter, faultEngine, testEventSource::stream)
            .debounceWindow(TEST_DEBOUNCE).resyncInterval(TEST_RESYNC)
            .cloudEventSink(emittedEvents::add)
            .driftPolicyEngine(driftEngine)
            .exemptionStore(exemptionStore)
            .build();
    }

    private DesiredNode node(String id) {
        return new DesiredNode(NodeId.of(id), new TestSpec(), HumanGating.NONE);
    }

    @Test
    void noDriftPolicy_driftedNodeIsReconciled() {
        loop = buildLoop();
        DesiredNode a = node("a");
        var graph = factory.of(List.of(a), List.of());
        actualAdapter.setStatus(NodeId.of("a"), NodeStatus.DRIFTED);
        loop.start("t1", graph);
        await().atMost(AWAIT).until(() -> !testExecutor.executedPlans.isEmpty());
        var plan = testExecutor.executedPlans.get(0);
        assertFalse(plan.flatAdditions().isEmpty());
    }

    @Test
    void exemptPolicy_driftedNodeNotReprovisioned() {
        DriftPolicy exempt = (id, s, n, c) ->
            DriftDecision.exempt(new ExemptionSpec(new RevertCondition.Never(), Map.of()));
        loop = buildLoop(exempt);
        DesiredNode a = node("a");
        var graph = factory.of(List.of(a), List.of());
        actualAdapter.setStatus(NodeId.of("a"), NodeStatus.DRIFTED);
        loop.start("t1", graph);
        await().atMost(AWAIT).until(() -> emittedEvents.stream()
            .anyMatch(e -> e.getType().equals(DesiredStateEventTypes.RECONCILIATION_COMPLETED)));
        assertTrue(testExecutor.executedPlans.isEmpty() ||
            testExecutor.executedPlans.stream().allMatch(TransitionPlan::isEmpty));
    }

    @Test
    void exemptPolicy_emitsNodeDriftExemptedEvent() {
        DriftPolicy exempt = (id, s, n, c) ->
            DriftDecision.exempt(new ExemptionSpec(new RevertCondition.Never(), Map.of()));
        loop = buildLoop(exempt);
        DesiredNode a = node("a");
        var graph = factory.of(List.of(a), List.of());
        actualAdapter.setStatus(NodeId.of("a"), NodeStatus.DRIFTED);
        loop.start("t1", graph);
        await().atMost(AWAIT).until(() -> emittedEvents.stream()
            .anyMatch(e -> e.getType().equals(DesiredStateEventTypes.NODE_DRIFT_EXEMPTED)));
        assertTrue(emittedEvents.stream()
            .noneMatch(e -> e.getType().equals(DesiredStateEventTypes.NODE_DRIFTED)));
    }

    @Test
    void exemptionStoreGrant_exemptsWithoutPolicy() {
        loop = buildLoop();
        DesiredNode a = node("a");
        var graph = factory.of(List.of(a), List.of());
        exemptionStore.grant("t1", NodeId.of("a"), new Exemption(
            NodeId.of("a"), new ExemptionSpec(new RevertCondition.Never(), Map.of()),
            Instant.now(), null));
        actualAdapter.setStatus(NodeId.of("a"), NodeStatus.DRIFTED);
        loop.start("t1", graph);
        await().atMost(AWAIT).until(() -> emittedEvents.stream()
            .anyMatch(e -> e.getType().equals(DesiredStateEventTypes.NODE_DRIFT_EXEMPTED)));
        assertTrue(testExecutor.executedPlans.isEmpty() ||
            testExecutor.executedPlans.stream().allMatch(TransitionPlan::isEmpty));
    }

    private static class TestSpec implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("test"); }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=ReconciliationLoopDriftPolicyTest`
Expected: compilation failure — builder methods don't exist yet

- [ ] **Step 3: Add ReconciliationEventEmitter.nodeDriftExempted()**

Add method to `ReconciliationEventEmitter`:

```java
public CloudEvent nodeDriftExempted(NodeDriftExemptedData data) {
    return base(DesiredStateEventTypes.NODE_DRIFT_EXEMPTED)
            .withData("application/json", serialize(data))
            .build();
}
```

- [ ] **Step 4: Create ExemptionEvictionListener**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.*;
import java.util.Set;

public class ExemptionEvictionListener implements GlobalReconciliationListener {

    private final ExemptionStore store;

    public ExemptionEvictionListener(ExemptionStore store) {
        this.store = store;
    }

    @Override
    public void onReconciliationCycleCompleted(String tenancyId,
            DesiredStateGraph desired, ActualState actual) {
        store.evict(tenancyId, desired.nodes().keySet());
    }

    @Override
    public void onTenantStopped(String tenancyId) {
        store.evict(tenancyId, Set.of());
    }
}
```

- [ ] **Step 5: Modify TransitionPlanner — add exemptNodes parameter**

Add new `plan()` overloads with `Set<NodeId> exemptNodes`. Update the full-signature method to add the exemption check in the caller loop after `decideAction()`:

```java
// After: StepAction action = decideAction(status, node.targetStatus());
// After: if (action == null) { continue; }
// Add:
if (status == NodeStatus.DRIFTED && exemptNodes.contains(nodeId)) { continue; }
```

Existing overloads without `exemptNodes` delegate with `Set.of()`.

- [ ] **Step 6: Modify ReconciliationLoop — add drift policy fields**

Add fields `DriftPolicyEngine driftPolicyEngine` and `ExemptionStore exemptionStore` to `ReconciliationLoop`. Update the full constructor to accept these (with null-safe defaults: `new DriftPolicyEngine(List.of())` and `new InMemoryExemptionStore()`). Update the protected constructor. Update the Builder with `.driftPolicyEngine()` and `.exemptionStore()` methods.

- [ ] **Step 7: Modify detectDrift() — add drift policy evaluation**

Revise `detectDrift()` to accept `Set<NodeId> exemptNodesOut`. For each drifted node:

1. Check `exemptionStore.get(tenancyId, nodeId)` — if present and `!exemption.shouldRevert(Instant.now(), status)`, add to `exemptNodesOut`, continue
2. If exemption exists but `shouldRevert()` returns true, call `exemptionStore.revoke(tenancyId, nodeId)`, fall through
3. Evaluate `driftPolicyEngine.evaluate()` — if EXEMPT, compute `expiresAt`, call `exemptionStore.grant()`, add to `exemptNodesOut`, continue
4. If RECONCILE, proceed with existing FaultEvent/FaultPolicyEngine logic

Do NOT emit events inside detectDrift — exempt nodes are collected in `exemptNodesOut`.

- [ ] **Step 8: Modify reconcile() and reconcileTypes() — thread exempt nodes**

```java
Set<NodeId> exemptNodes = new HashSet<>();
desired = detectDrift(desired, actual, driftedNodesOut, exemptNodes);
TransitionPlan plan = plan(desired, actual, exemptNodes);
```

Update `TenantLoop.plan()` to accept and forward `exemptNodes`.

- [ ] **Step 9: Modify emitCycleEvents() — emit NodeDriftExemptedData**

After the existing driftedNodes emission loop, add emission for exemptNodes. Accept `Set<NodeId> exemptNodes` as a parameter. For each exempt node, create `NodeDriftExemptedData` and emit via `eventEmitter.nodeDriftExempted()`. Update `ReconciliationCompletedData` construction to include `exemptedCount`.

- [ ] **Step 10: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=ReconciliationLoopDriftPolicyTest`
Expected: all 4 tests PASS

- [ ] **Step 11: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 12: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java runtime-core/src/main/java/io/casehub/desiredstate/runtime/TransitionPlanner.java runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationEventEmitter.java runtime-core/src/main/java/io/casehub/desiredstate/runtime/ExemptionEvictionListener.java runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopDriftPolicyTest.java
git commit -m "feat(#160): ReconciliationLoop + TransitionPlanner drift policy integration"
```

## Batch 3: Framework Wiring + Test Fixtures

### Task 6: CDI wiring, Spring auto-config, test fixtures

**Files:**
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/RuntimeBeans.java` — add `@Produces` for `DriftPolicyEngine`, `DefaultExemptionStore`, `ExemptionEvictionListener`, update `reconciliationLoop()` method
- Modify: `runtime-spring/src/main/java/io/casehub/desiredstate/runtime/spring/DesiredStateRuntimeAutoConfiguration.java` — add `@Bean` for `DriftPolicyEngine`, default `ExemptionStore`, `ExemptionEvictionListener`, update `reconciliationLoop()` method
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DefaultExemptionStore.java`
- Create: `testing/src/main/java/io/casehub/desiredstate/testing/MockDriftPolicy.java`
- Create: `testing/src/main/java/io/casehub/desiredstate/testing/MockExemptionStore.java`

**Interfaces:**
- Consumes: `DriftPolicyEngine` (Task 4), `ExemptionStore`, `InMemoryExemptionStore`, `ExemptionEvictionListener` (Task 5), `DriftPolicy` (Task 2)
- Produces: CDI and Spring wiring for all drift policy beans, `DefaultExemptionStore`, `MockDriftPolicy`, `MockExemptionStore`

- [ ] **Step 1: Create DefaultExemptionStore**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.InMemoryExemptionStore;

public class DefaultExemptionStore extends InMemoryExemptionStore {
}
```

- [ ] **Step 2: Create MockDriftPolicy**

```java
package io.casehub.desiredstate.testing;

import io.casehub.desiredstate.api.*;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class MockDriftPolicy implements DriftPolicy {

    private final ConcurrentHashMap<NodeId, DriftDecision> decisions = new ConcurrentHashMap<>();
    private DriftDecision defaultDecision = DriftDecision.reconcile();

    public void setDecision(NodeId nodeId, DriftDecision decision) {
        decisions.put(nodeId, decision);
    }

    public void setDefaultDecision(DriftDecision decision) {
        this.defaultDecision = decision;
    }

    @Override
    public DriftDecision evaluate(NodeId nodeId, NodeStatus status,
                                  DesiredNode node, DriftContext context) {
        return decisions.getOrDefault(nodeId, defaultDecision);
    }
}
```

- [ ] **Step 3: Create MockExemptionStore**

```java
package io.casehub.desiredstate.testing;

import io.casehub.desiredstate.api.*;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

public class MockExemptionStore implements ExemptionStore {

    private record Key(String tenancyId, NodeId nodeId) {}

    private final ConcurrentHashMap<Key, Exemption> exemptions = new ConcurrentHashMap<>();

    @Override
    public void grant(String tenancyId, NodeId nodeId, Exemption exemption) {
        exemptions.put(new Key(tenancyId, nodeId), exemption);
    }

    @Override
    public void revoke(String tenancyId, NodeId nodeId) {
        exemptions.remove(new Key(tenancyId, nodeId));
    }

    @Override
    public Optional<Exemption> get(String tenancyId, NodeId nodeId) {
        return Optional.ofNullable(exemptions.get(new Key(tenancyId, nodeId)));
    }

    @Override
    public Map<NodeId, Exemption> getActive(String tenancyId) {
        return exemptions.entrySet().stream()
            .filter(e -> e.getKey().tenancyId().equals(tenancyId))
            .collect(Collectors.toMap(e -> e.getKey().nodeId(), Map.Entry::getValue));
    }

    @Override
    public void evict(String tenancyId, Set<NodeId> retainedNodes) {
        exemptions.keySet().removeIf(key ->
            key.tenancyId().equals(tenancyId) && !retainedNodes.contains(key.nodeId()));
    }

    public int size() { return exemptions.size(); }
    public void clear() { exemptions.clear(); }
}
```

- [ ] **Step 4: Update RuntimeBeans — add DriftPolicyEngine and ExemptionStore producers**

Add to `RuntimeBeans`:

```java
@Produces @ApplicationScoped
public DriftPolicyEngine driftPolicyEngine(Instance<DriftPolicy> policies) {
    return new DriftPolicyEngine(policies.stream()
        .sorted(Comparator.comparingInt(p -> {
            var priority = p.getClass().getAnnotation(jakarta.annotation.Priority.class);
            return priority != null ? -priority.value() : 0;
        }))
        .toList());
}

@Produces @DefaultBean @ApplicationScoped
public DefaultExemptionStore defaultExemptionStore() {
    return new DefaultExemptionStore();
}

@Produces @ApplicationScoped
public ExemptionEvictionListener exemptionEvictionListener(ExemptionStore store) {
    return new ExemptionEvictionListener(store);
}
```

Update the `reconciliationLoop()` method to accept `DriftPolicyEngine` and `ExemptionStore` parameters, passing them to the constructor.

- [ ] **Step 5: Update DesiredStateRuntimeAutoConfiguration — add Spring beans**

Add to `DesiredStateRuntimeAutoConfiguration`:

```java
@Bean @ConditionalOnMissingBean
public DriftPolicyEngine driftPolicyEngine(List<DriftPolicy> policies) {
    policies.sort(Comparator.comparingInt(p -> {
        var priority = p.getClass().getAnnotation(jakarta.annotation.Priority.class);
        var order = p.getClass().getAnnotation(org.springframework.core.annotation.Order.class);
        if (priority != null) return -priority.value();
        if (order != null) return -order.value();
        return 0;
    }));
    return new DriftPolicyEngine(policies);
}

@Bean @ConditionalOnMissingBean
public ExemptionStore exemptionStore() {
    return new InMemoryExemptionStore();
}

@Bean
public ExemptionEvictionListener exemptionEvictionListener(ExemptionStore store) {
    return new ExemptionEvictionListener(store);
}
```

Update the `reconciliationLoop()` method to accept `DriftPolicyEngine` and `ExemptionStore` parameters.

- [ ] **Step 6: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/desiredstate/runtime/DefaultExemptionStore.java runtime/src/main/java/io/casehub/desiredstate/runtime/RuntimeBeans.java runtime-spring/src/main/java/io/casehub/desiredstate/runtime/spring/DesiredStateRuntimeAutoConfiguration.java testing/src/main/java/io/casehub/desiredstate/testing/MockDriftPolicy.java testing/src/main/java/io/casehub/desiredstate/testing/MockExemptionStore.java
git commit -m "feat(#160): CDI + Spring wiring, test fixtures for drift policy layer"
```

### Task 7: CLAUDE.md update

**Files:**
- Modify: `CLAUDE.md` — add DriftPolicy to Core SPIs table, DriftPolicyEngine to Core Runtime Types, update module descriptions

**Interfaces:**
- Consumes: all types from Tasks 1-6
- Produces: updated documentation

- [ ] **Step 1: Update CLAUDE.md**

Add to `## Core SPIs` table:
- `DriftPolicy` row: `evaluate(NodeId, NodeStatus, DesiredNode, DriftContext) → DriftDecision`
- `ExemptionStore` row: `grant`, `revoke`, `get`, `getActive`, `evict`

Add to `## Core Runtime Types` table:
- `DriftPolicyEngine` — first-EXEMPT-wins aggregation of DriftPolicy beans by @Priority
- `DriftDecision` — sealed: Reconcile | Exempt(ExemptionSpec)
- `DriftContext` — tenancyId + graph + actual
- `ExemptionSpec` — revertCondition + metadata
- `RevertMode` — enum: DURATION, SCHEDULE, STATUS_CHANGE, NEVER
- `RevertCondition` — sealed: OnDuration | OnSchedule | OnStatusChange | Never. `mode()` default method
- `Exemption` — nodeId + spec + grantedAt + expiresAt. `shouldRevert(Instant, NodeStatus)`
- `InMemoryExemptionStore` — ConcurrentHashMap default
- `DefaultExemptionStore` — @DefaultBean wrapping InMemoryExemptionStore
- `ExemptionEvictionListener` — GlobalReconciliationListener for exemption eviction
- `NodeDriftExemptedData` — CloudEvent data for permitted drift

Update module descriptions for `api/`, `runtime-core/`, `runtime/`, `runtime-spring/`, `testing/` to mention drift policy types.

- [ ] **Step 2: Run full build to verify nothing broke**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#160): update CLAUDE.md with drift policy layer types and SPIs"
```

## References

- [2026-10-01-drift-policy-layer-design.md] — design spec this plan implements
- ReconciliationLoop.java:706-741 — detectDrift implementation
- ReconciliationLoop.java:595-641 — reconcile method
- ReconciliationLoop.java:647-692 — reconcileTypes method
- ReconciliationLoop.java:355-429 — Builder class
- ReconciliationLoop.java:126-161 — full constructor
- TransitionPlanner.java:34-93 — plan method with decideAction
- FaultPolicyEngine.java — chain evaluation pattern
- FaultCountStore.java — pluggable store SPI pattern
- InMemoryFaultCountStore.java — in-memory store pattern
- FaultCountEvictionListener.java — eviction listener pattern
- DefaultFaultCountStore.java — @DefaultBean store wrapper pattern
- ReconciliationEventEmitter.java — CloudEvent emission pattern
- RuntimeBeans.java:116-133 — CDI reconciliationLoop producer
- DesiredStateRuntimeAutoConfiguration.java:130-147 — Spring reconciliationLoop bean
- ReconciliationCompletedData.java — cycle summary record
- DesiredStateEventTypes.java — event type constants
- NodeDriftedData.java — existing drift event
- ReconciliationLoopTest.java — existing test patterns
- GitHub #160 — drift policy layer issue
- GitHub #153 — IoT consumer requirements (parent)
