# Plugin Catalog and Versioning Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #180 — feat: plugin catalog and versioning
**Issue group:** #180
**Blocking dependency:** casehubio/platform#662 — generic catalog SPI + filesystem repository

**Goal:** Enable YAML plugins to load dynamically from the filesystem without recompile or restart,
with semver versioning, dependency declaration, and fail-fast validation.

**Architecture:** Two-layer split — platform provides generic catalog infrastructure (blocked on
platform#662), desiredstate provides domain-specific activation. Dynamic plugins bypass CDI —
`DesiredStatePluginActivator` manages them internally using `ConcurrentHashMap`-backed routers.
Policy engines gain extracted interfaces (`FaultPolicyEvaluator`, `DriftPolicyEvaluator`) to
support both static and dynamic implementations.

**Tech Stack:** Java 21, Quarkus (CDI/ArC), Spring Boot, Jackson YAML, `com.github.zafarkhaja:java-semver`

## Global Constraints

- Pure-Java SPI tier: no Quarkus, no JPA, no reactive types in SPI signatures
- `PluginManifest` name = NodeType string (identity coherence)
- Version: semver (`major.minor.patch`), backward-compatible with int (interpreted as `N.0.0`)
- Single version per plugin per deployment (no multi-version support)
- Dynamic plugins bypass CDI — managed by `DesiredStatePluginActivator` internally
- `ReconciliationStateStore.allTenancyIds()` is a new method (added in this work)
- Platform `PluginRegistry.unregister()` required (blocked on platform#662)

---

## Batch 1: PluginHeader Evolution and Manifest Model

### Task 1: Evolve PluginHeader — semver version, description, requires section

**Files:**
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginHeader.java`
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginParser.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginRequires.java`
- Modify: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/model/PluginParserTest.java`
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/model/PluginParserTest.java`

**Interfaces:**
- Consumes: existing `PluginHeader`, `PluginParser`
- Produces: `PluginHeader(type, version String, resyncInterval, auth, description, requires)`,
  `PluginRequires(runtimeVersionRange, primitives, types)`, backward-compat int→semver parsing

- [ ] **Step 1: Write failing test — semver string version parsing**

```java
@Test
void parseSemverVersion() throws IOException {
    InputStream yaml = toStream("""
        plugin:
          type: test-type
          version: 1.2.0
        spec:
          fields:
            name:
              type: string
              required: true
        actual-state:
          steps:
            - compare-state:
                present-when: "true"
        provisioner:
          provision:
            steps: []
          deprovision:
            steps: []
        cbr:
          features: []
          outcome-signals: {}
        ras:
          situations: []
        """);
    PluginModel model = PluginParser.parse(yaml);
    assertThat(model.header().version()).isEqualTo("1.2.0");
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn --batch-mode test -pl plugin/runtime -Dtest=PluginParserTest#parseSemverVersion`
Expected: compilation error — `version()` returns `int`, not `String`

- [ ] **Step 3: Create PluginRequires record**

```java
package io.casehub.desiredstate.plugin.model;

import java.util.List;
import java.util.Optional;

public record PluginRequires(
    Optional<String> runtimeVersionRange,
    List<String> primitives,
    List<String> types
) {
    public static final PluginRequires NONE = new PluginRequires(
        Optional.empty(), List.of(), List.of());
}
```

- [ ] **Step 4: Evolve PluginHeader — version int→String, add description and requires**

Use `ide_edit_member` on `PluginHeader` to change the record declaration:

```java
public record PluginHeader(
    String type,
    String version,
    String resyncInterval,
    Map<String, PluginAuthStanza> auth,
    String description,
    PluginRequires requires
) {}
```

- [ ] **Step 5: Update PluginParser.parseHeader — handle int and string version, parse requires**

Use `ide_replace_member` on `parseHeader`:

```java
String type = requireText(node, "type", "plugin.type");

JsonNode versionNode = node.get("version");
String version;
if (versionNode != null && versionNode.isNumber()) {
    version = versionNode.asInt() + ".0.0";
} else if (versionNode != null && versionNode.isTextual()) {
    version = versionNode.asText();
} else {
    version = "1.0.0";
}

String resyncInterval = textOrNull(node, "resyncInterval");
String description = textOrNull(node, "description");

Map<String, PluginAuthStanza> auth = new LinkedHashMap<>();
JsonNode authNode = node.get("auth");
if (authNode != null && authNode.isObject()) {
    authNode.fields().forEachRemaining(e -> {
        String credentialRef = requireText(e.getValue(), "credentialRef",
                                           "plugin.auth." + e.getKey() + ".credentialRef");
        auth.put(e.getKey(), new PluginAuthStanza(credentialRef));
    });
}

PluginRequires requires = parseRequires(node.get("requires"));
return new PluginHeader(type, version, resyncInterval, Map.copyOf(auth),
                        description, requires);
```

Add `parseRequires` method:

```java
private static PluginRequires parseRequires(JsonNode node) {
    if (node == null || node.isMissingNode()) {
        return PluginRequires.NONE;
    }
    Optional<String> runtime = Optional.ofNullable(textOrNull(node, "runtime"));
    List<String> primitives = textList(node, "primitives");
    List<String> types = textList(node, "types");
    return new PluginRequires(runtime, primitives, types);
}
```

- [ ] **Step 6: Fix compilation errors in PluginDescriptor and PluginDiscovery**

`PluginDescriptor` stores `int version` — change to `String version`.
`PluginDiscovery.toDescriptor()` calls `plugin.header().version()` — now returns String.
Update all callers.

- [ ] **Step 7: Write test — int version backward compatibility**

```java
@Test
void parseIntVersionAsBackwardCompat() throws IOException {
    InputStream yaml = toStream("""
        plugin:
          type: test-type
          version: 2
        ...
        """);
    PluginModel model = PluginParser.parse(yaml);
    assertThat(model.header().version()).isEqualTo("2.0.0");
}
```

- [ ] **Step 8: Write test — requires section parsing**

```java
@Test
void parseRequiresSection() throws IOException {
    InputStream yaml = toStream("""
        plugin:
          type: test-type
          version: 1.0.0
          requires:
            runtime: ">=1.0.0"
            primitives:
              - rest-call
              - compare-state
            types:
              - human-review
        ...
        """);
    PluginModel model = PluginParser.parse(yaml);
    PluginRequires req = model.header().requires();
    assertThat(req.runtimeVersionRange()).contains(">=1.0.0");
    assertThat(req.primitives()).containsExactly("rest-call", "compare-state");
    assertThat(req.types()).containsExactly("human-review");
}
```

- [ ] **Step 9: Write test — missing requires defaults to NONE**

```java
@Test
void missingRequiresDefaultsToNone() throws IOException {
    // existing plugin YAML without requires section
    PluginModel model = PluginParser.parse(existingMinimalPlugin());
    assertThat(model.header().requires()).isEqualTo(PluginRequires.NONE);
}
```

- [ ] **Step 10: Run all PluginParserTest tests**

Run: `mvn --batch-mode test -pl plugin/runtime -Dtest=PluginParserTest`
Expected: all pass

- [ ] **Step 11: Run full module build to catch any remaining compilation issues**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 12: Commit**

```bash
git add plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginHeader.java
git add plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginParser.java
git add plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginRequires.java
git add plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginDescriptor.java
git add plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginDiscovery.java
git add plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/model/PluginParserTest.java
git commit -m "feat: evolve PluginHeader — semver version, description, requires section (#180)"
```

---

## Batch 2: Policy Interface Extraction and Dynamic Routing

### Task 2: Extract FaultPolicyEvaluator and DriftPolicyEvaluator interfaces

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/FaultPolicyEvaluator.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/DriftPolicyEvaluator.java`
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/FaultPolicyEngine.java` — implements FaultPolicyEvaluator
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DriftPolicyEngine.java` — implements DriftPolicyEvaluator
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java` — field types → interfaces
- Test: existing ReconciliationLoop tests must still pass

**Interfaces:**
- Consumes: existing `FaultPolicyEngine`, `DriftPolicyEngine`, `ReconciliationLoop`
- Produces: `FaultPolicyEvaluator` interface, `DriftPolicyEvaluator` interface. `ReconciliationLoop.Builder` accepts interface types.

- [ ] **Step 1: Create FaultPolicyEvaluator interface**

```java
package io.casehub.desiredstate.api;

import java.util.List;

public interface FaultPolicyEvaluator {
    List<GraphMutation<DesiredNode>> evaluate(String tenancyId, FaultEvent event,
                                              DesiredStateGraph current, ActualState actual);
}
```

- [ ] **Step 2: Create DriftPolicyEvaluator interface**

```java
package io.casehub.desiredstate.api;

public interface DriftPolicyEvaluator {
    DriftDecision evaluate(NodeId nodeId, NodeStatus status,
                           DesiredNode node, DriftContext context);
}
```

- [ ] **Step 3: Make FaultPolicyEngine implement FaultPolicyEvaluator**

Use `ide_edit_member` on the class declaration to add `implements FaultPolicyEvaluator`.

- [ ] **Step 4: Make DriftPolicyEngine implement DriftPolicyEvaluator**

Use `ide_edit_member` on the class declaration to add `implements DriftPolicyEvaluator`.

- [ ] **Step 5: Change ReconciliationLoop field types to interfaces**

Use `ide_find_references` on `FaultPolicyEngine` and `DriftPolicyEngine` to find all usages in ReconciliationLoop. Change field types, constructor parameter types, and Builder parameter types from concrete classes to interfaces.

- [ ] **Step 6: Run full build to verify no breakage**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all existing tests pass unchanged

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/FaultPolicyEvaluator.java
git add api/src/main/java/io/casehub/desiredstate/api/DriftPolicyEvaluator.java
git add runtime-core/ runtime/
git commit -m "refactor: extract FaultPolicyEvaluator and DriftPolicyEvaluator interfaces (#180)"
```

### Task 3: Create DynamicNodeProvisionerRouter and DynamicActualStateAdapterRouter

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DynamicNodeProvisionerRouter.java`
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DynamicActualStateAdapterRouter.java`
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DynamicFaultPolicyEngine.java`
- Create: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DynamicDriftPolicyEngine.java`
- Test: `runtime-core/src/test/java/io/casehub/desiredstate/runtime/DynamicNodeProvisionerRouterTest.java`
- Test: `runtime-core/src/test/java/io/casehub/desiredstate/runtime/DynamicFaultPolicyEngineTest.java`

**Interfaces:**
- Consumes: `NodeProvisionerRouter`, `ActualStateAdapterRouter`, `FaultPolicyEvaluator`, `DriftPolicyEvaluator`
- Produces: Dynamic routers/engines with `register()`/`unregister()` methods. `ConcurrentHashMap`-backed for routers, `CopyOnWriteArrayList`-backed for policy engines.

- [ ] **Step 1: Write failing test — DynamicNodeProvisionerRouter register and route**

```java
@Test
void registerAndRoute() {
    var router = new DynamicNodeProvisionerRouter();
    var provisioner = new MockNodeProvisioner(Set.of(NodeType.of("test-type")));
    router.register(provisioner);

    var node = DesiredNode.of(NodeId.of("n1"), NodeType.of("test-type"), new TestSpec());
    var result = router.provision(node, new ProvisionContext("tenant-1", null, null));
    assertThat(result).isInstanceOf(ProvisionResult.Success.class);
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn --batch-mode test -pl runtime-core -Dtest=DynamicNodeProvisionerRouterTest#registerAndRoute`
Expected: compilation error — class doesn't exist

- [ ] **Step 3: Implement DynamicNodeProvisionerRouter**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.*;
import java.time.Duration;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

public class DynamicNodeProvisionerRouter implements NodeProvisionerRouter {

    private final ConcurrentHashMap<NodeType, NodeProvisioner> routing = new ConcurrentHashMap<>();

    public void register(NodeProvisioner provisioner) {
        for (NodeType type : provisioner.handledTypes()) {
            NodeProvisioner existing = routing.putIfAbsent(type, provisioner);
            if (existing != null) {
                throw new IllegalStateException(
                    "Type collision: " + type + " already handled by " + existing);
            }
        }
    }

    public void unregister(NodeType type) {
        routing.remove(type);
    }

    @Override
    public ProvisionResult provision(DesiredNode node, ProvisionContext ctx) {
        NodeProvisioner p = routing.get(node.type());
        if (p == null) return ProvisionResult.failed("No provisioner for type: " + node.type());
        return p.provision(node, ctx);
    }

    @Override
    public DeprovisionResult deprovision(DesiredNode node, DeprovisionContext ctx) {
        NodeProvisioner p = routing.get(node.type());
        if (p == null) return DeprovisionResult.failed("No provisioner for type: " + node.type());
        return p.deprovision(node, ctx);
    }

    @Override
    public SuspendResult suspend(DesiredNode node, SuspendContext ctx) {
        NodeProvisioner p = routing.get(node.type());
        if (p == null) return SuspendResult.failed("No provisioner for type: " + node.type());
        return p.suspend(node, ctx);
    }

    @Override
    public ResumeResult resume(DesiredNode node, ResumeContext ctx) {
        NodeProvisioner p = routing.get(node.type());
        if (p == null) return ResumeResult.failed("No provisioner for type: " + node.type());
        return p.resume(node, ctx);
    }

    @Override
    public Duration resyncIntervalFor(NodeType type) {
        NodeProvisioner p = routing.get(type);
        return p != null ? p.resyncInterval() : Duration.ofMinutes(5);
    }

    @Override
    public boolean supportsStatefulLifecycle(NodeType type) {
        NodeProvisioner p = routing.get(type);
        return p != null && p.supportsStatefulLifecycle();
    }
}
```

- [ ] **Step 4: Write test — unregister removes routing**

```java
@Test
void unregisterRemovesRouting() {
    var router = new DynamicNodeProvisionerRouter();
    var provisioner = new MockNodeProvisioner(Set.of(NodeType.of("test-type")));
    router.register(provisioner);
    router.unregister(NodeType.of("test-type"));

    var node = DesiredNode.of(NodeId.of("n1"), NodeType.of("test-type"), new TestSpec());
    var result = router.provision(node, new ProvisionContext("tenant-1", null, null));
    assertThat(result).isInstanceOf(ProvisionResult.Failed.class);
}
```

- [ ] **Step 5: Write test — type collision on duplicate register**

```java
@Test
void registerDuplicateTypeThrows() {
    var router = new DynamicNodeProvisionerRouter();
    var p1 = new MockNodeProvisioner(Set.of(NodeType.of("test-type")));
    var p2 = new MockNodeProvisioner(Set.of(NodeType.of("test-type")));
    router.register(p1);
    assertThatThrownBy(() -> router.register(p2))
        .isInstanceOf(IllegalStateException.class)
        .hasMessageContaining("Type collision");
}
```

- [ ] **Step 6: Implement DynamicActualStateAdapterRouter** (same pattern)

- [ ] **Step 7: Implement DynamicFaultPolicyEngine**

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.*;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

public class DynamicFaultPolicyEngine implements FaultPolicyEvaluator {
    private final CopyOnWriteArrayList<FaultPolicy> policies;

    public DynamicFaultPolicyEngine(List<FaultPolicy> staticPolicies) {
        this.policies = new CopyOnWriteArrayList<>(staticPolicies);
    }

    public void addPolicy(FaultPolicy policy) { policies.add(policy); }
    public void removePolicy(FaultPolicy policy) { policies.remove(policy); }

    @Override
    public List<GraphMutation<DesiredNode>> evaluate(String tenancyId, FaultEvent event,
                                                      DesiredStateGraph current, ActualState actual) {
        // same logic as FaultPolicyEngine — iterate policies, merge mutations
        // (copy from FaultPolicyEngine.onFault, adapted to use this.policies)
    }
}
```

- [ ] **Step 8: Implement DynamicDriftPolicyEngine** (same pattern)

- [ ] **Step 9: Write tests for DynamicFaultPolicyEngine — add/remove/evaluate**

- [ ] **Step 10: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 11: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/desiredstate/runtime/Dynamic*.java
git add runtime-core/src/test/java/io/casehub/desiredstate/runtime/Dynamic*.java
git commit -m "feat: dynamic routers and policy engines for runtime plugin management (#180)"
```

---

## Batch 3: ReconciliationStateStore Evolution and ExtensionRegistry

### Task 4: Add allTenancyIds() to ReconciliationStateStore

**Files:**
- Modify: `api/src/main/java/io/casehub/desiredstate/api/ReconciliationStateStore.java`
- Modify: `runtime-core/src/main/java/io/casehub/desiredstate/runtime/InMemoryReconciliationStateStore.java` (in api/)
- Modify: `persistence-jpa/` — JPA implementation
- Modify: `persistence-spring-jpa/` — Spring JPA implementation

**Interfaces:**
- Consumes: existing `ReconciliationStateStore`
- Produces: `allTenancyIds() → Set<String>` on the SPI interface

- [ ] **Step 1: Add default method to ReconciliationStateStore interface**

```java
default Set<String> allTenancyIds() {
    throw new UnsupportedOperationException("allTenancyIds not implemented");
}
```

- [ ] **Step 2: Implement in InMemoryReconciliationStateStore**

```java
@Override
public Set<String> allTenancyIds() {
    return Set.copyOf(snapshots.keySet());
}
```

- [ ] **Step 3: Implement in JPA store** (simple `SELECT DISTINCT tenancy_id` query)

- [ ] **Step 4: Implement in Spring JPA store**

- [ ] **Step 5: Write tests, run full build, commit**

### Task 5: Create ExtensionRegistry

**Files:**
- Create: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/ExtensionRegistry.java`
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/ExtensionRegistryTest.java`

**Interfaces:**
- Consumes: `NodeType`, `DesiredStatePluginManifest` (created in Batch 4)
- Produces: `ExtensionRegistry` — query surface for available types (dynamic + static)

- [ ] **Step 1: Create ExtensionRegistry**

```java
package io.casehub.desiredstate.plugin.api;

import io.casehub.desiredstate.api.NodeType;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class ExtensionRegistry {
    private final Set<NodeType> staticTypes = ConcurrentHashMap.newKeySet();
    private final ConcurrentHashMap<NodeType, Object> dynamicManifests = new ConcurrentHashMap<>();

    public void registerStatic(NodeType type) { staticTypes.add(type); }

    public void registerDynamic(NodeType type, Object manifest) {
        dynamicManifests.put(type, manifest);
    }

    public void unregisterDynamic(NodeType type) { dynamicManifests.remove(type); }

    public Set<NodeType> availableTypes() {
        Set<NodeType> all = new HashSet<>(staticTypes);
        all.addAll(dynamicManifests.keySet());
        return Set.copyOf(all);
    }

    public boolean isTypeAvailable(NodeType type) {
        return staticTypes.contains(type) || dynamicManifests.containsKey(type);
    }

    public Set<NodeType> dynamicTypes() { return Set.copyOf(dynamicManifests.keySet()); }
    public Set<NodeType> staticTypes() { return Set.copyOf(staticTypes); }
}
```

- [ ] **Step 2: Write tests — register static and dynamic, query, unregister**

- [ ] **Step 3: Run build, commit**

---

## Batch 4: DesiredStatePluginActivator and PluginCatalog Integration

**Blocked on:** casehubio/platform#662 (PluginManifest SPI, PluginRepository, PluginCatalog, FilesystemPluginRepository, PluginRegistry.unregister())

### Task 6: Create DesiredStatePluginManifest

**Files:**
- Create: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/DesiredStatePluginManifest.java`

**Interfaces:**
- Consumes: platform `PluginManifest` (from platform#662)
- Produces: `DesiredStatePluginManifest` extending `PluginManifest` with desiredstate-specific fields

### Task 7: Create DesiredStatePluginActivator

**Files:**
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/DesiredStatePluginActivator.java`
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/DesiredStatePluginActivatorTest.java`

**Interfaces:**
- Consumes: `DynamicNodeProvisionerRouter`, `DynamicActualStateAdapterRouter`, `DynamicFaultPolicyEngine`, `DynamicDriftPolicyEngine`, `ExtensionRegistry`, platform `PluginRegistry`, `PluginPackage`
- Produces: 10-step activation cycle with rollback, deactivation with guard

### Task 8: Integrate PluginCatalog — evolve PluginDiscovery, refactor PluginRuntimeBeans

**Files:**
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginDiscovery.java`
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginRuntimeBeans.java`
- Modify: `plugin/spring/src/main/java/io/casehub/desiredstate/plugin/spring/DesiredStatePluginAutoConfiguration.java`

**Interfaces:**
- Consumes: platform `PluginCatalog`, `FilesystemPluginRepository`, `DesiredStatePluginActivator`
- Produces: Refactored wiring that consumes from PluginCatalog (classpath + filesystem sources)

---

## Batch 5: Filesystem Watcher and Reload Endpoint

**Blocked on:** casehubio/platform#662 (FilesystemPluginRepository, directory-watcher)

### Task 9: Wire filesystem watching and reload endpoint

**Files:**
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginReloadEndpoint.java`
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginRuntimeBeans.java`

### Task 10: Update plugin testing framework for dynamic lifecycle

**Files:**
- Modify: `plugin/testing/` — add dynamic loading test support

---

## Batch 6: Example and Documentation Updates

### Task 11: Update monitoring-endpoint example with requires section

**Files:**
- Modify: `examples/plugin-yaml/src/main/resources/META-INF/desiredstate/plugins/monitoring-endpoint.yaml`

### Task 12: Update consumer and contributor guides

**Files:**
- Modify: `docs/guides/consumer-guide.md`
- Modify: `docs/guides/contributor-guide.md`

---

## References

- `specs/issue-180-plugin-catalog-versioning/2026-10-26-plugin-catalog-versioning-design.md` — design spec
- `specs/issue-180-plugin-catalog-versioning/decisions.md` — 11 reviewed decisions
- `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginHeader.java:5` — current header
- `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginParser.java:27` — current parser
- `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginDiscovery.java:19` — current discovery
- `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginRuntimeBeans.java:37` — Quarkus wiring
- `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DefaultNodeProvisionerRouter.java` — immutable router
- `runtime-core/src/main/java/io/casehub/desiredstate/runtime/FaultPolicyEngine.java` — immutable policy engine
- `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DriftPolicyEngine.java` — immutable drift engine
- `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java` — reconciliation loop
- Platform `PluginRegistry` — `yaml-plugin-api/src/main/java/io/casehub/yaml/plugin/api/PluginRegistry.java`
- GitHub #180 — plugin catalog and versioning
- GitHub casehubio/platform#662 — platform catalog infrastructure prerequisite
