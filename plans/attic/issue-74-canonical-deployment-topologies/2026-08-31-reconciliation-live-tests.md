# Reconciliation + Live K8s Deployment Tests — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/casehub-ops#82 — feat: reconciliation + live K8s deployment tests (profile-gated)
**Issue group:** casehubio/casehub-ops#74

**Goal:** Add profile-gated reconciliation integration tests and live K8s deployment tests to the topology-tests module, proving that YAML-compiled topology exemplars produce correct transition plans, respect dependency ordering, detect drift, and escalate faults.

**Architecture:** Synchronous TransitionPlanner + SimpleTransitionExecutor tests (no async ReconciliationLoop). A FailableNodeProvisioner test fixture wraps provisioning with per-node failure injection. Tests tagged with JUnit `@Tag` and gated by Maven profiles. Live K8s tests guarded by `@EnabledIf` for K8s availability.

**Tech Stack:** JUnit 5, AssertJ, Maven Surefire profiles, desiredstate runtime (TransitionPlanner, SimpleTransitionExecutor, FaultPolicyEngine, ThresholdFaultPolicy), desiredstate testing module (MockActualStateAdapter)

## Global Constraints

- All test classes must use `*Test.java` suffix (never `*IT.java` — silently skipped by surefire)
- Maven profile surefire config must use `combine.self="override"` to prevent parent merge
- `mvn --batch-mode test` must pass without any profile — reconciliation and live tests excluded by default
- Tests are plain JUnit (not `@QuarkusTest`) — no CDI container needed
- All work in `topology-tests` module within the `casehub-ops` repo

---

## Batch 1: Test Infrastructure + Maven Profile Configuration

### Task 1: pom.xml — Dependencies and Maven profiles

**Files:**
- Modify: `topology-tests/pom.xml`

**Interfaces:**
- Produces: Maven profiles `reconciliation` and `infra-live`; `casehub-desiredstate-testing` on test classpath

- [ ] **Step 1: Add desiredstate-testing dependency to pom.xml**

Add to `<dependencies>` section after the existing `casehub-desiredstate` dependency:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-desiredstate-testing</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Add surefire default exclusion and profiles**

Add surefire plugin configuration and profiles to `pom.xml`:

```xml
<build>
    <plugins>
        <plugin>
            <artifactId>maven-surefire-plugin</artifactId>
            <configuration>
                <excludedGroups>reconciliation,infra-live</excludedGroups>
            </configuration>
        </plugin>
    </plugins>
</build>

<profiles>
    <profile>
        <id>reconciliation</id>
        <build>
            <plugins>
                <plugin>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <configuration combine.self="override">
                        <groups>reconciliation</groups>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    </profile>
    <profile>
        <id>infra-live</id>
        <build>
            <plugins>
                <plugin>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <configuration combine.self="override">
                        <groups>infra-live</groups>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

- [ ] **Step 3: Verify default build still passes**

Run: `mvn --batch-mode test -pl topology-tests -f /Users/mdproctor/claude/casehub/slots/165/ops/pom.xml`
Expected: All existing compilation tests pass, no reconciliation/live tests run

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/165/ops add topology-tests/pom.xml
git -C /Users/mdproctor/claude/casehub/slots/165/ops commit -m "wip: add desiredstate-testing dep + Maven profiles for reconciliation/live tests Refs casehubio/casehub-ops#82"
```

### Task 2: FailableNodeProvisioner test fixture

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/FailableNodeProvisioner.java`

**Interfaces:**
- Produces: `FailableNodeProvisioner` implementing `NodeProvisioner` — `failNode(String nodeId, int times)`, `setHandledTypes(Set<NodeType>)`, `provisioned` (ordered list), `deprovisioned` (ordered list), `clear()`

- [ ] **Step 1: Write the failing test for FailableNodeProvisioner**

Create test file `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/FailableNodeProvisionerTest.java`:

```java
package io.casehub.ops.topology.reconciliation;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("reconciliation")
class FailableNodeProvisionerTest {

    private FailableNodeProvisioner provisioner;

    @BeforeEach
    void setUp() {
        provisioner = new FailableNodeProvisioner();
        provisioner.setHandledTypes(Set.of(NodeType.of("k8s_deployment")));
    }

    @Test
    void provision_succeeds_by_default() {
        DesiredNode node = new DesiredNode(NodeId.of("app"), new TestSpec("k8s_deployment"), HumanGating.NONE);
        ProvisionResult result = provisioner.provision(node, new ProvisionContext("tenant", null));
        assertThat(result).isInstanceOf(ProvisionResult.Success.class);
        assertThat(provisioner.provisioned).containsExactly(node);
    }

