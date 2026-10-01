# Drift Policy Layer — Permitted Drift Exemptions with Revert Modes

**Issue:** casehubio/casehub-desiredstate#160
**Parent:** casehubio/casehub-desiredstate#153 (IoT consumer requirements)
**Date:** 2026-10-01
**Status:** Design

## Problem

The reconciliation loop detects drift (`NodeStatus.DRIFTED`) but has no policy model for permitted drift. Every drifted node immediately:

1. Generates a `FaultEvent(NODE_DEGRADED)` through `FaultPolicyEngine`
2. Gets a `StepAction.PROVISION` from `TransitionPlanner.decideAction(DRIFTED, ACTIVE)`
3. Emits a `NodeDriftedData` CloudEvent

There is no way to say "this drift is permitted — don't reconcile yet." Any domain with reactive overrides needs this: IoT thermostats overridden by users, SOC emergency manual scaling, scheduled maintenance windows.

Drift exemption is a pre-planning concern ("should I act on this drift?"), distinct from fault response ("action failed, what now?"). Different lifecycle, different question.

## Design

### New API Types (api/)

#### DriftPolicy SPI

```java
public interface DriftPolicy {
    DriftDecision evaluate(NodeId nodeId, NodeStatus status,
                           DesiredNode node, DriftContext context);
}
```

Domain implementations return `DriftDecision.reconcile()` (default) or `DriftDecision.exempt(spec)`. Policies **must** declare `@Priority` for deterministic evaluation ordering.

#### DriftDecision

```java
public sealed interface DriftDecision {
    record Reconcile() implements DriftDecision {}
    record Exempt(ExemptionSpec spec) implements DriftDecision {}

    static DriftDecision reconcile() { return new Reconcile(); }
    static DriftDecision exempt(ExemptionSpec spec) { return new Exempt(spec); }
}
```

#### ExemptionSpec

```java
public record ExemptionSpec(RevertCondition revertCondition,
                            Map<String, String> metadata) {
    public ExemptionSpec {
        Objects.requireNonNull(revertCondition);
        metadata = metadata != null ? Map.copyOf(metadata) : Map.of();
    }
}
```

`RevertMode` is not a separate field — it is derivable from `RevertCondition.mode()` (see below). This avoids the consistency hazard of mismatched mode/condition pairs.

#### RevertMode

Enum for serialization and display. Derived from `RevertCondition`, never stored independently in `ExemptionSpec`.

```java
public enum RevertMode { DURATION, SCHEDULE, STATUS_CHANGE, NEVER }
```

Note: The original platform#486 "event" revert mode ("revert when no-motion for 10m") describes domain events, not `NodeStatus` changes. Arbitrary domain event revert is handled by imperative `exemptionStore.revoke()` from domain event handlers. `STATUS_CHANGE` covers the reconciliation-level concern: revert when the node's actual status transitions (e.g., DRIFTED → PRESENT).

#### RevertCondition

Declarative sealed type — serializable, inspectable, loggable. Replaces the original `Predicate<ActualState>` proposal (not serializable, incompatible with api/ SPI patterns and JPA persistence).

```java
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
        public OnDuration { Objects.requireNonNull(duration); }
    }
    record OnSchedule(String cronExpression) implements RevertCondition {
        public OnSchedule { Objects.requireNonNull(cronExpression); }
    }
    record OnStatusChange(Set<NodeStatus> triggerStatuses)
            implements RevertCondition {
        public OnStatusChange {
            Objects.requireNonNull(triggerStatuses);
            triggerStatuses = Set.copyOf(triggerStatuses);
        }
    }
    record Never() implements RevertCondition {}
}
```

#### DriftContext

```java
public record DriftContext(String tenancyId, DesiredStateGraph graph,
                           ActualState actual) {
    public DriftContext {
        Objects.requireNonNull(tenancyId);
        Objects.requireNonNull(graph);
        Objects.requireNonNull(actual);
    }
}
```

