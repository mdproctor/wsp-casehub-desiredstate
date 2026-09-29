# Orchestration Primitives Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #150 — Leverage yaml-core orchestration primitives for plugin provisioners and parallel execution
**Issue group:** #150

**Goal:** Integrate yaml-core orchestration primitives into the desired-state runtime across three areas: plugin provisioner step evaluation (migration to StructuralStepEvaluator), parallel node provisioning (ParallelTransitionExecutor), and declarative node lifecycle state machines (StatefulNodeProvisioner).

**Architecture:** TransitionPlan gains layer-structured phases (List<List<OrderedStep>>) produced by TransitionPlanner's existing topological sort. A shared NodeStepExecutor extracts per-node execution logic (human gating, approval lifecycle, hooks, OTel) from SimpleTransitionExecutor. ParallelTransitionExecutor executes layers concurrently via virtual threads with optional JDK Semaphore rate limiting. Plugin provisioners migrate from archived yaml-step-core/StepPipelineExecutor to yaml-step-runtime/StructuralStepEvaluator. StatefulNodeProvisioner wraps provisioners with OrcStateMachine<NodeLifecycleState> enforcement, with lifecycle definitions declared in plugin YAML.

**Tech Stack:** Java 21+ (virtual threads), Quarkus CDI, yaml-core orchestration (OrcStateMachine), yaml-step-runtime (StructuralStepEvaluator), JDK CountDownLatch/Semaphore, OpenTelemetry tracing

## Global Constraints

- All new types in `api/` must be pure Java — no CDI, no Quarkus annotations, no framework imports
- `runtime-core/` is framework-neutral — constructor injection only, no CDI annotations
- `runtime/` contains CDI bridges — `@Produces`, `@DefaultBean`, `@ApplicationScoped`
- New `NodeProvisioner` default methods must not break existing implementations (binary compatibility)
- `TransitionPlan` changes must be backward-compatible — existing constructors preserved
- Plugin YAML backward compatibility — existing flat step lists must work unchanged
- `CaseTransitionExecutor` migration is mechanical — use `flatRemovals()`/`flatAdditions()` accessors
- OTel context must propagate to virtual threads in ParallelTransitionExecutor

---

## Batch 1: API Types + TransitionPlan Layered Output

Safe wrap point: all new API types exist, TransitionPlan and TransitionPlanner produce layered data, all existing callers migrated to flat-view accessors. Build passes. No behavioral change.

### Task 1: New API types

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/NodeLifecycleState.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/TransitionAction.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/NodeLifecycleDefinition.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/LifecycleStateEnteredData.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/LifecycleStateExitedData.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/NodeProvisioner.java:55` (add `maxConcurrency()`)
- Modify: `api/src/main/java/io/casehub/desiredstate/api/DesiredStateEventTypes.java` (add lifecycle constants)
- Modify: `api/src/main/java/io/casehub/desiredstate/api/PendingApprovalHandler.java` (javadoc thread-safety)
- Test: `api/src/test/java/io/casehub/desiredstate/api/NodeLifecycleDefinitionTest.java`

**Interfaces:**
- Produces: `NodeLifecycleState` enum (8 values: ABSENT, PROVISIONING, PRESENT, DRIFTED, DEPROVISIONING, SUSPENDING, SUSPENDED, RESUMING)
- Produces: `TransitionAction` sealed interface with `EmitEvent(String eventType)` variant
- Produces: `NodeLifecycleDefinition` record with `nodeType()`, `transitions()`, `onEnter()`, `onExit()`, `validate()` method
- Produces: `NodeProvisioner.maxConcurrency()` returning `OptionalInt.empty()` by default
- Produces: `DesiredStateEventTypes.LIFECYCLE_STATE_ENTERED`, `LIFECYCLE_STATE_EXITED` constants

- [ ] **Step 1: Write NodeLifecycleState enum**

```java
package io.casehub.desiredstate.api;

public enum NodeLifecycleState {
    ABSENT,
    PROVISIONING,
    PRESENT,
    DRIFTED,
    DEPROVISIONING,
    SUSPENDING,
    SUSPENDED,
    RESUMING;

    public boolean isTransient() {
        return this == PROVISIONING || this == DEPROVISIONING
               || this == SUSPENDING || this == RESUMING;
    }

    public static NodeLifecycleState fromNodeStatus(NodeStatus status) {
        return switch (status) {
            case PRESENT -> PRESENT;
            case ABSENT -> ABSENT;
            case DRIFTED -> DRIFTED;
            case SUSPENDED -> SUSPENDED;
            case UNKNOWN -> null;
        };
    }
}
```

- [ ] **Step 2: Write TransitionAction sealed interface**

```java
package io.casehub.desiredstate.api;

public sealed interface TransitionAction {
    record EmitEvent(String eventType) implements TransitionAction {
        public EmitEvent {
            java.util.Objects.requireNonNull(eventType, "eventType must not be null");
        }
    }
}
```

- [ ] **Step 3: Write NodeLifecycleDefinition record with validate()**

```java
package io.casehub.desiredstate.api;

import java.util.*;

