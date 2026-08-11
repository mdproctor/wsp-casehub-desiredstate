# Flaky Test Elimination — Design Spec

**Issue:** #93
**Date:** 2026-07-27
**Scope:** casehub-desiredstate (primary), casehub-ops (follow-up)

## Problem

75 time-sensitive test patterns across 8 test files make CI flaky under load.
Three root causes, not one:

1. **Missing test infrastructure** — `testing/` module provides `MockNodeProvisioner`
   but no other mocks. Three identical test doubles are copy-pasted across test files
   with fragile cross-file inner class references.
2. **No observable cycle completion signal used** — tests sleep instead of polling
   existing observable state (executedPlans, capturedEvents, loop.getDesired(), world state).
3. **No centralized timeout policy** — magic numbers (200ms, 500ms, 2s, 5s, 6s)
   scattered across 8 files with no rationale for any choice.

## Design

### Part 1 — Test infrastructure enrichment (`testing/` module)

Add four new types to `casehub-desiredstate-testing`:

#### `MockActualStateAdapter`

Superset of all 4 existing variants (ReconciliationLoopTest's configurable types,
CloudEventTest's fixed types, LifecycleTest's per-node setStatus, LifecycleManagerTest's
named makePresent/makeAbsent):

```java
public class MockActualStateAdapter implements ActualStateAdapter {
    private final Map<NodeId, NodeStatus> statuses = new ConcurrentHashMap<>();
    private volatile Set<NodeType> handledTypes = Set.of(NodeType.of("test"));

    public void setStatuses(Map<NodeId, NodeStatus> statuses) { ... }
    public void setStatus(NodeId id, NodeStatus status) { ... }
    public void makePresent(NodeId id) { setStatus(id, NodeStatus.PRESENT); }
    public void makeAbsent(NodeId id) { setStatus(id, NodeStatus.ABSENT); }
    public void setHandledTypes(Set<NodeType> types) { ... }
    public void clear() { statuses.clear(); }

    @Override public Set<NodeType> handledTypes() { return handledTypes; }
    @Override public ActualState readActual(DesiredStateGraph desired, String tenancyId) {
        return new ActualState(Map.copyOf(statuses));
    }
}
```

Replaces inner classes in: ReconciliationLoopTest, ReconciliationLoopCloudEventTest,
ReconciliationTracingTest, ReconciliationLoopLifecycleTest, LifecycleManagerTest.

#### `MockTransitionExecutor`

Superset of the full version (records + configurable fail/reject/deprovision-fail)
and the always-succeed variants (ImmediateSuccessExecutor, SucceedingExecutor):

```java
public class MockTransitionExecutor implements TransitionExecutor {
    public final CopyOnWriteArrayList<TransitionPlan> executedPlans = new CopyOnWriteArrayList<>();
    public final Set<NodeId> failNodes = ConcurrentHashMap.newKeySet();
    public final Set<NodeId> failDeprovisionNodes = ConcurrentHashMap.newKeySet();
    public final Set<NodeId> rejectNodes = ConcurrentHashMap.newKeySet();

    @Override
    public TransitionResult execute(TransitionPlan plan, String tenancyId) {
        executedPlans.add(plan);
        Map<NodeId, StepOutcome> outcomes = new LinkedHashMap<>();
        for (OrderedStep step : plan.removals()) {
            if (failDeprovisionNodes.contains(step.node().id())) {
                outcomes.put(step.node().id(), new StepOutcome.Failed("test deprovision failure"));
            } else {
                outcomes.put(step.node().id(), new StepOutcome.Succeeded());
            }
        }
        for (OrderedStep step : plan.additions()) {
            if (rejectNodes.contains(step.node().id())) {
                outcomes.put(step.node().id(), new StepOutcome.Rejected("test rejection"));
            } else if (failNodes.contains(step.node().id())) {
                outcomes.put(step.node().id(), new StepOutcome.Failed("test failure"));
            } else {
                outcomes.put(step.node().id(), new StepOutcome.Succeeded());
            }
        }
        return new TransitionResult(outcomes);
    }
}
```

