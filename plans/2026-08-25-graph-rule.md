# @GraphRule Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #106 — @GraphRule — parameterized + imperative graph growth rules
**Issue group:** #106

**Goal:** Add a fixed-point graph rewriting phase to the annotation-driven compilation pipeline with parameterized and imperative rule signatures.

**Architecture:** New annotations (`@GraphRule`, `@Match`, `@DirectDep`, `@Reaches`, `@NotExists`) are scanned by the existing Jandex-based build extension. A `GraphRuleEngine` in `annotations/runtime` executes the fixed-point loop at Quarkus runtime init, after @GoalMethod produces the base graph. The engine handles both parameterized pattern matching (cross-product of bindings, BFS reachability, absence guards) and imperative rules (raw graph access). Conflict detection and cycle pre-validation run per iteration.

**Tech Stack:** Quarkus build extensions (Jandex, Gizmo, SyntheticBeanBuildItem), Java 21 sealed interfaces, QuarkusUnitTest

## Global Constraints

- Pre-release platform — breaking changes to GraphDescriptor are free
- All new annotations in package `io.casehub.desiredstate.annotations`
- All new runtime types in package `io.casehub.desiredstate.annotations.runtime`
- Reuse existing `ConflictingMutationException` from api/
- Reuse existing `GraphMutation` sealed interface and `GraphMutations` utility from api/
- Reuse existing `ImmutableDesiredStateGraph.withMutation()` for applying mutations
- Tests use `QuarkusUnitTest` with `@RegisterExtension` pattern
- `GraphMutation.targetNodeId()` must be extracted from `GraphDiff` (runtime) to `GraphMutation` (api) as a default method

---

## Batch 1: Foundation — annotations, IR types, API extraction

### Task 1: Annotations, Direction enum, IR descriptor records

**Files:**
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/GraphRule.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/Match.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DirectDep.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/Reaches.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/NotExists.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/Direction.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/PatternKind.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/PatternParameterDescriptor.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleDescriptor.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/ResolvedGraphRule.java`
- Test: `annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/GraphRuleTypesTest.java`

**Interfaces:**
- Produces: `@GraphRule(graph: String)`, `@Match(type: String)`, `@DirectDep(type, of, direction)`, `@Reaches(type, of, direction)`, `@NotExists(type, of, direction)`, `Direction.DEPENDENCIES|DEPENDENTS`, `PatternKind` enum, `PatternParameterDescriptor(kind, nodeType, of, direction)`, `GraphRuleDescriptor(methodName, imperative, patterns, sourceClassName)`, `ResolvedGraphRule(name, method, instance, imperative, patterns)`

- [ ] **Step 1: Write test for annotation presence and attributes**

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.annotations.DirectDep;
import io.casehub.desiredstate.annotations.GraphRule;
import io.casehub.desiredstate.annotations.Match;
import io.casehub.desiredstate.annotations.NotExists;
import io.casehub.desiredstate.annotations.Reaches;
import org.junit.jupiter.api.Test;

import java.lang.annotation.ElementType;
import java.lang.annotation.RetentionPolicy;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class GraphRuleTypesTest {

    @Test
    void graphRuleAnnotationHasCorrectTargets() {
        var targets = GraphRule.class.getAnnotation(java.lang.annotation.Target.class).value();
        assertThat(targets).containsExactlyInAnyOrder(ElementType.TYPE, ElementType.METHOD);
    }

    @Test
    void graphRuleAnnotationHasRuntimeRetention() {
        var retention = GraphRule.class.getAnnotation(java.lang.annotation.Retention.class).value();
        assertThat(retention).isEqualTo(RetentionPolicy.RUNTIME);
    }

    @Test
    void graphRuleDefaultGraphIsEmpty() throws Exception {
        var graphMethod = GraphRule.class.getMethod("graph");
        assertThat(graphMethod.getDefaultValue()).isEqualTo("");
    }

    @Test
    void matchAnnotationTargetsParameter() {
        var targets = Match.class.getAnnotation(java.lang.annotation.Target.class).value();
        assertThat(targets).containsExactly(ElementType.PARAMETER);
    }

    @Test
    void directDepHasDefaultDirection() throws Exception {
        var dirMethod = DirectDep.class.getMethod("direction");
        assertThat(dirMethod.getDefaultValue()).isEqualTo(Direction.DEPENDENCIES);
    }

    @Test
    void directDepHasDefaultOfEmpty() throws Exception {
        var ofMethod = DirectDep.class.getMethod("of");
        assertThat(ofMethod.getDefaultValue()).isEqualTo("");
    }

    @Test
    void reachesHasDefaultDirection() throws Exception {
        var dirMethod = Reaches.class.getMethod("direction");
        assertThat(dirMethod.getDefaultValue()).isEqualTo(Direction.DEPENDENCIES);
    }

    @Test
    void notExistsHasDefaultDirection() throws Exception {
        var dirMethod = NotExists.class.getMethod("direction");
        assertThat(dirMethod.getDefaultValue()).isEqualTo(Direction.DEPENDENCIES);
    }

    @Test
    void directionEnumHasTwoValues() {
        assertThat(Direction.values()).containsExactly(Direction.DEPENDENCIES, Direction.DEPENDENTS);
    }

    @Test
    void patternKindEnumValues() {
        assertThat(PatternKind.values()).containsExactly(
                PatternKind.MATCH, PatternKind.DIRECT_DEP,
                PatternKind.REACHES, PatternKind.NOT_EXISTS);
    }

    @Test
    void patternParameterDescriptorConstruction() {
        var desc = new PatternParameterDescriptor(
                PatternKind.MATCH, "transformer", "", Direction.DEPENDENCIES);
        assertThat(desc.kind()).isEqualTo(PatternKind.MATCH);
        assertThat(desc.nodeType()).isEqualTo("transformer");
        assertThat(desc.of()).isEmpty();
        assertThat(desc.direction()).isEqualTo(Direction.DEPENDENCIES);
    }

    @Test
    void graphRuleDescriptorConstruction() {
        var desc = new GraphRuleDescriptor("myRule", true, List.of(), "com.example.MyClass");
        assertThat(desc.methodName()).isEqualTo("myRule");
        assertThat(desc.imperative()).isTrue();
        assertThat(desc.patterns()).isEmpty();
        assertThat(desc.sourceClassName()).isEqualTo("com.example.MyClass");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=GraphRuleTypesTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — annotations and types don't exist yet.

- [ ] **Step 3: Create all annotation files**

Create `GraphRule.java`:
```java
package io.casehub.desiredstate.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface GraphRule {
    String graph() default "";
}
```

Create `Match.java`:
```java
package io.casehub.desiredstate.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface Match {
    String type();
}
```

Create `DirectDep.java`:
```java
package io.casehub.desiredstate.annotations;

import io.casehub.desiredstate.annotations.runtime.Direction;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface DirectDep {
    String type();
    String of() default "";
    Direction direction() default Direction.DEPENDENCIES;
}
```

Create `Reaches.java`:
```java
package io.casehub.desiredstate.annotations;

import io.casehub.desiredstate.annotations.runtime.Direction;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface Reaches {
    String type();
    String of() default "";
    Direction direction() default Direction.DEPENDENCIES;
}
```

Create `NotExists.java`:
```java
package io.casehub.desiredstate.annotations;

import io.casehub.desiredstate.annotations.runtime.Direction;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface NotExists {
    String type();
    String of() default "";
    Direction direction() default Direction.DEPENDENCIES;
}
```

Create `Direction.java`:
```java
package io.casehub.desiredstate.annotations.runtime;

public enum Direction {
    DEPENDENCIES,
    DEPENDENTS
}
```

Create `PatternKind.java`:
```java
package io.casehub.desiredstate.annotations.runtime;

public enum PatternKind {
    MATCH,
    DIRECT_DEP,
    REACHES,
    NOT_EXISTS
}
```

Create `PatternParameterDescriptor.java`:
```java
package io.casehub.desiredstate.annotations.runtime;

public record PatternParameterDescriptor(
        PatternKind kind,
        String nodeType,
        String of,
        Direction direction) {}
```

Create `GraphRuleDescriptor.java`:
```java
package io.casehub.desiredstate.annotations.runtime;

import java.util.List;

public record GraphRuleDescriptor(
        String methodName,
        boolean imperative,
        List<PatternParameterDescriptor> patterns,
        String sourceClassName) {}
