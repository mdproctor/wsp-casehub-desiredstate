# Bridge Cloud Example Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #208 — feat: bridge adapter teaching example — mock cloud-resource domain
**Issue group:** #207, #208, #209

**Goal:** Create a self-contained bridge adapter teaching example at `examples/bridge-cloud/` demonstrating `BridgeProvider` + `PlanCapable` + `ImportCapable` with a mock cloud-resource domain and mixed graph (bridge + Java nodes).

**Architecture:** A `MockCloudProvider` implements the bridge SPIs with in-memory state. A `CloudGoalCompiler` produces a mixed graph where bridge-managed infrastructure nodes (VMs, networks, storage) coexist with Java-provisioned application deployment nodes. Tests use the bridge testing TCK for contract validation and runtime-core for integration tests.

**Tech Stack:** Java 21+, JUnit 5, AssertJ, casehub-desiredstate-api, casehub-desiredstate-bridge-api, casehub-desiredstate-runtime-core, casehub-desiredstate-bridge (runtime), casehub-desiredstate-bridge-testing

## Global Constraints

- Pure Java — no Quarkus or Spring dependencies (uses runtime-core constructors directly)
- Module path: `examples/bridge-cloud/`
- Package: `io.casehub.desiredstate.example.bridge.cloud`
- Parent: `casehub-desiredstate-parent` (root pom, not examples aggregator)
- Bridge manifest at `META-INF/desiredstate/bridges/mock-cloud.yaml`

---

## Batch 1: Module scaffold + MockCloudProvider with TCK tests

### Task 1: Module scaffold and MockCloudProvider

**Files:**
- Create: `examples/bridge-cloud/pom.xml`
- Create: `examples/bridge-cloud/src/main/java/io/casehub/desiredstate/example/bridge/cloud/MockCloudProvider.java`
- Create: `examples/bridge-cloud/src/main/resources/META-INF/desiredstate/bridges/mock-cloud.yaml`
- Modify: `pom.xml` (root — add `examples/bridge-cloud` module)
- Test: `examples/bridge-cloud/src/test/java/io/casehub/desiredstate/example/bridge/cloud/MockCloudProviderContractTest.java`
- Test: `examples/bridge-cloud/src/test/java/io/casehub/desiredstate/example/bridge/cloud/MockCloudPlanContractTest.java`
- Test: `examples/bridge-cloud/src/test/java/io/casehub/desiredstate/example/bridge/cloud/MockCloudImportContractTest.java`

**Interfaces:**
- Produces: `MockCloudProvider` implementing `BridgeProvider`, `PlanCapable`, `ImportCapable` with `protocolId()` → `"mock-cloud"`, `supportedTypes()` → `{"cloud:vm", "cloud:network", "cloud:storage"}`

- [ ] **Step 1: Create pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-desiredstate-parent</artifactId>
        <version>0.4-SNAPSHOT</version>
        <relativePath>../../pom.xml</relativePath>
    </parent>

    <artifactId>casehub-desiredstate-example-bridge-cloud</artifactId>

    <name>CaseHub Desired State :: Example :: Bridge Cloud</name>
    <description>Teaching example — mock cloud-resource domain demonstrating BridgeProvider
        with PlanCapable and ImportCapable, mixed graph with bridge and Java nodes.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-bridge-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-runtime-core</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-bridge</artifactId>
        </dependency>

        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-bridge-testing</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-testing</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

- [ ] **Step 2: Add module to root pom.xml**

Add `<module>examples/bridge-cloud</module>` after the `examples/plugin-yaml` line in the root pom.xml `<modules>` section.

- [ ] **Step 3: Create bridge manifest YAML**

Create `examples/bridge-cloud/src/main/resources/META-INF/desiredstate/bridges/mock-cloud.yaml`:

```yaml
bridge:
  protocol: mock-cloud
  types:
    - nodeType: "cloud:vm"
      stateMapping:
        running: PRESENT
        stopped: SUSPENDED
        terminated: ABSENT
        _notFound: ABSENT
        _default: UNKNOWN
      defaultErrorCategory: TRANSIENT
    - nodeType: "cloud:network"
      stateMapping:
        active: PRESENT
        deleted: ABSENT
        _notFound: ABSENT
        _default: UNKNOWN
      defaultErrorCategory: TRANSIENT
    - nodeType: "cloud:storage"
      stateMapping:
        available: PRESENT
        deleted: ABSENT
        _notFound: ABSENT
        _default: UNKNOWN
      defaultErrorCategory: TRANSIENT
```

- [ ] **Step 4: Write TCK contract tests (failing)**

Create `MockCloudProviderContractTest.java`:

```java
package io.casehub.desiredstate.example.bridge.cloud;

import io.casehub.desiredstate.bridge.*;
import io.casehub.desiredstate.bridge.testing.BridgeProviderContractTest;
import java.util.Map;
import java.util.Set;

class MockCloudProviderContractTest extends BridgeProviderContractTest {

    @Override
    protected BridgeProvider createProvider() {
        return new MockCloudProvider();
    }

    @Override
    protected String expectedProtocolId() {
        return "mock-cloud";
    }

    @Override
    protected Set<String> expectedSupportedTypes() {
        return Set.of("cloud:vm", "cloud:network", "cloud:storage");
    }

    @Override
    protected BridgeRequest sampleApplyRequest() {
        return new BridgeRequest("cloud:vm", "vm-1",
            Map.of("instance_type", "t3.micro", "ami", "ami-abc123"),
            "tenant-1");
    }

    @Override
    protected BridgeRequest sampleDestroyRequest() {
        return new BridgeRequest("cloud:vm", "vm-1", Map.of(), "tenant-1");
    }

    @Override
    protected BridgeReadRequest sampleReadRequest() {
        return new BridgeReadRequest("cloud:vm", "vm-1", "tenant-1");
    }
}
```

Create `MockCloudPlanContractTest.java`:

```java
package io.casehub.desiredstate.example.bridge.cloud;

import io.casehub.desiredstate.bridge.*;
import io.casehub.desiredstate.bridge.testing.PlanCapableContractTest;
import java.util.Map;

class MockCloudPlanContractTest extends PlanCapableContractTest {

    @Override
    protected PlanCapable createPlanCapableProvider() {
        return new MockCloudProvider();
    }

    @Override
    protected BridgeRequest samplePlanRequest() {
        return new BridgeRequest("cloud:vm", "vm-1",
            Map.of("instance_type", "t3.micro"),
            "tenant-1");
    }
}
```

Create `MockCloudImportContractTest.java`:

```java
package io.casehub.desiredstate.example.bridge.cloud;

import io.casehub.desiredstate.bridge.*;
import io.casehub.desiredstate.bridge.testing.ImportCapableContractTest;

class MockCloudImportContractTest extends ImportCapableContractTest {

    private final MockCloudProvider provider = new MockCloudProvider();

    @Override
    protected ImportCapable createImportCapableProvider() {
        provider.seedResource("cloud:vm", "i-existing",
            java.util.Map.of("status", "running", "instance_type", "t3.micro"));
        return provider;
    }

    @Override protected String sampleTypeId() { return "cloud:vm"; }
    @Override protected String sampleImportId() { return "i-existing"; }
    @Override protected String sampleTenancyId() { return "tenant-1"; }
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl examples/bridge-cloud -am`
Expected: COMPILATION_ERROR — `MockCloudProvider` does not exist yet.

- [ ] **Step 6: Implement MockCloudProvider**

Create `MockCloudProvider.java`:

```java
package io.casehub.desiredstate.example.bridge.cloud;

import io.casehub.desiredstate.bridge.*;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class MockCloudProvider implements BridgeProvider, PlanCapable, ImportCapable {

    private static final Set<String> TYPES = Set.of("cloud:vm", "cloud:network", "cloud:storage");
    private final Map<String, Map<String, Object>> resources = new ConcurrentHashMap<>();

    @Override
    public String protocolId() {
        return "mock-cloud";
    }

    @Override
    public Set<String> supportedTypes() {
        return TYPES;
    }

    @Override
    public BridgeResult apply(BridgeRequest request) {
        var state = new HashMap<>(request.spec());
        state.put("id", request.resourceId());
        state.put("status", defaultStatus(request.typeId()));
        resources.put(key(request.typeId(), request.resourceId()), state);
        return new BridgeResult.Success(Map.copyOf(state));
    }

    @Override
    public BridgeResult destroy(BridgeRequest request) {
        resources.remove(key(request.typeId(), request.resourceId()));
        return new BridgeResult.Success(Map.of());
    }

    @Override
    public BridgeStateResult readState(BridgeReadRequest request) {
        var state = resources.get(key(request.typeId(), request.resourceId()));
        if (state == null) {
            return new BridgeStateResult.NotFound();
        }
        return new BridgeStateResult.Found(Map.copyOf(state));
    }

    @Override
    public BridgePlan plan(BridgeRequest request) {
        var existing = resources.get(key(request.typeId(), request.resourceId()));
        ChangeAction action;
        if (existing == null) {
            action = ChangeAction.CREATE;
        } else if (specDiffers(existing, request.spec())) {
            action = ChangeAction.UPDATE;
        } else {
            return new BridgePlan(
                UUID.randomUUID().toString(), request.tenancyId(), request.typeId(),
                "No changes", List.of(), null);
        }
        return new BridgePlan(
            UUID.randomUUID().toString(), request.tenancyId(), request.typeId(),
            action + " " + request.resourceId(),
            List.of(new ResourceChange(request.typeId(), request.resourceId(), action)),
            request.spec());
    }

    @Override
    public BridgeResult applyPlan(BridgePlan plan) {
        if (plan.resourceChanges().isEmpty()) {
            return new BridgeResult.Success(Map.of());
        }
        var change = plan.resourceChanges().getFirst();
        if (change.action() == ChangeAction.DELETE) {
            resources.remove(key(change.typeId(), change.resourceId()));
            return new BridgeResult.Success(Map.of());
        }
        var spec = plan.rawPlan() instanceof Map<?,?> m ? toStringMap(m) : Map.<String, Object>of();
        var state = new HashMap<>(spec);
        state.put("id", change.resourceId());
        state.put("status", defaultStatus(change.typeId()));
        resources.put(key(change.typeId(), change.resourceId()), state);
        return new BridgeResult.Success(Map.copyOf(state));
    }

    @Override
    public BridgeImportResult importResource(String typeId, String importId, String tenancyId) {
        var state = resources.get(key(typeId, importId));
        if (state == null) {
            return new BridgeImportResult.Failed("Resource not found: " + importId);
        }
        return new BridgeImportResult.Success(Map.copyOf(state));
    }

    public void seedResource(String typeId, String resourceId, Map<String, Object> state) {
        resources.put(key(typeId, resourceId), new HashMap<>(state));
    }

    public void injectDrift(String typeId, String resourceId, String field, Object value) {
        var state = resources.get(key(typeId, resourceId));
        if (state != null) {
            state.put(field, value);
        }
    }

    private String key(String typeId, String resourceId) {
        return typeId + "::" + resourceId;
    }

    private String defaultStatus(String typeId) {
        return switch (typeId) {
            case "cloud:vm" -> "running";
            case "cloud:network" -> "active";
            case "cloud:storage" -> "available";
            default -> "unknown";
        };
    }

    private boolean specDiffers(Map<String, Object> existing, Map<String, Object> desired) {
        for (var entry : desired.entrySet()) {
            if (!Objects.equals(existing.get(entry.getKey()), entry.getValue())) {
                return true;
            }
        }
        return false;
    }

    @SuppressWarnings("unchecked")
    private Map<String, Object> toStringMap(Map<?, ?> m) {
        var result = new HashMap<String, Object>();
        m.forEach((k, v) -> result.put(String.valueOf(k), v));
        return result;
    }
}
```