Tests needing "always succeed" use it with no fail sets configured.
Replaces: ReconciliationLoopTest.TestTransitionExecutor (3 copies),
ImmediateSuccessExecutor, SucceedingExecutor.

#### `MockEventSource`

Single implementation replacing 3 identical copies:

```java
public class MockEventSource implements EventSource {
    private final AtomicReference<MultiEmitter<? super StateEvent>> emitterRef = new AtomicReference<>();
    private final Multi<StateEvent> multi;

    public MockEventSource() {
        this.multi = Multi.createFrom().emitter(emitter -> emitterRef.set(emitter));
    }

    public void emit(StateEvent event) {
        MultiEmitter<? super StateEvent> emitter = emitterRef.get();
        if (emitter != null) emitter.emit(event);
    }

    @Override
    public Multi<StateEvent> stream() { return multi; }
}
```

#### `TestTimeouts`

Centralized timeout policy:

```java
public final class TestTimeouts {
    public static final Duration AWAIT = Duration.ofSeconds(10);
    public static final Duration MUTINY_AWAIT = Duration.ofSeconds(5);
    private TestTimeouts() {}
}
```

### Part 2 — Pattern replacements

Applied across all 8 test files after Part 1 consolidation.

#### Pattern A: `Thread.sleep(N); assert(condition)` → Awaitility polling (15 instances)

**Files:** LifecycleManagerTest (3), ReconciliationLoopCbrOutcomeTest (6), ExpansionLifecycleTest (6)

Replace with `await().atMost(AWAIT).until(() -> condition)` where the condition is
the same thing the assertion checks.

For **negative assertions** (e.g., CbrOutcomeTest.noCbrProposals_noOutcomeEvents —
"no CBR outcome events were emitted"), wait for the reconciliation cycle to complete
first, then assert absence. CbrOutcomeTest has `capturedEvents` with a CloudEvent
sink — wait for `RECONCILIATION_COMPLETED` event, then assert no CBR_OUTCOME events.

For **LifecycleManagerTest**: poll `loop.getDesired("t1")` for expected graph content.
No CloudEvent sink available; graph state is the observable.

For **ExpansionLifecycleTest**: poll `world.isBuilt(nodeId)`, `loop.getDesired("t1").nodes()`,
`adapter.readActual()` for expected domain state.

#### Pattern B: `await().atMost(2s)` → `await().atMost(AWAIT)` (~30 instances)

**Files:** ReconciliationLoopTest, ReconciliationLoopCloudEventTest,
ReconciliationTracingTest

Mechanical timeout bump. Conditions are already precise — they trigger immediately
when ready. The timeout is a backstop, not a timing mechanism.

#### Pattern C: `await().during(N).atMost(M)` → structural guarantee (1 instance)

**File:** ReconciliationLoopTest.stop_preventsSubsequentReconciliation (line 190)

`stop()` synchronously cancels the Mutiny subscription. After stop, emitted events
have no receiver. Replace `await().during()` with:

```java
loop.stop("test-tenant");
int planCountAfterStop = testExecutor.executedPlans.size();
testEventSource.emit(new StateEvent(NodeId.of("a"), NodeStatus.ABSENT, "ignored"));
assertThat(testExecutor.executedPlans).hasSize(planCountAfterStop);
```

No timing needed — the structural guarantee is sufficient.

#### Pattern D: `await().pollDelay(N).until(() -> true)` → test restructure (1 instance)

**File:** ReconciliationLoopTest.eventDriven_triggersReconciliation (line 113)

This is a disguised sleep waiting for an initial no-diff cycle before testing
event-driven reconciliation. The initial cycle produces no plan (actual matches
desired), so there's no observable side-effect.

Fix: restructure the test. The concern is "event-driven reconciliation works."
The initial cycle completes within the debounce period (50ms). Instead of
explicitly waiting for it, just push the event and wait for the event-driven
cycle. The debounce ensures the event-driven cycle fires after the initial
cycle. Remove the intermediate wait and `executedPlans.clear()` — the first
executed plan may be from either cycle, but the event-driven plan is identifiable
by its content (node "a" addition).