```

Create `ResolvedGraphRule.java`:
```java
package io.casehub.desiredstate.annotations.runtime;

import java.lang.reflect.Method;
import java.util.List;

public record ResolvedGraphRule(
        String name,
        Method method,
        Object instance,
        boolean imperative,
        List<PatternParameterDescriptor> patterns) {}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=GraphRuleTypesTest`
Expected: All 12 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/GraphRule.java \
       annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/Match.java \
       annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DirectDep.java \
       annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/Reaches.java \
       annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/NotExists.java \
       annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/Direction.java \
       annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/PatternKind.java \
       annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/PatternParameterDescriptor.java \
       annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleDescriptor.java \
       annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/ResolvedGraphRule.java \
       annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/GraphRuleTypesTest.java
git commit -m "feat(#106): @GraphRule annotations, Direction enum, IR descriptor records"
```

---

### Task 2: GraphDescriptor extension + GraphMutation.targetNodeId() + exception classes

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphDescriptor.java` — add `graphRules` field
- Modify: `api/src/main/java/io/casehub/desiredstate/api/GraphMutation.java` — add `targetNodeId()` default method
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/GraphDiff.java` — delegate to `GraphMutation.targetNodeId()`
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java` — pass `List.of()` for graphRules in all GraphDescriptor constructions
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleNonConvergenceException.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleCycleException.java`
- Test: `api/src/test/java/io/casehub/desiredstate/api/GraphMutationTargetNodeIdTest.java`

**Interfaces:**
- Consumes: `GraphRuleDescriptor` from Task 1
- Produces: `GraphDescriptor(... graphRules)`, `GraphMutation.targetNodeId() → NodeId`, `GraphRuleNonConvergenceException(rules, maxIterations)`, `GraphRuleCycleException(ruleNames, cyclePath)`

- [ ] **Step 1: Write test for GraphMutation.targetNodeId()**

```java
package io.casehub.desiredstate.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class GraphMutationTargetNodeIdTest {

    @Test
    void addNodeTargetNodeId() {
        var node = new DesiredNode(NodeId.of("a"), new TestSpec(), HumanGating.NONE);
        var mutation = new GraphMutation.AddNode(node);
        assertThat(mutation.targetNodeId()).isEqualTo(NodeId.of("a"));
    }

    @Test
    void removeNodeTargetNodeId() {
        var mutation = new GraphMutation.RemoveNode(NodeId.of("b"));
        assertThat(mutation.targetNodeId()).isEqualTo(NodeId.of("b"));
    }

    @Test
    void updateNodeTargetNodeId() {
        var node = new DesiredNode(NodeId.of("c"), new TestSpec(), HumanGating.NONE);
        var mutation = new GraphMutation.UpdateNode(NodeId.of("c"), node);
        assertThat(mutation.targetNodeId()).isEqualTo(NodeId.of("c"));
    }

    @Test
    void addDependencyTargetNodeIdIsNull() {
        var mutation = new GraphMutation.AddDependency(new Dependency(NodeId.of("a"), NodeId.of("b")));
        assertThat(mutation.targetNodeId()).isNull();
    }

    @Test
    void removeDependencyTargetNodeIdIsNull() {
        var mutation = new GraphMutation.RemoveDependency(new Dependency(NodeId.of("a"), NodeId.of("b")));
        assertThat(mutation.targetNodeId()).isNull();
    }

    record TestSpec() implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("test"); }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=GraphMutationTargetNodeIdTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `targetNodeId()` not defined on `GraphMutation`.

- [ ] **Step 3: Add targetNodeId() to GraphMutation**

Add default method to `GraphMutation` sealed interface in `api/src/main/java/io/casehub/desiredstate/api/GraphMutation.java`:

```java
public sealed interface GraphMutation {

    default NodeId targetNodeId() {
        return switch (this) {
            case AddNode m -> m.node().id();
            case RemoveNode m -> m.id();
            case UpdateNode m -> m.id();
            case AddDependency ignored -> null;
            case RemoveDependency ignored -> null;
        };
    }

    record AddNode(DesiredNode node) implements GraphMutation {}
    record RemoveNode(NodeId id) implements GraphMutation {}
    record UpdateNode(NodeId id, DesiredNode adaptedNode) implements GraphMutation {}
    record AddDependency(Dependency dependency) implements GraphMutation {}
    record RemoveDependency(Dependency dependency) implements GraphMutation {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl api -Dtest=GraphMutationTargetNodeIdTest`
Expected: All 5 tests PASS.

- [ ] **Step 5: Update GraphDiff.targetNodeId() to delegate**

In `runtime/src/main/java/io/casehub/desiredstate/runtime/GraphDiff.java`, replace the static method body:

```java
static NodeId targetNodeId(GraphMutation mutation) {
    return mutation.targetNodeId();
}
```

- [ ] **Step 6: Verify existing tests still pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=GraphDiffTest`
Expected: PASS (behavior unchanged).

- [ ] **Step 7: Extend GraphDescriptor with graphRules field**

Update `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphDescriptor.java`:

```java
package io.casehub.desiredstate.annotations.runtime;

import java.util.List;

public record GraphDescriptor(
        String namespace,
        String name,
        String interfaceName,
        String implClassName,
        List<NodeDescriptor> nodes,
        List<DependencyDescriptor> dependencies,
        List<FaultPolicyDescriptor> faultPolicies,
        GoalMethodDescriptor goalMethod,
        List<GraphRuleDescriptor> graphRules) {}
```

- [ ] **Step 8: Update all GraphDescriptor construction sites to pass List.of()**

In `DesiredStateAnnotationsProcessor.java`, every `new GraphDescriptor(...)` call needs `List.of()` appended as the 9th argument. There are 3 construction sites:

1. Line ~96 in `buildGraphDescriptor` merge path (merged with class nodes)
2. Line ~130 in the class-only graph path
3. Line ~358 in `buildGraphDescriptor` return statement

Update each to append `, List.of()` after the goalMethod argument.

- [ ] **Step 9: Create exception classes**

Create `GraphRuleNonConvergenceException.java`:
```java
package io.casehub.desiredstate.annotations.runtime;

import java.util.List;
import java.util.stream.Collectors;

public class GraphRuleNonConvergenceException extends RuntimeException {
    private final List<String> activeRuleNames;
    private final int maxIterations;

    public GraphRuleNonConvergenceException(List<ResolvedGraphRule> activeRules, int maxIterations) {
        super("Graph rules did not converge after " + maxIterations + " iterations. "
              + "Rules still producing mutations: "
              + activeRules.stream().map(ResolvedGraphRule::name).collect(Collectors.joining(", "))
              + ". Non-converging rules are usually caused by non-idempotent mutations. "
              + "Check that parameterized rules use @NotExists guards and imperative rules "
              + "check graph state before producing mutations.");
        this.activeRuleNames = activeRules.stream().map(ResolvedGraphRule::name).toList();
        this.maxIterations = maxIterations;
    }

    public List<String> getActiveRuleNames() { return activeRuleNames; }
    public int getMaxIterations() { return maxIterations; }
}
```

Create `GraphRuleCycleException.java`:
```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.api.NodeId;

import java.util.List;
import java.util.stream.Collectors;

public class GraphRuleCycleException extends RuntimeException {
    private final List<String> ruleNames;
    private final List<NodeId> cyclePath;

    public GraphRuleCycleException(List<String> ruleNames, List<NodeId> cyclePath) {
        super("Graph rules introduced a cycle: "
              + cyclePath.stream().map(NodeId::value).collect(Collectors.joining(" → "))
              + ". Rules: " + String.join(", ", ruleNames));
        this.ruleNames = ruleNames;
        this.cyclePath = cyclePath;
    }

    public List<String> getRuleNames() { return ruleNames; }
    public List<NodeId> getCyclePath() { return cyclePath; }
}
```

- [ ] **Step 10: Run full build to verify nothing is broken**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all existing tests pass with the extended GraphDescriptor.

- [ ] **Step 11: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/GraphMutation.java \
       api/src/test/java/io/casehub/desiredstate/api/GraphMutationTargetNodeIdTest.java \
       runtime/src/main/java/io/casehub/desiredstate/runtime/GraphDiff.java \
       annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphDescriptor.java \
       annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleNonConvergenceException.java \
       annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleCycleException.java \
       annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java
git commit -m "feat(#106): GraphDescriptor extension, GraphMutation.targetNodeId(), rule exceptions"
```

---

## Batch 2: GraphRuleEngine — fixed-point loop and pattern matching

### Task 3: Engine core — imperative rules, fixed-point loop, conflict detection, cycle pre-validation, mutation ordering

**Files:**
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngine.java`
- Test: `annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngineTest.java`

**Interfaces:**
- Consumes: `ResolvedGraphRule`, `PatternParameterDescriptor`, `Direction`, `GraphRuleNonConvergenceException`, `GraphRuleCycleException`, `ConflictingMutationException` (api/), `GraphMutation.targetNodeId()`, `ImmutableDesiredStateGraph` (via `DesiredStateGraph` interface + `DefaultDesiredStateGraphFactory`)
- Produces: `GraphRuleEngine.evaluate(DesiredStateGraph, List<ResolvedGraphRule>) → DesiredStateGraph`

- [ ] **Step 1: Write test — imperative rule modifies graph**

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.api.ConflictingMutationException;
import io.casehub.desiredstate.api.Dependency;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.GraphMutation;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Method;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class GraphRuleEngineTest {

    private final DefaultDesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();
    private final GraphRuleEngine engine = new GraphRuleEngine();

    record Spec(String name, String typeValue) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of(typeValue); }
    }