**Graph scope note:** `DriftContext.graph` receives the working graph for the current reconciliation cycle. In `reconcile()`, this is the full desired graph. In `reconcileTypes()` (interval-grouped reconciliation), this is the type-filtered graph. A `DriftPolicy` that makes decisions based on graph topology will receive an incomplete graph during type-filtered reconciliation — this is consistent with how `FaultPolicy` receives the working graph today.

#### Exemption

```java
public record Exemption(NodeId nodeId, ExemptionSpec spec,
                        Instant grantedAt, Instant expiresAt) {
    public Exemption {
        Objects.requireNonNull(nodeId);
        Objects.requireNonNull(spec);
        Objects.requireNonNull(grantedAt);
    }

    public boolean shouldRevert(Instant now, NodeStatus currentStatus) {
        if (expiresAt != null && now.isAfter(expiresAt)) {
            return true;
        }
        return switch (spec.revertCondition()) {
            case RevertCondition.OnDuration d ->
                now.isAfter(grantedAt.plus(d.duration()));
            case RevertCondition.OnSchedule s ->
                false; // cron evaluation delegated to platform utility
            case RevertCondition.OnStatusChange sc ->
                sc.triggerStatuses().contains(currentStatus);
            case RevertCondition.Never n -> false;
        };
    }
}
```

`shouldRevert()` pattern-matches on the `RevertCondition` variant, checking both time-based expiry and status-based conditions. `OnSchedule` evaluation is deferred to platform cron utilities (out of scope for this issue).

#### ExemptionStore SPI

Pluggable storage, keyed by `(tenancyId, nodeId)`. One active exemption per node — new grants overwrite old.

```java
public interface ExemptionStore {
    void grant(String tenancyId, NodeId nodeId, Exemption exemption);
    void revoke(String tenancyId, NodeId nodeId);
    Optional<Exemption> get(String tenancyId, NodeId nodeId);
    Map<NodeId, Exemption> getActive(String tenancyId);
    void evict(String tenancyId, Set<NodeId> retainedNodes);
}
```

#### InMemoryExemptionStore

Default implementation in `api/` (same pattern as `InMemoryFaultCountStore`). `ConcurrentHashMap` with `(tenancyId, nodeId)` composite key. Thread-safe. Not CDI-managed — used as builder default.

#### NodeDriftExemptedData

```java
public record NodeDriftExemptedData(
        String tenancyId, String nodeId, String nodeType,
        String revertMode, String revertCondition,
        long graphVersion, String parentNodeId) {
    public NodeDriftExemptedData {
        Objects.requireNonNull(tenancyId);
        Objects.requireNonNull(nodeId);
        Objects.requireNonNull(nodeType);
        Objects.requireNonNull(revertMode);
    }
}
```

### Runtime Integration (runtime-core/)

#### DriftPolicyEngine

Aggregates multiple `DriftPolicy` beans. Evaluates in `@Priority` order (highest first). First EXEMPT terminates evaluation.

```java
public class DriftPolicyEngine {
    private final List<DriftPolicy> policies;

    public DriftPolicyEngine(List<DriftPolicy> policies) {
        this.policies = List.copyOf(policies);
    }

    public DriftDecision evaluate(NodeId nodeId, NodeStatus status,
                                  DesiredNode node, DriftContext context) {
        for (DriftPolicy policy : policies) {
            DriftDecision decision = policy.evaluate(nodeId, status,
                                                     node, context);
            if (decision instanceof DriftDecision.Exempt) {
                return decision;
            }
        }
        return DriftDecision.reconcile();
    }
}
```

#### ReconciliationLoop Changes

**New fields:**
- `DriftPolicyEngine driftPolicyEngine`
- `ExemptionStore exemptionStore`

**Builder additions:**
- `.driftPolicyEngine(DriftPolicyEngine)`
- `.exemptionStore(ExemptionStore)`

**detectDrift() revised flow:**