public record NodeLifecycleDefinition(
    NodeType nodeType,
    Set<Transition> transitions,
    Map<NodeLifecycleState, List<TransitionAction>> onEnter,
    Map<NodeLifecycleState, List<TransitionAction>> onExit
) {
    public record Transition(NodeLifecycleState from, NodeLifecycleState to) {}

    public NodeLifecycleDefinition {
        Objects.requireNonNull(nodeType);
        transitions = Set.copyOf(transitions);
        onEnter = Map.copyOf(onEnter);
        onExit = Map.copyOf(onExit);
    }

    public boolean supportsSuspendResume() {
        return transitions.stream().anyMatch(t ->
            t.from() == NodeLifecycleState.SUSPENDING || t.to() == NodeLifecycleState.SUSPENDING
            || t.from() == NodeLifecycleState.RESUMING || t.to() == NodeLifecycleState.RESUMING);
    }

    public List<String> validate() {
        List<String> errors = new ArrayList<>();
        Set<NodeLifecycleState> transientStates = EnumSet.of(
            NodeLifecycleState.PROVISIONING, NodeLifecycleState.DEPROVISIONING,
            NodeLifecycleState.SUSPENDING, NodeLifecycleState.RESUMING);
        Set<NodeLifecycleState> persistentStates = EnumSet.of(
            NodeLifecycleState.ABSENT, NodeLifecycleState.PRESENT,
            NodeLifecycleState.DRIFTED, NodeLifecycleState.SUSPENDED);

        for (NodeLifecycleState transient_ : transientStates) {
            boolean referenced = transitions.stream().anyMatch(t -> t.from() == transient_ || t.to() == transient_);
            if (referenced) {
                boolean hasExit = transitions.stream().anyMatch(t ->
                    t.from() == transient_ && persistentStates.contains(t.to()));
                if (!hasExit) {
                    errors.add("Transient state " + transient_ + " has no exit transition to a persistent state");
                }
            }
        }

        boolean absentAsFrom = transitions.stream().anyMatch(t -> t.from() == NodeLifecycleState.ABSENT);
        if (!absentAsFrom) {
            errors.add("ABSENT must appear as a 'from' state in at least one transition");
        }

        Set<NodeLifecycleState> allStates = EnumSet.noneOf(NodeLifecycleState.class);
        transitions.forEach(t -> { allStates.add(t.from()); allStates.add(t.to()); });
        for (NodeLifecycleState s : onEnter.keySet()) {
            if (!allStates.contains(s)) errors.add("onEnter references orphan state " + s);
        }
        for (NodeLifecycleState s : onExit.keySet()) {
            if (!allStates.contains(s)) errors.add("onExit references orphan state " + s);
        }
        return errors;
    }
}
```

- [ ] **Step 4: Write test for NodeLifecycleDefinition.validate()**

```java
@Test
void validDefinition_noErrors() {
    var def = new NodeLifecycleDefinition(NodeType.of("test"),
        Set.of(
            new NodeLifecycleDefinition.Transition(NodeLifecycleState.ABSENT, NodeLifecycleState.PROVISIONING),
            new NodeLifecycleDefinition.Transition(NodeLifecycleState.PROVISIONING, NodeLifecycleState.PRESENT),
            new NodeLifecycleDefinition.Transition(NodeLifecycleState.PROVISIONING, NodeLifecycleState.DRIFTED),
            new NodeLifecycleDefinition.Transition(NodeLifecycleState.PRESENT, NodeLifecycleState.DEPROVISIONING),
            new NodeLifecycleDefinition.Transition(NodeLifecycleState.DEPROVISIONING, NodeLifecycleState.ABSENT)
        ), Map.of(), Map.of());
    assertThat(def.validate()).isEmpty();
}

@Test
void transientStateWithNoExit_producesError() {
    var def = new NodeLifecycleDefinition(NodeType.of("test"),
        Set.of(
            new NodeLifecycleDefinition.Transition(NodeLifecycleState.ABSENT, NodeLifecycleState.PROVISIONING)
        ), Map.of(), Map.of());
    assertThat(def.validate()).anyMatch(e -> e.contains("PROVISIONING") && e.contains("no exit"));
}

@Test
void supportsSuspendResume_trueWhenSuspendTransitionsExist() {
    var def = new NodeLifecycleDefinition(NodeType.of("test"),
        Set.of(
            new NodeLifecycleDefinition.Transition(NodeLifecycleState.ABSENT, NodeLifecycleState.PROVISIONING),
            new NodeLifecycleDefinition.Transition(NodeLifecycleState.PROVISIONING, NodeLifecycleState.PRESENT),
            new NodeLifecycleDefinition.Transition(NodeLifecycleState.PRESENT, NodeLifecycleState.SUSPENDING),
            new NodeLifecycleDefinition.Transition(NodeLifecycleState.SUSPENDING, NodeLifecycleState.SUSPENDED)
        ), Map.of(), Map.of());
    assertThat(def.supportsSuspendResume()).isTrue();
}
```

- [ ] **Step 5: Run tests, verify pass**

Run: `mvn --batch-mode -pl api test -Dtest=NodeLifecycleDefinitionTest`

- [ ] **Step 6: Add maxConcurrency() to NodeProvisioner, lifecycle constants to DesiredStateEventTypes, thread-safety javadoc to PendingApprovalHandler, create payload records**

Add to `NodeProvisioner.java` after `supportsStatefulLifecycle()`:
```java
default OptionalInt maxConcurrency() {
    return OptionalInt.empty();
}
```

Add to `DesiredStateEventTypes.java`:
```java
public static final String LIFECYCLE_STATE_ENTERED = "io.casehub.desiredstate.lifecycle.state-entered";
public static final String LIFECYCLE_STATE_EXITED = "io.casehub.desiredstate.lifecycle.state-exited";
```

Create `LifecycleStateEnteredData.java`:
```java
public record LifecycleStateEnteredData(
    String tenancyId, String nodeId, String nodeType,
    String state, String previousState, String customEventType
) {}
```

Create `LifecycleStateExitedData.java` (same pattern with `nextState`).

Add to `PendingApprovalHandler` javadoc: `Implementations must be thread-safe — check() and recordPending() may be called concurrently for different nodes by ParallelTransitionExecutor.`

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/NodeLifecycleState.java api/src/main/java/io/casehub/desiredstate/api/TransitionAction.java api/src/main/java/io/casehub/desiredstate/api/NodeLifecycleDefinition.java api/src/main/java/io/casehub/desiredstate/api/LifecycleStateEnteredData.java api/src/main/java/io/casehub/desiredstate/api/LifecycleStateExitedData.java api/src/main/java/io/casehub/desiredstate/api/NodeProvisioner.java api/src/main/java/io/casehub/desiredstate/api/DesiredStateEventTypes.java api/src/main/java/io/casehub/desiredstate/api/PendingApprovalHandler.java api/src/test/java/io/casehub/desiredstate/api/NodeLifecycleDefinitionTest.java
git commit -m "feat(#150): add API types for orchestration primitives — NodeLifecycleState, TransitionAction, NodeLifecycleDefinition, maxConcurrency(), lifecycle event types"
```

