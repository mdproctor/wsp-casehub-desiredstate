# Graph Rules & Invariants Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #115 — @GraphRule standalone class discovery and wildcard graph matching
**Issue group:** #115, #113, #107

**Goal:** Extend the annotation infrastructure with standalone @GraphRule class discovery,
@Tier(nodeType) build-time validation, and @GraphInvariant declarative graph validation.

**Architecture:** Three extensions to the #106 annotation infrastructure: (1) standalone
@GraphRule classes with include/exclude graph matching, (2) optional @Tier(nodeType) for
build-time validation eliminating runtime probes, (3) @GraphInvariant with a separate
engine using universal quantification. Shared pattern matching primitives are extracted
to a utility class.

**Tech Stack:** Java 21, Quarkus build extensions (Jandex, Gizmo, @Recorder), JUnit 5

## Global Constraints

- All new types in `annotations/runtime/` module (`io.casehub.desiredstate.annotations.runtime` package) unless stated otherwise
- Annotations in `io.casehub.desiredstate.annotations` package (same module, different package)
- Tests use `DefaultDesiredStateGraphFactory` for graph construction
- Processor tests use `@QuarkusTest` in the deployment module
- Build extension tests go in `annotations/deployment/src/test/`
- Follow existing record-based IR pattern (GraphRuleDescriptor, ResolvedGraphRule, etc.)
- `PatternParameterDescriptor`, `PatternKind`, `Direction` are reused across rules and invariants

---

## Batch 1: Foundation — Extract Shared Pattern Matching + @Tier(nodeType)

### Task 1: Extract PatternMatchingSupport from GraphRuleEngine

**Files:**
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/PatternMatchingSupport.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngine.java`
- Test: `annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngineTest.java` (existing — must still pass)

**Interfaces:**
- Produces: `PatternMatchingSupport` with static methods: `resolveReference(...)`, `findDirectNeighbors(...)`, `findReachable(...)`, `existsGlobal(...)`, `existsRelational(...)`, `getParameterNames(Method)`, `crossProduct(List<List<DesiredNode>>)`

- [ ] **Step 1: Create PatternMatchingSupport with extracted methods**

Create the utility class with the 7 stateless methods extracted from GraphRuleEngine. These are pure functions of graph state — no engine-specific logic.

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.NodeType;
import java.lang.reflect.Method;
import java.lang.reflect.Parameter;
import java.util.*;

public final class PatternMatchingSupport {

    private PatternMatchingSupport() {}

    public static DesiredNode resolveReference(PatternParameterDescriptor p, int paramIndex,
            String[] paramNames, Map<String, DesiredNode> bindings) {
        String ref = p.of().isEmpty()
                ? (paramIndex > 0 ? paramNames[paramIndex - 1] : null)
                : p.of();
        if (ref == null || !bindings.containsKey(ref)) {
            throw new IllegalStateException("Cannot resolve binding reference for parameter at index "
                    + paramIndex + " — no preceding binding or named reference '" + p.of() + "'");
        }
        return bindings.get(ref);
    }

    public static List<DesiredNode> findDirectNeighbors(DesiredStateGraph graph,
            DesiredNode refNode, PatternParameterDescriptor p) {
        NodeType targetType = NodeType.of(p.nodeType());
        var nodeIds = p.direction() == Direction.DEPENDENCIES
                ? graph.dependenciesOf(refNode.id())
                : graph.dependentsOf(refNode.id());
        return nodeIds.stream()
                .filter(id -> graph.nodes().containsKey(id))
                .map(id -> graph.nodes().get(id))
                .filter(n -> n.type().equals(targetType))
                .toList();
    }

    public static List<DesiredNode> findReachable(DesiredStateGraph graph,
            DesiredNode refNode, PatternParameterDescriptor p) {
        NodeType targetType = NodeType.of(p.nodeType());
        List<DesiredNode> reachable = new ArrayList<>();
        Set<io.casehub.desiredstate.api.NodeId> visited = new HashSet<>();
        Deque<io.casehub.desiredstate.api.NodeId> queue = new ArrayDeque<>();
        queue.add(refNode.id());
        visited.add(refNode.id());
        while (!queue.isEmpty()) {
            var current = queue.poll();
            var neighbors = p.direction() == Direction.DEPENDENCIES
                    ? graph.dependenciesOf(current)
                    : graph.dependentsOf(current);
            for (var neighbor : neighbors) {
                if (visited.add(neighbor) && graph.nodes().containsKey(neighbor)) {
                    DesiredNode node = graph.nodes().get(neighbor);
                    if (node.type().equals(targetType)) {
                        reachable.add(node);
                    }
                    queue.add(neighbor);
                }
            }
        }
        return reachable;
    }

    public static boolean existsGlobal(DesiredStateGraph graph, PatternParameterDescriptor p) {
        NodeType targetType = NodeType.of(p.nodeType());
        return graph.nodes().values().stream().anyMatch(n -> n.type().equals(targetType));
    }

    public static boolean existsRelational(DesiredStateGraph graph, DesiredNode refNode,
            PatternParameterDescriptor p) {
        NodeType targetType = NodeType.of(p.nodeType());
        var nodeIds = p.direction() == Direction.DEPENDENCIES
                ? graph.dependenciesOf(refNode.id())
                : graph.dependentsOf(refNode.id());
        return nodeIds.stream()
                .filter(id -> graph.nodes().containsKey(id))
                .map(id -> graph.nodes().get(id))
                .anyMatch(n -> n.type().equals(targetType));
    }

    public static String[] getParameterNames(Method method) {
        Parameter[] params = method.getParameters();
        String[] names = new String[params.length];
        for (int i = 0; i < params.length; i++) {
            names[i] = params[i].getName();
        }
        return names;
    }

    public static List<List<DesiredNode>> crossProduct(List<List<DesiredNode>> sets) {
        List<List<DesiredNode>> result = new ArrayList<>();
        result.add(new ArrayList<>());
        for (List<DesiredNode> set : sets) {
            List<List<DesiredNode>> newResult = new ArrayList<>();
            for (List<DesiredNode> existing : result) {
                for (DesiredNode item : set) {
                    List<DesiredNode> extended = new ArrayList<>(existing);
                    extended.add(item);
                    newResult.add(extended);
                }
            }
            result = newResult;
        }
        return result;
    }
}
```

- [ ] **Step 2: Refactor GraphRuleEngine to delegate to PatternMatchingSupport**

Replace the 7 private methods in GraphRuleEngine with calls to PatternMatchingSupport static methods. The engine keeps its evaluation logic (expandBindings, expandChain, invokeRule, evaluate, detectConflicts, etc.) — only the stateless primitives move.

In GraphRuleEngine, change each call site:
- `resolveReference(...)` → `PatternMatchingSupport.resolveReference(...)`
- `findDirectNeighbors(...)` → `PatternMatchingSupport.findDirectNeighbors(...)`
- `findReachable(...)` → `PatternMatchingSupport.findReachable(...)`
- `existsGlobal(...)` → `PatternMatchingSupport.existsGlobal(...)`
- `existsRelational(...)` → `PatternMatchingSupport.existsRelational(...)`
- `getParameterNames(...)` → `PatternMatchingSupport.getParameterNames(...)`
- `crossProduct(...)` → `PatternMatchingSupport.crossProduct(...)`

Remove the 7 private methods from GraphRuleEngine.

