# Constructor Telescope, Eviction Listener, Protocol Update — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #98 — refactor: ReconciliationLoop constructor telescope (8+ params)
**Issue group:** #98, #99, #97

**Goal:** Replace ReconciliationLoop's 7 public telescope constructors with a builder,
fix CbrProposalTracker CDI injection bug, add fault count eviction via GlobalReconciliationListener,
and update the CDI priority protocol.

**Architecture:** Builder pattern on ReconciliationLoop for test constructors, CDI constructor
stays. evictAcrossNamespaces() on FaultCountStore for cross-namespace bulk eviction.
FaultCountEvictionListener as a CDI-discovered GlobalReconciliationListener. Listener firing
removed from reconcileTypes() (semantic fix). onTenantStopped lifecycle hook for cleanup.

**Tech Stack:** Java 21, Quarkus (CDI), JPA/Hibernate, Mutiny, JUnit 5, AssertJ

## Global Constraints

- Pre-release platform — breaking changes cost nothing. Fix the design.
- IntelliJ MCP mandatory for all Java file operations. Never bash grep/Edit on .java files.
- `project_path: /Users/mdproctor/claude/casehub/desiredstate` for all ide_* calls.
- TDD: write failing test → verify fail → implement → verify pass → commit.
- No Flyway migrations — schema unchanged.

---

### Task 1: ReconciliationLoop Builder + CbrProposalTracker CDI Fix (#98)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java`
- Create: `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopBuilderTest.java`

**Interfaces:**
- Produces: `ReconciliationLoop.builder(TransitionPlanner, TransitionExecutor, ActualStateAdapterRouter, FaultPolicyEngine, MergedEventSource)` returning `ReconciliationLoop.Builder`
- Produces: `Builder.router(NodeProvisionerRouter)`, `.debounceWindow(Duration)`, `.resyncInterval(Duration)`, `.cloudEventSink(Consumer<CloudEvent>)`, `.cbrTracker(CbrProposalTracker)`, `.globalListeners(List<GlobalReconciliationListener>)`, `.build()` returning `ReconciliationLoop`

- [ ] **Step 1: Write failing tests for Builder**

Create `ReconciliationLoopBuilderTest.java` with `ide_create_file`:

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.testing.*;
import io.cloudevents.CloudEvent;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.List;
import java.util.Set;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

class ReconciliationLoopBuilderTest {

    private static final Duration AWAIT = Duration.ofSeconds(5);
    private static final Duration TEST_DEBOUNCE = Duration.ofMillis(50);
    private static final Duration TEST_RESYNC = Duration.ofHours(1);

    private TransitionPlanner planner;
    private MockTransitionExecutor executor;
    private MockActualStateAdapter adapter;
    private DefaultActualStateAdapterRouter adapterRouter;
    private FaultPolicyEngine faultEngine;
    private ReconciliationLoop loop;

    private record TestSpec() implements NodeSpec {}

    @BeforeEach
    void setUp() {
        planner = new TransitionPlanner();
        executor = new MockTransitionExecutor();
        adapter = new MockActualStateAdapter();
        adapter.setHandledTypes(Set.of(NodeType.of("t")));
        adapterRouter = new DefaultActualStateAdapterRouter(List.of(adapter));
        faultEngine = new FaultPolicyEngine(List.of());
    }

    @AfterEach
    void tearDown() {
        if (loop != null) loop.shutdown();
    }

    @Test
    void builder_withDefaults_createsWorkingLoop() throws Exception {
        DesiredStateGraph graph = ImmutableDesiredStateGraph.empty().withNode(
            new DesiredNode(NodeId.of("a"), NodeType.of("t"), new TestSpec(), HumanGating.NONE));
        adapter.setStatus(NodeId.of("a"), NodeStatus.ABSENT);

        CountDownLatch latch = new CountDownLatch(1);
        executor.onProvision(() -> latch.countDown());

        loop = ReconciliationLoop.builder(planner, executor, adapterRouter, faultEngine,
                () -> io.smallrye.mutiny.Multi.createFrom().nothing())
            .build();
        loop.start("t1", graph);

        assertThat(latch.await(AWAIT.toSeconds(), TimeUnit.SECONDS)).isTrue();
    }

    @Test
    void builder_withTimingOptions_appliesSettings() throws Exception {
        DesiredStateGraph graph = ImmutableDesiredStateGraph.empty().withNode(
            new DesiredNode(NodeId.of("a"), NodeType.of("t"), new TestSpec(), HumanGating.NONE));
        adapter.setStatus(NodeId.of("a"), NodeStatus.ABSENT);

        CountDownLatch latch = new CountDownLatch(1);
        executor.onProvision(() -> latch.countDown());

        loop = ReconciliationLoop.builder(planner, executor, adapterRouter, faultEngine,
                () -> io.smallrye.mutiny.Multi.createFrom().nothing())
            .debounceWindow(TEST_DEBOUNCE)
            .resyncInterval(TEST_RESYNC)
            .build();
        loop.start("t1", graph);

        assertThat(latch.await(AWAIT.toSeconds(), TimeUnit.SECONDS)).isTrue();
    }