    // --- Helper to build imperative ResolvedGraphRule from a static method in this class ---
    private ResolvedGraphRule imperativeRule(String methodName) {
        try {
            Method m = GraphRuleEngineTest.class.getDeclaredMethod(methodName, DesiredStateGraph.class);
            return new ResolvedGraphRule(methodName, m, null, true, List.of());
        } catch (NoSuchMethodException e) {
            throw new RuntimeException(e);
        }
    }

    static List<GraphMutation> addMonitorRule(DesiredStateGraph graph) {
        if (graph.nodes().containsKey(NodeId.of("monitor"))) return List.of();
        return List.of(new GraphMutation.AddNode(
                new DesiredNode(NodeId.of("monitor"), new Spec("monitor", "monitor"), HumanGating.NONE)));
    }

    @Test
    void imperativeRuleAddsNode() {
        var graph = factory.of(
                List.of(new DesiredNode(NodeId.of("sink"), new Spec("sink", "sink"), HumanGating.NONE)),
                List.of());
        var result = engine.evaluate(graph, List.of(imperativeRule("addMonitorRule")));
        assertThat(result.nodes()).containsKey(NodeId.of("monitor"));
        assertThat(result.nodes()).hasSize(2);
    }

    @Test
    void emptyRuleListReturnsGraphUnchanged() {
        var graph = factory.of(
                List.of(new DesiredNode(NodeId.of("a"), new Spec("a", "a"), HumanGating.NONE)),
                List.of());
        var result = engine.evaluate(graph, List.of());
        assertThat(result.nodes()).hasSize(1);
    }

    static List<GraphMutation> alwaysMutateRule(DesiredStateGraph graph) {
        return List.of(new GraphMutation.AddNode(
                new DesiredNode(NodeId.of("x-" + graph.version()), new Spec("x", "x"), HumanGating.NONE)));
    }

    @Test
    void nonConvergenceThrowsException() {
        var graph = factory.of(List.of(), List.of());
        assertThatThrownBy(() -> engine.evaluate(graph, List.of(imperativeRule("alwaysMutateRule"))))
                .isInstanceOf(GraphRuleNonConvergenceException.class)
                .hasMessageContaining("alwaysMutateRule")
                .hasMessageContaining("100");
    }

    static List<GraphMutation> addNodeA(DesiredStateGraph graph) {
        return List.of(new GraphMutation.AddNode(
                new DesiredNode(NodeId.of("dup"), new Spec("a", "a"), HumanGating.NONE)));
    }

    static List<GraphMutation> addNodeADifferent(DesiredStateGraph graph) {
        return List.of(new GraphMutation.AddNode(
                new DesiredNode(NodeId.of("dup"), new Spec("b", "b"), HumanGating.NONE)));
    }

    @Test
    void conflictingMutationsThrowException() {
        var graph = factory.of(List.of(), List.of());
        assertThatThrownBy(() -> engine.evaluate(graph,
                List.of(imperativeRule("addNodeA"), imperativeRule("addNodeADifferent"))))
                .isInstanceOf(ConflictingMutationException.class)
                .hasMessageContaining("dup");
    }

    static List<GraphMutation> duplicateMutationRule(DesiredStateGraph graph) {
        if (graph.nodes().containsKey(NodeId.of("d"))) return List.of();
        var node = new DesiredNode(NodeId.of("d"), new Spec("d", "d"), HumanGating.NONE);
        return List.of(new GraphMutation.AddNode(node));
    }

    static List<GraphMutation> duplicateMutationRule2(DesiredStateGraph graph) {
        if (graph.nodes().containsKey(NodeId.of("d"))) return List.of();
        var node = new DesiredNode(NodeId.of("d"), new Spec("d", "d"), HumanGating.NONE);
        return List.of(new GraphMutation.AddNode(node));
    }

    @Test
    void identicalDuplicateMutationsDeduplicated() {
        var graph = factory.of(List.of(), List.of());
        var result = engine.evaluate(graph,
                List.of(imperativeRule("duplicateMutationRule"), imperativeRule("duplicateMutationRule2")));
        assertThat(result.nodes()).containsKey(NodeId.of("d"));
    }

    static List<GraphMutation> createCycleRule(DesiredStateGraph graph) {
        return List.of(new GraphMutation.AddDependency(new Dependency(NodeId.of("b"), NodeId.of("a"))));
    }

    @Test
    void cycleIntroducedByRuleThrowsException() {
        var graph = factory.of(
                List.of(
                        new DesiredNode(NodeId.of("a"), new Spec("a", "a"), HumanGating.NONE),
                        new DesiredNode(NodeId.of("b"), new Spec("b", "b"), HumanGating.NONE)),
                List.of(new Dependency(NodeId.of("a"), NodeId.of("b"))));
        assertThatThrownBy(() -> engine.evaluate(graph, List.of(imperativeRule("createCycleRule"))))
                .isInstanceOf(GraphRuleCycleException.class);
    }

    static List<GraphMutation> addEdgeAndRemoveNode(DesiredStateGraph graph) {
        if (!graph.nodes().containsKey(NodeId.of("b"))) return List.of();
        return List.of(
                new GraphMutation.RemoveNode(NodeId.of("b")),
                new GraphMutation.AddDependency(new Dependency(NodeId.of("c"), NodeId.of("a"))));
    }

    @Test
    void removeNodeBeforeAddDependencyNoFalsePositiveCycle() {
        var graph = factory.of(
                List.of(
                        new DesiredNode(NodeId.of("a"), new Spec("a", "a"), HumanGating.NONE),
                        new DesiredNode(NodeId.of("b"), new Spec("b", "b"), HumanGating.NONE),
                        new DesiredNode(NodeId.of("c"), new Spec("c", "c"), HumanGating.NONE)),
                List.of(new Dependency(NodeId.of("a"), NodeId.of("b")),
                        new Dependency(NodeId.of("b"), NodeId.of("c"))));
        var result = engine.evaluate(graph, List.of(imperativeRule("addEdgeAndRemoveNode")));
        assertThat(result.nodes()).doesNotContainKey(NodeId.of("b"));
        assertThat(result.dependencies()).contains(new Dependency(NodeId.of("c"), NodeId.of("a")));
    }

    static List<GraphMutation> contradictoryEdgeRule1(DesiredStateGraph graph) {
        return List.of(new GraphMutation.AddDependency(new Dependency(NodeId.of("a"), NodeId.of("b"))));
    }

    static List<GraphMutation> contradictoryEdgeRule2(DesiredStateGraph graph) {
        return List.of(new GraphMutation.RemoveDependency(new Dependency(NodeId.of("a"), NodeId.of("b"))));
    }