- [ ] **Step 3: Run existing GraphRuleEngineTest**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=GraphRuleEngineTest`
Expected: All existing tests PASS — the refactoring is behavior-preserving.

- [ ] **Step 4: Run full annotations module tests**

Run: `mvn --batch-mode test -pl annotations/runtime`
Expected: All tests PASS.

- [ ] **Step 5: Commit**

```
feat(#115): extract PatternMatchingSupport from GraphRuleEngine Refs #115

Shared stateless methods (resolveReference, findDirectNeighbors,
findReachable, existsGlobal, existsRelational, getParameterNames,
crossProduct) extracted for reuse by GraphInvariantEngine.
```

---

### Task 2: @Tier(nodeType) — Annotation, Descriptor, Validation, Recorder

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/Tier.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/TierDescriptor.java`
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java`
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java`
- Test: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/TierNodeTypeValidationTest.java`

**Interfaces:**
- Consumes: `TierDescriptor(int threshold, String reviewMethodName)` (existing)
- Produces: `TierDescriptor(int threshold, String reviewMethodName, String nodeType)` (extended)

- [ ] **Step 1: Write failing test — @Tier with nodeType compiles successfully**

Create `TierNodeTypeValidationTest.java` in the deployment test directory. Use the existing Quarkus test pattern from `GraphRuleValidationTest.java`.

```java
package io.casehub.desiredstate.annotations.deployment;

import io.quarkus.test.QuarkusUnitTest;
import org.jboss.shrinkwrap.api.asset.StringAsset;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import static org.junit.jupiter.api.Assertions.*;

class TierNodeTypeValidationTest {

    @RegisterExtension
    static final QuarkusUnitTest config = new QuarkusUnitTest()
            .withApplicationRoot(root -> root
                    .addClasses(TierNodeTypeTestApp.class,
                            TierNodeTypeTestApp.ValidatorSpec.class,
                            TierNodeTypeTestApp.AiReviewSpec.class)
                    .addAsResource(new StringAsset(""), "application.properties"));

    @Test
    void tierWithNodeTypeCompiles() {
        // If we get here, the build step accepted the @Tier(nodeType) attribute
        assertTrue(true);
    }
}
```

Create the test app class with @Tier(nodeType):

```java
package io.casehub.desiredstate.annotations.deployment;

import io.casehub.desiredstate.annotations.*;
import io.casehub.desiredstate.api.*;

@DesiredState(namespace = "test", name = "tier-nodetype")
interface TierNodeTypeTestApp {
    record ValidatorSpec(String name) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("validator"); }
    }
    record AiReviewSpec(String detail) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("ai-review"); }
    }

    @Node("v1")
    @FaultPolicyDef(faultTypes = "PROVISION_FAILED", tiers = {
            @Tier(threshold = 3, review = "createReview", nodeType = "ai-review")
    })
    default ValidatorSpec v1() { return new ValidatorSpec("v"); }

    default AiReviewSpec createReview(FaultEvent event, DesiredStateGraph graph) {
        return new AiReviewSpec("review");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=TierNodeTypeValidationTest`
Expected: Compilation error — `nodeType` attribute not found on `@Tier`.

- [ ] **Step 3: Add nodeType attribute to @Tier annotation**

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Tier {
    int threshold();
    String review();
    String nodeType() default "";
}
```

- [ ] **Step 4: Update TierDescriptor record**

```java
public record TierDescriptor(int threshold, String reviewMethodName, String nodeType) {}
```

- [ ] **Step 5: Update processor — read nodeType from @Tier annotation**

In `DesiredStateAnnotationsProcessor.buildFaultPolicyDescriptor()`, read the nodeType value from the tier annotation and pass it to TierDescriptor. Find all construction sites of TierDescriptor and add the third argument.

Search for `new TierDescriptor(` to find all construction sites. Add `stringValueOrDefault(tierAnn, index, "nodeType", "")` as the third argument.

- [ ] **Step 6: Update recorder — use nodeType when present**

In `DesiredStateGraphRecorder.createFaultPolicy()`, at the tier processing loop (line ~194), when `td.nodeType()` is non-empty, override the `ReviewSpecFactory.nodeType()`:

```java
for (TierDescriptor td : descriptor.tiers()) {
    Method reviewMethod = implClass.getMethod(td.reviewMethodName(),
            FaultEvent.class, DesiredStateGraph.class);
    io.casehub.desiredstate.api.ReviewSpecFactory factory = (event, graph) -> {
        try {
            return (NodeSpec) reviewMethod.invoke(instance, event, graph);
        } catch (Exception e) {
            throw new RuntimeException("Review method invocation failed: "
                    + reviewMethod.getName(), e);
        }
    };
    if (!td.nodeType().isEmpty()) {
        NodeType declaredType = NodeType.of(td.nodeType());
        factory = new io.casehub.desiredstate.api.ReviewSpecFactory() {
            private final io.casehub.desiredstate.api.ReviewSpecFactory delegate = (event, graph) -> {
                try {
                    return (NodeSpec) reviewMethod.invoke(instance, event, graph);
                } catch (Exception e) {
                    throw new RuntimeException("Review method invocation failed: "
                            + reviewMethod.getName(), e);
                }
            };
            @Override
            public NodeSpec create(FaultEvent event, DesiredStateGraph graph) {
                return delegate.create(event, graph);
            }
            @Override
            public NodeType nodeType() { return declaredType; }
        };
    }
    builder.tier(td.threshold(),
            io.casehub.desiredstate.api.FaultPolicy.addReviewNode(factory));
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=TierNodeTypeValidationTest`
Expected: PASS

- [ ] **Step 8: Run full deployment module tests**

Run: `mvn --batch-mode test -pl annotations/deployment`
Expected: All tests PASS (existing tests unaffected by optional attribute with default).

- [ ] **Step 9: Commit**

```
feat(#113): @Tier(nodeType) attribute for build-time validation Refs #113

Optional nodeType on @Tier bypasses the runtime ReviewSpecFactory probe.
When present, the recorder uses the declared type directly.
```

---

## Batch 2: Standalone @GraphRule Discovery (#115)

### Task 3: Graph Matching Utility + Standalone Processor Scanning

**Files:**
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphPatternMatcher.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/GraphRule.java`
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java`
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java`
- Test: `annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/GraphPatternMatcherTest.java`
- Test: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/StandaloneGraphRuleTest.java`

**Interfaces:**
- Produces: `GraphPatternMatcher.matches(String[] patterns, String graphKey) → boolean`

- [ ] **Step 1: Write failing test — GraphPatternMatcher**

```java
package io.casehub.desiredstate.annotations.runtime;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class GraphPatternMatcherTest {

    @Test
    void exactMatch() {
        assertTrue(GraphPatternMatcher.matches(
                new String[]{"pipeline:medallion"}, "pipeline:medallion"));
        assertFalse(GraphPatternMatcher.matches(
                new String[]{"pipeline:medallion"}, "pipeline:batch"));
    }

    @Test
    void namespaceWildcard() {
        assertTrue(GraphPatternMatcher.matches(
                new String[]{"pipeline:*"}, "pipeline:medallion"));
        assertTrue(GraphPatternMatcher.matches(
                new String[]{"pipeline:*"}, "pipeline:batch"));
        assertFalse(GraphPatternMatcher.matches(
                new String[]{"pipeline:*"}, "analytics:report"));
    }

    @Test
    void globalWildcard() {
        assertTrue(GraphPatternMatcher.matches(
                new String[]{"*:*"}, "pipeline:medallion"));
        assertTrue(GraphPatternMatcher.matches(
                new String[]{"*:*"}, "analytics:report"));
    }

    @Test
    void excludePattern() {
        assertTrue(GraphPatternMatcher.matches(
                new String[]{"*:*", "!debug:*"}, "pipeline:medallion"));
        assertFalse(GraphPatternMatcher.matches(
                new String[]{"*:*", "!debug:*"}, "debug:trace"));
    }

    @Test
    void reIncludeAfterExclude() {
        var patterns = new String[]{"*:*", "!internal:*", "internal:monitoring"};
        assertTrue(GraphPatternMatcher.matches(patterns, "pipeline:medallion"));
        assertFalse(GraphPatternMatcher.matches(patterns, "internal:debug"));
        assertTrue(GraphPatternMatcher.matches(patterns, "internal:monitoring"));
    }

    @Test
    void lastMatchWins() {
        var patterns = new String[]{"pipeline:*", "!pipeline:debug", "pipeline:debug"};
        assertTrue(GraphPatternMatcher.matches(patterns, "pipeline:debug"));
    }

    @Test
    void emptyPatternsMatchNothing() {
        assertFalse(GraphPatternMatcher.matches(new String[]{}, "pipeline:medallion"));
    }

    @Test
    void multipleIncludes() {
        var patterns = new String[]{"pipeline:*", "analytics:*"};
        assertTrue(GraphPatternMatcher.matches(patterns, "pipeline:medallion"));
        assertTrue(GraphPatternMatcher.matches(patterns, "analytics:report"));
        assertFalse(GraphPatternMatcher.matches(patterns, "debug:trace"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=GraphPatternMatcherTest`
Expected: FAIL — class not found.

- [ ] **Step 3: Implement GraphPatternMatcher**

```java
package io.casehub.desiredstate.annotations.runtime;

public final class GraphPatternMatcher {

    private GraphPatternMatcher() {}

    public static boolean matches(String[] patterns, String graphKey) {
        boolean result = false;
        for (String pattern : patterns) {
            if (pattern.startsWith("!")) {
                if (matchesSingle(pattern.substring(1), graphKey)) {
                    result = false;
                }
            } else {
                if (matchesSingle(pattern, graphKey)) {
                    result = true;
                }
            }
        }
        return result;
    }

    private static boolean matchesSingle(String pattern, String key) {
        if ("*:*".equals(pattern)) return true;
        if (pattern.endsWith(":*")) {
            String namespace = pattern.substring(0, pattern.length() - 1);
            return key.startsWith(namespace);
        }
        return pattern.equals(key);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=GraphPatternMatcherTest`
Expected: All PASS.

- [ ] **Step 5: Change @GraphRule.graph() from String to String[]**

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface GraphRule {
    String[] graph() default {};
}
```

- [ ] **Step 6: Update processor to handle String[] graph attribute**

In `DesiredStateAnnotationsProcessor`, update all reads of the `graph` attribute. The existing interface path ignores the graph attribute (it's scoped to the declaring interface). The standalone scanning is new.

Find all `grAnn.value("graph")` reads and update to handle `String[]`. For the existing interface path in `buildGraphDescriptor`, `graph()` is ignored — no change needed there.

Add standalone class scanning to `generateDesiredStateGraphs()` after the interface loop and before class-only graph processing:

```java
// Scan standalone @GraphRule classes
Map<String[], List<GraphRuleDescriptor>> standaloneRulesByPattern = new LinkedHashMap<>();
for (AnnotationInstance grAnn : index.getAnnotations(GRAPH_RULE)) {
    if (grAnn.target().kind() != AnnotationTarget.Kind.CLASS) continue;
    ClassInfo classInfo = grAnn.target().asClass();
    AnnotationValue graphVal = grAnn.value("graph");
    if (graphVal == null) continue;
    String[] graphPatterns = graphVal.asStringArray();
    if (graphPatterns.length == 0) continue;

    List<GraphRuleDescriptor> classRules = new ArrayList<>();
    for (MethodInfo method : classInfo.methods()) {
        if (method.hasAnnotation(GRAPH_RULE)
                && !java.lang.reflect.Modifier.isStatic(method.flags())
                && java.lang.reflect.Modifier.isPublic(method.flags())) {
            classRules.add(buildGraphRuleDescriptor(method, index,
                    classInfo.name().toString()));
        }
    }
    if (!classRules.isEmpty()) {
        standaloneRulesByPattern.put(graphPatterns, classRules);
    }
}
```

Then, in the existing interface processing loop, after building the `GraphDescriptor`, merge matching standalone rules:

```java
// Merge standalone rules that match this graph
List<GraphRuleDescriptor> allRules = new ArrayList<>(descriptor.graphRules());
for (var entry : standaloneRulesByPattern.entrySet()) {
    if (GraphPatternMatcher.matches(entry.getKey(), graphKey)) {
        allRules.addAll(entry.getValue());
    }
}
if (allRules.size() != descriptor.graphRules().size()) {
    descriptor = new GraphDescriptor(descriptor.namespace(), descriptor.name(),
            descriptor.interfaceName(), descriptor.implClassName(),
            descriptor.nodes(), descriptor.dependencies(),
            descriptor.faultPolicies(), descriptor.goalMethod(), allRules);
}
```

- [ ] **Step 7: Add standalone validation to AnnotationValidationStep**

Add a new method `validateStandaloneGraphRules(IndexView index, Set<String> knownGraphKeys, List<String> errors, List<String> warnings)` and call it from `validate()`:

```java
private void validateStandaloneGraphRules(IndexView index,
        Set<String> knownGraphKeys, List<String> errors, List<String> warnings) {
    for (AnnotationInstance grAnn : index.getAnnotations(GRAPH_RULE)) {
        if (grAnn.target().kind() != AnnotationTarget.Kind.CLASS) continue;
        ClassInfo classInfo = grAnn.target().asClass();

        if (java.lang.reflect.Modifier.isAbstract(classInfo.flags())
                || java.lang.reflect.Modifier.isInterface(classInfo.flags())) {
            errors.add("@GraphRule class " + classInfo.name().local()
                    + " must be concrete with a no-arg constructor");
            continue;
        }

        boolean hasNoArgCtor = classInfo.constructors().stream()
                .anyMatch(c -> c.parametersCount() == 0);
        if (!hasNoArgCtor) {
            errors.add("@GraphRule class " + classInfo.name().local()
                    + " must be concrete with a no-arg constructor");
            continue;
        }

        AnnotationValue graphVal = grAnn.value("graph");
        if (graphVal == null || graphVal.asStringArray().length == 0) {
            errors.add("@GraphRule on class " + classInfo.name().local()
                    + " requires graph attribute");
            continue;
        }

        String[] patterns = graphVal.asStringArray();
        boolean hasInclude = false;
        for (String p : patterns) {
            if (!p.startsWith("!")) { hasInclude = true; break; }
        }
        if (!hasInclude) {
            errors.add("@GraphRule on class " + classInfo.name().local()
                    + " graph has no include patterns — at least one non-! entry required");
            continue;
        }

        boolean matchesAny = knownGraphKeys.stream()
                .anyMatch(k -> GraphPatternMatcher.matches(patterns, k));
        if (!matchesAny) {
            warnings.add("@GraphRule class " + classInfo.name().local()
                    + " graph '" + String.join(", ", patterns)
                    + "' does not match any declared graph");
        }

        for (MethodInfo method : classInfo.methods()) {
            if (!method.hasAnnotation(GRAPH_RULE)) continue;
            if (!java.lang.reflect.Modifier.isPublic(method.flags())) {
                errors.add("@GraphRule on '" + method.name() + "' in "
                        + classInfo.name().local() + " must be public");
            }
            if (!method.returnType().name().equals(JAVA_LIST)) {
                errors.add("@GraphRule '" + method.name() + "' in "
                        + classInfo.name().local() + " must return List<GraphMutation>");
            }
        }
    }
}
```

Pass `interfaceGraphKeys` (the `Set<String>` already built in the main validate loop) to this method.

- [ ] **Step 8: Write standalone processor integration test**

Create `StandaloneGraphRuleTest.java`:

```java
package io.casehub.desiredstate.annotations.deployment;

import io.casehub.desiredstate.annotations.*;
import io.casehub.desiredstate.annotations.runtime.Direction;
import io.casehub.desiredstate.api.*;
import io.quarkus.test.QuarkusUnitTest;
import jakarta.inject.Inject;
import org.jboss.shrinkwrap.api.asset.StringAsset;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class StandaloneGraphRuleTest {

    @RegisterExtension
    static final QuarkusUnitTest config = new QuarkusUnitTest()
            .withApplicationRoot(root -> root
                    .addClasses(TestGraph.class, TestGraph.SimpleSpec.class,
                            TestStandaloneRules.class)
                    .addAsResource(new StringAsset(""), "application.properties"));

    @Inject
    @DesiredStateQualifier(namespace = "test", name = "standalone-rule")
    GoalCompiler<Object> compiler;

    @Test
    void standaloneRuleFires() {
        var result = compiler.compile(null,
                new io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory());
        var graph = ((CompilationResult.SingleGraph) result).graph();
        assertTrue(graph.nodes().containsKey(NodeId.of("monitor-s1")),
                "Standalone rule should have added monitor node");
    }
}
```

With test classes:

```java
@DesiredState(namespace = "test", name = "standalone-rule")
interface TestGraph {
    record SimpleSpec(String name) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of(name); }
    }

    @Node("s1") default SimpleSpec sink() { return new SimpleSpec("sink"); }
}

