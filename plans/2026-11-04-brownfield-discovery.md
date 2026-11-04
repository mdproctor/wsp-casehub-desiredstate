# Brownfield Discovery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #221 — epic: brownfield discovery — scan environments and generate desired-state YAML
**Issue group:** #222, #223, #224, #225, #226

**Goal:** Enable brownfield adoption by scanning existing environments and generating desired-state YAML for operator review.

**Architecture:** Five components across four existing modules. `DiscoveryCapable` mixin on `BridgeProvider` (bridge/api). `DiscoveryAdapter` SPI with router (api/ + runtime-core/). `YamlGraphSerializer` utility (yaml/runtime). `DiscoveryOrchestrator` composing the pipeline (runtime-core/). CDI/Spring wiring in runtime/, runtime-spring/, bridge/runtime.

**Tech Stack:** Java 21+, Jackson YAML, JUnit 5, Quarkus CDI, Spring Boot auto-config

## Global Constraints

- Pre-release platform — no backward-compatibility concerns.
- All new types in `api/` are pure Java (no framework deps).
- `runtime-core/` uses constructor injection (no CDI, no Spring).
- Follow existing SPI mixin pattern (`ImportCapable`, `PlanCapable`, `SuspendCapable`).
- Follow existing router pattern (`DefaultActualStateAdapterRouter`).
- Tests use `MockBridgeProvider` from `bridge/testing`.

---

## Batch 1: Foundation types — bridge and API layer

### Task 1: DiscoveryCapable mixin and DiscoveredResource (#222)

**Files:**
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/DiscoveryCapable.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/DiscoveredResource.java`

**Interfaces:**
- Consumes: nothing
- Produces: `DiscoveryCapable.discover(String typeId, String tenancyId) → List<DiscoveredResource>`; `DiscoveredResource(String resourceId, String typeId, Map<String, Object> state)`

- [ ] **Step 1: Write DiscoveryCapable interface**

```java
package io.casehub.desiredstate.bridge;

import java.util.List;

public interface DiscoveryCapable {
    List<DiscoveredResource> discover(String typeId, String tenancyId);
}
```

- [ ] **Step 2: Write DiscoveredResource record**

```java
package io.casehub.desiredstate.bridge;

import java.util.Map;

public record DiscoveredResource(
    String resourceId,
    String typeId,
    Map<String, Object> state
) {
    public DiscoveredResource {
        if (state == null) state = Map.of();
    }
}
```

- [ ] **Step 3: Commit**

```bash
git add bridge/api/src/main/java/io/casehub/desiredstate/bridge/DiscoveryCapable.java bridge/api/src/main/java/io/casehub/desiredstate/bridge/DiscoveredResource.java
git commit -m "feat(#222): DiscoveryCapable mixin and DiscoveredResource record"
```

### Task 2: DiscoveryAdapter SPI, DiscoveredNode, and MapNodeSpec (#223)

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/DiscoveryAdapter.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/DiscoveredNode.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/MapNodeSpec.java`

**Interfaces:**
- Consumes: `NodeType`, `NodeSpec` from api/
- Produces: `DiscoveryAdapter.handledTypes() → Set<NodeType>`; `DiscoveryAdapter.discover(Set<NodeType>, String) → List<DiscoveredNode>`; `DiscoveredNode(String resourceId, NodeType type, Map<String, Object> state, Map<String, Object> spec)`; `DiscoveredNode.toNodeSpec() → MapNodeSpec`; `MapNodeSpec(NodeType, Map<String, Object>) implements NodeSpec`

- [ ] **Step 1: Write MapNodeSpec**

```java
package io.casehub.desiredstate.api;

import java.util.Map;

public record MapNodeSpec(NodeType nodeType, Map<String, Object> fields)
        implements NodeSpec {

    public MapNodeSpec {
        if (fields == null) fields = Map.of();
    }

    @Override
    public NodeType nodeType() {
        return nodeType;
    }
}
```

- [ ] **Step 2: Write DiscoveredNode record**

```java
package io.casehub.desiredstate.api;

import java.util.Map;

public record DiscoveredNode(
    String resourceId,
    NodeType type,
    Map<String, Object> state,
    Map<String, Object> spec
) {
    public DiscoveredNode {
        if (state == null) state = Map.of();
        if (spec == null) spec = Map.of();
    }

    public MapNodeSpec toNodeSpec() {
        return new MapNodeSpec(type, spec);
    }
}
```

- [ ] **Step 3: Write DiscoveryAdapter interface**

```java
package io.casehub.desiredstate.api;

import java.util.List;
import java.util.Set;

public interface DiscoveryAdapter {
    Set<NodeType> handledTypes();
    List<DiscoveredNode> discover(Set<NodeType> types, String tenancyId);
}
```

- [ ] **Step 4: Compile and verify**

Run: `mvn --batch-mode compile -pl api/ -am`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/DiscoveryAdapter.java api/src/main/java/io/casehub/desiredstate/api/DiscoveredNode.java api/src/main/java/io/casehub/desiredstate/api/MapNodeSpec.java
git commit -m "feat(#223): DiscoveryAdapter SPI, DiscoveredNode, and MapNodeSpec"
```

---

## Batch 2: Router and bridge adapter

### Task 3: DefaultDiscoveryAdapterRouter (#223)

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DefaultDiscoveryAdapterRouter.java`
- Create: `runtime-core/src/test/java/io/casehub/desiredstate/runtime/DefaultDiscoveryAdapterRouterTest.java`

**Interfaces:**
- Consumes: `DiscoveryAdapter`, `DiscoveredNode`, `NodeType` from api/
- Produces: `DefaultDiscoveryAdapterRouter(Collection<DiscoveryAdapter>)`; `.discover(Set<NodeType>, String) → List<DiscoveredNode>`; `.allHandledTypes() → Set<NodeType>`

- [ ] **Step 1: Write failing test — routes discovery to correct adapter**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.DiscoveredNode;
import io.casehub.desiredstate.api.DiscoveryAdapter;
import io.casehub.desiredstate.api.NodeType;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;