- [ ] **Step 7: Run TCK tests to verify they pass**

Run: `mvn --batch-mode test -pl examples/bridge-cloud -am`
Expected: ALL PASS — 3 test classes, ~12 tests total (BridgeProviderContractTest: 8 tests, PlanCapableContractTest: 3 tests, ImportCapableContractTest: 2 tests).

- [ ] **Step 8: Commit**

```bash
git add examples/bridge-cloud/ pom.xml
git commit -m "feat(#208): bridge-cloud example — module scaffold and MockCloudProvider with TCK tests"
```

---

## Batch 2: Blueprint, GoalCompiler, AppDeploymentProvisioner, and integration test

### Task 2: CloudBlueprint and CloudGoalCompiler

**Files:**
- Create: `examples/bridge-cloud/src/main/java/io/casehub/desiredstate/example/bridge/cloud/CloudBlueprint.java`
- Create: `examples/bridge-cloud/src/main/java/io/casehub/desiredstate/example/bridge/cloud/CloudGoalCompiler.java`
- Create: `examples/bridge-cloud/src/main/java/io/casehub/desiredstate/example/bridge/cloud/AppDeploymentSpec.java`
- Create: `examples/bridge-cloud/src/main/java/io/casehub/desiredstate/example/bridge/cloud/CloudNodeTypes.java`
- Test: `examples/bridge-cloud/src/test/java/io/casehub/desiredstate/example/bridge/cloud/CloudGoalCompilerTest.java`

**Interfaces:**
- Consumes: `BridgeNodeSpec` (from bridge-api — wraps `Map<String, Object>`)
- Produces: `CloudBlueprint` (builder API), `CloudGoalCompiler` (compiles blueprint → mixed graph), `AppDeploymentSpec` (NodeSpec for Java-provisioned app nodes), `CloudNodeTypes` (NodeType constants)

- [ ] **Step 1: Write CloudGoalCompiler test (failing)**

