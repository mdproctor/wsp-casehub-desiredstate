# Spring Discovery Classes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #155 — Spring: yaml/annotations/plugin/ts-dsl Spring modules could use generators
**Issue group:** #155, #143

**Goal:** Extract domain logic from 4 hand-written Spring auto-configs into framework-neutral discovery classes, reducing each auto-config to ~15 lines of glue and enabling future generation.

**Architecture:** Each surface's runtime module gains a discovery class that calls existing factories and returns `List<BeanRegistration>`. Spring auto-configs iterate the registrations. Shared Jandex/resource utilities are extracted into `SpringJandexSupport` in `runtime-spring/`. A follow-up issue covers generating the trivial glue code.

**Tech Stack:** Java 21, Spring Boot auto-configuration, Jandex, existing factory classes (GoalCompilerFactory, YamlGoalCompilerFactory, TsGoalCompilerFactory, FaultPolicyFactory, PluginParser)

## Global Constraints

- No behavior change — all existing Spring auto-config functionality must be preserved
- No new dependencies between modules beyond what already exists
- Discovery classes must be framework-neutral (no Spring, no CDI imports)
- `BeanRegistration` goes in `api/` — same module as `GoalCompiler`, `ThresholdFaultPolicy`

---

## Batch 1: Foundation — BeanRegistration + SpringJandexSupport

### Task 1: BeanRegistration record in api module

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/BeanRegistration.java`
- Test: `api/src/test/java/io/casehub/desiredstate/api/BeanRegistrationTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `BeanRegistration(String name, Class<?> type, Object instance)` — used by all discovery classes and Spring auto-configs

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.desiredstate.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class BeanRegistrationTest {

    @Test
    void carriesNameTypeAndInstance() {
        var reg = new BeanRegistration("myBean", String.class, "hello");
        assertThat(reg.name()).isEqualTo("myBean");
        assertThat(reg.type()).isEqualTo(String.class);
        assertThat(reg.instance()).isEqualTo("hello");
    }

    @Test
    void rejectsNullName() {
        assertThatThrownBy(() -> new BeanRegistration(null, String.class, "hello"))
            .isInstanceOf(NullPointerException.class);
    }

    @Test
    void rejectsNullType() {
        assertThatThrownBy(() -> new BeanRegistration("myBean", null, "hello"))
            .isInstanceOf(NullPointerException.class);
    }

    @Test
    void rejectsNullInstance() {
        assertThatThrownBy(() -> new BeanRegistration("myBean", String.class, null))
            .isInstanceOf(NullPointerException.class);
    }
}
```

Use `ide_create_file` to create the test file.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -pl api -Dtest=BeanRegistrationTest test`
Expected: FAIL — `BeanRegistration` class does not exist

- [ ] **Step 3: Write minimal implementation**

Use `ide_create_file`:

```java
package io.casehub.desiredstate.api;

import java.util.Objects;

public record BeanRegistration(String name, Class<?> type, Object instance) {
    public BeanRegistration {
        Objects.requireNonNull(name, "name");
        Objects.requireNonNull(type, "type");
        Objects.requireNonNull(instance, "instance");
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -pl api -Dtest=BeanRegistrationTest test`
Expected: PASS — 4 tests

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/BeanRegistration.java api/src/test/java/io/casehub/desiredstate/api/BeanRegistrationTest.java
git commit -m "feat(#155): BeanRegistration record — shared return type for discovery classes"
```

### Task 2: SpringJandexSupport utility in runtime-spring

**Files:**
- Create: `runtime-spring/src/main/java/io/casehub/desiredstate/runtime/spring/SpringJandexSupport.java`
- Test: `runtime-spring/src/test/java/io/casehub/desiredstate/runtime/spring/SpringJandexSupportTest.java`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `SpringJandexSupport.loadCompositeIndex() → IndexView` — loads all `META-INF/jandex.idx` from classpath
  - `SpringJandexSupport.scanNodeTypes(IndexView) → Map<String, String>` — scans `@NodeTypeId` annotated `NodeSpec` implementors

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.desiredstate.runtime.spring;

import org.jboss.jandex.IndexView;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class SpringJandexSupportTest {

    @Test
    void loadCompositeIndex_returnsNonNullIndex() {
        IndexView index = SpringJandexSupport.loadCompositeIndex();
        assertThat(index).isNotNull();
    }

    @Test
    void scanNodeTypes_returnsEmptyMapForEmptyIndex() {
        IndexView index = org.jboss.jandex.Index.of();
        Map<String, String> types = SpringJandexSupport.scanNodeTypes(index);
        assertThat(types).isEmpty();
    }
}
```

