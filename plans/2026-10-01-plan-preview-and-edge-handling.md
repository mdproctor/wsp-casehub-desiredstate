# Plan Preview & Edge Handling Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #130 — plan preview before execution (terraform-plan-style approval gate)
**Issue group:** #130, #159

**Goal:** Add a plan approval gate to the reconciliation loop and type-level ordering constraints to the graph/planner.

**Architecture:** PlanApprovalGate injects between plan() and execute() in ReconciliationLoop using skip-and-recheck semantics. OrderingConstraint lives on DesiredStateGraph and TransitionPlanner resolves them as virtual in-degree entries during BFS. Flat-graph fast-path skips topological sort when no edges and no constraints exist.

**Tech Stack:** Java 21, Quarkus CDI, Spring Boot auto-configuration, JUnit 5

## Global Constraints

- Foundation tier — no upward dependencies to engine or work modules from api/ or runtime-core/
- api/ is pure Java with CDI annotations provided-scope only
- All graph mutations return new immutable instances (ImmutableDesiredStateGraph pattern)
- @DefaultBean for Quarkus, @ConditionalOnMissingBean for Spring
- Every commit references an issue

---

## Batch 1: Ordering Constraints (#159) — type-level ordering + flat-graph fast-path

### Task 1: OrderingConstraint type + DesiredStateGraph + ImmutableDesiredStateGraph

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/OrderingConstraint.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/DesiredStateGraph.java`
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraph.java`
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DefaultDesiredStateGraphFactory.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraphTest.java` (existing, extend)

**Interfaces:**
- Produces: `OrderingConstraint(NodeType before, NodeType after)` — record in api/
- Produces: `DesiredStateGraph.orderingConstraints()` — default returns `Set.of()`
- Produces: `DesiredStateGraph.withOrderingConstraints(Set<OrderingConstraint>)` — returns new graph
- Produces: `DesiredStateGraphFactory.of(Collection<DesiredNode>, Collection<Dependency>, Set<OrderingConstraint>)` — default method

- [ ] **Step 1: Write failing tests for OrderingConstraint and graph constraint propagation**

Add to `ImmutableDesiredStateGraphTest.java`:

```java
@Test
void orderingConstraints_defaultEmpty() {
    DesiredStateGraph graph = factory.empty();
    assertEquals(Set.of(), graph.orderingConstraints());
}

@Test
void withOrderingConstraints_returnsNewGraphWithConstraints() {
    DesiredStateGraph graph = factory.empty();
    OrderingConstraint constraint = new OrderingConstraint(
        NodeType.of("breaker"), NodeType.of("equipment"));
    DesiredStateGraph constrained = graph.withOrderingConstraints(Set.of(constraint));
    assertEquals(Set.of(constraint), constrained.orderingConstraints());
    assertEquals(Set.of(), graph.orderingConstraints()); // original unchanged
}

@Test
void withNode_preservesOrderingConstraints() {
    OrderingConstraint constraint = new OrderingConstraint(
        NodeType.of("breaker"), NodeType.of("equipment"));
    DesiredStateGraph graph = factory.empty().withOrderingConstraints(Set.of(constraint));
    DesiredNode node = new DesiredNode(NodeId.of("n1"), new TestSpec("breaker"), HumanGating.NONE);
    DesiredStateGraph updated = graph.withNode(node);
    assertEquals(Set.of(constraint), updated.orderingConstraints());
}

@Test
void overlay_mergesOrderingConstraints() {
    OrderingConstraint c1 = new OrderingConstraint(NodeType.of("a"), NodeType.of("b"));
    OrderingConstraint c2 = new OrderingConstraint(NodeType.of("c"), NodeType.of("d"));
    DesiredStateGraph g1 = factory.empty().withOrderingConstraints(Set.of(c1));
    DesiredStateGraph g2 = factory.empty().withOrderingConstraints(Set.of(c2));
    DesiredStateGraph merged = g1.overlay(g2);
    assertEquals(Set.of(c1, c2), merged.orderingConstraints());
}

@Test
void filterByTypes_filtersConstraintsToMatchingTypes() {
    OrderingConstraint kept = new OrderingConstraint(NodeType.of("a"), NodeType.of("b"));
    OrderingConstraint removed = new OrderingConstraint(NodeType.of("a"), NodeType.of("c"));
    DesiredNode nodeA = new DesiredNode(NodeId.of("n1"), new TestSpec("a"), HumanGating.NONE);
    DesiredNode nodeB = new DesiredNode(NodeId.of("n2"), new TestSpec("b"), HumanGating.NONE);
    DesiredNode nodeC = new DesiredNode(NodeId.of("n3"), new TestSpec("c"), HumanGating.NONE);
    DesiredStateGraph graph = factory.of(List.of(nodeA, nodeB, nodeC), List.of())
        .withOrderingConstraints(Set.of(kept, removed));
    DesiredStateGraph filtered = graph.filterByTypes(Set.of(NodeType.of("a"), NodeType.of("b")));
    assertEquals(Set.of(kept), filtered.orderingConstraints());
}

