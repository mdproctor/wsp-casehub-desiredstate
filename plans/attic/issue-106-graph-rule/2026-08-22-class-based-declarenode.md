# Class-Based @DeclareNode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #105 — class-based @DesiredNode — CDI-style composable node declarations
**Issue group:** #105

**Goal:** Add class-based `@DeclareNode` annotation that enables distributed node declarations across modules, complementing the existing interface-based `@DesiredState` model.

**Architecture:** Extend the existing `@DesiredState` annotations pipeline — Jandex scan → processor → descriptor records → recorder → CDI beans. Refactor `NodeDescriptor` to a sealed interface (`InterfaceNode | ClassNode`), extend `@DependsOn` with type-safe `Class<? extends NodeSpec>[]` references, restructure the validator for cross-model validation.

**Tech Stack:** Java 21, Quarkus build extensions (Jandex, Gizmo, SyntheticBeanBuildItem), JUnit 5 + QuarkusUnitTest

## Global Constraints

- Pre-release platform — breaking changes are free. Fix the design, never protect callers.
- No changes to API module (`desiredstate-api`), runtime module, or any module outside `annotations/`.
- `@DeclareNode` classes must be stateless — no-arg constructor, no mutable state.
- Node IDs are persistence-critical — always explicit, never derived from class names.
- Use `ide_*` tools for all Java source file operations. Bash only for git/build commands.

---

## Batch 1: Foundation — sealed NodeDescriptor refactoring

### Task 1: Refactor NodeDescriptor to sealed interface and update FaultPolicyDescriptor

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/NodeDescriptor.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/FaultPolicyDescriptor.java`
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java`
- Modify: `examples/pipeline-annotated/src/test/java/io/casehub/desiredstate/example/pipeline/annotated/MedallionPipelineTest.java`
- Test: existing tests in `annotations/deployment/src/test/` (regression)

**Interfaces:**
- Produces: `NodeDescriptor` sealed interface with `InterfaceNode` and `ClassNode` variants. `FaultPolicyDescriptor` with `sourceClassName` field (null for interface-sourced).

- [ ] **Step 1: Run existing tests to establish baseline**

```bash
mvn --batch-mode -pl annotations/deployment -am test
```
Expected: all 4 test classes pass (DesiredStateAnnotationsProcessorTest, FaultPolicyWiringTest, GoalMethodCompositionTest, ValidationErrorTest).

- [ ] **Step 2: Refactor NodeDescriptor to sealed interface**

Replace the contents of `NodeDescriptor.java` with:

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.api.HumanGating;

public sealed interface NodeDescriptor
        permits NodeDescriptor.InterfaceNode, NodeDescriptor.ClassNode {

    String id();

    record InterfaceNode(String id, String methodName, String returnTypeName,
                         HumanGating humanGating) implements NodeDescriptor {}

    record ClassNode(String id, String className) implements NodeDescriptor {}
}
```

- [ ] **Step 3: Add sourceClassName to FaultPolicyDescriptor**

Replace `FaultPolicyDescriptor.java` with:

```java
package io.casehub.desiredstate.annotations.runtime;

import java.util.List;

public record FaultPolicyDescriptor(
        List<String> faultTypes,
        List<String> nodeTypes,
        List<String> ignoreTypes,
        String namespace,
        List<TierDescriptor> tiers,
        String sourceClassName) {}
```

- [ ] **Step 4: Update DesiredStateAnnotationsProcessor — NodeDescriptor constructor**

In `buildGraphDescriptor()`, change the `NodeDescriptor` constructor call:

```java
// FROM:
nodes.add(new NodeDescriptor(nodeId, method.name(),
        method.returnType().name().toString(), gating));
// TO:
nodes.add(new NodeDescriptor.InterfaceNode(nodeId, method.name(),
        method.returnType().name().toString(), gating));