Alternatively: use `ReconciliationListener` with a `CountDownLatch` to wait
for the initial cycle, then proceed. This is more explicit but adds plumbing.

Decision: use Awaitility to wait for the initial no-diff cycle to complete via
`loop.getDesired("test-tenant")` being queryable (confirms start ran). Then
clear plans and proceed.

#### Pattern E: `latch.await(tight)` → `latch.await(AWAIT)` (8 instances)

**Files:** ReconciliationLoopSchedulingTest (3), ReconciliationLoopLifecycleTest (3),
ReconciliationLoopRequestReconciliationTest (2)

Standardize all CountDownLatch timeouts to `AWAIT.toSeconds()`.

#### Pattern F: Mutiny `.await().atMost(1s)` → `.await().atMost(MUTINY_AWAIT)` (3 instances)

**File:** DefaultMergedEventSourceTest

Finite streams that complete immediately. Timeout is a backstop.
The existing 10s timeout (failedStreamDoesNotKillOtherStreams) is already fine.

### Part 3 — Test file consolidation

For each test file:

1. Delete inner class test doubles that are replaced by `testing/` module mocks
2. Replace cross-file inner class references (`ReconciliationLoopCloudEventTest.TestActualStateAdapter`)
   with `testing/` module imports
3. Replace magic number timeouts with `TestTimeouts.AWAIT` / `TestTimeouts.MUTINY_AWAIT`
4. Apply pattern replacements from Part 2

**Test-specific doubles that stay local** (not consolidated):
- ReconciliationLoopSchedulingTest's counting adapter (AtomicInteger + CountDownLatch in readActual)
- ReconciliationLoopRequestReconciliationTest's baseline-tracking adapter
- ReconciliationTracingTest's SucceedingProvisioner and FailingProvisioner (provisioner variants, not adapter/executor)

### Part 4 — casehub-ops (follow-up)

Out of scope for this branch. File a follow-up issue to apply the same patterns
to casehub-ops tests using the new mocks from `casehub-desiredstate-testing`.

## Files Changed

### New files (testing/ module)
- `testing/src/main/java/io/casehub/desiredstate/testing/MockActualStateAdapter.java`
- `testing/src/main/java/io/casehub/desiredstate/testing/MockTransitionExecutor.java`
- `testing/src/main/java/io/casehub/desiredstate/testing/MockEventSource.java`
- `testing/src/main/java/io/casehub/desiredstate/testing/TestTimeouts.java`

### Modified files (runtime tests)
- `runtime/src/test/java/.../ReconciliationLoopTest.java`
- `runtime/src/test/java/.../ReconciliationLoopCloudEventTest.java`
- `runtime/src/test/java/.../ReconciliationLoopCbrOutcomeTest.java`
- `runtime/src/test/java/.../ReconciliationLoopSchedulingTest.java`
- `runtime/src/test/java/.../ReconciliationLoopLifecycleTest.java`
- `runtime/src/test/java/.../ReconciliationLoopRequestReconciliationTest.java`
- `runtime/src/test/java/.../ReconciliationTracingTest.java`
- `runtime/src/test/java/.../LifecycleManagerTest.java`
- `runtime/src/test/java/.../DefaultMergedEventSourceTest.java`

### Modified files (example tests)
- `examples/expansion/src/test/java/.../ExpansionLifecycleTest.java`

## No production code changes

The ReconciliationLoop API is sound. The problem is entirely in test code
and test infrastructure. No changes to api/, runtime/, engine-adapter/,
work-adapter/, ras-adapter/, or example production code.

## Success Criteria

- [ ] All Thread.sleep calls removed from test code (15 instances)
- [ ] All `await().during()` patterns eliminated (1 instance)
- [ ] All `await().until(() -> true)` patterns eliminated (1 instance)
- [ ] All timeouts standardized to TestTimeouts constants
- [ ] All duplicated test doubles replaced with testing/ module mocks
- [ ] All cross-file inner class references eliminated
- [ ] `mvn install` passes — all tests green
- [ ] No test takes >10s under normal conditions