```
for each desired node with actual status DRIFTED:
    1. Check existing exemption in ExemptionStore:
       - If shouldRevert(now, currentStatus) returns true:
         revoke and proceed to step 2
       - If active and not reverted: add to exemptNodes,
         continue to next node
    2. Evaluate DriftPolicyEngine:
       - EXEMPT: compute expiresAt from RevertCondition, call
         exemptionStore.grant(), add to exemptNodes
       - RECONCILE: create FaultEvent, evaluate through
         FaultPolicyEngine (existing behavior)
```

**Event emission:** `NodeDriftExemptedData` events are NOT emitted inside `detectDrift()`. All CloudEvent emission is deferred to `emitCycleEvents()`, which iterates the `exemptNodes` set — same pattern as `NodeDriftedData` emission from the `driftedNodes` set. This preserves the existing architectural separation: `detectDrift()` is a computation step with no side effects beyond its return value and output parameters.

**FaultPolicy interaction:** When a node is exempted, no `FaultEvent(NODE_DEGRADED)` is created. This means existing `FaultPolicy` implementations (e.g., `SchemaDriftFaultPolicy`, `ZoneRebalanceFaultPolicy`) will never see exempted nodes. This is by design — an exempted node should not trigger fault responses. However, this means registering a `DriftPolicy` that exempts a node type will silently suppress ALL `FaultPolicy` processing for that node's drift. Domain developers must be aware that drift exemption is "all or nothing" — it suppresses both fault processing and re-provisioning.

**Method signature change:**
```java
private DesiredStateGraph detectDrift(DesiredStateGraph desired,
                                      ActualState actual,
                                      Set<NodeId> driftedNodesOut,
                                      Set<NodeId> exemptNodesOut)
```

**reconcile() and reconcileTypes() threading:**
```java
Set<NodeId> exemptNodes = new HashSet<>();
desired = detectDrift(desired, actual, driftedNodesOut, exemptNodes);
TransitionPlan plan = plan(desired, actual, exemptNodes);
```

#### TransitionPlanner Changes

**New plan() overload:**
```java
public TransitionPlan plan(DesiredStateGraph desired, ActualState actual,
                           Set<NodeId> exemptNodes) {
    return plan(desired, actual, null, type -> false, exemptNodes);
}
```

**Full signature:**
```java
public TransitionPlan plan(DesiredStateGraph desired, ActualState actual,
                           DesiredStateGraph previousDesired,
                           Predicate<NodeType> supportsStateful,
                           Set<NodeId> exemptNodes)
```

**Exemption check in caller loop, not in decideAction():** `decideAction(NodeStatus, TargetStatus)` remains a pure stateless classification function (exhaustive switch on the status×target matrix). The exemption check is applied AFTER `decideAction()` returns, in the calling loop inside `plan()`:

```java
StepAction action = decideAction(status, node.targetStatus());
if (action == null) { continue; }
if (status == NodeStatus.DRIFTED && exemptNodes.contains(nodeId)) { continue; }
```

This preserves `decideAction()` as a pure status classifier and adds the exemption as an override in the caller — where `nodeId` and `exemptNodes` context naturally live.

Backward-compatible: existing overloads pass `Set.of()` for exemptNodes.

#### TenantLoop.plan() Changes

The private `TenantLoop.plan()` method gains a `Set<NodeId> exemptNodes` parameter and forwards it to `TransitionPlanner.plan()`:

```java
private TransitionPlan plan(DesiredStateGraph desired, ActualState actual,
                            Set<NodeId> exemptNodes) {
    DesiredStateGraph previousDesired =
        reconciliationStateStore.load(tenancyId).orElse(null);
    TransitionPlan plan = planner.plan(desired, actual, previousDesired,
        type -> router != null && router.supportsStatefulLifecycle(type),
        exemptNodes);
    reconciliationStateStore.store(tenancyId, desired);
    return plan;
}
```

#### ReconciliationEventEmitter

New method:
```java
public CloudEvent nodeDriftExempted(NodeDriftExemptedData data)
```

