# Plan Preview & Edge Handling — Design Spec

**Date:** 2026-10-01
**Issues:** #130 (plan preview before execution), #159 (flat graph optimisation + ordering constraints)
**Status:** Design
**Repo:** casehubio/casehub-desiredstate

---

## 1. Overview

Two features that both enhance the `TransitionPlan` pipeline:

**Plan Preview & Approval Gate (#130):** A `PlanApprovalGate` injectable component sits between
`plan()` and `execute()` in `ReconciliationLoop`. It evaluates a `PlanApprovalPolicy` SPI to
decide auto-approve vs require-review, delegates presentation and tracking to a
`PlanApprovalHandler` SPI, and uses skip-and-recheck semantics. While a plan is pending approval,
full-graph reconciliation cycles perform only approval-status polling — drift detection, planning,
execution, fault feedback, and event emission are all skipped. Type-filtered `reconcileTypes()`
continues independently.

**Edge Handling (#159):** `OrderingConstraint(NodeType before, NodeType after)` on
`DesiredStateGraph` declares type-level execution ordering without graph edges.
`TransitionPlanner` resolves constraints as virtual in-degree entries during BFS. Fast-path: when
`dependencies().isEmpty() && orderingConstraints().isEmpty()`, topological sort is skipped
entirely — all nodes fan out as a single layer.

---

## 2. New API Types (api/)

### 2.1 OrderingConstraint

```java
public record OrderingConstraint(NodeType before, NodeType after) {
    public OrderingConstraint {
        Objects.requireNonNull(before, "OrderingConstraint.before must not be null");
        Objects.requireNonNull(after, "OrderingConstraint.after must not be null");
        if (before.equals(after)) {
            throw new IllegalArgumentException(
                "Self-referencing ordering constraint: " + before);
        }
    }
}
```

Type-level ordering: "all nodes of type `before` must be provisioned before any node of type
`after`." Instance-level ordering already exists via `Dependency` edges.

### 2.2 PlanApprovalDecision

```java
public sealed interface PlanApprovalDecision {
    record AutoApprove() implements PlanApprovalDecision {}
    record RequireApproval(String reason) implements PlanApprovalDecision {}
}
```

Returned by `PlanApprovalPolicy.evaluate()`. Stateless — no side effects.

### 2.3 GateDecision

```java
public sealed interface GateDecision {
    record Execute(TransitionPlan plan) implements GateDecision {}
    record AwaitingApproval(String planReference) implements GateDecision {}
    record Rejected(String planReference, String reason) implements GateDecision {}
}
```

Returned by `PlanApprovalGate`. Drives the reconciliation loop's control flow.

### 2.4 PendingPlan

```java
public record PendingPlan(TransitionPlan plan, String planReference, int desiredVersion) {}
```

Internal record stored by `PlanApprovalGate`. The `desiredVersion` enables lazy staleness
detection — if the desired graph's version changes, the pending plan is invalidated.

### 2.5 CloudEvent Data Records

```java
public record PlanAwaitingApprovalData(
    String tenancyId, String planReference,
    int additionCount, int removalCount, int suspensionCount, int resumptionCount,
    String reason) {}

public record PlanApprovedData(
    String tenancyId, String planReference, PlanApproval approval) {}

public record PlanRejectedData(
    String tenancyId, String planReference, String reason) {}

public record PlanInvalidatedData(
    String tenancyId, String planReference, String reason) {}
```

### 2.6 FaultType Extension

Add `PLAN_REJECTED` to `FaultType`. Plan rejection fires through the existing fault pipeline so
operators have visibility via CloudEvents and the reconciliation state store.

### 2.7 DesiredStateEventTypes Extension

Add constants:
- `PLAN_AWAITING_APPROVAL` — `io.casehub.desiredstate.plan.awaiting_approval`
- `PLAN_APPROVED` — `io.casehub.desiredstate.plan.approved`
- `PLAN_REJECTED` — `io.casehub.desiredstate.plan.rejected`
- `PLAN_INVALIDATED` — `io.casehub.desiredstate.plan.invalidated`

---

## 3. New SPIs (api/)

### 3.1 PlanApprovalPolicy

```java
public interface PlanApprovalPolicy {
    PlanApprovalDecision evaluate(TransitionPlan plan, String tenancyId);
}
```

Stateless domain logic. Evaluates whether a transition plan requires human review. Domain
implementors decide the criteria — e.g., auto-approve drift corrections (removals = 0, all
additions are re-provisions of existing types), require review for topology changes (new node
types appearing, nodes being removed).

**Default:** `NoOpPlanApprovalPolicy` (`@DefaultBean`) returns `AutoApprove` for all plans.
Preserves backward compatibility — existing deployments auto-execute without changes.

### 3.2 PlanApprovalHandler

```java
public interface PlanApprovalHandler {
    String submit(TransitionPlan plan, String tenancyId, String reason);
    ApprovalCheckResult check(String planReference, String tenancyId);
    void cancel(String planReference, String tenancyId);
}
```

Stateful infrastructure. Handles plan presentation, storage, and approval tracking. Reuses
`ApprovalCheckResult` from the per-node approval model — the variants (None, Pending, Approved,
Rejected) map directly.

- `submit()` — presents the plan for review, returns a plan reference
- `check()` — polls approval status (called each cycle while pending)
- `cancel()` — cancels a pending plan (called on invalidation)

**Default:** `NoOpPlanApprovalHandler` (`@DefaultBean`) returns `ApprovalCheckResult.None()` for
all checks. Combined with `NoOpPlanApprovalPolicy`, this means plans are never submitted.

**Naming:** Follows the per-node convention — `PendingApprovalHandler` → `PlanApprovalHandler`.

---

## 4. PlanApprovalGate (runtime-core/)

```java
public class PlanApprovalGate {
    private final PlanApprovalPolicy policy;
    private final PlanApprovalHandler handler;
    private final ConcurrentHashMap<String, PendingPlan> pendingPlans = new ConcurrentHashMap<>();

    public Optional<GateDecision> checkPending(String tenancyId, int currentDesiredVersion) { ... }
    public GateDecision evaluateNewPlan(TransitionPlan plan, String tenancyId) { ... }
}
```

### 4.1 checkPending()

Called at the start of `reconcile()`, before any other operation.

1. Look up pending plan for `tenancyId`
2. If none → return `Optional.empty()` (proceed to normal reconciliation)
3. If pending plan exists:
   a. Compare `pendingPlan.desiredVersion()` against `currentDesiredVersion`
   b. If versions differ → invalidate: call `handler.cancel()`, remove from map, emit
      `PLAN_INVALIDATED` CloudEvent, return `Optional.empty()`
   c. If versions match → call `handler.check(planReference, tenancyId)`
      - `Approved(approval)` → remove from map, return `Execute(pendingPlan.plan())`
      - `Pending` → return `AwaitingApproval(planReference)`
      - `Rejected(ref, reason)` → remove from map, return `Rejected(ref, reason)`
      - `None` → stale reference, remove from map, return `Optional.empty()`

### 4.2 evaluateNewPlan()

Called after `plan()` produces a non-empty plan, when no pending plan exists.

1. Call `policy.evaluate(plan, tenancyId)`
2. `AutoApprove` → return `Execute(plan)`
3. `RequireApproval(reason)` → call `handler.submit(plan, tenancyId, reason)`, store
   `PendingPlan(plan, reference, plan.after().version())`, return `AwaitingApproval(reference)`

### 4.3 Staleness Detection

Lazy, version-based. No explicit invalidation hooks needed in `updateDesired()` or
`compareAndSetDesired()`. The gate detects version mismatch on the next `checkPending()` call.
Between desired-state update and the next cycle, a stale pending plan exists in memory — this is
harmless because only `reconcile()` queries the gate.

### 4.4 Thread Safety

`ConcurrentHashMap` for pending plans. `checkPending()` and `evaluateNewPlan()` are called from
`TenantLoop.reconcile()`, which is single-threaded per tenant. Cross-tenant access is safe via
`ConcurrentHashMap`. `handler.check()` and `handler.cancel()` must be thread-safe per the SPI
contract (same requirement as `PendingApprovalHandler`).

---

## 5. ReconciliationLoop Integration

### 5.1 New Field

```java
private final PlanApprovalGate approvalGate;
```

Added to `ReconciliationLoop` constructor and `Builder`. Optional — `Builder` defaults to a gate
wrapping `NoOpPlanApprovalPolicy` and `NoOpPlanApprovalHandler`.

### 5.2 reconcile() Changes

Two integration points:

**Point 1 — Cycle start (before readActual):**
```java
Optional<GateDecision> pending = approvalGate.checkPending(tenancyId, desiredRef.get().version());
if (pending.isPresent()) {
    switch (pending.get()) {
        case GateDecision.Execute(var approvedPlan) -> {
            TransitionResult result = execute(approvedPlan, tenancyId);
            ActualState actual = readActual(approvedPlan.after(), tenancyId);
            faultFeedback(approvedPlan.after(), approvedPlan, result, actual);
            emitCycleEvents(approvedPlan.after(), approvedPlan, result, actual, Set.of(), Set.of());
        }
        case GateDecision.AwaitingApproval(var ref) -> { /* skip cycle */ }
        case GateDecision.Rejected(var ref, var reason) -> {
            // Fire PLAN_REJECTED fault through existing fault pipeline
        }
    }
    return;
}
```

When pending: entire cycle is skipped (no readActual, no detectDrift, no plan, no execute). This
prevents drift-triggered mutations from invalidating the pending plan.

When approved: execute the stored plan. Read actual state post-execution for fault feedback and
event emission. Drift detection is skipped — driftedNodes and exemptNodes are empty sets.

**Point 2 — After plan() produces a non-empty plan:**
```java
GateDecision decision = approvalGate.evaluateNewPlan(plan, tenancyId);
switch (decision) {
    case GateDecision.Execute(var approvedPlan) -> {
        TransitionResult result = execute(approvedPlan, tenancyId);
        // ... existing faultFeedback, emitCycleEvents
    }
    case GateDecision.AwaitingApproval(var ref) -> {
        // Emit PLAN_AWAITING_APPROVAL CloudEvent
        return;
    }
    // GateDecision.Rejected cannot occur from evaluateNewPlan —
    // policy returns AutoApprove or RequireApproval only
}
```

### 5.3 reconcileTypes() — No Change

Type-filtered reconciliation continues independently during plan approval pending. It serves a
different concern (provisioner resync by type) and operates on filtered graphs that do not
interact with the full-graph approval gate.

### 5.4 Plan-Level vs Per-Node Approval

These are independent concerns at different layers:

| Aspect | Plan-Level (PlanApprovalGate) | Per-Node (PendingApprovalHandler) |
|--------|------------------------------|----------------------------------|
| Scope | Entire TransitionPlan | Individual DesiredNode |
| Layer | ReconciliationLoop | NodeStepExecutor |
| Question | "Should this coordinated change proceed?" | "Is this node authorized to change?" |
| SPI | PlanApprovalPolicy + PlanApprovalHandler | PendingApprovalHandler |

Both can fire in the same deployment. An approved plan still has nodes that may need per-node
authorization. Plan rejection fires `FaultType.PLAN_REJECTED` through the existing fault pipeline.

---

## 6. DesiredStateGraph Extension

### 6.1 Interface Change

```java
public interface DesiredStateGraph {
    // ... existing methods ...

    default Set<OrderingConstraint> orderingConstraints() {
        return Set.of();
    }

    DesiredStateGraph withOrderingConstraints(Set<OrderingConstraint> constraints);
}
```

Default returns empty set — backward compatible. All existing DesiredStateGraph implementations
(ImmutableDesiredStateGraph, test mocks) continue to work unchanged.

### 6.2 ImmutableDesiredStateGraph Changes

New field: `Set<OrderingConstraint> orderingConstraints`.

**Constructor:** accepts ordering constraints, defaults to `Set.of()`.

**Mutation methods:** all `with*` and `without*` methods propagate ordering constraints to the new
instance.

**Composition:**
- `overlay()` — merges constraints from both graphs (`Set.union`)
- `connect()` — merges constraints from both graphs (`Set.union`)
- `filterByTypes(types)` — keeps only constraints where both `before` and `after` types are in the
  filter set

**withOrderingConstraints():** returns a new graph with the given constraints replacing existing
ones (set, not merge — GoalCompiler sets constraints during compilation).

### 6.3 DefaultDesiredStateGraphFactory

Add overloaded factory method:
```java
default DesiredStateGraph of(Map<NodeId, DesiredNode> nodes, Set<Dependency> dependencies,
                              Set<OrderingConstraint> orderingConstraints) {
    return of(nodes, dependencies).withOrderingConstraints(orderingConstraints);
}
```

---

## 7. TransitionPlanner Changes

### 7.1 Flat-Graph Fast-Path

In `plan()`, after computing `toAdd`/`toSuspend`/`toResume` sets, before topological sort:

```java
if (desired.dependencies().isEmpty() && desired.orderingConstraints().isEmpty()) {
    // Single-layer plan — skip topological sort entirely
    List<OrderedStep> addSteps = toAdd.stream()
        .map(id -> new OrderedStep(desired.nodes().get(id), StepAction.PROVISION)).toList();
    // ... same for suspend, resume
    return new TransitionPlan(
        List.of(removals),
        suspendSteps.isEmpty() ? List.of() : List.of(suspendSteps),
        resumeSteps.isEmpty() ? List.of() : List.of(resumeSteps),
        addSteps.isEmpty() ? List.of() : List.of(addSteps),
        before, desired);
}
```

Fast-path condition: no dependency edges AND no ordering constraints. Both must be empty because
ordering constraints introduce virtual edges that require topological sort.

### 7.2 Virtual Edges from Ordering Constraints

When ordering constraints are present, `topologicalSort()` gains two additions:

**Input phase (in-degree counting):**
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

**BFS phase (layer processing):**
```java
for (NodeId current : layer) {
    // Existing: real dependents
    for (NodeId dependent : graph.dependentsOf(current)) { ... }
    // New: virtual dependents from ordering constraints
    for (NodeId dependent : virtualReverse.getOrDefault(current, Set.of())) {
        int newDegree = inDegree.merge(dependent, -1, Integer::sum);
        if (newDegree == 0) queue.add(dependent);
    }
}
```

**Complexity:** O(|A|·|B|) per constraint where A and B are the constrained-type node sets within
`toSort`. For the stated use case (1-5 constraints, 10-200 nodes per type), this is negligible.
The quadratic scaling with node count is dominated by provisioning cost.

**Cycle detection:** The existing cycle check (`processed != toSort.size()`) catches cycles
introduced by conflicting constraints (e.g., A before B AND B before A).

### 7.3 Helper

```java
private Set<NodeId> nodesOfType(Set<NodeId> candidates, DesiredStateGraph graph, NodeType type) {
    Set<NodeId> result = new HashSet<>();
    for (NodeId nodeId : candidates) {
        if (graph.nodes().get(nodeId).type().equals(type)) {
            result.add(nodeId);
        }
    }
    return result;
}
```

---

## 8. Surface Integration

### 8.1 YAML

```yaml
desiredState:
  name: iot-fleet
  orderingConstraints:
    - before: power-breaker
      after: equipment
  nodes:
    breaker-1:
      type: power-breaker
    equip-1:
      type: equipment
```

`YamlGraph` gains `List<YamlOrderingConstraint> orderingConstraints` field. `YamlGoalCompilerFactory`
resolves type names via `NodeSpecRegistry` and sets constraints on the compiled graph via
`withOrderingConstraints()`.

### 8.2 Annotations

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface OrderBefore {
    Class<? extends NodeSpec> value();  // "before" type
    Class<? extends NodeSpec> after();
}
```

On `@DesiredState`:
```java
public @interface DesiredState {
    // ... existing ...
    OrderBefore[] orderBefore() default {};
}
```

`DescriptorScanner` extracts `OrderBefore` annotations, resolves `NodeType` from each
`NodeSpec.nodeType()`, and includes them in `GraphDescriptor`. `GoalCompilerFactory` sets
constraints on the compiled graph.

### 8.3 TypeScript DSL

```typescript
defineGraph({
  nodes: { ... },
  orderingConstraints: [
    { before: 'power-breaker', after: 'equipment' }
  ]
})
```

`TsEnvelope` gains `orderingConstraints` field. `TsGoalCompilerFactory` resolves and sets
constraints.

---

## 9. CDI Wiring (runtime/)

### 9.1 New @DefaultBean Fallbacks

```java
@DefaultBean @ApplicationScoped
public class NoOpPlanApprovalPolicy implements PlanApprovalPolicy {
    public PlanApprovalDecision evaluate(TransitionPlan plan, String tenancyId) {
        return new PlanApprovalDecision.AutoApprove();
    }
}

@DefaultBean @ApplicationScoped
public class NoOpPlanApprovalHandler implements PlanApprovalHandler {
    public String submit(TransitionPlan plan, String tenancyId, String reason) {
        return "noop";
    }
    public ApprovalCheckResult check(String planReference, String tenancyId) {
        return new ApprovalCheckResult.None();
    }
    public void cancel(String planReference, String tenancyId) {}
}
```

### 9.2 RuntimeBeans

`RuntimeBeans` gains `@Produces` method for `PlanApprovalGate`:
```java
@Produces @ApplicationScoped
PlanApprovalGate planApprovalGate(PlanApprovalPolicy policy, PlanApprovalHandler handler) {
    return new PlanApprovalGate(policy, handler);
}
```

### 9.3 ReconciliationLoop Injection

`ReconciliationLoop` constructor gains `PlanApprovalGate` parameter. CDI wiring in RuntimeBeans
passes it through. Builder gains `.approvalGate(PlanApprovalGate)` method with NoOp default.

---

## 10. Spring Wiring (runtime-spring/)

`SpringAutoConfiguration` gains:
- `@Bean @ConditionalOnMissingBean PlanApprovalPolicy` returning `NoOpPlanApprovalPolicy`
- `@Bean @ConditionalOnMissingBean PlanApprovalHandler` returning `NoOpPlanApprovalHandler`
- `@Bean PlanApprovalGate` wiring policy and handler
- `ReconciliationLoop` bean updated to receive `PlanApprovalGate`

---

## 11. Testing Strategy

### 11.1 PlanApprovalGate (unit tests in runtime-core/)

- `checkPending` with no pending plan → returns empty
- `checkPending` with pending + version match + Approved → returns Execute
- `checkPending` with pending + version match + Pending → returns AwaitingApproval
- `checkPending` with pending + version match + Rejected → returns Rejected
- `checkPending` with pending + version mismatch → invalidates, returns empty
- `evaluateNewPlan` with AutoApprove policy → returns Execute
- `evaluateNewPlan` with RequireApproval policy → stores, returns AwaitingApproval
- Thread safety: concurrent `checkPending` for different tenants

### 11.2 TransitionPlanner (unit tests in runtime-core/)

- Flat-graph fast-path: no edges, no constraints → single-layer plan
- Flat-graph with constraints: not fast-path → multi-layer plan
- Virtual edges: constraint A→B, nodes of both types → correct layer ordering
- Virtual edges: multiple constraints → correct chained ordering
- Conflicting constraints (A→B, B→A) → cycle detection exception
- Mixed: real edges + ordering constraints → both respected
- Empty toSort set with constraints → no error

### 11.3 OrderingConstraint on DesiredStateGraph (unit tests in runtime-core/)

- `withOrderingConstraints()` returns new graph with constraints
- `overlay()` merges constraints from both graphs
- `connect()` merges constraints from both graphs
- `filterByTypes()` filters constraints to matching types
- `withNode()`/`withoutNode()` preserves constraints
- Default `orderingConstraints()` returns empty set

### 11.4 ReconciliationLoop Integration (integration tests)

- Full cycle with approval gate: plan generated → RequireApproval → loop returns → next cycle
  checks → Approved → execute
- Full cycle with auto-approve: plan generated → AutoApprove → immediate execute
- Desired state change during pending → invalidation → re-plan
- Ordering constraints with parallel executor: layers respected
- Flat-graph fast-path: all nodes in single parallel layer

### 11.5 Example Updates

Pipeline example (or pipeline-annotated, pipeline-yaml): add ordering constraints between tiers
(Bronze before Silver before Gold) to demonstrate the feature. These already have natural ordering
via `@DependsOn` — the ordering constraint is an alternative declaration.

---

## 12. Out of Scope

- **Plan diff rendering:** Runtime emits structured CloudEvent data. Rendering (HTML, CLI,
  Slack) is a consumer concern — not this spec.
- **JPA-backed PlanApprovalStore:** Pending plans are in-memory. If durability is needed, a store
  SPI can be added later. Pending plans are cheap to regenerate (re-plan on restart).
- **Plan approval UI:** The `PlanApprovalHandler` SPI enables integration with any UI. Building
  the UI is a separate concern.
- **Instance-level ordering constraints:** Already exists as `Dependency` edges. Type-level fills
  the gap.

---

## References

- `ReconciliationLoop.java:624-671` — reconcile() flow
- `ReconciliationLoop.java:735-799` — detectDrift with faultPolicyEngine mutation
- `TransitionPlanner.java:134-186` — topologicalSort BFS algorithm
- `PendingApprovalHandler.java` — per-node approval pattern
- `ApprovalCheckResult.java` — reused sealed type
- `DesiredStateGraph.java` — graph SPI interface
- `ImmutableDesiredStateGraph.java` — default graph implementation
- `LifecycleManager.java:24-52` — CompilationResult → graph extraction (motivated D7 revision)
- `Dependency.java` — instance-level ordering precedent
- `DriftPolicyEngine` — injectable component pattern precedent
- `ReconciliationEventEmitter` — CloudEvent emission pattern
- `DesiredStateEventTypes` — event type constant pattern
- `FaultType` — fault type enum (extended with PLAN_REJECTED)
- #130 — plan preview issue
- #159 — edge handling issue
- #153 — IoT consumer requirements (parent of #159)
- Decision review: `reviews/casehub-desiredstate/issue-130-decision-20261001-153802/`