@GraphRule(graph = "test:*")
class TestStandaloneRules {
    @GraphRule
    public List<GraphMutation> addMonitor(
            @Match(type = "sink") DesiredNode sink,
            @NotExists(type = "monitor", of = "sink",
                    direction = Direction.DEPENDENTS) Void guard) {
        return GraphMutations.addNodeDependingOn(
                new DesiredNode(NodeId.of("monitor-" + sink.id().value()),
                        new TestGraph.SimpleSpec("monitor"), HumanGating.NONE),
                sink.id());
    }
}
```

- [ ] **Step 9: Run test to verify it passes**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=StandaloneGraphRuleTest`
Expected: PASS.

- [ ] **Step 10: Run full build**

Run: `mvn --batch-mode test -pl annotations/runtime,annotations/deployment`
Expected: All tests PASS.

- [ ] **Step 11: Commit**

```
feat(#115): standalone @GraphRule class discovery with include/exclude matching Refs #115

GraphPatternMatcher implements ordered evaluation with ! prefix exclusions.
@GraphRule.graph() is now String[] supporting exact, namespace wildcard,
global wildcard, and exclusion patterns. Standalone classes scanned via
Jandex, matched to graphs, and merged into GraphDescriptor.
```

---

### Task 4: Complete @GraphRule Parameter Validation