#### DesiredStateEventTypes

New constant:
```java
public static final String NODE_DRIFT_EXEMPTED =
    "io.casehub.desiredstate.node.drift.exempted";
```

#### ReconciliationCompletedData

New field: `int exemptedCount` — number of drift-exempt nodes this cycle.

#### ExemptionEvictionListener

`@ApplicationScoped` `GlobalReconciliationListener`. Calls `exemptionStore.evict(tenancyId, retainedNodes)` after each cycle and on tenant stop. Same pattern as `FaultCountEvictionListener`.

### CDI Wiring (runtime/)

No separate `CdiDriftPolicyEngine` bridge class — follows the `FaultPolicyEngine` pattern where the engine is produced directly in `RuntimeBeans`.

No `DefaultDriftPolicy` bean — `DriftPolicyEngine.evaluate()` already returns `DriftDecision.reconcile()` when the policy list is empty or all policies return RECONCILE. A `@DefaultBean` that returns RECONCILE adds no behavior. `FaultPolicy` has no equivalent default bean.

| Bean | Type | Notes |
|------|------|-------|
| `DefaultExemptionStore` | `@DefaultBean @ApplicationScoped` | Wraps `InMemoryExemptionStore`. Yields to JPA store when persistence-jpa on classpath |
| `ExemptionEvictionListener` | `@ApplicationScoped` | `GlobalReconciliationListener` for eviction |
| `RuntimeBeans` additions | `@Produces` | `DriftPolicyEngine` from `Instance<DriftPolicy>` sorted by `@Priority`; wire `DriftPolicyEngine` and `ExemptionStore` into `ReconciliationLoop` constructor |

**ReconciliationLoop constructor:** The public constructor gains `DriftPolicyEngine` and `ExemptionStore` parameters (same pattern as the existing `FaultPolicyEngine` and `ReconciliationStateStore` parameters). `RuntimeBeans` produces these and passes them to the constructor. The builder also gains corresponding methods for test/consumer use.

### Spring Auto-Configuration (runtime-spring/)

All new beans added to the existing `DesiredStateRuntimeAutoConfiguration` class (not a separate auto-configuration — follows the established single-class convention):
- `@Bean DriftPolicyEngine` — collects `List<DriftPolicy>`, sorts by `@Order`/`@Priority`
- `@Bean @ConditionalOnMissingBean ExemptionStore` — defaults to `InMemoryExemptionStore`
- Wire into `ReconciliationLoop` constructor

### Test Fixtures (testing/)

- `MockDriftPolicy` — configurable per-node decisions
- `MockExemptionStore` — in-memory with assertion helpers

## Breaking Changes

**NodeDriftedData semantic change:** Exempt nodes no longer emit `NodeDriftedData`. Existing consumers counting `NodeDriftedData` for total drift visibility will undercount after this change. Migration: consumers wanting total drift counts must subscribe to both `NODE_DRIFTED` and `NODE_DRIFT_EXEMPTED` event types.

## Testing Strategy

### Unit Tests

- `DriftPolicyEngineTest` — single policy, chain, first-EXEMPT-wins, priority ordering, all-RECONCILE default
- `InMemoryExemptionStoreTest` — grant, revoke, get, getActive, evict, overwrite semantics
- `RevertConditionTest` — OnDuration expiry, OnStatusChange matching, OnSchedule cron evaluation, Never
- `ExemptionTest` — isExpired logic

### Integration Tests