### Task 2: TransitionPlan layered output + TransitionPlanner + caller migration

**Files:**
- Modify: `api/src/main/java/io/casehub/desiredstate/api/TransitionPlan.java:6-31`
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/TransitionPlanner.java:120-167`
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/SimpleTransitionExecutor.java:63-87` (use flatX accessors)
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java` (multiple sites — use flatX accessors)
- Modify: `engine-adapter/src/main/java/io/casehub/desiredstate/engine/CaseTransitionExecutor.java` (use flatX accessors)
- Modify: `testing/src/main/java/io/casehub/desiredstate/testing/MockTransitionExecutor.java` (use flatX accessors)
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/TransitionPlannerTest.java` (update for layered output)

**Interfaces:**
- Consumes: existing `TransitionPlan`, `TransitionPlanner.topologicalSort()`
- Produces: `TransitionPlan` with `List<List<OrderedStep>>` fields, `flatRemovals()` / `flatSuspensions()` / `flatResumptions()` / `flatAdditions()` accessors, backward-compatible constructors

- [ ] **Step 1: Write test for layered TransitionPlan output**

Add to `TransitionPlannerTest.java`:
```java
@Test
void layeredAdditions_groupsNodesByTopologicalDepth() {
    // A -> B -> C  (A is root)
    var graph = factory.of(
        Map.of(NodeId.of("A"), new DesiredNode(NodeId.of("A"), spec),
               NodeId.of("B"), new DesiredNode(NodeId.of("B"), spec),
               NodeId.of("C"), new DesiredNode(NodeId.of("C"), spec)),
        Set.of(new Dependency(NodeId.of("B"), NodeId.of("A")),
               new Dependency(NodeId.of("C"), NodeId.of("B"))));
    ActualState actual = new ActualState(Map.of());

    TransitionPlan plan = planner.plan(graph, actual);

    assertEquals(3, plan.additions().size(), "3 layers");
    assertEquals(1, plan.additions().get(0).size(), "Layer 0: A");
    assertEquals(NodeId.of("A"), plan.additions().get(0).get(0).node().id());
    assertEquals(1, plan.additions().get(1).size(), "Layer 1: B");
    assertEquals(1, plan.additions().get(2).size(), "Layer 2: C");
    assertEquals(3, plan.flatAdditions().size(), "flat view: 3 nodes");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -pl runtime test -Dtest=TransitionPlannerTest#layeredAdditions_groupsNodesByTopologicalDepth`
Expected: FAIL — `additions()` returns `List<OrderedStep>`, not `List<List<OrderedStep>>`

- [ ] **Step 3: Modify TransitionPlan record**

Change fields from `List<OrderedStep>` to `List<List<OrderedStep>>`. Add `flatX()` accessors. Keep backward-compatible constructors that wrap flat lists as single-layer. Update compact constructor for deep immutability.

- [ ] **Step 4: Modify TransitionPlanner.topologicalSort() to return layers**

Change `topologicalSort()` return type from `List<NodeId>` to `List<List<NodeId>>`. Use BFS-level tracking in Kahn's algorithm — collect all nodes with in-degree 0 as one layer, then process them together. The `topologicalSortReverse()` method reverses the layer list AND the nodes within each layer.

Update `plan()` to build `List<List<OrderedStep>>` from layered `List<List<NodeId>>`.

- [ ] **Step 5: Migrate all callers to flat-view accessors**

56 references to `plan.removals()` etc. across:
- `SimpleTransitionExecutor.execute()` — use `plan.flatRemovals()`, `plan.flatSuspensions()`, `plan.flatResumptions()`, `plan.flatAdditions()`
- `ReconciliationLoop.TenantLoop` — 6 sites: `plan()`, `faultFeedback()`, `emitCycleEvents()` methods — all use `flatX()` accessors
- `CaseTransitionExecutor` — 4 sites: `execute()`, `buildCaseDefinition()`, `buildOptimisticResult()` — use `flatX()` accessors
- `MockTransitionExecutor` — use `flatRemovals()`
- Example tests — use `flatRemovals()`, `flatAdditions()`
- `TransitionPlannerTest` — existing tests that call `plan.removals()` now get `List<List<OrderedStep>>`, need to use `plan.flatRemovals()` for flat iteration

Use `ide_find_references` on `removals()`, `additions()`, `suspensions()`, `resumptions()` to find all sites. Use `ide_replace_member` or `ide_edit_member` for each change.

- [ ] **Step 6: Run full build to verify migration**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all callers migrated, tests pass

- [ ] **Step 7: Run the layered test to verify it passes**

