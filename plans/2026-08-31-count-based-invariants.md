# Count-Based Graph Invariants Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #127 — feat: count-based graph invariants (cardinality constraints)
**Issue group:** #127

**Goal:** Add `minCount`/`maxCount` cardinality constraints to `@Match`,
`@DirectDep`, and `@Reaches` pattern annotations and YAML patterns, evaluated
by `GraphInvariantEngine` at two levels: match-level (global node count) and
expansion-level (per-anchor neighbor count).

**Architecture:** Extend `PatternParameterDescriptor` with two int fields
(`minCount`, `maxCount`) plus a 4-arg backward-compatible constructor.
`GraphInvariantEngine` adds a match-cardinality pre-check (count-only, no
expansion) and expansion-level cardinality counting (distinct binding values
per anchor). YAML `YamlPattern` gets boxed `Integer` fields. Build-time
validation catches `@NotExists` with cardinality and invalid ranges.

**Tech Stack:** Java 21, Quarkus build extensions (Jandex), JUnit 5, AssertJ

## Global Constraints

- Foundation tier — no upward deps from `api/` or `annotations/` to engine/work
- Pre-release — no backward compatibility concerns
- `PatternParameterDescriptor` 4-arg constructor must remain for existing call sites
- `-1` sentinel for annotation `int` attributes means "not specified"
- `@NotExists` with cardinality is a build-time error

---

## Batch 1: Foundation — PatternParameterDescriptor + Engine