- `ReconciliationLoopDriftPolicyTest`:
  - Drifted node with no DriftPolicy → reconciled (backward compatible)
  - Drifted node with EXEMPT policy → not re-provisioned, `NodeDriftExemptedData` emitted, no `NodeDriftedData`
  - Duration-based exemption expiry → reconciled on next cycle after duration elapses
  - `OnStatusChange` revert → reconciled when node status matches trigger
  - Multiple policies with `@Priority` → highest-priority EXEMPT wins
  - Imperative `exemptionStore.grant()` → node exempt without DriftPolicy involvement
  - Imperative `exemptionStore.revoke()` + policy returns RECONCILE → node reconciled on next cycle
  - Imperative `exemptionStore.revoke()` without policy change → policy re-grants on next cycle (documenting the D5 contract: revoke must be paired with policy input updates)
  - Retriggering: policy returns fresh EXEMPT each cycle, duration resets
  - `ReconciliationCompletedData.exemptedCount` reflects exempt node count
  - Type-filtered `reconcileTypes()` respects exemptions
  - Eviction: exempt nodes removed from graph are evicted from store

### CloudEvent Tests

- `ReconciliationLoopCloudEventTest` additions:
  - `NodeDriftExemptedData` emitted with correct revertMode, revertCondition
  - Exempt nodes do NOT emit `NodeDriftedData`

## Module Placement

| New type | Module | Package |
|----------|--------|---------|
| `DriftPolicy` | `api/` | `io.casehub.desiredstate.api` |
| `DriftDecision` | `api/` | `io.casehub.desiredstate.api` |
| `DriftContext` | `api/` | `io.casehub.desiredstate.api` |
| `ExemptionSpec` | `api/` | `io.casehub.desiredstate.api` |
| `RevertMode` | `api/` | `io.casehub.desiredstate.api` (enum, derived via `RevertCondition.mode()`) |
| `RevertCondition` | `api/` | `io.casehub.desiredstate.api` |
| `Exemption` | `api/` | `io.casehub.desiredstate.api` |
| `ExemptionStore` | `api/` | `io.casehub.desiredstate.api` |
| `InMemoryExemptionStore` | `api/` | `io.casehub.desiredstate.api` |
| `NodeDriftExemptedData` | `api/` | `io.casehub.desiredstate.api` |
| `DriftPolicyEngine` | `runtime-core/` | `io.casehub.desiredstate.runtime` |
| `ExemptionEvictionListener` | `runtime/` | `io.casehub.desiredstate.runtime` |
| `DefaultExemptionStore` | `runtime/` | `io.casehub.desiredstate.runtime` |
| `RuntimeBeans` additions | `runtime/` | `@Produces DriftPolicyEngine` from `Instance<DriftPolicy>` |
| `DesiredStateRuntimeAutoConfiguration` additions | `runtime-spring/` | `@Bean DriftPolicyEngine`, `@Bean ExemptionStore` |
| `MockDriftPolicy` | `testing/` | `io.casehub.desiredstate.testing` |
| `MockExemptionStore` | `testing/` | `io.casehub.desiredstate.testing` |

## Not in Scope

- Drift policy declaration surfaces (YAML `driftPolicy:`, annotation `@DriftPolicyDef`) — layers on top of this SPI
- Domain-specific drift policies (IoT, ops implement in their own repos)
- JPA-backed `ExemptionStore` (persistence-jpa/) — separate issue, follows the JpaFaultCountStore pattern
- OnSchedule cron evaluation utility — may reuse platform's existing cron infrastructure

## References

- ReconciliationLoop.java:706-741 — current detectDrift implementation
- TransitionPlanner.java:95-108 — decideAction mapping DRIFTED→PROVISION
- FaultPolicyEngine.java — chain evaluation pattern (structural reference for DriftPolicyEngine)
- FaultCountStore.java — pluggable store SPI pattern
- InMemoryFaultCountStore.java — in-memory store implementation pattern
- FaultCountEvictionListener.java — eviction listener pattern
- ReconciliationEventEmitter.java — CloudEvent emission pattern
- NodeDriftedData.java — existing drift event record
- DesiredStateEventTypes.java — event type constants
- casehubio/platform#486 § Drift policy model — platform-level drift policy design
- casehubio/casehub-desiredstate#153 — IoT consumer requirements (parent issue)
- casehubio/iot#120 — IoT desired state integration
- casehubio/casehub-ops#26 — SOC adaptive ops
