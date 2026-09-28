# Suspend/Resume Lifecycle Verbs on NodeProvisioner SPI

**Date:** 2026-09-28
**Issue:** casehubio/casehub-desiredstate#152
**Status:** Design
**Forcing function:** casehubio/claudony#234 — agent pool tmux sessions carry persisted state

---

## 1. Problem

NodeProvisioner supports two verbs: `provision()` (create) and `deprovision()` (destroy). This covers
stateless resources (DNS records, API routes) but not stateful resources where the process is ephemeral
but state persists (CLI agent sessions, containers, VMs, cached computation environments).

Destroying and recreating a stateful resource wastes its persisted state. The correct lifecycle is
suspend (stop the process, state persists on disk/storage) and resume (restart the process, restore
from persisted state). A provisioner that doesn't support these verbs uses the stateless path — destroy
on idle, create on demand.

## 2. Design Principles

1. **Stateful layers on stateless.** Suspend/resume are additive verbs. Every stateful resource still
   supports provision (first-time create) and deprovision (permanent destroy). The capability is
   additive, not a fork.

2. **Desired graph is the single source of truth.** The GoalCompiler declares lifecycle intent per
   node via a target status annotation on DesiredNode. No out-of-band state, no separate APIs.

3. **Provisioner has domain knowledge.** The planner decides *what* action to take; the provisioner
   decides *how*. When a node is ABSENT but target is SUSPENDED, the planner calls resume() — the
   provisioner knows whether persisted state is recoverable.

4. **Backward compatible.** All changes are default methods, new constructors with defaults, or
   additive enum values. Existing provisioners compile and run unchanged.

## 3. API Changes

### 3.1 TargetStatus enum (new — api/)

```java
public enum TargetStatus {
    ACTIVE,
    SUSPENDED
}
```

Declares the GoalCompiler's intent for a node's lifecycle state.

### 3.2 DesiredNode (modified — api/)

```java
public record DesiredNode(
    NodeId id, NodeSpec spec, HumanGating humanGating,
    HookDescriptor hooks, TargetStatus targetStatus
) {
    // Existing 3-arg constructor → defaults hooks=null, targetStatus=ACTIVE
    public DesiredNode(NodeId id, NodeSpec spec, HumanGating humanGating) {
        this(id, spec, humanGating, null, TargetStatus.ACTIVE);
    }

    // Existing 4-arg constructor → defaults targetStatus=ACTIVE
    public DesiredNode(NodeId id, NodeSpec spec, HumanGating humanGating, HookDescriptor hooks) {
        this(id, spec, humanGating, hooks, TargetStatus.ACTIVE);
    }
}
```

### 3.3 NodeStatus (modified — api/)

```java
public enum NodeStatus {
    PRESENT,
    ABSENT,
    DRIFTED,
    UNKNOWN,
    SUSPENDED  // new — resource dormant, state persisted
}
```

The ActualStateAdapter reports SUSPENDED when a resource is idle but its persisted state is
recoverable (e.g. tmux process killed, conversation history on disk).

**Impact:** All `switch` statements on NodeStatus gain a new case. Java's exhaustive switch
enforcement flags every omission at compile time.

### 3.4 StepAction (modified — api/)

```java
public enum StepAction { PROVISION, DEPROVISION, SUSPEND, RESUME }
```

### 3.5 SuspendResult / ResumeResult (new — api/)

```java
public sealed interface SuspendResult {
    record Success() implements SuspendResult {}
    record Failed(String reason) implements SuspendResult {}
    record PendingApproval(NodeId nodeId, String planReference) implements SuspendResult {}
}

public sealed interface ResumeResult {
    record Success() implements ResumeResult {}
    record Failed(String reason) implements ResumeResult {}
    record PendingApproval(NodeId nodeId, String planReference) implements ResumeResult {}
}
```

### 3.6 SuspendContext / ResumeContext (new — api/)

