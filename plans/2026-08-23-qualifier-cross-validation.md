# GoalCompiler Qualifier + Cross-Model Validation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #110 — fix: apply @DesiredStateQualifier to GoalCompiler beans for multi-graph disambiguation
**Issue group:** #110, #111

**Goal:** Apply `@DesiredStateQualifier` to all GoalCompiler beans, fix `@DependsOn(nodes=...)` NPE on interface methods, and add cross-model validation (duplicate IDs, ref resolution, cycle detection).

**Architecture:** Three changes in the annotations module only. Part 1 modifies the processor to add qualifiers and fix two bugs (`@DependsOn(nodes)` NPE, fault policy merge). Part 2 restructures the validator from independent per-model loops to a single-scan architecture with `MergedGraph` per graph key — reference validation and cycle detection move to the merged structure, enabling cross-model checks.

**Tech Stack:** Quarkus build extensions (Jandex, SyntheticBeanBuildItem), CDI qualifiers, QuarkusUnitTest

## Global Constraints

- All changes scoped to `annotations/deployment/` and `annotations/runtime/` — no runtime, api, or example changes
- Pre-release platform — breaking changes cost nothing
- IntelliJ MCP required for all code editing
- Build: `mvn --batch-mode install`
- Tests: `mvn --batch-mode test -pl annotations/deployment -Dtest=<TestClass>`

---

## Batch 1: Processor Fixes — qualifier, @DependsOn(nodes), fault policy merge

After this batch: GoalCompiler beans carry `@DesiredStateQualifier`, `@DependsOn(nodes=...)` works on interface methods, and class fault policies survive merge into interface graphs.

### Task 1: GoalCompiler Qualifier Application

**Files:**
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java`
- Create: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/QualifierSingleGraphTest.java`
- Create: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/QualifierMultiGraphTest.java`

**Interfaces:**
- Consumes: `DesiredStateQualifier` annotation (`annotations/runtime`), `GoalCompiler` SPI (`api`)
- Produces: GoalCompiler beans with `@Default` + `@DesiredStateQualifier(namespace, name)` qualifiers

- [ ] **Step 1: Write QualifierSingleGraphTest**

```java
package io.casehub.desiredstate.annotations.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.desiredstate.annotations.DesiredState;
import io.casehub.desiredstate.annotations.DesiredStateQualifier;
import io.casehub.desiredstate.annotations.Node;
import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.DesiredStateGraphFactory;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.quarkus.test.QuarkusUnitTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

class QualifierSingleGraphTest {