```java
package io.casehub.desiredstate.example.bridge.cloud;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.bridge.BridgeNodeSpec;
import io.casehub.desiredstate.runtime.ImmutableDesiredStateGraph;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class CloudGoalCompilerTest {

    private final DesiredStateGraphFactory factory = ImmutableDesiredStateGraph.factory();
    private final CloudGoalCompiler compiler = new CloudGoalCompiler();

    @Test
    void compilesNetworkVmAndAppNodes() {
        var blueprint = CloudBlueprint.builder()
            .network("network-1", Map.of("cidr", "10.0.0.0/16"))
            .vm("vm-web", "network-1", Map.of("instance_type", "t3.micro"))
            .app("app-web", "vm-web", "web-service", "1.0.0")
            .build();

        var result = compiler.compile(blueprint, factory);
        assertThat(result).isInstanceOf(CompilationResult.SingleGraph.class);
        var graph = ((CompilationResult.SingleGraph) result).graph();

        assertThat(graph.nodes()).hasSize(3);
    }

    @Test
    void networkIsRootVmDependsOnNetworkAppDependsOnVm() {
        var blueprint = CloudBlueprint.builder()
            .network("net-1", Map.of("cidr", "10.0.0.0/16"))
            .vm("vm-1", "net-1", Map.of("instance_type", "t3.micro"))
            .app("app-1", "vm-1", "my-app", "1.0.0")
            .build();

        var graph = ((CompilationResult.SingleGraph) compiler.compile(blueprint, factory)).graph();

        assertThat(graph.roots()).extracting(DesiredNode::id)
            .containsExactly(NodeId.of("net-1"));
        assertThat(graph.dependenciesOf(NodeId.of("vm-1")))
            .containsExactly(NodeId.of("net-1"));
        assertThat(graph.dependenciesOf(NodeId.of("app-1")))
            .containsExactly(NodeId.of("vm-1"));
    }

    @Test
    void bridgeNodesUseBridgeNodeSpecJavaNodesUseAppDeploymentSpec() {
        var blueprint = CloudBlueprint.builder()
            .network("net-1", Map.of("cidr", "10.0.0.0/16"))
            .vm("vm-1", "net-1", Map.of("instance_type", "t3.micro"))
            .app("app-1", "vm-1", "my-app", "1.0.0")
            .build();

        var graph = ((CompilationResult.SingleGraph) compiler.compile(blueprint, factory)).graph();

        var netNode = graph.node(NodeId.of("net-1")).orElseThrow();
        assertThat(netNode.spec()).isInstanceOf(BridgeNodeSpec.class);
        assertThat(netNode.type()).isEqualTo(CloudNodeTypes.NETWORK);

        var vmNode = graph.node(NodeId.of("vm-1")).orElseThrow();
        assertThat(vmNode.spec()).isInstanceOf(BridgeNodeSpec.class);
        assertThat(vmNode.type()).isEqualTo(CloudNodeTypes.VM);

        var appNode = graph.node(NodeId.of("app-1")).orElseThrow();
        assertThat(appNode.spec()).isInstanceOf(AppDeploymentSpec.class);
        assertThat(appNode.type()).isEqualTo(CloudNodeTypes.APP_DEPLOYMENT);
    }

    @Test
    void storageIsIndependentNode() {
        var blueprint = CloudBlueprint.builder()
            .storage("bucket-1", Map.of("region", "us-east-1"))
            .build();

        var graph = ((CompilationResult.SingleGraph) compiler.compile(blueprint, factory)).graph();

        assertThat(graph.nodes()).hasSize(1);
        assertThat(graph.roots()).hasSize(1);
        var node = graph.nodes().iterator().next();
        assertThat(node.type()).isEqualTo(CloudNodeTypes.STORAGE);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl examples/bridge-cloud -Dtest=CloudGoalCompilerTest`
Expected: COMPILATION_ERROR — `CloudBlueprint`, `CloudGoalCompiler`, `CloudNodeTypes`, `AppDeploymentSpec` do not exist.

- [ ] **Step 3: Implement CloudNodeTypes**

```java
package io.casehub.desiredstate.example.bridge.cloud;

import io.casehub.desiredstate.api.NodeType;

public final class CloudNodeTypes {
    public static final NodeType VM = NodeType.of("cloud:vm");
    public static final NodeType NETWORK = NodeType.of("cloud:network");
    public static final NodeType STORAGE = NodeType.of("cloud:storage");
    public static final NodeType APP_DEPLOYMENT = NodeType.of("app:deployment");

    private CloudNodeTypes() {}
}
```

- [ ] **Step 4: Implement AppDeploymentSpec**

```java
package io.casehub.desiredstate.example.bridge.cloud;

import io.casehub.desiredstate.api.NodeSpec;

public record AppDeploymentSpec(String appName, String version, String vmNodeId) implements NodeSpec {}
```

- [ ] **Step 5: Implement CloudBlueprint**