```java
public record SuspendContext(String tenancyId, DesiredStateGraph graph) {
    // Optional approval for re-entry after PendingApproval
    public SuspendContext withApproval(PlanApproval approval) { ... }
    public PlanApproval approval() { ... }
    public boolean hasApproval() { ... }
}

public record ResumeContext(String tenancyId, DesiredStateGraph graph) {
    public ResumeContext withApproval(PlanApproval approval) { ... }
    public PlanApproval approval() { ... }
    public boolean hasApproval() { ... }
}
```

Same pattern as ProvisionContext/DeprovisionContext. The provisioner reads state references
(conversationId, snapshot path, etc.) from the NodeSpec — the generic runtime stays domain-agnostic.

### 3.7 NodeProvisioner (modified — api/)

```java
public interface NodeProvisioner {
    Set<NodeType> handledTypes();
    default Duration resyncInterval() { return Duration.ofMinutes(5); }

    ProvisionResult provision(DesiredNode node, ProvisionContext context);
    DeprovisionResult deprovision(DesiredNode node, DeprovisionContext context);

    // Stateful lifecycle — opt-in via default methods
    default SuspendResult suspend(DesiredNode node, SuspendContext context) {
        return new SuspendResult.Failed("suspend not supported");
    }
    default ResumeResult resume(DesiredNode node, ResumeContext context) {
        return new ResumeResult.Failed("resume not supported");
    }
    default boolean supportsStatefulLifecycle() { return false; }
}
```

Provisioners that support stateful lifecycle override all three methods. The runtime checks
`supportsStatefulLifecycle()` to decide the idle strategy.

### 3.8 NodeProvisionerRouter (modified — api/)

```java
public interface NodeProvisionerRouter {
    ProvisionResult provision(DesiredNode node, ProvisionContext context);
    DeprovisionResult deprovision(DesiredNode node, DeprovisionContext context);
    Duration resyncIntervalFor(NodeType type);
    Set<NodeType> allHandledTypes();

    // New
    SuspendResult suspend(DesiredNode node, SuspendContext context);
    ResumeResult resume(DesiredNode node, ResumeContext context);
    boolean supportsStatefulLifecycle(NodeType type);
}
```

`DefaultNodeProvisionerRouter` delegates to the correct provisioner by NodeType, same as
provision/deprovision.

## 4. TransitionPlan Changes

### 4.1 TransitionPlan (modified — api/)

```java
public record TransitionPlan(
    List<OrderedStep> removals,
    List<OrderedStep> suspensions,    // new
    List<OrderedStep> resumptions,    // new
    List<OrderedStep> additions,
    DesiredStateGraph before,
    DesiredStateGraph after
) { ... }
```

**Execution order:** removals → suspensions → resumptions → additions.

- Removals before suspensions: permanently removed nodes free resources before idle nodes suspend.
- Suspensions before resumptions: suspend nodes that are becoming idle before resuming nodes that
  are becoming active (avoids resource contention).
- Resumptions before additions: restore existing state before creating new resources (dependencies
  may need their state restored before new dependents can provision).

The existing 4-arg constructor is preserved for backward compatibility, defaulting suspensions
and resumptions to empty lists.

### 4.2 TransitionPlanner (modified — runtime-core/)

The planner gains a new decision matrix. For each node, it considers the actual status and the
target status (or absence from the desired graph):

| Actual Status | Target: ACTIVE | Target: SUSPENDED | Not in desired graph |
|---|---|---|---|
| PRESENT | no-op | SUSPEND | DEPROVISION |
| ABSENT | PROVISION | RESUME (attempt) | no-op |
| SUSPENDED | RESUME | no-op | DEPROVISION |
| DRIFTED | PROVISION | SUSPEND | DEPROVISION |
| UNKNOWN | PROVISION | RESUME (attempt) | no-op |

**ABSENT + SUSPENDED → RESUME:** The provisioner has domain knowledge the planner doesn't. A tmux
session's process may be gone (ABSENT) but its conversation history persists on disk. `resume()`
detects whether the state is recoverable. If it isn't, the provisioner returns Failed and the
node faults — the fault policy handles recovery.