    @RegisterExtension
    static final QuarkusUnitTest test = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(
                    QualifiedGraph.class, QSpec.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**");

    public record QSpec(String data) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("q"); }
    }

    @DesiredState(namespace = "qual", name = "single")
    public interface QualifiedGraph {
        @Node("q-node")
        default QSpec qNode() { return new QSpec("data"); }
    }

    @SuppressWarnings("unchecked")
    @Inject
    GoalCompiler unqualified;

    @SuppressWarnings("unchecked")
    @Inject
    @DesiredStateQualifier(namespace = "qual", name = "single")
    GoalCompiler qualified;

    private final DesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();

    @Test
    void unqualifiedInjectionStillResolves() {
        CompilationResult result = unqualified.compile(null, factory);
        assertThat(result).isInstanceOf(CompilationResult.SingleGraph.class);
        DesiredStateGraph graph = ((CompilationResult.SingleGraph) result).graph();
        assertThat(graph.nodes()).hasSize(1);
    }

    @Test
    void qualifiedInjectionResolves() {
        CompilationResult result = qualified.compile(null, factory);
        assertThat(result).isInstanceOf(CompilationResult.SingleGraph.class);
        DesiredStateGraph graph = ((CompilationResult.SingleGraph) result).graph();
        assertThat(graph.nodes()).hasSize(1);
    }

    @Test
    void bothInjectionsReturnSameGraph() {
        DesiredStateGraph g1 = ((CompilationResult.SingleGraph) unqualified.compile(null, factory)).graph();
        DesiredStateGraph g2 = ((CompilationResult.SingleGraph) qualified.compile(null, factory)).graph();
        assertThat(g1.nodes().keySet()).isEqualTo(g2.nodes().keySet());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=QualifierSingleGraphTest`
Expected: FAIL — qualified injection cannot resolve (no qualifier on bean)

- [ ] **Step 3: Implement qualifier on registerGoalCompilerBean**

Use `ide_replace_member` on `DesiredStateAnnotationsProcessor#registerGoalCompilerBean` to add namespace/name parameters and qualifier registration:

```java
@SuppressWarnings("rawtypes")
private void registerGoalCompilerBean(
        RuntimeValue<GoalCompiler> runtimeValue,
        BuildProducer<SyntheticBeanBuildItem> syntheticBeans,
        String namespace, String name) {
    syntheticBeans.produce(
            SyntheticBeanBuildItem.configure(GoalCompiler.class)
                    .scope(ApplicationScoped.class)
                    .unremovable()
                    .setRuntimeInit()
                    .runtimeValue(runtimeValue)
                    .addQualifier(jakarta.enterprise.inject.Default.class)
                    .addQualifier()
                        .annotation(io.casehub.desiredstate.annotations.DesiredStateQualifier.class)
                        .addValue("namespace", namespace)
                        .addValue("name", name)
                        .done()
                    .done());
}
```

Update both call sites in `generateDesiredStateGraphs`:
1. Interface path (~line 101): `registerGoalCompilerBean(runtimeValue, syntheticBeans, descriptor.namespace(), descriptor.name());`
2. Class-only path (~line 133): `registerGoalCompilerBean(runtimeValue, syntheticBeans, ns, nm);`

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=QualifierSingleGraphTest`
Expected: PASS

- [ ] **Step 5: Write QualifierMultiGraphTest**

```java
package io.casehub.desiredstate.annotations.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.desiredstate.annotations.DesiredState;
import io.casehub.desiredstate.annotations.DesiredStateQualifier;
import io.casehub.desiredstate.annotations.Node;
import io.casehub.desiredstate.api.CompilationResult;
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

class QualifierMultiGraphTest {

    @RegisterExtension
    static final QuarkusUnitTest test = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(
                    GraphA.class, GraphB.class, SpecA.class, SpecB.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**");

    public record SpecA(String data) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("type-a"); }
    }

    public record SpecB(String data) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("type-b"); }
    }

    @DesiredState(namespace = "multi", name = "graph-a")
    public interface GraphA {
        @Node("node-a")
        default SpecA nodeA() { return new SpecA("a"); }
    }

    @DesiredState(namespace = "multi", name = "graph-b")
    public interface GraphB {
        @Node("node-b")
        default SpecB nodeB() { return new SpecB("b"); }
    }

    @SuppressWarnings("unchecked")
    @Inject
    @DesiredStateQualifier(namespace = "multi", name = "graph-a")
    GoalCompiler compilerA;

    @SuppressWarnings("unchecked")
    @Inject
    @DesiredStateQualifier(namespace = "multi", name = "graph-b")
    GoalCompiler compilerB;

    private final DesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();

    @Test
    void eachQualifierResolvesCorrectGraph() {
        DesiredStateGraph graphA = ((CompilationResult.SingleGraph) compilerA.compile(null, factory)).graph();
        DesiredStateGraph graphB = ((CompilationResult.SingleGraph) compilerB.compile(null, factory)).graph();

        assertThat(graphA.nodes()).containsKey(NodeId.of("node-a"));
        assertThat(graphA.nodes()).doesNotContainKey(NodeId.of("node-b"));

        assertThat(graphB.nodes()).containsKey(NodeId.of("node-b"));
        assertThat(graphB.nodes()).doesNotContainKey(NodeId.of("node-a"));
    }

    @Test
    void eachGraphHasOneNode() {
        DesiredStateGraph graphA = ((CompilationResult.SingleGraph) compilerA.compile(null, factory)).graph();
        DesiredStateGraph graphB = ((CompilationResult.SingleGraph) compilerB.compile(null, factory)).graph();
        assertThat(graphA.nodes()).hasSize(1);
        assertThat(graphB.nodes()).hasSize(1);
    }
}
```

- [ ] **Step 6: Run multi-graph test**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=QualifierMultiGraphTest`
Expected: PASS (qualifier already implemented)

- [ ] **Step 7: Run all existing tests — verify no regression**

Run: `mvn --batch-mode test -pl annotations/deployment`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add annotations/deployment/
git commit -m "feat(#110): apply @DesiredStateQualifier to GoalCompiler beans

Always register @Default + @DesiredStateQualifier(namespace, name) on
every GoalCompiler synthetic bean. Single-graph unqualified injection
still works. Multi-graph apps use qualifier to disambiguate.

Refs #110"
```

---

### Task 2: @DependsOn(nodes) on Interface Methods + Fault Policy Merge

**Files:**
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java`
- Create: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/InterfaceNodesDependencyTest.java`
- Create: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/MergedFaultPolicyTest.java`

**Interfaces:**
- Consumes: `@DependsOn` annotation with `value` + `nodes` attributes, `@DeclareNode`, `@FaultPolicyDef`
- Produces: Fixed processor handling of `@DependsOn(nodes=...)` on interface methods, merged fault policies

- [ ] **Step 1: Write InterfaceNodesDependencyTest**

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

class InterfaceNodesDependencyTest {