```java
package io.casehub.desiredstate.example.bridge.cloud;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public record CloudBlueprint(
    List<NetworkEntry> networks,
    List<VmEntry> vms,
    List<AppEntry> apps,
    List<StorageEntry> storages
) {
    public record NetworkEntry(String id, Map<String, Object> spec) {}
    public record VmEntry(String id, String networkId, Map<String, Object> spec) {}
    public record AppEntry(String id, String vmId, String appName, String version) {}
    public record StorageEntry(String id, Map<String, Object> spec) {}

    public static Builder builder() { return new Builder(); }

    public static class Builder {
        private final List<NetworkEntry> networks = new ArrayList<>();
        private final List<VmEntry> vms = new ArrayList<>();
        private final List<AppEntry> apps = new ArrayList<>();
        private final List<StorageEntry> storages = new ArrayList<>();

        public Builder network(String id, Map<String, Object> spec) {
            networks.add(new NetworkEntry(id, spec));
            return this;
        }

        public Builder vm(String id, String networkId, Map<String, Object> spec) {
            vms.add(new VmEntry(id, networkId, spec));
            return this;
        }

        public Builder app(String id, String vmId, String appName, String version) {
            apps.add(new AppEntry(id, vmId, appName, version));
            return this;
        }

        public Builder storage(String id, Map<String, Object> spec) {
            storages.add(new StorageEntry(id, spec));
            return this;
        }

        public CloudBlueprint build() {
            return new CloudBlueprint(
                List.copyOf(networks), List.copyOf(vms),
                List.copyOf(apps), List.copyOf(storages));
        }
    }
}
```

- [ ] **Step 6: Implement CloudGoalCompiler**

```java
package io.casehub.desiredstate.example.bridge.cloud;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.bridge.BridgeNodeSpec;

import java.util.ArrayList;
import java.util.List;

public class CloudGoalCompiler implements GoalCompiler<CloudBlueprint> {

    @Override
    public CompilationResult compile(CloudBlueprint goals, DesiredStateGraphFactory factory) {
        List<DesiredNode> nodes = new ArrayList<>();
        List<Dependency> dependencies = new ArrayList<>();

        for (var network : goals.networks()) {
            nodes.add(new DesiredNode(
                NodeId.of(network.id()),
                new BridgeNodeSpec(CloudNodeTypes.NETWORK, network.spec()),
                HumanGating.NONE));
        }

        for (var vm : goals.vms()) {
            NodeId vmId = NodeId.of(vm.id());
            nodes.add(new DesiredNode(
                vmId,
                new BridgeNodeSpec(CloudNodeTypes.VM, vm.spec()),
                HumanGating.NONE));
            dependencies.add(new Dependency(vmId, NodeId.of(vm.networkId())));
        }

        for (var app : goals.apps()) {
            NodeId appId = NodeId.of(app.id());
            nodes.add(new DesiredNode(
                appId,
                new AppDeploymentSpec(app.appName(), app.version(), app.vmId()),
                HumanGating.NONE));
            dependencies.add(new Dependency(appId, NodeId.of(app.vmId())));
        }

        for (var storage : goals.storages()) {
            nodes.add(new DesiredNode(
                NodeId.of(storage.id()),
                new BridgeNodeSpec(CloudNodeTypes.STORAGE, storage.spec()),
                HumanGating.NONE));
        }

        return CompilationResult.single(factory.of(nodes, dependencies));
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl examples/bridge-cloud -Dtest=CloudGoalCompilerTest`
Expected: ALL PASS (4 tests).

- [ ] **Step 8: Commit**

```bash
git add examples/bridge-cloud/
git commit -m "feat(#208): CloudBlueprint, CloudGoalCompiler, and mixed graph structure"
```

### Task 3: AppDeploymentProvisioner and integration test

**Files:**
- Create: `examples/bridge-cloud/src/main/java/io/casehub/desiredstate/example/bridge/cloud/AppDeploymentProvisioner.java`
- Create: `examples/bridge-cloud/src/main/java/io/casehub/desiredstate/example/bridge/cloud/CloudWorld.java`
- Test: `examples/bridge-cloud/src/test/java/io/casehub/desiredstate/example/bridge/cloud/CloudBridgeIntegrationTest.java`