    @Test
    void contradictoryEdgeMutationsThrowConflict() {
        var graph = factory.of(
                List.of(
                        new DesiredNode(NodeId.of("a"), new Spec("a", "a"), HumanGating.NONE),
                        new DesiredNode(NodeId.of("b"), new Spec("b", "b"), HumanGating.NONE)),
                List.of());
        assertThatThrownBy(() -> engine.evaluate(graph,
                List.of(imperativeRule("contradictoryEdgeRule1"), imperativeRule("contradictoryEdgeRule2"))))
                .isInstanceOf(ConflictingMutationException.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=GraphRuleEngineTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `GraphRuleEngine` doesn't exist.

- [ ] **Step 3: Implement GraphRuleEngine**

Create `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngine.java`:

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.api.ConflictingMutationException;
import io.casehub.desiredstate.api.Dependency;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.GraphMutation;
import io.casehub.desiredstate.api.NodeId;

import java.lang.reflect.InvocationTargetException;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.HashMap;
import java.util.HashSet;
import java.util.LinkedHashMap;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

public class GraphRuleEngine {

    private static final int MAX_ITERATIONS = 100;

    public DesiredStateGraph evaluate(DesiredStateGraph graph, List<ResolvedGraphRule> rules) {
        for (int iteration = 0; iteration < MAX_ITERATIONS; iteration++) {
            List<RuleContribution> contributions = new ArrayList<>();

            for (ResolvedGraphRule rule : rules) {
                List<GraphMutation> mutations = rule.imperative()
                        ? evaluateImperative(rule, graph)
                        : evaluateParameterized(rule, graph);
                if (!mutations.isEmpty()) {
                    contributions.add(new RuleContribution(rule.name(), mutations));
                }
            }

            List<GraphMutation> allMutations = contributions.stream()
                    .flatMap(c -> c.mutations().stream()).toList();

            if (allMutations.isEmpty()) {
                return graph;
            }

            List<GraphMutation> deduped = deduplicateMutations(allMutations);
            detectNodeConflicts(deduped);
            detectEdgeConflicts(deduped);
            validateNoCycles(graph, deduped, contributions);
            graph = applyMutations(graph, sortByType(deduped));
        }

        List<ResolvedGraphRule> activeRules = rules.stream()
                .filter(r -> !(r.imperative()
                        ? evaluateImperative(r, graph)
                        : evaluateParameterized(r, graph)).isEmpty())
                .toList();
        throw new GraphRuleNonConvergenceException(
                activeRules.isEmpty() ? rules : activeRules, MAX_ITERATIONS);
    }

    @SuppressWarnings("unchecked")
    List<GraphMutation> evaluateImperative(ResolvedGraphRule rule, DesiredStateGraph graph) {
        try {
            return (List<GraphMutation>) rule.method().invoke(rule.instance(), graph);
        } catch (InvocationTargetException e) {
            throw new RuntimeException("Rule " + rule.name() + " failed: " + e.getCause().getMessage(),
                    e.getCause());
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Rule " + rule.name() + " inaccessible", e);
        }
    }

    List<GraphMutation> evaluateParameterized(ResolvedGraphRule rule, DesiredStateGraph graph) {
        // Parameterized evaluation implemented in Task 4
        return List.of();
    }

    private List<GraphMutation> deduplicateMutations(List<GraphMutation> mutations) {
        var seen = new LinkedHashSet<>(mutations);
        return new ArrayList<>(seen);
    }

    private void detectNodeConflicts(List<GraphMutation> mutations) {
        Map<NodeId, GraphMutation> byNodeId = new LinkedHashMap<>();
        for (GraphMutation m : mutations) {
            NodeId target = m.targetNodeId();
            if (target == null) continue;
            GraphMutation existing = byNodeId.get(target);
            if (existing != null && !existing.equals(m)) {
                throw new ConflictingMutationException(target, existing, m);
            }
            byNodeId.put(target, m);
        }
    }

    private void detectEdgeConflicts(List<GraphMutation> mutations) {
        Map<Dependency, GraphMutation> addEdges = new HashMap<>();
        Map<Dependency, GraphMutation> removeEdges = new HashMap<>();
        for (GraphMutation m : mutations) {
            switch (m) {
                case GraphMutation.AddDependency add -> addEdges.put(add.dependency(), m);
                case GraphMutation.RemoveDependency rem -> removeEdges.put(rem.dependency(), m);
                default -> {}
            }
        }
        for (var entry : addEdges.entrySet()) {
            GraphMutation remove = removeEdges.get(entry.getKey());
            if (remove != null) {
                throw new ConflictingMutationException(
                        entry.getKey().from(), entry.getValue(), remove);
            }
        }
    }

    private void validateNoCycles(DesiredStateGraph graph, List<GraphMutation> mutations,
                                   List<RuleContribution> contributions) {
        Map<NodeId, Set<NodeId>> edges = new HashMap<>();
        Set<NodeId> nodes = new HashSet<>(graph.nodes().keySet());

        for (var dep : graph.dependencies()) {
            edges.computeIfAbsent(dep.from(), k -> new HashSet<>()).add(dep.to());
        }

        Set<NodeId> removedNodes = new HashSet<>();
        for (GraphMutation m : mutations) {
            if (m instanceof GraphMutation.RemoveDependency rem) {
                Set<NodeId> set = edges.get(rem.dependency().from());
                if (set != null) set.remove(rem.dependency().to());
            }
        }
        for (GraphMutation m : mutations) {
            if (m instanceof GraphMutation.RemoveNode rem) {
                removedNodes.add(rem.id());
                nodes.remove(rem.id());
                edges.remove(rem.id());
                for (Set<NodeId> targets : edges.values()) {
                    targets.remove(rem.id());
                }
            }
        }
        for (GraphMutation m : mutations) {
            if (m instanceof GraphMutation.AddNode add) {
                nodes.add(add.node().id());
            }
        }
        for (GraphMutation m : mutations) {
            if (m instanceof GraphMutation.AddDependency add) {
                NodeId from = add.dependency().from();
                NodeId to = add.dependency().to();
                if (nodes.contains(from) && nodes.contains(to)) {
                    edges.computeIfAbsent(from, k -> new HashSet<>()).add(to);
                }
            }
        }

        List<NodeId> cyclePath = detectCycleInEdges(edges, nodes);
        if (cyclePath != null) {
            List<String> ruleNames = contributions.stream()
                    .map(RuleContribution::ruleName).toList();
            throw new GraphRuleCycleException(ruleNames, cyclePath);
        }
    }

    private List<NodeId> detectCycleInEdges(Map<NodeId, Set<NodeId>> edges, Set<NodeId> nodes) {
        Set<NodeId> visited = new HashSet<>();
        Set<NodeId> inStack = new HashSet<>();
        for (NodeId node : nodes) {
            if (!visited.contains(node)) {
                List<NodeId> path = new ArrayList<>();
                if (hasCycle(node, visited, inStack, path, edges)) {
                    return path;
                }
            }
        }
        return null;
    }

    private boolean hasCycle(NodeId node, Set<NodeId> visited, Set<NodeId> inStack,
                              List<NodeId> path, Map<NodeId, Set<NodeId>> edges) {
        visited.add(node);
        inStack.add(node);
        path.add(node);

        for (NodeId dep : edges.getOrDefault(node, Set.of())) {
            if (!visited.contains(dep)) {
                if (hasCycle(dep, visited, inStack, path, edges)) return true;
            } else if (inStack.contains(dep)) {
                path.add(dep);
                return true;
            }
        }

        inStack.remove(node);
        path.remove(path.size() - 1);
        return false;
    }

    private List<GraphMutation> sortByType(List<GraphMutation> mutations) {
        return mutations.stream().sorted(Comparator.comparingInt(m -> switch (m) {
            case GraphMutation.AddNode ignored -> 0;
            case GraphMutation.UpdateNode ignored -> 1;
            case GraphMutation.RemoveDependency ignored -> 2;
            case GraphMutation.RemoveNode ignored -> 3;
            case GraphMutation.AddDependency ignored -> 4;
        })).toList();
    }

    private DesiredStateGraph applyMutations(DesiredStateGraph graph, List<GraphMutation> sorted) {
        for (GraphMutation m : sorted) {
            graph = graph.withMutation(m);
        }
        return graph;
    }

    private record RuleContribution(String ruleName, List<GraphMutation> mutations) {}
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=GraphRuleEngineTest`
Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngine.java \
       annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngineTest.java
git commit -m "feat(#106): GraphRuleEngine — imperative rules, fixed-point loop, conflict + cycle detection"
```

---

### Task 4: Parameterized pattern matching — @Match, @DirectDep, @Reaches, @NotExists

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngine.java` — implement `evaluateParameterized()`
- Modify: `annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngineTest.java` — add parameterized tests

**Interfaces:**
- Consumes: `GraphRuleEngine.evaluateParameterized()` (stub from Task 3), `PatternParameterDescriptor`, `PatternKind`, `Direction`, `DesiredStateGraph.dependenciesOf()`, `DesiredStateGraph.dependentsOf()`
- Produces: Full parameterized evaluation in `GraphRuleEngine`

- [ ] **Step 1: Write tests for parameterized evaluation**

Add to `GraphRuleEngineTest.java`:

```java
// --- Parameterized rule helpers ---

private ResolvedGraphRule parameterizedRule(String methodName,
        List<PatternParameterDescriptor> patterns) {
    try {
        Class<?>[] paramTypes = new Class<?>[patterns.size()];
        for (int i = 0; i < patterns.size(); i++) {
            paramTypes[i] = patterns.get(i).kind() == PatternKind.NOT_EXISTS
                    ? Void.class : io.casehub.desiredstate.api.DesiredNode.class;
        }
        Method m = GraphRuleEngineTest.class.getDeclaredMethod(methodName, paramTypes);
        return new ResolvedGraphRule(methodName, m, null, false, patterns);
    } catch (NoSuchMethodException e) {
        throw new RuntimeException(e);
    }
}

static List<GraphMutation> addValidatorForTransformer(
        io.casehub.desiredstate.api.DesiredNode transformer) {
    return List.of(new GraphMutation.AddNode(
            new DesiredNode(NodeId.of("validator-" + transformer.id().value()),
                    new Spec("validator", "validator"), HumanGating.NONE)));
}

@Test
void matchBindsNodesByType() {
    var graph = factory.of(List.of(
            new DesiredNode(NodeId.of("tx1"), new Spec("tx1", "transformer"), HumanGating.NONE),
            new DesiredNode(NodeId.of("tx2"), new Spec("tx2", "transformer"), HumanGating.NONE),
            new DesiredNode(NodeId.of("src"), new Spec("src", "source"), HumanGating.NONE)),
            List.of());

    var rule = parameterizedRule("addValidatorForTransformer", List.of(
            new PatternParameterDescriptor(PatternKind.MATCH, "transformer", "", Direction.DEPENDENCIES)));

    var result = engine.evaluate(graph, List.of(rule));
    assertThat(result.nodes()).containsKey(NodeId.of("validator-tx1"));
    assertThat(result.nodes()).containsKey(NodeId.of("validator-tx2"));
    assertThat(result.nodes()).doesNotContainKey(NodeId.of("validator-src"));
}

static List<GraphMutation> bindDirectDep(
        io.casehub.desiredstate.api.DesiredNode matched,
        io.casehub.desiredstate.api.DesiredNode dep) {
    return List.of(new GraphMutation.AddNode(
            new DesiredNode(NodeId.of("found-" + dep.id().value()),
                    new Spec("found", "found"), HumanGating.NONE)));
}

@Test
void directDepBindsDirectDependency() {
    var graph = factory.of(List.of(
            new DesiredNode(NodeId.of("tx"), new Spec("tx", "transformer"), HumanGating.NONE),
            new DesiredNode(NodeId.of("src"), new Spec("src", "source"), HumanGating.NONE)),
            List.of(new Dependency(NodeId.of("tx"), NodeId.of("src"))));

    var rule = parameterizedRule("bindDirectDep", List.of(
            new PatternParameterDescriptor(PatternKind.MATCH, "transformer", "", Direction.DEPENDENCIES),
            new PatternParameterDescriptor(PatternKind.DIRECT_DEP, "source", "", Direction.DEPENDENCIES)));

    var result = engine.evaluate(graph, List.of(rule));
    assertThat(result.nodes()).containsKey(NodeId.of("found-src"));
}

@Test
void directDepDependentsDirection() {
    var graph = factory.of(List.of(
            new DesiredNode(NodeId.of("src"), new Spec("src", "source"), HumanGating.NONE),
            new DesiredNode(NodeId.of("tx"), new Spec("tx", "transformer"), HumanGating.NONE)),
            List.of(new Dependency(NodeId.of("tx"), NodeId.of("src"))));

    var rule = parameterizedRule("bindDirectDep", List.of(
            new PatternParameterDescriptor(PatternKind.MATCH, "source", "", Direction.DEPENDENCIES),
            new PatternParameterDescriptor(PatternKind.DIRECT_DEP, "transformer", "", Direction.DEPENDENTS)));

    var result = engine.evaluate(graph, List.of(rule));
    assertThat(result.nodes()).containsKey(NodeId.of("found-tx"));
}

static List<GraphMutation> bindReachable(
        io.casehub.desiredstate.api.DesiredNode matched,
        io.casehub.desiredstate.api.DesiredNode reached) {
    return List.of(new GraphMutation.AddNode(
            new DesiredNode(NodeId.of("reached-" + reached.id().value()),
                    new Spec("reached", "reached"), HumanGating.NONE)));
}

@Test
void reachesFindsTransitiveNode() {
    var graph = factory.of(List.of(
            new DesiredNode(NodeId.of("tx"), new Spec("tx", "transformer"), HumanGating.NONE),
            new DesiredNode(NodeId.of("mid"), new Spec("mid", "middle"), HumanGating.NONE),
            new DesiredNode(NodeId.of("src"), new Spec("src", "source"), HumanGating.NONE)),
            List.of(new Dependency(NodeId.of("tx"), NodeId.of("mid")),
                    new Dependency(NodeId.of("mid"), NodeId.of("src"))));

    var rule = parameterizedRule("bindReachable", List.of(
            new PatternParameterDescriptor(PatternKind.MATCH, "transformer", "", Direction.DEPENDENCIES),
            new PatternParameterDescriptor(PatternKind.REACHES, "source", "", Direction.DEPENDENCIES)));

    var result = engine.evaluate(graph, List.of(rule));
    assertThat(result.nodes()).containsKey(NodeId.of("reached-src"));
}

static List<GraphMutation> guardedRule(
        io.casehub.desiredstate.api.DesiredNode transformer, Void guard) {
    return List.of(new GraphMutation.AddNode(
            new DesiredNode(NodeId.of("validator-" + transformer.id().value()),
                    new Spec("validator", "validator"), HumanGating.NONE)));
}

@Test
void notExistsGlobalGuardPreventsRule() {
    var graph = factory.of(List.of(
            new DesiredNode(NodeId.of("tx"), new Spec("tx", "transformer"), HumanGating.NONE),
            new DesiredNode(NodeId.of("v"), new Spec("v", "validator"), HumanGating.NONE)),
            List.of());

    var rule = parameterizedRule("guardedRule", List.of(
            new PatternParameterDescriptor(PatternKind.MATCH, "transformer", "", Direction.DEPENDENCIES),
            new PatternParameterDescriptor(PatternKind.NOT_EXISTS, "validator", "", Direction.DEPENDENCIES)));

    var result = engine.evaluate(graph, List.of(rule));
    assertThat(result.nodes()).hasSize(2);
}

@Test
void notExistsGlobalGuardAllowsRuleWhenAbsent() {
    var graph = factory.of(List.of(
            new DesiredNode(NodeId.of("tx"), new Spec("tx", "transformer"), HumanGating.NONE)),
            List.of());

    var rule = parameterizedRule("guardedRule", List.of(
            new PatternParameterDescriptor(PatternKind.MATCH, "transformer", "", Direction.DEPENDENCIES),
            new PatternParameterDescriptor(PatternKind.NOT_EXISTS, "validator", "", Direction.DEPENDENCIES)));

    var result = engine.evaluate(graph, List.of(rule));
    assertThat(result.nodes()).containsKey(NodeId.of("validator-tx"));
}

@Test
void notExistsRelationalGuardChecksNamedBinding() {
    var graph = factory.of(List.of(
            new DesiredNode(NodeId.of("tx"), new Spec("tx", "transformer"), HumanGating.NONE),
            new DesiredNode(NodeId.of("v"), new Spec("v", "validator"), HumanGating.NONE)),
            List.of(new Dependency(NodeId.of("v"), NodeId.of("tx"))));

    var rule = parameterizedRule("guardedRule", List.of(
            new PatternParameterDescriptor(PatternKind.MATCH, "transformer", "", Direction.DEPENDENCIES),
            new PatternParameterDescriptor(PatternKind.NOT_EXISTS, "validator", "transformer", Direction.DEPENDENTS)));

    var result = engine.evaluate(graph, List.of(rule));
    assertThat(result.nodes()).hasSize(2);
}

@Test
void notExistsRelationalGuardAllowsWhenNoRelation() {
    var graph = factory.of(List.of(
            new DesiredNode(NodeId.of("tx"), new Spec("tx", "transformer"), HumanGating.NONE),
            new DesiredNode(NodeId.of("v"), new Spec("v", "validator"), HumanGating.NONE)),
            List.of());

    var rule = parameterizedRule("guardedRule", List.of(
            new PatternParameterDescriptor(PatternKind.MATCH, "transformer", "", Direction.DEPENDENCIES),
            new PatternParameterDescriptor(PatternKind.NOT_EXISTS, "validator", "transformer", Direction.DEPENDENTS)));

    var result = engine.evaluate(graph, List.of(rule));
    assertThat(result.nodes()).containsKey(NodeId.of("validator-tx"));
}

@Test
void fixedPointConvergenceWithGuard() {
    var graph = factory.of(List.of(
            new DesiredNode(NodeId.of("tx"), new Spec("tx", "transformer"), HumanGating.NONE)),
            List.of());

    var rule = parameterizedRule("guardedRule", List.of(
            new PatternParameterDescriptor(PatternKind.MATCH, "transformer", "", Direction.DEPENDENCIES),
            new PatternParameterDescriptor(PatternKind.NOT_EXISTS, "validator", "", Direction.DEPENDENCIES)));

    var result = engine.evaluate(graph, List.of(rule));
    assertThat(result.nodes()).containsKey(NodeId.of("validator-tx"));
    assertThat(result.nodes()).hasSize(2);
}
```

- [ ] **Step 2: Run tests to verify new tests fail**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=GraphRuleEngineTest`
Expected: New parameterized tests FAIL (evaluateParameterized returns empty list).

- [ ] **Step 3: Implement evaluateParameterized()**

Replace the stub `evaluateParameterized()` in `GraphRuleEngine.java` with the full implementation. The algorithm:

1. Collect @Match parameters — enumerate matching nodes per @Match, form cross-product
2. For each candidate tuple, walk the chain: @DirectDep queries neighbors, @Reaches does BFS, @NotExists guards
3. Invoke rule method for each surviving tuple
4. Collect mutations

```java
@SuppressWarnings("unchecked")
List<GraphMutation> evaluateParameterized(ResolvedGraphRule rule, DesiredStateGraph graph) {
    List<GraphMutation> allMutations = new ArrayList<>();
    List<List<DesiredNode>> matchSets = new ArrayList<>();
    List<PatternParameterDescriptor> patterns = rule.patterns();

    // Phase 1: collect match sets for @Match parameters
    for (PatternParameterDescriptor p : patterns) {
        if (p.kind() == PatternKind.MATCH) {
            NodeType targetType = NodeType.of(p.nodeType());
            List<DesiredNode> matches = graph.nodes().values().stream()
                    .filter(n -> n.type().equals(targetType)).toList();
            matchSets.add(matches);
        }
    }

    // Phase 2: iterate cross-product of match sets
    List<List<DesiredNode>> crossProduct = crossProduct(matchSets);
    for (List<DesiredNode> matchTuple : crossProduct) {
        expandAndInvoke(rule, graph, patterns, matchTuple, allMutations);
    }

    return allMutations;
}

private void expandAndInvoke(ResolvedGraphRule rule, DesiredStateGraph graph,
        List<PatternParameterDescriptor> patterns, List<DesiredNode> matchTuple,
        List<GraphMutation> allMutations) {
    // Build initial bindings from @Match parameters
    Map<String, DesiredNode> bindings = new LinkedHashMap<>();
    String[] paramNames = getParameterNames(rule.method());
    int matchIdx = 0;
    List<Object> args = new ArrayList<>();

    for (int i = 0; i < patterns.size(); i++) {
        PatternParameterDescriptor p = patterns.get(i);
        if (p.kind() == PatternKind.MATCH) {
            DesiredNode node = matchTuple.get(matchIdx++);
            bindings.put(paramNames[i], node);
            args.add(node);
        } else {
            args.add(null); // placeholder — filled during chain walk
        }
    }

    // Phase 3: walk chain for non-@Match parameters, expanding set-producing joins
    expandChain(rule, graph, patterns, paramNames, bindings, args, 0, matchIdx, allMutations);
}

private void expandChain(ResolvedGraphRule rule, DesiredStateGraph graph,
        List<PatternParameterDescriptor> patterns, String[] paramNames,
        Map<String, DesiredNode> bindings, List<Object> args,
        int paramIndex, int nextMatchIdx, List<GraphMutation> allMutations) {

    // Find next non-Match parameter
    int idx = paramIndex;
    while (idx < patterns.size() && patterns.get(idx).kind() == PatternKind.MATCH) {
        idx++;
    }
    if (idx >= patterns.size()) {
        // All parameters bound — invoke the rule
        invokeRule(rule, args, allMutations);
        return;
    }

    PatternParameterDescriptor p = patterns.get(idx);
    DesiredNode refNode = resolveReference(p, idx, paramNames, bindings);

    switch (p.kind()) {
        case DIRECT_DEP -> {
            List<DesiredNode> neighbors = findDirectNeighbors(graph, refNode, p);
            if (neighbors.isEmpty()) return; // discard tuple
            for (DesiredNode neighbor : neighbors) {
                var newBindings = new LinkedHashMap<>(bindings);
                var newArgs = new ArrayList<>(args);
                newBindings.put(paramNames[idx], neighbor);
                newArgs.set(idx, neighbor);
                expandChain(rule, graph, patterns, paramNames, newBindings, newArgs,
                        idx + 1, nextMatchIdx, allMutations);
            }
        }
        case REACHES -> {
            List<DesiredNode> reachable = findReachable(graph, refNode, p);
            if (reachable.isEmpty()) return;
            for (DesiredNode reached : reachable) {
                var newBindings = new LinkedHashMap<>(bindings);
                var newArgs = new ArrayList<>(args);
                newBindings.put(paramNames[idx], reached);
                newArgs.set(idx, reached);
                expandChain(rule, graph, patterns, paramNames, newBindings, newArgs,
                        idx + 1, nextMatchIdx, allMutations);
            }
        }
        case NOT_EXISTS -> {
            boolean exists = p.of().isEmpty()
                    ? existsGlobal(graph, p)
                    : existsRelational(graph, bindings.get(p.of()), p);
            if (exists) return; // discard tuple
            var newArgs = new ArrayList<>(args);
            newArgs.set(idx, null); // Void parameter
            expandChain(rule, graph, patterns, paramNames, bindings, newArgs,
                    idx + 1, nextMatchIdx, allMutations);
        }
        default -> throw new IllegalStateException("Unexpected pattern kind: " + p.kind());
    }
}

private DesiredNode resolveReference(PatternParameterDescriptor p, int paramIndex,
        String[] paramNames, Map<String, DesiredNode> bindings) {
    if (!p.of().isEmpty()) {
        return bindings.get(p.of());
    }
    // Sequential chaining: find previous bound parameter
    for (int i = paramIndex - 1; i >= 0; i--) {
        DesiredNode prev = bindings.get(paramNames[i]);
        if (prev != null) return prev;
    }
    return null;
}

private List<DesiredNode> findDirectNeighbors(DesiredStateGraph graph,
        DesiredNode refNode, PatternParameterDescriptor p) {
    NodeType targetType = NodeType.of(p.nodeType());
    Set<NodeId> neighbors = p.direction() == Direction.DEPENDENCIES
            ? graph.dependenciesOf(refNode.id())
            : graph.dependentsOf(refNode.id());
    return neighbors.stream()
            .map(id -> graph.nodes().get(id))
            .filter(n -> n != null && n.type().equals(targetType))
            .toList();
}

private List<DesiredNode> findReachable(DesiredStateGraph graph,
        DesiredNode refNode, PatternParameterDescriptor p) {
    NodeType targetType = NodeType.of(p.nodeType());
    List<DesiredNode> found = new ArrayList<>();
    Set<NodeId> visited = new HashSet<>();
    java.util.ArrayDeque<NodeId> queue = new java.util.ArrayDeque<>();
    queue.add(refNode.id());
    visited.add(refNode.id());

    while (!queue.isEmpty()) {
        NodeId current = queue.poll();
        Set<NodeId> neighbors = p.direction() == Direction.DEPENDENCIES
                ? graph.dependenciesOf(current)
                : graph.dependentsOf(current);
        for (NodeId neighbor : neighbors) {
            if (visited.add(neighbor)) {
                DesiredNode node = graph.nodes().get(neighbor);
                if (node != null && node.type().equals(targetType)) {
                    found.add(node);
                }
                queue.add(neighbor);
            }
        }
    }
    return found;
}

private boolean existsGlobal(DesiredStateGraph graph, PatternParameterDescriptor p) {
    NodeType targetType = NodeType.of(p.nodeType());
    return graph.nodes().values().stream().anyMatch(n -> n.type().equals(targetType));
}

private boolean existsRelational(DesiredStateGraph graph, DesiredNode refNode,
        PatternParameterDescriptor p) {
    NodeType targetType = NodeType.of(p.nodeType());
    Set<NodeId> neighbors = p.direction() == Direction.DEPENDENCIES
            ? graph.dependenciesOf(refNode.id())
            : graph.dependentsOf(refNode.id());
    return neighbors.stream()
            .map(id -> graph.nodes().get(id))
            .anyMatch(n -> n != null && n.type().equals(targetType));
}

@SuppressWarnings("unchecked")
private void invokeRule(ResolvedGraphRule rule, List<Object> args,
        List<GraphMutation> allMutations) {
    try {
        var result = (List<GraphMutation>) rule.method().invoke(rule.instance(), args.toArray());
        if (result != null && !result.isEmpty()) {
            allMutations.addAll(result);
        }
    } catch (InvocationTargetException e) {
        throw new RuntimeException("Rule " + rule.name() + " failed", e.getCause());
    } catch (IllegalAccessException e) {
        throw new RuntimeException("Rule " + rule.name() + " inaccessible", e);
    }
}

private String[] getParameterNames(Method method) {
    var params = method.getParameters();
    String[] names = new String[params.length];
    for (int i = 0; i < params.length; i++) {
        names[i] = params[i].getName();
    }
    return names;
}

private List<List<DesiredNode>> crossProduct(List<List<DesiredNode>> sets) {
    List<List<DesiredNode>> result = new ArrayList<>();
    result.add(List.of());
    for (List<DesiredNode> set : sets) {
        List<List<DesiredNode>> newResult = new ArrayList<>();
        for (List<DesiredNode> existing : result) {
            for (DesiredNode item : set) {
                List<DesiredNode> combined = new ArrayList<>(existing);
                combined.add(item);
                newResult.add(combined);
            }
        }
        result = newResult;
    }
    return result;
}
```

Note: `-parameters` is already configured in `casehub-parent` POM — no POM changes needed.

- [ ] **Step 4: Run tests to verify all pass**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=GraphRuleEngineTest`
Expected: All tests PASS (imperative + parameterized).

- [ ] **Step 5: Commit**

```bash
git add annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngine.java \
       annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngineTest.java
git commit -m "feat(#106): parameterized pattern matching — @Match, @DirectDep, @Reaches, @NotExists"
```

---

## Batch 3: Build-time integration — processor, validation, recorder, example

### Task 5: Processor scanning + validation + recorder + pipeline-annotated example

**Files:**
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java` — scan @GraphRule methods, standalone class discovery, wildcard matching, recorder invocation
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java` — all validation checks from spec Part 6
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java` — resolve rules and invoke engine
- Modify: `examples/pipeline-annotated/src/main/java/io/casehub/desiredstate/example/pipeline/annotated/MedallionPipeline.java` — add @GraphRule example
- Create: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/GraphRuleProcessorTest.java`
- Create: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/GraphRuleValidationTest.java`
- Modify: `examples/pipeline-annotated/src/test/java/io/casehub/desiredstate/example/pipeline/annotated/MedallionPipelineTest.java` — verify rule-generated nodes

**Interfaces:**
- Consumes: All Task 1-4 types. `DesiredStateAnnotationsProcessor.buildGraphDescriptor()`, `AnnotationValidationStep.validate()`, `DesiredStateGraphRecorder.createGoalCompiler()`, `GraphRuleEngine.evaluate()`
- Produces: End-to-end @GraphRule working in Quarkus apps

- [ ] **Step 1: Write GraphRuleProcessorTest**

```java
package io.casehub.desiredstate.annotations.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.desiredstate.annotations.DesiredState;
import io.casehub.desiredstate.annotations.DependsOn;
import io.casehub.desiredstate.annotations.GraphRule;
import io.casehub.desiredstate.annotations.Match;
import io.casehub.desiredstate.annotations.Node;
import io.casehub.desiredstate.annotations.NotExists;
import io.casehub.desiredstate.annotations.runtime.Direction;
import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.GraphMutation;
import io.casehub.desiredstate.api.GraphMutations;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.quarkus.test.QuarkusUnitTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import java.util.List;

class GraphRuleProcessorTest {

    @RegisterExtension
    static final QuarkusUnitTest test = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(
                    RuleGraph.class, TxSpec.class, SinkSpec.class, MonitorSpec.class))
            .overrideConfigKey("quarkus.arc.exclude-types",
                    "io.casehub.desiredstate.runtime.**");

    public record TxSpec(String name) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("transformer"); }
    }

    public record SinkSpec(String name) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("sink"); }
    }

    public record MonitorSpec(String target) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("monitor"); }
    }

    @DesiredState(namespace = "test", name = "rules")
    public interface RuleGraph {
        @Node("tx1") default TxSpec tx1() { return new TxSpec("tx1"); }
        @Node("sink1") @DependsOn("tx1") default SinkSpec sink1() { return new SinkSpec("sink1"); }

        @GraphRule
        static List<GraphMutation> ensureMonitoring(
                @Match(type = "sink") DesiredNode sink,
                @NotExists(type = "monitor", of = "sink", direction = Direction.DEPENDENTS) Void guard) {
            return GraphMutations.addNodeDependingOn(
                    new DesiredNode(NodeId.of("monitor-" + sink.id().value()),
                            new MonitorSpec(sink.id().value()), HumanGating.NONE),
                    sink.id());
        }
    }

    @SuppressWarnings("unchecked")
    @Inject GoalCompiler compiler;

    private final DefaultDesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();

    @Test
    void graphRuleAddsNodesAtCompileTime() {
        CompilationResult result = compiler.compile(null, factory);
        DesiredStateGraph graph = ((CompilationResult.SingleGraph) result).graph();
        assertThat(graph.nodes()).containsKey(NodeId.of("monitor-sink1"));
        assertThat(graph.nodes().get(NodeId.of("monitor-sink1")).type())
                .isEqualTo(NodeType.of("monitor"));
    }

    @Test
    void graphRuleCreatesCorrectDependency() {
        CompilationResult result = compiler.compile(null, factory);
        DesiredStateGraph graph = ((CompilationResult.SingleGraph) result).graph();
        assertThat(graph.dependenciesOf(NodeId.of("monitor-sink1")))
                .contains(NodeId.of("sink1"));
    }

    @Test
    void imperativeRuleOnInterfaceWorks() {
        // The rule above is parameterized — just verify it compiled and ran
        CompilationResult result = compiler.compile(null, factory);
        DesiredStateGraph graph = ((CompilationResult.SingleGraph) result).graph();
        assertThat(graph.nodes()).hasSize(3); // tx1, sink1, monitor-sink1
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=GraphRuleProcessorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — processor doesn't scan @GraphRule yet.

- [ ] **Step 3: Implement processor scanning**

In `DesiredStateAnnotationsProcessor.java`:

1. Add DotName constants for new annotations:
```java
private static final DotName GRAPH_RULE = DotName.createSimple(
        "io.casehub.desiredstate.annotations.GraphRule");
private static final DotName MATCH = DotName.createSimple(
        "io.casehub.desiredstate.annotations.Match");
private static final DotName DIRECT_DEP = DotName.createSimple(
        "io.casehub.desiredstate.annotations.DirectDep");
private static final DotName REACHES = DotName.createSimple(
        "io.casehub.desiredstate.annotations.Reaches");
private static final DotName NOT_EXISTS = DotName.createSimple(
        "io.casehub.desiredstate.annotations.NotExists");
```

2. In `buildGraphDescriptor()`, after the @GoalMethod scan, add @GraphRule scanning:
```java
List<GraphRuleDescriptor> graphRules = new ArrayList<>();
for (MethodInfo method : dsClass.methods()) {
    if (method.hasAnnotation(GRAPH_RULE) && isStaticMethod(method)) {
        graphRules.add(buildGraphRuleDescriptor(method, index));
    }
}
```

3. Add standalone class scanning method `scanStandaloneGraphRules(IndexView)` — iterates `index.getAnnotations(GRAPH_RULE)` for CLASS targets, collects methods annotated @GraphRule.

4. In `generateDesiredStateGraphs()`, after building interface graph descriptors, merge standalone rules by matching graph patterns (exact and wildcard).

5. Pass `graphRules` (merged) into the `GraphDescriptor` constructor.

6. Add `buildGraphRuleDescriptor(MethodInfo, IndexView)` method that inspects parameters for @Match/@DirectDep/@Reaches/@NotExists and builds `PatternParameterDescriptor` for each.

- [ ] **Step 4: Implement recorder integration**

In `DesiredStateGraphRecorder.createGoalCompiler()`, after the graph customizer loop and before the return statement:

```java
if (!descriptor.graphRules().isEmpty()) {
    List<ResolvedGraphRule> resolved = resolveRules(
            descriptor.graphRules(), implClass, instance);
    GraphRuleEngine engine = new GraphRuleEngine();
    graph = engine.evaluate(graph, resolved);
}
```

Add `resolveRules()` method that reflects @GraphRule methods and builds `ResolvedGraphRule` instances. For standalone classes: instantiate via `classLoader.loadClass(sourceClassName).getDeclaredConstructor().newInstance()`.

For the @GoalMethod path, add the same rule evaluation inside the GoalCompiler lambda, after the goal method call.

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=GraphRuleProcessorTest`
Expected: All 3 tests PASS.

- [ ] **Step 6: Write GraphRuleValidationTest**

```java
package io.casehub.desiredstate.annotations.deployment;

import io.casehub.desiredstate.annotations.DesiredState;
import io.casehub.desiredstate.annotations.GraphRule;
import io.casehub.desiredstate.annotations.Match;
import io.casehub.desiredstate.annotations.Node;
import io.casehub.desiredstate.annotations.NotExists;
import io.casehub.desiredstate.annotations.runtime.Direction;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.GraphMutation;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.quarkus.test.QuarkusUnitTest;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.fail;

class GraphRuleValidationTest {

    public record Spec() implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("t"); }
    }

    // --- Non-static interface method ---

    @DesiredState(namespace = "v", name = "nonstatic")
    public interface NonStaticRule {
        @Node("a") default Spec a() { return new Spec(); }
        @GraphRule
        default List<GraphMutation> badRule(DesiredStateGraph graph) { return List.of(); }
    }

    @RegisterExtension
    static final QuarkusUnitTest nonStaticTest = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(NonStaticRule.class, Spec.class))
            .overrideConfigKey("quarkus.arc.exclude-types", "io.casehub.desiredstate.runtime.**")
            .assertException(t -> assertThat(t).hasMessageContaining("must be a static method"));

    @Test
    void nonStaticInterfaceRuleFails() {}

    // --- Wrong return type ---

    @DesiredState(namespace = "v", name = "wrongreturn")
    public interface WrongReturnRule {
        @Node("a") default Spec a() { return new Spec(); }
        @GraphRule
        static String badRule(DesiredStateGraph graph) { return ""; }
    }

    @RegisterExtension
    static final QuarkusUnitTest wrongReturnTest = new QuarkusUnitTest()
            .withApplicationRoot(root -> root.addClasses(WrongReturnRule.class, Spec.class))
            .overrideConfigKey("quarkus.arc.exclude-types", "io.casehub.desiredstate.runtime.**")
            .assertException(t -> assertThat(t).hasMessageContaining("must return List<GraphMutation>"));

    @Test
    void wrongReturnTypeFails() {}
}
```

Note: Each negative test requires its own `@RegisterExtension` and test class because `assertException` fails the entire deployment. Use separate inner interfaces per validation scenario. Full set of negative tests covers: non-static, wrong return type, `of` references unknown param, @NotExists with `of` but omitted direction, non-public standalone method, non-instantiable standalone class, missing graph attribute on standalone.

- [ ] **Step 7: Implement validation in AnnotationValidationStep**

Add validation checks to `AnnotationValidationStep.validate()`:

1. For each @DesiredState interface, scan static methods with @GraphRule:
   - Must be static → error if not
   - Must return `List` (Jandex check: `method.returnType().name().equals(DotName.createSimple("java.util.List"))`) → error if not
   - Imperative: first param must be DesiredStateGraph
   - Parameterized: first non-@Match param with no `of` must have a preceding param → error
   - @NotExists with `of` but no explicit direction → error
   - `of` references validate against parameter names

2. For standalone @GraphRule classes:
   - Must be concrete, not abstract, not interface
   - Must have no-arg constructor
   - Must have non-empty `graph` attribute
   - Methods: must be public

- [ ] **Step 8: Run validation tests**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=GraphRuleValidationTest`
Expected: All negative tests PASS (each correctly rejects invalid configuration).

- [ ] **Step 9: Add @GraphRule to pipeline-annotated example**

In `MedallionPipeline.java`, add a monitoring rule:
```java
@GraphRule
static List<GraphMutation> ensureMonitoring(
        @Match(type = "sink") DesiredNode sink,
        @NotExists(type = "monitor", of = "sink", direction = Direction.DEPENDENTS) Void guard) {
    return GraphMutations.addNodeDependingOn(
            new DesiredNode(NodeId.of("monitor-" + sink.id().value()),
                    new MonitorSpec(sink.id().value()), HumanGating.NONE),
            sink.id());
}
```

Create `MonitorSpec` record in the pipeline example:
```java
public record MonitorSpec(String target) implements NodeSpec {
    @Override public NodeType nodeType() { return NodeType.of("monitor"); }
}
```

Update `MedallionPipelineTest.java` to verify the rule-generated node:
```java
@Test
void graphRuleAddsMonitorNode() {
    DesiredStateGraph graph = compileSingleGraph();
    assertThat(graph.nodes()).containsKey(NodeId.of("monitor-warehouse-sink"));
    assertThat(graph.nodes().get(NodeId.of("monitor-warehouse-sink")).type())
            .isEqualTo(NodeType.of("monitor"));
}
```

- [ ] **Step 10: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules compile, all tests pass.

- [ ] **Step 11: Commit**

```bash
git add annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java \
       annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java \
       annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java \
       annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/GraphRuleProcessorTest.java \
       annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/GraphRuleValidationTest.java \
       examples/pipeline-annotated/src/main/java/io/casehub/desiredstate/example/pipeline/annotated/MedallionPipeline.java \
       examples/pipeline-annotated/src/main/java/io/casehub/desiredstate/example/pipeline/annotated/MonitorSpec.java \
       examples/pipeline-annotated/src/test/java/io/casehub/desiredstate/example/pipeline/annotated/MedallionPipelineTest.java
git commit -m "feat(#106): processor scanning, validation, recorder integration, pipeline-annotated example"
```

---

## References

- [2026-08-24-graph-rule-design.md](/Users/mdproctor/claude/public/casehub-desiredstate/specs/issue-106-graph-rule/2026-08-24-graph-rule-design.md) — design spec this plan implements
- [DesiredStateAnnotationsProcessor.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java) — existing processor (modified in Task 2, 5)
- [AnnotationValidationStep.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java) — existing validation (modified in Task 5)
- [DesiredStateGraphRecorder.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java) — existing recorder (modified in Task 5)
- [GraphMutation.java](/Users/mdproctor/claude/casehub/desiredstate/api/src/main/java/io/casehub/desiredstate/api/GraphMutation.java) — sealed mutation types (modified in Task 2)
- [GraphDiff.java:80](/Users/mdproctor/claude/casehub/desiredstate/runtime/src/main/java/io/casehub/desiredstate/runtime/GraphDiff.java) — existing targetNodeId (delegated in Task 2)
- [GraphDescriptor.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphDescriptor.java) — IR record (extended in Task 2)
- [ConflictingMutationException.java](/Users/mdproctor/claude/casehub/desiredstate/api/src/main/java/io/casehub/desiredstate/api/ConflictingMutationException.java) — existing exception (reused)
- [ImmutableDesiredStateGraph.java](/Users/mdproctor/claude/casehub/desiredstate/runtime/src/main/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraph.java) — withMutation(), cycle detection
- [decisions.md](/Users/mdproctor/claude/public/casehub-desiredstate/specs/issue-106-graph-rule/decisions.md) — 10 design decisions
- [GitHub #106](https://github.com/casehubio/casehub-desiredstate/issues/106) — focal issue
