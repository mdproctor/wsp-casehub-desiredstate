# Count-Based Graph Invariants (Cardinality Constraints)

**Date:** 2026-08-31
**Issue:** casehubio/casehub-desiredstate#127
**Status:** Draft

## Motivation

The `GraphInvariantEngine` validates structural patterns via `@Match` +
`@DirectDep`/`@Reaches`/`@NotExists`. These are boolean existence checks —
"node A must have a dependency of type B" or "no node of type C should exist
reachable from D."

There is no way to express cardinality constraints:

- "HA topology requires at least 3 compute instances" (count of matched nodes)
- "Load balancer must route to at least 2 target services" (count of dependencies)
- "Service mesh requires exactly 1 control plane" (singleton constraint)

Cardinality operates at two levels in the evaluation pipeline:

1. **Match-level** — how many nodes of a given type exist globally in the graph
2. **Expansion-level** — how many neighbors/reachable nodes exist per anchor

Both levels are addressed by adding `minCount`/`maxCount` attributes to the
existing pattern annotations and YAML model (D1).

---

## Part 1: Annotation Changes

### @Match

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface Match {
    String type();
    int minCount() default -1;
    int maxCount() default -1;
}
```

`-1` sentinel means "not specified" — the engine applies the default
(`minCount=0`, `maxCount=unlimited` for `@Match`).

### @DirectDep

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface DirectDep {
    String type();
    String of() default "";
    Direction direction();
    int minCount() default -1;
    int maxCount() default -1;
}
```

Default when not specified: `minCount=1`, `maxCount=unlimited`.

### @Reaches

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface Reaches {
    String type();
    String of() default "";
    Direction direction();
    int minCount() default -1;
    int maxCount() default -1;
}
```

Default when not specified: `minCount=1`, `maxCount=unlimited`.

### @NotExists

No changes. Cardinality on `@NotExists` is a build-time error — it is a
boolean guard, not a countable pattern.

---

## Part 2: PatternParameterDescriptor

```java
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

The 4-arg constructor preserves source compatibility for all existing call
sites — they continue to produce descriptors with `UNSPECIFIED` cardinality,
which resolves to the current defaults via `effectiveMinCount()`/`effectiveMaxCount()`.

---

## Part 3: GraphInvariantEngine Changes

### Match-level cardinality (D2)

When any `@Match` parameter has an explicit cardinality constraint, the
invariant becomes a count-only assertion. The engine counts nodes matching
each constrained `@Match` type and checks bounds. No cross-product, no
expansion chain, no method body invocation.

```java
private boolean hasMatchCardinalityConstraint(List<PatternParameterDescriptor> patterns) {
    return patterns.stream()
            .anyMatch(p -> p.kind() == PatternKind.MATCH && p.hasCardinalityConstraint());
}
```

In `validateParameterized()` and `validateDeclarative()`, before the existing
anchor/expansion logic:

```java
if (hasMatchCardinalityConstraint(patterns)) {
    validateMatchCardinality(invariantName, sourceClass, graph, patterns, violations);
    return;
}
// ... existing per-anchor expansion logic ...
```