**Interfaces:**
- Consumes: `CloudGoalCompiler`, `CloudBlueprint`, `MockCloudProvider`, `AppDeploymentSpec`, `CloudNodeTypes`
- Produces: `AppDeploymentProvisioner` (NodeProvisioner for app:deployment), `CloudWorld` (in-memory app state)

- [ ] **Step 1: Write integration test (failing)**

```java
package io.casehub.desiredstate.example.bridge.cloud;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.bridge.BridgeNodeSpec;
import io.casehub.desiredstate.runtime.*;
import io.casehub.desiredstate.bridge.runtime.BridgeDiscovery;
import io.casehub.desiredstate.bridge.runtime.BridgeManifestParser;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;

class CloudBridgeIntegrationTest {

    private MockCloudProvider cloudProvider;
    private AppDeploymentProvisioner appProvisioner;
    private CloudWorld world;
    private TransitionPlanner planner;
    private DesiredStateGraphFactory graphFactory;

    @BeforeEach
    void setUp() {
        cloudProvider = new MockCloudProvider();
        world = new CloudWorld();
        appProvisioner = new AppDeploymentProvisioner(world);
        graphFactory = ImmutableDesiredStateGraph.factory();
        planner = new TransitionPlanner();
    }

    @Test
    void mixedGraphProvisionsBridgeAndJavaNodes() {
        var blueprint = CloudBlueprint.builder()
            .network("net-1", Map.of("cidr", "10.0.0.0/16"))
            .vm("vm-1", "net-1", Map.of("instance_type", "t3.micro"))
            .app("app-1", "vm-1", "my-service", "1.0.0")
            .storage("bucket-1", Map.of("region", "us-east-1"))
            .build();

        var compiler = new CloudGoalCompiler();
        var graph = ((CompilationResult.SingleGraph) compiler.compile(blueprint, graphFactory)).graph();

        var descriptor = BridgeManifestParser.parse(
            getClass().getResourceAsStream("/META-INF/desiredstate/bridges/mock-cloud.yaml"));
        var bridgeBeans = BridgeDiscovery.discoverAll(List.of(descriptor), List.of(cloudProvider));

        var bridgeProvisioner = (NodeProvisioner) bridgeBeans.stream()
            .filter(b -> b.type() == NodeProvisioner.class)
            .findFirst().orElseThrow().instance();

        var router = new DefaultNodeProvisionerRouter(List.of(bridgeProvisioner, appProvisioner));

        var actual = ActualState.empty();
        var plan = planner.plan(graph, actual);

        var executor = new SimpleTransitionExecutor(
            new NodeStepExecutor(router, new NoOpHumanNodeHandler(), new NoOpPendingApprovalHandler()));
        var result = executor.execute(plan, "tenant-1");

        assertThat(result.outcomes().values())
            .allMatch(o -> o instanceof StepOutcome.Succeeded || o instanceof StepOutcome.AlreadyConverged);
        assertThat(world.deployments()).containsKey("app-1");
        assertThat(world.deployments().get("app-1").appName()).isEqualTo("my-service");
    }

    @Test
    void importSeedsStateFromExistingCloudResource() {
        cloudProvider.seedResource("cloud:vm", "i-existing",
            Map.of("status", "running", "instance_type", "t3.large"));

        var result = cloudProvider.importResource("cloud:vm", "i-existing", "tenant-1");

        assertThat(result).isInstanceOf(io.casehub.desiredstate.bridge.BridgeImportResult.Success.class);
        var success = (io.casehub.desiredstate.bridge.BridgeImportResult.Success) result;
        assertThat(success.state()).containsEntry("instance_type", "t3.large");
    }

    @Test
    void planDetectsCreateUpdateAndNoChange() {
        var createReq = new io.casehub.desiredstate.bridge.BridgeRequest(
            "cloud:vm", "vm-new", Map.of("instance_type", "t3.micro"), "tenant-1");
        var createPlan = cloudProvider.plan(createReq);
        assertThat(createPlan.resourceChanges()).hasSize(1);
        assertThat(createPlan.resourceChanges().getFirst().action())
            .isEqualTo(io.casehub.desiredstate.bridge.ChangeAction.CREATE);

        cloudProvider.apply(createReq);

        var noChangePlan = cloudProvider.plan(createReq);
        assertThat(noChangePlan.resourceChanges()).isEmpty();

        var updateReq = new io.casehub.desiredstate.bridge.BridgeRequest(
            "cloud:vm", "vm-new", Map.of("instance_type", "t3.large"), "tenant-1");
        var updatePlan = cloudProvider.plan(updateReq);
        assertThat(updatePlan.resourceChanges()).hasSize(1);
        assertThat(updatePlan.resourceChanges().getFirst().action())
            .isEqualTo(io.casehub.desiredstate.bridge.ChangeAction.UPDATE);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl examples/bridge-cloud -Dtest=CloudBridgeIntegrationTest`