Run: `mvn --batch-mode -pl runtime test -Dtest=TransitionPlannerTest#layeredAdditions_groupsNodesByTopologicalDepth`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#150): TransitionPlan layered output + TransitionPlanner BFS layers + caller migration to flatX() accessors"
```

---

## Batch 2: NodeStepExecutor Extraction

Safe wrap point: per-node execution logic extracted. SimpleTransitionExecutor delegates to NodeStepExecutor. All tests pass. No behavioral change.

### Task 3: Extract NodeStepExecutor from SimpleTransitionExecutor

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/NodeStepExecutor.java`
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/SimpleTransitionExecutor.java`
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/RuntimeBeans.java` (produce NodeStepExecutor)
- Test: `runtime-core/src/test/java/io/casehub/desiredstate/runtime/NodeStepExecutorTest.java`

**Interfaces:**
- Consumes: `NodeProvisionerRouter`, `HumanNodeHandler`, `PendingApprovalHandler`, `LifecycleStepExecutor`
- Produces: `NodeStepExecutor.execute(DesiredNode, StepAction, DesiredStateGraph, String tenancyId) → StepOutcome`

- [ ] **Step 1: Write test for NodeStepExecutor provision delegation**

```java
@Test
void provision_delegatesToRouter() {
    var router = mock(NodeProvisionerRouter.class);
    when(router.provision(any(), any())).thenReturn(new ProvisionResult.Success());
    var executor = new NodeStepExecutor(router, new NoOpHumanNodeHandler(),
        new NoOpPendingApprovalHandler(), new NoOpLifecycleStepExecutor());

    DesiredNode node = new DesiredNode(NodeId.of("n1"), testSpec);
    StepOutcome outcome = executor.execute(node, StepAction.PROVISION, graph, "tenant-1");

    assertInstanceOf(StepOutcome.Succeeded.class, outcome);
    verify(router).provision(eq(node), any(ProvisionContext.class));
}
```

- [ ] **Step 2: Run test, verify fail**

- [ ] **Step 3: Create NodeStepExecutor**

Extract the four `executeX` methods from `SimpleTransitionExecutor` into `NodeStepExecutor`. Add a public `execute(DesiredNode, StepAction, DesiredStateGraph, String)` that dispatches to the private methods. Preserve all cross-cutting logic: human gating check, approval lifecycle, lifecycle hooks, OTel spans.

- [ ] **Step 4: Refactor SimpleTransitionExecutor to delegate**

Replace the four `executeX` methods with calls to `NodeStepExecutor.execute()`. The `execute(TransitionPlan, String)` method iterates flat views and delegates per node. Constructor takes `NodeStepExecutor` instead of four individual dependencies.

- [ ] **Step 5: Update RuntimeBeans to produce NodeStepExecutor**

Add `@Produces @DefaultBean` method for `NodeStepExecutor` in `RuntimeBeans`. Update `simpleTransitionExecutor()` to accept `NodeStepExecutor`.

- [ ] **Step 6: Run full test suite**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — behavioral parity with pre-extraction code

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "refactor(#150): extract NodeStepExecutor from SimpleTransitionExecutor — shared per-node execution logic"
```

---

## Batch 3: ParallelTransitionExecutor

Safe wrap point: ParallelTransitionExecutor exists, activated via preference key. Tests verify concurrent layer execution, semaphore rate limiting, failure propagation, OTel context. SimpleTransitionExecutor remains default.

### Task 4: ParallelTransitionExecutor implementation

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ParallelTransitionExecutor.java`
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/RuntimeBeans.java:85-92` (conditional production)
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DesiredStatePreferenceKeys.java` (add TRANSITION_EXECUTOR key)
- Test: `runtime-core/src/test/java/io/casehub/desiredstate/runtime/ParallelTransitionExecutorTest.java`

**Interfaces:**
- Consumes: `NodeStepExecutor.execute()`, `TransitionPlan.additions()` (layered), `NodeProvisionerRouter.maxConcurrency()`, `DesiredStateGraph.dependenciesOf()`
- Produces: `ParallelTransitionExecutor implements TransitionExecutor` with `execute(TransitionPlan, String) → TransitionResult`

- [ ] **Step 1: Write test for parallel layer execution**

```java
@Test
void independentNodes_executeInParallel() {
    // Two nodes in same layer (no dependencies)
    var nodeA = new DesiredNode(NodeId.of("A"), testSpec);
    var nodeB = new DesiredNode(NodeId.of("B"), testSpec);
    var additions = List.of(List.of(
        new OrderedStep(nodeA, StepAction.PROVISION),
        new OrderedStep(nodeB, StepAction.PROVISION)));
    var plan = new TransitionPlan(List.of(), List.of(), List.of(), additions, graph, graph);

    var executionOrder = Collections.synchronizedList(new ArrayList<String>());
    var latch = new CountDownLatch(2);
    var stepExecutor = new NodeStepExecutor(/*...*/) {
        // Override to track concurrent execution
    };

    var executor = new ParallelTransitionExecutor(stepExecutor, router, Duration.ofMinutes(5));
    TransitionResult result = executor.execute(plan, "tenant-1");

    assertEquals(2, result.outcomes().size());
    assertInstanceOf(StepOutcome.Succeeded.class, result.outcomes().get(NodeId.of("A")));
    assertInstanceOf(StepOutcome.Succeeded.class, result.outcomes().get(NodeId.of("B")));
}
```

- [ ] **Step 2: Write test for failure propagation to dependents**