Use `ide_create_file`.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -pl runtime-spring -Dtest=SpringJandexSupportTest test`
Expected: FAIL — class does not exist

- [ ] **Step 3: Write minimal implementation**

Extract the identical code from `DesiredStateYamlAutoConfiguration.loadCompositeJandexIndex()` and `scanNodeTypes()`:

```java
package io.casehub.desiredstate.runtime.spring;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeTypeId;
import org.jboss.jandex.AnnotationInstance;
import org.jboss.jandex.ClassInfo;
import org.jboss.jandex.CompositeIndex;
import org.jboss.jandex.DotName;
import org.jboss.jandex.Index;
import org.jboss.jandex.IndexReader;
import org.jboss.jandex.IndexView;

import java.io.IOException;
import java.io.InputStream;
import java.net.URL;
import java.util.ArrayList;
import java.util.Enumeration;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public final class SpringJandexSupport {

    private static final DotName NODE_SPEC = DotName.createSimple(NodeSpec.class.getName());
    private static final DotName NODE_TYPE_ID = DotName.createSimple(NodeTypeId.class.getName());

    private SpringJandexSupport() {}

    public static IndexView loadCompositeIndex() {
        List<IndexView> indexes = new ArrayList<>();
        try {
            Enumeration<URL> resources = Thread.currentThread()
                    .getContextClassLoader()
                    .getResources("META-INF/jandex.idx");
            while (resources.hasMoreElements()) {
                try (InputStream is = resources.nextElement().openStream()) {
                    indexes.add(new IndexReader(is).read());
                }
            }
        } catch (IOException e) {
            throw new IllegalStateException("Failed to load Jandex indexes", e);
        }
        return indexes.isEmpty() ? Index.of() : CompositeIndex.create(indexes);
    }

    public static Map<String, String> scanNodeTypes(IndexView index) {
        Map<String, String> registry = new HashMap<>();
        for (AnnotationInstance ann : index.getAnnotations(NODE_TYPE_ID)) {
            if (ann.target().kind() == org.jboss.jandex.AnnotationTarget.Kind.CLASS) {
                ClassInfo cls = ann.target().asClass();
                if (index.getAllKnownImplementors(NODE_SPEC).contains(cls)) {
                    String typeId = ann.value().asString();
                    registry.put(typeId, cls.name().toString());
                }
            }
        }
        return registry;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -pl runtime-spring -Dtest=SpringJandexSupportTest test`
Expected: PASS — 2 tests

- [ ] **Step 5: Commit**

```bash
git add runtime-spring/src/main/java/io/casehub/desiredstate/runtime/spring/SpringJandexSupport.java runtime-spring/src/test/java/io/casehub/desiredstate/runtime/spring/SpringJandexSupportTest.java
git commit -m "feat(#155): SpringJandexSupport — shared Jandex loading and NodeType scanning"
```

## Batch 2: Discovery classes — annotations + plugin

### Task 3: AnnotationsDiscovery in annotations/runtime

**Files:**
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/AnnotationsDiscovery.java`
- Test: `annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/AnnotationsDiscoveryTest.java`

**Interfaces:**
- Consumes: `BeanRegistration` from api, `DescriptorScanner.scanGraphs(IndexView)`, `DescriptorScanner.scanFaultPolicies(IndexView)`, `GoalCompilerFactory.create(GraphDescriptor)`, `FaultPolicyFactory.create(FaultPolicyDescriptor, String)`
- Produces: `AnnotationsDiscovery.discover(IndexView index) → List<BeanRegistration>` — returns GoalCompiler and ThresholdFaultPolicy beans

- [ ] **Step 1: Write the failing test**

Build a Jandex index from the pipeline-annotated example's classes (they use `@DesiredState`, `@Node`, `@FaultPolicyDef`). Use `org.jboss.jandex.Index.of(Class<?>...)` to create an in-memory index.

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.api.BeanRegistration;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.ThresholdFaultPolicy;
import org.jboss.jandex.Index;
import org.jboss.jandex.IndexView;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class AnnotationsDiscoveryTest {

    @Test
    void discover_returnsEmptyForEmptyIndex() {
        IndexView index = Index.of();
        List<BeanRegistration> beans = new AnnotationsDiscovery().discover(index);
        assertThat(beans).isEmpty();
    }

    @Test
    void discover_returnsGoalCompilerForAnnotatedGraph() throws Exception {
        // Index a class annotated with @DesiredState and @Node
        // Use a minimal test fixture class defined in this test file
        IndexView index = Index.of(TestAnnotatedGraph.class, TestNodeSpec.class);
        List<BeanRegistration> beans = new AnnotationsDiscovery().discover(index);
        assertThat(beans).anyMatch(b -> b.type().equals(GoalCompiler.class));
    }
}
```

The test fixture classes (`TestAnnotatedGraph`, `TestNodeSpec`) should be minimal `@DesiredState` + `@Node` annotated classes in the test package, following the same pattern as the pipeline-annotated example. Create them as inner records/classes or separate test fixtures.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -pl annotations/runtime -Dtest=AnnotationsDiscoveryTest test`
Expected: FAIL — `AnnotationsDiscovery` class does not exist

- [ ] **Step 3: Write minimal implementation**

Extract the domain logic from `DesiredStateAnnotationsAutoConfiguration.afterSingletonsInstantiated()`:

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.annotations.core.DescriptorScanner;
import io.casehub.desiredstate.api.BeanRegistration;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.ThresholdFaultPolicy;
import org.jboss.jandex.IndexView;

import java.util.ArrayList;
import java.util.List;

public class AnnotationsDiscovery {

    public List<BeanRegistration> discover(IndexView index) {
        List<BeanRegistration> beans = new ArrayList<>();

        for (GraphDescriptor gd : DescriptorScanner.scanGraphs(index)) {
            GoalCompiler<?> compiler = GoalCompilerFactory.create(gd);
            beans.add(new BeanRegistration(
                "goalCompiler_" + gd.namespace() + "_" + gd.name(),
                GoalCompiler.class, compiler));
        }

        for (FaultPolicyDescriptor fpd : DescriptorScanner.scanFaultPolicies(index)) {
            ThresholdFaultPolicy policy = FaultPolicyFactory.create(fpd, fpd.sourceClassName());
            beans.add(new BeanRegistration(
                "faultPolicy_" + fpd.namespace(),
                ThresholdFaultPolicy.class, policy));
        }

        return beans;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -pl annotations/runtime -Dtest=AnnotationsDiscoveryTest test`
Expected: PASS

- [ ] **Step 5: Refactor Spring auto-config to use AnnotationsDiscovery**

Replace the body of `DesiredStateAnnotationsAutoConfiguration.afterSingletonsInstantiated()` with:

```java
@Override
public void afterSingletonsInstantiated() {
    IndexView index = SpringJandexSupport.loadCompositeIndex();
    new AnnotationsDiscovery().discover(index)
        .forEach(reg -> context.registerBean(reg.name(), reg.type(), reg::instance));
}
```

Remove the now-unused `loadJandexIndexes()` private method. Update imports — remove Jandex imports that are no longer used, add `SpringJandexSupport` and `AnnotationsDiscovery` imports.

- [ ] **Step 6: Run full build to verify no regression**

Run: `mvn --batch-mode -pl api,annotations/core,annotations/runtime,annotations/spring install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/AnnotationsDiscovery.java annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/AnnotationsDiscoveryTest.java annotations/spring/src/main/java/io/casehub/desiredstate/annotations/spring/DesiredStateAnnotationsAutoConfiguration.java
git commit -m "feat(#155): AnnotationsDiscovery — extract annotation-driven bean discovery"
```

### Task 4: PluginDiscovery in plugin/runtime

**Files:**
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginDiscovery.java`
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/PluginDiscoveryTest.java`
- Modify: `plugin/spring/src/main/java/io/casehub/desiredstate/plugin/spring/DesiredStatePluginAutoConfiguration.java`

**Interfaces:**
- Consumes: `BeanRegistration` from api, `PluginParser.parse(InputStream)`, `PluginDescriptor` record
- Produces: `PluginDiscovery.discover(ClassLoader classLoader) → List<BeanRegistration>` — returns PluginDescriptor beans

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.desiredstate.plugin.runtime;

import io.casehub.desiredstate.api.BeanRegistration;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class PluginDiscoveryTest {

    @Test
    void discover_returnsEmptyWhenNoPluginsOnClasspath() {
        // Use a classloader that has no META-INF/desiredstate/plugins/*.yaml
        ClassLoader emptyLoader = new java.net.URLClassLoader(new java.net.URL[0], null);
        List<BeanRegistration> beans = new PluginDiscovery().discover(emptyLoader);
        assertThat(beans).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -pl plugin/runtime -Dtest=PluginDiscoveryTest test`
Expected: FAIL — class does not exist

- [ ] **Step 3: Write minimal implementation**

Extract from `DesiredStatePluginAutoConfiguration.afterSingletonsInstantiated()`:

```java
package io.casehub.desiredstate.plugin.runtime;

import io.casehub.desiredstate.api.BeanRegistration;
import io.casehub.desiredstate.plugin.model.PluginModel;
import io.casehub.desiredstate.plugin.model.PluginParser;

import java.io.IOException;
import java.io.InputStream;
import java.net.URL;
import java.time.Duration;
import java.util.ArrayList;
import java.util.Enumeration;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class PluginDiscovery {

    public List<BeanRegistration> discover(ClassLoader classLoader) {
        List<BeanRegistration> beans = new ArrayList<>();
        try {
            for (String suffix : List.of(".yaml", ".yml")) {
                Enumeration<URL> resources = classLoader.getResources(
                    "META-INF/desiredstate/plugins" + "/*" + suffix);
                // ClassLoader.getResources doesn't support wildcards —
                // fall back to scanning known pattern
            }
            // Use the classloader to find plugin resources
            List<PluginModel> plugins = discoverPlugins(classLoader);
            for (PluginModel plugin : plugins) {
                String type = plugin.header().type();
                Map<String, String> authRefs = plugin.header().auth() != null
                    ? plugin.header().auth().entrySet().stream()
                        .collect(Collectors.toMap(Map.Entry::getKey, e -> e.getValue().credentialRef()))
                    : Map.of();
                Duration resync = plugin.header().resyncInterval() != null
                    ? Duration.parse("PT" + plugin.header().resyncInterval())
                    : Duration.ofMinutes(5);
                PluginDescriptor descriptor = new PluginDescriptor(
                    type, plugin.header().version(), resync, authRefs,
                    plugin.spec(), plugin.actualStateSteps(),
                    plugin.provisioner().provisionSteps(),
                    plugin.provisioner().deprovisionSteps(),
                    plugin.faultPolicies(), plugin.cbr(), plugin.ras());
                beans.add(new BeanRegistration("pluginDescriptor_" + type,
                    PluginDescriptor.class, descriptor));
            }
        } catch (IOException e) {
            throw new IllegalStateException("Failed to discover desired state plugins", e);
        }
        return beans;
    }

    private List<PluginModel> discoverPlugins(ClassLoader classLoader) throws IOException {
        // Plugin discovery needs Spring's PathMatchingResourcePatternResolver
        // or a framework-neutral alternative. Since ClassLoader.getResources
        // doesn't support wildcards, accept a List<InputStream> or use
        // a callback pattern.
        // For now, this method needs the caller to provide discovered resources.
        return List.of();
    }
}
```

**Note:** `ClassLoader.getResources()` does not support wildcards. The plugin/spring module currently uses Spring's `PathMatchingResourcePatternResolver` for `classpath*:META-INF/desiredstate/plugins/*.yaml`. The discovery class should accept the already-discovered `List<InputStream>` or `List<PluginModel>` rather than doing classpath scanning itself — the classpath scanning is framework-specific (Spring uses `PathMatchingResourcePatternResolver`, Quarkus uses build-time discovery).

Revised signature:

```java
public List<BeanRegistration> discover(List<PluginModel> plugins) {
    List<BeanRegistration> beans = new ArrayList<>();
    for (PluginModel plugin : plugins) {
        String type = plugin.header().type();
        Map<String, String> authRefs = plugin.header().auth() != null
            ? plugin.header().auth().entrySet().stream()
                .collect(Collectors.toMap(Map.Entry::getKey, e -> e.getValue().credentialRef()))
            : Map.of();
        Duration resync = plugin.header().resyncInterval() != null
            ? Duration.parse("PT" + plugin.header().resyncInterval())
            : Duration.ofMinutes(5);
        PluginDescriptor descriptor = new PluginDescriptor(
            type, plugin.header().version(), resync, authRefs,
            plugin.spec(), plugin.actualStateSteps(),
            plugin.provisioner().provisionSteps(),
            plugin.provisioner().deprovisionSteps(),
            plugin.faultPolicies(), plugin.cbr(), plugin.ras());
        beans.add(new BeanRegistration("pluginDescriptor_" + type,
            PluginDescriptor.class, descriptor));
    }
    return beans;
}
```

The Spring auto-config handles resource discovery and passes parsed models to the discovery class.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -pl plugin/runtime -Dtest=PluginDiscoveryTest test`
Expected: PASS

- [ ] **Step 5: Refactor Spring auto-config**

Replace `DesiredStatePluginAutoConfiguration.afterSingletonsInstantiated()`:

```java
@Override
public void afterSingletonsInstantiated() {
    try {
        List<PluginModel> plugins = discoverPlugins();
        new PluginDiscovery().discover(plugins)
            .forEach(reg -> context.registerBean(reg.name(), reg.type(), reg::instance));
    } catch (IOException e) {
        throw new IllegalStateException("Failed to discover desired state plugins", e);
    }
}
```

Keep the `discoverPlugins()` private method (it uses Spring's `PathMatchingResourcePatternResolver`). Remove the inline PluginDescriptor construction.

- [ ] **Step 6: Run build to verify no regression**

Run: `mvn --batch-mode -pl api,plugin/api,plugin/runtime,plugin/spring install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginDiscovery.java plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/PluginDiscoveryTest.java plugin/spring/src/main/java/io/casehub/desiredstate/plugin/spring/DesiredStatePluginAutoConfiguration.java
git commit -m "feat(#155): PluginDiscovery — extract plugin model to descriptor conversion"
```

## Batch 3: Discovery classes — YAML + TypeScript DSL

### Task 5: YamlDiscovery in yaml/runtime

**Files:**
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlDiscovery.java`
- Test: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlDiscoveryTest.java`
- Modify: `yaml/spring/src/main/java/io/casehub/desiredstate/yaml/spring/DesiredStateYamlAutoConfiguration.java`

**Interfaces:**
- Consumes: `BeanRegistration`, `YamlGoalCompilerFactory.create(...)`, `YamlGoalCompilerFactory.createLifecycle(...)`, `YamlFaultPolicyBuilder.build(...)`, `YamlInvariantConverter`
- Produces: `YamlDiscovery.discover(List<YamlGraph> graphs, Map<String, String> typeRegistry, Map<String, YamlModule> modules) → List<BeanRegistration>`

This is the most complex discovery class (207 lines of source → ~80 lines of domain logic). The classpath scanning (YAML files, modules, Jandex) stays in the Spring auto-config; the factory invocations move to the discovery class.

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.desiredstate.yaml;

import io.casehub.desiredstate.api.BeanRegistration;
import io.casehub.desiredstate.api.GoalCompiler;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class YamlDiscoveryTest {

    @Test
    void discover_returnsEmptyForEmptyGraphList() {
        List<BeanRegistration> beans = new YamlDiscovery().discover(
            List.of(), Map.of(), Map.of());
        assertThat(beans).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -pl yaml/runtime -Dtest=YamlDiscoveryTest test`
Expected: FAIL — class does not exist

- [ ] **Step 3: Write minimal implementation**

Extract from `DesiredStateYamlAutoConfiguration.afterSingletonsInstantiated()`. The discovery class takes pre-discovered inputs (graphs, type registry, modules) and calls factories:

```java
package io.casehub.desiredstate.yaml;

import io.casehub.desiredstate.annotations.runtime.DependencyDescriptor;
import io.casehub.desiredstate.annotations.runtime.GraphDescriptor;
import io.casehub.desiredstate.annotations.runtime.NodeDescriptor;
import io.casehub.desiredstate.annotations.runtime.ResolvedInvariant;
import io.casehub.desiredstate.api.BeanRegistration;
import io.casehub.desiredstate.api.FaultPolicy;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.InMemoryFaultCountStore;
import io.casehub.desiredstate.api.ThresholdFaultPolicy;
import io.casehub.desiredstate.yaml.model.YamlFaultPolicy;
import io.casehub.desiredstate.yaml.model.YamlGraph;
import io.casehub.desiredstate.yaml.model.YamlInvariant;
import io.casehub.desiredstate.yaml.model.YamlNode;
import io.casehub.yaml.core.module.YamlModule;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public class YamlDiscovery {

    public List<BeanRegistration> discover(
            List<YamlGraph> graphs,
            Map<String, String> typeRegistry,
            Map<String, YamlModule> modules) {

        List<BeanRegistration> beans = new ArrayList<>();

        for (YamlGraph yamlGraph : graphs) {
            String ns = yamlGraph.desiredState().namespace();
            String name = yamlGraph.desiredState().name();
            Map<String, Object> variables = yamlGraph.variables() != null
                ? yamlGraph.variables() : Map.of();
            List<ResolvedInvariant> invariants = buildInvariants(yamlGraph.invariants());

            GoalCompiler<?> compiler;
            if (yamlGraph.lifecycle() != null) {
                compiler = YamlGoalCompilerFactory.createLifecycle(
                    yamlGraph, typeRegistry, variables, invariants);
            } else {
                GraphDescriptor descriptor = toGraphDescriptor(yamlGraph, typeRegistry);
                compiler = YamlGoalCompilerFactory.create(
                    descriptor, typeRegistry, variables,
                    invariants, yamlGraph, modules, List.of(), List.of());
            }
            beans.add(new BeanRegistration(
                "goalCompiler_yaml_" + ns + "_" + name,
                GoalCompiler.class, compiler));

            for (YamlFaultPolicy yamlPolicy : yamlGraph.faultPolicy()) {
                ThresholdFaultPolicy policy = YamlFaultPolicyBuilder.build(
                    yamlPolicy, typeRegistry, new InMemoryFaultCountStore());
                beans.add(new BeanRegistration(
                    "faultPolicy_yaml_" + ns + "_" + yamlPolicy.namespace(),
                    FaultPolicy.class, policy));
            }
        }

        return beans;
    }

    // Move buildInvariants() and toGraphDescriptor() from the auto-config
    private List<ResolvedInvariant> buildInvariants(Map<String, YamlInvariant> yamlInvariants) {
        List<ResolvedInvariant> invariants = new ArrayList<>();
        for (Map.Entry<String, YamlInvariant> entry : yamlInvariants.entrySet()) {
            invariants.add(YamlInvariantConverter.toDeclarativeInvariant(
                entry.getKey(), entry.getValue()));
        }
        return invariants;
    }

    private GraphDescriptor toGraphDescriptor(YamlGraph yamlGraph, Map<String, String> typeRegistry) {
        List<NodeDescriptor> nodes = new ArrayList<>();
        List<DependencyDescriptor> deps = new ArrayList<>();

        for (Map.Entry<String, YamlNode> entry : yamlGraph.nodes().entrySet()) {
            String nodeId = entry.getKey();
            YamlNode yamlNode = entry.getValue();
            String specClassName = typeRegistry.get(yamlNode.type());
            nodes.add(new NodeDescriptor.InlineNode(
                nodeId, specClassName,
                yamlNode.spec() != null ? yamlNode.spec() : Map.of(),
                yamlNode.humanGating()));
            for (String dep : yamlNode.dependencyNodeIds()) {
                deps.add(new DependencyDescriptor(nodeId, dep));
            }
        }

        return new GraphDescriptor(
            yamlGraph.desiredState().namespace(),
            yamlGraph.desiredState().name(),
            null, null, nodes, deps,
            List.of(), null, List.of(), List.of());
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -pl yaml/runtime -Dtest=YamlDiscoveryTest test`
Expected: PASS

- [ ] **Step 5: Refactor Spring auto-config**

Replace `DesiredStateYamlAutoConfiguration.afterSingletonsInstantiated()` body:

```java
@Override
public void afterSingletonsInstantiated() {
    try {
        IndexView index = SpringJandexSupport.loadCompositeIndex();
        Map<String, String> typeRegistry = SpringJandexSupport.scanNodeTypes(index);
        if (typeRegistry.isEmpty()) {
            return;
        }
        ObjectMapper yamlMapper = new ObjectMapper(YAMLFactory.builder()
            .enable(YAMLParser.Feature.PARSE_BOOLEAN_LIKE_WORDS_AS_STRINGS).build());
        List<YamlGraph> graphs = discoverYamlGraphs(yamlMapper);
        Map<String, YamlModule> modules = discoverModules(yamlMapper);

        new YamlDiscovery().discover(graphs, typeRegistry, modules)
            .forEach(reg -> context.registerBean(reg.name(), reg.type(), reg::instance));
    } catch (IOException e) {
        throw new IllegalStateException("Failed to discover YAML desired state graphs", e);
    }
}
```

Remove `scanNodeTypes()`, `loadCompositeJandexIndex()`, `buildInvariants()`, `toGraphDescriptor()` from the auto-config. Keep `discoverYamlGraphs()` and `discoverModules()` (Spring-specific resource scanning).

- [ ] **Step 6: Run build to verify no regression**

Run: `mvn --batch-mode -pl api,annotations/core,annotations/runtime,yaml/runtime,yaml/spring install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlDiscovery.java yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlDiscoveryTest.java yaml/spring/src/main/java/io/casehub/desiredstate/yaml/spring/DesiredStateYamlAutoConfiguration.java
git commit -m "feat(#155): YamlDiscovery — extract YAML graph compilation and fault policy building"
```

### Task 6: TsDslDiscovery in ts-dsl/runtime

**Files:**
- Create: `ts-dsl/runtime/src/main/java/io/casehub/desiredstate/ts/TsDslDiscovery.java`
- Test: `ts-dsl/runtime/src/test/java/io/casehub/desiredstate/ts/TsDslDiscoveryTest.java`
- Modify: `ts-dsl/spring/src/main/java/io/casehub/desiredstate/ts/spring/DesiredStateTsDslAutoConfiguration.java`

**Interfaces:**
- Consumes: `BeanRegistration`, `TsGoalCompilerFactory.create(...)`, `TsGoalCompilerFactory.createLifecycle(...)`, `TsEnvelope`, `TsLifecycleEnvelope`
- Produces: `TsDslDiscovery.discover(List<DiscoveredEnvelope> envelopes, Map<String, String> typeRegistry) → List<BeanRegistration>`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.desiredstate.ts;

import io.casehub.desiredstate.api.BeanRegistration;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class TsDslDiscoveryTest {

    @Test
    void discover_returnsEmptyForEmptyEnvelopeList() {
        List<BeanRegistration> beans = new TsDslDiscovery().discover(
            List.of(), Map.of());
        assertThat(beans).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -pl ts-dsl/runtime -Dtest=TsDslDiscoveryTest test`
Expected: FAIL — class does not exist

- [ ] **Step 3: Write minimal implementation**

```java
package io.casehub.desiredstate.ts;

import io.casehub.desiredstate.annotations.runtime.DependencyDescriptor;
import io.casehub.desiredstate.annotations.runtime.GraphDescriptor;
import io.casehub.desiredstate.annotations.runtime.NodeDescriptor;
import io.casehub.desiredstate.api.BeanRegistration;
import io.casehub.desiredstate.api.GoalCompiler;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public class TsDslDiscovery {

    public record DiscoveredEnvelope(String name, TsEnvelope single, TsLifecycleEnvelope lifecycle) {}

    public List<BeanRegistration> discover(
            List<DiscoveredEnvelope> envelopes,
            Map<String, String> typeRegistry) {

        List<BeanRegistration> beans = new ArrayList<>();

        for (DiscoveredEnvelope discovered : envelopes) {
            GoalCompiler<?> compiler;
            if (discovered.lifecycle() != null) {
                compiler = TsGoalCompilerFactory.createLifecycle(
                    discovered.lifecycle(), typeRegistry, List.of());
            } else if (discovered.single() != null) {
                GraphDescriptor descriptor = toGraphDescriptor(discovered.single(), typeRegistry);
                compiler = TsGoalCompilerFactory.create(
                    descriptor, typeRegistry, List.of(), List.of(), List.of());
            } else {
                continue;
            }
            beans.add(new BeanRegistration(
                "goalCompiler_ts_" + discovered.name(),
                GoalCompiler.class, compiler));
        }

        return beans;
    }

    private GraphDescriptor toGraphDescriptor(TsEnvelope envelope, Map<String, String> typeRegistry) {
        List<NodeDescriptor> nodes = new ArrayList<>();
        for (TsEnvelopeNode en : envelope.nodes()) {
            String specClassName = typeRegistry.get(en.type());
            nodes.add(new NodeDescriptor.InlineNode(en.id(), specClassName,
                en.spec() != null ? en.spec() : Map.of(), null));
        }
        return new GraphDescriptor(
            envelope.namespace(), envelope.name(),
            null, null, nodes, envelope.dependencies(),
            List.of(), null, List.of(), List.of());
    }
}
```

Note: The `DiscoveredEnvelope` record moves from the Spring auto-config into the discovery class — it's a domain concept (a discovered TS graph that could be single or lifecycle), not Spring-specific.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -pl ts-dsl/runtime -Dtest=TsDslDiscoveryTest test`
Expected: PASS

- [ ] **Step 5: Refactor Spring auto-config**

Replace `DesiredStateTsDslAutoConfiguration.afterSingletonsInstantiated()`:

```java
@Override
public void afterSingletonsInstantiated() {
    try {
        IndexView index = SpringJandexSupport.loadCompositeIndex();
        Map<String, String> typeRegistry = SpringJandexSupport.scanNodeTypes(index);
        if (typeRegistry.isEmpty()) {
            return;
        }
        ObjectMapper mapper = new ObjectMapper();
        List<TsDslDiscovery.DiscoveredEnvelope> envelopes = discoverTsEnvelopes(mapper);

        new TsDslDiscovery().discover(envelopes, typeRegistry)
            .forEach(reg -> context.registerBean(reg.name(), reg.type(), reg::instance));
    } catch (IOException e) {
        throw new IllegalStateException("Failed to discover TypeScript DSL desired state graphs", e);
    }
}
```

Remove `scanNodeTypes()`, `loadCompositeJandexIndex()`, `toGraphDescriptor()`, and the inner `DiscoveredEnvelope` record. Keep `discoverTsEnvelopes()` (Spring-specific resource scanning). Update its return type to use `TsDslDiscovery.DiscoveredEnvelope`.

- [ ] **Step 6: Run build to verify no regression**

Run: `mvn --batch-mode -pl api,annotations/runtime,ts-dsl/runtime,ts-dsl/spring install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add ts-dsl/runtime/src/main/java/io/casehub/desiredstate/ts/TsDslDiscovery.java ts-dsl/runtime/src/test/java/io/casehub/desiredstate/ts/TsDslDiscoveryTest.java ts-dsl/spring/src/main/java/io/casehub/desiredstate/ts/spring/DesiredStateTsDslAutoConfiguration.java
git commit -m "feat(#155): TsDslDiscovery — extract TypeScript envelope compilation"
```

## Batch 4: Documentation + follow-up issue

### Task 7: Update CLAUDE.md, guides, and file follow-up issue

**Files:**
- Modify: `CLAUDE.md` — update module descriptions for annotations/runtime, yaml/runtime, plugin/runtime, ts-dsl/runtime, and all 4 spring modules
- Modify: `docs/guides/contributor-guide.md` — add section on discovery class pattern
- Modify: `ARC42STORIES.MD` — add discovery class pattern if applicable

**Interfaces:**
- Consumes: all discovery classes from Tasks 3-6
- Produces: documentation, follow-up GitHub issue for glue code generation

- [ ] **Step 1: Update CLAUDE.md module table**

Add discovery class references to each runtime module description. Update spring module descriptions to note they're now thin glue over discovery classes.

- [ ] **Step 2: Update contributor guide**

Add a subsection under the Spring auto-configuration section explaining the discovery class pattern: framework-neutral logic in runtime, trivial glue in spring.

- [ ] **Step 3: File follow-up issue for glue code generation**

```bash
gh issue create --repo casehubio/casehub-desiredstate \
  --title "feat: generate Spring auto-config glue from discovery classes" \
  --body "## Context

Follow-up to #155. The 4 Spring auto-configs are now ~15-line glue classes that:
1. Call SpringJandexSupport for index loading
2. Do framework-specific resource discovery (PathMatchingResourcePatternResolver)
3. Pass discovered resources to a discovery class
4. Iterate BeanRegistrations and call registerBean()

## What

Extend spring-generator (or create a new generator) to:
1. Scan for classes with discover() methods returning List<BeanRegistration>
2. Generate the SmartInitializingSingleton glue code
3. Wire the verify goal for drift detection

## Scale
Scale: S | Complexity: Med"
```

- [ ] **Step 4: Run full build**

Run: `mvn --batch-mode install -pl '!work-adapter'`
Expected: BUILD SUCCESS (excluding pre-existing work-adapter failure)

- [ ] **Step 5: Commit**

```bash
git add CLAUDE.md docs/guides/contributor-guide.md ARC42STORIES.MD
git commit -m "docs(#155): update module docs for discovery class pattern"
```

## References

- [2026-10-01-spring-discovery-classes-design.md] — design spec
- [annotations/spring/DesiredStateAnnotationsAutoConfiguration.java] — 75-line source
- [yaml/spring/DesiredStateYamlAutoConfiguration.java] — 207-line source (most complex)
- [plugin/spring/DesiredStatePluginAutoConfiguration.java] — 82-line source
- [ts-dsl/spring/DesiredStateTsDslAutoConfiguration.java] — 148-line source
- [GoalCompilerFactory] — framework-neutral factory (annotations)
- [YamlGoalCompilerFactory] — framework-neutral factory (YAML)
- [TsGoalCompilerFactory] — framework-neutral factory (TypeScript DSL)
- [FaultPolicyFactory] — framework-neutral factory (annotations fault policies)
- [YamlFaultPolicyBuilder] — YAML fault policy builder
- [PluginParser] — YAML plugin parser
- [spring-generator/AbstractVerifyMojo] — verify framework for drift detection
- [GitHub #155] — focal issue