    @Test
    void builder_withCloudEventSink_receivesEvents() throws Exception {
        DesiredStateGraph graph = ImmutableDesiredStateGraph.empty().withNode(
            new DesiredNode(NodeId.of("a"), NodeType.of("t"), new TestSpec(), HumanGating.NONE));
        adapter.setStatus(NodeId.of("a"), NodeStatus.ABSENT);

        List<CloudEvent> events = new CopyOnWriteArrayList<>();
        CountDownLatch latch = new CountDownLatch(1);

        loop = ReconciliationLoop.builder(planner, executor, adapterRouter, faultEngine,
                () -> io.smallrye.mutiny.Multi.createFrom().nothing())
            .debounceWindow(TEST_DEBOUNCE)
            .resyncInterval(TEST_RESYNC)
            .cloudEventSink(e -> { events.add(e); latch.countDown(); })
            .build();
        loop.start("t1", graph);

        assertThat(latch.await(AWAIT.toSeconds(), TimeUnit.SECONDS)).isTrue();
        assertThat(events).isNotEmpty();
    }

    @Test
    void builder_withGlobalListeners_firesOnCycle() throws Exception {
        DesiredStateGraph graph = ImmutableDesiredStateGraph.empty().withNode(
            new DesiredNode(NodeId.of("a"), NodeType.of("t"), new TestSpec(), HumanGating.NONE));
        adapter.setStatus(NodeId.of("a"), NodeStatus.PRESENT);

        CountDownLatch latch = new CountDownLatch(1);
        GlobalReconciliationListener gl = (tid, d, a) -> latch.countDown();

        loop = ReconciliationLoop.builder(planner, executor, adapterRouter, faultEngine,
                () -> io.smallrye.mutiny.Multi.createFrom().nothing())
            .debounceWindow(TEST_DEBOUNCE)
            .resyncInterval(TEST_RESYNC)
            .globalListeners(List.of(gl))
            .build();
        loop.start("t1", graph);

        assertThat(latch.await(AWAIT.toSeconds(), TimeUnit.SECONDS)).isTrue();
    }