```java
@Test
void failedNode_skipsDependents() {
    // Layer 0: A (fails), Layer 1: B (depends on A)
    var nodeA = new DesiredNode(NodeId.of("A"), testSpec);
    var nodeB = new DesiredNode(NodeId.of("B"), testSpec);
    var additions = List.of(
        List.of(new OrderedStep(nodeA, StepAction.PROVISION)),
        List.of(new OrderedStep(nodeB, StepAction.PROVISION)));
    var graph = factory.of(
        Map.of(NodeId.of("A"), nodeA, NodeId.of("B"), nodeB),
        Set.of(new Dependency(NodeId.of("B"), NodeId.of("A"))));
    var plan = new TransitionPlan(List.of(), List.of(), List.of(), additions, graph, graph);

    // Mock stepExecutor to fail for A
    when(stepExecutor.execute(eq(nodeA), any(), any(), any()))
        .thenReturn(new StepOutcome.Failed("provision error"));

    var executor = new ParallelTransitionExecutor(stepExecutor, router, Duration.ofMinutes(5));
    TransitionResult result = executor.execute(plan, "tenant-1");

    assertInstanceOf(StepOutcome.Failed.class, result.outcomes().get(NodeId.of("A")));
    assertInstanceOf(StepOutcome.Failed.class, result.outcomes().get(NodeId.of("B")));
    assertThat(((StepOutcome.Failed) result.outcomes().get(NodeId.of("B"))).reason())
        .contains("dependency");
}
```

- [ ] **Step 3: Write test for semaphore rate limiting**

```java
@Test
void maxConcurrency_limitsConcurrentCalls() {
    // 4 nodes in same layer, maxConcurrency=2
    // Verify no more than 2 execute simultaneously
}
```

- [ ] **Step 4: Write test for layer timeout**

```java
@Test
void layerTimeout_marksUnfinishedAsFailed() {
    // Node that blocks forever, verify timeout produces Failed outcome
}
```

- [ ] **Step 5: Run tests, verify fail**

- [ ] **Step 6: Implement ParallelTransitionExecutor**

Virtual threads via `Executors.newVirtualThreadPerTaskExecutor()`. JDK `CountDownLatch` per layer. JDK `Semaphore` per NodeType (created from `NodeProvisionerRouter` maxConcurrency values). OTel context propagation via `Context.current().with(layerSpan)`. Failed node tracking via `ConcurrentHashMap<NodeId, String>`. Layer timeout via `latch.await(timeout, SECONDS)`.

- [ ] **Step 7: Add preference key and conditional CDI production**

Add `TRANSITION_EXECUTOR` to `DesiredStatePreferenceKeys`. Update `RuntimeBeans.simpleTransitionExecutor()` to check preference and produce `ParallelTransitionExecutor` when `"parallel"` is configured.

- [ ] **Step 8: Run tests, verify pass**

Run: `mvn --batch-mode install`

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#150): add ParallelTransitionExecutor — layer-based concurrent provisioning with semaphore rate limiting"
```

---

## Batch 4: Plugin Provisioner Migration

Safe wrap point: plugin module compiles against yaml-step-runtime. Plugin YAML gains retry, parallel, barrier, select, try-catch constructs. Existing plugins work unchanged.

### Task 5: Plugin dependency migration + PluginParser ResolvedStep output

**Files:**
- Modify: `plugin/runtime/pom.xml` (yaml-step-core → yaml-step-runtime)
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginParser.java:92-124` (parseSteps → ResolvedStep)
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginDescriptor.java` (StepDef → ResolvedStep)
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginProvisionerDef.java` (StepDef → ResolvedStep)
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginModel.java` (if needed)
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/model/PluginParserTest.java`

**Interfaces:**
- Consumes: `ResolvedStep` sealed interface from yaml-step-runtime (`ResolvedStep.PluginStep`, `ResolvedStep.BlockStep`, `ResolvedStep.ParallelStep`, etc.)
- Produces: `PluginDescriptor` with `List<ResolvedStep>` step fields, lifecycle parsing into `NodeLifecycleDefinition`

- [ ] **Step 1: Write test for PluginParser producing ResolvedStep with retry decorator**

```java
@Test
void parseSteps_withRetryDecorator_producesDecoratedStep() {
    String yaml = """
        header:
          type: test-type
          version: 1
        spec: {}
        provisioner:
          provision:
            steps:
              - rest-call:
                  method: GET
                  url: http://example.com
                retry:
                  max: 3
                  backoff: exponential
                  delay: 1s
          deprovision:
            steps: []
        """;
    PluginModel model = PluginParser.parse(new ByteArrayInputStream(yaml.getBytes()));
    var steps = model.provisioner().provisionSteps();
    assertEquals(1, steps.size());
    assertInstanceOf(ResolvedStep.PluginStep.class, steps.get(0));
    assertFalse(steps.get(0).decorators().isEmpty());
    assertThat(steps.get(0).decorators()).containsKey("retry");
}
```

- [ ] **Step 2: Write test for lifecycle parsing**

```java
@Test
void parseLifecycle_producesNodeLifecycleDefinition() {
    String yaml = """
        header:
          type: test-type
          version: 1
        spec: {}
        provisioner:
          provision:
            steps: []
          deprovision:
            steps: []
        lifecycle:
          transitions:
            - from: ABSENT, to: PROVISIONING
            - from: PROVISIONING, to: PRESENT
            - from: PROVISIONING, to: DRIFTED
            - from: PRESENT, to: DEPROVISIONING
            - from: DEPROVISIONING, to: ABSENT
          on-enter:
            PRESENT:
              emit: "node.ready"
        """;
    PluginModel model = PluginParser.parse(new ByteArrayInputStream(yaml.getBytes()));
    assertNotNull(model.lifecycle());
    assertEquals(5, model.lifecycle().transitions().size());
    assertTrue(model.lifecycle().onEnter().containsKey(NodeLifecycleState.PRESENT));
}
```