```java
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

### Expansion-level cardinality (D3)

For `@DirectDep`/`@Reaches` with explicit cardinality, the engine counts
expansions per anchor and checks bounds. This replaces the existing boolean
"zero expansions = structural violation" check.

In the per-anchor evaluation loop, after collecting expansions:

```java
for (List<DesiredNode> anchor : expectedAnchors) {
    List<Map<String, DesiredNode>> expansions = byAnchor.get(anchor);
    int expansionCount = expansions != null ? expansions.size() : 0;

    // Check expansion-level cardinality for non-MATCH patterns
    for (int i = 0; i < patterns.size(); i++) {
        PatternParameterDescriptor p = patterns.get(i);
        if (p.kind() == PatternKind.MATCH || p.kind() == PatternKind.NOT_EXISTS) continue;
        if (!p.hasCardinalityConstraint() && expansionCount == 0) {
            // Default behavior: zero expansions = structural violation
            violations.add(structuralViolation(invariantName, sourceClass, anchor));
            break;
        }
        if (p.hasCardinalityConstraint()) {
            // Count expansions for this specific pattern binding
            long bindingCount = countExpansionsForBinding(expansions, bindingNames[i]);
            if (bindingCount < p.effectiveMinCount()) {
                violations.add(cardinalityViolation(invariantName, sourceClass, anchor,
                        p, bindingCount, "at least", p.effectiveMinCount()));
            }
            if (bindingCount > p.effectiveMaxCount()) {
                violations.add(cardinalityViolation(invariantName, sourceClass, anchor,
                        p, bindingCount, "at most", p.effectiveMaxCount()));
            }
        }
    }

    // Invoke method body per expansion for value checks (unchanged)
    if (expansions != null) {
        for (Map<String, DesiredNode> binding : expansions) {
            invokeReflectiveInvariant(invariant, buildArgs(binding, paramNames), violations);
        }
    }
}
```

The counting logic uses distinct values per binding name: for chained patterns
like `@DirectDep(minCount=2) → @Reaches`, a single anchor may produce many
fully-expanded tuples (2 targets x 3 endpoints = 6 tuples), but the cardinality
check counts distinct values bound to the constrained parameter's name — 2
distinct `target` bindings, not 6 tuples.

```java
long bindingCount = expansions.stream()
        .map(b -> b.get(bindingNames[constrainedIndex]))
        .distinct()
        .count();
```

---

## Part 4: YAML Surface (D4)

### YamlPattern

```java
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

    public int effectiveMinCount(PatternKind kind) {
        if (minCount != null) return minCount;
        return kind == PatternKind.MATCH ? 0 : 1;
    }

    public int effectiveMaxCount() {
        return maxCount != null ? maxCount : Integer.MAX_VALUE;
    }
}
```

`Integer` (boxed) allows Jackson to distinguish absent fields (`null`) from
explicit `0`. `YamlInvariantConverter` maps `null` to
`PatternParameterDescriptor.UNSPECIFIED` when constructing descriptors.
```

Jackson deserialises `minCount`/`maxCount` from YAML. Using `Integer` (boxed)
ensures absent fields deserialise as `null`, distinguishable from explicit `0`.

### YamlInvariantConverter

`toDeclarativeInvariant()` passes through `minCount`/`maxCount` from
`YamlPattern` to `PatternParameterDescriptor` via the 6-arg constructor.

### YAML examples

```yaml
invariants:
  ha-minimum-instances:
    match:
      instance: { type: compute_instance, minCount: 3 }
    message: "HA requires at least 3 compute instances"

  lb-routing:
    match:
      lb: { type: load_balancer }
    directDep:
      target: { type: target, of: lb, direction: DEPENDENTS, minCount: 2 }
    message: "Load balancer ${match.lb.id} must route to at least 2 targets"

  singleton-control-plane:
    match:
      cp: { type: control_plane, minCount: 1, maxCount: 1 }
    message: "Exactly one control plane required"
```

---

## Part 5: Build-Time Validation

In `AnnotationValidationStep`, add checks for cardinality attributes:

| Check | Error message |
|-------|---------------|
| `@NotExists` with `minCount` or `maxCount` | `@NotExists on parameter 'guard' does not support minCount/maxCount — it is a boolean guard` |
| `minCount < 0` (excluding -1 sentinel) | `@Match on parameter 'instance' has invalid minCount: -2` |
| `maxCount < 0` (excluding -1 sentinel) | Same pattern |
| `minCount > maxCount` (both specified) | `@Match on parameter 'instance' has minCount (5) > maxCount (3)` |
| `maxCount = 0` on `@DirectDep`/`@Reaches` | Warning: `maxCount=0 on @DirectDep 'target' means no expansions allowed — consider @NotExists instead` |

YAML build-time validation (`YamlDesiredStateValidator` or equivalent) applies
the same checks when processing `YamlInvariant` patterns.

---

## Part 6: Annotation Examples

### Match-level: minimum count

```java
@GraphInvariant
static void haMinimumInstances(
        @Match(type = "compute_instance", minCount = 3) DesiredNode instance) {}
```

### Match-level: singleton constraint

```java
@GraphInvariant
static void singleControlPlane(
        @Match(type = "control_plane", minCount = 1, maxCount = 1) DesiredNode cp) {}
```

### Expansion-level: minimum dependencies