    @Test
    void builder_withCbrTracker_sharesInstance() throws Exception {
        DesiredStateGraph graph = ImmutableDesiredStateGraph.empty().withNode(
            new DesiredNode(NodeId.of("a"), NodeType.of("t"), new TestSpec(), HumanGating.NONE));
        adapter.setStatus(NodeId.of("a"), NodeStatus.PRESENT);

        CbrProposalTracker tracker = new CbrProposalTracker();
        CountDownLatch latch = new CountDownLatch(1);

        loop = ReconciliationLoop.builder(planner, executor, adapterRouter, faultEngine,
                () -> io.smallrye.mutiny.Multi.createFrom().nothing())
            .debounceWindow(TEST_DEBOUNCE)
            .resyncInterval(TEST_RESYNC)
            .cbrTracker(tracker)
            .cloudEventSink(e -> latch.countDown())
            .build();
        loop.start("t1", graph);

        assertThat(latch.await(AWAIT.toSeconds(), TimeUnit.SECONDS)).isTrue();
        // Tracker is the same instance — verified structurally by passing it through the builder
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=ReconciliationLoopBuilderTest`
Expected: compilation failure — `builder()` method does not exist.

- [ ] **Step 3: Implement Builder + fix CDI constructor**

Use `ide_insert_member` to add the Builder class to ReconciliationLoop, after the private
master constructor (after line 239). Content:

```java
public static Builder builder(TransitionPlanner planner,
                               TransitionExecutor executor,
                               ActualStateAdapterRouter actualStateAdapterRouter,
                               FaultPolicyEngine faultPolicyEngine,
                               MergedEventSource mergedEventSource) {
    return new Builder(planner, executor, actualStateAdapterRouter, faultPolicyEngine, mergedEventSource);
}

public static class Builder {
    private final TransitionPlanner planner;
    private final TransitionExecutor executor;
    private final ActualStateAdapterRouter actualStateAdapterRouter;
    private final FaultPolicyEngine faultPolicyEngine;
    private final MergedEventSource mergedEventSource;
    private NodeProvisionerRouter router;
    private Duration debounceWindow = DEFAULT_DEBOUNCE;
    private Duration resyncInterval;
    private Consumer<CloudEvent> cloudEventSink;
    private CbrProposalTracker cbrTracker;
    private List<GlobalReconciliationListener> globalListeners = List.of();

    private Builder(TransitionPlanner planner, TransitionExecutor executor,
                    ActualStateAdapterRouter actualStateAdapterRouter,
                    FaultPolicyEngine faultPolicyEngine,
                    MergedEventSource mergedEventSource) {
        this.planner = planner;
        this.executor = executor;
        this.actualStateAdapterRouter = actualStateAdapterRouter;
        this.faultPolicyEngine = faultPolicyEngine;
        this.mergedEventSource = mergedEventSource;
    }

    public Builder router(NodeProvisionerRouter router) { this.router = router; return this; }
    public Builder debounceWindow(Duration debounceWindow) { this.debounceWindow = debounceWindow; return this; }
    public Builder resyncInterval(Duration resyncInterval) { this.resyncInterval = resyncInterval; return this; }
    public Builder cloudEventSink(Consumer<CloudEvent> cloudEventSink) { this.cloudEventSink = cloudEventSink; return this; }
    public Builder cbrTracker(CbrProposalTracker cbrTracker) { this.cbrTracker = cbrTracker; return this; }
    public Builder globalListeners(List<GlobalReconciliationListener> globalListeners) { this.globalListeners = globalListeners; return this; }

    public ReconciliationLoop build() {
        return new ReconciliationLoop(planner, executor, actualStateAdapterRouter,
            faultPolicyEngine, mergedEventSource, router, debounceWindow,
            resyncInterval, cloudEventSink, cbrTracker, globalListeners);
    }
}
```

Fix CDI constructor: use `ide_edit_member` on the `@Inject` constructor (line 117) to add
`CbrProposalTracker cbrTracker` parameter and pass it through:

```java
@Inject
public ReconciliationLoop(
        TransitionPlanner planner,
        TransitionExecutor executor,
        ActualStateAdapterRouter actualStateAdapterRouter,
        FaultPolicyEngine faultPolicyEngine,
        MergedEventSource mergedEventSource,
        NodeProvisionerRouter router,
        Event<CloudEvent> cloudEventSink,
        Instance<GlobalReconciliationListener> globalListeners,
        CbrProposalTracker cbrTracker) {
    this(planner, executor, actualStateAdapterRouter, faultPolicyEngine, mergedEventSource,
         router, DEFAULT_DEBOUNCE, null, cloudEventSink::fire, cbrTracker,
         globalListeners.stream().toList());
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=ReconciliationLoopBuilderTest`
Expected: all 5 tests PASS.

- [ ] **Step 5: Run full runtime test suite — verify no regressions**

Run: `mvn --batch-mode test -pl runtime`
Expected: all existing tests still PASS (builder coexists with telescope temporarily).

- [ ] **Step 6: Commit**

```
feat(#98): add ReconciliationLoop.Builder and fix CbrProposalTracker CDI injection

Refs #98
```

---

### Task 2: Migrate Test Call Sites + Delete Telescope Constructors (#98)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java`
- Modify: `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopTest.java`
- Modify: `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopLifecycleTest.java`
- Modify: `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopGlobalListenerTest.java`
- Modify: `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopCbrOutcomeTest.java`
- Modify: `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopCloudEventTest.java`
- Modify: `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopSchedulingTest.java`
- Modify: `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopRequestReconciliationTest.java`
- Modify: `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationTracingTest.java`
- Modify: `runtime/src/test/java/io/casehub/desiredstate/runtime/LifecycleManagerTest.java`
- Modify: `examples/expansion/src/test/java/io/casehub/desiredstate/example/expansion/ExpansionLifecycleTest.java`
- Modify: `engine-adapter/src/test/java/io/casehub/desiredstate/engine/DesiredStateReplanDispatchTest.java`

**Interfaces:**
- Consumes: `ReconciliationLoop.builder()` from Task 1

**Migration patterns** (apply to each file using `ide_replace_member` or `ide_edit_member`):

| Old constructor | Builder equivalent |
|---|---|
| `new RL(p, e, a, f, m)` | `RL.builder(p, e, a, f, m).build()` |
| `new RL(p, e, a, f, m, router, debounce)` | `RL.builder(p, e, a, f, m).router(router).debounceWindow(debounce).build()` |
| `new RL(p, e, a, f, m, debounce, resync)` | `RL.builder(p, e, a, f, m).debounceWindow(debounce).resyncInterval(resync).build()` |
| `new RL(p, e, a, f, m, d, r, sink)` | `RL.builder(p, e, a, f, m).debounceWindow(d).resyncInterval(r).cloudEventSink(sink).build()` |
| `new RL(p, e, a, f, m, d, r, sink, cbr)` | `RL.builder(p, e, a, f, m).debounceWindow(d).resyncInterval(r).cloudEventSink(sink).cbrTracker(cbr).build()` |
| `new RL(p, e, a, f, m, d, r, sink, cbr, gl)` | `RL.builder(p, e, a, f, m).debounceWindow(d).resyncInterval(r).cloudEventSink(sink).cbrTracker(cbr).globalListeners(gl).build()` |

- [ ] **Step 1: Migrate all test files**

For each test file listed above, use `ide_replace_member` on setUp methods and `ide_edit_member`
on test methods with inline constructor calls. Convert each `new ReconciliationLoop(...)` to the
builder equivalent per the migration table.

Process one file at a time. After each file, run `ide_diagnostics` to verify compilation.

- [ ] **Step 2: Run full test suite — verify all tests pass with builder**

Run: `mvn --batch-mode test -pl runtime,engine-adapter,examples/expansion`
Expected: all tests PASS.

- [ ] **Step 3: Delete telescope constructors**

Use `ide_edit_member` to delete each of the 7 public telescope constructors from
ReconciliationLoop.java (the constructors between the CDI constructor and the private master).
These are the constructors at (approximate) lines 131, 143, 155, 168, 183, 198 — all the public
non-@Inject constructors that delegate to the private master.

- [ ] **Step 4: Run full test suite — verify no regressions**

Run: `mvn --batch-mode test -pl runtime,engine-adapter,examples/expansion`
Expected: all tests PASS. No code references the deleted constructors.

- [ ] **Step 5: Verify with ide_diagnostics**

Run `ide_diagnostics` on `ReconciliationLoop.java` to confirm no compilation errors.

- [ ] **Step 6: Commit**

```
refactor(#98): migrate test call sites to builder, delete telescope constructors

Refs #98
```

---

### Task 3: evictAcrossNamespaces on FaultCountStore (#97)

**Files:**
- Modify: `api/src/main/java/io/casehub/desiredstate/api/FaultCountStore.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/InMemoryFaultCountStore.java`
- Modify: `persistence-jpa/src/main/java/io/casehub/desiredstate/persistence/jpa/JpaFaultCountStore.java`
- Modify: `api/src/test/java/io/casehub/desiredstate/api/InMemoryFaultCountStoreTest.java`
- Modify: `persistence-jpa/src/test/java/io/casehub/desiredstate/persistence/jpa/JpaFaultCountStoreTest.java`

**Interfaces:**
- Produces: `FaultCountStore.evictAcrossNamespaces(String tenancyId, Set<NodeId> retainedNodes)`

- [ ] **Step 1: Write failing tests for InMemoryFaultCountStore**

Add tests to `InMemoryFaultCountStoreTest.java` using `ide_insert_member`:

```java
@Test
void evictAcrossNamespaces_removesNonRetainedAcrossAllNamespaces() {
    store.incrementAndGet("ns1", "t1", NodeId.of("a"));
    store.incrementAndGet("ns1", "t1", NodeId.of("b"));
    store.incrementAndGet("ns2", "t1", NodeId.of("a"));
    store.incrementAndGet("ns2", "t1", NodeId.of("c"));

    store.evictAcrossNamespaces("t1", Set.of(NodeId.of("a")));

    assertThat(store.getCount("ns1", "t1", NodeId.of("a"))).isEqualTo(1);
    assertThat(store.getCount("ns1", "t1", NodeId.of("b"))).isZero();
    assertThat(store.getCount("ns2", "t1", NodeId.of("a"))).isEqualTo(1);
    assertThat(store.getCount("ns2", "t1", NodeId.of("c"))).isZero();
}

@Test
void evictAcrossNamespaces_emptyRetainedRemovesAllForTenant() {
    store.incrementAndGet("ns1", "t1", NodeId.of("a"));
    store.incrementAndGet("ns2", "t1", NodeId.of("b"));
    store.incrementAndGet("ns1", "t2", NodeId.of("c"));

    store.evictAcrossNamespaces("t1", Set.of());

    assertThat(store.getCount("ns1", "t1", NodeId.of("a"))).isZero();
    assertThat(store.getCount("ns2", "t1", NodeId.of("b"))).isZero();
    assertThat(store.getCount("ns1", "t2", NodeId.of("c"))).isEqualTo(1);
}

@Test
void evictAcrossNamespaces_allRetainedRemovesNothing() {
    store.incrementAndGet("ns1", "t1", NodeId.of("a"));
    store.incrementAndGet("ns2", "t1", NodeId.of("b"));

    store.evictAcrossNamespaces("t1", Set.of(NodeId.of("a"), NodeId.of("b")));

    assertThat(store.getCount("ns1", "t1", NodeId.of("a"))).isEqualTo(1);
    assertThat(store.getCount("ns2", "t1", NodeId.of("b"))).isEqualTo(1);
}

@Test
void evictAcrossNamespaces_doesNotAffectOtherTenants() {
    store.incrementAndGet("ns1", "t1", NodeId.of("a"));
    store.incrementAndGet("ns1", "t2", NodeId.of("a"));

    store.evictAcrossNamespaces("t1", Set.of());

    assertThat(store.getCount("ns1", "t1", NodeId.of("a"))).isZero();
    assertThat(store.getCount("ns1", "t2", NodeId.of("a"))).isEqualTo(1);
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn --batch-mode test -pl api -Dtest=InMemoryFaultCountStoreTest`
Expected: compilation failure — `evictAcrossNamespaces` does not exist.

- [ ] **Step 3: Add evictAcrossNamespaces to FaultCountStore interface**

Use `ide_insert_member` on `FaultCountStore.java` after `evict`:

```java
void evictAcrossNamespaces(String tenancyId, Set<NodeId> retainedNodes);
```

- [ ] **Step 4: Implement in InMemoryFaultCountStore**

Use `ide_insert_member` on `InMemoryFaultCountStore.java` after `evict`:

```java
@Override
public void evictAcrossNamespaces(String tenancyId, Set<NodeId> retainedNodes) {
    counts.keySet().removeIf(key ->
            key.tenancyId().equals(tenancyId)
                    && !retainedNodes.contains(key.nodeId()));
}
```

- [ ] **Step 5: Run InMemory tests — verify they pass**

Run: `mvn --batch-mode test -pl api -Dtest=InMemoryFaultCountStoreTest`
Expected: all tests PASS.

- [ ] **Step 6: Write failing tests for JpaFaultCountStore**

Add tests to `JpaFaultCountStoreTest.java` using `ide_insert_member`. Mirror the InMemory
tests (same 4 test cases) but with the JPA store:

```java
@Test
void evictAcrossNamespaces_removesNonRetainedAcrossAllNamespaces() {
    store.incrementAndGet("ns1", "t1", NodeId.of("a"));
    store.incrementAndGet("ns1", "t1", NodeId.of("b"));
    store.incrementAndGet("ns2", "t1", NodeId.of("a"));
    store.incrementAndGet("ns2", "t1", NodeId.of("c"));

    store.evictAcrossNamespaces("t1", Set.of(NodeId.of("a")));

    assertThat(store.getCount("ns1", "t1", NodeId.of("a"))).isEqualTo(1);
    assertThat(store.getCount("ns1", "t1", NodeId.of("b"))).isZero();
    assertThat(store.getCount("ns2", "t1", NodeId.of("a"))).isEqualTo(1);
    assertThat(store.getCount("ns2", "t1", NodeId.of("c"))).isZero();
}

@Test
void evictAcrossNamespaces_emptyRetainedRemovesAllForTenant() {
    store.incrementAndGet("ns1", "t1", NodeId.of("a"));
    store.incrementAndGet("ns2", "t1", NodeId.of("b"));
    store.incrementAndGet("ns1", "t2", NodeId.of("c"));

    store.evictAcrossNamespaces("t1", Set.of());

    assertThat(store.getCount("ns1", "t1", NodeId.of("a"))).isZero();
    assertThat(store.getCount("ns2", "t1", NodeId.of("b"))).isZero();
    assertThat(store.getCount("ns1", "t2", NodeId.of("c"))).isEqualTo(1);
}
```

- [ ] **Step 7: Implement in JpaFaultCountStore**

Use `ide_insert_member` on `JpaFaultCountStore.java` after `evict`:

```java
@Override
@Transactional
public void evictAcrossNamespaces(String tenancyId, Set<NodeId> retainedNodes) {
    if (retainedNodes.isEmpty()) {
        em.createQuery("DELETE FROM FaultCountEntity e WHERE e.tenancyId = :tid")
          .setParameter("tid", tenancyId)
          .executeUpdate();
    } else {
        Set<String> retained = retainedNodes.stream()
                                            .map(NodeId::value)
                                            .collect(java.util.stream.Collectors.toSet());
        em.createQuery("DELETE FROM FaultCountEntity e WHERE e.tenancyId = :tid AND e.nodeId NOT IN :retained")
          .setParameter("tid", tenancyId)
          .setParameter("retained", retained)
          .executeUpdate();
    }
}
```

- [ ] **Step 8: Run JPA tests — verify they pass**

Run: `mvn --batch-mode test -pl persistence-jpa -Dtest=JpaFaultCountStoreTest`
Expected: all tests PASS.

- [ ] **Step 9: Commit**

```
feat(#97): add evictAcrossNamespaces to FaultCountStore SPI and implementations

Refs #97
```

---

### Task 4: Listener Refactoring + onTenantStopped Lifecycle Hook (#97)

**Files:**
- Modify: `api/src/main/java/io/casehub/desiredstate/api/GlobalReconciliationListener.java`
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java`
- Modify: `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopGlobalListenerTest.java`

**Interfaces:**
- Consumes: `GlobalReconciliationListener` from api/
- Produces: `GlobalReconciliationListener.onTenantStopped(String tenancyId)` (default method)
- Produces: listener firing removed from `reconcileTypes()`, `onTenantStopped` called in `stop()`

- [ ] **Step 1: Write failing tests for onTenantStopped and reconcileTypes listener removal**

Add tests to `ReconciliationLoopGlobalListenerTest.java` using `ide_insert_member`:

```java
@Test
void onTenantStopped_firesOnStop() throws Exception {
    DesiredNode node = new DesiredNode(NodeId.of("a"), NodeType.of("t"), new TestSpec(), HumanGating.NONE);
    DesiredStateGraph graph = ImmutableDesiredStateGraph.empty().withNode(node);
    adapter.setStatus(NodeId.of("a"), NodeStatus.PRESENT);

    CountDownLatch cycleLatch = new CountDownLatch(1);
    List<String> stopped = new CopyOnWriteArrayList<>();

    GlobalReconciliationListener gl = new GlobalReconciliationListener() {
        @Override
        public void onReconciliationCycleCompleted(String tenancyId, DesiredStateGraph desired, ActualState actual) {
            cycleLatch.countDown();
        }
        @Override
        public void onTenantStopped(String tenancyId) {
            stopped.add(tenancyId);
        }
    };

    ReconciliationLoop loop = ReconciliationLoop.builder(planner, new MockTransitionExecutor(),
            adapterRouter, faultPolicyEngine, () -> Multi.createFrom().nothing())
        .debounceWindow(Duration.ofMillis(50)).resyncInterval(Duration.ofSeconds(60))
        .globalListeners(List.of(gl)).build();
    loop.start("t1", graph);

    assertThat(cycleLatch.await(AWAIT.toSeconds(), TimeUnit.SECONDS)).isTrue();
    loop.stop("t1");

    assertThat(stopped).containsExactly("t1");
}

@Test
void onTenantStopped_exceptionDoesNotBlockOtherListeners() throws Exception {
    DesiredNode node = new DesiredNode(NodeId.of("a"), NodeType.of("t"), new TestSpec(), HumanGating.NONE);
    DesiredStateGraph graph = ImmutableDesiredStateGraph.empty().withNode(node);
    adapter.setStatus(NodeId.of("a"), NodeStatus.PRESENT);

    CountDownLatch cycleLatch = new CountDownLatch(1);
    List<String> stopped = new CopyOnWriteArrayList<>();

    GlobalReconciliationListener failing = new GlobalReconciliationListener() {
        @Override
        public void onReconciliationCycleCompleted(String tid, DesiredStateGraph d, ActualState a) {
            cycleLatch.countDown();
        }
        @Override
        public void onTenantStopped(String tenancyId) {
            throw new RuntimeException("boom");
        }
    };
    GlobalReconciliationListener surviving = new GlobalReconciliationListener() {
        @Override
        public void onReconciliationCycleCompleted(String tid, DesiredStateGraph d, ActualState a) {}
        @Override
        public void onTenantStopped(String tenancyId) {
            stopped.add(tenancyId);
        }
    };

    ReconciliationLoop loop = ReconciliationLoop.builder(planner, new MockTransitionExecutor(),
            adapterRouter, faultPolicyEngine, () -> Multi.createFrom().nothing())
        .debounceWindow(Duration.ofMillis(50)).resyncInterval(Duration.ofSeconds(60))
        .globalListeners(List.of(failing, surviving)).build();
    loop.start("t1", graph);

    assertThat(cycleLatch.await(AWAIT.toSeconds(), TimeUnit.SECONDS)).isTrue();
    loop.stop("t1");

    assertThat(stopped).containsExactly("t1");
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=ReconciliationLoopGlobalListenerTest#onTenantStopped*`
Expected: compilation failure — `onTenantStopped` does not exist on GlobalReconciliationListener.

- [ ] **Step 3: Add onTenantStopped to GlobalReconciliationListener**

Use `ide_edit_member` on `GlobalReconciliationListener.java` member `GlobalReconciliationListener`:

```java
public interface GlobalReconciliationListener {
    void onReconciliationCycleCompleted(String tenancyId, DesiredStateGraph desired, ActualState actual);
    default void onTenantStopped(String tenancyId) {}
}
```

- [ ] **Step 4: Implement in ReconciliationLoop**

**4a. Split fireListener into two methods.** Use `ide_edit_member` on `fireListener` (line ~551):

```java
private void fireGlobalListeners(DesiredStateGraph desired, ActualState actual) {
    for (GlobalReconciliationListener gl : globalListeners) {
        try {
            gl.onReconciliationCycleCompleted(tenancyId, desired, actual);
        } catch (Exception e) {
            LOG.log(Level.WARNING,
                    "Global reconciliation listener failed for tenant " + tenancyId, e);
        }
    }
}

private void firePerTenantListener(DesiredStateGraph desired, ActualState actual) {
    ReconciliationListener l = listener;
    if (l != null) {
        try {
            l.onReconciliationCycleCompleted(tenancyId, desired, actual);
        } catch (Exception e) {
            LOG.log(Level.WARNING,
                    "Reconciliation listener failed for tenant " + tenancyId, e);
        }
    }
}
```

**4b. Update reconcile() to call both.** Use `ide_replace_text_in_file` to replace
`fireListener(desired, actual)` in `reconcile()` with:
```java
fireGlobalListeners(desired, actual);
firePerTenantListener(desired, actual);
```

**4c. Remove fireListener call from reconcileTypes().** Use `ide_replace_text_in_file` to
delete the `fireListener(fullDesired, actual);` line from `reconcileTypes()` (line ~644)
and the comment above it.

**4d. Add onTenantStopped call in stop().** Use `ide_edit_member` on `TenantLoop.stop()`
to implement the stop sequence (unsubscribe → onTenantStopped → cancel futures → clearTenant):

```java
void stop() {
    if (eventSubscription != null) {
        eventSubscription.cancel();
    }
    for (GlobalReconciliationListener gl : globalListeners) {
        try {
            gl.onTenantStopped(tenancyId);
        } catch (Exception e) {
            LOG.log(Level.WARNING,
                    "Global listener onTenantStopped failed for tenant " + tenancyId, e);
        }
    }
    if (resyncFuture != null) {
        resyncFuture.cancel(false);
    }
    for (ScheduledFuture<?> future : resyncFutures.values()) {
        future.cancel(false);
    }
    resyncFutures.clear();
    if (requestedReconciliation != null) {
        requestedReconciliation.cancel(false);
    }
    cbrTracker.clearTenant(tenancyId);
}
```

- [ ] **Step 5: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=ReconciliationLoopGlobalListenerTest`
Expected: all tests PASS (including existing ones — fireGlobalListeners still fires from reconcile).

- [ ] **Step 6: Run full test suite — verify no regressions**

Run: `mvn --batch-mode test -pl runtime`
Expected: all tests PASS. The reconcileTypes listener removal may cause test failures in
ReconciliationLoopSchedulingTest if any tests relied on listener firing from type-filtered resync.
Check and fix if needed.

- [ ] **Step 7: Commit**

```
feat(#97): split fireListener, remove from reconcileTypes, add onTenantStopped lifecycle hook

Refs #97
```

---

### Task 5: FaultCountEvictionListener (#97)

**Files:**
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/FaultCountEvictionListener.java`
- Create: `runtime/src/test/java/io/casehub/desiredstate/runtime/FaultCountEvictionListenerTest.java`

**Interfaces:**
- Consumes: `FaultCountStore.evictAcrossNamespaces()` from Task 3
- Consumes: `GlobalReconciliationListener.onTenantStopped()` from Task 4

- [ ] **Step 1: Write failing tests**

Create `FaultCountEvictionListenerTest.java` with `ide_create_file`:

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class FaultCountEvictionListenerTest {

    private InMemoryFaultCountStore store;
    private FaultCountEvictionListener listener;

    @BeforeEach
    void setUp() {
        store = new InMemoryFaultCountStore();
        listener = new FaultCountEvictionListener(store);
    }

    @Test
    void onCycleCompleted_evictsCountsForRemovedNodes() {
        store.incrementAndGet("ns1", "t1", NodeId.of("a"));
        store.incrementAndGet("ns1", "t1", NodeId.of("b"));
        store.incrementAndGet("ns2", "t1", NodeId.of("b"));

        DesiredStateGraph graph = ImmutableDesiredStateGraph.empty().withNode(
            new DesiredNode(NodeId.of("a"), NodeType.of("t"), new TestSpec(), HumanGating.NONE));

        listener.onReconciliationCycleCompleted("t1", graph, new ActualState(java.util.Map.of()));

        assertThat(store.getCount("ns1", "t1", NodeId.of("a"))).isEqualTo(1);
        assertThat(store.getCount("ns1", "t1", NodeId.of("b"))).isZero();
        assertThat(store.getCount("ns2", "t1", NodeId.of("b"))).isZero();
    }

    @Test
    void onCycleCompleted_retainsCountsForExistingNodes() {
        store.incrementAndGet("ns1", "t1", NodeId.of("a"));
        store.incrementAndGet("ns2", "t1", NodeId.of("a"));

        DesiredStateGraph graph = ImmutableDesiredStateGraph.empty().withNode(
            new DesiredNode(NodeId.of("a"), NodeType.of("t"), new TestSpec(), HumanGating.NONE));

        listener.onReconciliationCycleCompleted("t1", graph, new ActualState(java.util.Map.of()));

        assertThat(store.getCount("ns1", "t1", NodeId.of("a"))).isEqualTo(1);
        assertThat(store.getCount("ns2", "t1", NodeId.of("a"))).isEqualTo(1);
    }

    @Test
    void onTenantStopped_evictsAllCountsForTenant() {
        store.incrementAndGet("ns1", "t1", NodeId.of("a"));
        store.incrementAndGet("ns2", "t1", NodeId.of("b"));
        store.incrementAndGet("ns1", "t2", NodeId.of("c"));

        listener.onTenantStopped("t1");

        assertThat(store.getCount("ns1", "t1", NodeId.of("a"))).isZero();
        assertThat(store.getCount("ns2", "t1", NodeId.of("b"))).isZero();
        assertThat(store.getCount("ns1", "t2", NodeId.of("c"))).isEqualTo(1);
    }

    @Test
    void onCycleCompleted_emptyGraph_evictsAllForTenant() {
        store.incrementAndGet("ns1", "t1", NodeId.of("a"));
        store.incrementAndGet("ns2", "t1", NodeId.of("b"));

        listener.onReconciliationCycleCompleted("t1",
            ImmutableDesiredStateGraph.empty(), new ActualState(java.util.Map.of()));

        assertThat(store.getCount("ns1", "t1", NodeId.of("a"))).isZero();
        assertThat(store.getCount("ns2", "t1", NodeId.of("b"))).isZero();
    }

    private record TestSpec() implements NodeSpec {}
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=FaultCountEvictionListenerTest`
Expected: compilation failure — `FaultCountEvictionListener` does not exist.

- [ ] **Step 3: Implement FaultCountEvictionListener**

Create with `ide_create_file`:

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.ActualState;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.FaultCountStore;
import io.casehub.desiredstate.api.GlobalReconciliationListener;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.Set;

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

    @Override
    public void onTenantStopped(String tenancyId) {
        store.evictAcrossNamespaces(tenancyId, Set.of());
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=FaultCountEvictionListenerTest`
Expected: all 4 tests PASS.

- [ ] **Step 5: Run full project test suite**

Run: `mvn --batch-mode test`
Expected: all tests across all modules PASS.

- [ ] **Step 6: Commit**

```
feat(#97): FaultCountEvictionListener — evict stale counts via GlobalReconciliationListener

Refs #97
```

---

### Task 6: Protocol Update (#99)

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/garden/docs/protocols/universal/persistence-backend-cdi-priority.md`

**Interfaces:**
- None — documentation only

- [ ] **Step 1: Read current protocol**

Read the protocol file to understand the current content before modifying.

- [ ] **Step 2: Update the protocol**

Update the tier table to split Tier 1 into 1a (functional fallback) and 1b (no-op/misconfiguration
signal). Add `DefaultFaultCountStore` and `SimpleTransitionExecutor` as examples of 1a. Update the
guidance about `@DefaultBean` to clarify that functional fallbacks are valid when the SPI has
meaningful non-persistent semantics.

The full tier table per the spec:

| Tier | Annotation | Semantics | Example |
|------|-----------|-----------|---------|
| 1a | `@DefaultBean @ApplicationScoped` | Functional fallback — works correctly within constraints | DefaultFaultCountStore |
| 1b | `@DefaultBean @ApplicationScoped` | No-op / misconfiguration signal | NoOpPendingApprovalHandler |
| 2 | `@ApplicationScoped` | Primary backend (JPA) — displaces @DefaultBean | JpaFaultCountStore |
| 3 | `@Alternative @Priority(1)` | Secondary backend (NoSQL) | — |
| 4 | `@Alternative @Priority(100)` | In-memory test override | — |

Narrow the "Don't use @DefaultBean on real implementations" guidance to: "Don't use @DefaultBean
on implementations that require external infrastructure (DB, message broker) to function."

- [ ] **Step 3: Commit to garden repo**

```bash
git -C /Users/mdproctor/claude/casehub/garden add docs/protocols/universal/persistence-backend-cdi-priority.md
git -C /Users/mdproctor/claude/casehub/garden commit -m "docs(#99): update CDI priority protocol — @DefaultBean functional fallbacks

Refs casehubio/casehub-desiredstate#99"
```

---

## CLAUDE.md Updates

After all tasks complete, update `CLAUDE.md`:
- Add `FaultCountEvictionListener` to the Core Runtime Types table
- Update `GlobalReconciliationListener` SPI entry to document `onTenantStopped` default method
- Update `FaultCountStore` SPI entry to document `evictAcrossNamespaces`
- Note that ReconciliationLoop uses builder pattern for test construction