### Task 1: Extend PatternParameterDescriptor with cardinality fields

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/PatternParameterDescriptor.java`
- Test: `annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/PatternParameterDescriptorTest.java`

**Interfaces:**
- Produces: `PatternParameterDescriptor(PatternKind, String, String, Direction, int, int)` canonical constructor; `PatternParameterDescriptor(PatternKind, String, String, Direction)` compact constructor preserving existing call sites; `UNSPECIFIED = -1` constant; `effectiveMinCount()`, `effectiveMaxCount()`, `hasCardinalityConstraint()` methods

- [ ] **Step 1: Write the failing test**

Create `PatternParameterDescriptorTest.java`:

```java
package io.casehub.desiredstate.annotations.runtime;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class PatternParameterDescriptorTest {

    @Test
    void fourArgConstructorSetsUnspecifiedCardinality() {
        var ppd = new PatternParameterDescriptor(
                PatternKind.MATCH, "sink", "", Direction.DEPENDENCIES);
        assertEquals(PatternParameterDescriptor.UNSPECIFIED, ppd.minCount());
        assertEquals(PatternParameterDescriptor.UNSPECIFIED, ppd.maxCount());
        assertFalse(ppd.hasCardinalityConstraint());
    }

    @Test
    void effectiveMinCountDefaultsForMatch() {
        var ppd = new PatternParameterDescriptor(
                PatternKind.MATCH, "sink", "", Direction.DEPENDENCIES);
        assertEquals(0, ppd.effectiveMinCount());
        assertEquals(Integer.MAX_VALUE, ppd.effectiveMaxCount());
    }

    @Test
    void effectiveMinCountDefaultsForDirectDep() {
        var ppd = new PatternParameterDescriptor(
                PatternKind.DIRECT_DEP, "target", "lb", Direction.DEPENDENTS);
        assertEquals(1, ppd.effectiveMinCount());
        assertEquals(Integer.MAX_VALUE, ppd.effectiveMaxCount());
    }

    @Test
    void explicitCardinalityOverridesDefaults() {
        var ppd = new PatternParameterDescriptor(
                PatternKind.MATCH, "instance", "", Direction.DEPENDENCIES, 3, 10);
        assertEquals(3, ppd.effectiveMinCount());
        assertEquals(10, ppd.effectiveMaxCount());
        assertTrue(ppd.hasCardinalityConstraint());
    }

    @Test
    void hasCardinalityConstraintTrueWhenOnlyMinSet() {
        var ppd = new PatternParameterDescriptor(
                PatternKind.MATCH, "instance", "", Direction.DEPENDENCIES,
                3, PatternParameterDescriptor.UNSPECIFIED);
        assertTrue(ppd.hasCardinalityConstraint());
    }

    @Test
    void hasCardinalityConstraintTrueWhenOnlyMaxSet() {
        var ppd = new PatternParameterDescriptor(
                PatternKind.REACHES, "endpoint", "router", Direction.DEPENDENTS,
                PatternParameterDescriptor.UNSPECIFIED, 5);
        assertTrue(ppd.hasCardinalityConstraint());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=PatternParameterDescriptorTest`
Expected: Compilation error — no 4-arg constructor, no `UNSPECIFIED`, no `effectiveMinCount()`, etc.

- [ ] **Step 3: Implement PatternParameterDescriptor**

Replace `PatternParameterDescriptor.java`:

```java
package io.casehub.desiredstate.annotations.runtime;

public record PatternParameterDescriptor(
        PatternKind kind,
        String nodeType,
        String of,
        Direction direction,
        int minCount,
        int maxCount) {

    public static final int UNSPECIFIED = -1;

    public PatternParameterDescriptor(PatternKind kind, String nodeType,
                                       String of, Direction direction) {
        this(kind, nodeType, of, direction, UNSPECIFIED, UNSPECIFIED);
    }

    public int effectiveMinCount() {
        if (minCount != UNSPECIFIED) return minCount;
        return kind == PatternKind.MATCH ? 0 : 1;
    }

    public int effectiveMaxCount() {
        return maxCount != UNSPECIFIED ? maxCount : Integer.MAX_VALUE;
    }

    public boolean hasCardinalityConstraint() {
        return minCount != UNSPECIFIED || maxCount != UNSPECIFIED;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=PatternParameterDescriptorTest`
Expected: All 6 tests PASS

- [ ] **Step 5: Verify existing tests still pass**

Run: `mvn --batch-mode test -pl annotations/runtime`
Expected: All existing tests PASS — the 4-arg constructor keeps all call sites working.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/desiredstate add annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/PatternParameterDescriptor.java annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/PatternParameterDescriptorTest.java
git -C /Users/mdproctor/claude/casehub/desiredstate commit -m "feat(#127): extend PatternParameterDescriptor with minCount/maxCount cardinality fields"
```

### Task 2: Match-level cardinality in GraphInvariantEngine

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphInvariantEngine.java`
- Modify: `annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/GraphInvariantEngineTest.java`

**Interfaces:**
- Consumes: `PatternParameterDescriptor.hasCardinalityConstraint()`, `effectiveMinCount()`, `effectiveMaxCount()` from Task 1
- Produces: Match-level cardinality evaluation in `validateParameterized()` and `validateDeclarative()` — when any `@Match` pattern has cardinality, the invariant becomes a count-only check

- [ ] **Step 1: Write failing tests for match-level cardinality**

Add to `GraphInvariantEngineTest.java` — a static no-op method for cardinality invariants, plus test methods:

```java
public static void haMinimum(DesiredNode instance) {}

@Test
void matchMinCountViolation_tooFewNodes() {
    var graph = factory.of(
            List.of(
                    new DesiredNode(NodeId.of("i1"), new Spec("i1", "compute"), HumanGating.NONE),
                    new DesiredNode(NodeId.of("i2"), new Spec("i2", "compute"), HumanGating.NONE)),
            List.of());

    var invariant = parameterizedInvariant("haMinimum",
            List.of(new PatternParameterDescriptor(
                    PatternKind.MATCH, "compute", "", Direction.DEPENDENCIES, 3, -1)));

    var ex = assertThrows(GraphInvariantViolationsException.class,
            () -> engine.validate(graph, List.of(invariant)));
    assertEquals(1, ex.violations().size());
    assertTrue(ex.violations().get(0).message().contains("at least 3"));
    assertTrue(ex.violations().get(0).message().contains("found 2"));
}

@Test
void matchMinCountPasses_exactlyEnough() {
    var graph = factory.of(
            List.of(
                    new DesiredNode(NodeId.of("i1"), new Spec("i1", "compute"), HumanGating.NONE),
                    new DesiredNode(NodeId.of("i2"), new Spec("i2", "compute"), HumanGating.NONE),
                    new DesiredNode(NodeId.of("i3"), new Spec("i3", "compute"), HumanGating.NONE)),
            List.of());

    var invariant = parameterizedInvariant("haMinimum",
            List.of(new PatternParameterDescriptor(
                    PatternKind.MATCH, "compute", "", Direction.DEPENDENCIES, 3, -1)));

    assertDoesNotThrow(() -> engine.validate(graph, List.of(invariant)));
}

@Test
void matchMaxCountViolation_tooManyNodes() {
    var graph = factory.of(
            List.of(
                    new DesiredNode(NodeId.of("cp1"), new Spec("cp1", "control-plane"), HumanGating.NONE),
                    new DesiredNode(NodeId.of("cp2"), new Spec("cp2", "control-plane"), HumanGating.NONE)),
            List.of());

    var invariant = parameterizedInvariant("haMinimum",
            List.of(new PatternParameterDescriptor(
                    PatternKind.MATCH, "control-plane", "", Direction.DEPENDENCIES, 1, 1)));

    var ex = assertThrows(GraphInvariantViolationsException.class,
            () -> engine.validate(graph, List.of(invariant)));
    assertEquals(1, ex.violations().size());
    assertTrue(ex.violations().get(0).message().contains("at most 1"));
}

@Test
void matchSingletonPasses() {
    var graph = factory.of(
            List.of(new DesiredNode(NodeId.of("cp1"), new Spec("cp1", "control-plane"), HumanGating.NONE)),
            List.of());

    var invariant = parameterizedInvariant("haMinimum",
            List.of(new PatternParameterDescriptor(
                    PatternKind.MATCH, "control-plane", "", Direction.DEPENDENCIES, 1, 1)));

    assertDoesNotThrow(() -> engine.validate(graph, List.of(invariant)));
}

@Test
void matchSingletonViolation_zeroNodes() {
    var graph = factory.of(List.of(), List.of());

    var invariant = parameterizedInvariant("haMinimum",
            List.of(new PatternParameterDescriptor(
                    PatternKind.MATCH, "control-plane", "", Direction.DEPENDENCIES, 1, 1)));

    var ex = assertThrows(GraphInvariantViolationsException.class,
            () -> engine.validate(graph, List.of(invariant)));
    assertEquals(1, ex.violations().size());
    assertTrue(ex.violations().get(0).message().contains("at least 1"));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=GraphInvariantEngineTest#matchMinCountViolation_tooFewNodes`
Expected: FAIL — no cardinality logic yet

- [ ] **Step 3: Implement match-level cardinality in GraphInvariantEngine**

Add private methods to `GraphInvariantEngine`:

```java
private boolean hasMatchCardinalityConstraint(List<PatternParameterDescriptor> patterns) {
    return patterns.stream()
            .anyMatch(p -> p.kind() == PatternKind.MATCH && p.hasCardinalityConstraint());
}

private void validateMatchCardinality(String invariantName, String sourceClass,
        DesiredStateGraph graph, List<PatternParameterDescriptor> patterns,
        List<GraphViolation> violations) {
    for (PatternParameterDescriptor p : patterns) {
        if (p.kind() != PatternKind.MATCH) continue;
        long count = countMatchingNodes(graph, p.nodeType());
        if (count < p.effectiveMinCount()) {
            violations.add(new GraphViolation(invariantName, sourceClass,
                    invariantName + ": expected at least " + p.effectiveMinCount()
                    + " node(s) of type '" + p.nodeType() + "', found " + count,
                    List.of()));
        }
        if (count > p.effectiveMaxCount()) {
            violations.add(new GraphViolation(invariantName, sourceClass,
                    invariantName + ": expected at most " + p.effectiveMaxCount()
                    + " node(s) of type '" + p.nodeType() + "', found " + count,
                    List.of()));
        }
    }
}

private long countMatchingNodes(DesiredStateGraph graph, String nodeType) {
    if ("*".equals(nodeType)) return graph.nodes().size();
    NodeType target = NodeType.of(nodeType);
    return graph.nodes().values().stream()
            .filter(n -> n.type().equals(target))
            .count();
}
```

Insert the cardinality pre-check at the top of `validateParameterized()` (before the existing anchor logic):

```java
if (hasMatchCardinalityConstraint(patterns)) {
    validateMatchCardinality(invariant.name(),
            invariant.method().getDeclaringClass().getName(),
            graph, patterns, violations);
    return;
}
```

Insert the same pre-check at the top of `validateDeclarative()`:

```java
if (hasMatchCardinalityConstraint(patterns)) {
    String source = invariant.messageTemplate() != null ? invariant.messageTemplate() : "yaml";
    validateMatchCardinality(invariant.name(), "yaml", graph, patterns, violations);
    return;
}
```

- [ ] **Step 4: Run all tests to verify they pass**

Run: `mvn --batch-mode test -pl annotations/runtime`
Expected: All tests PASS (new cardinality + existing)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/desiredstate add annotations/runtime/src/
git -C /Users/mdproctor/claude/casehub/desiredstate commit -m "feat(#127): match-level cardinality in GraphInvariantEngine"
```

### Task 3: Expansion-level cardinality in GraphInvariantEngine

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphInvariantEngine.java`
- Modify: `annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/GraphInvariantEngineTest.java`

**Interfaces:**
- Consumes: `PatternParameterDescriptor.hasCardinalityConstraint()`, `effectiveMinCount()`, `effectiveMaxCount()` from Task 1
- Produces: Expansion-level cardinality evaluation — counts distinct binding values per anchor for constrained `@DirectDep`/`@Reaches` patterns

- [ ] **Step 1: Write failing tests for expansion-level cardinality**

Add to `GraphInvariantEngineTest.java`:

```java
public static void lbMinTargets(DesiredNode lb, DesiredNode target) {}

@Test
void expansionMinCountViolation_tooFewDeps() {
    var graph = factory.of(
            List.of(
                    new DesiredNode(NodeId.of("lb1"), new Spec("lb1", "load-balancer"), HumanGating.NONE),
                    new DesiredNode(NodeId.of("t1"), new Spec("t1", "target"), HumanGating.NONE)),
            List.of(new Dependency(NodeId.of("t1"), NodeId.of("lb1"))));

    var invariant = parameterizedInvariant("lbMinTargets",
            List.of(
                    new PatternParameterDescriptor(PatternKind.MATCH, "load-balancer", "", Direction.DEPENDENCIES),
                    new PatternParameterDescriptor(PatternKind.DIRECT_DEP, "target", "lb",
                            Direction.DEPENDENTS, 2, -1)));

    var ex = assertThrows(GraphInvariantViolationsException.class,
            () -> engine.validate(graph, List.of(invariant)));
    assertEquals(1, ex.violations().size());
    assertTrue(ex.violations().get(0).message().contains("at least 2"));
}

@Test
void expansionMinCountPasses_enoughDeps() {
    var graph = factory.of(
            List.of(
                    new DesiredNode(NodeId.of("lb1"), new Spec("lb1", "load-balancer"), HumanGating.NONE),
                    new DesiredNode(NodeId.of("t1"), new Spec("t1", "target"), HumanGating.NONE),
                    new DesiredNode(NodeId.of("t2"), new Spec("t2", "target"), HumanGating.NONE),
                    new DesiredNode(NodeId.of("t3"), new Spec("t3", "target"), HumanGating.NONE)),
            List.of(
                    new Dependency(NodeId.of("t1"), NodeId.of("lb1")),
                    new Dependency(NodeId.of("t2"), NodeId.of("lb1")),
                    new Dependency(NodeId.of("t3"), NodeId.of("lb1"))));

    var invariant = parameterizedInvariant("lbMinTargets",
            List.of(
                    new PatternParameterDescriptor(PatternKind.MATCH, "load-balancer", "", Direction.DEPENDENCIES),
                    new PatternParameterDescriptor(PatternKind.DIRECT_DEP, "target", "lb",
                            Direction.DEPENDENTS, 2, -1)));

    assertDoesNotThrow(() -> engine.validate(graph, List.of(invariant)));
}

@Test
void expansionMaxCountViolation_tooManyDeps() {
    var graph = factory.of(
            List.of(
                    new DesiredNode(NodeId.of("svc1"), new Spec("svc1", "service"), HumanGating.NONE),
                    new DesiredNode(NodeId.of("db1"), new Spec("db1", "database"), HumanGating.NONE),
                    new DesiredNode(NodeId.of("db2"), new Spec("db2", "database"), HumanGating.NONE)),
            List.of(
                    new Dependency(NodeId.of("svc1"), NodeId.of("db1")),
                    new Dependency(NodeId.of("svc1"), NodeId.of("db2"))));

    var invariant = parameterizedInvariant("lbMinTargets",
            List.of(
                    new PatternParameterDescriptor(PatternKind.MATCH, "service", "", Direction.DEPENDENCIES),
                    new PatternParameterDescriptor(PatternKind.DIRECT_DEP, "database", "lb",
                            Direction.DEPENDENCIES, -1, 1)));

    var ex = assertThrows(GraphInvariantViolationsException.class,
            () -> engine.validate(graph, List.of(invariant)));
    assertEquals(1, ex.violations().size());
    assertTrue(ex.violations().get(0).message().contains("at most 1"));
}

@Test
void defaultDirectDepBehavior_unchanged() {
    // Zero expansions with no cardinality = structural violation (existing behavior)
    var graph = factory.of(
            List.of(
                    new DesiredNode(NodeId.of("sink1"), new Spec("s1", "sink"), HumanGating.NONE)),
            List.of());

    var invariant = parameterizedInvariant("sinkMustHaveUpstream",
            List.of(
                    new PatternParameterDescriptor(PatternKind.MATCH, "sink", "", Direction.DEPENDENCIES),
                    new PatternParameterDescriptor(PatternKind.DIRECT_DEP, "data-source", "sink", Direction.DEPENDENCIES)));

    var ex = assertThrows(GraphInvariantViolationsException.class,
            () -> engine.validate(graph, List.of(invariant)));
    assertEquals(1, ex.violations().size());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=GraphInvariantEngineTest#expansionMinCountViolation_tooFewDeps`
Expected: FAIL — no expansion-level cardinality logic yet

- [ ] **Step 3: Implement expansion-level cardinality**

Modify the per-anchor evaluation loop in `validateParameterized()`. After grouping by anchor and building expected anchors, replace the expansion check:

In the loop over `expectedAnchors`, after getting `expansions` from `byAnchor.get(anchor)`:

```java
for (List<DesiredNode> anchor : expectedAnchors) {
    List<Map<String, DesiredNode>> expansions = byAnchor.get(anchor);

    // Check expansion-level cardinality
    boolean cardinalityFailed = false;
    for (int i = 0; i < patterns.size(); i++) {
        PatternParameterDescriptor p = patterns.get(i);
        if (p.kind() == PatternKind.MATCH || p.kind() == PatternKind.NOT_EXISTS) continue;
        if (p.hasCardinalityConstraint()) {
            long bindingCount = expansions == null ? 0
                    : expansions.stream()
                            .map(b -> b.get(paramNames[i]))
                            .distinct()
                            .count();
            if (bindingCount < p.effectiveMinCount()) {
                String anchorDesc = anchor.stream()
                        .map(n -> n.id().value()).collect(Collectors.joining(", "));
                violations.add(new GraphViolation(invariant.name(),
                        invariant.method().getDeclaringClass().getName(),
                        invariant.name() + " for [" + anchorDesc + "]: expected at least "
                        + p.effectiveMinCount() + " '" + p.nodeType()
                        + "' binding(s), found " + bindingCount,
                        anchor.stream().map(DesiredNode::id).toList()));
                cardinalityFailed = true;
            }
            if (bindingCount > p.effectiveMaxCount()) {
                String anchorDesc = anchor.stream()
                        .map(n -> n.id().value()).collect(Collectors.joining(", "));
                violations.add(new GraphViolation(invariant.name(),
                        invariant.method().getDeclaringClass().getName(),
                        invariant.name() + " for [" + anchorDesc + "]: expected at most "
                        + p.effectiveMaxCount() + " '" + p.nodeType()
                        + "' binding(s), found " + bindingCount,
                        anchor.stream().map(DesiredNode::id).toList()));
                cardinalityFailed = true;
            }
        }
    }

    if (!cardinalityFailed) {
        if (expansions == null || expansions.isEmpty()) {
            // Default: zero expansions = structural violation (unchanged)
            String anchorDesc = anchor.stream()
                    .map(n -> n.id().value()).collect(Collectors.joining(", "));
            violations.add(new GraphViolation(invariant.name(),
                    invariant.method().getDeclaringClass().getName(),
                    invariant.name() + " violated for [" + anchorDesc + "]",
                    anchor.stream().map(DesiredNode::id).toList()));
        } else {
            for (Map<String, DesiredNode> binding : expansions) {
                List<Object> args = new ArrayList<>(paramNames.length);
                for (String paramName : paramNames) {
                    args.add(binding.get(paramName));
                }
                invokeReflectiveInvariant(invariant, args, violations);
            }
        }
    }
}
```

Apply the same pattern in `validateDeclarative()` — but without method invocation (declarative invariants have no method body). Replace the existing expansion check with the cardinality-aware version.

- [ ] **Step 4: Run all tests to verify they pass**

Run: `mvn --batch-mode test -pl annotations/runtime`
Expected: All tests PASS (new expansion-level + existing)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/desiredstate add annotations/runtime/src/
git -C /Users/mdproctor/claude/casehub/desiredstate commit -m "feat(#127): expansion-level cardinality in GraphInvariantEngine"
```

---

## Batch 2: Annotation Surface + Build Validation

### Task 4: Add cardinality to @Match, @DirectDep, @Reaches annotations

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/Match.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DirectDep.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/Reaches.java`

**Interfaces:**
- Produces: `minCount()` and `maxCount()` attributes (default `-1`) on `@Match`, `@DirectDep`, `@Reaches`

- [ ] **Step 1: Add attributes to @Match**

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface Match {
    String type();
    int minCount() default -1;
    int maxCount() default -1;
}
```

- [ ] **Step 2: Add attributes to @DirectDep**

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface DirectDep {
    String type();
    String of() default "";
    Direction direction() default Direction.DEPENDENCIES;
    int minCount() default -1;
    int maxCount() default -1;
}
```

- [ ] **Step 3: Add attributes to @Reaches**

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface Reaches {
    String type();
    String of() default "";
    Direction direction() default Direction.DEPENDENCIES;
    int minCount() default -1;
    int maxCount() default -1;
}
```

- [ ] **Step 4: Verify existing tests pass**

Run: `mvn --batch-mode test -pl annotations/runtime`
Expected: All PASS — new attributes have defaults, no existing code references them yet.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/desiredstate add annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/Match.java annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DirectDep.java annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/Reaches.java
git -C /Users/mdproctor/claude/casehub/desiredstate commit -m "feat(#127): add minCount/maxCount to @Match, @DirectDep, @Reaches annotations"
```

### Task 5: Wire annotations into processor + build-time validation

**Files:**
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java` (method `buildPatternForParameter` at line 527)
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java` (method `validatePatternParameters` at line 357)
- Test: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessorTest.java`

**Interfaces:**
- Consumes: `@Match.minCount()`, `@Match.maxCount()`, `@DirectDep.minCount()`, etc. from Task 4; `PatternParameterDescriptor` 6-arg constructor from Task 1
- Produces: Build-time cardinality extraction and validation errors

- [ ] **Step 1: Write failing test for cardinality extraction**

Add a test interface and test method to `DesiredStateAnnotationsProcessorTest` that uses `@Match(type = "compute", minCount = 3)` in a `@GraphInvariant` method. Assert that compilation succeeds and the invariant fires correctly when there are too few nodes.

```java
@Test
void graphInvariantWithMatchCardinality_violatesWhenTooFew() {
    // Create a graph with 2 compute nodes, invariant requires 3
    // Use existing test patterns: create @DesiredState interface with @GraphInvariant
    // that has @Match(type = "compute", minCount = 3)
    // Assert GraphInvariantViolationsException is thrown
}
```

The exact test shape follows the existing `GraphInvariantProcessorTest` pattern — a Quarkus `@QuarkusTest` with the interface declared inline.

- [ ] **Step 2: Write failing test for @NotExists cardinality validation**

```java
@Test
void notExistsWithCardinality_buildError() {
    // @GraphInvariant method with @NotExists(type = "x", minCount = 2)
    // Should produce a build-time error
}
```

- [ ] **Step 3: Write failing test for minCount > maxCount validation**

```java
@Test
void minCountGreaterThanMaxCount_buildError() {
    // @Match(type = "x", minCount = 5, maxCount = 3)
    // Should produce a build-time error
}
```

- [ ] **Step 4: Implement cardinality extraction in buildPatternForParameter**

Modify `buildPatternForParameter()` in `DesiredStateAnnotationsProcessor.java` to read `minCount`/`maxCount` from the annotation and pass them to the 6-arg `PatternParameterDescriptor` constructor:

```java
if (annName.equals(MATCH)) {
    int minCount = ann.valueWithDefault(index, "minCount").asInt();
    int maxCount = ann.valueWithDefault(index, "maxCount").asInt();
    return new PatternParameterDescriptor(
            PatternKind.MATCH, ann.value("type").asString(),
            "", Direction.DEPENDENCIES, minCount, maxCount);
}
if (annName.equals(DIRECT_DEP)) {
    int minCount = ann.valueWithDefault(index, "minCount").asInt();
    int maxCount = ann.valueWithDefault(index, "maxCount").asInt();
    return new PatternParameterDescriptor(
            PatternKind.DIRECT_DEP,
            ann.value("type").asString(),
            ann.valueWithDefault(index, "of").asString(),
            Direction.valueOf(ann.valueWithDefault(index, "direction").asEnum()),
            minCount, maxCount);
}
if (annName.equals(REACHES)) {
    int minCount = ann.valueWithDefault(index, "minCount").asInt();
    int maxCount = ann.valueWithDefault(index, "maxCount").asInt();
    return new PatternParameterDescriptor(
            PatternKind.REACHES,
            ann.value("type").asString(),
            ann.valueWithDefault(index, "of").asString(),
            Direction.valueOf(ann.valueWithDefault(index, "direction").asEnum()),
            minCount, maxCount);
}
// NOT_EXISTS: no changes — still uses 4-arg constructor (no cardinality)
```

- [ ] **Step 5: Implement cardinality validation in validatePatternParameters**

Add to `validatePatternParameters()` in `AnnotationValidationStep.java`, inside the annotation loop:

```java
// After existing checks for each annotation...
if (annName.equals(MATCH) || annName.equals(DIRECT_DEP) || annName.equals(REACHES)) {
    AnnotationValue minVal = ann.value("minCount");
    AnnotationValue maxVal = ann.value("maxCount");
    int minCount = minVal != null ? minVal.asInt() : -1;
    int maxCount = maxVal != null ? maxVal.asInt() : -1;
    if (minCount != -1 && minCount < 0) {
        errors.add("@" + annName.local() + " on parameter '"
                + paramName + "' has invalid minCount: " + minCount);
    }
    if (maxCount != -1 && maxCount < 0) {
        errors.add("@" + annName.local() + " on parameter '"
                + paramName + "' has invalid maxCount: " + maxCount);
    }
    if (minCount != -1 && maxCount != -1 && minCount > maxCount) {
        errors.add("@" + annName.local() + " on parameter '"
                + paramName + "' has minCount (" + minCount
                + ") > maxCount (" + maxCount + ")");
    }
}

if (annName.equals(NOT_EXISTS)) {
    AnnotationValue minVal = ann.value("minCount");
    AnnotationValue maxVal = ann.value("maxCount");
    if (minVal != null || maxVal != null) {
        errors.add("@NotExists on parameter '" + paramName
                + "' does not support minCount/maxCount — it is a boolean guard");
    }
}
```

Note: `@NotExists` does not have `minCount`/`maxCount` attributes, so this check
guards against future additions or annotation misuse — if someone adds the fields
to `@NotExists` later, this validation catches it. Alternatively, since `@NotExists`
has no such attributes, this check may be skipped. Include it if `@NotExists` could
be confused with a pattern that should have cardinality.

- [ ] **Step 6: Run all tests**

Run: `mvn --batch-mode test -pl annotations/deployment`
Expected: All PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/desiredstate add annotations/deployment/src/
git -C /Users/mdproctor/claude/casehub/desiredstate commit -m "feat(#127): wire cardinality into processor + build-time validation"
```

---

## Batch 3: YAML Surface

### Task 6: Add cardinality to YamlPattern and wire through converter

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlPattern.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlInvariantConverter.java`
- Modify: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/model/YamlInvariantModelTest.java`
- Modify: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlInvariantEvaluationTest.java`

**Interfaces:**
- Consumes: `PatternParameterDescriptor` 6-arg constructor from Task 1
- Produces: `YamlPattern.minCount` (Integer), `YamlPattern.maxCount` (Integer); `YamlInvariantConverter` passes cardinality through to `PatternParameterDescriptor`

- [ ] **Step 1: Write failing test for YAML match-level cardinality**

Add to `YamlInvariantEvaluationTest.java`:

```java
@Test
void declarativeInvariant_matchMinCount_violated() {
    DesiredStateGraph graph = factory.of(
            List.of(
                    new DesiredNode(NodeId.of("i1"), new Spec("i1", "compute"), HumanGating.NONE),
                    new DesiredNode(NodeId.of("i2"), new Spec("i2", "compute"), HumanGating.NONE)),
            List.of());

    YamlInvariant yamlInv = new YamlInvariant(
            List.of(),
            Map.of("instance", new YamlPattern("compute", null, Direction.DEPENDENCIES, 3, null)),
            Map.of(), Map.of(), Map.of(),
            "HA requires at least 3 compute instances");

    ResolvedInvariant invariant = YamlInvariantConverter.toDeclarativeInvariant(
            "ha-minimum", yamlInv);

    var ex = assertThrows(GraphInvariantViolationsException.class,
            () -> engine.validate(graph, List.of(invariant)));
    assertThat(ex.violations()).hasSize(1);
    assertThat(ex.violations().get(0).message()).contains("at least 3");
}

@Test
void declarativeInvariant_matchMinCount_passes() {
    DesiredStateGraph graph = factory.of(
            List.of(
                    new DesiredNode(NodeId.of("i1"), new Spec("i1", "compute"), HumanGating.NONE),
                    new DesiredNode(NodeId.of("i2"), new Spec("i2", "compute"), HumanGating.NONE),
                    new DesiredNode(NodeId.of("i3"), new Spec("i3", "compute"), HumanGating.NONE)),
            List.of());

    YamlInvariant yamlInv = new YamlInvariant(
            List.of(),
            Map.of("instance", new YamlPattern("compute", null, Direction.DEPENDENCIES, 3, null)),
            Map.of(), Map.of(), Map.of(),
            "HA requires at least 3 compute instances");

    ResolvedInvariant invariant = YamlInvariantConverter.toDeclarativeInvariant(
            "ha-minimum", yamlInv);

    assertDoesNotThrow(() -> engine.validate(graph, List.of(invariant)));
}

@Test
void declarativeInvariant_expansionMinCount_violated() {
    DesiredStateGraph graph = factory.of(
            List.of(
                    new DesiredNode(NodeId.of("lb1"), new Spec("lb1", "load-balancer"), HumanGating.NONE),
                    new DesiredNode(NodeId.of("t1"), new Spec("t1", "target"), HumanGating.NONE)),
            List.of(new Dependency(NodeId.of("t1"), NodeId.of("lb1"))));

    YamlInvariant yamlInv = new YamlInvariant(
            List.of(),
            Map.of("lb", new YamlPattern("load-balancer", null, Direction.DEPENDENCIES)),
            Map.of("target", new YamlPattern("target", "lb", Direction.DEPENDENTS, 2, null)),
            Map.of(), Map.of(),
            "LB ${match.lb.id} must route to at least 2 targets");

    ResolvedInvariant invariant = YamlInvariantConverter.toDeclarativeInvariant(
            "lb-routing", yamlInv);

    var ex = assertThrows(GraphInvariantViolationsException.class,
            () -> engine.validate(graph, List.of(invariant)));
    assertThat(ex.violations()).hasSize(1);
}

@Test
void declarativeInvariant_noCardinality_existingBehaviorPreserved() {
    // Existing test with no cardinality — 3-arg YamlPattern constructor
    DesiredStateGraph graph = factory.of(
            List.of(
                    new DesiredNode(NodeId.of("sink-1"), new Spec("s1", "sink"), HumanGating.NONE)),
            List.of());

    YamlInvariant yamlInv = new YamlInvariant(
            List.of(),
            Map.of("sink", new YamlPattern("sink", null, Direction.DEPENDENCIES)),
            Map.of("upstream", new YamlPattern("transformer", "sink", Direction.DEPENDENCIES)),
            Map.of(), Map.of(), null);

    ResolvedInvariant invariant = YamlInvariantConverter.toDeclarativeInvariant(
            "every-sink-has-upstream", yamlInv);

    var ex = assertThrows(GraphInvariantViolationsException.class,
            () -> engine.validate(graph, List.of(invariant)));
    assertThat(ex.violations()).hasSize(1);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlInvariantEvaluationTest#declarativeInvariant_matchMinCount_violated`
Expected: Compilation error — `YamlPattern` has no 5-arg constructor

- [ ] **Step 3: Extend YamlPattern with cardinality fields**

Replace `YamlPattern.java`:

```java
package io.casehub.desiredstate.yaml.model;

import io.casehub.desiredstate.annotations.runtime.Direction;

public record YamlPattern(
        String type,
        String of,
        Direction direction,
        Integer minCount,
        Integer maxCount) {

    public YamlPattern {
        direction = direction != null ? direction : Direction.DEPENDENCIES;
    }

    public YamlPattern(String type, String of, Direction direction) {
        this(type, of, direction, null, null);
    }
}
```

- [ ] **Step 4: Update YamlInvariantConverter to pass cardinality through**

In `YamlInvariantConverter.toDeclarativeInvariant()`, replace the `PatternParameterDescriptor` construction for `match` entries:

```java
for (Map.Entry<String, YamlPattern> entry : yamlInvariant.match().entrySet()) {
    YamlPattern p = entry.getValue();
    int min = p.minCount() != null ? p.minCount() : PatternParameterDescriptor.UNSPECIFIED;
    int max = p.maxCount() != null ? p.maxCount() : PatternParameterDescriptor.UNSPECIFIED;
    patterns.add(new PatternParameterDescriptor(
            PatternKind.MATCH, p.type(),
            p.of() != null ? p.of() : "",
            p.direction(), min, max));
    bindingNamesList.add(entry.getKey());
}
```

In `addPatterns()`, same change:

```java
private static void addPatterns(Map<String, YamlPattern> section, PatternKind kind,
        List<PatternParameterDescriptor> patterns, List<String> bindingNames) {
    for (Map.Entry<String, YamlPattern> entry : section.entrySet()) {
        YamlPattern p = entry.getValue();
        int min = p.minCount() != null ? p.minCount() : PatternParameterDescriptor.UNSPECIFIED;
        int max = p.maxCount() != null ? p.maxCount() : PatternParameterDescriptor.UNSPECIFIED;
        patterns.add(new PatternParameterDescriptor(
                kind, p.type(),
                p.of() != null ? p.of() : "",
                p.direction(), min, max));
        bindingNames.add(entry.getKey());
    }
}
```

- [ ] **Step 5: Fix existing YamlPattern call sites**

The 3-arg constructor is preserved, so existing call sites in tests and production code continue to work. Verify by checking that `YamlRuleConverter` (if it exists) and existing YAML tests compile.

- [ ] **Step 6: Run all YAML tests**

Run: `mvn --batch-mode test -pl yaml/runtime`
Expected: All PASS

- [ ] **Step 7: Add YAML build-time validation for cardinality**

In `YamlDesiredStateProcessor.validateInvariants()` (line 497), add cardinality validation after existing pattern checks. Add a private helper:

```java
private void validatePatternCardinality(YamlPattern p, String ctx) {
    if (p.minCount() != null && p.minCount() < 0) {
        throw new RuntimeException(ctx + ": invalid minCount: " + p.minCount());
    }
    if (p.maxCount() != null && p.maxCount() < 0) {
        throw new RuntimeException(ctx + ": invalid maxCount: " + p.maxCount());
    }
    if (p.minCount() != null && p.maxCount() != null && p.minCount() > p.maxCount()) {
        throw new RuntimeException(ctx + ": minCount (" + p.minCount()
                + ") > maxCount (" + p.maxCount() + ")");
    }
}
```

Call it for each match, directDep, and reaches pattern in `validateInvariants()`. For notExists patterns, validate that no cardinality is specified:

```java
for (Map.Entry<String, YamlPattern> ne : inv.notExists().entrySet()) {
    if (ne.getValue().minCount() != null || ne.getValue().maxCount() != null) {
        throw new RuntimeException(ctx + ".notExists." + ne.getKey()
                + ": notExists does not support minCount/maxCount");
    }
}
```

- [ ] **Step 8: Run full module tests**

Run: `mvn --batch-mode test -pl yaml/runtime,yaml/deployment`
Expected: All PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/desiredstate add yaml/
git -C /Users/mdproctor/claude/casehub/desiredstate commit -m "feat(#127): YAML cardinality — YamlPattern minCount/maxCount + converter + validation"
```

---

## Batch 4: Integration + Full Build

### Task 7: Add cardinality invariants to pipeline examples

**Files:**
- Modify: `examples/pipeline-yaml/src/main/resources/META-INF/desiredstate/medallion-pipeline.yaml`
- Test: existing pipeline-yaml integration tests

**Interfaces:**
- Consumes: Full stack from Tasks 1–6

- [ ] **Step 1: Add a cardinality invariant to the YAML pipeline**

Add to the `invariants:` section of `medallion-pipeline.yaml`:

```yaml
  minimum-data-sources:
    match:
      source: { type: data-source, minCount: 1 }
    message: "Pipeline requires at least one data source"
```

- [ ] **Step 2: Run the pipeline-yaml example tests**

Run: `mvn --batch-mode test -pl examples/pipeline-yaml`
Expected: PASS — the pipeline has multiple data sources, so `minCount: 1` holds

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/desiredstate add examples/pipeline-yaml/
git -C /Users/mdproctor/claude/casehub/desiredstate commit -m "feat(#127): add cardinality invariant to pipeline-yaml example"
```

### Task 8: Full build verification

**Files:** None — verification only

- [ ] **Step 1: Run full project build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules compile and all tests pass

- [ ] **Step 2: Commit any fixes if needed**

If any downstream modules broke due to the `PatternParameterDescriptor` record change (e.g., pattern matching in switch expressions), fix and commit.

---

## References

- [2026-08-31-count-based-invariants-design.md] — design spec this plan implements
- `PatternParameterDescriptor.java` (annotations/runtime:3) — extended with cardinality
- `GraphInvariantEngine.java` (annotations/runtime:14) — engine evaluation logic
- `@Match`, `@DirectDep`, `@Reaches` (annotations/runtime) — annotation attributes
- `YamlPattern.java` (yaml/runtime:5) — YAML model
- `YamlInvariantConverter.java` (yaml/runtime:13) — YAML→descriptor bridge
- `DesiredStateAnnotationsProcessor.java:527` — `buildPatternForParameter` extraction
- `AnnotationValidationStep.java:357` — `validatePatternParameters` validation
- `YamlDesiredStateProcessor.java:497` — YAML invariant validation
- GitHub #127 — focal issue