- [ ] **Step 3: Run tests, verify fail**

- [ ] **Step 4: Update pom.xml dependency**

Replace `casehub-platform-yaml-step-core` with `casehub-platform-yaml-step-runtime` in `plugin/runtime/pom.xml`.

- [ ] **Step 5: Update PluginParser.parseSteps() for ResolvedStep output**

Rewrite `parseSteps()` to produce `List<ResolvedStep>` instead of `List<StepDef>`. Parse step YAML nodes into `ResolvedStep.PluginStep` for simple steps. Parse structural constructs (`parallel:`, `if:`, `try:`, `match:`, `select:`, `barrier:`, `block:`) into their corresponding `ResolvedStep` subtypes. Parse decorators (`retry:`, `loop:`, `deadline:`) into the step's `decorators()` map.

Add `parseLifecycle()` method that produces `NodeLifecycleDefinition` from the `lifecycle:` YAML section.

- [ ] **Step 6: Update PluginDescriptor and PluginProvisionerDef**

Change `List<StepDef>` fields to `List<ResolvedStep>`. Add `NodeLifecycleDefinition lifecycle` field to `PluginDescriptor` (nullable — not all plugins declare lifecycle).

- [ ] **Step 7: Run tests, verify pass**

Run: `mvn --batch-mode -pl plugin/runtime test`

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#150): migrate plugin module from yaml-step-core to yaml-step-runtime — ResolvedStep output + lifecycle parsing"
```

### Task 6: YamlPluginProvisioner + ActualStateStepExecutor migration to StructuralStepEvaluator

**Files:**
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/YamlPluginProvisioner.java`
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/ActualStateStepExecutor.java`
- Modify: `plugin/deployment/src/main/java/io/casehub/desiredstate/plugin/deployment/YamlPluginProcessor.java` (update validation)
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/YamlPluginProvisionerTest.java`
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/PluginIntegrationTest.java`

**Interfaces:**
- Consumes: `StructuralStepEvaluator.evaluate()`, `ScenarioScope` (from PrimitiveFactory), `ResolvedStep`
- Produces: Updated `YamlPluginProvisioner` using `StructuralStepEvaluator` with `ScenarioScope` lifecycle

- [ ] **Step 1: Write test for provisioner with retry decorator**

```java
@Test
void provision_withRetry_retriesOnFailure() {
    // Plugin step that fails twice then succeeds
    // Verify provision returns Success after retry
}
```

- [ ] **Step 2: Write test for backward compatibility (flat steps)**

```java
@Test
void provision_flatSteps_identicalToOldBehavior() {
    // Existing plugin with simple sequential steps
    // Verify same outcomes as before migration
}
```

- [ ] **Step 3: Run tests, verify fail**

- [ ] **Step 4: Update YamlPluginProvisioner**

Replace `StepPipelineExecutor` with `StructuralStepEvaluator`. Add `PrimitiveFactory` dependency for `ScenarioScope` creation. Wrap each provision/deprovision call in try-with-resources `ScenarioScope`. Build `StructuralStepEvaluator` per scope. Call `evaluator.preRegisterLatches()` then evaluate steps.

- [ ] **Step 5: Update ActualStateStepExecutor**

Same migration — replace `StepPipelineExecutor` with `StructuralStepEvaluator` wrapped in `ScenarioScope`.

- [ ] **Step 6: Update YamlPluginProcessor build-time validation**

Update validation to handle `ResolvedStep` model instead of `StepDef`. Add lifecycle validation (call `NodeLifecycleDefinition.validate()` at build time).

- [ ] **Step 7: Run tests, verify pass**

Run: `mvn --batch-mode -pl plugin/runtime,plugin/deployment test`

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#150): migrate YamlPluginProvisioner to StructuralStepEvaluator — plugin steps gain retry, parallel, barrier, select"
```

---

## Batch 5: StatefulNodeProvisioner + Lifecycle Enforcement

Safe wrap point: lifecycle state machines enforced for plugins that declare `lifecycle:` section. CloudEvents emitted on state transitions. Validation at build time and construction time. Full reconciliation cycle works with state machines.