```java
@GraphInvariant
static void lbMinTargets(
        @Match(type = "load_balancer") DesiredNode lb,
        @DirectDep(type = "target", of = "lb",
                   direction = Direction.DEPENDENTS, minCount = 2) DesiredNode target) {}
```

### Expansion-level: maximum reachable

```java
@GraphInvariant
static void limitedFanout(
        @Match(type = "router") DesiredNode router,
        @Reaches(type = "endpoint", of = "router",
                 direction = Direction.DEPENDENTS, maxCount = 10) DesiredNode endpoint) {}
```

---

## Part 7: Declarative Invariant Message Templates

For match-level cardinality violations on `DeclarativeInvariant`, the message
template cannot use `${match.X.id}` substitution — there is no single anchor
node. The engine uses the raw message string, or falls back to the generated
message ("expected at least N node(s) of type 'X', found M").

For expansion-level cardinality violations, `${match.X.id}` substitution
works as today — the anchor is bound.

---

## Part 8: Testing Strategy

### Unit tests (`annotations/runtime/`)

**GraphInvariantEngineTest — match-level cardinality:**
- `@Match(minCount=3)` with 2 nodes → violation
- `@Match(minCount=3)` with 3 nodes → passes
- `@Match(minCount=3)` with 5 nodes → passes
- `@Match(maxCount=1)` with 2 nodes → violation
- `@Match(minCount=1, maxCount=1)` with 0 nodes → minCount violation
- `@Match(minCount=1, maxCount=1)` with 1 node → passes
- `@Match(minCount=1, maxCount=1)` with 2 nodes → maxCount violation
- Multiple `@Match` params with cardinality → each checked independently
- `@Match` with cardinality — method body NOT invoked (count-only)

**GraphInvariantEngineTest — expansion-level cardinality:**
- `@DirectDep(minCount=2)` with 1 expansion → violation
- `@DirectDep(minCount=2)` with 3 expansions → passes, method invoked 3x
- `@Reaches(maxCount=5)` with 6 reachable → violation
- `@DirectDep(minCount=2)` with 0 expansions → minCount violation
- Default `@DirectDep` (no cardinality) with 0 expansions → structural violation (unchanged)
- Default `@DirectDep` (no cardinality) with 1+ expansions → passes (unchanged)

**GraphInvariantEngineTest — declarative (YAML-originated):**
- `DeclarativeInvariant` with match-level cardinality → count check, message template
- `DeclarativeInvariant` with expansion-level cardinality → per-anchor count check

### Build extension tests (`annotations/deployment/`)

**AnnotationValidationTest — cardinality validation:**
- `@NotExists` with `minCount` → build error
- `minCount > maxCount` → build error
- Negative `minCount` → build error
- `maxCount=0` on `@DirectDep` → build warning

### YAML tests (`yaml/runtime/`)

**YamlInvariantEvaluationTest — cardinality:**
- YAML invariant with `minCount` on match → count check
- YAML invariant with `minCount` on directDep → per-anchor count
- YAML `minCount`/`maxCount` absent → existing behavior preserved

### Integration tests (`examples/pipeline-annotated/`, `examples/pipeline-yaml/`)

- Add a cardinality invariant to the pipeline-annotated example
- Add a cardinality invariant to the pipeline-yaml example
- Verify both surfaces produce the same validation behavior

---

## References

- `GraphInvariantEngine.java` (annotations/runtime) — engine being extended
- `PatternParameterDescriptor.java` (annotations/runtime) — descriptor being extended
- `PatternEvaluator.java` (annotations/runtime) — expansion logic
- `PatternMatchingSupport.java` (annotations/runtime) — shared matching primitives
- `@Match`, `@DirectDep`, `@Reaches`, `@NotExists` annotations (annotations/runtime)
- `YamlPattern.java` (yaml/runtime) — YAML model being extended
- `YamlInvariantConverter.java` (yaml/runtime) — converter passing through new fields
- `YamlInvariantEvaluationTest.java` (yaml/runtime) — existing YAML invariant tests
- `GraphInvariantEngineTest.java` (annotations/runtime) — existing engine tests
- Issue #115 spec — `GraphInvariantEngine` architecture, parameterized evaluation, D1–D8
- Issue #127 — this feature request
- decisions.md — D1–D4 design decisions with rationale
