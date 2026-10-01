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
public record ExemptionSpec(RevertMode revertMode,
                            RevertCondition revertCondition,
                            Map<String, String> metadata) {
    public ExemptionSpec {
        Objects.requireNonNull(revertMode);
        Objects.requireNonNull(revertCondition);
        metadata = metadata != null ? Map.copyOf(metadata) : Map.of();
    }
}
```

#### RevertMode

```java
public enum RevertMode { DURATION, SCHEDULE, EVENT, NEVER }
```

#### RevertCondition

Declarative sealed type — serializable, inspectable, loggable. Replaces the original `Predicate<ActualState>` proposal (not serializable, incompatible with api/ SPI patterns and JPA persistence).

```java
public sealed interface RevertCondition {
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

#### Exemption

```java
public record Exemption(NodeId nodeId, ExemptionSpec spec,
                        Instant grantedAt, Instant expiresAt) {
    public Exemption {
        Objects.requireNonNull(nodeId);
        Objects.requireNonNull(spec);
        Objects.requireNonNull(grantedAt);
    }

    public boolean isExpired(Instant now) {
        return expiresAt != null && now.isAfter(expiresAt);
    }
}
```

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
       - If expired (OnDuration) or condition met (OnStatusChange/OnSchedule):
         revoke and proceed to step 2
       - If active and not expired: add to exemptNodes, emit
         NodeDriftExemptedData, continue to next node
    2. Evaluate DriftPolicyEngine:
       - EXEMPT: compute expiresAt from RevertCondition, call
         exemptionStore.grant(), add to exemptNodes, emit
         NodeDriftExemptedData
       - RECONCILE: create FaultEvent, evaluate through
         FaultPolicyEngine (existing behavior)
```

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

**decideAction change:** When `status == DRIFTED && target == ACTIVE`, check `exemptNodes.contains(nodeId)` — if true, return `null` (skip).

Backward-compatible: existing overloads pass `Set.of()` for exemptNodes.

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

| Bean | Type | Notes |
|------|------|-------|
| `CdiDriftPolicyEngine` | CDI bridge | `Instance<DriftPolicy>` → sorted by `@Priority` → `DriftPolicyEngine` |
| `DefaultDriftPolicy` | `@DefaultBean` | Returns `DriftDecision.reconcile()` for all nodes |
| `DefaultExemptionStore` | `@DefaultBean @ApplicationScoped` | Wraps `InMemoryExemptionStore` |
| `ExemptionEvictionListener` | `@ApplicationScoped` | `GlobalReconciliationListener` for eviction |
| `RuntimeBeans` additions | `@Produces` | Wire `DriftPolicyEngine` and `ExemptionStore` into `ReconciliationLoop.Builder` |

### Spring Auto-Configuration (runtime-spring/)

`SpringDriftPolicyAutoConfiguration`:
- `@Bean DriftPolicyEngine` — collects `List<DriftPolicy>`, sorts by `@Order`/`@Priority`
- `@Bean @ConditionalOnMissingBean ExemptionStore` — defaults to `InMemoryExemptionStore`
- Wire into `ReconciliationLoop.Builder`

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
  - Imperative `exemptionStore.revoke()` → node reconciled on next cycle even if policy would exempt
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
| `RevertMode` | `api/` | `io.casehub.desiredstate.api` |
| `RevertCondition` | `api/` | `io.casehub.desiredstate.api` |
| `Exemption` | `api/` | `io.casehub.desiredstate.api` |
| `ExemptionStore` | `api/` | `io.casehub.desiredstate.api` |
| `InMemoryExemptionStore` | `api/` | `io.casehub.desiredstate.api` |
| `NodeDriftExemptedData` | `api/` | `io.casehub.desiredstate.api` |
| `DriftPolicyEngine` | `runtime-core/` | `io.casehub.desiredstate.runtime` |
| `ExemptionEvictionListener` | `runtime/` | `io.casehub.desiredstate.runtime` |
| `CdiDriftPolicyEngine` | `runtime/` | `io.casehub.desiredstate.runtime` |
| `DefaultDriftPolicy` | `runtime/` | `io.casehub.desiredstate.runtime` |
| `DefaultExemptionStore` | `runtime/` | `io.casehub.desiredstate.runtime` |
| `SpringDriftPolicyAutoConfiguration` | `runtime-spring/` | `io.casehub.desiredstate.runtime.spring` |
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