### Task 7: StatefulNodeProvisioner + TransitionActionHandler

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/TransitionActionHandler.java`
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/StatefulNodeProvisioner.java`
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationEventEmitter.java` (add lifecycle methods)
- Test: `runtime-core/src/test/java/io/casehub/desiredstate/runtime/StatefulNodeProvisionerTest.java`

**Interfaces:**
- Consumes: `NodeProvisioner` (wrapped delegate), `NodeLifecycleDefinition`, `OrcStateMachine<NodeLifecycleState>` (from yaml-core), `TransitionActionHandler.execute()`
- Produces: `StatefulNodeProvisioner implements NodeProvisioner`, `TransitionActionHandler` functional interface

- [ ] **Step 1: Write test for valid provision transition**

```java
@Test
void provision_validTransition_delegatesAndTransitions() {
    var delegate = mock(NodeProvisioner.class);
    when(delegate.provision(any(), any())).thenReturn(new ProvisionResult.Success());
    when(delegate.handledTypes()).thenReturn(Set.of(NodeType.of("test")));

    var lifecycle = new NodeLifecycleDefinition(NodeType.of("test"),
        Set.of(
            new NodeLifecycleDefinition.Transition(ABSENT, PROVISIONING),
            new NodeLifecycleDefinition.Transition(PROVISIONING, PRESENT),
            new NodeLifecycleDefinition.Transition(PROVISIONING, DRIFTED)),
        Map.of(), Map.of());

    var provisioner = new StatefulNodeProvisioner(delegate, lifecycle, (a, n, s, t) -> {});
    var node = new DesiredNode(NodeId.of("n1"), testSpec);
    var result = provisioner.provision(node, new ProvisionContext("t1", graph));

    assertInstanceOf(ProvisionResult.Success.class, result);
}
```

- [ ] **Step 2: Write test for illegal transition**

```java
@Test
void provision_illegalTransition_returnsFailed() {
    // Node already in PRESENT state, provision called again
    // Verify Failed("illegal lifecycle transition")
}
```

- [ ] **Step 3: Write test for failure recovery (PROVISIONING → DRIFTED)**

```java
@Test
void provision_failure_transitionsToDrifted() {
    var delegate = mock(NodeProvisioner.class);
    when(delegate.provision(any(), any())).thenReturn(new ProvisionResult.Failed("timeout"));
    // ... setup lifecycle with PROVISIONING → DRIFTED transition
    var result = provisioner.provision(node, context);
    assertInstanceOf(ProvisionResult.Failed.class, result);
    // Verify internal state machine is in DRIFTED
}
```

- [ ] **Step 4: Write test for state reconstruction from ActualState**

```java
@Test
void stateReconstruction_derivesFromActualState() {
    // Node with PRESENT actual status
    // Verify state machine starts in PRESENT, not ABSENT
}
```

- [ ] **Step 5: Write test for event emission on state enter**

```java
@Test
void onEnterAction_emitsEvent() {
    var emitted = new ArrayList<TransitionAction>();
    var handler = (TransitionActionHandler) (action, nodeId, state, tenancyId) -> emitted.add(action);
    // ... lifecycle with on-enter PRESENT: emit "node.ready"
    provisioner.provision(node, context);
    assertThat(emitted).hasSize(2); // PROVISIONING enter + PRESENT enter
}
```

- [ ] **Step 6: Run tests, verify fail**

- [ ] **Step 7: Implement TransitionActionHandler functional interface**

```java
@FunctionalInterface
public interface TransitionActionHandler {
    void execute(TransitionAction action, NodeId nodeId, NodeLifecycleState state, String tenancyId);
}
```

- [ ] **Step 8: Implement StatefulNodeProvisioner**

Constructor takes `NodeProvisioner delegate`, `NodeLifecycleDefinition lifecycle`, `TransitionActionHandler actionHandler`. Uses `ConcurrentHashMap<NodeId, OrcStateMachine<NodeLifecycleState>>` for per-node state machines. Creates machines via `DefaultOrcStateMachine.builder()` with transitions from lifecycle definition. Registers `onEnter`/`onExit` handlers wrapped in try-catch-log guards. Implements provision/deprovision/suspend/resume flows with transient state transitions and failure recovery. Forwards `handledTypes()`, `resyncInterval()`, `maxConcurrency()` to delegate. Overrides `supportsStatefulLifecycle()` based on `lifecycle.supportsSuspendResume()`.

- [ ] **Step 9: Add lifecycle event methods to ReconciliationEventEmitter**

Add `lifecycleStateEntered(LifecycleStateEnteredData)` and `lifecycleStateExited(LifecycleStateExitedData)` methods following the same pure-function CloudEvent building pattern.

- [ ] **Step 10: Run tests, verify pass**

Run: `mvn --batch-mode -pl runtime-core test -Dtest=StatefulNodeProvisionerTest`

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat(#150): add StatefulNodeProvisioner — OrcStateMachine lifecycle enforcement with event emission"
```

### Task 8: CDI wiring — StatefulNodeProvisionerRouter + CdiTransitionActionHandler

**Files:**
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/StatefulNodeProvisionerRouter.java`
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/CdiTransitionActionHandler.java`
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/CdiNodeProvisionerRouter.java` (add `@DefaultBean`)
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/StatefulNodeProvisionerRouterTest.java`

**Interfaces:**
- Consumes: `Instance<NodeProvisioner>`, `Instance<NodeLifecycleDefinition>`, `StatefulNodeProvisioner`, `ReconciliationEventEmitter`
- Produces: `StatefulNodeProvisionerRouter extends DefaultNodeProvisionerRouter` (displaces `CdiNodeProvisionerRouter`), `CdiTransitionActionHandler implements TransitionActionHandler`

- [ ] **Step 1: Write test for provisioner wrapping**

```java
@Test
void provisioner_withLifecycleDefinition_getsWrapped() {
    // Provisioner for type "cloud-vm" + lifecycle definition for "cloud-vm"
    // Verify routing calls go through StatefulNodeProvisioner
}
```

- [ ] **Step 2: Write test for provisioner without lifecycle passes through**

```java
@Test
void provisioner_withoutLifecycle_notWrapped() {
    // Provisioner for type "simple-task" with no lifecycle definition
    // Verify routing calls go directly to delegate
}
```

- [ ] **Step 3: Write test for startup validation — CTE incompatibility**

```java
@Test
void cteActive_withSuspendResumeLifecycle_failsStartup() {
    // CaseTransitionExecutor on classpath + lifecycle with suspend transitions
    // Verify IllegalStateException at construction
}
```

- [ ] **Step 4: Run tests, verify fail**

- [ ] **Step 5: Implement CdiTransitionActionHandler**

Injects `ReconciliationEventEmitter` and `Event<CloudEvent>`. Dispatches `TransitionAction.EmitEvent` to `ReconciliationEventEmitter.lifecycleStateEntered()` / `lifecycleStateExited()`.