    @Test
    void provision_fails_when_node_configured_to_fail() {
        provisioner.failNode("app", 2);
        DesiredNode node = new DesiredNode(NodeId.of("app"), new TestSpec("k8s_deployment"), HumanGating.NONE);
        ProvisionContext ctx = new ProvisionContext("tenant", null);

        ProvisionResult r1 = provisioner.provision(node, ctx);
        assertThat(r1).isInstanceOf(ProvisionResult.Failed.class);

        ProvisionResult r2 = provisioner.provision(node, ctx);
        assertThat(r2).isInstanceOf(ProvisionResult.Failed.class);

        ProvisionResult r3 = provisioner.provision(node, ctx);
        assertThat(r3).isInstanceOf(ProvisionResult.Success.class);
    }

    @Test
    void deprovision_succeeds_by_default() {
        DesiredNode node = new DesiredNode(NodeId.of("app"), new TestSpec("k8s_deployment"), HumanGating.NONE);
        DeprovisionResult result = provisioner.deprovision(node, new DeprovisionContext("tenant", null));
        assertThat(result).isInstanceOf(DeprovisionResult.Success.class);
        assertThat(provisioner.deprovisioned).containsExactly(node);
    }

    @Test
    void records_provision_order() {
        DesiredNode a = new DesiredNode(NodeId.of("a"), new TestSpec("k8s_deployment"), HumanGating.NONE);
        DesiredNode b = new DesiredNode(NodeId.of("b"), new TestSpec("k8s_deployment"), HumanGating.NONE);
        ProvisionContext ctx = new ProvisionContext("tenant", null);
        provisioner.provision(a, ctx);
        provisioner.provision(b, ctx);
        assertThat(provisioner.provisioned).containsExactly(a, b);
    }

    @Test
    void clear_resets_state() {
        provisioner.failNode("x", 1);
        DesiredNode node = new DesiredNode(NodeId.of("x"), new TestSpec("k8s_deployment"), HumanGating.NONE);
        provisioner.provision(node, new ProvisionContext("tenant", null));
        provisioner.clear();
        assertThat(provisioner.provisioned).isEmpty();
        ProvisionResult result = provisioner.provision(node, new ProvisionContext("tenant", null));
        assertThat(result).isInstanceOf(ProvisionResult.Success.class);
    }

    private record TestSpec(String type) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of(type); }
    }
}
```

- [ ] **Step 2: Run test to verify it fails (class not found)**

Run: `mvn --batch-mode test -pl topology-tests -Preconciliation -Dtest=FailableNodeProvisionerTest -f /Users/mdproctor/claude/casehub/slots/165/ops/pom.xml`
Expected: Compilation failure — `FailableNodeProvisioner` does not exist

- [ ] **Step 3: Implement FailableNodeProvisioner**

Create `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/FailableNodeProvisioner.java`:

```java
package io.casehub.ops.topology.reconciliation;

import io.casehub.desiredstate.api.*;

import java.time.Duration;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.atomic.AtomicInteger;

public class FailableNodeProvisioner implements NodeProvisioner {

    public final CopyOnWriteArrayList<DesiredNode> provisioned = new CopyOnWriteArrayList<>();
    public final CopyOnWriteArrayList<DesiredNode> deprovisioned = new CopyOnWriteArrayList<>();

    private final Map<String, AtomicInteger> failureBudgets = new ConcurrentHashMap<>();
    private volatile Set<NodeType> handledTypes = Set.of();

    @Override
    public Set<NodeType> handledTypes() {
        return handledTypes;
    }

    @Override
    public Duration resyncInterval() {
        return Duration.ofMinutes(5);
    }

    @Override
    public ProvisionResult provision(DesiredNode node, ProvisionContext context) {
        provisioned.add(node);
        AtomicInteger budget = failureBudgets.get(node.id().value());
        if (budget != null && budget.getAndDecrement() > 0) {
            return new ProvisionResult.Failed("injected failure for " + node.id().value());
        }
        return new ProvisionResult.Success();
    }

    @Override
    public DeprovisionResult deprovision(DesiredNode node, DeprovisionContext context) {
        deprovisioned.add(node);
        return new DeprovisionResult.Success();
    }

    public void failNode(String nodeId, int times) {
        failureBudgets.put(nodeId, new AtomicInteger(times));
    }

    public void setHandledTypes(Set<NodeType> types) {
        this.handledTypes = Set.copyOf(types);
    }

