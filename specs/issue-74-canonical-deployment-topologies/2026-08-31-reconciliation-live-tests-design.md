# Reconciliation + Live K8s Deployment Tests — Implementation Design

**Date:** 2026-08-31
**Issue:** casehubio/casehub-ops#82
**Parent spec:** `wsp-casehub-ops/specs/canonical-deployment-topologies/2026-08-29-canonical-deployment-topologies-design.md` §8
**Decisions:** `specs/issue-74-canonical-deployment-topologies/decisions.md`

---

## 1. Scope

Add two profile-gated test layers to the `topology-tests` module in casehub-ops:

1. **Reconciliation tests** (`-Preconciliation`): synchronous planner + executor tests with stubbed provisioning and deterministic fault injection
2. **Live K8s tests** (`-Pinfra-live`): real K8s deployments, skipped when no cluster available

Both layers build on the existing `TopologyTestBase` compilation infrastructure.

---

## 2. Test Infrastructure

### 2.1 FailableNodeProvisioner

Implements `NodeProvisioner` (desiredstate SPI). Test fixture in `topology-tests`.

```java
public class FailableNodeProvisioner implements NodeProvisioner {
    // Records all provision/deprovision calls in order
    public final List<DesiredNode> provisioned;
    public final List<DesiredNode> deprovisioned;

    // Per-NodeId failure injection: failNode("db", 3) → first 3 provisions fail
    void failNode(String nodeId, int times);

    // Per-NodeType failure: all nodes of this type fail
    void failType(NodeType type);

    // Configure handled types
    void setHandledTypes(Set<NodeType> types);
}
```

Provides ordering verification and deterministic failure injection for fault policy testing.

### 2.2 ReconciliationTestBase

Extends `TopologyTestBase` with reconciliation infrastructure:

- `TransitionPlanner` — plan computation
- `SimpleTransitionExecutor` — synchronous execution with real provisioner routing
- `DefaultNodeProvisionerRouter` — routes by NodeType to `FailableNodeProvisioner`
- `FaultPolicyEngine` — evaluates fault policies against transition results
- `ThresholdFaultPolicy` — configurable escalation tiers
- `MockActualStateAdapter` — controllable actual state
- `DefaultActualStateAdapterRouter` — routes adapter calls
- No-op `HumanNodeHandler`, `PendingApprovalHandler`, `LifecycleStepExecutor`

Helper methods:
- `planFromEmpty(graph)` — plans transitions from empty actual state
- `executeTransition(plan)` — executes via SimpleTransitionExecutor
- `assertOrderedBefore(plan, nodeIdA, nodeIdB)` — verifies topological ordering
- `assertDependencyChain(plan, ids...)` — verifies a full dependency chain ordering
- `reconcileWithFaults(graph, faultPolicy, failNodes)` — full cycle: plan → execute → fault feedback

### 2.3 MockActualStateAdapter Usage

From `desiredstate/testing` module. Supports:
- `setAllPresent(graph)` — all nodes present (for drift tests)
- `setStatus(nodeId, status)` — per-node control
- Dynamic `handledTypes` configuration

---

## 3. Reconciliation Test Classes

One class per architecture family, testing the core exemplars:

| Class | Exemplar(s) | Key verifications |
|-------|-------------|-------------------|
| `SingleServiceReconciliationTest` | single-node-blog | Namespace → deployment → service ordering |
| `MultiTierReconciliationTest` | single-node-dev, lb-cluster-ecommerce | Linear chain ordering (web → app → data), lifecycle phase handling |
| `MicroservicesReconciliationTest` | ha-multi-az-trading | forEach-stamped nodes, cross-AZ ordering |
| `EventDrivenReconciliationTest` | lb-cluster-telemetry | Broker-centric dependency ordering |
| `SidecarMeshReconciliationTest` | lb-cluster-logistics | Mesh control plane before sidecar proxies |

### 3.1 Test Categories