class DefaultDiscoveryAdapterRouterTest {

    private static final NodeType VM = NodeType.of("cloud:vm");
    private static final NodeType STORAGE = NodeType.of("cloud:storage");

    @Test
    void routesDiscoveryToCorrectAdapter() {
        DiscoveryAdapter vmAdapter = new StubDiscoveryAdapter(
            Set.of(VM),
            List.of(new DiscoveredNode("i-1", VM, Map.of("status", "running"), Map.of("name", "vm-1")))
        );
        DiscoveryAdapter storageAdapter = new StubDiscoveryAdapter(
            Set.of(STORAGE),
            List.of(new DiscoveredNode("s-1", STORAGE, Map.of("status", "available"), Map.of("name", "bucket-1")))
        );

        var router = new DefaultDiscoveryAdapterRouter(List.of(vmAdapter, storageAdapter));
        var results = router.discover(Set.of(VM, STORAGE), "tenant-1");

        assertEquals(2, results.size());
        assertTrue(results.stream().anyMatch(n -> n.resourceId().equals("i-1")));
        assertTrue(results.stream().anyMatch(n -> n.resourceId().equals("s-1")));
    }

    @Test
    void allHandledTypesReturnsAllTypes() {
        DiscoveryAdapter adapter = new StubDiscoveryAdapter(
            Set.of(VM, STORAGE), List.of()
        );
        var router = new DefaultDiscoveryAdapterRouter(List.of(adapter));
        assertEquals(Set.of(VM, STORAGE), router.allHandledTypes());
    }

    @Test
    void rejectsDuplicateTypes() {
        DiscoveryAdapter a1 = new StubDiscoveryAdapter(Set.of(VM), List.of());
        DiscoveryAdapter a2 = new StubDiscoveryAdapter(Set.of(VM), List.of());
        assertThrows(IllegalArgumentException.class,
            () -> new DefaultDiscoveryAdapterRouter(List.of(a1, a2)));
    }

    @Test
    void skipsUnhandledTypes() {
        DiscoveryAdapter adapter = new StubDiscoveryAdapter(Set.of(VM), List.of());
        var router = new DefaultDiscoveryAdapterRouter(List.of(adapter));
        var results = router.discover(Set.of(STORAGE), "tenant-1");
        assertTrue(results.isEmpty());
    }