    public void clear() {
        provisioned.clear();
        deprovisioned.clear();
        failureBudgets.clear();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl topology-tests -Preconciliation -Dtest=FailableNodeProvisionerTest -f /Users/mdproctor/claude/casehub/slots/165/ops/pom.xml`
Expected: All 5 tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/165/ops add topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/
git -C /Users/mdproctor/claude/casehub/slots/165/ops commit -m "wip: FailableNodeProvisioner test fixture with per-node failure injection Refs casehubio/casehub-ops#82"
```

### Task 3: ReconciliationTestBase

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/ReconciliationTestBase.java`

**Interfaces:**
- Consumes: `TopologyTestBase.compileSingleGraph()`, `TopologyTestBase.compileLifecycle()`, `FailableNodeProvisioner`
- Produces: `planFromEmpty(graph)`, `executeTransition(plan, tenancyId)`, `buildExecutor(provisioner)`, `assertOrderedBefore(plan, nodeIdA, nodeIdB)`, `buildFaultPolicy()`, `evaluateFault(tenancyId, faultEvent, graph, actual)`

- [ ] **Step 1: Write a minimal failing test using ReconciliationTestBase**

Create `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/ReconciliationTestBaseVerificationTest.java`:

```java
package io.casehub.ops.topology.reconciliation;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("reconciliation")
class ReconciliationTestBaseVerificationTest extends ReconciliationTestBase {

    @Test
    void planFromEmpty_produces_additions_for_all_nodes() throws IOException {
        DesiredStateGraph graph = compileSingleGraph("topologies/single-service/single-node-blog.yaml");
        TransitionPlan plan = planFromEmpty(graph);
        assertThat(plan.additions()).hasSize(3);
        assertThat(plan.removals()).isEmpty();
    }

    @Test
    void assertOrderedBefore_passes_for_correct_ordering() throws IOException {
        DesiredStateGraph graph = compileSingleGraph("topologies/single-service/single-node-blog.yaml");
        TransitionPlan plan = planFromEmpty(graph);
        assertOrderedBefore(plan, "blog-namespace", "ghost");
        assertOrderedBefore(plan, "ghost", "ghost-service");
    }

    @Test
    void executeTransition_records_provisioned_nodes() throws IOException {
        DesiredStateGraph graph = compileSingleGraph("topologies/single-service/single-node-blog.yaml");
        TransitionPlan plan = planFromEmpty(graph);
        TransitionResult result = executeTransition(plan, "test-tenant");
        assertThat(result.outcomes().values())
                .allMatch(o -> o instanceof StepOutcome.Succeeded);
        assertThat(provisioner.provisioned).hasSize(3);
    }
}
```

- [ ] **Step 2: Run test to verify it fails (class not found)**

Run: `mvn --batch-mode test -pl topology-tests -Preconciliation -Dtest=ReconciliationTestBaseVerificationTest -f /Users/mdproctor/claude/casehub/slots/165/ops/pom.xml`
Expected: Compilation failure — `ReconciliationTestBase` does not exist

- [ ] **Step 3: Implement ReconciliationTestBase**

Create `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/ReconciliationTestBase.java`:

```java
package io.casehub.ops.topology.reconciliation;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.runtime.*;
import io.casehub.desiredstate.testing.MockActualStateAdapter;
import io.casehub.ops.topology.compilation.TopologyTestBase;
import org.junit.jupiter.api.BeforeEach;

import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

abstract class ReconciliationTestBase extends TopologyTestBase {

    protected FailableNodeProvisioner provisioner;
    protected TransitionPlanner planner;
    protected MockActualStateAdapter actualAdapter;
    private FaultPolicyEngine faultEngine;

    @BeforeEach
    void setUpReconciliation() {
        provisioner = new FailableNodeProvisioner();
        provisioner.setHandledTypes(allInfraTypes());
        planner = new TransitionPlanner();
        actualAdapter = new MockActualStateAdapter();
        actualAdapter.setHandledTypes(allInfraTypes());
        faultEngine = new FaultPolicyEngine(List.of());
    }

    protected void setFaultPolicies(List<FaultPolicy> policies) {
        faultEngine = new FaultPolicyEngine(policies);
    }

    protected TransitionPlan planFromEmpty(DesiredStateGraph graph) {
        return planner.plan(graph, new ActualState(Map.of()));
    }

    protected TransitionPlan planWithActual(DesiredStateGraph graph, ActualState actual) {
        return planner.plan(graph, actual);
    }

    protected TransitionResult executeTransition(TransitionPlan plan, String tenancyId) {
        SimpleTransitionExecutor executor = new SimpleTransitionExecutor(
                new DefaultNodeProvisionerRouter(List.of(provisioner)),
                new NoOpHumanNodeHandler(),
                new NoOpPendingApprovalHandler(),
                (step, tid) -> new StepOutcome.Succeeded());
        return executor.execute(plan, tenancyId);
    }

    protected List<GraphMutation> evaluateFault(String tenancyId, FaultEvent event,
                                                 DesiredStateGraph graph, ActualState actual) {
        return faultEngine.evaluate(tenancyId, event, graph, actual);
    }

    protected static void assertOrderedBefore(TransitionPlan plan, String beforeId, String afterId) {
        List<OrderedStep> additions = plan.additions();
        int beforeIdx = -1;
        int afterIdx = -1;
        for (int i = 0; i < additions.size(); i++) {
            String id = additions.get(i).node().id().value();
            if (id.equals(beforeId)) beforeIdx = i;
            if (id.equals(afterId)) afterIdx = i;
        }
        assertThat(beforeIdx)
                .as("Expected '%s' (idx=%d) before '%s' (idx=%d) in plan additions",
                        beforeId, beforeIdx, afterId, afterIdx)
                .isGreaterThanOrEqualTo(0);
        assertThat(afterIdx).isGreaterThanOrEqualTo(0);
        assertThat(beforeIdx).isLessThan(afterIdx);
    }

    private static Set<NodeType> allInfraTypes() {
        return Set.of(
                NodeType.of("k8s_namespace"),
                NodeType.of("k8s_deployment"),
                NodeType.of("k8s_service"),
                NodeType.of("k8s-ingress"),
                NodeType.of("load_balancer"),
                NodeType.of("dns_failover"),
                NodeType.of("data_replication"),
                NodeType.of("mesh_control_plane"),
                NodeType.of("sidecar_proxy"),
                NodeType.of("infra-review"));
    }
}
```

Note: `TopologyTestBase` needs its access modifier changed from package-private to `public` (or `protected`) so `ReconciliationTestBase` in the `reconciliation` package can extend it. Use `ide_edit_member` to change `abstract class TopologyTestBase` to `public abstract class TopologyTestBase`.

- [ ] **Step 4: Make TopologyTestBase public**

Change `abstract class TopologyTestBase` to `public abstract class TopologyTestBase` in `topology-tests/src/test/java/io/casehub/ops/topology/compilation/TopologyTestBase.java` (line 30). Also make `compileSingleGraph`, `compileLifecycle`, `compileExemplar`, and `buildTypeRegistry` methods public.

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn --batch-mode test -pl topology-tests -Preconciliation -Dtest=ReconciliationTestBaseVerificationTest -f /Users/mdproctor/claude/casehub/slots/165/ops/pom.xml`
Expected: All 3 tests PASS

- [ ] **Step 6: Verify default build still passes (no reconciliation tests run)**

Run: `mvn --batch-mode test -pl topology-tests -f /Users/mdproctor/claude/casehub/slots/165/ops/pom.xml`
Expected: Only compilation tests run, reconciliation tests excluded

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/165/ops add topology-tests/src/test/java/io/casehub/ops/topology/
git -C /Users/mdproctor/claude/casehub/slots/165/ops commit -m "wip: ReconciliationTestBase with plan/execute helpers + TopologyTestBase visibility Refs casehubio/casehub-ops#82"
```

---

## Batch 2: Reconciliation Test Classes

### Task 4: SingleServiceReconciliationTest

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/SingleServiceReconciliationTest.java`

**Interfaces:**
- Consumes: `ReconciliationTestBase.planFromEmpty()`, `assertOrderedBefore()`, `executeTransition()`, `compileSingleGraph()`

- [ ] **Step 1: Write the test class**

```java
package io.casehub.ops.topology.reconciliation;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("reconciliation")
class SingleServiceReconciliationTest extends ReconciliationTestBase {

    @Test
    void ghostBlog_plansAllNodesFromEmpty() throws IOException {
        DesiredStateGraph graph = compileSingleGraph("topologies/single-service/single-node-blog.yaml");
        TransitionPlan plan = planFromEmpty(graph);
        assertThat(plan.additions()).hasSize(3);
        assertThat(plan.removals()).isEmpty();
    }

    @Test
    void ghostBlog_namespacePrecedesDeployment() throws IOException {
        DesiredStateGraph graph = compileSingleGraph("topologies/single-service/single-node-blog.yaml");
        TransitionPlan plan = planFromEmpty(graph);
        assertOrderedBefore(plan, "blog-namespace", "ghost");
    }

    @Test
    void ghostBlog_deploymentPrecedesService() throws IOException {
        DesiredStateGraph graph = compileSingleGraph("topologies/single-service/single-node-blog.yaml");
        TransitionPlan plan = planFromEmpty(graph);
        assertOrderedBefore(plan, "ghost", "ghost-service");
    }

    @Test
    void ghostBlog_executionSucceeds() throws IOException {
        DesiredStateGraph graph = compileSingleGraph("topologies/single-service/single-node-blog.yaml");
        TransitionPlan plan = planFromEmpty(graph);
        TransitionResult result = executeTransition(plan, "test");
        assertThat(result.outcomes().values())
                .allMatch(o -> o instanceof StepOutcome.Succeeded);
    }

    @Test
    void ghostBlog_driftDetection() throws IOException {
        DesiredStateGraph graph = compileSingleGraph("topologies/single-service/single-node-blog.yaml");
        Map<NodeId, NodeStatus> statuses = Map.of(
                NodeId.of("blog-namespace"), NodeStatus.PRESENT,
                NodeId.of("ghost"), NodeStatus.PRESENT,
                NodeId.of("ghost-service"), NodeStatus.ABSENT);
        TransitionPlan plan = planWithActual(graph, new ActualState(statuses));
        assertThat(plan.additions()).hasSize(1);
        assertThat(plan.additions().get(0).node().id()).isEqualTo(NodeId.of("ghost-service"));
        assertThat(plan.removals()).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests**

Run: `mvn --batch-mode test -pl topology-tests -Preconciliation -Dtest=SingleServiceReconciliationTest -f /Users/mdproctor/claude/casehub/slots/165/ops/pom.xml`
Expected: All 5 tests PASS

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/165/ops add topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/SingleServiceReconciliationTest.java
git -C /Users/mdproctor/claude/casehub/slots/165/ops commit -m "wip: SingleServiceReconciliationTest — plan, ordering, drift Refs casehubio/casehub-ops#82"
```

### Task 5: MultiTierReconciliationTest (with fault escalation)

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/MultiTierReconciliationTest.java`

**Interfaces:**
- Consumes: `ReconciliationTestBase`, `ThresholdFaultPolicy`, `FaultPolicy.addReviewNode()`, `InfraReviewSpec`

- [ ] **Step 1: Write the test class**

```java
package io.casehub.ops.topology.reconciliation;

import io.casehub.desiredstate.api.*;
import io.casehub.ops.api.infra.InfraReviewSpec;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("reconciliation")
class MultiTierReconciliationTest extends ReconciliationTestBase {

    private static final String EXEMPLAR = "topologies/multi-tier/lb-cluster-ecommerce.yaml";

    @Test
    void ecommerce_plansAllNodesFromEmpty() throws IOException {
        CompilationResult.Lifecycle lifecycle = compileLifecycle(EXEMPLAR);
        for (Phase phase : lifecycle.phases()) {
            TransitionPlan plan = planFromEmpty(phase.graph());
            assertThat(plan.additions()).isNotEmpty();
            assertThat(plan.removals()).isEmpty();
        }
    }

    @Test
    void ecommerce_dataPhasePrecedesAppPhase() throws IOException {
        CompilationResult.Lifecycle lifecycle = compileLifecycle(EXEMPLAR);
        List<Phase> phases = lifecycle.phases();
        assertThat(phases).hasSizeGreaterThanOrEqualTo(2);
        assertThat(phases.get(0).id()).isEqualTo("data");
        assertThat(phases.get(1).id()).isEqualTo("application");
    }

    @Test
    void ecommerce_singleNodeDev_linearChainOrdering() throws IOException {
        DesiredStateGraph graph = compileSingleGraph("topologies/multi-tier/single-node-dev.yaml");
        // single-node-dev uses lifecycle — test the first phase individually
        CompilationResult.Lifecycle lifecycle = compileLifecycle("topologies/multi-tier/single-node-dev.yaml");
        // Data phase contains product-db
        Phase dataPhase = lifecycle.phases().get(0);
        assertThat(dataPhase.graph().nodes().containsKey(NodeId.of("product-db"))).isTrue();
    }

    @Test
    void ecommerce_driftOnDataTier_reProvisions() throws IOException {
        CompilationResult.Lifecycle lifecycle = compileLifecycle(EXEMPLAR);
        Phase dataPhase = lifecycle.phases().get(0);
        DesiredStateGraph dataGraph = dataPhase.graph();

        // All data nodes present except product-db which drifted
        Map<NodeId, NodeStatus> statuses = Map.of(
                NodeId.of("product-db"), NodeStatus.DRIFTED);
        TransitionPlan plan = planWithActual(dataGraph, new ActualState(statuses));
        assertThat(plan.additions()).hasSize(1);
        assertThat(plan.additions().get(0).node().id()).isEqualTo(NodeId.of("product-db"));
    }

    @Test
    void ecommerce_faultEscalation_createsReviewNodeAfterThreshold() throws IOException {
        CompilationResult.Lifecycle lifecycle = compileLifecycle(EXEMPLAR);
        Phase dataPhase = lifecycle.phases().get(0);
        DesiredStateGraph graph = dataPhase.graph();
        ActualState actual = new ActualState(Map.of());

        ThresholdFaultPolicy policy = ThresholdFaultPolicy.builder()
                .faultTypes(Set.of(FaultType.PROVISION_FAILED))
                .tier(3, FaultPolicy.addReviewNode(
                        (event, current) -> new InfraReviewSpec(event.node(), event.detail())))
                .build();
        setFaultPolicies(List.of(policy));

        NodeId faultedNode = NodeId.of("product-db");
        FaultEvent event = new FaultEvent(faultedNode, FaultType.PROVISION_FAILED, "connection refused");

        // First two faults: no mutations (below threshold)
        assertThat(evaluateFault("tenant", event, graph, actual)).isEmpty();
        assertThat(evaluateFault("tenant", event, graph, actual)).isEmpty();

        // Third fault: threshold reached — review node added
        List<GraphMutation> mutations = evaluateFault("tenant", event, graph, actual);
        assertThat(mutations).isNotEmpty();
        assertThat(mutations).anyMatch(m -> m instanceof GraphMutation.AddNode addNode
                && addNode.node().type().equals(NodeType.of("infra-review")));
    }

    @Test
    void ecommerce_executionWithFailure_recordsOutcome() throws IOException {
        CompilationResult.Lifecycle lifecycle = compileLifecycle(EXEMPLAR);
        Phase dataPhase = lifecycle.phases().get(0);
        TransitionPlan plan = planFromEmpty(dataPhase.graph());

        provisioner.failNode("product-db", 1);
        TransitionResult result = executeTransition(plan, "test");

        assertThat(result.outcomes().get(NodeId.of("product-db")))
                .isInstanceOf(StepOutcome.Failed.class);
    }
}
```

- [ ] **Step 2: Run tests**

Run: `mvn --batch-mode test -pl topology-tests -Preconciliation -Dtest=MultiTierReconciliationTest -f /Users/mdproctor/claude/casehub/slots/165/ops/pom.xml`
Expected: All 6 tests PASS

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/165/ops add topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/MultiTierReconciliationTest.java
git -C /Users/mdproctor/claude/casehub/slots/165/ops commit -m "wip: MultiTierReconciliationTest — lifecycle phases, drift, fault escalation Refs casehubio/casehub-ops#82"
```

### Task 6: MicroservicesReconciliationTest

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/MicroservicesReconciliationTest.java`

**Interfaces:**
- Consumes: `ReconciliationTestBase`

- [ ] **Step 1: Write the test class**

```java
package io.casehub.ops.topology.reconciliation;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("reconciliation")
class MicroservicesReconciliationTest extends ReconciliationTestBase {

    private static final String EXEMPLAR = "topologies/microservices/ha-multi-az-trading.yaml";

    @Test
    void tradingPlatform_plansAllNodesFromEmpty() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        TransitionPlan plan = planFromEmpty(graph);
        assertThat(plan.additions()).isNotEmpty();
        assertThat(plan.removals()).isEmpty();
    }

    @Test
    void tradingPlatform_namespacePrecedesDeployments() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        TransitionPlan plan = planFromEmpty(graph);
        // Namespace must come before all deployment nodes
        for (OrderedStep step : plan.additions()) {
            if (step.node().type().equals(NodeType.of("k8s_deployment"))) {
                assertOrderedBefore(plan, "trading-ns", step.node().id().value());
            }
        }
    }