- [ ] **Step 6: Implement StatefulNodeProvisionerRouter**

`@ApplicationScoped`. Injects `Instance<NodeProvisioner>`, `Instance<NodeLifecycleDefinition>`, `PreferenceProvider`, `CdiTransitionActionHandler`, `Instance<TransitionExecutor>`. At construction: validates lifecycle definitions, wraps matching provisioners with `StatefulNodeProvisioner`, passes wrapped+unwrapped collection to `super()`.

- [ ] **Step 7: Add @DefaultBean to CdiNodeProvisionerRouter**

Change `CdiNodeProvisionerRouter` from `@ApplicationScoped` to `@DefaultBean @ApplicationScoped` so `StatefulNodeProvisionerRouter` can displace it.

- [ ] **Step 8: Run tests, verify pass**

Run: `mvn --batch-mode -pl runtime test`

- [ ] **Step 9: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat(#150): CDI wiring for lifecycle state machines — StatefulNodeProvisionerRouter, CdiTransitionActionHandler"
```

---

## Batch 6: Spring Support + Documentation

Safe wrap point: all three applications work on both Quarkus and Spring. Documentation updated.

### Task 9: Spring auto-configuration

**Files:**
- Modify: `runtime-spring/src/main/java/io/casehub/desiredstate/runtime/spring/DesiredStateAutoConfiguration.java`
- Modify: `plugin/spring/src/main/java/io/casehub/desiredstate/plugin/spring/DesiredStatePluginAutoConfiguration.java`

**Interfaces:**
- Consumes: all types from batches 1-5
- Produces: Spring `@Bean` methods for `NodeStepExecutor`, `ParallelTransitionExecutor` (conditional), `StatefulNodeProvisionerRouter` (wrapping pattern), `CdiTransitionActionHandler` equivalent

- [ ] **Step 1: Add Spring beans for ParallelTransitionExecutor (conditional)**

`@ConditionalOnProperty(name = "desiredstate.transition.executor", havingValue = "parallel")` for ParallelTransitionExecutor. Default `SimpleTransitionExecutor` via `@ConditionalOnMissingBean`.

- [ ] **Step 2: Add Spring beans for lifecycle support**

Spring equivalent of `StatefulNodeProvisionerRouter` — collect `NodeLifecycleDefinition` beans, wrap provisioners. Spring `TransitionActionHandler` using `ApplicationEventPublisher`.

- [ ] **Step 3: Run Spring module tests**

Run: `mvn --batch-mode -pl runtime-spring,plugin/spring test`

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat(#150): Spring auto-configuration for parallel executor and lifecycle state machines"
```

### Task 10: Documentation updates

**Files:**
- Modify: `CLAUDE.md` (module table updates)
- Modify: `ARC42STORIES.MD` (runtime-core module, new types)
- Modify: `docs/guides/consumer-guide.md` (parallel executor activation, plugin orchestration)
- Modify: `docs/guides/contributor-guide.md` (NodeStepExecutor, StatefulNodeProvisioner internals)

- [ ] **Step 1: Update CLAUDE.md module table**

Add `NodeStepExecutor`, `ParallelTransitionExecutor`, `StatefulNodeProvisioner`, `NodeLifecycleState`, `NodeLifecycleDefinition`, `TransitionActionHandler` to the runtime-core entries. Update plugin module description for ResolvedStep migration.

- [ ] **Step 2: Update ARC42STORIES**

Add `runtime-core/` as explicit module in §5. Document the NodeStepExecutor extraction pattern. Document ParallelTransitionExecutor as the middle-ground executor option.

- [ ] **Step 3: Update consumer guide**

Add section on parallel executor activation (`desiredstate.transition.executor=parallel`). Add section on plugin orchestration capabilities (retry, parallel, barrier, etc.). Add section on lifecycle state machines in plugin YAML.

- [ ] **Step 4: Update contributor guide**

Document NodeStepExecutor as the extension point for cross-cutting per-node concerns. Document StatefulNodeProvisioner decorator pattern.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "docs(#150): update CLAUDE.md, ARC42STORIES, consumer and contributor guides for orchestration primitives"
```

---

## References

- [2026-09-28-orchestration-primitives-design.md] — design spec this plan implements
- [api/src/main/java/io/casehub/desiredstate/api/TransitionPlan.java] — TransitionPlan record (56 references to migrate)
- [api/src/main/java/io/casehub/desiredstate/api/NodeProvisioner.java:55] — supportsStatefulLifecycle() existing SPI
- [runtime-core/src/main/java/io/casehub/desiredstate/runtime/SimpleTransitionExecutor.java] — extraction source for NodeStepExecutor
- [runtime-core/src/main/java/io/casehub/desiredstate/runtime/TransitionPlanner.java:120-167] — topological sort producing layers
- [runtime/src/main/java/io/casehub/desiredstate/runtime/RuntimeBeans.java:85-92] — CDI executor production
- [plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginParser.java:92-124] — parseSteps migration
- [plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/YamlPluginProvisioner.java] — StructuralStepEvaluator migration
- [platform/yaml-step-runtime/src/main/java/io/casehub/yaml/step/eval/StructuralStepEvaluator.java] — target step evaluator
- [platform/yaml-core/src/main/java/io/casehub/yaml/core/orchestration/OrcStateMachine.java] — lifecycle state machine primitive
- [engine-adapter/src/main/java/io/casehub/desiredstate/engine/CaseTransitionExecutor.java] — caller migration for flatX() accessors
- [GitHub #150] — focal issue