**Stateless fallback:** When `supportsStatefulLifecycle()` returns false for a node's type, the
planner substitutes: SUSPEND → DEPROVISION, RESUME → PROVISION. The provisioner never sees
suspend/resume calls — it uses the stateless destroy/create path. This is the key backward
compatibility mechanism.

## 5. Executor Changes

### 5.1 SimpleTransitionExecutor (modified — runtime-core/)

Gains `executeSuspend()` and `executeResume()` methods following the same pattern as
`executeProvision()`/`executeDeprovision()`:

1. Check `requiresHuman(StepAction.SUSPEND)` / `requiresHuman(StepAction.RESUME)` → delegate to HumanNodeHandler
2. Check PendingApprovalHandler for prior approval state
3. Execute pre-hooks (if HookDescriptor has suspend/resume hooks)
4. Call `router.suspend()`/`router.resume()`
5. Handle result: Success → post-hooks → Succeeded, Failed → Failed, PendingApproval → recordPending

The execute() method gains two new loops between removals and additions:

```java
for (OrderedStep step : plan.suspensions()) {
    outcomes.put(step.node().id(), executeSuspend(step.node(), ...));
}
for (OrderedStep step : plan.resumptions()) {
    outcomes.put(step.node().id(), executeResume(step.node(), ...));
}
```

### 5.2 CaseTransitionExecutor (modified — engine-adapter/)

The buildCaseDefinition method gains suspend/resume phases between removal and addition phases.
Worker(Workflow) bindings for suspend/resume steps follow the same pattern as provision/deprovision
bindings. DesiredStateDispatch gains `desiredstate:suspend` and `desiredstate:resume` dispatch
registrations.

### 5.3 HumanNodeHandler (modified — api/)

```java
public interface HumanNodeHandler {
    StepOutcome onProvision(DesiredNode node, ProvisionContext context);
    default StepOutcome onDeprovision(DesiredNode node, DeprovisionContext context) {
        return new StepOutcome.Skipped("human deprovision not handled");
    }
    // New
    default StepOutcome onSuspend(DesiredNode node, SuspendContext context) {
        return new StepOutcome.Skipped("human suspend not handled");
    }
    default StepOutcome onResume(DesiredNode node, ResumeContext context) {
        return new StepOutcome.Skipped("human resume not handled");
    }
}
```

### 5.4 HumanGating (modified — api/)

```java
public enum HumanGating {
    NONE,
    PROVISION_ONLY,
    DEPROVISION_ONLY,
    SUSPEND_ONLY,      // new
    RESUME_ONLY,       // new
    ALL
}
```

`requiresHuman(StepAction)` updated to handle the new values. `ALL` gates all four actions.

## 6. HookDescriptor Impact

If HookDescriptor supports per-action hooks (provisionPre, provisionPost, etc.), it gains:
- `suspendPre()`, `suspendPost()`
- `resumePre()`, `resumePost()`

These are optional — nodes without suspend/resume hooks return empty lists.

## 7. YAML Plugin Impact

### 7.1 Plugin Model (plugin/)

`PluginProvisionerDef` gains optional `suspendSteps` and `resumeSteps` fields.
`PluginDescriptor` gains matching step lists.

### 7.2 YamlPluginProvisioner (plugin/)

`suspend()` and `resume()` methods follow the same dispatch pattern as `provision()`/`deprovision()`:
lookup plugin → build context → execute steps → map result.

`supportsStatefulLifecycle()` returns true when the plugin declares non-empty suspend/resume sections.

### 7.3 YAML Surface

```yaml
plugins:
  my-stateful-plugin:
    spec:
      conversationId: string
    provisioner:
      create:
        - rest-call: ...
      delete:
        - rest-call: ...
      suspend:          # new — optional
        - rest-call: ...
      resume:           # new — optional
        - rest-call: ...
```