    @Test
    void tradingPlatform_forEachStampsMultipleNodes() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        // forEach on market-data with 3 AZs should produce 3 nodes
        long marketDataCount = graph.nodes().keySet().stream()
                .filter(id -> id.value().startsWith("market-data"))
                .count();
        assertThat(marketDataCount).isEqualTo(3);
    }

    @Test
    void tradingPlatform_settlementDependsOnRiskEngine() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        TransitionPlan plan = planFromEmpty(graph);
        assertOrderedBefore(plan, "risk-engine", "settlement");
    }

    @Test
    void tradingPlatform_driftOnSingleAzNode() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        // Set all present except one AZ node
        Map<NodeId, NodeStatus> statuses = new java.util.HashMap<>();
        graph.nodes().keySet().forEach(id -> statuses.put(id, NodeStatus.PRESENT));
        NodeId driftedNode = graph.nodes().keySet().stream()
                .filter(id -> id.value().startsWith("market-data"))
                .findFirst().orElseThrow();
        statuses.put(driftedNode, NodeStatus.ABSENT);

        TransitionPlan plan = planWithActual(graph, new ActualState(statuses));
        assertThat(plan.additions()).hasSize(1);
        assertThat(plan.additions().get(0).node().id()).isEqualTo(driftedNode);
    }

    @Test
    void tradingPlatform_executionSucceeds() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        TransitionPlan plan = planFromEmpty(graph);
        TransitionResult result = executeTransition(plan, "test");
        assertThat(result.outcomes().values())
                .allMatch(o -> o instanceof StepOutcome.Succeeded);
    }
}
```

- [ ] **Step 2: Run tests**

Run: `mvn --batch-mode test -pl topology-tests -Preconciliation -Dtest=MicroservicesReconciliationTest -f /Users/mdproctor/claude/casehub/slots/165/ops/pom.xml`
Expected: All 6 tests PASS

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/165/ops add topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/MicroservicesReconciliationTest.java
git -C /Users/mdproctor/claude/casehub/slots/165/ops commit -m "wip: MicroservicesReconciliationTest — forEach stamps, ordering, drift Refs casehubio/casehub-ops#82"
```