```

- [ ] **Step 5: Update DesiredStateAnnotationsProcessor — FaultPolicyDescriptor constructor**

In `buildFaultPolicyDescriptor()`, add `null` for `sourceClassName`:

```java
// FROM:
return new FaultPolicyDescriptor(faultTypes, nodeTypes, ignoreTypes, namespace, tiers);
// TO:
return new FaultPolicyDescriptor(faultTypes, nodeTypes, ignoreTypes, namespace, tiers, null);
```

- [ ] **Step 6: Update DesiredStateGraphRecorder — buildNodes pattern matching**

Replace the `buildNodes` method body:

```java
private static List<DesiredNode> buildNodes(Class<?> implClass, Object instance,
        GraphDescriptor descriptor) throws Exception {
    ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    List<DesiredNode> nodes = new ArrayList<>();
    for (NodeDescriptor nd : descriptor.nodes()) {
        switch (nd) {
            case NodeDescriptor.InterfaceNode in -> {
                Method method = implClass.getMethod(in.methodName());
                NodeSpec spec = (NodeSpec) method.invoke(instance);
                nodes.add(new DesiredNode(NodeId.of(in.id()), spec, in.humanGating()));
            }
            case NodeDescriptor.ClassNode cn -> {
                Class<?> nodeClass = classLoader.loadClass(cn.className());
                NodeSpec spec = (NodeSpec) nodeClass.getDeclaredConstructor().newInstance();
                nodes.add(new DesiredNode(NodeId.of(cn.id()), spec, spec.humanGating()));
            }
        }
    }
    return List.copyOf(nodes);
}
```

- [ ] **Step 7: Update DesiredStateGraphRecorder — createFaultPolicy sourceClassName**

In `createFaultPolicy()`, use `descriptor.sourceClassName()` when non-null to load the class for review method lookup, falling back to `implClassName`:

```java
public RuntimeValue<ThresholdFaultPolicy> createFaultPolicy(
        FaultPolicyDescriptor descriptor, String implClassName) {
    try {
        String className = descriptor.sourceClassName() != null
                ? descriptor.sourceClassName() : implClassName;
        Class<?> sourceClass = Thread.currentThread().getContextClassLoader()
                .loadClass(className);
        Object instance = sourceClass.getDeclaredConstructor().newInstance();
        // ... rest unchanged
```

- [ ] **Step 8: Update MedallionPipelineTest — NodeDescriptor and FaultPolicyDescriptor constructors**

In `buildDescriptorFromAnnotations()`:

```java
// NodeDescriptor:
nodes.add(new NodeDescriptor.InterfaceNode(nodeAnn.value(), method.getName(),
        method.getReturnType().getName(), nodeAnn.humanGating()));

// FaultPolicyDescriptor:
faultPolicies.add(new FaultPolicyDescriptor(
        Arrays.asList(fpAnn.faultTypes()),
        Arrays.asList(fpAnn.nodeTypes()),
        Arrays.asList(fpAnn.ignoreTypes()),
        fpAnn.namespace(),
        tiers, null));
```

- [ ] **Step 9: Run tests to verify no regression**

```bash
mvn --batch-mode -pl annotations/deployment -am test
mvn --batch-mode -pl examples/pipeline-annotated -am test
```
Expected: all existing tests pass. The sealed interface refactoring is purely structural.

- [ ] **Step 10: Commit**

```bash
git add annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/NodeDescriptor.java annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/FaultPolicyDescriptor.java annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java examples/pipeline-annotated/src/test/java/io/casehub/desiredstate/example/pipeline/annotated/MedallionPipelineTest.java
git commit -m "refactor(#105): sealed NodeDescriptor + FaultPolicyDescriptor sourceClassName

Refs #105"
```

---

## Batch 2: @DeclareNode core — annotation, processor, class-only graphs

### Task 2: Create @DeclareNode annotation, extend @DependsOn, implement class scan and class-only graph path

**Files:**
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DeclareNode.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DependsOn.java`
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java`
- Create test: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/ClassBasedNodeTest.java`

**Interfaces:**
- Consumes: `NodeDescriptor.ClassNode(id, className)` from Task 1
- Produces: `@DeclareNode` annotation, processor class-scan path, recorder class-only GoalCompiler

- [ ] **Step 1: Write ClassBasedNodeTest**

```java
package io.casehub.desiredstate.annotations.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.desiredstate.annotations.DeclareNode;
import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.DesiredStateGraphFactory;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.quarkus.test.QuarkusUnitTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

class ClassBasedNodeTest {

    @RegisterExtension
    static final QuarkusUnitTest test = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(
                    TestLoadBalancer.class, TestDnsRecord.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**");

    @DeclareNode(namespace = "test", name = "infra", id = "load-balancer")
    public static class TestLoadBalancer implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("lb"); }

        @Override
        public HumanGating humanGating() { return HumanGating.PROVISION_ONLY; }
    }

    @DeclareNode(namespace = "test", name = "infra", id = "dns-record")
    public static class TestDnsRecord implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("dns"); }
    }

    @SuppressWarnings("unchecked")
    @Inject
    GoalCompiler compiler;

    private final DesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();

    @Test
    void classBasedNodesProduceGoalCompiler() {
        CompilationResult result = compiler.compile(null, factory);
        assertThat(result).isInstanceOf(CompilationResult.SingleGraph.class);

        DesiredStateGraph graph = ((CompilationResult.SingleGraph) result).graph();
        assertThat(graph.nodes()).hasSize(2);
    }

    @Test
    void nodeIdMatchesAnnotation() {
        DesiredStateGraph graph = compileSingleGraph();
        assertThat(graph.nodes().get(NodeId.of("load-balancer"))).isNotNull();
        assertThat(graph.nodes().get(NodeId.of("dns-record"))).isNotNull();
    }

    @Test
    void nodeTypeFromNodeSpecMethod() {
        DesiredStateGraph graph = compileSingleGraph();
        assertThat(graph.nodes().get(NodeId.of("load-balancer")).type())
                .isEqualTo(NodeType.of("lb"));
        assertThat(graph.nodes().get(NodeId.of("dns-record")).type())
                .isEqualTo(NodeType.of("dns"));
    }

    @Test
    void nodeSpecDataPreserved() {
        DesiredStateGraph graph = compileSingleGraph();
        assertThat(graph.nodes().get(NodeId.of("load-balancer")).spec())
                .isInstanceOf(TestLoadBalancer.class);
    }

    @Test
    void humanGatingFromNodeSpecOverride() {
        DesiredStateGraph graph = compileSingleGraph();
        DesiredNode lb = graph.nodes().get(NodeId.of("load-balancer"));
        assertThat(lb.humanGating()).isEqualTo(HumanGating.PROVISION_ONLY);
        assertThat(lb.requiresHuman()).isTrue();

        DesiredNode dns = graph.nodes().get(NodeId.of("dns-record"));
        assertThat(dns.humanGating()).isEqualTo(HumanGating.NONE);
        assertThat(dns.requiresHuman()).isFalse();
    }

    @Test
    void classOnlyGraphIgnoresGoals() {
        CompilationResult result1 = compiler.compile(null, factory);
        CompilationResult result2 = compiler.compile("ignored", factory);
        DesiredStateGraph g1 = ((CompilationResult.SingleGraph) result1).graph();
        DesiredStateGraph g2 = ((CompilationResult.SingleGraph) result2).graph();
        assertThat(g1.nodes().keySet()).isEqualTo(g2.nodes().keySet());
    }

    private DesiredStateGraph compileSingleGraph() {
        return ((CompilationResult.SingleGraph) compiler.compile(null, factory)).graph();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn --batch-mode -pl annotations/deployment -am test -Dtest=ClassBasedNodeTest
```
Expected: compilation error — `DeclareNode` class does not exist.

- [ ] **Step 3: Create @DeclareNode annotation**

Create `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DeclareNode.java`:

```java
package io.casehub.desiredstate.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface DeclareNode {
    String namespace() default "";
    String name() default "";
    String id();
}
```

- [ ] **Step 4: Extend @DependsOn — add nodes attribute and TYPE target**

Modify `DependsOn.java`:

```java
package io.casehub.desiredstate.annotations;

import io.casehub.desiredstate.api.NodeSpec;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
public @interface DependsOn {
    String[] value() default {};
    Class<? extends NodeSpec>[] nodes() default {};
}
```

- [ ] **Step 5: Implement processor — scan @DeclareNode and build class-only graphs**

In `DesiredStateAnnotationsProcessor.java`, add a new constant:

```java
private static final DotName DECLARE_NODE = DotName.createSimple(
        "io.casehub.desiredstate.annotations.DeclareNode");
```

In `generateDesiredStateGraphs()`, after the existing `@DesiredState` loop, add class-based scanning and class-only graph handling. The processor must:

1. Build a map of `(namespace, name) → List<ClassNode descriptors>` from all `@DeclareNode` classes
2. For each (namespace, name) that has NO matching `@DesiredState` interface, create a class-only `GraphDescriptor` with null `interfaceName`/`implClassName`
3. Register a `GoalCompiler` bean for each class-only graph

Add helper method `buildClassNodeDescriptors()`:

```java
private Map<String, List<NodeDescriptor.ClassNode>> scanDeclareNodes(IndexView index) {
    Map<String, List<NodeDescriptor.ClassNode>> byGraph = new HashMap<>();
    for (AnnotationInstance ann : index.getAnnotations(DECLARE_NODE)) {
        ClassInfo classInfo = ann.target().asClass();
        String namespace = stringValueOrDefault(ann, index, "namespace", "");
        String name = stringValueOrDefault(ann, index, "name", "");
        String id = ann.value("id").asString();
        String graphKey = namespace + ":" + name;
        byGraph.computeIfAbsent(graphKey, k -> new ArrayList<>())
                .add(new NodeDescriptor.ClassNode(id, classInfo.name().toString()));
    }
    return byGraph;
}
```

In `generateDesiredStateGraphs()`, after the existing `@DesiredState` loop, track which graph keys already have interface-based graphs. For unmatched class-only graph keys, create and register:

```java
Map<String, List<NodeDescriptor.ClassNode>> classNodesByGraph = scanDeclareNodes(index);
Set<String> interfaceGraphKeys = new HashSet<>();
// ... populate interfaceGraphKeys in the existing @DesiredState loop

for (var entry : classNodesByGraph.entrySet()) {
    if (interfaceGraphKeys.contains(entry.getKey())) continue;
    String[] parts = entry.getKey().split(":", 2);
    String ns = parts[0];
    String nm = parts.length > 1 ? parts[1] : "";
    List<NodeDescriptor> nodes = new ArrayList<>(entry.getValue());
    GraphDescriptor descriptor = new GraphDescriptor(ns, nm, null, null,
            nodes, List.of(), List.of(), null);

    @SuppressWarnings("rawtypes")
    RuntimeValue<GoalCompiler> runtimeValue = recorder.createGoalCompiler(descriptor);
    syntheticBeans.produce(
            SyntheticBeanBuildItem.configure(GoalCompiler.class)
                    .scope(ApplicationScoped.class)
                    .unremovable()
                    .setRuntimeInit()
                    .runtimeValue(runtimeValue)
                    .done());
}
```

- [ ] **Step 6: Implement recorder — class-only path in createGoalCompiler**

In `createGoalCompiler()`, branch on null `implClassName`:

```java
public RuntimeValue<GoalCompiler> createGoalCompiler(GraphDescriptor descriptor) {
    try {
        List<Dependency> capturedDeps = buildDependencies(descriptor);

        if (descriptor.implClassName() == null) {
            List<DesiredNode> capturedNodes = buildClassOnlyNodes(descriptor);
            return new RuntimeValue<>((GoalCompiler) (goals, factory) ->
                    CompilationResult.single(factory.of(capturedNodes, capturedDeps)));
        }

        // ... existing interface path unchanged
```

Add `buildClassOnlyNodes()`:

```java
private static List<DesiredNode> buildClassOnlyNodes(GraphDescriptor descriptor) {
    ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    List<DesiredNode> nodes = new ArrayList<>();
    for (NodeDescriptor nd : descriptor.nodes()) {
        if (nd instanceof NodeDescriptor.ClassNode cn) {
            try {
                Class<?> nodeClass = classLoader.loadClass(cn.className());
                NodeSpec spec = (NodeSpec) nodeClass.getDeclaredConstructor().newInstance();
                nodes.add(new DesiredNode(NodeId.of(cn.id()), spec, spec.humanGating()));
            } catch (Exception e) {
                throw new RuntimeException("Failed to instantiate @DeclareNode class: "
                        + cn.className(), e);
            }
        }
    }
    return List.copyOf(nodes);
}
```

- [ ] **Step 7: Run tests**

```bash
mvn --batch-mode -pl annotations/deployment -am test -Dtest=ClassBasedNodeTest
```
Expected: all 6 tests pass.

- [ ] **Step 8: Run all existing tests to verify no regression**

```bash
mvn --batch-mode -pl annotations/deployment -am test
```
Expected: all tests pass (existing + new).

- [ ] **Step 9: Commit**

```bash
git add annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DeclareNode.java annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DependsOn.java annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/ClassBasedNodeTest.java
git commit -m "feat(#105): @DeclareNode annotation + class-only graph support

Refs #105"
```

### Task 3: Graph merge, type-safe dependencies, CDI qualifier

**Files:**
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java`
- Create test: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/MergedGraphTest.java`
- Create test: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/ClassBasedDependencyTest.java`

**Interfaces:**
- Consumes: `@DeclareNode` annotation, `@DependsOn(nodes=...)`, `NodeDescriptor.ClassNode`, processor class scan from Task 2
- Produces: merged graphs combining interface + class nodes; type-safe dependency resolution; `@DesiredStateQualifier` on all GoalCompiler beans

- [ ] **Step 1: Write MergedGraphTest**

```java
package io.casehub.desiredstate.annotations.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.desiredstate.annotations.DeclareNode;
import io.casehub.desiredstate.annotations.DependsOn;
import io.casehub.desiredstate.annotations.DesiredState;
import io.casehub.desiredstate.annotations.Node;
import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.Dependency;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.DesiredStateGraphFactory;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.quarkus.test.QuarkusUnitTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

class MergedGraphTest {

    @RegisterExtension
    static final QuarkusUnitTest test = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(
                    BaseGraph.class, ExtensionNode.class, TestSpec.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**");

    public record TestSpec(String data) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("test"); }
    }

    @DesiredState(namespace = "merge", name = "test")
    public interface BaseGraph {
        @Node("base-node")
        default TestSpec baseNode() {
            return new TestSpec("base");
        }
    }

    @DeclareNode(namespace = "merge", name = "test", id = "extension-node")
    @DependsOn("base-node")
    public static class ExtensionNode implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("ext"); }
    }

    @SuppressWarnings("unchecked")
    @Inject
    GoalCompiler compiler;

    private final DesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();

    @Test
    void mergedGraphContainsBothInterfaceAndClassNodes() {
        DesiredStateGraph graph = compileSingleGraph();
        assertThat(graph.nodes()).hasSize(2);
        assertThat(graph.nodes().get(NodeId.of("base-node"))).isNotNull();
        assertThat(graph.nodes().get(NodeId.of("extension-node"))).isNotNull();
    }

    @Test
    void crossModelStringDependencyResolved() {
        DesiredStateGraph graph = compileSingleGraph();
        assertThat(graph.dependencies())
                .contains(new Dependency(NodeId.of("extension-node"), NodeId.of("base-node")));
    }

    private DesiredStateGraph compileSingleGraph() {
        return ((CompilationResult.SingleGraph) compiler.compile(null, factory)).graph();
    }
}
```

- [ ] **Step 2: Write ClassBasedDependencyTest**

```java
package io.casehub.desiredstate.annotations.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.desiredstate.annotations.DeclareNode;
import io.casehub.desiredstate.annotations.DependsOn;
import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.Dependency;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.DesiredStateGraphFactory;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.quarkus.test.QuarkusUnitTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