Plugins that omit `suspend:`/`resume:` are stateless — `supportsStatefulLifecycle()` returns false.

## 8. Testing Module Impact

`MockNodeProvisioner` gains `suspend()`, `resume()`, `supportsStatefulLifecycle()` with configurable
behavior (success/failure/pending-approval per call, call recording).

## 9. CloudEvent Impact

New event types in `DesiredStateEventTypes`:
- `io.casehub.desiredstate.node.suspended` — emitted when a node is successfully suspended
- `io.casehub.desiredstate.node.resumed` — emitted when a node is successfully resumed

Data records: `NodeSuspendedData`, `NodeResumedData` (mirror `NodeRecoveredData` pattern).

## 10. Persistence Impact

### 10.1 FaultCountStore

No changes — fault counts are keyed by `(namespace, tenancyId, nodeId)`. Suspend/resume actions
that fail produce `FaultEvent`s through the existing fault path.

### 10.2 ReconciliationStateStore / GraphSerializer

`DesiredNode` serialization must include the `targetStatus` field. `GraphSerializer` (used by
`JpaReconciliationStateStore`) needs to handle the new field — Jackson will serialize it naturally
since it's a record component.

## 11. Composition Engine Impact

`CrossDomainCompositionEngine` operates on `DesiredStateGraph` and `DomainNodeSpec`. Since
`DesiredNode.targetStatus` is carried through the graph, composition works transparently — each
domain's GoalCompiler sets its own target statuses, and the composition engine merges them.

## 12. Annotation/YAML/TS-DSL Surface Impact

### 12.1 Annotations

`@Node` and `@DeclareNode` gain an optional `targetStatus` attribute defaulting to `ACTIVE`.
Build-time processor validates values.

### 12.2 YAML Surface

`YamlNode` gains an optional `targetStatus` field. `YamlGraph` → `DesiredStateGraph` mapping
carries it through.

### 12.3 TypeScript DSL

`node()` helper gains an optional `targetStatus` property in the node options.

## 13. Scope Boundaries

**In scope:**
- api/ types (TargetStatus, NodeStatus.SUSPENDED, StepAction.SUSPEND/RESUME, SuspendResult,
  ResumeResult, SuspendContext, ResumeContext, NodeProvisioner changes, NodeProvisionerRouter changes,
  HumanNodeHandler changes, HumanGating changes)
- runtime-core/ (TransitionPlanner, SimpleTransitionExecutor, DefaultNodeProvisionerRouter)
- testing/ (MockNodeProvisioner)
- TransitionPlan structure changes
- CloudEvent types

**Deferred:**
- engine-adapter/ CaseTransitionExecutor changes (separate issue — requires engine-flow coordination)
- work-adapter/ PendingApproval handler changes for suspend/resume
- YAML plugin suspend/resume sections (separate issue — depends on yaml-step-runtime)
- Annotation/YAML/TS-DSL surface attributes (separate issues per surface)
- HookDescriptor suspend/resume hooks
- Persistence migration for targetStatus (GraphSerializer handles it naturally; no schema migration needed)

## References

- api/src/main/java/io/casehub/desiredstate/api/NodeProvisioner.java — current SPI
- api/src/main/java/io/casehub/desiredstate/api/NodeStatus.java — current status enum
- api/src/main/java/io/casehub/desiredstate/api/StepAction.java — current action enum
- api/src/main/java/io/casehub/desiredstate/api/DesiredNode.java — current node record
- api/src/main/java/io/casehub/desiredstate/api/TransitionPlan.java — current plan record
- runtime-core/src/main/java/io/casehub/desiredstate/runtime/TransitionPlanner.java — planning logic
- runtime-core/src/main/java/io/casehub/desiredstate/runtime/SimpleTransitionExecutor.java — execution
- casehubio/claudony#234 — forcing function (agent pool tmux sessions)
- docs/research/2026-06-07-desired-state-management-research.md — original research