**Files:**
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java`
- Modify: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/GraphRuleValidationTest.java`

**Interfaces:**
- Consumes: Existing `validateGraphRules(ClassInfo, List<String>)` method
- Produces: Extended validation covering parameter annotations

- [ ] **Step 1: Write failing tests for parameter validation**

Add test cases to `GraphRuleValidationTest.java`:

```java
@RegisterExtension
static final QuarkusUnitTest notExistsNoDirection = new QuarkusUnitTest()
        .withApplicationRoot(root -> root
                .addClasses(NotExistsNoDirectionGraph.class,
                        NotExistsNoDirectionGraph.Spec.class)
                .addAsResource(new StringAsset(""), "application.properties"))
        .assertException(t -> assertTrue(t.getMessage()
                .contains("specifies 'of' without explicit direction")));

@Test
void notExistsWithOfButNoDirection() {}

// Test app: @NotExists(type = "monitor", of = "sink") without direction
@DesiredState(namespace = "test", name = "bad-notexists")
interface NotExistsNoDirectionGraph {
    record Spec(String n) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of(n); }
    }
    @Node("s1") default Spec s1() { return new Spec("sink"); }

    @GraphRule
    static List<GraphMutation> bad(
            @Match(type = "sink") DesiredNode sink,
            @NotExists(type = "monitor", of = "sink") Void guard) {
        return List.of();
    }
}
```

