# Suspend/Resume Lifecycle Verbs Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #152 — feat: suspend/resume lifecycle verbs on NodeProvisioner SPI
**Issue group:** #152

**Goal:** Extend NodeProvisioner with opt-in suspend/resume lifecycle verbs, enabling
stateful resources to idle without losing persisted state.

**Architecture:** Additive SPI extension — new default methods on NodeProvisioner, new
TargetStatus enum on DesiredNode, SUSPENDED value on NodeStatus. TransitionPlanner gains a
decision matrix comparing actual status vs target status. SimpleTransitionExecutor dispatches
to new router methods. HumanGating migrates from enum to EnumSet-based record.

**Tech Stack:** Java 21+, Quarkus (CDI), JUnit 5, OpenTelemetry

## Global Constraints

- All new api/ types must be in `io.casehub.desiredstate.api` package
- All runtime-core/ types must be in `io.casehub.desiredstate.runtime` package
- Backward compatibility: existing provisioners must compile and run unchanged
- Default methods on NodeProvisioner return Failed/false for stateless provisioners
- `mvn --batch-mode install` must pass after each batch
- Use `ide_insert_member` for new methods, `ide_replace_member` for modifying existing methods
- Use `ide_refactor_rename` for renames, never bash mv/sed on source files

---

## Batch 1: API Foundation — new types and enum extensions

After this batch: all new api/ types exist, existing types are extended, the project compiles,
and unit tests verify the new types. No runtime behavior changes yet.