@Test
void orderingConstraint_rejectsSelfReferencing() {
    assertThrows(IllegalArgumentException.class, () ->
        new OrderingConstraint(NodeType.of("a"), NodeType.of("a")));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=ImmutableDesiredStateGraphTest -Dcheckstyle.skip -Denforcer.skip`
Expected: Compilation failure — `OrderingConstraint` does not exist.

- [ ] **Step 3: Create OrderingConstraint record**

Create `api/src/main/java/io/casehub/desiredstate/api/OrderingConstraint.java`:

```java
package io.casehub.desiredstate.api;

import java.util.Objects;

public record OrderingConstraint(NodeType before, NodeType after) {
    public OrderingConstraint {
        Objects.requireNonNull(before, "OrderingConstraint.before must not be null");
        Objects.requireNonNull(after, "OrderingConstraint.after must not be null");
        if (before.equals(after)) {
            throw new IllegalArgumentException("Self-referencing ordering constraint: " + before);
        }
    }
}
```

- [ ] **Step 4: Add orderingConstraints() and withOrderingConstraints() to DesiredStateGraph**

Add to `DesiredStateGraph.java`:

```java
default Set<OrderingConstraint> orderingConstraints() {
    return Set.of();
}

DesiredStateGraph withOrderingConstraints(Set<OrderingConstraint> constraints);
```

- [ ] **Step 5: Implement in ImmutableDesiredStateGraph**

Add `orderingConstraints` field to `ImmutableDesiredStateGraph`:
- New constructor parameter: `Set<OrderingConstraint> orderingConstraints`
- Store as `Set.copyOf(orderingConstraints)`
- `orderingConstraints()` returns the field
- All `with*`/`without*` methods propagate the field to the new instance
- `overlay()` and `connect()` merge constraints via `Set.union`
- Override `filterByTypes()` to filter constraints where both types are in the filter set
- `withOrderingConstraints()` returns new graph with given constraints
- Update `empty()` factory to pass `Set.of()`

- [ ] **Step 6: Add factory default method to DesiredStateGraphFactory**

Add to `DesiredStateGraphFactory.java`:

```java
default DesiredStateGraph of(Collection<DesiredNode> nodes, Collection<Dependency> deps,
                              Set<OrderingConstraint> orderingConstraints) {
    return of(nodes, deps).withOrderingConstraints(orderingConstraints);
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=ImmutableDesiredStateGraphTest -Dcheckstyle.skip -Denforcer.skip`
Expected: All tests pass.

- [ ] **Step 8: Run ide_diagnostics to check for compilation errors**

Run `ide_diagnostics` on `api/` and `runtime-core/`.

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/OrderingConstraint.java
git add api/src/main/java/io/casehub/desiredstate/api/DesiredStateGraph.java
git add api/src/main/java/io/casehub/desiredstate/api/DesiredStateGraphFactory.java
git add runtime-core/src/main/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraph.java
git add runtime/src/test/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraphTest.java
git commit -m "feat(#159): OrderingConstraint type + DesiredStateGraph + ImmutableDesiredStateGraph support"
```

---

### Task 2: TransitionPlanner flat-graph fast-path + virtual edges

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/TransitionPlanner.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/TransitionPlannerTest.java` (existing, extend)

**Interfaces:**
- Consumes: `DesiredStateGraph.orderingConstraints()` from Task 1
- Consumes: `DesiredStateGraph.dependencies()` from existing API
- Consumes: `OrderingConstraint(NodeType before, NodeType after)` from Task 1

- [ ] **Step 1: Write failing tests for flat-graph fast-path**

Add to `TransitionPlannerTest.java`:

```java
@Test
void flatGraph_noEdgesNoConstraints_singleLayerPlan() {
    DesiredNode a = new DesiredNode(NodeId.of("a"), new TestSpec("A"), HumanGating.NONE);
    DesiredNode b = new DesiredNode(NodeId.of("b"), new TestSpec("B"), HumanGating.NONE);
    DesiredNode c = new DesiredNode(NodeId.of("c"), new TestSpec("C"), HumanGating.NONE);

    DesiredStateGraph desired = factory.of(List.of(a, b, c), List.of());
    ActualState actual = new ActualState(Map.of());

    TransitionPlan plan = planner.plan(desired, actual);

    assertEquals(1, plan.additions().size(), "Should be single layer");
    assertEquals(3, plan.additions().getFirst().size());
}
```

- [ ] **Step 2: Write failing tests for virtual edges from ordering constraints**

```java
@Test
void orderingConstraints_enforcesTypeLevelOrdering() {
    NodeType typeA = NodeType.of("breaker");
    NodeType typeB = NodeType.of("equipment");
    DesiredNode a1 = new DesiredNode(NodeId.of("a1"), new TestSpec("breaker"), HumanGating.NONE);
    DesiredNode b1 = new DesiredNode(NodeId.of("b1"), new TestSpec("equipment"), HumanGating.NONE);
    DesiredNode b2 = new DesiredNode(NodeId.of("b2"), new TestSpec("equipment"), HumanGating.NONE);

    OrderingConstraint constraint = new OrderingConstraint(typeA, typeB);
    DesiredStateGraph desired = factory.of(List.of(a1, b1, b2), List.of())
        .withOrderingConstraints(Set.of(constraint));

    ActualState actual = new ActualState(Map.of());
    TransitionPlan plan = planner.plan(desired, actual);

    List<NodeId> order = plan.flatAdditions().stream()
        .map(s -> s.node().id()).toList();
    int idxA1 = order.indexOf(NodeId.of("a1"));
    int idxB1 = order.indexOf(NodeId.of("b1"));
    int idxB2 = order.indexOf(NodeId.of("b2"));

    assertTrue(idxA1 < idxB1, "breaker should come before equipment");
    assertTrue(idxA1 < idxB2, "breaker should come before equipment");
}

@Test
void orderingConstraints_chainedConstraints() {
    NodeType typeA = NodeType.of("first");
    NodeType typeB = NodeType.of("second");
    NodeType typeC = NodeType.of("third");
    DesiredNode a = new DesiredNode(NodeId.of("a"), new TestSpec("first"), HumanGating.NONE);
    DesiredNode b = new DesiredNode(NodeId.of("b"), new TestSpec("second"), HumanGating.NONE);
    DesiredNode c = new DesiredNode(NodeId.of("c"), new TestSpec("third"), HumanGating.NONE);

    DesiredStateGraph desired = factory.of(List.of(a, b, c), List.of())
        .withOrderingConstraints(Set.of(
            new OrderingConstraint(typeA, typeB),
            new OrderingConstraint(typeB, typeC)));

    ActualState actual = new ActualState(Map.of());
    TransitionPlan plan = planner.plan(desired, actual);

    List<NodeId> order = plan.flatAdditions().stream()
        .map(s -> s.node().id()).toList();
    assertTrue(order.indexOf(NodeId.of("a")) < order.indexOf(NodeId.of("b")));
    assertTrue(order.indexOf(NodeId.of("b")) < order.indexOf(NodeId.of("c")));
}

@Test
void orderingConstraints_conflictingThrowsCycleException() {
    NodeType typeA = NodeType.of("a-type");
    NodeType typeB = NodeType.of("b-type");
    DesiredNode a = new DesiredNode(NodeId.of("a"), new TestSpec("a-type"), HumanGating.NONE);
    DesiredNode b = new DesiredNode(NodeId.of("b"), new TestSpec("b-type"), HumanGating.NONE);

    DesiredStateGraph desired = factory.of(List.of(a, b), List.of())
        .withOrderingConstraints(Set.of(
            new OrderingConstraint(typeA, typeB),
            new OrderingConstraint(typeB, typeA)));

    ActualState actual = new ActualState(Map.of());
    assertThrows(IllegalStateException.class, () -> planner.plan(desired, actual));
}

@Test
void orderingConstraints_mixedWithRealEdges() {
    NodeType typeX = NodeType.of("x-type");
    NodeType typeY = NodeType.of("y-type");
    DesiredNode x = new DesiredNode(NodeId.of("x"), new TestSpec("x-type"), HumanGating.NONE);
    DesiredNode y = new DesiredNode(NodeId.of("y"), new TestSpec("y-type"), HumanGating.NONE);
    DesiredNode z = new DesiredNode(NodeId.of("z"), new TestSpec("z-type"), HumanGating.NONE);

    // z depends on y (real edge), x before y (ordering constraint)
    DesiredStateGraph desired = factory.of(
        List.of(x, y, z),
        List.of(new Dependency(NodeId.of("z"), NodeId.of("y"))))
        .withOrderingConstraints(Set.of(new OrderingConstraint(typeX, typeY)));

    ActualState actual = new ActualState(Map.of());
    TransitionPlan plan = planner.plan(desired, actual);

    List<NodeId> order = plan.flatAdditions().stream()
        .map(s -> s.node().id()).toList();
    assertTrue(order.indexOf(NodeId.of("x")) < order.indexOf(NodeId.of("y")));
    assertTrue(order.indexOf(NodeId.of("y")) < order.indexOf(NodeId.of("z")));
}

@Test
void flatGraph_withConstraints_notFastPath() {
    NodeType typeA = NodeType.of("a-type");
    NodeType typeB = NodeType.of("b-type");
    DesiredNode a = new DesiredNode(NodeId.of("a"), new TestSpec("a-type"), HumanGating.NONE);
    DesiredNode b = new DesiredNode(NodeId.of("b"), new TestSpec("b-type"), HumanGating.NONE);

    DesiredStateGraph desired = factory.of(List.of(a, b), List.of())
        .withOrderingConstraints(Set.of(new OrderingConstraint(typeA, typeB)));

    ActualState actual = new ActualState(Map.of());
    TransitionPlan plan = planner.plan(desired, actual);

    // Not single-layer — constraint enforces ordering
    assertEquals(2, plan.additions().size(), "Should be two layers due to constraint");
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=TransitionPlannerTest -Dcheckstyle.skip -Denforcer.skip`
Expected: Tests fail — new test methods reference `OrderingConstraint`, `withOrderingConstraints`.

- [ ] **Step 4: Implement flat-graph fast-path in TransitionPlanner.plan()**

After the `toAdd`/`toSuspend`/`toResume` computation (line ~91), before the topological sort calls, add the fast-path check:

```java
boolean hasEdges = !desired.dependencies().isEmpty();
boolean hasConstraints = !desired.orderingConstraints().isEmpty();
if (!hasEdges && !hasConstraints) {
    List<OrderedStep> addSteps = toAdd.stream()
        .map(id -> new OrderedStep(desired.nodes().get(id), StepAction.PROVISION)).toList();
    List<OrderedStep> suspendSteps = toSuspend.stream()
        .map(id -> new OrderedStep(desired.nodes().get(id), StepAction.SUSPEND)).toList();
    List<OrderedStep> resumeSteps = toResume.stream()
        .map(id -> new OrderedStep(desired.nodes().get(id), StepAction.RESUME)).toList();
    DesiredStateGraph before = previousDesired != null ? previousDesired : desired;
    return new TransitionPlan(
        List.of(removals),
        suspendSteps.isEmpty() ? List.of() : List.of(suspendSteps),
        resumeSteps.isEmpty() ? List.of() : List.of(resumeSteps),
        addSteps.isEmpty() ? List.of() : List.of(addSteps),
        before, desired);
}
```

- [ ] **Step 5: Implement virtual edges in topologicalSort()**

Extend `topologicalSort()` to handle ordering constraints:

1. After initial in-degree counting from real dependencies, add virtual in-degree for constraint-matching pairs:

```java
Map<NodeId, Set<NodeId>> virtualReverse = new HashMap<>();
for (OrderingConstraint c : graph.orderingConstraints()) {
    Set<NodeId> beforeNodes = nodesOfType(toSort, graph, c.before());
    Set<NodeId> afterNodes = nodesOfType(toSort, graph, c.after());
    for (NodeId afterNode : afterNodes) {
        inDegree.merge(afterNode, beforeNodes.size(), Integer::sum);
    }
    for (NodeId beforeNode : beforeNodes) {
        virtualReverse.computeIfAbsent(beforeNode, k -> new HashSet<>()).addAll(afterNodes);
    }
}
```

2. In the BFS loop, after processing real dependents, add virtual dependent processing:

```java
Set<NodeId> virtualDeps = virtualReverse.getOrDefault(current, Set.of());
for (NodeId dependent : virtualDeps) {
    if (toSort.contains(dependent)) {
        int newDegree = inDegree.merge(dependent, -1, Integer::sum);
        if (newDegree == 0) {
            queue.add(dependent);
        }
    }
}
```

3. Add helper method:

```java
private Set<NodeId> nodesOfType(Set<NodeId> candidates, DesiredStateGraph graph, NodeType type) {
    Set<NodeId> result = new HashSet<>();
    for (NodeId nodeId : candidates) {
        DesiredNode node = graph.nodes().get(nodeId);
        if (node != null && node.type().equals(type)) {
            result.add(nodeId);
        }
    }
    return result;
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=TransitionPlannerTest -Dcheckstyle.skip -Denforcer.skip`
Expected: All tests pass.

- [ ] **Step 7: Run full runtime module tests**

Run: `mvn --batch-mode test -pl runtime -Dcheckstyle.skip -Denforcer.skip`
Expected: All tests pass (no regressions in existing tests).

- [ ] **Step 8: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/desiredstate/runtime/TransitionPlanner.java
git add runtime/src/test/java/io/casehub/desiredstate/runtime/TransitionPlannerTest.java
git commit -m "feat(#159): flat-graph fast-path + virtual edges from ordering constraints in TransitionPlanner"
```

---

## Batch 2: Plan Approval Gate (#130) — approval types + gate + loop integration

### Task 3: Plan approval API types + SPIs

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/PlanApprovalDecision.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/GateDecision.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/PendingPlan.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/PlanApprovalPolicy.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/PlanApprovalHandler.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/PlanAwaitingApprovalData.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/PlanApprovedData.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/PlanRejectedData.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/PlanInvalidatedData.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/FaultType.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/DesiredStateEventTypes.java`

**Interfaces:**
- Produces: `PlanApprovalPolicy.evaluate(TransitionPlan, String tenancyId) → PlanApprovalDecision`
- Produces: `PlanApprovalHandler.submit(TransitionPlan, String, String) → String`
- Produces: `PlanApprovalHandler.check(String, String) → ApprovalCheckResult`
- Produces: `PlanApprovalHandler.cancel(String, String) → void`
- Produces: `GateDecision.Execute | AwaitingApproval | Rejected`
- Produces: `PendingPlan(TransitionPlan, String, int)`
- Produces: `FaultType.PLAN_REJECTED`
- Produces: `DesiredStateEventTypes.PLAN_*` constants

- [ ] **Step 1: Create all API types**

Create each file. These are pure types — no test cycle needed because they're records/sealed interfaces with no logic beyond validation.

`PlanApprovalDecision.java`:
```java
package io.casehub.desiredstate.api;

public sealed interface PlanApprovalDecision {
    record AutoApprove() implements PlanApprovalDecision {}
    record RequireApproval(String reason) implements PlanApprovalDecision {}
}
```

`GateDecision.java`:
```java
package io.casehub.desiredstate.api;

public sealed interface GateDecision {
    record Execute(TransitionPlan plan) implements GateDecision {}
    record AwaitingApproval(String planReference) implements GateDecision {}
    record Rejected(String planReference, String reason) implements GateDecision {}
}
```

`PendingPlan.java`:
```java
package io.casehub.desiredstate.api;

import java.util.Objects;

public record PendingPlan(TransitionPlan plan, String planReference, int desiredVersion) {
    public PendingPlan {
        Objects.requireNonNull(plan);
        Objects.requireNonNull(planReference);
    }
}
```

`PlanApprovalPolicy.java`:
```java
package io.casehub.desiredstate.api;

public interface PlanApprovalPolicy {
    PlanApprovalDecision evaluate(TransitionPlan plan, String tenancyId);
}
```

`PlanApprovalHandler.java`:
```java
package io.casehub.desiredstate.api;

public interface PlanApprovalHandler {
    String submit(TransitionPlan plan, String tenancyId, String reason);
    ApprovalCheckResult check(String planReference, String tenancyId);
    void cancel(String planReference, String tenancyId);
}
```

`PlanAwaitingApprovalData.java`:
```java
package io.casehub.desiredstate.api;

public record PlanAwaitingApprovalData(
    String tenancyId, String planReference,
    int additionCount, int removalCount, int suspensionCount, int resumptionCount,
    String reason) {}
```

`PlanApprovedData.java`:
```java
package io.casehub.desiredstate.api;

public record PlanApprovedData(String tenancyId, String planReference, PlanApproval approval) {}
```

`PlanRejectedData.java`:
```java
package io.casehub.desiredstate.api;

public record PlanRejectedData(String tenancyId, String planReference, String reason) {}
```

`PlanInvalidatedData.java`:
```java
package io.casehub.desiredstate.api;

public record PlanInvalidatedData(String tenancyId, String planReference, String reason) {}
```

- [ ] **Step 2: Add PLAN_REJECTED to FaultType**

Modify `FaultType.java` — add `PLAN_REJECTED` to the enum:

```java
public enum FaultType {
    NODE_DESTROYED, NODE_DEGRADED, PROVISION_FAILED, DEPROVISION_FAILED,
    SUSPEND_FAILED, RESUME_FAILED,
    HUMAN_NODE_TIMEOUT, DEPENDENCY_UNAVAILABLE, APPROVAL_REJECTED,
    PLAN_REJECTED
}
```

- [ ] **Step 3: Add plan event type constants to DesiredStateEventTypes**

Add to `DesiredStateEventTypes.java`:

```java
public static final String PLAN_AWAITING_APPROVAL =
    "io.casehub.desiredstate.plan.awaiting-approval";
public static final String PLAN_APPROVED =
    "io.casehub.desiredstate.plan.approved";
public static final String PLAN_REJECTED =
    "io.casehub.desiredstate.plan.rejected";
public static final String PLAN_INVALIDATED =
    "io.casehub.desiredstate.plan.invalidated";
```

- [ ] **Step 4: Compile api/ module to verify**

Run: `mvn --batch-mode compile -pl api -Dcheckstyle.skip -Denforcer.skip`
Expected: Compilation success.

- [ ] **Step 5: Run ide_diagnostics on api/**

Verify no errors across the module.

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/PlanApprovalDecision.java
git add api/src/main/java/io/casehub/desiredstate/api/GateDecision.java
git add api/src/main/java/io/casehub/desiredstate/api/PendingPlan.java
git add api/src/main/java/io/casehub/desiredstate/api/PlanApprovalPolicy.java
git add api/src/main/java/io/casehub/desiredstate/api/PlanApprovalHandler.java
git add api/src/main/java/io/casehub/desiredstate/api/PlanAwaitingApprovalData.java
git add api/src/main/java/io/casehub/desiredstate/api/PlanApprovedData.java
git add api/src/main/java/io/casehub/desiredstate/api/PlanRejectedData.java
git add api/src/main/java/io/casehub/desiredstate/api/PlanInvalidatedData.java
git add api/src/main/java/io/casehub/desiredstate/api/FaultType.java
git add api/src/main/java/io/casehub/desiredstate/api/DesiredStateEventTypes.java
git commit -m "feat(#130): plan approval API types, SPIs, CloudEvent data, and event type constants"
```

---

### Task 4: PlanApprovalGate + ReconciliationLoop integration + CDI/Spring wiring

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/PlanApprovalGate.java`
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/NoOpPlanApprovalPolicy.java`
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/NoOpPlanApprovalHandler.java`
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java`
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/RuntimeBeans.java`
- Modify: `runtime-spring/src/main/java/io/casehub/desiredstate/runtime/spring/DesiredStateRuntimeAutoConfiguration.java`
- Create: `runtime/src/test/java/io/casehub/desiredstate/runtime/PlanApprovalGateTest.java`
- Create: `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopPlanApprovalTest.java`

**Interfaces:**
- Consumes: `PlanApprovalPolicy`, `PlanApprovalHandler`, `GateDecision`, `PendingPlan`, `ApprovalCheckResult` from Task 3
- Consumes: `ReconciliationLoop` constructor from existing code
- Produces: `PlanApprovalGate.checkPending(String, int) → Optional<GateDecision>`
- Produces: `PlanApprovalGate.evaluateNewPlan(TransitionPlan, String) → GateDecision`

- [ ] **Step 1: Write failing tests for PlanApprovalGate**

Create `runtime/src/test/java/io/casehub/desiredstate/runtime/PlanApprovalGateTest.java`:

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;

class PlanApprovalGateTest {

    private final DefaultDesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();

    private TransitionPlan makePlan(int version) {
        DesiredStateGraph before = factory.empty();
        DesiredNode node = new DesiredNode(NodeId.of("n1"),
            new SimpleSpec(), HumanGating.NONE);
        DesiredStateGraph after = factory.of(List.of(node), List.of());
        return new TransitionPlan(
            List.of(), List.of(), List.of(),
            List.of(List.of(new OrderedStep(node, StepAction.PROVISION))),
            before, after);
    }

    @Test
    void checkPending_noPendingPlan_returnsEmpty() {
        PlanApprovalGate gate = new PlanApprovalGate(
            (plan, tid) -> new PlanApprovalDecision.AutoApprove(),
            new StubHandler());
        Optional<GateDecision> result = gate.checkPending("t1", 1);
        assertTrue(result.isEmpty());
    }

    @Test
    void evaluateNewPlan_autoApprove_returnsExecute() {
        PlanApprovalGate gate = new PlanApprovalGate(
            (plan, tid) -> new PlanApprovalDecision.AutoApprove(),
            new StubHandler());
        TransitionPlan plan = makePlan(1);
        GateDecision decision = gate.evaluateNewPlan(plan, "t1");
        assertInstanceOf(GateDecision.Execute.class, decision);
    }

    @Test
    void evaluateNewPlan_requireApproval_storesAndReturnsAwaiting() {
        StubHandler handler = new StubHandler();
        PlanApprovalGate gate = new PlanApprovalGate(
            (plan, tid) -> new PlanApprovalDecision.RequireApproval("topology change"),
            handler);
        TransitionPlan plan = makePlan(1);
        GateDecision decision = gate.evaluateNewPlan(plan, "t1");
        assertInstanceOf(GateDecision.AwaitingApproval.class, decision);
        assertTrue(handler.submitted);
    }

    @Test
    void checkPending_versionMatch_approved_returnsExecute() {
        StubHandler handler = new StubHandler();
        handler.checkResult = new ApprovalCheckResult.Approved(
            new PlanApproval("ref1", "admin", java.time.Instant.now()));
        PlanApprovalGate gate = new PlanApprovalGate(
            (plan, tid) -> new PlanApprovalDecision.RequireApproval("review"),
            handler);
        TransitionPlan plan = makePlan(1);
        gate.evaluateNewPlan(plan, "t1");
        Optional<GateDecision> result = gate.checkPending("t1", plan.after().version());
        assertTrue(result.isPresent());
        assertInstanceOf(GateDecision.Execute.class, result.get());
    }

    @Test
    void checkPending_versionMismatch_invalidates() {
        StubHandler handler = new StubHandler();
        handler.checkResult = new ApprovalCheckResult.Pending("ref1");
        PlanApprovalGate gate = new PlanApprovalGate(
            (plan, tid) -> new PlanApprovalDecision.RequireApproval("review"),
            handler);
        TransitionPlan plan = makePlan(1);
        gate.evaluateNewPlan(plan, "t1");
        Optional<GateDecision> result = gate.checkPending("t1", plan.after().version() + 99);
        assertTrue(result.isEmpty());
        assertTrue(handler.cancelled);
    }

    @Test
    void checkPending_versionMatch_rejected_returnsRejected() {
        StubHandler handler = new StubHandler();
        handler.checkResult = new ApprovalCheckResult.Rejected("ref1", "too risky");
        PlanApprovalGate gate = new PlanApprovalGate(
            (plan, tid) -> new PlanApprovalDecision.RequireApproval("review"),
            handler);
        TransitionPlan plan = makePlan(1);
        gate.evaluateNewPlan(plan, "t1");
        Optional<GateDecision> result = gate.checkPending("t1", plan.after().version());
        assertTrue(result.isPresent());
        assertInstanceOf(GateDecision.Rejected.class, result.get());
    }

    record SimpleSpec() implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("simple"); }
    }

    static class StubHandler implements PlanApprovalHandler {
        boolean submitted = false;
        boolean cancelled = false;
        ApprovalCheckResult checkResult = new ApprovalCheckResult.Pending("ref1");

        @Override
        public String submit(TransitionPlan plan, String tenancyId, String reason) {
            submitted = true;
            return "ref1";
        }
        @Override
        public ApprovalCheckResult check(String planReference, String tenancyId) {
            return checkResult;
        }
        @Override
        public void cancel(String planReference, String tenancyId) {
            cancelled = true;
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=PlanApprovalGateTest -Dcheckstyle.skip -Denforcer.skip`
Expected: Compilation failure — `PlanApprovalGate` does not exist.

- [ ] **Step 3: Implement PlanApprovalGate**

Create `runtime-core/src/main/java/io/casehub/desiredstate/runtime/PlanApprovalGate.java`:

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.*;

import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

public class PlanApprovalGate {

    private final PlanApprovalPolicy policy;
    private final PlanApprovalHandler handler;
    private final ConcurrentHashMap<String, PendingPlan> pendingPlans = new ConcurrentHashMap<>();

    public PlanApprovalGate(PlanApprovalPolicy policy, PlanApprovalHandler handler) {
        this.policy = policy;
        this.handler = handler;
    }

    public Optional<GateDecision> checkPending(String tenancyId, int currentDesiredVersion) {
        PendingPlan pending = pendingPlans.get(tenancyId);
        if (pending == null) {
            return Optional.empty();
        }

        if (pending.desiredVersion() != currentDesiredVersion) {
            pendingPlans.remove(tenancyId);
            handler.cancel(pending.planReference(), tenancyId);
            return Optional.empty();
        }

        ApprovalCheckResult result = handler.check(pending.planReference(), tenancyId);
        return switch (result) {
            case ApprovalCheckResult.Approved approved -> {
                pendingPlans.remove(tenancyId);
                yield Optional.of(new GateDecision.Execute(pending.plan()));
            }
            case ApprovalCheckResult.Pending p ->
                Optional.of(new GateDecision.AwaitingApproval(p.planReference()));
            case ApprovalCheckResult.Rejected rejected -> {
                pendingPlans.remove(tenancyId);
                yield Optional.of(new GateDecision.Rejected(
                    rejected.planReference(), rejected.reason()));
            }
            case ApprovalCheckResult.None none -> {
                pendingPlans.remove(tenancyId);
                yield Optional.empty();
            }
        };
    }

    public GateDecision evaluateNewPlan(TransitionPlan plan, String tenancyId) {
        PlanApprovalDecision decision = policy.evaluate(plan, tenancyId);
        return switch (decision) {
            case PlanApprovalDecision.AutoApprove a ->
                new GateDecision.Execute(plan);
            case PlanApprovalDecision.RequireApproval r -> {
                String reference = handler.submit(plan, tenancyId, r.reason());
                pendingPlans.put(tenancyId,
                    new PendingPlan(plan, reference, plan.after().version()));
                yield new GateDecision.AwaitingApproval(reference);
            }
        };
    }
}
```

- [ ] **Step 4: Run PlanApprovalGate tests to verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=PlanApprovalGateTest -Dcheckstyle.skip -Denforcer.skip`
Expected: All tests pass.

- [ ] **Step 5: Integrate PlanApprovalGate into ReconciliationLoop**

Modify `ReconciliationLoop.java`:

1. Add field: `private final PlanApprovalGate approvalGate;`
2. Update full constructor to accept `PlanApprovalGate approvalGate` parameter (after `exemptionStore`). Default: `this.approvalGate = approvalGate != null ? approvalGate : new PlanApprovalGate(...)` with inline NoOp implementations.
3. Update protected constructor to pass `null` for approvalGate.
4. Update `Builder` class: add `private PlanApprovalGate approvalGate;` field, `.approvalGate(PlanApprovalGate)` method, pass to constructor in `build()`.
5. In `TenantLoop.reconcile()`, add two integration points as specified in the design spec (Section 5.2).

- [ ] **Step 6: Create NoOp @DefaultBean implementations**

Create `runtime/src/main/java/io/casehub/desiredstate/runtime/NoOpPlanApprovalPolicy.java`:
```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.PlanApprovalDecision;
import io.casehub.desiredstate.api.PlanApprovalPolicy;
import io.casehub.desiredstate.api.TransitionPlan;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpPlanApprovalPolicy implements PlanApprovalPolicy {
    @Override
    public PlanApprovalDecision evaluate(TransitionPlan plan, String tenancyId) {
        return new PlanApprovalDecision.AutoApprove();
    }
}
```

Create `runtime/src/main/java/io/casehub/desiredstate/runtime/NoOpPlanApprovalHandler.java`:
```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.ApprovalCheckResult;
import io.casehub.desiredstate.api.PlanApprovalHandler;
import io.casehub.desiredstate.api.TransitionPlan;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpPlanApprovalHandler implements PlanApprovalHandler {
    @Override
    public String submit(TransitionPlan plan, String tenancyId, String reason) {
        return "noop";
    }
    @Override
    public ApprovalCheckResult check(String planReference, String tenancyId) {
        return new ApprovalCheckResult.None();
    }
    @Override
    public void cancel(String planReference, String tenancyId) {}
}
```

- [ ] **Step 7: Update RuntimeBeans to produce PlanApprovalGate and update ReconciliationLoop producer**

Add to `RuntimeBeans.java`:
```java
@Produces
@ApplicationScoped
public PlanApprovalGate planApprovalGate(PlanApprovalPolicy policy, PlanApprovalHandler handler) {
    return new PlanApprovalGate(policy, handler);
}
```

Update the `reconciliationLoop` producer to accept `PlanApprovalGate` parameter and pass it to the constructor.

- [ ] **Step 8: Update Spring DesiredStateRuntimeAutoConfiguration**

Add to `DesiredStateRuntimeAutoConfiguration.java`:
```java
@Bean
@ConditionalOnMissingBean
public PlanApprovalPolicy planApprovalPolicy() {
    return (plan, tenancyId) -> new PlanApprovalDecision.AutoApprove();
}

@Bean
@ConditionalOnMissingBean
public PlanApprovalHandler planApprovalHandler() {
    return new PlanApprovalHandler() {
        @Override public String submit(TransitionPlan plan, String tenancyId, String reason) { return "noop"; }
        @Override public ApprovalCheckResult check(String ref, String tenancyId) { return new ApprovalCheckResult.None(); }
        @Override public void cancel(String ref, String tenancyId) {}
    };
}

@Bean
@ConditionalOnMissingBean
public PlanApprovalGate planApprovalGate(PlanApprovalPolicy policy, PlanApprovalHandler handler) {
    return new PlanApprovalGate(policy, handler);
}
```

Update the `reconciliationLoop` bean to accept `PlanApprovalGate` and pass it.

- [ ] **Step 9: Write ReconciliationLoop plan approval integration test**

Create `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopPlanApprovalTest.java` with tests:
- Auto-approve policy: plan executes immediately (same as current behavior)
- RequireApproval policy: plan stored, loop returns, next cycle checks
- Approved plan: execution proceeds
- Rejected plan: fault event emitted
- Desired state change during pending: plan invalidated, re-planned next cycle

- [ ] **Step 10: Run all tests**

Run: `mvn --batch-mode test -pl runtime -Dcheckstyle.skip -Denforcer.skip`
Expected: All tests pass.

Run: `mvn --batch-mode test -pl runtime-spring -Dcheckstyle.skip -Denforcer.skip`
Expected: All tests pass.

- [ ] **Step 11: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/desiredstate/runtime/PlanApprovalGate.java
git add runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java
git add runtime/src/main/java/io/casehub/desiredstate/runtime/NoOpPlanApprovalPolicy.java
git add runtime/src/main/java/io/casehub/desiredstate/runtime/NoOpPlanApprovalHandler.java
git add runtime/src/main/java/io/casehub/desiredstate/runtime/RuntimeBeans.java
git add runtime-spring/src/main/java/io/casehub/desiredstate/runtime/spring/DesiredStateRuntimeAutoConfiguration.java
git add runtime/src/test/java/io/casehub/desiredstate/runtime/PlanApprovalGateTest.java
git add runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopPlanApprovalTest.java
git commit -m "feat(#130): PlanApprovalGate, ReconciliationLoop integration, CDI + Spring wiring"
```

---

## Batch 3: Surface Integration — ordering constraints on YAML, annotations, TS DSL

### Task 5: YAML surface ordering constraints

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlGraph.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlOrderingConstraint.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGoalCompilerFactory.java` (add constraint resolution)
- Modify or create: YAML surface test for ordering constraints

**Interfaces:**
- Consumes: `OrderingConstraint(NodeType, NodeType)` from Task 1
- Consumes: `DesiredStateGraph.withOrderingConstraints()` from Task 1

- [ ] **Step 1: Create YamlOrderingConstraint model**

Create `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlOrderingConstraint.java`:
```java
package io.casehub.desiredstate.yaml.model;

public record YamlOrderingConstraint(String before, String after) {}
```

- [ ] **Step 2: Add orderingConstraints field to YamlGraph**

Add `List<YamlOrderingConstraint> orderingConstraints` parameter to `YamlGraph` record. Default to `List.of()` in compact constructor.

- [ ] **Step 3: Resolve constraints in YamlGoalCompilerFactory**

In the factory method that compiles `YamlGraph` → `DesiredStateGraph`, resolve `YamlOrderingConstraint` type names to `NodeType` via `NodeSpecRegistry` and call `graph.withOrderingConstraints()`.

- [ ] **Step 4: Write test for YAML ordering constraint parsing and compilation**

Test that a YAML graph with `orderingConstraints` compiles to a `DesiredStateGraph` with the correct `OrderingConstraint` set.

- [ ] **Step 5: Run yaml module tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dcheckstyle.skip -Denforcer.skip`
Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add yaml/runtime/
git commit -m "feat(#159): YAML surface ordering constraint support"
```

---

### Task 6: Annotation surface ordering constraints

**Files:**
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/OrderBefore.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DesiredState.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphDescriptor.java`
- Modify: `annotations/core/src/main/java/io/casehub/desiredstate/annotations/core/DescriptorScanner.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GoalCompilerFactory.java`

**Interfaces:**
- Consumes: `OrderingConstraint(NodeType, NodeType)` from Task 1
- Consumes: `DesiredStateGraph.withOrderingConstraints()` from Task 1

- [ ] **Step 1: Create @OrderBefore annotation**

Create `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/OrderBefore.java`:
```java
package io.casehub.desiredstate.annotations;

import io.casehub.desiredstate.api.NodeSpec;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface OrderBefore {
    Class<? extends NodeSpec> value();
    Class<? extends NodeSpec> after();
}
```

- [ ] **Step 2: Add orderBefore() to @DesiredState**

Add `OrderBefore[] orderBefore() default {};` to the existing `@DesiredState` annotation.

- [ ] **Step 3: Extend GraphDescriptor with ordering constraint descriptors**

Add `List<OrderingConstraintDescriptor> orderingConstraints` to `GraphDescriptor` record. Create `OrderingConstraintDescriptor(String beforeType, String afterType)` record.

- [ ] **Step 4: Extend DescriptorScanner to extract @OrderBefore**

In `DescriptorScanner`, scan `@DesiredState.orderBefore()` annotations, resolve `NodeSpec.nodeType()` for each class, and add `OrderingConstraintDescriptor` entries to `GraphDescriptor`.

- [ ] **Step 5: Extend GoalCompilerFactory to apply constraints**

When building the graph from `GraphDescriptor`, resolve `OrderingConstraintDescriptor` entries to `OrderingConstraint` and call `graph.withOrderingConstraints()`.

- [ ] **Step 6: Write test for annotation-based ordering constraints**

- [ ] **Step 7: Run annotation module tests**

Run: `mvn --batch-mode test -pl annotations/runtime,annotations/core -Dcheckstyle.skip -Denforcer.skip`
Expected: All tests pass.

- [ ] **Step 8: Commit**

```bash
git add annotations/
git commit -m "feat(#159): annotation surface ordering constraint support (@OrderBefore)"
```

---

### Task 7: TypeScript DSL surface ordering constraints

**Files:**
- Modify: `ts-dsl/runtime/src/main/java/io/casehub/desiredstate/ts/TsEnvelope.java`
- Create: `ts-dsl/runtime/src/main/java/io/casehub/desiredstate/ts/TsOrderingConstraint.java`
- Modify: `ts-dsl/runtime/src/main/java/io/casehub/desiredstate/ts/TsGoalCompilerFactory.java`
- Modify: `ts-dsl/sdk/src/` (TypeScript SDK — `defineGraph()` types)

**Interfaces:**
- Consumes: `OrderingConstraint(NodeType, NodeType)` from Task 1
- Consumes: `DesiredStateGraph.withOrderingConstraints()` from Task 1

- [ ] **Step 1: Create TsOrderingConstraint record**

Create `ts-dsl/runtime/src/main/java/io/casehub/desiredstate/ts/TsOrderingConstraint.java`:
```java
package io.casehub.desiredstate.ts;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public record TsOrderingConstraint(String before, String after) {}
```

- [ ] **Step 2: Add orderingConstraints to TsEnvelope**

Add `List<TsOrderingConstraint> orderingConstraints` parameter to `TsEnvelope` record. Default to `List.of()` in compact constructor.

- [ ] **Step 3: Resolve constraints in TsGoalCompilerFactory**

In the factory method that compiles `TsEnvelope` → `DesiredStateGraph`, resolve type names to `NodeType` and call `graph.withOrderingConstraints()`.

- [ ] **Step 4: Update TypeScript SDK types**

Add `orderingConstraints` to `defineGraph()` input type:
```typescript
orderingConstraints?: Array<{ before: string; after: string }>
```

Update envelope transformation to pass constraints through.

- [ ] **Step 5: Write test for TS DSL ordering constraints**

- [ ] **Step 6: Run ts-dsl module tests**

Run: `mvn --batch-mode test -pl ts-dsl/runtime -Dcheckstyle.skip -Denforcer.skip`
Expected: All tests pass.

- [ ] **Step 7: Commit**

```bash
git add ts-dsl/
git commit -m "feat(#159): TypeScript DSL surface ordering constraint support"
```

---

## References

- [2026-10-01-plan-preview-and-edge-handling-design.md] — design spec this plan implements
- `ReconciliationLoop.java:127-177` — constructor and field layout
- `ReconciliationLoop.java:624-671` — reconcile() flow
- `RuntimeBeans.java:142-162` — ReconciliationLoop CDI producer
- `DesiredStateRuntimeAutoConfiguration.java:153-173` — Spring ReconciliationLoop bean
- `TransitionPlanner.java:134-186` — topologicalSort BFS algorithm
- `ImmutableDesiredStateGraph.java` — graph implementation
- `TransitionPlannerTest.java` — existing test patterns
- `ImmutableDesiredStateGraphTest.java` — existing graph test patterns
- `YamlGraph.java` — YAML model record pattern
- `TsEnvelope.java` — TS DSL model record pattern
- `GraphDescriptor.java` — annotation descriptor record pattern
- GitHub #130 — plan preview issue
- GitHub #159 — edge handling issue