### Task 7: EventDrivenReconciliationTest + SidecarMeshReconciliationTest

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/EventDrivenReconciliationTest.java`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/SidecarMeshReconciliationTest.java`

**Interfaces:**
- Consumes: `ReconciliationTestBase`

- [ ] **Step 1: Write EventDrivenReconciliationTest**

```java
package io.casehub.ops.topology.reconciliation;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("reconciliation")
class EventDrivenReconciliationTest extends ReconciliationTestBase {

    private static final String EXEMPLAR = "topologies/event-driven/lb-cluster-telemetry.yaml";

    @Test
    void telemetry_plansAllNodesFromEmpty() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        TransitionPlan plan = planFromEmpty(graph);
        assertThat(plan.additions()).isNotEmpty();
        assertThat(plan.removals()).isEmpty();
    }

    @Test
    void telemetry_namespacePrecedesBroker() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        TransitionPlan plan = planFromEmpty(graph);
        assertOrderedBefore(plan, "telemetry-ns", "kafka-broker");
    }

    @Test
    void telemetry_brokerPrecedesConsumers() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        TransitionPlan plan = planFromEmpty(graph);
        assertOrderedBefore(plan, "kafka-broker", "telemetry-ingest");
        assertOrderedBefore(plan, "kafka-broker", "telemetry-processor");
    }

    @Test
    void telemetry_processorPrecedesTimeseries() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        TransitionPlan plan = planFromEmpty(graph);
        assertOrderedBefore(plan, "telemetry-processor", "timeseries-db");
    }

    @Test
    void telemetry_driftOnBroker() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        Map<NodeId, NodeStatus> statuses = new java.util.HashMap<>();
        graph.nodes().keySet().forEach(id -> statuses.put(id, NodeStatus.PRESENT));
        statuses.put(NodeId.of("kafka-broker"), NodeStatus.DRIFTED);
        TransitionPlan plan = planWithActual(graph, new ActualState(statuses));
        assertThat(plan.additions()).hasSize(1);
        assertThat(plan.additions().get(0).node().id()).isEqualTo(NodeId.of("kafka-broker"));
    }

    @Test
    void telemetry_executionSucceeds() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        TransitionPlan plan = planFromEmpty(graph);
        TransitionResult result = executeTransition(plan, "test");
        assertThat(result.outcomes().values())
                .allMatch(o -> o instanceof StepOutcome.Succeeded);
    }
}
```