### Task 1: TargetStatus enum + StepAction extension + FaultType extension

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/TargetStatus.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/StepAction.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/FaultType.java`
- Test: `api/src/test/java/io/casehub/desiredstate/api/TargetStatusTest.java`

**Interfaces:**
- Produces: `TargetStatus.ACTIVE`, `TargetStatus.SUSPENDED` enum values; `StepAction.SUSPEND`, `StepAction.RESUME` values; `FaultType.SUSPEND_FAILED`, `FaultType.RESUME_FAILED` values

- [ ] **Step 1: Write test for TargetStatus enum**

```java
package io.casehub.desiredstate.api;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class TargetStatusTest {
    @Test
    void values() {
        assertEquals(2, TargetStatus.values().length);
        assertNotNull(TargetStatus.ACTIVE);
        assertNotNull(TargetStatus.SUSPENDED);
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=TargetStatusTest`
Expected: COMPILATION_ERROR — TargetStatus does not exist

- [ ] **Step 3: Create TargetStatus enum**

```java
package io.casehub.desiredstate.api;

public enum TargetStatus {
    ACTIVE,
    SUSPENDED
}
```

- [ ] **Step 4: Add SUSPEND and RESUME to StepAction**

Modify `api/src/main/java/io/casehub/desiredstate/api/StepAction.java`:

```java
package io.casehub.desiredstate.api;

public enum StepAction { PROVISION, DEPROVISION, SUSPEND, RESUME }
```

- [ ] **Step 5: Add SUSPEND_FAILED and RESUME_FAILED to FaultType**

Modify `api/src/main/java/io/casehub/desiredstate/api/FaultType.java`:

```java
package io.casehub.desiredstate.api;

public enum FaultType {
    NODE_DESTROYED, NODE_DEGRADED, PROVISION_FAILED, DEPROVISION_FAILED,
    SUSPEND_FAILED, RESUME_FAILED,
    HUMAN_NODE_TIMEOUT, DEPENDENCY_UNAVAILABLE, APPROVAL_REJECTED
}
```

- [ ] **Step 6: Fix all exhaustive switch compilation errors on StepAction**

Adding SUSPEND/RESUME to StepAction will break exhaustive switches across the codebase.
Use `ide_find_references` on `StepAction` to find all switch statements. Key locations:

- `HumanGating.java:10` — `requiresHuman(StepAction)` switch
- `DesiredStateDispatch` in engine-adapter/ — `dispatch()` switch
- Any other exhaustive switches

For `HumanGating`, this will be fully replaced in Task 2. For now, add temporary cases:
```java
case SUSPEND, RESUME -> false;
```

For `DesiredStateDispatch`, add placeholder cases:
```java
case SUSPEND -> throw new UnsupportedOperationException("suspend dispatch not yet supported");
case RESUME -> throw new UnsupportedOperationException("resume dispatch not yet supported");
```

- [ ] **Step 7: Run test — verify it passes**

Run: `mvn --batch-mode test -pl api -Dtest=TargetStatusTest`
Expected: PASS

- [ ] **Step 8: Compile full project to verify no breakage**

Run: `mvn --batch-mode compile`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/TargetStatus.java
git add api/src/main/java/io/casehub/desiredstate/api/StepAction.java
git add api/src/main/java/io/casehub/desiredstate/api/FaultType.java
git add api/src/test/java/io/casehub/desiredstate/api/TargetStatusTest.java
git add -A  # catch any switch-fix files
git commit -m "feat(#152): add TargetStatus enum, extend StepAction and FaultType with suspend/resume"
```

---

### Task 2: HumanGating — migrate from enum to EnumSet-based record

**Files:**
- Modify: `api/src/main/java/io/casehub/desiredstate/api/HumanGating.java` (replace entirely)
- Modify: all callsites that reference `HumanGating.NONE`, `HumanGating.PROVISION_ONLY`, etc.
- Test: `api/src/test/java/io/casehub/desiredstate/api/HumanGatingTest.java`

**Interfaces:**
- Consumes: `StepAction.SUSPEND`, `StepAction.RESUME` from Task 1
- Produces: `HumanGating.NONE`, `HumanGating.all()`, `HumanGating.of(StepAction...)`, `requiresHuman(StepAction)`, `any()`, `merge(HumanGating)`

- [ ] **Step 1: Write test for new HumanGating record**

```java
package io.casehub.desiredstate.api;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class HumanGatingTest {
    @Test
    void noneGatesNothing() {
        assertFalse(HumanGating.NONE.requiresHuman(StepAction.PROVISION));
        assertFalse(HumanGating.NONE.requiresHuman(StepAction.SUSPEND));
        assertFalse(HumanGating.NONE.any());
    }

    @Test
    void ofGatesSpecificActions() {
        HumanGating gating = HumanGating.of(StepAction.PROVISION, StepAction.SUSPEND);
        assertTrue(gating.requiresHuman(StepAction.PROVISION));
        assertTrue(gating.requiresHuman(StepAction.SUSPEND));
        assertFalse(gating.requiresHuman(StepAction.DEPROVISION));
        assertFalse(gating.requiresHuman(StepAction.RESUME));
        assertTrue(gating.any());
    }

    @Test
    void allGatesEverything() {
        HumanGating gating = HumanGating.all();
        assertTrue(gating.requiresHuman(StepAction.PROVISION));
        assertTrue(gating.requiresHuman(StepAction.DEPROVISION));
        assertTrue(gating.requiresHuman(StepAction.SUSPEND));
        assertTrue(gating.requiresHuman(StepAction.RESUME));
    }

    @Test
    void mergeCombinesActions() {
        HumanGating a = HumanGating.of(StepAction.PROVISION);
        HumanGating b = HumanGating.of(StepAction.SUSPEND);
        HumanGating merged = a.merge(b);
        assertTrue(merged.requiresHuman(StepAction.PROVISION));
        assertTrue(merged.requiresHuman(StepAction.SUSPEND));
        assertFalse(merged.requiresHuman(StepAction.DEPROVISION));
        assertFalse(merged.requiresHuman(StepAction.RESUME));
    }
}
```

- [ ] **Step 2: Replace HumanGating enum with record**

Replace the entire contents of `api/src/main/java/io/casehub/desiredstate/api/HumanGating.java`:

```java
package io.casehub.desiredstate.api;

import java.util.EnumSet;
import java.util.Objects;
import java.util.Set;

public record HumanGating(Set<StepAction> gatedActions) {

    public static final HumanGating NONE = new HumanGating(Set.of());

    public HumanGating {
        Objects.requireNonNull(gatedActions, "gatedActions must not be null");
        gatedActions = gatedActions.isEmpty()
                ? Set.of()
                : Set.copyOf(EnumSet.copyOf(gatedActions));
    }

    public static HumanGating all() {
        return new HumanGating(EnumSet.allOf(StepAction.class));
    }

    public static HumanGating of(StepAction... actions) {
        if (actions.length == 0) return NONE;
        return new HumanGating(EnumSet.copyOf(Set.of(actions)));
    }

    public boolean requiresHuman(StepAction action) {
        return gatedActions.contains(action);
    }

    public boolean any() {
        return !gatedActions.isEmpty();
    }

    public HumanGating merge(HumanGating other) {
        if (gatedActions.isEmpty()) return other;
        if (other.gatedActions.isEmpty()) return this;
        EnumSet<StepAction> merged = EnumSet.noneOf(StepAction.class);
        merged.addAll(gatedActions);
        merged.addAll(other.gatedActions);
        return new HumanGating(merged);
    }
}
```

- [ ] **Step 3: Fix all callsites referencing old HumanGating enum values**

Use `ide_find_references` on `HumanGating` to find all callsites. Key migrations:
- `HumanGating.NONE` → stays the same (static field)
- `HumanGating.PROVISION_ONLY` → `HumanGating.of(StepAction.PROVISION)`
- `HumanGating.DEPROVISION_ONLY` → `HumanGating.of(StepAction.DEPROVISION)`
- `HumanGating.ALL` → `HumanGating.all()`
- `humanGating.name()` → needs adjustment (record has no `.name()` like an enum)
- Jackson/serialization: `HumanGating.valueOf(str)` → custom deserialization

Search for `.name()` calls on HumanGating (used in GraphSerializer, OTel spans, etc.)
and replace with appropriate serialization.

- [ ] **Step 4: Run tests**

Run: `mvn --batch-mode test -pl api`
Expected: PASS

- [ ] **Step 5: Compile full project**

Run: `mvn --batch-mode compile`
Expected: BUILD SUCCESS (may need to fix callsites in other modules)

- [ ] **Step 6: Fix any remaining compilation errors across all modules**

Run: `mvn --batch-mode test`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(#152): migrate HumanGating from enum to EnumSet-based record"
```

---

### Task 3: SuspendResult, ResumeResult, SuspendContext, ResumeContext, NodeStatus.SUSPENDED

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/SuspendResult.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/ResumeResult.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/SuspendContext.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/ResumeContext.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/NodeStatus.java`
- Test: `api/src/test/java/io/casehub/desiredstate/api/SuspendResumeTypesTest.java`

**Interfaces:**
- Consumes: `NodeId`, `DesiredStateGraph`, `PlanApproval` from existing api/
- Produces: `SuspendResult` (Success/Failed/PendingApproval), `ResumeResult` (same), `SuspendContext`, `ResumeContext`, `NodeStatus.SUSPENDED`

- [ ] **Step 1: Write test for new types**

```java
package io.casehub.desiredstate.api;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class SuspendResumeTypesTest {
    @Test
    void suspendResultVariants() {
        SuspendResult success = new SuspendResult.Success();
        SuspendResult failed = new SuspendResult.Failed("reason");
        SuspendResult pending = new SuspendResult.PendingApproval(NodeId.of("n1"), "plan-ref");
        assertInstanceOf(SuspendResult.Success.class, success);
        assertInstanceOf(SuspendResult.Failed.class, failed);
        assertEquals("reason", ((SuspendResult.Failed) failed).reason());
    }

    @Test
    void resumeResultVariants() {
        ResumeResult success = new ResumeResult.Success();
        ResumeResult failed = new ResumeResult.Failed("reason");
        ResumeResult pending = new ResumeResult.PendingApproval(NodeId.of("n1"), "plan-ref");
        assertInstanceOf(ResumeResult.Success.class, success);
    }

    @Test
    void suspendContextWithApproval() {
        SuspendContext ctx = new SuspendContext("tenant-1", null);
        assertFalse(ctx.hasApproval());
        PlanApproval approval = new PlanApproval("ref", "admin", java.time.Instant.now());
        SuspendContext withApproval = ctx.withApproval(approval);
        assertTrue(withApproval.hasApproval());
        assertEquals(approval, withApproval.approval());
    }

    @Test
    void resumeContextWithApproval() {
        ResumeContext ctx = new ResumeContext("tenant-1", null);
        assertFalse(ctx.hasApproval());
    }

    @Test
    void nodeStatusSuspended() {
        assertNotNull(NodeStatus.SUSPENDED);
        assertEquals(5, NodeStatus.values().length);
    }
}
```

- [ ] **Step 2: Run test — verify compilation failure**

Run: `mvn --batch-mode test -pl api -Dtest=SuspendResumeTypesTest`
Expected: COMPILATION_ERROR

- [ ] **Step 3: Create SuspendResult**

```java
package io.casehub.desiredstate.api;

public sealed interface SuspendResult {
    record Success() implements SuspendResult {}
    record Failed(String reason) implements SuspendResult {}
    record PendingApproval(NodeId nodeId, String planReference) implements SuspendResult {}
}
```

- [ ] **Step 4: Create ResumeResult**

```java
package io.casehub.desiredstate.api;

public sealed interface ResumeResult {
    record Success() implements ResumeResult {}
    record Failed(String reason) implements ResumeResult {}
    record PendingApproval(NodeId nodeId, String planReference) implements ResumeResult {}
}
```

- [ ] **Step 5: Create SuspendContext**

Model on existing `ProvisionContext` — read it first with `ide_file_structure` to match the
exact pattern (constructor, withApproval, approval, hasApproval). Then create:

```java
package io.casehub.desiredstate.api;

public record SuspendContext(String tenancyId, DesiredStateGraph graph, PlanApproval approval) {
    public SuspendContext(String tenancyId, DesiredStateGraph graph) {
        this(tenancyId, graph, null);
    }
    public SuspendContext withApproval(PlanApproval approval) {
        return new SuspendContext(tenancyId, graph, approval);
    }
    public boolean hasApproval() { return approval != null; }
}
```

- [ ] **Step 6: Create ResumeContext**

Same pattern as SuspendContext:

```java
package io.casehub.desiredstate.api;

public record ResumeContext(String tenancyId, DesiredStateGraph graph, PlanApproval approval) {
    public ResumeContext(String tenancyId, DesiredStateGraph graph) {
        this(tenancyId, graph, null);
    }
    public ResumeContext withApproval(PlanApproval approval) {
        return new ResumeContext(tenancyId, graph, approval);
    }
    public boolean hasApproval() { return approval != null; }
}
```

- [ ] **Step 7: Add SUSPENDED to NodeStatus**

Modify `api/src/main/java/io/casehub/desiredstate/api/NodeStatus.java`:

```java
package io.casehub.desiredstate.api;

public enum NodeStatus {
    PRESENT,
    ABSENT,
    DRIFTED,
    UNKNOWN,
    SUSPENDED
}
```

- [ ] **Step 8: Fix all exhaustive switch compilation errors on NodeStatus**

Use `ide_find_references` on `NodeStatus` to find all switch statements. Add `SUSPENDED`
cases. Key locations:
- `TransitionPlanner.java:39-42` — removal switch → treat SUSPENDED like PRESENT (remove it)
- `TransitionPlanner.java:58-61` — addition switch → treat SUSPENDED like PRESENT (don't re-provision)
- Any other exhaustive switches

For now, these are temporary fixes — the TransitionPlanner will be fully reworked in Batch 2.
Use the simplest correct mapping:
- In removal logic: `case SUSPENDED -> true` (deprovision suspended nodes not in desired graph)
- In addition logic: `case SUSPENDED -> false` (don't re-provision suspended nodes)

- [ ] **Step 9: Run tests**

Run: `mvn --batch-mode test -pl api -Dtest=SuspendResumeTypesTest`
Expected: PASS

- [ ] **Step 10: Compile full project**

Run: `mvn --batch-mode compile`
Expected: BUILD SUCCESS

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat(#152): add SuspendResult, ResumeResult, contexts, NodeStatus.SUSPENDED"
```

---

### Task 4: DesiredNode targetStatus + NodeProvisioner suspend/resume + Router + HumanNodeHandler

**Files:**
- Modify: `api/src/main/java/io/casehub/desiredstate/api/DesiredNode.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/NodeProvisioner.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/NodeProvisionerRouter.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/HumanNodeHandler.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/CompletionCondition.java`
- Test: `api/src/test/java/io/casehub/desiredstate/api/DesiredNodeTargetStatusTest.java`

**Interfaces:**
- Consumes: `TargetStatus`, `SuspendResult`, `ResumeResult`, `SuspendContext`, `ResumeContext` from Tasks 1/3
- Produces: `DesiredNode(id, spec, humanGating, hooks, targetStatus)`, `NodeProvisioner.suspend()`, `NodeProvisioner.resume()`, `NodeProvisioner.supportsStatefulLifecycle()`, `NodeProvisionerRouter.suspend()`, `NodeProvisionerRouter.resume()`, `NodeProvisionerRouter.supportsStatefulLifecycle(NodeType)`, `HumanNodeHandler.onSuspend()`, `HumanNodeHandler.onResume()`, `CompletionCondition.allSatisfied()`

- [ ] **Step 1: Write test for DesiredNode with targetStatus**

```java
package io.casehub.desiredstate.api;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class DesiredNodeTargetStatusTest {
    private static final NodeSpec SPEC = new NodeSpec() {
        @Override public NodeType nodeType() { return NodeType.of("test"); }
    };

    @Test
    void threeArgConstructorDefaultsToActive() {
        DesiredNode node = new DesiredNode(NodeId.of("n1"), SPEC, HumanGating.NONE);
        assertEquals(TargetStatus.ACTIVE, node.targetStatus());
    }

    @Test
    void fourArgConstructorDefaultsToActive() {
        DesiredNode node = new DesiredNode(NodeId.of("n1"), SPEC, HumanGating.NONE, null);
        assertEquals(TargetStatus.ACTIVE, node.targetStatus());
    }

    @Test
    void fiveArgConstructorSetsSuspended() {
        DesiredNode node = new DesiredNode(NodeId.of("n1"), SPEC, HumanGating.NONE, null, TargetStatus.SUSPENDED);
        assertEquals(TargetStatus.SUSPENDED, node.targetStatus());
    }

    @Test
    void allSatisfiedWithMixedTargets() {
        // Tested more thoroughly once graph construction is available
        assertNotNull(CompletionCondition.allSatisfied());
    }
}
```

- [ ] **Step 2: Run test — verify compilation failure**

Run: `mvn --batch-mode test -pl api -Dtest=DesiredNodeTargetStatusTest`
Expected: COMPILATION_ERROR

- [ ] **Step 3: Add targetStatus to DesiredNode**

Replace `api/src/main/java/io/casehub/desiredstate/api/DesiredNode.java`:

```java
package io.casehub.desiredstate.api;

import java.util.Objects;

public record DesiredNode(NodeId id, NodeSpec spec, HumanGating humanGating,
                          HookDescriptor hooks, TargetStatus targetStatus) {

    public DesiredNode(NodeId id, NodeSpec spec, HumanGating humanGating) {
        this(id, spec, humanGating, null, TargetStatus.ACTIVE);
    }

    public DesiredNode(NodeId id, NodeSpec spec, HumanGating humanGating, HookDescriptor hooks) {
        this(id, spec, humanGating, hooks, TargetStatus.ACTIVE);
    }

    public DesiredNode {
        Objects.requireNonNull(id, "DesiredNode id must not be null");
        Objects.requireNonNull(spec, "DesiredNode spec must not be null");
        Objects.requireNonNull(humanGating, "DesiredNode humanGating must not be null");
        Objects.requireNonNull(targetStatus, "DesiredNode targetStatus must not be null");
    }

    public NodeType type() {
        return spec.nodeType();
    }

    public boolean requiresHuman(StepAction action) {
        return humanGating.requiresHuman(action) || spec.humanGating().requiresHuman(action);
    }

    public boolean requiresHuman() {
        return humanGating.any() || spec.humanGating().any();
    }
}
```

- [ ] **Step 4: Add suspend/resume defaults to NodeProvisioner**

Add to `api/src/main/java/io/casehub/desiredstate/api/NodeProvisioner.java` after `deprovision`:

```java
default SuspendResult suspend(DesiredNode node, SuspendContext context) {
    return new SuspendResult.Failed("suspend not supported");
}

default ResumeResult resume(DesiredNode node, ResumeContext context) {
    return new ResumeResult.Failed("resume not supported");
}

default boolean supportsStatefulLifecycle() { return false; }
```

- [ ] **Step 5: Add suspend/resume to NodeProvisionerRouter**

Add to `api/src/main/java/io/casehub/desiredstate/api/NodeProvisionerRouter.java`:

```java
SuspendResult suspend(DesiredNode node, SuspendContext context);
ResumeResult resume(DesiredNode node, ResumeContext context);
boolean supportsStatefulLifecycle(NodeType type);
```

- [ ] **Step 6: Add onSuspend/onResume defaults to HumanNodeHandler**

Add to `api/src/main/java/io/casehub/desiredstate/api/HumanNodeHandler.java`:

```java
default StepOutcome onSuspend(DesiredNode node, SuspendContext context) {
    return new StepOutcome.Skipped("human suspend not handled");
}

default StepOutcome onResume(DesiredNode node, ResumeContext context) {
    return new StepOutcome.Skipped("human resume not handled");
}
```

- [ ] **Step 7: Add allSatisfied() to CompletionCondition**

Add to `api/src/main/java/io/casehub/desiredstate/api/CompletionCondition.java`:

```java
static CompletionCondition allSatisfied() {
    return (desired, actual) -> desired.nodes().entrySet().stream().allMatch(e -> {
        NodeStatus status = actual.statuses().getOrDefault(e.getKey(), NodeStatus.UNKNOWN);
        return switch (e.getValue().targetStatus()) {
            case ACTIVE -> status == NodeStatus.PRESENT;
            case SUSPENDED -> status == NodeStatus.SUSPENDED;
        };
    });
}
```

- [ ] **Step 8: Fix compilation errors from DesiredNode constructor changes**

Use `ide_find_references` on `DesiredNode` to find all callsites. The 3-arg and 4-arg
constructors are preserved, so most callsites should compile unchanged. Any code that
directly constructs via the canonical 4-arg record constructor (positional) may need
the 5th argument — add `TargetStatus.ACTIVE` as default.

- [ ] **Step 9: Run tests**

Run: `mvn --batch-mode test -pl api`
Expected: PASS

- [ ] **Step 10: Compile full project**

Run: `mvn --batch-mode compile`
Expected: BUILD SUCCESS

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat(#152): add targetStatus to DesiredNode, suspend/resume on NodeProvisioner SPI"
```

---

### Task 5: TransitionPlan extension + CloudEvent types

**Files:**
- Modify: `api/src/main/java/io/casehub/desiredstate/api/TransitionPlan.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/DesiredStateEventTypes.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/ReconciliationCompletedData.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/NodeSuspendedData.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/NodeResumedData.java`
- Test: `api/src/test/java/io/casehub/desiredstate/api/TransitionPlanSuspendResumeTest.java`

**Interfaces:**
- Consumes: `OrderedStep`, `DesiredStateGraph` from existing api/
- Produces: `TransitionPlan(removals, suspensions, resumptions, additions, before, after)`, `NodeSuspendedData`, `NodeResumedData`, `ReconciliationCompletedData` with suspension/resumption counts

- [ ] **Step 1: Write test for extended TransitionPlan**

```java
package io.casehub.desiredstate.api;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class TransitionPlanSuspendResumeTest {
    @Test
    void fourArgConstructorDefaultsToEmptyLists() {
        DesiredStateGraph graph = DesiredStateGraph.empty();
        TransitionPlan plan = new TransitionPlan(List.of(), List.of(), graph, graph);
        assertTrue(plan.suspensions().isEmpty());
        assertTrue(plan.resumptions().isEmpty());
        assertTrue(plan.isEmpty());
    }

    @Test
    void sixArgConstructorAcceptsSuspensionsAndResumptions() {
        DesiredStateGraph graph = DesiredStateGraph.empty();
        TransitionPlan plan = new TransitionPlan(
            List.of(), List.of(), List.of(), List.of(), graph, graph);
        assertTrue(plan.isEmpty());
    }

    @Test
    void isEmptyConsidersSuspensionsAndResumptions() {
        DesiredStateGraph graph = DesiredStateGraph.empty();
        NodeSpec spec = new NodeSpec() {
            @Override public NodeType nodeType() { return NodeType.of("test"); }
        };
        DesiredNode node = new DesiredNode(NodeId.of("n1"), spec, HumanGating.NONE);
        OrderedStep step = new OrderedStep(node, StepAction.SUSPEND);
        TransitionPlan plan = new TransitionPlan(
            List.of(), List.of(step), List.of(), List.of(), graph, graph);
        assertFalse(plan.isEmpty());
    }
}
```

- [ ] **Step 2: Run test — verify compilation failure**

Run: `mvn --batch-mode test -pl api -Dtest=TransitionPlanSuspendResumeTest`
Expected: COMPILATION_ERROR

- [ ] **Step 3: Extend TransitionPlan record**

Replace `api/src/main/java/io/casehub/desiredstate/api/TransitionPlan.java`:

```java
package io.casehub.desiredstate.api;

import java.util.List;
import java.util.Objects;

public record TransitionPlan(
    List<OrderedStep> removals,
    List<OrderedStep> suspensions,
    List<OrderedStep> resumptions,
    List<OrderedStep> additions,
    DesiredStateGraph before, DesiredStateGraph after
) {
    public TransitionPlan(List<OrderedStep> removals, List<OrderedStep> additions,
                          DesiredStateGraph before, DesiredStateGraph after) {
        this(removals, List.of(), List.of(), additions, before, after);
    }

    public TransitionPlan {
        removals = List.copyOf(removals);
        suspensions = List.copyOf(suspensions);
        resumptions = List.copyOf(resumptions);
        additions = List.copyOf(additions);
        Objects.requireNonNull(before, "TransitionPlan.before must not be null");
        Objects.requireNonNull(after, "TransitionPlan.after must not be null");
    }

    public boolean isEmpty() {
        return removals.isEmpty() && suspensions.isEmpty()
            && resumptions.isEmpty() && additions.isEmpty();
    }
}
```

- [ ] **Step 4: Add CloudEvent types**

Add to `DesiredStateEventTypes.java`:

```java
public static final String NODE_SUSPENDED =
    "io.casehub.desiredstate.node.suspended";
public static final String NODE_RESUMED =
    "io.casehub.desiredstate.node.resumed";
```

- [ ] **Step 5: Create NodeSuspendedData and NodeResumedData**

Model on existing `NodeRecoveredData` — read it first. Create both records with the
same shape: `tenancyId`, `nodeId`, `nodeType`, `graphVersion`, `parentNodeId`.

- [ ] **Step 6: Extend ReconciliationCompletedData**

Add `suspensionsCount` and `resumptionsCount` fields:

```java
public record ReconciliationCompletedData(
        String tenancyId, long graphVersion,
        int nodeCount, int additionsCount, int removalsCount,
        int suspensionsCount, int resumptionsCount,
        int faultCount, Instant timestamp) {
    // Backward-compatible constructor
    public ReconciliationCompletedData(String tenancyId, long graphVersion,
            int nodeCount, int additionsCount, int removalsCount,
            int faultCount, Instant timestamp) {
        this(tenancyId, graphVersion, nodeCount, additionsCount, removalsCount,
             0, 0, faultCount, timestamp);
    }
    // ... validation in compact constructor
}
```

- [ ] **Step 7: Fix compilation errors from TransitionPlan constructor changes**

Use `ide_find_references` on `TransitionPlan` constructor. The 4-arg constructor is
preserved, so most callsites should compile. Fix any that use the canonical 6-arg form.

- [ ] **Step 8: Fix compilation errors from ReconciliationCompletedData changes**

Use `ide_find_references` on `ReconciliationCompletedData` constructor. The old 7-arg
constructor is preserved as a backward-compatible overload.

- [ ] **Step 9: Run tests**

Run: `mvn --batch-mode test -pl api`
Expected: PASS

- [ ] **Step 10: Compile full project**

Run: `mvn --batch-mode compile`
Expected: BUILD SUCCESS

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat(#152): extend TransitionPlan with suspensions/resumptions, add CloudEvent types"
```

---

## Batch 2: Runtime — planner, executor, router, reconciliation loop

After this batch: the runtime correctly plans and executes suspend/resume transitions,
including stateless fallback. The reconciliation loop emits correct fault types and
CloudEvents for suspend/resume operations.

### Task 6: DefaultNodeProvisionerRouter — suspend/resume dispatch

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DefaultNodeProvisionerRouter.java`
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/CdiNodeProvisionerRouter.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/DefaultNodeProvisionerRouterTest.java`

**Interfaces:**
- Consumes: `NodeProvisioner.suspend()`, `NodeProvisioner.resume()`, `NodeProvisioner.supportsStatefulLifecycle()` from Task 4
- Produces: Router dispatching suspend/resume to correct provisioner by NodeType

- [ ] **Step 1: Write test for router suspend/resume dispatch**

Add to existing `DefaultNodeProvisionerRouterTest`:

```java
@Test
void suspendDelegatesToCorrectProvisioner() {
    // Create a stateful provisioner that returns Success on suspend
    // Register it, then call router.suspend() with a node of the correct type
    // Assert Success is returned
}

@Test
void resumeDelegatesToCorrectProvisioner() {
    // Similar to suspend test
}

@Test
void supportsStatefulLifecycleReturnsTrueForStatefulProvisioner() {
    // Register a provisioner with supportsStatefulLifecycle()=true
    // Assert router.supportsStatefulLifecycle(type) returns true
}

@Test
void supportsStatefulLifecycleReturnsFalseForStatelessProvisioner() {
    // Standard provisioner (default methods)
    // Assert router.supportsStatefulLifecycle(type) returns false
}
```

Read the existing test file first to match the test fixture pattern exactly.

- [ ] **Step 2: Run test — verify failure**

Run: `mvn --batch-mode test -pl runtime -Dtest=DefaultNodeProvisionerRouterTest`

- [ ] **Step 3: Implement suspend/resume/supportsStatefulLifecycle in DefaultNodeProvisionerRouter**

Read the existing `provision()` and `deprovision()` methods in `DefaultNodeProvisionerRouter`
to match the pattern. Add `suspend()`, `resume()`, and `supportsStatefulLifecycle()` methods
that look up the provisioner by NodeType and delegate.

- [ ] **Step 4: Update CdiNodeProvisionerRouter if needed**

Check if `CdiNodeProvisionerRouter` extends `DefaultNodeProvisionerRouter` or overrides
methods. If it extends, the new methods are inherited. If it overrides, add the new methods.

- [ ] **Step 5: Run tests — verify pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=DefaultNodeProvisionerRouterTest`

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#152): add suspend/resume dispatch to DefaultNodeProvisionerRouter"
```

---

### Task 7: TransitionPlanner — decision matrix with stateless fallback

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/TransitionPlanner.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/TransitionPlannerTest.java` (existing + new tests)

**Interfaces:**
- Consumes: `TargetStatus`, `NodeStatus.SUSPENDED`, `StepAction.SUSPEND/RESUME`, `TransitionPlan` (6-arg) from Batch 1
- Produces: `plan(desired, actual, previousDesired, supportsStatefulLifecycle)` method with Predicate<NodeType> parameter

- [ ] **Step 1: Write tests for each cell in the decision matrix**

Read existing `TransitionPlannerTest` first. Add tests:

```java
// Target ACTIVE tests (existing behavior, verify unchanged)
@Test void presentActiveNoOp() { ... }
@Test void absentActiveProvision() { ... }
@Test void suspendedActiveResume() { ... }
@Test void driftedActiveProvision() { ... }
@Test void unknownActiveProvision() { ... }

// Target SUSPENDED tests (new behavior)
@Test void presentSuspendedSuspend() { ... }
@Test void absentSuspendedResume() { ... }
@Test void suspendedSuspendedNoOp() { ... }
@Test void driftedSuspendedSuspend() { ... }
@Test void unknownSuspendedResume() { ... }

// Not in graph tests (existing behavior, verify unchanged for SUSPENDED actual)
@Test void suspendedNotInGraphDeprovision() { ... }

// Stateless fallback tests
@Test void suspendFallsBackToDeprovisionWhenNotSupported() { ... }
@Test void resumeFallsBackToProvisionWhenNotSupported() { ... }

// Ordering tests
@Test void suspensionsTopologicallySortedLeavesFirst() { ... }
@Test void resumptionsTopologicallySortedRootsFirst() { ... }
```

Each test should construct a minimal graph, set actual state, set target status on nodes,
call `plan()`, and assert the correct step lists.

- [ ] **Step 2: Run tests — verify failure**

Run: `mvn --batch-mode test -pl runtime -Dtest=TransitionPlannerTest`

- [ ] **Step 3: Implement the new plan() overload**

Add a new `plan(desired, actual, previousDesired, Predicate<NodeType> supportsStateful)` method.
The existing 2-arg and 3-arg methods delegate with `type -> false` (stateless-only fallback).

The implementation follows the decision matrix:

```java
public TransitionPlan plan(DesiredStateGraph desired, ActualState actual,
                           DesiredStateGraph previousDesired,
                           Predicate<NodeType> supportsStateful) {
    List<OrderedStep> removals = new ArrayList<>();
    List<OrderedStep> suspensions = new ArrayList<>();
    Set<NodeId> toResume = new HashSet<>();
    Set<NodeId> toAdd = new HashSet<>();

    // Phase 1: Nodes in actual but NOT in desired → deprovision (including SUSPENDED)
    for (Map.Entry<NodeId, NodeStatus> entry : actual.statuses().entrySet()) {
        NodeId nodeId = entry.getKey();
        if (!desired.nodes().containsKey(nodeId)) {
            boolean remove = switch (entry.getValue()) {
                case PRESENT, DRIFTED, SUSPENDED -> true;
                case ABSENT, UNKNOWN -> false;
            };
            if (remove) { /* build removal step from previousDesired or UnknownSpec */ }
        }
    }

    // Phase 2: Nodes in desired → compare actual vs targetStatus
    for (Map.Entry<NodeId, DesiredNode> entry : desired.nodes().entrySet()) {
        NodeId nodeId = entry.getKey();
        DesiredNode node = entry.getValue();
        NodeStatus status = actual.statuses().getOrDefault(nodeId, NodeStatus.UNKNOWN);
        TargetStatus target = node.targetStatus();

        StepAction action = decideAction(status, target);
        if (action == null) continue; // no-op

        // Stateless fallback
        if ((action == StepAction.SUSPEND || action == StepAction.RESUME)
                && !supportsStateful.test(node.type())) {
            action = (action == StepAction.SUSPEND) ? StepAction.DEPROVISION : StepAction.PROVISION;
        }

        switch (action) {
            case PROVISION -> toAdd.add(nodeId);
            case DEPROVISION -> removals.add(new OrderedStep(node, StepAction.DEPROVISION));
            case SUSPEND -> suspensions.add(new OrderedStep(node, StepAction.SUSPEND));
            case RESUME -> toResume.add(nodeId);
        }
    }

    // Topological sort additions (roots-first) and resumptions (roots-first)
    // Topological sort suspensions (leaves-first, same as removals)
    ...
}

private StepAction decideAction(NodeStatus actual, TargetStatus target) {
    return switch (target) {
        case ACTIVE -> switch (actual) {
            case PRESENT -> null;
            case ABSENT, UNKNOWN, DRIFTED -> StepAction.PROVISION;
            case SUSPENDED -> StepAction.RESUME;
        };
        case SUSPENDED -> switch (actual) {
            case SUSPENDED -> null;
            case PRESENT, DRIFTED -> StepAction.SUSPEND;
            case ABSENT, UNKNOWN -> StepAction.RESUME;
        };
    };
}
```

- [ ] **Step 4: Run tests — verify pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=TransitionPlannerTest`

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#152): implement suspend/resume decision matrix in TransitionPlanner"
```

---

### Task 8: SimpleTransitionExecutor — executeSuspend/executeResume

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/SimpleTransitionExecutor.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/SimpleTransitionExecutorTest.java`

**Interfaces:**
- Consumes: `NodeProvisionerRouter.suspend()`, `NodeProvisionerRouter.resume()`, `HumanNodeHandler.onSuspend()`, `HumanNodeHandler.onResume()`, `TransitionPlan.suspensions()`, `TransitionPlan.resumptions()` from Batch 1 and Task 6
- Produces: `executeSuspend()`, `executeResume()` methods; `execute()` method that processes all four step lists

- [ ] **Step 1: Write tests for suspend/resume execution**

Read existing `SimpleTransitionExecutorTest` to match fixture pattern. Add tests:

```java
@Test void executeSuspendCallsRouterSuspend() { ... }
@Test void executeSuspendWithHumanGatingDelegatesToHandler() { ... }
@Test void executeSuspendFailedReturnsFailedOutcome() { ... }
@Test void executeResumeCallsRouterResume() { ... }
@Test void executeResumeWithHumanGatingDelegatesToHandler() { ... }
@Test void executionOrderIsRemovalsSuspensionsResumptionsAdditions() { ... }
@Test void suspendPendingApprovalRecordsPending() { ... }
```

- [ ] **Step 2: Run tests — verify failure**

Run: `mvn --batch-mode test -pl runtime -Dtest=SimpleTransitionExecutorTest`

- [ ] **Step 3: Implement executeSuspend method**

Copy the pattern from `executeProvision()`. Replace:
- `StepAction.PROVISION` → `StepAction.SUSPEND`
- `humanNodeHandler.onProvision()` → `humanNodeHandler.onSuspend()`
- `router.provision()` → `router.suspend()`
- `ProvisionResult` → `SuspendResult`
- `ProvisionContext` → `SuspendContext`

- [ ] **Step 4: Implement executeResume method**

Same pattern, with Resume types.

- [ ] **Step 5: Update execute() to include suspensions and resumptions loops**

Between the removals and additions loops:

```java
for (OrderedStep step : plan.suspensions()) {
    StepOutcome outcome = executeSuspend(step.node(), plan.before(), tenancyId);
    outcomes.put(step.node().id(), outcome);
}
for (OrderedStep step : plan.resumptions()) {
    StepOutcome outcome = executeResume(step.node(), plan.after(), tenancyId);
    outcomes.put(step.node().id(), outcome);
}
```

- [ ] **Step 6: Run tests — verify pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=SimpleTransitionExecutorTest`

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(#152): add executeSuspend/executeResume to SimpleTransitionExecutor"
```

---

### Task 9: ReconciliationLoop — fault feedback + recovery + CloudEvents

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopTest.java`

**Interfaces:**
- Consumes: `TransitionPlan.suspensions()`, `TransitionPlan.resumptions()`, `FaultType.SUSPEND_FAILED`, `FaultType.RESUME_FAILED`, `NodeStatus.SUSPENDED`, `DesiredNode.targetStatus()`, `NodeSuspendedData`, `NodeResumedData`, `ReconciliationCompletedData` (extended) from Batch 1
- Produces: Correct fault type classification for suspend/resume failures, target-status-aware recovery detection, CloudEvents for node.suspended/node.resumed, extended ReconciliationCompletedData with suspension/resumption counts

- [ ] **Step 1: Write tests for fault feedback with suspend/resume**

Read existing `ReconciliationLoopTest` for fixture pattern. Add tests:

```java
@Test void suspendFailureProducesSuspendFailedFaultType() { ... }
@Test void resumeFailureProducesResumeFailedFaultType() { ... }
@Test void recoveryDetectionRecognizesSuspendedAsRecoveredForSuspendedTarget() { ... }
@Test void reconciliationCompletedIncludesSuspensionAndResumptionCounts() { ... }
```

- [ ] **Step 2: Run tests — verify failure**

Run: `mvn --batch-mode test -pl runtime -Dtest=ReconciliationLoopTest`

- [ ] **Step 3: Update plan() call to pass supportsStatefulLifecycle predicate**

In `TenantLoop.plan()` (line 728), change:
```java
TransitionPlan plan = planner.plan(desired, actual, previousDesired);
```
to:
```java
TransitionPlan plan = planner.plan(desired, actual, previousDesired,
    type -> router != null && router.supportsStatefulLifecycle(type));
```

- [ ] **Step 4: Update faultFeedback() for 4-action fault types**

Replace the binary `removalNodeIds` set with a `Map<NodeId, StepAction>`:

```java
Map<NodeId, StepAction> plannedActions = new HashMap<>();
for (OrderedStep step : plan.removals()) plannedActions.put(step.node().id(), StepAction.DEPROVISION);
for (OrderedStep step : plan.suspensions()) plannedActions.put(step.node().id(), StepAction.SUSPEND);
for (OrderedStep step : plan.resumptions()) plannedActions.put(step.node().id(), StepAction.RESUME);
for (OrderedStep step : plan.additions()) plannedActions.put(step.node().id(), StepAction.PROVISION);
```

Then in the fault type determination:
```java
FaultType faultType = switch (plannedActions.getOrDefault(entry.getKey(), StepAction.PROVISION)) {
    case PROVISION -> FaultType.PROVISION_FAILED;
    case DEPROVISION -> FaultType.DEPROVISION_FAILED;
    case SUSPEND -> FaultType.SUSPEND_FAILED;
    case RESUME -> FaultType.RESUME_FAILED;
};
```

- [ ] **Step 5: Update emitCycleEvents() — planNodes, faultType, recovery detection**

1. Add suspensions/resumptions to `planNodes` map:
```java
for (OrderedStep step : plan.suspensions()) planNodes.put(step.node().id(), step.node());
for (OrderedStep step : plan.resumptions()) planNodes.put(step.node().id(), step.node());
```

2. Replace `removalNodeIds` with `plannedActions` map (same as faultFeedback).

3. Update recovery detection to be target-status-aware:
```java
for (NodeId problemNode : activeProblems) {
    NodeStatus status = actual.statuses().get(problemNode);
    DesiredNode node = desired.nodes().get(problemNode);
    boolean recovered;
    if (node != null && node.targetStatus() == TargetStatus.SUSPENDED) {
        recovered = status == NodeStatus.SUSPENDED;
    } else {
        recovered = status == NodeStatus.PRESENT;
    }
    if (recovered) { ... }
}
```

4. Update `ReconciliationCompletedData` construction to include suspension/resumption counts:
```java
new ReconciliationCompletedData(tenancyId, version, desired.nodes().size(),
    plan.additions().size(), plan.removals().size(),
    plan.suspensions().size(), plan.resumptions().size(),
    faultCount, Instant.now())
```

- [ ] **Step 6: Add OTel span attributes for suspensions/resumptions**

In `TenantLoop.plan()`, add:
```java
span.setAttribute(AttributeKey.longKey("desiredstate.suspensions"), plan.suspensions().size());
span.setAttribute(AttributeKey.longKey("desiredstate.resumptions"), plan.resumptions().size());
```

- [ ] **Step 7: Run tests — verify pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=ReconciliationLoopTest`

- [ ] **Step 8: Compile and run full test suite**

Run: `mvn --batch-mode test`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#152): update ReconciliationLoop for suspend/resume fault types, recovery, events"
```

---

## Batch 3: Testing + engine-adapter guards + persistence

After this batch: MockNodeProvisioner supports stateful lifecycle, engine-adapter compiles
with fail-fast guards, GraphSerializer handles targetStatus, full project builds and tests pass.

### Task 10: MockNodeProvisioner + engine-adapter guards + GraphSerializer

**Files:**
- Modify: `testing/src/main/java/io/casehub/desiredstate/testing/MockNodeProvisioner.java`
- Modify: `engine-adapter/src/main/java/io/casehub/desiredstate/engine/CaseTransitionExecutor.java`
- Modify: GraphSerializer in `persistence-jpa/` or `persistence-jpa-common/` (find via `ide_find_class`)
- Test: `testing/src/test/java/io/casehub/desiredstate/testing/MockNodeProvisionerTest.java` (if exists)

**Interfaces:**
- Consumes: all api/ types from Batch 1
- Produces: MockNodeProvisioner with configurable suspend/resume behavior; CaseTransitionExecutor fail-fast guard; GraphSerializer targetStatus serialization

- [ ] **Step 1: Update MockNodeProvisioner**

Read existing `MockNodeProvisioner.java` to understand the recording/configuration pattern.
Add:
- `suspend()` and `resume()` methods matching the existing provision/deprovision pattern
- `supportsStatefulLifecycle()` returning a configurable boolean
- Call recording for suspend/resume calls
- Builder/configuration for suspend/resume result behavior

- [ ] **Step 2: Add CaseTransitionExecutor fail-fast guard**

In `CaseTransitionExecutor.execute()`, add before the removal loop:

```java
if (!plan.suspensions().isEmpty() || !plan.resumptions().isEmpty()) {
    throw new UnsupportedOperationException(
        "CaseTransitionExecutor does not yet support suspend/resume. " +
        "Use SimpleTransitionExecutor or wait for engine-adapter support.");
}
```

- [ ] **Step 3: Update GraphSerializer for targetStatus**

Find GraphSerializer via `ide_find_class`. Update:

Serialization — add after the humanGating line:
```java
nodeObj.put("targetStatus", node.targetStatus().name());
```

Deserialization — add targetStatus read with backward-compatible default:
```java
TargetStatus targetStatus = nodeObj.has("targetStatus")
    ? TargetStatus.valueOf(nodeObj.get("targetStatus").asText())
    : TargetStatus.ACTIVE;
```

Pass `targetStatus` to the new 5-arg DesiredNode constructor.

- [ ] **Step 4: Update GraphSerializer for HumanGating (now a record, not enum)**

Since HumanGating changed from enum to record, `humanGating.name()` no longer works.
Update serialization to serialize the set of gated actions:
```java
ArrayNode gatedArray = mapper.createArrayNode();
for (StepAction action : node.humanGating().gatedActions()) {
    gatedArray.add(action.name());
}
nodeObj.set("humanGating", gatedArray);
```

Deserialization:
```java
HumanGating gating;
JsonNode gatingNode = nodeObj.get("humanGating");
if (gatingNode.isArray()) {
    Set<StepAction> actions = EnumSet.noneOf(StepAction.class);
    for (JsonNode a : gatingNode) actions.add(StepAction.valueOf(a.asText()));
    gating = actions.isEmpty() ? HumanGating.NONE : HumanGating.of(actions.toArray(StepAction[]::new));
} else {
    // Backward compat: old enum string format
    gating = switch (gatingNode.asText()) {
        case "NONE" -> HumanGating.NONE;
        case "PROVISION_ONLY" -> HumanGating.of(StepAction.PROVISION);
        case "DEPROVISION_ONLY" -> HumanGating.of(StepAction.DEPROVISION);
        case "ALL" -> HumanGating.all();
        default -> HumanGating.NONE;
    };
}
```

- [ ] **Step 5: Run full test suite**

Run: `mvn --batch-mode test`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#152): update MockNodeProvisioner, CTE guard, GraphSerializer for suspend/resume"
```

---

## References

- [2026-09-28-suspend-resume-lifecycle-design.md] — design spec this plan implements
- api/src/main/java/io/casehub/desiredstate/api/NodeProvisioner.java:26 — current SPI
- api/src/main/java/io/casehub/desiredstate/api/NodeStatus.java:6 — current status enum
- api/src/main/java/io/casehub/desiredstate/api/StepAction.java:3 — current action enum
- api/src/main/java/io/casehub/desiredstate/api/DesiredNode.java:5 — current node record
- api/src/main/java/io/casehub/desiredstate/api/TransitionPlan.java:6 — current plan record
- api/src/main/java/io/casehub/desiredstate/api/HumanGating.java:3 — current gating enum
- api/src/main/java/io/casehub/desiredstate/api/FaultType.java:3 — current fault types
- api/src/main/java/io/casehub/desiredstate/api/CompletionCondition.java:4 — completion condition
- api/src/main/java/io/casehub/desiredstate/api/ReconciliationCompletedData.java:6 — event data
- api/src/main/java/io/casehub/desiredstate/api/DesiredStateEventTypes.java:3 — event type URIs
- runtime-core/src/main/java/io/casehub/desiredstate/runtime/TransitionPlanner.java:27 — planner
- runtime-core/src/main/java/io/casehub/desiredstate/runtime/SimpleTransitionExecutor.java:38 — executor
- runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java:88 — loop
- engine-adapter/src/main/java/io/casehub/desiredstate/engine/CaseTransitionExecutor.java:44 — CTE
- GitHub #152 — focal issue