Each class covers:

1. **Plan correctness** — compile from YAML, plan from empty actual state, assert correct additions count and zero removals
2. **Provision ordering** — assert topological ordering respects dependency edges (namespace before deployments, data tier before app tier)
3. **Drift detection** — set all nodes present, mark one as ABSENT/DRIFTED, re-plan, assert only the drifted node is in additions
4. **Fault escalation** — inject failures for a specific node via FailableNodeProvisioner, run plan+execute+fault N times, verify ThresholdFaultPolicy creates a review node after threshold

---

## 4. Live K8s Test Class

Single class: `LiveDeploymentTest` tagged `@Tag("infra-live")`.

Guarded by `@EnabledIf` checking K8s API reachability. Tests:

1. Namespace creation — compile single-service exemplar, verify namespace node provisions successfully
2. Deployment Ready — verify deployment reaches Ready state
3. Service DNS resolution — verify service is DNS-resolvable within cluster

Uses real Docker images (nginx) but scoped to K8s-native types only.

---

## 5. Maven Profile Configuration

```xml
<!-- Default: exclude reconciliation and live tags -->
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <excludedGroups>reconciliation,infra-live</excludedGroups>
    </configuration>
</plugin>

<!-- -Preconciliation: run reconciliation tests -->
<profile>
    <id>reconciliation</id>
    <build><plugins><plugin>
        <artifactId>maven-surefire-plugin</artifactId>
        <configuration combine.self="override">
            <groups>reconciliation</groups>
        </configuration>
    </plugin></plugins></build>
</profile>

<!-- -Pinfra-live: run live K8s tests -->
<profile>
    <id>infra-live</id>
    <build><plugins><plugin>
        <artifactId>maven-surefire-plugin</artifactId>
        <configuration combine.self="override">
            <groups>infra-live</groups>
        </configuration>
    </plugin></plugins></build>
</profile>
```

---

## 6. Dependencies (topology-tests pom.xml additions)

```xml
<!-- Already present: casehub-desiredstate, casehub-desiredstate-yaml, casehub-ops-api, casehub-ops-infra -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-desiredstate-testing</artifactId>
    <scope>test</scope>
</dependency>
<!-- For live tests: Kubernetes client -->
<dependency>
    <groupId>io.fabric8</groupId>
    <artifactId>kubernetes-client</artifactId>
    <scope>test</scope>
    <optional>true</optional>
</dependency>
```

---

## 7. Acceptance Criteria Mapping

| Criterion | Test | Location |
|-----------|------|----------|
| TransitionPlan correctness for core exemplars | Plan assertions in all 5 reconciliation test classes | `reconciliation/` |
| Provision ordering respects dependency edges | `assertOrderedBefore` in each test class | `reconciliation/` |
| Drift detection triggers re-provisioning | Drift test in each class — set present, mark absent, re-plan | `reconciliation/` |
| Fault policy escalation creates review node | Fault test in MultiTierReconciliationTest (3 failures → review node) | `reconciliation/` |
| Live tests verify namespace, deployment, service | LiveDeploymentTest | `live/` |
| Profile-gated, default build passes without K8s | Surefire tag exclusion, @EnabledIf guard | pom.xml |

---

## References

- `wsp-casehub-ops/specs/canonical-deployment-topologies/2026-08-29-canonical-deployment-topologies-design.md` §8 — verification strategy
- `runtime/src/test/java/.../ReconciliationLoopTest.java` — existing async reconciliation test pattern
- `testing/src/main/java/.../MockNodeProvisioner.java` — existing mock provisioner pattern
- `testing/src/main/java/.../MockActualStateAdapter.java` — actual state control
- `infra/src/main/java/.../InfraFaultPolicy.java` — ThresholdFaultPolicy usage pattern
- GE-20260516-3a27dc — `combine.self="override"` for Maven surefire profiles
- GE-20260416-ca1c71 — `*IT.java` silent skip gotcha