- [ ] **Step 2: Write SidecarMeshReconciliationTest**

```java
package io.casehub.ops.topology.reconciliation;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("reconciliation")
class SidecarMeshReconciliationTest extends ReconciliationTestBase {

    private static final String EXEMPLAR = "topologies/sidecar-mesh/lb-cluster-logistics.yaml";

    @Test
    void logistics_plansAllNodesFromEmpty() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        TransitionPlan plan = planFromEmpty(graph);
        assertThat(plan.additions()).isNotEmpty();
        assertThat(plan.removals()).isEmpty();
    }

    @Test
    void logistics_namespacePrecedesDeployments() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        TransitionPlan plan = planFromEmpty(graph);
        for (OrderedStep step : plan.additions()) {
            if (step.node().type().equals(NodeType.of("k8s_deployment"))) {
                assertOrderedBefore(plan, "logistics-ns", step.node().id().value());
            }
        }
    }

    @Test
    void logistics_meshControlPlanePresentInGraph() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        boolean hasMeshControlPlane = graph.nodes().values().stream()
                .anyMatch(n -> n.type().equals(NodeType.of("mesh_control_plane")));
        assertThat(hasMeshControlPlane).isTrue();
    }

    @Test
    void logistics_driftOnFleetApi() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        Map<NodeId, NodeStatus> statuses = new java.util.HashMap<>();
        graph.nodes().keySet().forEach(id -> statuses.put(id, NodeStatus.PRESENT));
        statuses.put(NodeId.of("fleet-api"), NodeStatus.ABSENT);
        TransitionPlan plan = planWithActual(graph, new ActualState(statuses));
        assertThat(plan.additions()).hasSize(1);
        assertThat(plan.additions().get(0).node().id()).isEqualTo(NodeId.of("fleet-api"));
    }

    @Test
    void logistics_executionSucceeds() throws IOException {
        DesiredStateGraph graph = compileSingleGraph(EXEMPLAR);
        TransitionPlan plan = planFromEmpty(graph);
        TransitionResult result = executeTransition(plan, "test");
        assertThat(result.outcomes().values())
                .allMatch(o -> o instanceof StepOutcome.Succeeded);
    }
}
```