Add similar tests for:
- `@DirectDep` as first parameter with no `of` → error
- `of` references unknown parameter name → error
- Imperative method with wrong first param type → error

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=GraphRuleValidationTest`
Expected: FAIL — validation not yet implemented.

- [ ] **Step 3: Extend validateGraphRules to check parameter annotations**

Replace the existing `validateGraphRules` with a version that walks parameters:

```java
private void validateGraphRules(ClassInfo dsClass, IndexView index,
        List<String> errors) {
    for (MethodInfo method : dsClass.methods()) {
        if (!method.hasAnnotation(GRAPH_RULE)) continue;

        if (!java.lang.reflect.Modifier.isStatic(method.flags())) {
            errors.add("@GraphRule on '" + method.name() + "' in "
                    + dsClass.name().local() + " must be a static method");
        }
        if (!method.returnType().name().equals(JAVA_LIST)) {
            errors.add("@GraphRule '" + method.name() + "' in "
                    + dsClass.name().local() + " must return List<GraphMutation>");
        }

        validatePatternParameters(method, dsClass.name().local(), index, errors);
    }
}
```

Add `validatePatternParameters` method (shared between @GraphRule and @GraphInvariant):

```java
private void validatePatternParameters(MethodInfo method, String className,
        IndexView index, List<String> errors) {
    if (method.parametersCount() == 1
            && method.parameterType(0).name().equals(DESIRED_STATE_GRAPH)) {
        return; // imperative — valid
    }
    if (method.parametersCount() == 0) return;

    // Check if it looks imperative but wrong type
    boolean hasPatternAnnotations = false;
    for (var ann : method.annotations()) {
        if (ann.target().kind() == AnnotationTarget.Kind.METHOD_PARAMETER) {
            DotName n = ann.name();
            if (n.equals(MATCH) || n.equals(DIRECT_DEP) || n.equals(REACHES)
                    || n.equals(NOT_EXISTS)) {
                hasPatternAnnotations = true;
                break;
            }
        }
    }
    if (!hasPatternAnnotations && method.parametersCount() == 1) {
        errors.add("@GraphRule '" + method.name() + "' imperative method "
                + "first parameter must be DesiredStateGraph");
        return;
    }

    Set<String> paramNames = new LinkedHashSet<>();
    String previousParamName = null;
    for (int i = 0; i < method.parametersCount(); i++) {
        String paramName = method.parameterName(i);
        paramNames.add(paramName);

        for (var ann : method.annotations()) {
            if (ann.target().kind() != AnnotationTarget.Kind.METHOD_PARAMETER) continue;
            if (ann.target().asMethodParameter().position() != i) continue;

            DotName annName = ann.name();

            if ((annName.equals(DIRECT_DEP) || annName.equals(REACHES))) {
                String of = stringValueOrDefault(ann, index, "of", "");
                if (of.isEmpty() && previousParamName == null) {
                    errors.add("@" + annName.local() + " on parameter '"
                            + paramName + "' uses sequential chaining but has no "
                            + "preceding parameter — use @Match as the first "
                            + "parameter or specify 'of' explicitly");
                }
                if (!of.isEmpty() && !paramNames.contains(of)) {
                    errors.add("@" + annName.local() + " 'of' references '"
                            + of + "' — no parameter named '" + of + "' in "
                            + method.name());
                }
            }

            if (annName.equals(NOT_EXISTS)) {
                String of = stringValueOrDefault(ann, index, "of", "");
                if (!of.isEmpty()) {
                    // Check direction is explicitly set (not default)
                    AnnotationValue dirVal = ann.value("direction");
                    if (dirVal == null) {
                        errors.add("@NotExists on parameter '" + paramName
                                + "' specifies 'of' without explicit direction "
                                + "— DEPENDENCIES and DEPENDENTS have opposite "
                                + "semantics; specify direction");
                    }
                    if (!paramNames.contains(of)) {
                        errors.add("@NotExists 'of' references '" + of
                                + "' — no parameter named '" + of + "' in "
                                + method.name());
                    }
                }
            }
        }
        previousParamName = paramName;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=GraphRuleValidationTest`
Expected: All PASS.

- [ ] **Step 5: Run full deployment tests**

Run: `mvn --batch-mode test -pl annotations/deployment`
Expected: All tests PASS.

- [ ] **Step 6: Commit**

```
feat(#115): complete @GraphRule parameter validation Refs #115

Validates: @NotExists with of but no direction, @DirectDep/@Reaches as
first param with no of, of references unknown parameter, imperative
method wrong first param type.
```

---

## Batch 3: @GraphInvariant (#107)

### Task 5: Annotation, IR Types, Exception Types

**Files:**
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/GraphInvariant.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphInvariantDescriptor.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/ResolvedGraphInvariant.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphViolation.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphViolationException.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphInvariantViolationsException.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphDescriptor.java`

**Interfaces:**
- Produces: `@GraphInvariant` annotation, `GraphInvariantDescriptor`, `ResolvedGraphInvariant`, `GraphViolation`, `GraphViolationException`, `GraphInvariantViolationsException`

- [ ] **Step 1: Create @GraphInvariant annotation**

```java
package io.casehub.desiredstate.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface GraphInvariant {
    String[] graph() default {};
}
```

- [ ] **Step 2: Create IR and exception types**

GraphInvariantDescriptor:
```java
package io.casehub.desiredstate.annotations.runtime;
import java.util.List;
public record GraphInvariantDescriptor(
        String methodName,
        boolean imperative,
        List<PatternParameterDescriptor> patterns,
        String sourceClassName) {}
```

ResolvedGraphInvariant:
```java
package io.casehub.desiredstate.annotations.runtime;
import java.lang.reflect.Method;
import java.util.List;
public record ResolvedGraphInvariant(
        String name,
        Method method,
        Object instance,
        boolean imperative,
        List<PatternParameterDescriptor> patterns) {}
```

GraphViolation:
```java
package io.casehub.desiredstate.annotations.runtime;
import io.casehub.desiredstate.api.NodeId;
import java.util.List;
public record GraphViolation(
        String invariantName,
        String sourceClassName,
        String message,
        List<NodeId> affectedNodes) {}
```

GraphViolationException:
```java
package io.casehub.desiredstate.annotations.runtime;
import io.casehub.desiredstate.api.NodeId;
import java.util.List;
public class GraphViolationException extends RuntimeException {
    private final List<NodeId> affectedNodes;
    public GraphViolationException(String message) {
        super(message);
        this.affectedNodes = List.of();
    }
    public GraphViolationException(String message, NodeId... nodes) {
        super(message);
        this.affectedNodes = List.of(nodes);
    }
    public List<NodeId> affectedNodes() { return affectedNodes; }
}
```

GraphInvariantViolationsException:
```java
package io.casehub.desiredstate.annotations.runtime;
import java.util.List;
public class GraphInvariantViolationsException extends RuntimeException {
    private final List<GraphViolation> violations;
    public GraphInvariantViolationsException(List<GraphViolation> violations) {
        super("Graph invariant violations:\n" + violations.stream()
                .map(v -> "  - " + v.invariantName() + ": " + v.message())
                .collect(java.util.stream.Collectors.joining("\n")));
        this.violations = List.copyOf(violations);
    }
    public List<GraphViolation> violations() { return violations; }
}
```

- [ ] **Step 3: Extend GraphDescriptor with graphInvariants**

```java
public record GraphDescriptor(
        String namespace,
        String name,
        String interfaceName,
        String implClassName,
        List<NodeDescriptor> nodes,
        List<DependencyDescriptor> dependencies,
        List<FaultPolicyDescriptor> faultPolicies,
        GoalMethodDescriptor goalMethod,
        List<GraphRuleDescriptor> graphRules,
        List<GraphInvariantDescriptor> graphInvariants) {}
```

Update all construction sites of GraphDescriptor (in processor, tests, and recorder) to add the 10th parameter `List.of()` for existing sites.

- [ ] **Step 4: Verify compilation**

Run: `mvn --batch-mode compile -pl annotations/runtime,annotations/deployment`
Expected: Compiles successfully.

- [ ] **Step 5: Commit**

```
feat(#107): @GraphInvariant annotation, IR types, exception types Refs #107

New annotation, GraphInvariantDescriptor, ResolvedGraphInvariant,
GraphViolation, GraphViolationException, GraphInvariantViolationsException.
GraphDescriptor extended with 10th component.
```

---

### Task 6: GraphInvariantEngine with Universal Quantification

**Files:**
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphInvariantEngine.java`
- Test: `annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/GraphInvariantEngineTest.java`

**Interfaces:**
- Consumes: `PatternMatchingSupport` (from Task 1), `ResolvedGraphInvariant`, `GraphViolation`, `GraphViolationException`, `GraphInvariantViolationsException`
- Produces: `GraphInvariantEngine.validate(DesiredStateGraph, List<ResolvedGraphInvariant>)`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import org.junit.jupiter.api.Test;
import java.lang.reflect.Method;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class GraphInvariantEngineTest {

    private final DefaultDesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();
    private final GraphInvariantEngine engine = new GraphInvariantEngine();

    record Spec(String name, String typeValue) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of(typeValue); }
    }

    // --- parameterized: structural violation when anchor fails to expand ---

    public static void sinkMustHaveUpstream(DesiredNode sink, DesiredNode upstream) {}

    @Test
    void parameterizedViolationWhenAnchorFailsToExpand() {
        // sink3 has no data-source dependency → structural violation
        var graph = factory.of(
                List.of(
                        new DesiredNode(NodeId.of("sink1"), new Spec("s1", "sink"), HumanGating.NONE),
                        new DesiredNode(NodeId.of("ds1"), new Spec("d1", "data-source"), HumanGating.NONE),
                        new DesiredNode(NodeId.of("sink3"), new Spec("s3", "sink"), HumanGating.NONE)),
                List.of(new Dependency(NodeId.of("sink1"), NodeId.of("ds1"))));

        var invariant = parameterizedInvariant("sinkMustHaveUpstream",
                List.of(
                        new PatternParameterDescriptor(PatternKind.MATCH, "sink", "", Direction.DEPENDENCIES),
                        new PatternParameterDescriptor(PatternKind.DIRECT_DEP, "data-source", "sink", Direction.DEPENDENCIES)));

        var ex = assertThrows(GraphInvariantViolationsException.class,
                () -> engine.validate(graph, List.of(invariant)));
        assertEquals(1, ex.violations().size());
        assertTrue(ex.violations().get(0).message().contains("sink3"));
    }

    // --- parameterized: passes when all anchors expand ---

    @Test
    void parameterizedPassesWhenAllAnchorsExpand() {
        var graph = factory.of(
                List.of(
                        new DesiredNode(NodeId.of("sink1"), new Spec("s1", "sink"), HumanGating.NONE),
                        new DesiredNode(NodeId.of("ds1"), new Spec("d1", "data-source"), HumanGating.NONE)),
                List.of(new Dependency(NodeId.of("sink1"), NodeId.of("ds1"))));

        var invariant = parameterizedInvariant("sinkMustHaveUpstream",
                List.of(
                        new PatternParameterDescriptor(PatternKind.MATCH, "sink", "", Direction.DEPENDENCIES),
                        new PatternParameterDescriptor(PatternKind.DIRECT_DEP, "data-source", "sink", Direction.DEPENDENCIES)));

        assertDoesNotThrow(() -> engine.validate(graph, List.of(invariant)));
    }

    // --- vacuously true when no @Match anchors ---

    @Test
    void vacuouslyTrueWhenNoMatchAnchors() {
        var graph = factory.of(
                List.of(new DesiredNode(NodeId.of("ds1"), new Spec("d1", "data-source"), HumanGating.NONE)),
                List.of());

        var invariant = parameterizedInvariant("sinkMustHaveUpstream",
                List.of(
                        new PatternParameterDescriptor(PatternKind.MATCH, "sink", "", Direction.DEPENDENCIES),
                        new PatternParameterDescriptor(PatternKind.DIRECT_DEP, "data-source", "sink", Direction.DEPENDENCIES)));

        assertDoesNotThrow(() -> engine.validate(graph, List.of(invariant)));
    }

    // --- imperative: violation ---

    public static void checkNoOrphans(DesiredStateGraph graph) {
        throw new GraphViolationException("Orphaned node found", NodeId.of("orphan1"));
    }

    @Test
    void imperativeViolation() {
        var graph = factory.of(List.of(), List.of());
        var invariant = imperativeInvariant("checkNoOrphans");

        var ex = assertThrows(GraphInvariantViolationsException.class,
                () -> engine.validate(graph, List.of(invariant)));
        assertEquals(1, ex.violations().size());
    }

    // --- imperative: passes ---

    public static void checkAlwaysPasses(DesiredStateGraph graph) {}

    @Test
    void imperativePasses() {
        var graph = factory.of(List.of(), List.of());
        var invariant = imperativeInvariant("checkAlwaysPasses");
        assertDoesNotThrow(() -> engine.validate(graph, List.of(invariant)));
    }

    // --- empty invariant list ---

    @Test
    void emptyInvariantListNoException() {
        var graph = factory.of(List.of(), List.of());
        assertDoesNotThrow(() -> engine.validate(graph, List.of()));
    }

    // --- multiple violations collected ---

    @Test
    void multipleViolationsCollected() {
        var graph = factory.of(
                List.of(
                        new DesiredNode(NodeId.of("sink1"), new Spec("s1", "sink"), HumanGating.NONE),
                        new DesiredNode(NodeId.of("sink2"), new Spec("s2", "sink"), HumanGating.NONE)),
                List.of());

        var invariant = parameterizedInvariant("sinkMustHaveUpstream",
                List.of(
                        new PatternParameterDescriptor(PatternKind.MATCH, "sink", "", Direction.DEPENDENCIES),
                        new PatternParameterDescriptor(PatternKind.DIRECT_DEP, "data-source", "sink", Direction.DEPENDENCIES)));

        var ex = assertThrows(GraphInvariantViolationsException.class,
                () -> engine.validate(graph, List.of(invariant)));
        assertEquals(2, ex.violations().size());
    }

    // --- helpers ---

    private ResolvedGraphInvariant parameterizedInvariant(String methodName,
            List<PatternParameterDescriptor> patterns) {
        try {
            Class<?>[] paramTypes = new Class<?>[patterns.size()];
            for (int i = 0; i < patterns.size(); i++) {
                paramTypes[i] = patterns.get(i).kind() == PatternKind.NOT_EXISTS
                        ? Void.class : DesiredNode.class;
            }
            Method method = getClass().getMethod(methodName, paramTypes);
            return new ResolvedGraphInvariant(methodName, method, null, false, patterns);
        } catch (Exception e) { throw new RuntimeException(e); }
    }

    private ResolvedGraphInvariant imperativeInvariant(String methodName) {
        try {
            Method method = getClass().getMethod(methodName, DesiredStateGraph.class);
            return new ResolvedGraphInvariant(methodName, method, null, true, List.of());
        } catch (Exception e) { throw new RuntimeException(e); }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=GraphInvariantEngineTest`
Expected: FAIL — `GraphInvariantEngine` class not found.

- [ ] **Step 3: Implement GraphInvariantEngine**

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeType;
import java.lang.reflect.Method;
import java.util.*;

public class GraphInvariantEngine {

    public void validate(DesiredStateGraph graph, List<ResolvedGraphInvariant> invariants) {
        List<GraphViolation> violations = new ArrayList<>();
        for (ResolvedGraphInvariant invariant : invariants) {
            if (invariant.imperative()) {
                validateImperative(invariant, graph, violations);
            } else {
                validateParameterized(invariant, graph, violations);
            }
        }
        if (!violations.isEmpty()) {
            throw new GraphInvariantViolationsException(violations);
        }
    }

    private void validateImperative(ResolvedGraphInvariant invariant,
            DesiredStateGraph graph, List<GraphViolation> violations) {
        try {
            Object instance = invariant.instance();
            if (instance != null) {
                invariant.method().invoke(instance, graph);
            } else {
                invariant.method().invoke(null, graph);
            }
        } catch (java.lang.reflect.InvocationTargetException e) {
            if (e.getCause() instanceof GraphViolationException gve) {
                violations.add(new GraphViolation(invariant.name(),
                        invariant.method().getDeclaringClass().getName(),
                        gve.getMessage(), gve.affectedNodes()));
            } else {
                throw new RuntimeException("Invariant method failed: "
                        + invariant.name(), e.getCause());
            }
        } catch (Exception e) {
            throw new RuntimeException("Invariant method invocation failed: "
                    + invariant.name(), e);
        }
    }

    private void validateParameterized(ResolvedGraphInvariant invariant,
            DesiredStateGraph graph, List<GraphViolation> violations) {
        // Find @Match parameters and enumerate anchors
        List<Integer> matchIndices = new ArrayList<>();
        for (int i = 0; i < invariant.patterns().size(); i++) {
            if (invariant.patterns().get(i).kind() == PatternKind.MATCH) {
                matchIndices.add(i);
            }
        }

        // Enumerate @Match cross-product
        List<List<DesiredNode>> matchSets = new ArrayList<>();
        for (int idx : matchIndices) {
            NodeType targetType = NodeType.of(invariant.patterns().get(idx).nodeType());
            List<DesiredNode> matches = graph.nodes().values().stream()
                    .filter(n -> n.type().equals(targetType))
                    .toList();
            matchSets.add(matches);
        }

        if (matchSets.isEmpty() || matchSets.stream().anyMatch(List::isEmpty)) {
            return; // vacuously true — no anchors to check
        }

        List<List<DesiredNode>> anchorTuples = PatternMatchingSupport.crossProduct(matchSets);
        String[] paramNames = PatternMatchingSupport.getParameterNames(invariant.method());

        // For each anchor tuple: check if remaining parameters can bind
        for (List<DesiredNode> anchorTuple : anchorTuples) {
            Map<String, DesiredNode> bindings = new LinkedHashMap<>();
            for (int i = 0; i < matchIndices.size(); i++) {
                bindings.put(paramNames[matchIndices.get(i)], anchorTuple.get(i));
            }

            List<List<Object>> expandedArgs = new ArrayList<>();
            expandChainForInvariant(invariant, graph, invariant.patterns(),
                    paramNames, bindings, new ArrayList<>(anchorTuple.stream()
                            .map(n -> (Object) n).toList()),
                    matchIndices.isEmpty() ? 0 : matchIndices.get(matchIndices.size() - 1) + 1,
                    expandedArgs);

            if (expandedArgs.isEmpty()) {
                // Structural violation — this anchor tuple could not fully expand
                String anchorDesc = anchorTuple.stream()
                        .map(n -> n.id().value())
                        .collect(java.util.stream.Collectors.joining(", "));
                violations.add(new GraphViolation(invariant.name(),
                        invariant.method().getDeclaringClass().getName(),
                        invariant.name() + " violated for [" + anchorDesc + "]",
                        anchorTuple.stream().map(DesiredNode::id).toList()));
            } else {
                // Value check — invoke method for each expanded tuple
                for (List<Object> args : expandedArgs) {
                    invokeInvariant(invariant, args, violations);
                }
            }
        }
    }

    private void expandChainForInvariant(ResolvedGraphInvariant invariant,
            DesiredStateGraph graph, List<PatternParameterDescriptor> patterns,
            String[] paramNames, Map<String, DesiredNode> bindings,
            List<Object> args, int startIndex, List<List<Object>> results) {
        if (startIndex >= patterns.size()) {
            results.add(new ArrayList<>(args));
            return;
        }

        PatternParameterDescriptor p = patterns.get(startIndex);
        switch (p.kind()) {
            case MATCH -> {
                // Already handled in anchor enumeration — skip
                expandChainForInvariant(invariant, graph, patterns, paramNames,
                        bindings, args, startIndex + 1, results);
            }
            case DIRECT_DEP -> {
                DesiredNode ref = PatternMatchingSupport.resolveReference(
                        p, startIndex, paramNames, bindings);
                List<DesiredNode> neighbors = PatternMatchingSupport.findDirectNeighbors(
                        graph, ref, p);
                for (DesiredNode neighbor : neighbors) {
                    Map<String, DesiredNode> newBindings = new LinkedHashMap<>(bindings);
                    newBindings.put(paramNames[startIndex], neighbor);
                    List<Object> newArgs = new ArrayList<>(args);
                    newArgs.add(neighbor);
                    expandChainForInvariant(invariant, graph, patterns, paramNames,
                            newBindings, newArgs, startIndex + 1, results);
                }
            }
            case REACHES -> {
                DesiredNode ref = PatternMatchingSupport.resolveReference(
                        p, startIndex, paramNames, bindings);
                List<DesiredNode> reachable = PatternMatchingSupport.findReachable(
                        graph, ref, p);
                for (DesiredNode reached : reachable) {
                    Map<String, DesiredNode> newBindings = new LinkedHashMap<>(bindings);
                    newBindings.put(paramNames[startIndex], reached);
                    List<Object> newArgs = new ArrayList<>(args);
                    newArgs.add(reached);
                    expandChainForInvariant(invariant, graph, patterns, paramNames,
                            newBindings, newArgs, startIndex + 1, results);
                }
            }
            case NOT_EXISTS -> {
                if (p.of().isEmpty()) {
                    if (PatternMatchingSupport.existsGlobal(graph, p)) {
                        return; // filter — type exists, tuple discarded
                    }
                } else {
                    DesiredNode ref = PatternMatchingSupport.resolveReference(
                            p, startIndex, paramNames, bindings);
                    if (PatternMatchingSupport.existsRelational(graph, ref, p)) {
                        return; // filter — relation exists, tuple discarded
                    }
                }
                List<Object> newArgs = new ArrayList<>(args);
                newArgs.add(null); // Void parameter
                expandChainForInvariant(invariant, graph, patterns, paramNames,
                        bindings, newArgs, startIndex + 1, results);
            }
        }
    }

    private void invokeInvariant(ResolvedGraphInvariant invariant,
            List<Object> args, List<GraphViolation> violations) {
        try {
            Object instance = invariant.instance();
            if (instance != null) {
                invariant.method().invoke(instance, args.toArray());
            } else {
                invariant.method().invoke(null, args.toArray());
            }
        } catch (java.lang.reflect.InvocationTargetException e) {
            if (e.getCause() instanceof GraphViolationException gve) {
                violations.add(new GraphViolation(invariant.name(),
                        invariant.method().getDeclaringClass().getName(),
                        gve.getMessage(), gve.affectedNodes()));
            } else {
                throw new RuntimeException("Invariant method failed: "
                        + invariant.name(), e.getCause());
            }
        } catch (Exception e) {
            throw new RuntimeException("Invariant method invocation failed: "
                    + invariant.name(), e);
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=GraphInvariantEngineTest`
Expected: All PASS.

- [ ] **Step 5: Commit**

```
feat(#107): GraphInvariantEngine with universal quantification Refs #107

Single-pass validation using per-anchor-tuple expansion. Structural
violations when any @Match anchor fails to expand. Value violations
when method body throws GraphViolationException. Uses shared
PatternMatchingSupport utilities.
```

---

### Task 7: @GraphInvariant Processor, Recorder, and Build Extension Tests

**Files:**
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java`
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java`
- Test: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/GraphInvariantProcessorTest.java`

**Interfaces:**
- Consumes: `GraphInvariantEngine.validate(...)`, `GraphInvariantDescriptor`, `ResolvedGraphInvariant`, `GraphPatternMatcher.matches(...)`

- [ ] **Step 1: Write failing processor integration test**

```java
package io.casehub.desiredstate.annotations.deployment;

import io.casehub.desiredstate.annotations.*;
import io.casehub.desiredstate.annotations.runtime.*;
import io.casehub.desiredstate.api.*;
import io.quarkus.test.QuarkusUnitTest;
import jakarta.inject.Inject;
import org.jboss.shrinkwrap.api.asset.StringAsset;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class GraphInvariantProcessorTest {

    // --- invariant holds: graph compiles ---

    @RegisterExtension
    static final QuarkusUnitTest validConfig = new QuarkusUnitTest()
            .withApplicationRoot(root -> root
                    .addClasses(ValidInvariantGraph.class,
                            ValidInvariantGraph.Spec.class)
                    .addAsResource(new StringAsset(""), "application.properties"));

    @Inject
    @DesiredStateQualifier(namespace = "test", name = "valid-invariant")
    GoalCompiler<Object> validCompiler;

    @Test
    void invariantHoldsGraphCompiles() {
        var result = validCompiler.compile(null,
                new io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory());
        assertNotNull(result);
    }
}
```

With test graph:

```java
@DesiredState(namespace = "test", name = "valid-invariant")
interface ValidInvariantGraph {
    record Spec(String name, String type) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of(type); }
    }

    @Node("sink1") default Spec sink() { return new Spec("s1", "sink"); }
    @Node("ds1") @DependsOn("sink1") default Spec source() { return new Spec("d1", "data-source"); }

    @GraphInvariant
    static void sinkHasUpstream(
            @Match(type = "sink") DesiredNode sink,
            @DirectDep(type = "data-source", of = "sink",
                    direction = Direction.DEPENDENCIES) DesiredNode upstream) {}
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=GraphInvariantProcessorTest`
Expected: FAIL — @GraphInvariant not recognized by processor.

- [ ] **Step 3: Add @GraphInvariant scanning to processor**

In `DesiredStateAnnotationsProcessor`:

Add `GRAPH_INVARIANT` DotName constant:
```java
private static final DotName GRAPH_INVARIANT =
        DotName.createSimple("io.casehub.desiredstate.annotations.GraphInvariant");
```

In `buildGraphDescriptor()`, after the graphRules loop, add invariant scanning:
```java
List<GraphInvariantDescriptor> graphInvariants = new ArrayList<>();
for (MethodInfo method : dsClass.methods()) {
    if (method.hasAnnotation(GRAPH_INVARIANT)) {
        graphInvariants.add(buildGraphInvariantDescriptor(method, index,
                dsClass.name().toString()));
    }
}
```

Add `buildGraphInvariantDescriptor` method (mirrors `buildGraphRuleDescriptor`):
```java
private GraphInvariantDescriptor buildGraphInvariantDescriptor(MethodInfo method,
        IndexView index, String sourceClassName) {
    if (method.parametersCount() == 1
            && method.parameterType(0).name().equals(DESIRED_STATE_GRAPH)) {
        return new GraphInvariantDescriptor(method.name(), true, List.of(), sourceClassName);
    }
    List<PatternParameterDescriptor> patterns = new ArrayList<>();
    for (int i = 0; i < method.parametersCount(); i++) {
        PatternParameterDescriptor ppd = buildPatternForParameter(method, i, index);
        if (ppd != null) {
            patterns.add(ppd);
        }
    }
    return new GraphInvariantDescriptor(method.name(), false, patterns, sourceClassName);
}
```

Add standalone @GraphInvariant class scanning (parallel to standalone @GraphRule scanning):
```java
Map<String[], List<GraphInvariantDescriptor>> standaloneInvariants = new LinkedHashMap<>();
for (AnnotationInstance giAnn : index.getAnnotations(GRAPH_INVARIANT)) {
    if (giAnn.target().kind() != AnnotationTarget.Kind.CLASS) continue;
    ClassInfo classInfo = giAnn.target().asClass();
    AnnotationValue graphVal = giAnn.value("graph");
    if (graphVal == null) continue;
    String[] graphPatterns = graphVal.asStringArray();
    if (graphPatterns.length == 0) continue;

    List<GraphInvariantDescriptor> classInvariants = new ArrayList<>();
    for (MethodInfo method : classInfo.methods()) {
        if (method.hasAnnotation(GRAPH_INVARIANT)
                && !java.lang.reflect.Modifier.isStatic(method.flags())
                && java.lang.reflect.Modifier.isPublic(method.flags())) {
            classInvariants.add(buildGraphInvariantDescriptor(method, index,
                    classInfo.name().toString()));
        }
    }
    if (!classInvariants.isEmpty()) {
        standaloneInvariants.put(graphPatterns, classInvariants);
    }
}
```

Merge standalone invariants into each matching graph descriptor (same pattern as standalone rules).

Update the GraphDescriptor construction to include `graphInvariants`.

- [ ] **Step 4: Add @GraphInvariant validation to AnnotationValidationStep**

Add `validateGraphInvariants` method (mirrors `validateGraphRules`):
```java
private void validateGraphInvariants(ClassInfo dsClass, IndexView index,
        List<String> errors) {
    for (MethodInfo method : dsClass.methods()) {
        if (!method.hasAnnotation(GRAPH_INVARIANT)) continue;

        if (!java.lang.reflect.Modifier.isStatic(method.flags())) {
            errors.add("@GraphInvariant on '" + method.name() + "' in "
                    + dsClass.name().local() + " must be a static method");
        }

        // Parameterized invariants must return void
        boolean isImperative = method.parametersCount() == 1
                && method.parameterType(0).name().equals(DESIRED_STATE_GRAPH);
        if (!isImperative && !method.returnType().name().toString().equals("void")) {
            errors.add("@GraphInvariant '" + method.name()
                    + "' parameterized method must return void");
        }

        validatePatternParameters(method, dsClass.name().local(), index, errors);
    }
}
```

Call `validateGraphInvariants(dsClass, index, errors)` in the `validate()` method alongside `validateGraphRules`.

Also add standalone @GraphInvariant class validation (same checks as standalone @GraphRule: concrete, no-arg ctor, non-empty graph, has include patterns, public methods).

- [ ] **Step 5: Add invariant validation to recorder**

In `DesiredStateGraphRecorder`, add invariant resolution and wrapping after the rule-wrapping block:

```java
if (!descriptor.graphInvariants().isEmpty()) {
    List<ResolvedGraphInvariant> resolvedInvariants =
            resolveInvariants(descriptor.graphInvariants());
    GraphInvariantEngine invariantEngine = new GraphInvariantEngine();
    @SuppressWarnings("rawtypes")
    GoalCompiler inner = runtimeValue.getValue();
    runtimeValue = new RuntimeValue<>((GoalCompiler) (goals, factory) ->
            validateInvariantsOnResult(inner.compile(goals, factory),
                    resolvedInvariants, invariantEngine));
}
```

Add `resolveInvariants` (mirrors `resolveRules`):
```java
private List<ResolvedGraphInvariant> resolveInvariants(
        List<GraphInvariantDescriptor> descriptors) {
    List<ResolvedGraphInvariant> invariants = new ArrayList<>();
    ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    for (GraphInvariantDescriptor gid : descriptors) {
        try {
            Class<?> cls = classLoader.loadClass(gid.sourceClassName());
            Object instance = java.lang.reflect.Modifier.isInterface(cls.getModifiers())
                    ? null : cls.getDeclaredConstructor().newInstance();
            Method method = findInvariantMethod(cls, gid);
            invariants.add(new ResolvedGraphInvariant(gid.methodName(), method,
                    instance, gid.imperative(), gid.patterns()));
        } catch (Exception e) {
            throw new RuntimeException("Failed to resolve graph invariant: "
                    + gid.methodName(), e);
        }
    }
    return invariants;
}
```

Add `findInvariantMethod` (mirrors `findRuleMethod` but handles void return):
```java
private Method findInvariantMethod(Class<?> cls, GraphInvariantDescriptor gid)
        throws NoSuchMethodException {
    if (gid.imperative()) {
        return cls.getMethod(gid.methodName(), DesiredStateGraph.class);
    }
    Class<?>[] paramTypes = new Class<?>[gid.patterns().size()];
    for (int i = 0; i < gid.patterns().size(); i++) {
        paramTypes[i] = gid.patterns().get(i).kind() == PatternKind.NOT_EXISTS
                ? Void.class : DesiredNode.class;
    }
    return cls.getMethod(gid.methodName(), paramTypes);
}
```

Add `validateInvariantsOnResult` (mirrors `applyGraphRulesToResult`):
```java
private CompilationResult validateInvariantsOnResult(CompilationResult result,
        List<ResolvedGraphInvariant> invariants, GraphInvariantEngine engine) {
    if (result instanceof CompilationResult.SingleGraph sg) {
        engine.validate(sg.graph(), invariants);
    } else if (result instanceof CompilationResult.Lifecycle lifecycle) {
        for (Phase phase : lifecycle.phases()) {
            engine.validate(phase.graph(), invariants);
        }
    }
    return result;
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=GraphInvariantProcessorTest`
Expected: PASS.

- [ ] **Step 7: Run full build across annotations modules**

Run: `mvn --batch-mode test -pl annotations/runtime,annotations/deployment`
Expected: All tests PASS.

- [ ] **Step 8: Commit**

```
feat(#107): @GraphInvariant processor, recorder, and validation integration Refs #107

Processor scans interface and standalone @GraphInvariant methods, builds
descriptors, merges standalone invariants by graph matching. Validation
checks static method, void return for parameterized, parameter annotations.
Recorder resolves invariants and wraps GoalCompiler to validate each
compile() result.
```

---

## Batch 4: Integration Example + Full Build

### Task 8: Pipeline-Annotated Integration Example

**Files:**
- Modify: `examples/pipeline-annotated/src/main/java/io/casehub/desiredstate/example/pipeline/annotated/MedallionPipeline.java`
- Modify: `examples/pipeline-annotated/src/test/java/io/casehub/desiredstate/example/pipeline/annotated/MedallionPipelineTest.java`

**Interfaces:**
- Consumes: All features from Tasks 1-7

- [ ] **Step 1: Add @GraphInvariant to MedallionPipeline**

Add an invariant method to the existing interface:

```java
@GraphInvariant
static void everySinkHasUpstream(
        @Match(type = "sink") DesiredNode sink,
        @DirectDep(type = "transformer", of = "sink",
                direction = Direction.DEPENDENCIES) DesiredNode upstream) {
}
```

- [ ] **Step 2: Add @Tier(nodeType) to existing fault policy**

Update the existing `@Tier` annotations on MedallionPipeline to include `nodeType`:

```java
@Tier(threshold = 3, review = "createAiReview", nodeType = "ai-review")
```

- [ ] **Step 3: Update MedallionPipelineTest to verify invariant and monitoring rule**

Add test assertions verifying:
- The monitoring node added by the @GraphRule `ensureMonitoring` exists
- The @GraphInvariant `everySinkHasUpstream` doesn't throw (the graph satisfies it)
- The @Tier(nodeType) compiles without error

```java
@Test
void compiledGraphContainsRuleGeneratedNodesAndSatisfiesInvariants() {
    var result = compiler.compile(null, graphFactory);
    var graph = ((CompilationResult.SingleGraph) result).graph();

    // Rule-generated monitoring node exists
    assertTrue(graph.nodes().containsKey(NodeId.of("monitor-warehouse-sink")),
            "ensureMonitoring rule should have added monitor node for warehouse-sink");

    // Graph compiled successfully — invariants passed
    assertNotNull(graph);
}
```

- [ ] **Step 4: Run pipeline-annotated tests**

Run: `mvn --batch-mode test -pl examples/pipeline-annotated`
Expected: All PASS.

- [ ] **Step 5: Run full project build**

Run: `mvn --batch-mode install`
Expected: All modules compile and all tests PASS.

- [ ] **Step 6: Commit**

```
feat(#107): pipeline-annotated example with @GraphInvariant and @Tier(nodeType) Refs #107

Demonstrates @GraphInvariant, @Tier(nodeType), and existing @GraphRule
working together in a compiled graph.
```

---

## References

- [2026-08-26-graph-rules-invariants-design.md](/Users/mdproctor/claude/public/casehub-desiredstate/specs/issue-115-graph-rules-invariants/2026-08-26-graph-rules-invariants-design.md) — design spec this plan implements
- [decisions.md](/Users/mdproctor/claude/public/casehub-desiredstate/specs/issue-115-graph-rules-invariants/decisions.md) — 8 design decisions
- [GraphRuleEngine.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngine.java) — pattern matching source for extraction
- [DesiredStateAnnotationsProcessor.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java) — processor extended by Tasks 3, 7
- [AnnotationValidationStep.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java) — validation extended by Tasks 3, 4, 7
- [DesiredStateGraphRecorder.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java) — recorder extended by Tasks 2, 7
- [2026-08-24-graph-rule-design.md](/Users/mdproctor/claude/casehub/desiredstate/docs/specs/issue-106-graph-rule/2026-08-24-graph-rule-design.md) — parent spec
- GitHub #115, #113, #107