    private record StubDiscoveryAdapter(Set<NodeType> handledTypes,
                                         List<DiscoveredNode> nodes) implements DiscoveryAdapter {
        @Override
        public List<DiscoveredNode> discover(Set<NodeType> types, String tenancyId) {
            return nodes;
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime-core/ -Dtest=DefaultDiscoveryAdapterRouterTest -am`
Expected: COMPILATION_ERROR — `DefaultDiscoveryAdapterRouter` does not exist

- [ ] **Step 3: Write DefaultDiscoveryAdapterRouter**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.DiscoveredNode;
import io.casehub.desiredstate.api.DiscoveryAdapter;
import io.casehub.desiredstate.api.NodeType;

import java.util.ArrayList;
import java.util.Collection;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;

public class DefaultDiscoveryAdapterRouter {

    private final Map<NodeType, DiscoveryAdapter> routing;

    protected DefaultDiscoveryAdapterRouter() {
        this.routing = Map.of();
    }

    public DefaultDiscoveryAdapterRouter(Collection<DiscoveryAdapter> adapters) {
        Map<NodeType, DiscoveryAdapter> table = new LinkedHashMap<>();
        for (DiscoveryAdapter adapter : adapters) {
            for (NodeType type : adapter.handledTypes()) {
                DiscoveryAdapter existing = table.put(type, adapter);
                if (existing != null) {
                    throw new IllegalArgumentException(
                        "NodeType " + type.value() + " claimed by both "
                        + existing.getClass().getName() + " and "
                        + adapter.getClass().getName());
                }
            }
        }
        this.routing = Map.copyOf(table);
    }

    public List<DiscoveredNode> discover(Set<NodeType> types, String tenancyId) {
        List<DiscoveredNode> results = new ArrayList<>();
        for (NodeType type : types) {
            DiscoveryAdapter adapter = routing.get(type);
            if (adapter != null) {
                results.addAll(adapter.discover(Set.of(type), tenancyId));
            }
        }
        return results;
    }

    public Set<NodeType> allHandledTypes() {
        return routing.keySet();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl runtime-core/ -Dtest=DefaultDiscoveryAdapterRouterTest -am`
Expected: BUILD SUCCESS — 4 tests pass

- [ ] **Step 5: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/desiredstate/runtime/DefaultDiscoveryAdapterRouter.java runtime-core/src/test/java/io/casehub/desiredstate/runtime/DefaultDiscoveryAdapterRouterTest.java
git commit -m "feat(#223): DefaultDiscoveryAdapterRouter with routing and duplicate detection"
```

### Task 4: BridgeDiscoveryAdapter (#223)

**Files:**
- Create: `bridge/runtime/src/main/java/io/casehub/desiredstate/bridge/runtime/BridgeDiscoveryAdapter.java`
- Create: `bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/BridgeDiscoveryAdapterTest.java`
- Modify: `bridge/runtime/src/main/java/io/casehub/desiredstate/bridge/runtime/BridgeDiscovery.java`
- Modify: `bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/BridgeDiscoveryTest.java`

**Interfaces:**
- Consumes: `DiscoveryCapable`, `DiscoveredResource` from bridge/api; `DiscoveryAdapter`, `DiscoveredNode`, `NodeType` from api/; `BridgeDescriptor`, `BridgeTypeDescriptor` from bridge/api
- Produces: `BridgeDiscoveryAdapter(BridgeDescriptor, BridgeProvider)` implements `DiscoveryAdapter`

- [ ] **Step 1: Write failing test — translates DiscoveredResource to DiscoveredNode**

```java
package io.casehub.desiredstate.bridge.runtime;

import io.casehub.desiredstate.api.DiscoveredNode;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.bridge.BridgeDescriptor;
import io.casehub.desiredstate.bridge.BridgeLifecycleConfig;
import io.casehub.desiredstate.bridge.BridgeTypeDescriptor;
import io.casehub.desiredstate.bridge.DiscoveredResource;
import io.casehub.desiredstate.bridge.DiscoveryCapable;
import io.casehub.desiredstate.bridge.BridgeProvider;
import io.casehub.desiredstate.bridge.BridgeReadRequest;
import io.casehub.desiredstate.bridge.BridgeRequest;
import io.casehub.desiredstate.bridge.BridgeResult;
import io.casehub.desiredstate.bridge.BridgeStateResult;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;

class BridgeDiscoveryAdapterTest {

    private static final NodeType VM = NodeType.of("cloud:vm");

    @Test
    void discoversResourcesViaDiscoveryCapable() {
        var provider = new StubDiscoveryProvider(
            "test", Set.of("cloud:vm"),
            List.of(new DiscoveredResource("i-1", "cloud:vm",
                Map.of("name", "vm-1", "instance_type", "t3.medium", "status", "running")))
        );
        var typeDesc = new BridgeTypeDescriptor(VM,
            Map.of("running", NodeStatus.PRESENT), NodeStatus.DRIFTED,
            NodeStatus.ABSENT, Map.of(), null);
        var descriptor = new BridgeDescriptor("test", Set.of(VM),
            Map.of("cloud:vm", typeDesc), new BridgeLifecycleConfig(null, null));

        var adapter = new BridgeDiscoveryAdapter(descriptor, provider);

        assertEquals(Set.of(VM), adapter.handledTypes());

        var results = adapter.discover(Set.of(VM), "tenant-1");
        assertEquals(1, results.size());
        DiscoveredNode node = results.getFirst();
        assertEquals("i-1", node.resourceId());
        assertEquals(VM, node.type());
        assertNotNull(node.spec());
        assertEquals("vm-1", node.spec().get("name"));
    }

    @Test
    void skipsUnhandledTypes() {
        var provider = new StubDiscoveryProvider("test", Set.of("cloud:vm"), List.of());
        var descriptor = new BridgeDescriptor("test", Set.of(VM),
            Map.of("cloud:vm", new BridgeTypeDescriptor(VM, Map.of(), null, null, Map.of(), null)),
            new BridgeLifecycleConfig(null, null));

        var adapter = new BridgeDiscoveryAdapter(descriptor, provider);
        var results = adapter.discover(Set.of(NodeType.of("cloud:storage")), "tenant-1");
        assertTrue(results.isEmpty());
    }

    private static class StubDiscoveryProvider implements BridgeProvider, DiscoveryCapable {
        private final String protocol;
        private final Set<String> types;
        private final List<DiscoveredResource> resources;

        StubDiscoveryProvider(String protocol, Set<String> types,
                              List<DiscoveredResource> resources) {
            this.protocol = protocol;
            this.types = types;
            this.resources = resources;
        }

        @Override public String protocolId() { return protocol; }
        @Override public Set<String> supportedTypes() { return types; }
        @Override public BridgeResult apply(BridgeRequest r) { return null; }
        @Override public BridgeResult destroy(BridgeRequest r) { return null; }
        @Override public BridgeStateResult readState(BridgeReadRequest r) { return null; }
        @Override
        public List<DiscoveredResource> discover(String typeId, String tenancyId) {
            return resources.stream()
                .filter(r -> r.typeId().equals(typeId))
                .toList();
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl bridge/runtime -Dtest=BridgeDiscoveryAdapterTest -am`
Expected: COMPILATION_ERROR — `BridgeDiscoveryAdapter` does not exist

- [ ] **Step 3: Write BridgeDiscoveryAdapter**

```java
package io.casehub.desiredstate.bridge.runtime;

import io.casehub.desiredstate.api.DiscoveredNode;
import io.casehub.desiredstate.api.DiscoveryAdapter;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.bridge.BridgeDescriptor;
import io.casehub.desiredstate.bridge.BridgeProvider;
import io.casehub.desiredstate.bridge.BridgeTypeDescriptor;
import io.casehub.desiredstate.bridge.DiscoveredResource;
import io.casehub.desiredstate.bridge.DiscoveryCapable;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Set;

public class BridgeDiscoveryAdapter implements DiscoveryAdapter {

    private final BridgeDescriptor descriptor;
    private final DiscoveryCapable provider;

    public BridgeDiscoveryAdapter(BridgeDescriptor descriptor, BridgeProvider provider) {
        if (!(provider instanceof DiscoveryCapable dc)) {
            throw new IllegalArgumentException("BridgeProvider must implement DiscoveryCapable");
        }
        this.descriptor = descriptor;
        this.provider = dc;
    }

    @Override
    public Set<NodeType> handledTypes() {
        return descriptor.handledTypes();
    }

    @Override
    public List<DiscoveredNode> discover(Set<NodeType> types, String tenancyId) {
        List<DiscoveredNode> nodes = new ArrayList<>();
        for (NodeType type : types) {
            if (!descriptor.handledTypes().contains(type)) continue;
            var typeDesc = descriptor.typeDescriptors().get(type.value());
            if (typeDesc == null) continue;

            List<DiscoveredResource> resources = provider.discover(type.value(), tenancyId);
            for (DiscoveredResource r : resources) {
                Map<String, Object> spec = Map.copyOf(r.state());
                nodes.add(new DiscoveredNode(r.resourceId(), type, r.state(), spec));
            }
        }
        return nodes;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl bridge/runtime -Dtest=BridgeDiscoveryAdapterTest -am`
Expected: BUILD SUCCESS — 2 tests pass

- [ ] **Step 5: Update BridgeDiscovery.discoverAll to register BridgeDiscoveryAdapter**

Modify `BridgeDiscovery.discoverAll()` to conditionally register a `BridgeDiscoveryAdapter` when the provider implements `DiscoveryCapable`:

```java
// Add import at top:
import io.casehub.desiredstate.api.DiscoveryAdapter;
import io.casehub.desiredstate.bridge.DiscoveryCapable;

// Inside the for loop, after adapter bean registration, add:
            if (provider instanceof DiscoveryCapable) {
                var discoveryAdapter = new BridgeDiscoveryAdapter(descriptor, provider);
                beans.add(new BeanRegistration(
                    "bridge-discovery-" + descriptor.protocolId(),
                    DiscoveryAdapter.class, discoveryAdapter));
            }
```

- [ ] **Step 6: Update BridgeDiscoveryTest — verify DiscoveryAdapter registration**

Add a test to `BridgeDiscoveryTest`:

```java
    @Test
    void registersDiscoveryAdapterWhenProviderIsDiscoveryCapable() {
        var provider = new DiscoveryMockBridgeProvider("test-provider", "test_resource");
        var descriptor = BridgeManifestParser.parse(
            getClass().getResourceAsStream("/META-INF/desiredstate/bridges/test-bridge.yaml"));

        var beans = BridgeDiscovery.discoverAll(List.of(descriptor), List.of(provider));

        assertEquals(3, beans.size());
        assertTrue(beans.stream().anyMatch(b -> b.type() == DiscoveryAdapter.class));
    }

    @Test
    void doesNotRegisterDiscoveryAdapterWhenProviderLacksCapability() {
        var provider = new MockBridgeProvider("test-provider", "test_resource", "test_bucket");
        var descriptor = BridgeManifestParser.parse(
            getClass().getResourceAsStream("/META-INF/desiredstate/bridges/test-bridge.yaml"));

        var beans = BridgeDiscovery.discoverAll(List.of(descriptor), List.of(provider));

        assertEquals(2, beans.size());
        assertFalse(beans.stream().anyMatch(b -> b.type() == DiscoveryAdapter.class));
    }
```

Create `DiscoveryMockBridgeProvider` as a simple inner class or test helper that extends `MockBridgeProvider` and implements `DiscoveryCapable`:

```java
    private static class DiscoveryMockBridgeProvider extends MockBridgeProvider
            implements DiscoveryCapable {
        DiscoveryMockBridgeProvider(String protocol, String... types) {
            super(protocol, types);
        }
        @Override
        public List<DiscoveredResource> discover(String typeId, String tenancyId) {
            return List.of();
        }
    }
```

Add required imports:

```java
import io.casehub.desiredstate.api.DiscoveryAdapter;
import io.casehub.desiredstate.bridge.DiscoveredResource;
import io.casehub.desiredstate.bridge.DiscoveryCapable;
```

- [ ] **Step 7: Run all bridge/runtime tests**

Run: `mvn --batch-mode test -pl bridge/runtime -am`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add bridge/runtime/src/main/java/io/casehub/desiredstate/bridge/runtime/BridgeDiscoveryAdapter.java bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/BridgeDiscoveryAdapterTest.java bridge/runtime/src/main/java/io/casehub/desiredstate/bridge/runtime/BridgeDiscovery.java bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/BridgeDiscoveryTest.java
git commit -m "feat(#223): BridgeDiscoveryAdapter and conditional registration in BridgeDiscovery"
```

---

## Batch 3: Orchestrator and serializer

### Task 5: DiscoveryOrchestrator (#225, #226)

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DiscoveryOrchestrator.java`
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DiscoveryRequest.java`
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DiscoveryResult.java`
- Create: `runtime-core/src/test/java/io/casehub/desiredstate/runtime/DiscoveryOrchestratorTest.java`

**Interfaces:**
- Consumes: `DefaultDiscoveryAdapterRouter` from runtime-core/; `DesiredStateGraphFactory`, `ReconciliationStateStore`, `DiscoveredNode`, `MapNodeSpec`, `NodeType`, `NodeId`, `DesiredNode`, `DesiredStateGraph` from api/
- Produces: `DiscoveryOrchestrator(DefaultDiscoveryAdapterRouter, DesiredStateGraphFactory, ReconciliationStateStore)`; `.discover(DiscoveryRequest) → DiscoveryResult`; `.adoptBaseline(String tenancyId, DesiredStateGraph graph)`; `DiscoveryRequest(Set<NodeType> types, String tenancyId, String namespace, String name)`; `DiscoveryResult(List<DiscoveredNode> resources, DesiredStateGraph graph)`

- [ ] **Step 1: Write DiscoveryRequest and DiscoveryResult records**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.NodeType;
import java.util.Set;

public record DiscoveryRequest(
    Set<NodeType> types,
    String tenancyId,
    String namespace,
    String name
) {
    public DiscoveryRequest {
        if (types == null) types = Set.of();
    }
}
```

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.DiscoveredNode;
import java.util.List;

public record DiscoveryResult(
    List<DiscoveredNode> resources,
    DesiredStateGraph graph
) {}
```

- [ ] **Step 2: Write failing test — discovers resources and builds graph**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.DiscoveredNode;
import io.casehub.desiredstate.api.DiscoveryAdapter;
import io.casehub.desiredstate.api.InMemoryReconciliationStateStore;
import io.casehub.desiredstate.api.NodeType;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;

class DiscoveryOrchestratorTest {

    private static final NodeType VM = NodeType.of("cloud:vm");
    private static final NodeType STORAGE = NodeType.of("cloud:storage");

    @Test
    void discoversResourcesAndBuildsGraph() {
        DiscoveryAdapter adapter = new StubDiscoveryAdapter(
            Set.of(VM),
            List.of(
                new DiscoveredNode("i-1", VM, Map.of("status", "running"),
                    Map.of("name", "vm-1", "instance_type", "t3.medium")),
                new DiscoveredNode("i-2", VM, Map.of("status", "running"),
                    Map.of("name", "vm-2", "instance_type", "t3.large"))
            )
        );
        var router = new DefaultDiscoveryAdapterRouter(List.of(adapter));
        var stateStore = new InMemoryReconciliationStateStore();
        var graphFactory = new DefaultDesiredStateGraphFactory();

        var orchestrator = new DiscoveryOrchestrator(router, graphFactory, stateStore);
        var result = orchestrator.discover(
            new DiscoveryRequest(Set.of(VM), "tenant-1", "staging", "infra"));

        assertEquals(2, result.resources().size());
        assertEquals(2, result.graph().nodes().size());
        assertTrue(result.graph().dependencies().isEmpty());
    }

    @Test
    void discoversAllTypesWhenEmptyTypesRequested() {
        DiscoveryAdapter vmAdapter = new StubDiscoveryAdapter(Set.of(VM),
            List.of(new DiscoveredNode("i-1", VM, Map.of(), Map.of("name", "vm-1"))));
        DiscoveryAdapter storageAdapter = new StubDiscoveryAdapter(Set.of(STORAGE),
            List.of(new DiscoveredNode("s-1", STORAGE, Map.of(), Map.of("name", "bucket-1"))));
        var router = new DefaultDiscoveryAdapterRouter(List.of(vmAdapter, storageAdapter));
        var stateStore = new InMemoryReconciliationStateStore();
        var graphFactory = new DefaultDesiredStateGraphFactory();

        var orchestrator = new DiscoveryOrchestrator(router, graphFactory, stateStore);
        var result = orchestrator.discover(
            new DiscoveryRequest(Set.of(), "tenant-1", "staging", "infra"));

        assertEquals(2, result.resources().size());
    }

    @Test
    void adoptBaselineStoresGraphInReconciliationStateStore() {
        DiscoveryAdapter adapter = new StubDiscoveryAdapter(Set.of(VM),
            List.of(new DiscoveredNode("i-1", VM, Map.of(), Map.of("name", "vm-1"))));
        var router = new DefaultDiscoveryAdapterRouter(List.of(adapter));
        var stateStore = new InMemoryReconciliationStateStore();
        var graphFactory = new DefaultDesiredStateGraphFactory();

        var orchestrator = new DiscoveryOrchestrator(router, graphFactory, stateStore);
        var result = orchestrator.discover(
            new DiscoveryRequest(Set.of(VM), "tenant-1", "staging", "infra"));

        assertTrue(stateStore.load("tenant-1").isEmpty());
        orchestrator.adoptBaseline("tenant-1", result.graph());
        assertTrue(stateStore.load("tenant-1").isPresent());
        assertEquals(1, stateStore.load("tenant-1").get().nodes().size());
    }

    @Test
    void sanitizesResourceIdsForNodeIds() {
        DiscoveryAdapter adapter = new StubDiscoveryAdapter(Set.of(VM),
            List.of(new DiscoveredNode("i-ABC_123.test", VM, Map.of(), Map.of("name", "vm-1"))));
        var router = new DefaultDiscoveryAdapterRouter(List.of(adapter));
        var stateStore = new InMemoryReconciliationStateStore();
        var graphFactory = new DefaultDesiredStateGraphFactory();

        var orchestrator = new DiscoveryOrchestrator(router, graphFactory, stateStore);
        var result = orchestrator.discover(
            new DiscoveryRequest(Set.of(VM), "tenant-1", "staging", "infra"));

        var nodeId = result.graph().nodes().keySet().iterator().next();
        assertEquals("i-abc-123-test", nodeId.value());
    }

    private record StubDiscoveryAdapter(Set<NodeType> handledTypes,
                                         List<DiscoveredNode> nodes) implements DiscoveryAdapter {
        @Override
        public List<DiscoveredNode> discover(Set<NodeType> types, String tenancyId) {
            return nodes;
        }
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime-core/ -Dtest=DiscoveryOrchestratorTest -am`
Expected: COMPILATION_ERROR — `DiscoveryOrchestrator` does not exist

- [ ] **Step 4: Write DiscoveryOrchestrator**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.DesiredStateGraphFactory;
import io.casehub.desiredstate.api.DiscoveredNode;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.ReconciliationStateStore;

import java.util.List;
import java.util.Set;

public class DiscoveryOrchestrator {

    private final DefaultDiscoveryAdapterRouter router;
    private final DesiredStateGraphFactory graphFactory;
    private final ReconciliationStateStore stateStore;

    public DiscoveryOrchestrator(DefaultDiscoveryAdapterRouter router,
                                  DesiredStateGraphFactory graphFactory,
                                  ReconciliationStateStore stateStore) {
        this.router = router;
        this.graphFactory = graphFactory;
        this.stateStore = stateStore;
    }

    public DiscoveryResult discover(DiscoveryRequest request) {
        Set<NodeType> types = request.types().isEmpty()
            ? router.allHandledTypes()
            : request.types();
        List<DiscoveredNode> resources = router.discover(types, request.tenancyId());
        DesiredStateGraph graph = buildGraph(resources);
        return new DiscoveryResult(resources, graph);
    }

    public void adoptBaseline(String tenancyId, DesiredStateGraph graph) {
        stateStore.store(tenancyId, graph);
    }

    private DesiredStateGraph buildGraph(List<DiscoveredNode> resources) {
        List<DesiredNode> nodes = resources.stream()
            .map(this::toDesiredNode)
            .toList();
        return graphFactory.of(nodes, List.of());
    }

    private DesiredNode toDesiredNode(DiscoveredNode discovered) {
        return new DesiredNode(
            NodeId.of(sanitizeId(discovered.resourceId())),
            discovered.toNodeSpec(),
            HumanGating.NONE);
    }

    static String sanitizeId(String resourceId) {
        return resourceId.toLowerCase()
            .replaceAll("[^a-z0-9-]", "-")
            .replaceAll("-+", "-")
            .replaceFirst("^-", "")
            .replaceFirst("-$", "");
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl runtime-core/ -Dtest=DiscoveryOrchestratorTest -am`
Expected: BUILD SUCCESS — 4 tests pass

- [ ] **Step 6: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/desiredstate/runtime/DiscoveryOrchestrator.java runtime-core/src/main/java/io/casehub/desiredstate/runtime/DiscoveryRequest.java runtime-core/src/main/java/io/casehub/desiredstate/runtime/DiscoveryResult.java runtime-core/src/test/java/io/casehub/desiredstate/runtime/DiscoveryOrchestratorTest.java
git commit -m "feat(#225,#226): DiscoveryOrchestrator with graph building and adoption baseline"
```

### Task 6: YamlGraphSerializer (#224)

**Files:**
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphSerializer.java`
- Create: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlGraphSerializerTest.java`

**Interfaces:**
- Consumes: `DiscoveredNode`, `NodeType` from api/; `YamlGraph`, `YamlNode`, `YamlDesiredState` from yaml/runtime/model
- Produces: `YamlGraphSerializer.serialize(String namespace, String name, List<DiscoveredNode> resources) → String`

- [ ] **Step 1: Write failing test — serializes discovered resources to YAML**

```java
package io.casehub.desiredstate.yaml;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.desiredstate.api.DiscoveredNode;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.yaml.model.YamlGraph;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class YamlGraphSerializerTest {

    private static final NodeType VM = NodeType.of("cloud:vm");
    private static final NodeType STORAGE = NodeType.of("cloud:storage");

    @Test
    void serializesDiscoveredResourcesToYaml() throws Exception {
        var resources = List.of(
            new DiscoveredNode("i-abc123", VM, Map.of("status", "running"),
                Map.of("name", "web-server-1", "instance_type", "t3.medium")),
            new DiscoveredNode("my-bucket", STORAGE, Map.of("status", "available"),
                Map.of("name", "data-bucket", "region", "us-east-1"))
        );

        String yaml = YamlGraphSerializer.serialize("staging", "discovered-infra", resources);

        assertNotNull(yaml);
        assertTrue(yaml.contains("namespace: \"staging\""));
        assertTrue(yaml.contains("name: \"discovered-infra\""));
        assertTrue(yaml.contains("cloud:vm"));
        assertTrue(yaml.contains("cloud:storage"));
        assertTrue(yaml.contains("web-server-1"));
        assertTrue(yaml.contains("data-bucket"));

        ObjectMapper mapper = new ObjectMapper(new YAMLFactory());
        YamlGraph parsed = mapper.readValue(yaml, YamlGraph.class);
        assertEquals(2, parsed.nodes().size());
    }

    @Test
    void sanitizesResourceIdsToYamlKeys() throws Exception {
        var resources = List.of(
            new DiscoveredNode("i-ABC_123.test", VM, Map.of(), Map.of("name", "vm-1"))
        );

        String yaml = YamlGraphSerializer.serialize("ns", "name", resources);

        assertTrue(yaml.contains("i-abc-123-test:"));
    }

    @Test
    void emptyResourceListProducesValidYaml() throws Exception {
        String yaml = YamlGraphSerializer.serialize("ns", "name", List.of());

        assertNotNull(yaml);
        ObjectMapper mapper = new ObjectMapper(new YAMLFactory());
        YamlGraph parsed = mapper.readValue(yaml, YamlGraph.class);
        assertTrue(parsed.nodes().isEmpty());
    }

    @Test
    void roundTripPreservesSpec() throws Exception {
        var resources = List.of(
            new DiscoveredNode("vm-1", VM, Map.of(),
                Map.of("name", "my-vm", "instance_type", "t3.medium", "ami", "ami-12345"))
        );

        String yaml = YamlGraphSerializer.serialize("staging", "infra", resources);

        ObjectMapper mapper = new ObjectMapper(new YAMLFactory());
        YamlGraph parsed = mapper.readValue(yaml, YamlGraph.class);
        var node = parsed.nodes().get("vm-1");
        assertNotNull(node);
        assertEquals("cloud:vm", node.type());
        assertEquals("my-vm", node.spec().get("name"));
        assertEquals("t3.medium", node.spec().get("instance_type"));
        assertEquals("ami-12345", node.spec().get("ami"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlGraphSerializerTest -am`
Expected: COMPILATION_ERROR — `YamlGraphSerializer` does not exist

- [ ] **Step 3: Write YamlGraphSerializer**

```java
package io.casehub.desiredstate.yaml;

import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import com.fasterxml.jackson.dataformat.yaml.YAMLGenerator;
import io.casehub.desiredstate.api.DiscoveredNode;

import java.io.StringWriter;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public final class YamlGraphSerializer {

    private YamlGraphSerializer() {}

    public static String serialize(String namespace, String name,
                                    List<DiscoveredNode> resources) {
        Map<String, Object> root = new LinkedHashMap<>();

        Map<String, Object> desiredState = new LinkedHashMap<>();
        desiredState.put("namespace", namespace);
        desiredState.put("name", name);
        root.put("desiredState", desiredState);

        Map<String, Object> nodes = new LinkedHashMap<>();
        for (DiscoveredNode resource : resources) {
            String nodeId = sanitizeId(resource.resourceId());
            Map<String, Object> node = new LinkedHashMap<>();
            node.put("type", resource.type().value());
            if (!resource.spec().isEmpty()) {
                node.put("spec", new LinkedHashMap<>(resource.spec()));
            }
            nodes.put(nodeId, node);
        }
        root.put("nodes", nodes);

        try {
            YAMLFactory yamlFactory = YAMLFactory.builder()
                .disable(YAMLGenerator.Feature.WRITE_DOC_START_MARKER)
                .build();
            ObjectMapper mapper = new ObjectMapper(yamlFactory);
            mapper.configure(SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS, false);
            mapper.configure(JsonGenerator.Feature.AUTO_CLOSE_TARGET, false);
            StringWriter writer = new StringWriter();
            mapper.writeValue(writer, root);
            return writer.toString();
        } catch (Exception e) {
            throw new RuntimeException("Failed to serialize YAML", e);
        }
    }

    static String sanitizeId(String resourceId) {
        return resourceId.toLowerCase()
            .replaceAll("[^a-z0-9-]", "-")
            .replaceAll("-+", "-")
            .replaceFirst("^-", "")
            .replaceFirst("-$", "");
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlGraphSerializerTest -am`
Expected: BUILD SUCCESS — 4 tests pass

- [ ] **Step 5: Commit**

```bash
git add yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphSerializer.java yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlGraphSerializerTest.java
git commit -m "feat(#224): YamlGraphSerializer — serialize discovered resources as YAML"
```

---

## Batch 4: CDI/Spring wiring and integration test

### Task 7: CDI and Spring wiring (#225)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/RuntimeBeans.java`
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/CdiDiscoveryAdapterRouter.java`
- Modify: `runtime-spring/src/main/java/io/casehub/desiredstate/runtime/spring/DesiredStateRuntimeAutoConfiguration.java`

**Interfaces:**
- Consumes: `DefaultDiscoveryAdapterRouter`, `DiscoveryOrchestrator` from runtime-core/; `DiscoveryAdapter` from api/; `DesiredStateGraphFactory`, `ReconciliationStateStore` from api/
- Produces: CDI beans for `CdiDiscoveryAdapterRouter` and `DiscoveryOrchestrator`; Spring beans for same

- [ ] **Step 1: Create CdiDiscoveryAdapterRouter**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.DiscoveryAdapter;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

@ApplicationScoped
public class CdiDiscoveryAdapterRouter extends DefaultDiscoveryAdapterRouter {

    @Inject
    public CdiDiscoveryAdapterRouter(Instance<DiscoveryAdapter> adapters) {
        super(adapters.stream().toList());
    }

    CdiDiscoveryAdapterRouter() {
        super();
    }
}
```

- [ ] **Step 2: Add DiscoveryOrchestrator producer to RuntimeBeans**

Add to `RuntimeBeans.java`:

```java
    @Produces
    @ApplicationScoped
    public DiscoveryOrchestrator discoveryOrchestrator(
            CdiDiscoveryAdapterRouter router,
            io.casehub.desiredstate.api.DesiredStateGraphFactory graphFactory,
            ReconciliationStateStore stateStore) {
        return new DiscoveryOrchestrator(router, graphFactory, stateStore);
    }
```

- [ ] **Step 3: Add Spring beans to DesiredStateRuntimeAutoConfiguration**

Add to `DesiredStateRuntimeAutoConfiguration.java`:

```java
    @Bean
    @ConditionalOnMissingBean
    public DefaultDiscoveryAdapterRouter discoveryAdapterRouter(
            List<io.casehub.desiredstate.api.DiscoveryAdapter> adapters) {
        return new DefaultDiscoveryAdapterRouter(adapters);
    }

    @Bean
    @ConditionalOnMissingBean
    public DiscoveryOrchestrator discoveryOrchestrator(
            DefaultDiscoveryAdapterRouter router,
            DesiredStateGraphFactory graphFactory,
            ReconciliationStateStore stateStore) {
        return new DiscoveryOrchestrator(router, graphFactory, stateStore);
    }
```

Add import: `import io.casehub.desiredstate.runtime.DiscoveryOrchestrator;`

- [ ] **Step 4: Compile to verify wiring**

Run: `mvn --batch-mode compile -pl runtime/,runtime-spring/ -am`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/desiredstate/runtime/CdiDiscoveryAdapterRouter.java runtime/src/main/java/io/casehub/desiredstate/runtime/RuntimeBeans.java runtime-spring/src/main/java/io/casehub/desiredstate/runtime/spring/DesiredStateRuntimeAutoConfiguration.java
git commit -m "feat(#225): CDI and Spring wiring for DiscoveryOrchestrator and DiscoveryAdapterRouter"
```

### Task 8: MockCloudProvider discovery + integration test (#222, #225, #226)

**Files:**
- Modify: `examples/bridge-cloud/src/main/java/io/casehub/desiredstate/example/bridge/cloud/MockCloudProvider.java`
- Create: `examples/bridge-cloud/src/test/java/io/casehub/desiredstate/example/bridge/cloud/DiscoveryIntegrationTest.java`

**Interfaces:**
- Consumes: `DiscoveryCapable`, `DiscoveredResource` from bridge/api; `DiscoveryOrchestrator`, `DiscoveryRequest`, `DiscoveryResult` from runtime-core/; `YamlGraphSerializer` from yaml/runtime; `DefaultDiscoveryAdapterRouter` from runtime-core/; `BridgeDiscoveryAdapter` from bridge/runtime
- Produces: full end-to-end integration test

- [ ] **Step 1: Add DiscoveryCapable to MockCloudProvider**

Add `DiscoveryCapable` to the implements clause and add the `discover` method:

```java
// Add to implements clause:
// ... implements BridgeProvider, PlanCapable, ImportCapable, SuspendCapable, DiscoveryCapable

// Add import:
import io.casehub.desiredstate.bridge.DiscoveredResource;
import io.casehub.desiredstate.bridge.DiscoveryCapable;

// Add method:
    @Override
    public List<DiscoveredResource> discover(String typeId, String tenancyId) {
        return resources.entrySet().stream()
            .filter(e -> e.getKey().startsWith(typeId + "::"))
            .map(e -> {
                String resourceId = e.getKey().substring(e.getKey().indexOf("::") + 2);
                return new DiscoveredResource(resourceId, typeId, Map.copyOf(e.getValue()));
            })
            .toList();
    }
```

- [ ] **Step 2: Write integration test — full discovery pipeline**

```java
package io.casehub.desiredstate.example.bridge.cloud;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.desiredstate.api.InMemoryReconciliationStateStore;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.bridge.runtime.BridgeDiscoveryAdapter;
import io.casehub.desiredstate.bridge.runtime.BridgeManifestParser;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.casehub.desiredstate.runtime.DefaultDiscoveryAdapterRouter;
import io.casehub.desiredstate.runtime.DiscoveryOrchestrator;
import io.casehub.desiredstate.runtime.DiscoveryRequest;
import io.casehub.desiredstate.yaml.YamlGraphSerializer;
import io.casehub.desiredstate.yaml.model.YamlGraph;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;

class DiscoveryIntegrationTest {

    private MockCloudProvider provider;
    private DiscoveryOrchestrator orchestrator;
    private InMemoryReconciliationStateStore stateStore;

    @BeforeEach
    void setUp() {
        provider = new MockCloudProvider();
        provider.seedResource("cloud:vm", "i-1",
            Map.of("name", "web-1", "instance_type", "t3.medium", "status", "running"));
        provider.seedResource("cloud:vm", "i-2",
            Map.of("name", "web-2", "instance_type", "t3.large", "status", "running"));
        provider.seedResource("cloud:storage", "bucket-1",
            Map.of("name", "data-bucket", "region", "us-east-1", "status", "available"));

        var descriptor = BridgeManifestParser.parse(
            getClass().getResourceAsStream("/META-INF/desiredstate/bridges/mock-cloud.yaml"));
        var discoveryAdapter = new BridgeDiscoveryAdapter(descriptor, provider);
        var router = new DefaultDiscoveryAdapterRouter(List.of(discoveryAdapter));
        stateStore = new InMemoryReconciliationStateStore();
        var graphFactory = new DefaultDesiredStateGraphFactory();
        orchestrator = new DiscoveryOrchestrator(router, graphFactory, stateStore);
    }

    @Test
    void fullDiscoveryPipeline() throws Exception {
        var result = orchestrator.discover(
            new DiscoveryRequest(Set.of(), "tenant-1", "staging", "cloud-infra"));

        assertEquals(3, result.resources().size());
        assertEquals(3, result.graph().nodes().size());

        String yaml = YamlGraphSerializer.serialize(
            "staging", "cloud-infra", result.resources());
        assertNotNull(yaml);
        assertTrue(yaml.contains("cloud:vm"));
        assertTrue(yaml.contains("cloud:storage"));

        ObjectMapper mapper = new ObjectMapper(new YAMLFactory());
        YamlGraph parsed = mapper.readValue(yaml, YamlGraph.class);
        assertEquals(3, parsed.nodes().size());
    }

    @Test
    void adoptionBaselinePreventsDiffOnFirstReconciliation() {
        var result = orchestrator.discover(
            new DiscoveryRequest(Set.of(), "tenant-1", "staging", "cloud-infra"));

        assertTrue(stateStore.load("tenant-1").isEmpty());
        orchestrator.adoptBaseline("tenant-1", result.graph());
        assertTrue(stateStore.load("tenant-1").isPresent());

        var stored = stateStore.load("tenant-1").get();
        assertEquals(result.graph().nodes().size(), stored.nodes().size());
    }

    @Test
    void filterByType() {
        var result = orchestrator.discover(
            new DiscoveryRequest(Set.of(NodeType.of("cloud:vm")),
                "tenant-1", "staging", "vms-only"));

        assertEquals(2, result.resources().size());
        assertTrue(result.resources().stream()
            .allMatch(r -> r.type().equals(NodeType.of("cloud:vm"))));
    }
}
```

- [ ] **Step 3: Run integration test**

Run: `mvn --batch-mode test -pl examples/bridge-cloud -Dtest=DiscoveryIntegrationTest -am`
Expected: BUILD SUCCESS — 3 tests pass

- [ ] **Step 4: Commit**

```bash
git add examples/bridge-cloud/src/main/java/io/casehub/desiredstate/example/bridge/cloud/MockCloudProvider.java examples/bridge-cloud/src/test/java/io/casehub/desiredstate/example/bridge/cloud/DiscoveryIntegrationTest.java
git commit -m "feat(#222,#225,#226): MockCloudProvider DiscoveryCapable + integration test for full pipeline"
```

---

## Batch 5: Full build verification and CLAUDE.md update

### Task 9: Full build and CLAUDE.md update

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: all prior tasks
- Produces: updated CLAUDE.md with new types documented

- [ ] **Step 1: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules compile and all tests pass

- [ ] **Step 2: Update CLAUDE.md**

Add the new types to the appropriate module sections:

In `bridge/api` row's description, add: `DiscoveryCapable` (mixin — enumerate existing resources by type), `DiscoveredResource` (resourceId, typeId, state).

In `api/` row's description, add: `DiscoveryAdapter` (SPI — graph-less resource enumeration by type), `DiscoveredNode` (resourceId, type, state, spec), `MapNodeSpec` (generic NodeSpec backed by Map).

In `runtime-core/` row's description, add: `DefaultDiscoveryAdapterRouter`, `DiscoveryOrchestrator`, `DiscoveryRequest`, `DiscoveryResult`.

In `runtime/` row's description, add: `CdiDiscoveryAdapterRouter`.

In `yaml/runtime` row's description, add: `YamlGraphSerializer` (serialize discovered resources as YAML).

In `bridge/runtime` row's description, add: `BridgeDiscoveryAdapter`.

Add to Core SPIs table:

| `DiscoveryAdapter` | `handledTypes() → Set<NodeType>`, `discover(Set<NodeType>, String) → List<DiscoveredNode>` | Graph-less resource enumeration by type |

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#221): update CLAUDE.md with brownfield discovery types"
```

## References

- [2026-11-04-brownfield-discovery-design.md] — design spec this plan implements
- `bridge/api/ImportCapable.java` — existing mixin pattern
- `bridge/api/BridgeProvider.java` — mixin host interface
- `bridge/runtime/BridgeDiscovery.java` — bean registration pattern
- `bridge/runtime/BridgeActualStateAdapter.java` — bridge adapter pattern
- `runtime-core/DefaultActualStateAdapterRouter.java` — router pattern
- `runtime/RuntimeBeans.java` — CDI producer pattern
- `runtime-spring/DesiredStateRuntimeAutoConfiguration.java` — Spring auto-config pattern
- `examples/bridge-cloud/MockCloudProvider.java` — mock provider pattern
- `api/NodeSpec.java` — NodeSpec interface for MapNodeSpec
- `api/DesiredNode.java` — DesiredNode record consuming NodeSpec
- GitHub #221, #222, #223, #224, #225, #226