- [ ] **Step 3: Run all reconciliation tests**

Run: `mvn --batch-mode test -pl topology-tests -Preconciliation -f /Users/mdproctor/claude/casehub/slots/165/ops/pom.xml`
Expected: All reconciliation tests PASS

- [ ] **Step 4: Verify default build excludes reconciliation tests**

Run: `mvn --batch-mode test -pl topology-tests -f /Users/mdproctor/claude/casehub/slots/165/ops/pom.xml`
Expected: Only compilation tests run

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/165/ops add topology-tests/src/test/java/io/casehub/ops/topology/reconciliation/
git -C /Users/mdproctor/claude/casehub/slots/165/ops commit -m "wip: EventDriven + SidecarMesh reconciliation tests Refs casehubio/casehub-ops#82"
```

---

## Batch 3: Live K8s Tests

### Task 8: LiveDeploymentTest (profile-gated, skip-guarded)

**Files:**
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/live/LiveDeploymentTest.java`
- Create: `topology-tests/src/test/java/io/casehub/ops/topology/live/KubernetesAvailableCondition.java`

**Interfaces:**
- Consumes: `ReconciliationTestBase`

- [ ] **Step 1: Write KubernetesAvailableCondition**

```java
package io.casehub.ops.topology.live;

import org.junit.jupiter.api.extension.ConditionEvaluationResult;
import org.junit.jupiter.api.extension.ExecutionCondition;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.api.extension.ExtensionContext;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.net.HttpURLConnection;
import java.net.URI;

@Retention(RetentionPolicy.RUNTIME)
@ExtendWith(KubernetesAvailableCondition.KubeCondition.class)
@interface KubernetesAvailable {}

class KubernetesAvailableCondition {
    static class KubeCondition implements ExecutionCondition {
        @Override
        public ConditionEvaluationResult evaluateExecutionCondition(ExtensionContext context) {
            try {
                String kubeHost = System.getenv("KUBERNETES_SERVICE_HOST");
                if (kubeHost == null) {
                    // Try localhost kubectl proxy
                    HttpURLConnection conn = (HttpURLConnection) URI.create("http://localhost:8001/api").toURL().openConnection();
                    conn.setConnectTimeout(2000);
                    conn.setReadTimeout(2000);
                    int code = conn.getResponseCode();
                    if (code == 200) {
                        return ConditionEvaluationResult.enabled("K8s API reachable via kubectl proxy");
                    }
                }
                return ConditionEvaluationResult.disabled("K8s API not reachable");
            } catch (Exception e) {
                return ConditionEvaluationResult.disabled("K8s API not reachable: " + e.getMessage());
            }
        }
    }
}
```