    @RegisterExtension
    static final QuarkusUnitTest test = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(
                    InterfaceWithClassDep.class, ClassTarget.class, ISpec.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**");

    public record ISpec(String data) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("i"); }
    }

    @DeclareNode(namespace = "iface-dep", name = "test", id = "class-target")
    public static class ClassTarget implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("ct"); }
    }

    @DesiredState(namespace = "iface-dep", name = "test")
    public interface InterfaceWithClassDep {
        @Node("iface-node")
        @DependsOn(nodes = ClassTarget.class)
        default ISpec ifaceNode() { return new ISpec("iface"); }
    }

    @SuppressWarnings("unchecked")
    @Inject
    GoalCompiler compiler;

    private final DesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();

    @Test
    void interfaceNodesDependencyResolvedToClassTarget() {
        DesiredStateGraph graph = ((CompilationResult.SingleGraph) compiler.compile(null, factory)).graph();
        assertThat(graph.nodes()).hasSize(2);
        assertThat(graph.dependencies())
                .contains(new Dependency(NodeId.of("iface-node"), NodeId.of("class-target")));
    }

    @Test
    void noNpeWhenOnlyNodesAttributeUsed() {
        DesiredStateGraph graph = ((CompilationResult.SingleGraph) compiler.compile(null, factory)).graph();
        assertThat(graph.nodes().get(NodeId.of("iface-node"))).isNotNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=InterfaceNodesDependencyTest`
Expected: FAIL — NPE on `dependsOnAnn.value().asStringArray()` when only `nodes` is specified

- [ ] **Step 3: Fix processor buildGraphDescriptor — null-safe value() + nodes resolution**

Use `ide_replace_member` on `DesiredStateAnnotationsProcessor#buildGraphDescriptor`. Replace the `@DependsOn` handling block (the `if (dependsOnAnn != null)` block inside the `@Node` method loop) with null-safe code that handles both `value` and `nodes`:

```java
AnnotationInstance dependsOnAnn = method.annotation(DEPENDS_ON);
if (dependsOnAnn != null) {
    AnnotationValue stringDeps = dependsOnAnn.value();
    if (stringDeps != null) {
        for (String dep : stringDeps.asStringArray()) {
            deps.add(new DependencyDescriptor(nodeId, dep));
        }
    }

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
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=InterfaceNodesDependencyTest`
Expected: PASS

- [ ] **Step 5: Write MergedFaultPolicyTest**

```java
package io.casehub.desiredstate.annotations.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.desiredstate.annotations.DeclareNode;
import io.casehub.desiredstate.annotations.DesiredState;
import io.casehub.desiredstate.annotations.FaultPolicyDef;
import io.casehub.desiredstate.annotations.Node;
import io.casehub.desiredstate.annotations.Tier;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.FaultEvent;
import io.casehub.desiredstate.api.FaultPolicy;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.quarkus.test.QuarkusUnitTest;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

class MergedFaultPolicyTest {

    @RegisterExtension
    static final QuarkusUnitTest test = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(
                    MergeBase.class, MergeExtension.class, MSpec.class, ReviewSpec.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**");

    public record MSpec(String data) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("m"); }
    }

    public record ReviewSpec(String detail) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("review"); }
    }

    @DesiredState(namespace = "merge-fp", name = "test")
    public interface MergeBase {
        @Node("base-node")
        default MSpec baseNode() { return new MSpec("base"); }
    }

    @DeclareNode(namespace = "merge-fp", name = "test", id = "ext-node")
    @FaultPolicyDef(
            faultTypes = {"PROVISION_FAILED"},
            tiers = {@Tier(threshold = 3, review = "createReview")}
    )
    public static class MergeExtension implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("ext"); }

        public ReviewSpec createReview(FaultEvent event, DesiredStateGraph graph) {
            return new ReviewSpec("review");
        }
    }

    @Inject
    Instance<FaultPolicy> faultPolicies;

    @Test
    void classFaultPolicyPreservedInMergedGraph() {
        long count = faultPolicies.stream().count();
        assertThat(count).isGreaterThanOrEqualTo(1);
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=MergedFaultPolicyTest`
Expected: FAIL — no FaultPolicy bean produced (class policies dropped in merge)

- [ ] **Step 7: Fix processor merge path — merge class fault policies**

Use `ide_read_file` to find the exact merge block in `generateDesiredStateGraphs` (the `if (!classNodes.isEmpty())` block), then use `ide_replace_text_in_file` to add fault policy merging:

After `mergedDeps.addAll(resolveClassDependencies(classNodes, index));`, add:
```java
List<FaultPolicyDescriptor> mergedPolicies = new ArrayList<>(descriptor.faultPolicies());
mergedPolicies.addAll(collectClassFaultPolicies(classNodes, index));
```

And in the `new GraphDescriptor(...)` call, replace `descriptor.faultPolicies()` with `mergedPolicies`.

- [ ] **Step 8: Run test to verify it passes**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=MergedFaultPolicyTest`
Expected: PASS

- [ ] **Step 9: Run all tests — verify no regression**

Run: `mvn --batch-mode test -pl annotations/deployment`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```bash
git add annotations/deployment/
git commit -m "fix(#110): @DependsOn(nodes) on interface methods + fault policy merge

Fix NPE when @DependsOn(nodes=...) used on interface @Node methods
without string value. Add class ref resolution for interface methods
mirroring the existing @DeclareNode class handling.

Fix class fault policies silently dropped when @DeclareNode classes
merge into an interface-sourced graph.

Refs #110, #111"
```

---

## Batch 2: Validation Restructuring — single-scan + cross-model checks

After this batch: validation uses a single-scan architecture with MergedGraph. All cross-model validations work: duplicate IDs, string ref resolution, cycle detection, duplicate @DesiredState interfaces, @Customize on @DeclareNode, @DependsOn(nodes) target validation.

### Task 3: MergedGraph + Single-Scan Refactor

**Files:**
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java`

**Interfaces:**
- Consumes: Jandex `IndexView` with `@DesiredState`, `@DeclareNode`, `@Node`, `@DependsOn` annotations
- Produces: `MergedGraph` inner class, single-scan `validate()`, cross-model ref and cycle detection

This is a pure refactoring — no new validations. All existing tests must pass.

- [ ] **Step 1: Add MergedGraph inner class**

Use `ide_insert_member` on `AnnotationValidationStep` to add at the end of the class:

```java
private static class MergedGraph {
    final String graphKey;
    final Map<String, String> nodeIdToSource = new LinkedHashMap<>();
    final List<String> duplicateErrors = new ArrayList<>();
    final Map<String, List<String>> adjacency = new HashMap<>();

    MergedGraph(String graphKey) { this.graphKey = graphKey; }

    void addNode(String nodeId, String source) {
        String existing = nodeIdToSource.putIfAbsent(nodeId, source);
        if (existing != null) {
            duplicateErrors.add("Duplicate node id '" + nodeId
                + "' in graph '" + graphKey + "' — declared on "
                + existing + " and " + source);
        }
    }

    void addDependency(String fromId, String toId) {
        adjacency.computeIfAbsent(fromId, k -> new ArrayList<>()).add(toId);
    }

    void validateDuplicateIds(List<String> errors) {
        errors.addAll(duplicateErrors);
    }

    void validateDependencyRefs(List<String> errors) {
        for (var entry : adjacency.entrySet()) {
            for (String dep : entry.getValue()) {
                if (!nodeIdToSource.containsKey(dep)) {
                    errors.add("@DependsOn on '" + entry.getKey()
                        + "' in graph '" + graphKey + "' references '"
                        + dep + "' which is not declared as @Node or @DeclareNode");
                }
            }
        }
    }

    void detectCycles(List<String> errors) {
        Set<String> visited = new HashSet<>();
        Set<String> inStack = new HashSet<>();
        for (String node : adjacency.keySet()) {
            if (!visited.contains(node)) {
                Deque<String> path = new ArrayDeque<>();
                if (hasCycle(node, visited, inStack, path)) {
                    errors.add("Circular dependency detected in graph '"
                        + graphKey + "': " + String.join(" → ", path));
                }
            }
        }
    }

    private boolean hasCycle(String node, Set<String> visited,
            Set<String> inStack, Deque<String> path) {
        visited.add(node);
        inStack.add(node);
        path.addLast(node);

        for (String dep : adjacency.getOrDefault(node, List.of())) {
            if (!visited.contains(dep)) {
                if (hasCycle(dep, visited, inStack, path)) return true;
            } else if (inStack.contains(dep)) {
                path.addLast(dep);
                return true;
            }
        }

        inStack.remove(node);
        path.removeLast();
        return false;
    }
}
```

- [ ] **Step 2: Add resolveGraphKey helper**

Use `ide_insert_member` to add before `MergedGraph`:

```java
private String resolveGraphKey(AnnotationInstance ann, IndexView index) {
    String ns = stringValueOrDefault(ann, index, "namespace", "");
    String nm = stringValueOrDefault(ann, index, "name", "");
    return ns + ":" + nm;
}

private static String stringValueOrDefault(
        AnnotationInstance ann, IndexView index, String name, String defaultValue) {
    AnnotationValue value = ann.valueWithDefault(index, name);
    if (value == null) return defaultValue;
    String s = value.asString();
    return s != null ? s : defaultValue;
}
```

- [ ] **Step 3: Restructure validate() to single-scan with MergedGraph**

Use `ide_replace_member` on `AnnotationValidationStep#validate` to rewrite the method:

```java
@BuildStep
@Produce(ServiceStartBuildItem.class)
void validate(CombinedIndexBuildItem indexBuildItem) {
    IndexView index = indexBuildItem.getIndex();
    List<String> errors = new ArrayList<>();
    List<String> warnings = new ArrayList<>();

    Map<String, MergedGraph> graphsByKey = new LinkedHashMap<>();
    Map<String, String> interfacesByKey = new HashMap<>();

    // --- Phase 1: scan + per-element validation ---
    for (AnnotationInstance dsAnn : index.getAnnotations(DESIRED_STATE)) {
        ClassInfo dsClass = dsAnn.target().asClass();

        if (!java.lang.reflect.Modifier.isInterface(dsClass.flags())) {
            errors.add("@DesiredState on '" + dsClass.name().local()
                    + "' which is not an interface — @DesiredState must annotate an interface");
            continue;
        }

        String graphKey = resolveGraphKey(dsAnn, index);

        String existingIface = interfacesByKey.put(graphKey, dsClass.name().local());
        if (existingIface != null) {
            errors.add("Multiple @DesiredState interfaces with graph key '"
                + graphKey + "': " + existingIface + " and " + dsClass.name().local()
                + " — use a single interface per graph,"
                + " with @DeclareNode classes for extension nodes");
            continue;
        }

        MergedGraph mg = graphsByKey.computeIfAbsent(graphKey, k -> new MergedGraph(graphKey));

        Set<String> localNodeIds = new HashSet<>();
        Map<String, String> nodeIdToMethod = new HashMap<>();

        for (MethodInfo method : dsClass.methods()) {
            AnnotationInstance nodeAnn = method.annotation(NODE);
            if (nodeAnn != null) {
                String nodeId = nodeAnn.value().asString();

                if (!nodeIdToMethod.containsKey(nodeId)) {
                    localNodeIds.add(nodeId);
                    nodeIdToMethod.put(nodeId, method.name());
                } else {
                    errors.add("Duplicate @Node id '" + nodeId + "' on methods '"
                            + nodeIdToMethod.get(nodeId) + "' and '" + method.name() + "'");
                }

                if (!method.hasAnnotation(DotName.createSimple("java.lang.Override"))
                        && !isDefaultMethod(method)) {
                    errors.add("@Node on '" + method.name()
                            + "' must be a default method returning NodeSpec");
                }
                if (!implementsNodeSpec(method.returnType().name(), index)) {
                    errors.add("@Node '" + method.name() + "' return type "
                            + method.returnType().name().local()
                            + " does not implement NodeSpec");
                }

                mg.addNode(nodeId, "interface method "
                        + dsClass.name().local() + "#" + method.name());
                collectDepsIntoMergedGraph(method, index, nodeId, mg);
            }

            validateFaultPolicyOnMethod(method, dsClass, index, errors);
        }

        validateFaultPolicyFaultTypes(dsClass, index, errors);
        validateTierReviewMethods(dsClass, index, errors);
        validateGoalMethod(dsClass, index, errors);

        if (localNodeIds.isEmpty()) {
            warnings.add("@DesiredState '" + dsClass.name().local()
                    + "' has no @Node methods — graph will be empty");
        }
    }

    validateDeclareNodes(index, graphsByKey, errors, warnings);

    // --- Phase 2: cross-model validation ---
    for (MergedGraph mg : graphsByKey.values()) {
        mg.validateDuplicateIds(errors);
        mg.validateDependencyRefs(errors);
        mg.detectCycles(errors);
    }

    for (String warning : warnings) {
        LOG.warn(warning);
    }

    if (!errors.isEmpty()) {
        throw new RuntimeException(
                "Annotation validation failed:\n- " + String.join("\n- ", errors));
    }
}
```

- [ ] **Step 4: Add collectDepsIntoMergedGraph helper**

Use `ide_insert_member` to add:

```java
private void collectDepsIntoMergedGraph(MethodInfo method, IndexView index,
        String sourceNodeId, MergedGraph mg) {
    AnnotationInstance dependsOnAnn = method.annotation(DEPENDS_ON);
    if (dependsOnAnn == null) return;

    AnnotationValue stringDeps = dependsOnAnn.value();
    if (stringDeps != null) {
        for (String dep : stringDeps.asStringArray()) {
            mg.addDependency(sourceNodeId, dep);
        }
    }

    AnnotationValue classDeps = dependsOnAnn.value("nodes");
    if (classDeps != null) {
        for (var classRef : classDeps.asClassArray()) {
            ClassInfo targetClass = index.getClassByName(classRef.name());
            if (targetClass != null) {
                AnnotationInstance targetAnn = targetClass.declaredAnnotation(DECLARE_NODE);
                if (targetAnn != null) {
                    mg.addDependency(sourceNodeId,
                            targetAnn.value("id").asString());
                }
            }
        }
    }
}
```

- [ ] **Step 5: Update validateDeclareNodes to collect into MergedGraph**

Use `ide_replace_member` on `AnnotationValidationStep#validateDeclareNodes`. Change signature to accept `graphsByKey` and `warnings`, and add MergedGraph collection + `@DependsOn` deps:

```java
private void validateDeclareNodes(IndexView index,
        Map<String, MergedGraph> graphsByKey,
        List<String> errors, List<String> warnings) {
    for (AnnotationInstance dnAnn : index.getAnnotations(DECLARE_NODE)) {
        ClassInfo classInfo = dnAnn.target().asClass();
        String className = classInfo.name().local();

        if (java.lang.reflect.Modifier.isInterface(classInfo.flags())) {
            errors.add("@DeclareNode on interface '" + className
                    + "' — use @DesiredState for interfaces");
            continue;
        }
        if (java.lang.reflect.Modifier.isAbstract(classInfo.flags())) {
            errors.add("@DeclareNode on abstract class '" + className
                    + "' — must be concrete");
            continue;
        }
        if (!implementsNodeSpec(classInfo.name(), index)) {
            errors.add("@DeclareNode on '" + className
                    + "' which does not implement NodeSpec");
            continue;
        }

        if (classInfo.hasAnnotation(DESIRED_STATE)) {
            errors.add("'" + className
                    + "' has both @DeclareNode and @DesiredState — use one or the other");
        }

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

        String graphKey = resolveGraphKey(dnAnn, index);
        MergedGraph mg = graphsByKey.computeIfAbsent(graphKey, k -> new MergedGraph(graphKey));
        String nodeId = dnAnn.value("id").asString();
        mg.addNode(nodeId, "@DeclareNode class " + className);

        AnnotationInstance dependsOnAnn = classInfo.declaredAnnotation(DEPENDS_ON);
        if (dependsOnAnn != null) {
            AnnotationValue stringDeps = dependsOnAnn.value();
            if (stringDeps != null) {
                for (String dep : stringDeps.asStringArray()) {
                    mg.addDependency(nodeId, dep);
                }
            }
            AnnotationValue classDeps = dependsOnAnn.value("nodes");
            if (classDeps != null) {
                for (var classRef : classDeps.asClassArray()) {
                    ClassInfo targetClass = index.getClassByName(classRef.name());
                    if (targetClass != null) {
                        AnnotationInstance targetAnn =
                                targetClass.declaredAnnotation(DECLARE_NODE);
                        if (targetAnn != null) {
                            mg.addDependency(nodeId,
                                    targetAnn.value("id").asString());
                        }
                    }
                }
            }
        }
    }
}
```

- [ ] **Step 6: Remove old validateDependsOnRefs and detectCycles/hasCycle from top-level**

Use `ide_refactor_safe_delete` on `AnnotationValidationStep#validateDependsOnRefs`, `AnnotationValidationStep#detectCycles`, and `AnnotationValidationStep#hasCycle`. These are replaced by `MergedGraph.validateDependencyRefs()` and `MergedGraph.detectCycles()`.

- [ ] **Step 7: Run ALL existing tests — verify no regression**

Run: `mvn --batch-mode test -pl annotations/deployment`
Expected: ALL PASS — pure refactoring, no behavior change

- [ ] **Step 8: Verify with ide_diagnostics**

Run `ide_diagnostics` on `AnnotationValidationStep.java` to check for compilation errors.

- [ ] **Step 9: Commit**

```bash
git add annotations/deployment/
git commit -m "refactor(#111): single-scan validation with MergedGraph

Restructure AnnotationValidationStep from independent per-model loops
to single-scan architecture. MergedGraph per graph key collects nodes
and dependencies during scan. Reference validation and cycle detection
move from per-interface to MergedGraph, enabling cross-model checks.

Refs #111"
```

---

### Task 4: Cross-Model Validations + Additional Checks

**Files:**
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java`
- Create: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/CrossModelValidationTest.java`

**Interfaces:**
- Consumes: `MergedGraph` from Task 3
- Produces: Cross-model duplicate ID, string ref, cycle, duplicate @DesiredState, @Customize, @DependsOn(nodes) target validations

- [ ] **Step 1: Write CrossModelValidationTest with all negative test cases**

```java
package io.casehub.desiredstate.annotations.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.desiredstate.annotations.Customize;
import io.casehub.desiredstate.annotations.DeclareNode;
import io.casehub.desiredstate.annotations.DependsOn;
import io.casehub.desiredstate.annotations.DesiredState;
import io.casehub.desiredstate.annotations.Node;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.quarkus.test.QuarkusUnitTest;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

class CrossModelValidationTest {

    // --- Duplicate node ID across models ---

    @RegisterExtension
    static final QuarkusUnitTest duplicateId = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(
                    DupInterface.class, DupClass.class, DupSpec.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**")
            .assertException(t -> assertThat(t.getMessage())
                    .contains("Duplicate node id 'shared-id'"));

    public record DupSpec(String d) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("dup"); }
    }

    @DesiredState(namespace = "dup", name = "test")
    public interface DupInterface {
        @Node("shared-id")
        default DupSpec dupNode() { return new DupSpec("iface"); }
    }

    @DeclareNode(namespace = "dup", name = "test", id = "shared-id")
    public static class DupClass implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("dup"); }
    }

    // --- Cross-model string ref: class → interface ---

    @RegisterExtension
    static final QuarkusUnitTest crossRefUnresolved = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(
                    RefInterface.class, RefClass.class, RefSpec.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**")
            .assertException(t -> assertThat(t.getMessage())
                    .contains("references 'nonexistent'")
                    .contains("not declared as @Node or @DeclareNode"));

    public record RefSpec(String d) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("ref"); }
    }

    @DesiredState(namespace = "ref", name = "test")
    public interface RefInterface {
        @Node("ref-node")
        default RefSpec refNode() { return new RefSpec("r"); }
    }

    @DeclareNode(namespace = "ref", name = "test", id = "ref-class")
    @DependsOn("nonexistent")
    public static class RefClass implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("ref"); }
    }

    // --- Cross-model cycle ---

    @RegisterExtension
    static final QuarkusUnitTest crossCycle = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(
                    CycleInterface.class, CycleClass.class, CycleSpec.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**")
            .assertException(t -> assertThat(t.getMessage())
                    .contains("Circular dependency detected"));

    public record CycleSpec(String d) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("cyc"); }
    }

    @DesiredState(namespace = "cyc", name = "test")
    public interface CycleInterface {
        @Node("cycle-a")
        @DependsOn("cycle-b")
        default CycleSpec cycleA() { return new CycleSpec("a"); }
    }

    @DeclareNode(namespace = "cyc", name = "test", id = "cycle-b")
    @DependsOn("cycle-a")
    public static class CycleClass implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("cyc"); }
    }

    // --- Duplicate @DesiredState interfaces with same graph key ---

    @RegisterExtension
    static final QuarkusUnitTest duplicateDesiredState = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(
                    DupDsA.class, DupDsB.class, DupDsSpec.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**")
            .assertException(t -> assertThat(t.getMessage())
                    .contains("Multiple @DesiredState interfaces with graph key"));

    public record DupDsSpec(String d) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("dds"); }
    }

    @DesiredState(namespace = "dupds", name = "test")
    public interface DupDsA {
        @Node("ds-a")
        default DupDsSpec dsA() { return new DupDsSpec("a"); }
    }

    @DesiredState(namespace = "dupds", name = "test")
    public interface DupDsB {
        @Node("ds-b")
        default DupDsSpec dsB() { return new DupDsSpec("b"); }
    }

    // --- @Customize on @DeclareNode ---

    @RegisterExtension
    static final QuarkusUnitTest customizeOnDeclare = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(CustomizeOnClass.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**")
            .assertException(t -> assertThat(t.getMessage())
                    .contains("@Customize on @DeclareNode class")
                    .contains("@Customize requires a @DesiredState interface"));

    @DeclareNode(namespace = "cust", name = "test", id = "bad-cust")
    @Customize
    public static class CustomizeOnClass implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("c"); }
    }

    // --- @DependsOn(nodes) target lacks @DeclareNode ---

    @RegisterExtension
    static final QuarkusUnitTest nodesTargetNoDeclareNode = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(
                    TargetNoDeclare.class, SourceWithBadNodes.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**")
            .assertException(t -> assertThat(t.getMessage())
                    .contains("has no @DeclareNode annotation"));

    public static class TargetNoDeclare implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("t"); }
    }

    @DeclareNode(namespace = "nodes-v", name = "test", id = "bad-source")
    @DependsOn(nodes = TargetNoDeclare.class)
    public static class SourceWithBadNodes implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("s"); }
    }

    @Test
    void validationTestsAreInExtensions() {
    }
}
```

- [ ] **Step 2: Run test to verify which ones fail**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=CrossModelValidationTest`
Expected: Some pass (reusing existing validation), some fail (new cross-model checks not yet producing errors for all cases)

- [ ] **Step 3: Add @Customize DotName constant and check in validateDeclareNodes**

Use `ide_insert_member` to add the constant to `AnnotationValidationStep`:
```java
private static final DotName CUSTOMIZE = DotName.createSimple(
        "io.casehub.desiredstate.annotations.Customize");
```

In `validateDeclareNodes()`, add after the `@Node` check loop:
```java
if (classInfo.hasAnnotation(CUSTOMIZE)) {
    errors.add("@Customize on @DeclareNode class '" + className
            + "' — @Customize requires a @DesiredState interface");
}
```

- [ ] **Step 4: Add @DependsOn(nodes) target validation in validateDeclareNodes**

In `validateDeclareNodes()`, within the `classDeps` handling block, add validation before adding the dependency:

```java
if (classDeps != null) {
    for (var classRef : classDeps.asClassArray()) {
        ClassInfo targetClass = index.getClassByName(classRef.name());
        if (targetClass == null) {
            warnings.add("@DependsOn(nodes) on '" + className
                + "' references '" + classRef.name().local()
                + "' which is not in the Jandex index"
                + " (if the class is in an external JAR,"
                + " ensure a Jandex index is generated)");
        } else if (targetClass.declaredAnnotation(DECLARE_NODE) == null) {
            errors.add("@DependsOn(nodes) on '" + className
                + "' references '" + classRef.name().local()
                + "' which has no @DeclareNode annotation");
        } else if (!implementsNodeSpec(classRef.name(), index)) {
            errors.add("@DependsOn(nodes) on '" + className
                + "' references '" + classRef.name().local()
                + "' which does not implement NodeSpec");
        } else {
            mg.addDependency(nodeId,
                    targetClass.declaredAnnotation(DECLARE_NODE)
                               .value("id").asString());
        }
    }
}
```

Apply the same validation in `collectDepsIntoMergedGraph` for interface `@Node` methods. Replace the existing `classDeps` block with:

```java
AnnotationValue classDeps = dependsOnAnn.value("nodes");
if (classDeps != null) {
    for (var classRef : classDeps.asClassArray()) {
        ClassInfo targetClass = index.getClassByName(classRef.name());
        if (targetClass == null) {
            warnings.add("@DependsOn(nodes) on '" + sourceNodeId
                + "' references '" + classRef.name().local()
                + "' which is not in the Jandex index"
                + " (if the class is in an external JAR,"
                + " ensure a Jandex index is generated)");
        } else if (targetClass.declaredAnnotation(DECLARE_NODE) == null) {
            errors.add("@DependsOn(nodes) on '" + sourceNodeId
                + "' references '" + classRef.name().local()
                + "' which has no @DeclareNode annotation");
        } else if (!implementsNodeSpec(classRef.name(), index)) {
            errors.add("@DependsOn(nodes) on '" + sourceNodeId
                + "' references '" + classRef.name().local()
                + "' which does not implement NodeSpec");
        } else {
            mg.addDependency(sourceNodeId,
                    targetClass.declaredAnnotation(DECLARE_NODE)
                               .value("id").asString());
        }
    }
}
```

Update `collectDepsIntoMergedGraph` signature to accept `List<String> errors, List<String> warnings` parameters, and update its call site in `validate()`.

- [ ] **Step 5: Run all cross-model tests**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=CrossModelValidationTest`
Expected: ALL PASS

- [ ] **Step 6: Run full test suite — verify no regression**

Run: `mvn --batch-mode test -pl annotations/deployment`
Expected: ALL PASS

- [ ] **Step 7: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add annotations/deployment/
git commit -m "feat(#111): cross-model validation — duplicate IDs, refs, cycles, @Customize

Add cross-model validations on MergedGraph:
- Duplicate node ID across interface and class models
- Cross-model @DependsOn string ref resolution
- Cross-model circular dependency detection
- @Customize on @DeclareNode class → error
- @DependsOn(nodes) target without @DeclareNode → error
- @DependsOn(nodes) target not in Jandex → warning

Refs #111"
```

---

## References

- [2026-08-23-qualifier-cross-validation-design.md](/Users/mdproctor/claude/public/casehub-desiredstate/specs/issue-110-qualifier-cross-validation/2026-08-23-qualifier-cross-validation-design.md) — design spec this plan implements
- [DesiredStateAnnotationsProcessor.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java) — processor (qualifier + @DependsOn + fault policy)
- [AnnotationValidationStep.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java) — validator (single-scan restructure)
- [DesiredStateQualifier.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DesiredStateQualifier.java) — qualifier annotation
- [DependsOn.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DependsOn.java) — value + nodes attributes
- [Customize.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/Customize.java) — @Customize annotation
- [GitHub #110](https://github.com/casehubio/casehub-desiredstate/issues/110) — GoalCompiler qualifier disambiguation
- [GitHub #111](https://github.com/casehubio/casehub-desiredstate/issues/111) — cross-model validation phase 2