class ClassBasedDependencyTest {

    @RegisterExtension
    static final QuarkusUnitTest test = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(
                    DepTarget.class, DepSource.class, MixedSource.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**");

    @DeclareNode(namespace = "dep", name = "test", id = "target")
    public static class DepTarget implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("t"); }
    }

    @DeclareNode(namespace = "dep", name = "test", id = "source")
    @DependsOn(nodes = DepTarget.class)
    public static class DepSource implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("s"); }
    }

    @DeclareNode(namespace = "dep", name = "test", id = "mixed")
    @DependsOn(value = "target", nodes = DepSource.class)
    public static class MixedSource implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("m"); }
    }

    @SuppressWarnings("unchecked")
    @Inject
    GoalCompiler compiler;

    private final DesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();

    @Test
    void typeSafeDependencyResolved() {
        DesiredStateGraph graph = compileSingleGraph();
        assertThat(graph.dependencies())
                .contains(new Dependency(NodeId.of("source"), NodeId.of("target")));
    }

    @Test
    void mixedDependenciesBothResolved() {
        DesiredStateGraph graph = compileSingleGraph();
        assertThat(graph.dependencies())
                .contains(new Dependency(NodeId.of("mixed"), NodeId.of("target")))
                .contains(new Dependency(NodeId.of("mixed"), NodeId.of("source")));
    }

    private DesiredStateGraph compileSingleGraph() {
        return ((CompilationResult.SingleGraph) compiler.compile(null, factory)).graph();
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
mvn --batch-mode -pl annotations/deployment -am test -Dtest=MergedGraphTest,ClassBasedDependencyTest
```
Expected: failure — merge logic and dependency resolution not implemented.

- [ ] **Step 4: Implement processor — merge class nodes into interface graphs**

In `generateDesiredStateGraphs()`, modify the existing `@DesiredState` loop to merge class-based nodes. After building the interface-based `GraphDescriptor`, check for matching class nodes:

```java
// In the @DesiredState loop, after buildGraphDescriptor():
String graphKey = namespace + ":" + name;
interfaceGraphKeys.add(graphKey);

List<NodeDescriptor.ClassNode> classNodes = classNodesByGraph.getOrDefault(graphKey, List.of());
if (!classNodes.isEmpty()) {
    List<NodeDescriptor> mergedNodes = new ArrayList<>(descriptor.nodes());
    mergedNodes.addAll(classNodes);

    List<DependencyDescriptor> mergedDeps = new ArrayList<>(descriptor.dependencies());
    mergedDeps.addAll(resolveClassDependencies(classNodes, index));

    descriptor = new GraphDescriptor(descriptor.namespace(), descriptor.name(),
            descriptor.interfaceName(), descriptor.implClassName(),
            mergedNodes, mergedDeps, descriptor.faultPolicies(), descriptor.goalMethod());
}
```

Move `scanDeclareNodes()` call before the `@DesiredState` loop so both loops can use it.

- [ ] **Step 5: Implement processor — resolve @DependsOn(nodes=...) class references**

Add method to resolve class-based dependencies:

```java
private List<DependencyDescriptor> resolveClassDependencies(
        List<NodeDescriptor.ClassNode> classNodes, IndexView index) {
    List<DependencyDescriptor> deps = new ArrayList<>();
    for (NodeDescriptor.ClassNode cn : classNodes) {
        ClassInfo classInfo = index.getClassByName(DotName.createSimple(cn.className()));
        if (classInfo == null) continue;

        AnnotationInstance dependsOnAnn = classInfo.declaredAnnotation(DEPENDS_ON);
        if (dependsOnAnn == null) continue;

        AnnotationValue stringDeps = dependsOnAnn.value();
        if (stringDeps != null) {
            for (String dep : stringDeps.asStringArray()) {
                deps.add(new DependencyDescriptor(cn.id(), dep));
            }
        }

        AnnotationValue classDeps = dependsOnAnn.value("nodes");
        if (classDeps != null) {
            for (var classRef : classDeps.asClassArray()) {
                String targetClassName = classRef.name().toString();
                ClassInfo targetClass = index.getClassByName(classRef.name());
                if (targetClass != null) {
                    AnnotationInstance targetAnn = targetClass.declaredAnnotation(DECLARE_NODE);
                    if (targetAnn != null) {
                        String targetId = targetAnn.value("id").asString();
                        deps.add(new DependencyDescriptor(cn.id(), targetId));
                    }
                }
            }
        }
    }
    return deps;
}
```

Also apply the same `nodes` resolution for `@DependsOn(nodes=...)` on `@Node` methods (interface model referencing class nodes). In `buildGraphDescriptor()`, after the existing `@DependsOn` string handling:

```java
AnnotationValue classDeps = dependsOnAnn.value("nodes");
if (classDeps != null) {
    for (var classRef : classDeps.asClassArray()) {
        ClassInfo targetClass = index.getClassByName(classRef.name());
        if (targetClass != null) {
            AnnotationInstance targetAnn = targetClass.declaredAnnotation(DECLARE_NODE);
            if (targetAnn != null) {
                String targetId = targetAnn.value("id").asString();
                deps.add(new DependencyDescriptor(nodeId, targetId));
            }
        }
    }
}
```

Also resolve class dependencies for class-only graphs in the second loop.

- [ ] **Step 6: Implement processor — CDI qualifier on all GoalCompiler beans**

Add import for `DesiredStateQualifier` and apply to all `SyntheticBeanBuildItem` registrations (both in the `@DesiredState` loop and the class-only loop):

```java
var beanBuilder = SyntheticBeanBuildItem.configure(GoalCompiler.class)
        .scope(ApplicationScoped.class)
        .unremovable()
        .setRuntimeInit()
        .runtimeValue(runtimeValue);

if (!namespace.isEmpty() || !name.isEmpty()) {
    beanBuilder.addQualifier()
            .annotation(io.casehub.desiredstate.annotations.DesiredStateQualifier.class)
            .addValue("namespace", namespace)
            .addValue("name", name)
            .done();
}

syntheticBeans.produce(beanBuilder.done());
```

- [ ] **Step 7: Run tests**

```bash
mvn --batch-mode -pl annotations/deployment -am test
```
Expected: all tests pass — existing, ClassBasedNodeTest, MergedGraphTest, ClassBasedDependencyTest.

- [ ] **Step 8: Commit**

```bash
git add annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/MergedGraphTest.java annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/ClassBasedDependencyTest.java
git commit -m "feat(#105): graph merge, type-safe @DependsOn(nodes=...), CDI qualifier

Refs #105"
```

---

## Batch 3: Fault policy + validation

### Task 4: @FaultPolicyDef on @DeclareNode classes

**Files:**
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java`
- Create test: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/ClassBasedFaultPolicyTest.java`

**Interfaces:**
- Consumes: `FaultPolicyDescriptor(sourceClassName)` from Task 1, `@DeclareNode` scan from Task 2
- Produces: `ThresholdFaultPolicy` beans from `@FaultPolicyDef` on `@DeclareNode` classes with auto-inferred `nodeTypes`

- [ ] **Step 1: Write ClassBasedFaultPolicyTest**

```java
package io.casehub.desiredstate.annotations.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.desiredstate.annotations.DeclareNode;
import io.casehub.desiredstate.annotations.FaultPolicyDef;
import io.casehub.desiredstate.annotations.Tier;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.FaultEvent;
import io.casehub.desiredstate.api.FaultPolicy;
import io.casehub.desiredstate.api.FaultType;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.ThresholdFaultPolicy;
import io.quarkus.test.QuarkusUnitTest;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

class ClassBasedFaultPolicyTest {

    @RegisterExtension
    static final QuarkusUnitTest test = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(
                    FaultedNode.class, ReviewSpec.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**");

    public record ReviewSpec(NodeId faultedNode, String detail) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("review"); }
    }

    @DeclareNode(namespace = "fp", name = "test", id = "faulted-node")
    @FaultPolicyDef(
            faultTypes = {"PROVISION_FAILED"},
            tiers = {@Tier(threshold = 3, review = "createReview")}
    )
    public static class FaultedNode implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("faulted"); }

        public ReviewSpec createReview(FaultEvent event, DesiredStateGraph graph) {
            return new ReviewSpec(event.node(), event.detail());
        }
    }

    @Inject
    Instance<FaultPolicy> faultPolicies;

    @Test
    void faultPolicyBeanRegistered() {
        assertThat(faultPolicies.stream().count()).isGreaterThanOrEqualTo(1);
    }

    @Test
    void faultPolicyIsThresholdType() {
        FaultPolicy policy = faultPolicies.stream().findFirst().orElseThrow();
        assertThat(policy).isInstanceOf(ThresholdFaultPolicy.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn --batch-mode -pl annotations/deployment -am test -Dtest=ClassBasedFaultPolicyTest
```
Expected: failure — no FaultPolicy bean registered from class-based source.

- [ ] **Step 3: Implement processor — collect fault policies from @DeclareNode classes**

In `generateDesiredStateGraphs()`, when processing `@DeclareNode` classes, collect their `@FaultPolicyDef` annotations. For each, create a `FaultPolicyDescriptor` with `sourceClassName` set to the class name:

```java
private List<FaultPolicyDescriptor> collectClassFaultPolicies(
        ClassInfo classInfo, IndexView index) {
    List<FaultPolicyDescriptor> policies = new ArrayList<>();
    AnnotationInstance single = classInfo.declaredAnnotation(FAULT_POLICY_DEF);
    if (single != null) {
        policies.add(buildFaultPolicyDescriptor(single, index, classInfo.name().toString()));
    }
    AnnotationInstance container = classInfo.declaredAnnotation(FAULT_POLICIES);
    if (container != null) {
        for (AnnotationInstance nested : container.value().asNestedArray()) {
            policies.add(buildFaultPolicyDescriptor(nested, index, classInfo.name().toString()));
        }
    }
    return policies;
}
```

Add an overloaded `buildFaultPolicyDescriptor` that accepts `sourceClassName`:

```java
private FaultPolicyDescriptor buildFaultPolicyDescriptor(
        AnnotationInstance fpAnn, IndexView index, String sourceClassName) {
    // same body as existing method, but pass sourceClassName to constructor
    return new FaultPolicyDescriptor(faultTypes, nodeTypes, ignoreTypes, namespace, tiers,
            sourceClassName);
}
```

When building `GraphDescriptor` for class-only graphs (and merged graphs), include the collected fault policies. Register `ThresholdFaultPolicy` beans for class-sourced policies.

- [ ] **Step 4: Run tests**

```bash
mvn --batch-mode -pl annotations/deployment -am test
```
Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/ClassBasedFaultPolicyTest.java
git commit -m "feat(#105): @FaultPolicyDef on @DeclareNode classes with auto-inferred nodeTypes

Refs #105"
```

### Task 5: Two-phase validation and misuse detection

**Files:**
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java`
- Create test: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/ClassBasedValidationTest.java`

**Interfaces:**
- Consumes: `@DeclareNode` annotation, `@DependsOn(nodes=...)`, all processor paths
- Produces: Build-time errors for invalid `@DeclareNode` usage, cross-model duplicates, annotation misuse

- [ ] **Step 1: Write ClassBasedValidationTest**

Create with negative-test inner classes (each in its own nested `QuarkusUnitTest`). Key test cases:

```java
package io.casehub.desiredstate.annotations.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.desiredstate.annotations.DeclareNode;
import io.casehub.desiredstate.annotations.DesiredState;
import io.casehub.desiredstate.annotations.GoalMethod;
import io.casehub.desiredstate.annotations.Node;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.quarkus.test.QuarkusUnitTest;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

class ClassBasedValidationTest {

    @RegisterExtension
    static final QuarkusUnitTest notNodeSpec = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(NotNodeSpec.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**")
            .assertException(t -> assertThat(t.getMessage())
                    .contains("does not implement NodeSpec"));

    @DeclareNode(namespace = "v", name = "t", id = "bad")
    public static class NotNodeSpec {}

    @RegisterExtension
    static final QuarkusUnitTest onInterface = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(BadInterface.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**")
            .assertException(t -> assertThat(t.getMessage())
                    .contains("use @DesiredState for interfaces"));

    @DeclareNode(namespace = "v", name = "t", id = "iface")
    public interface BadInterface extends NodeSpec {}

    @RegisterExtension
    static final QuarkusUnitTest onAbstract = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(AbstractNode.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**")
            .assertException(t -> assertThat(t.getMessage())
                    .contains("must be concrete"));

    @DeclareNode(namespace = "v", name = "t", id = "abs")
    public abstract static class AbstractNode implements NodeSpec {}

    @RegisterExtension
    static final QuarkusUnitTest dualAnnotation = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(Dual.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**")
            .assertException(t -> assertThat(t.getMessage())
                    .contains("both @DeclareNode and @DesiredState"));

    @DeclareNode(namespace = "v", name = "t", id = "dual")
    @DesiredState
    public interface Dual extends NodeSpec {}

    @RegisterExtension
    static final QuarkusUnitTest goalMethodOnClass = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(GoalOnClass.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**")
            .assertException(t -> assertThat(t.getMessage())
                    .contains("@GoalMethod requires a @DesiredState interface"));

    @DeclareNode(namespace = "v", name = "t", id = "gm")
    public static class GoalOnClass implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("t"); }
        @GoalMethod
        public void goal() {}
    }

    @Test
    void validationTestsAreInExtensions() {
        // marker — the actual assertions are in assertException above
    }
}
```

Note: multiple `@RegisterExtension` fields in one test class is the established Quarkus pattern for testing multiple build-time error cases. Each extension is a separate Quarkus app that must fail.

- [ ] **Step 2: Run tests to verify they fail**

```bash
mvn --batch-mode -pl annotations/deployment -am test -Dtest=ClassBasedValidationTest
```
Expected: failures — no validation logic yet.

- [ ] **Step 3: Implement two-phase validation in AnnotationValidationStep**

Add `DECLARE_NODE` constant and a new scan loop in `validate()`:

```java
private static final DotName DECLARE_NODE = DotName.createSimple(
        "io.casehub.desiredstate.annotations.DeclareNode");
```

**Phase 1 — Collect:** After existing `@DesiredState` loop, add `@DeclareNode` validation:

```java
for (AnnotationInstance dnAnn : index.getAnnotations(DECLARE_NODE)) {
    ClassInfo classInfo = dnAnn.target().asClass();
    String className = classInfo.name().local();

    if (java.lang.reflect.Modifier.isInterface(classInfo.flags())) {
        errors.add("@DeclareNode on interface '" + className
                + "' — use @DesiredState for interfaces");
        continue;
    }
    if (java.lang.reflect.Modifier.isAbstract(classInfo.flags())) {
        errors.add("@DeclareNode on abstract class '" + className + "' — must be concrete");
        continue;
    }
    if (!implementsNodeSpec(classInfo.name(), index)) {
        errors.add("@DeclareNode on '" + className + "' which does not implement NodeSpec");
        continue;
    }

    // Dual annotation check
    if (classInfo.hasAnnotation(DESIRED_STATE)) {
        errors.add("'" + className
                + "' has both @DeclareNode and @DesiredState — use one or the other");
    }

    // Misuse checks
    for (MethodInfo method : classInfo.methods()) {
        if (method.hasAnnotation(GOAL_METHOD)) {
            errors.add("@GoalMethod on @DeclareNode class '" + className
                    + "' — @GoalMethod requires a @DesiredState interface");
        }
        if (method.hasAnnotation(NODE)) {
            errors.add("@Node on @DeclareNode class '" + className
                    + "' — @Node is for @DesiredState interfaces");
        }
    }

    // Track node ID for cross-model duplicate detection
    String id = dnAnn.value("id").asString();
    // ... add to global nodeId map
}
```

**Phase 2 — Cross-model:** After both loops, check for duplicate node IDs across models and validate `@DependsOn(nodes=...)` references.

- [ ] **Step 4: Run tests**

```bash
mvn --batch-mode -pl annotations/deployment -am test
```
Expected: all tests pass — existing + all new class-based tests + validation tests.

- [ ] **Step 5: Full build verification**

```bash
mvn --batch-mode install
```
Expected: full build passes including all modules and examples.

- [ ] **Step 6: Commit**

```bash
git add annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/ClassBasedValidationTest.java
git commit -m "feat(#105): two-phase validation + annotation misuse detection

Refs #105"
```

---

## References

- [2026-08-22-class-based-desirednode-design.md](/Users/mdproctor/claude/public/casehub-desiredstate/specs/issue-105-class-based-desirednode/2026-08-22-class-based-desirednode-design.md) — design spec
- [decisions.md](/Users/mdproctor/claude/public/casehub-desiredstate/specs/issue-105-class-based-desirednode/decisions.md) — 8 design decisions (D3, D7 revised)
- [NodeDescriptor.java:5](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/NodeDescriptor.java) — current flat record
- [DesiredStateAnnotationsProcessor.java:36](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java) — build extension
- [DesiredStateGraphRecorder.java:26](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java) — runtime recorder
- [AnnotationValidationStep.java:25](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java) — build-time validation
- [DesiredStateAnnotationsProcessorTest.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessorTest.java) — existing test pattern
- [#102 annotations spec](/Users/mdproctor/claude/casehub/desiredstate/docs/specs/issue-102-desiredstate-annotations/2026-08-20-desiredstate-annotations-design.md) — foundation spec
- GitHub #105 — focal issue