- [ ] **Step 2: Write LiveDeploymentTest**

```java
package io.casehub.ops.topology.live;

import io.casehub.desiredstate.api.*;
import io.casehub.ops.topology.reconciliation.ReconciliationTestBase;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("infra-live")
@KubernetesAvailable
class LiveDeploymentTest extends ReconciliationTestBase {

    @Test
    void singleServiceBlog_plansCorrectly() throws IOException {
        DesiredStateGraph graph = compileSingleGraph("topologies/single-service/single-node-blog.yaml");
        TransitionPlan plan = planFromEmpty(graph);
        assertThat(plan.additions()).hasSize(3);
        assertOrderedBefore(plan, "blog-namespace", "ghost");
    }

    // Future: when K8s backend provisioner is available, add:
    // - Namespace creation verification
    // - Deployment Ready state verification
    // - Service DNS resolution verification
}
```

- [ ] **Step 3: Run default build to confirm live tests don't run**

Run: `mvn --batch-mode test -pl topology-tests -f /Users/mdproctor/claude/casehub/slots/165/ops/pom.xml`
Expected: Only compilation tests run, live tests excluded by tag

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/165/ops add topology-tests/src/test/java/io/casehub/ops/topology/live/
git -C /Users/mdproctor/claude/casehub/slots/165/ops commit -m "wip: LiveDeploymentTest skeleton with K8s availability guard Refs casehubio/casehub-ops#82"
```

---

## Batch 4: Final Verification

### Task 9: Full build verification + cleanup

**Files:**
- No new files

- [ ] **Step 1: Run full default build**

Run: `mvn --batch-mode test -pl topology-tests -f /Users/mdproctor/claude/casehub/slots/165/ops/pom.xml`
Expected: All compilation tests pass, no reconciliation/live tests run

- [ ] **Step 2: Run full reconciliation suite**

Run: `mvn --batch-mode test -pl topology-tests -Preconciliation -f /Users/mdproctor/claude/casehub/slots/165/ops/pom.xml`
Expected: All reconciliation tests pass (FailableNodeProvisionerTest + ReconciliationTestBaseVerificationTest + 5 architecture tests)

- [ ] **Step 3: Verify acceptance criteria checklist**

- `mvn --batch-mode test` passes without K8s (default profile) ✓
- TransitionPlan correctness for 5 core exemplars ✓
- Provision ordering respects dependency edges ✓
- Drift detection triggers re-provisioning ✓
- Fault policy escalation creates review node after 3 failures ✓
- Both test layers gated by Maven profiles ✓

## References

- `wsp-casehub-desiredstate/specs/issue-74-canonical-deployment-topologies/2026-08-31-reconciliation-live-tests-design.md` — implementation design spec
- `wsp-casehub-ops/specs/canonical-deployment-topologies/2026-08-29-canonical-deployment-topologies-design.md` §8 — parent verification strategy
- `runtime/src/main/java/.../TransitionPlanner.java` — plan computation
- `runtime/src/main/java/.../SimpleTransitionExecutor.java` — synchronous execution
- `runtime/src/main/java/.../FaultPolicyEngine.java` — fault evaluation
- `api/src/main/java/.../ThresholdFaultPolicy.java` — threshold-based escalation
- `testing/src/main/java/.../MockActualStateAdapter.java` — controllable actual state
- `testing/src/main/java/.../MockNodeProvisioner.java` — provisioner mock pattern
- `topology-tests/src/test/java/.../TopologyTestBase.java` — compilation test base
- `infra/src/main/java/.../InfraFaultPolicy.java` — ThresholdFaultPolicy usage pattern
- `api/src/main/java/.../InfraReviewSpec.java` — review node spec
- GE-20260516-3a27dc — `combine.self="override"` for surefire profiles
- GE-20260416-ca1c71 — `*IT.java` naming gotcha
- casehubio/casehub-ops#82 — focal issue