Expected: COMPILATION_ERROR — `AppDeploymentProvisioner`, `CloudWorld` do not exist.

- [ ] **Step 3: Implement CloudWorld**

```java
package io.casehub.desiredstate.example.bridge.cloud;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class CloudWorld {

    private final Map<String, AppDeploymentSpec> deployments = new ConcurrentHashMap<>();

    public void deploy(String nodeId, AppDeploymentSpec spec) {
        deployments.put(nodeId, spec);
    }

    public void undeploy(String nodeId) {
        deployments.remove(nodeId);
    }

    public Map<String, AppDeploymentSpec> deployments() {
        return Map.copyOf(deployments);
    }
}
```

- [ ] **Step 4: Implement AppDeploymentProvisioner**

```java
package io.casehub.desiredstate.example.bridge.cloud;

import io.casehub.desiredstate.api.*;

import java.util.Set;

public class AppDeploymentProvisioner implements NodeProvisioner {

    private final CloudWorld world;

    public AppDeploymentProvisioner(CloudWorld world) {
        this.world = world;
    }

    @Override
    public Set<NodeType> handledTypes() {
        return Set.of(CloudNodeTypes.APP_DEPLOYMENT);
    }

    @Override
    public ProvisionResult provision(DesiredNode node, ProvisionContext context) {
        if (!(node.spec() instanceof AppDeploymentSpec spec)) {
            return new ProvisionResult.Failed("Expected AppDeploymentSpec");
        }
        world.deploy(node.id().value(), spec);
        return new ProvisionResult.Success(java.util.Map.of(
            "appName", spec.appName(), "version", spec.version()));
    }

    @Override
    public DeprovisionResult deprovision(DesiredNode node, DeprovisionContext context) {
        world.undeploy(node.id().value());
        return new DeprovisionResult.Success();
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl examples/bridge-cloud`
Expected: ALL PASS — TCK tests (3 classes) + GoalCompiler tests + integration tests.

- [ ] **Step 6: Run full module build**

Run: `mvn --batch-mode install -pl examples/bridge-cloud -am`
Expected: BUILD SUCCESS.

- [ ] **Step 7: Commit**

```bash
git add examples/bridge-cloud/
git commit -m "feat(#208): AppDeploymentProvisioner, CloudWorld, and mixed-graph integration test"
```

## References

- [2026-11-03-bridge-cloud-example-design.md] — design spec this plan implements
- `bridge/api/src/.../BridgeProvider.java` — core SPI interface
- `bridge/api/src/.../PlanCapable.java` — plan mixin
- `bridge/api/src/.../ImportCapable.java` — import mixin
- `bridge/testing/src/.../BridgeProviderContractTest.java` — TCK base class
- `bridge/testing/src/.../MockBridgeProvider.java` — mock pattern reference
- `bridge/runtime/src/.../BridgeDiscovery.java` — discovery for integration test
- `examples/dungeon/` — GoalCompiler + blueprint pattern reference
- GitHub #208 — feature issue
