# @GraphRule — Declarative Graph Rewriting During Compilation

**Date:** 2026-08-24
**Issue:** casehubio/casehub-desiredstate#106
**Status:** Draft

## Motivation

Static node declarations and `@GoalMethod` handle fixed and goal-driven graphs. Neither
handles implied structure — "every transformer needs a downstream validator," "every zone
needs a load balancer." These patterns require graph rewriting rules that fire during
compilation, growing and pruning the graph until it converges.

@GraphRule adds a fixed-point rewriting phase to the annotation-driven compilation pipeline.
Two signatures serve different complexity levels: parameterized rules (engine does the
matching) and imperative rules (developer does the matching).

## Scope

**In scope:**
- `@GraphRule` annotation (dual target: TYPE for standalone classes, METHOD for interface/standalone rules)
- `@Match`, `@DirectDep`, `@Reaches`, `@NotExists` parameter annotations
- `Direction` enum (DEPENDENCIES, DEPENDENTS)
- `GraphRuleEngine` — pattern matching + fixed-point loop in `annotations/runtime`
- `GraphRuleDescriptor`, `PatternParameterDescriptor`, `PatternKind` IR types
- `GraphDescriptor` extended with `List<GraphRuleDescriptor>`
- Processor integration: Jandex scan, standalone class discovery, wildcard graph matching
- Build-time validation in `AnnotationValidationStep`
- Recorder integration: rule resolution and engine invocation
- Conflict detection via `ConflictingMutationException`
- Non-convergence detection via `GraphRuleNonConvergenceException`
- Tests: engine unit tests, processor integration, validation errors, pipeline-annotated example

**Out of scope:**
- `@GraphInvariant` (#107) — separate issue, runs after @GraphRule convergence
- Shared pattern matching infrastructure with FaultPolicy (D10 — separate mechanisms)
- YAML or TypeScript rule declarations
- Runtime rule evaluation (rules fire at compile time only)

---

## Part 1: Compilation Pipeline Integration

@GraphRule slots into the existing compilation pipeline after @GoalMethod:

```
Discover @Node / @DeclareNode → Assemble initial graph → Resolve dependencies
  → Apply @Customize(graph) methods
  → Apply @GoalMethod (if present)
  → Fire @GraphRule methods (fixed-point loop)          ← NEW
  → [future: @GraphInvariant (#107)]
  → Emit final DesiredStateGraph
```

The fixed-point loop runs at runtime init (same phase as the recorder), because rule
methods need access to instantiated classes. The build step scans annotations, validates
signatures, and builds descriptors; the recorder resolves methods and invokes the engine.

For the @GoalMethod path, the rule engine wraps the GoalCompiler lambda — rules fire on
every `compile()` call, after the goal method produces its graph.

---

## Part 2: Annotations

### @GraphRule

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface GraphRule {
    String graph() default "";  // standalone classes: "namespace:name" or "namespace:*"
}
```

On `ElementType.TYPE`: marks a standalone rule container class. `graph` attribute scopes
all rules in the class to matching graphs. Wildcard: `"pipeline:*"` matches any graph
with namespace `"pipeline"`. Empty string (default) is invalid on standalone classes.

On `ElementType.METHOD`: marks a method as a graph rule.
- On `@DesiredState` interfaces: must be a `static` method. `graph` attribute ignored
  (scoped to the declaring interface's graph).
- On standalone classes: must be a `public` method. `graph` attribute ignored (inherited
  from class-level annotation).

### @Match

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface Match {
    String type();  // NodeType value
}
```

Binds a node of the given type. Independent — does not reference a previous binding.
Multiple `@Match` parameters create a cross-product of all matching nodes.

### @DirectDep

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface DirectDep {
    String type();
    String of() default "";
    Direction direction() default Direction.DEPENDENCIES;
}
```

Binds a node that is a direct edge neighbor of the previous or named binding.
- `direction = DEPENDENCIES` (default): checks `dependenciesOf()` — "what does the binding depend on?"
- `direction = DEPENDENTS`: checks `dependentsOf()` — "what depends on the binding?"
- `of`: names a binding by Java parameter name. Empty = sequential chaining (previous parameter).

### @Reaches

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface Reaches {
    String type();
    String of() default "";
    Direction direction() default Direction.DEPENDENCIES;
}
```

Binds a node that is transitively reachable from the previous or named binding.
- `direction = DEPENDENCIES` (default): walks dependency chain toward roots.
- `direction = DEPENDENTS`: walks dependent chain toward leaves.
- `of`: names a binding by Java parameter name. Empty = sequential chaining.

Reachability is computed via BFS from the reference binding, following edges in the
specified direction, stopping at the first node matching the target type.

### @NotExists

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface NotExists {
    String type();
    String of() default "";
    Direction direction() default Direction.DEPENDENCIES;
}
```

Absence guard — prevents the rule from firing when the pattern exists.

- **Global mode** (`of` empty): fires only if NO node of the target type exists in the
  entire graph. No parameter is bound — the Java parameter type must be `Void`.
- **Relational mode** (`of` non-empty): fires only if no node of the target type is
  found as an edge neighbor of the named binding in the specified direction. Rule authors
  should always specify `direction` explicitly when using `of` — the default DEPENDENCIES
  applies if omitted, but the build step emits a warning to encourage explicit direction
  for readability.

### Direction

```java
public enum Direction {
    DEPENDENCIES,  // forward edges: dependenciesOf()
    DEPENDENTS     // reverse edges: dependentsOf()
}
```

Lives in `annotations/runtime` (annotation-specific concept, not in api/).

---

## Part 3: Programming Model

### Interface rules (static methods)

```java
@DesiredState(namespace = "pipeline", name = "medallion")
public interface MedallionPipeline {

    @Node("tx") default TransformerSpec tx() { ... }
    @Node("sink") default SinkSpec sink() { ... }

    // Parameterized: ensure every transformer has a downstream validator
    @GraphRule
    static List<GraphMutation> ensureValidator(
            @Match(type = "transformer") DesiredNode transformer,
            @NotExists(type = "validator", of = "transformer",
                       direction = Direction.DEPENDENTS) Void guard) {
        return GraphMutations.addNodeDependingOn(
            new DesiredNode(NodeId.of("validator-" + transformer.id().value()),
                new ValidatorSpec(...), HumanGating.NONE),
            transformer.id());
    }

    // Imperative: full graph access
    @GraphRule
    static List<GraphMutation> rebalanceZones(DesiredStateGraph graph) {
        // arbitrary logic over the whole graph
        return List.of();
    }
}
```

### Standalone rule containers

```java
@GraphRule(graph = "pipeline:*")
public class PipelineRules {

    @GraphRule
    public List<GraphMutation> ensureMonitoring(
            @Match(type = "sink") DesiredNode sink,
            @NotExists(type = "monitor", of = "sink",
                       direction = Direction.DEPENDENTS) Void guard) {
        return GraphMutations.addNodeDependingOn(
            new DesiredNode(NodeId.of("monitor-" + sink.id().value()),
                new MonitorSpec(...), HumanGating.NONE),
            sink.id());
    }

    @GraphRule
    public List<GraphMutation> enforceSchemaValidation(DesiredStateGraph graph) {
        // imperative rule across all pipeline graphs
        return List.of();
    }

    // NOT a rule — no @GraphRule annotation
    private List<GraphMutation> helper(DesiredStateGraph graph) { ... }
}
```

### Sequential chaining with named bindings

```java
@GraphRule
static List<GraphMutation> chainExample(
        @Match(type = "source") DesiredNode source,           // bind source
        @DirectDep(type = "ingest") DesiredNode ingest,       // sequential: direct dep of source
        @Reaches(type = "sink", of = "source") DesiredNode sink) {  // named: reachable from source
    // source → ingest (direct), source →...→ sink (transitive)
    return List.of();
}
```

---

## Part 4: IR Types

### New records in `annotations/runtime`

```java
public enum PatternKind {
    MATCH,
    DIRECT_DEP,
    REACHES,
    NOT_EXISTS
}

public record PatternParameterDescriptor(
    PatternKind kind,
    String nodeType,
    String of,
    Direction direction
) {}

public record GraphRuleDescriptor(
    String methodName,
    boolean imperative,
    List<PatternParameterDescriptor> patterns,
    String sourceClassName
) {}
```

### GraphDescriptor extension

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
    List<GraphRuleDescriptor> graphRules          // NEW
) {}
```

---

## Part 5: Pattern Matching Engine

### GraphRuleEngine

Lives in `annotations/runtime`. Core class:

```java
public class GraphRuleEngine {

    private static final int MAX_ITERATIONS = 100;

    public DesiredStateGraph evaluate(
            DesiredStateGraph graph,
            List<ResolvedGraphRule> rules) {

        for (int iteration = 0; iteration < MAX_ITERATIONS; iteration++) {
            List<GraphMutation> allMutations = new ArrayList<>();

            for (ResolvedGraphRule rule : rules) {
                List<GraphMutation> mutations = rule.imperative()
                    ? evaluateImperative(rule, graph)
                    : evaluateParameterized(rule, graph);
                allMutations.addAll(mutations);
            }

            if (allMutations.isEmpty()) {
                return graph;  // quiescent
            }

            detectConflicts(allMutations);
            graph = applyMutations(graph, allMutations);
        }

        throw new GraphRuleNonConvergenceException(rules, MAX_ITERATIONS);
    }
}
```

### ResolvedGraphRule

Runtime-resolved form of `GraphRuleDescriptor`:

```java
record ResolvedGraphRule(
    String name,
    Method method,
    Object instance,
    boolean imperative,
    List<PatternParameterDescriptor> patterns
) {}
```

- `instance` is `null` for static interface methods
- `method` is the resolved `java.lang.reflect.Method`

### Parameterized evaluation algorithm

1. Find all `@Match` parameters — for each, enumerate all graph nodes matching that type
2. Form the cross-product of all `@Match` bindings
3. For each candidate tuple, walk the parameter chain:
   - `@DirectDep`: resolve reference binding (previous param or named via `of`). Query
     `dependenciesOf()` or `dependentsOf()` based on `direction`. Filter by target type.
     If no match, discard this tuple.
   - `@Reaches`: resolve reference binding. BFS along edges in specified `direction`.
     Stop at first node matching target type. If none found, discard this tuple.
   - `@NotExists` (global, `of` empty): scan all graph nodes for target type.
     If found, discard this tuple.
   - `@NotExists` (relational, `of` non-empty): resolve named binding. Query edges in
     specified `direction` for target type. If found, discard this tuple.
4. For each surviving tuple, invoke the rule method with bound nodes (and `null` for
   `@NotExists` parameters of type `Void`)
5. Collect returned `List<GraphMutation>` into the iteration's mutation list

### Conflict detection

Group mutations by target `NodeId`:
- `AddNode` → `node.id()`
- `RemoveNode` → `id`
- `UpdateNode` → `id`
- `AddDependency` / `RemoveDependency` → no conflict (edges, not nodes)

If multiple distinct mutations target the same `NodeId` in one iteration (e.g., two rules
both `AddNode` with the same ID but different specs), throw `ConflictingMutationException`.
Duplicate identical mutations are deduplicated silently.

### Non-convergence

`GraphRuleNonConvergenceException` extends `RuntimeException`. Message includes:
- The iteration cap (100)
- Names of rules that produced mutations in the final iteration
- Diagnostic: "Non-converging rules are usually caused by non-idempotent mutations.
  Check that parameterized rules use @NotExists guards and imperative rules check
  graph state before producing mutations."

---

## Part 6: Processor Integration

### Build step changes

`DesiredStateAnnotationsProcessor.generateDesiredStateGraphs()`:

1. Scan @GraphRule methods on @DesiredState interfaces (static methods only)
2. Scan standalone @GraphRule classes via Jandex
3. Match standalone rules to graphs via `graph` attribute (exact match or wildcard)
4. Build `GraphRuleDescriptor` instances and add to `GraphDescriptor`

### Standalone class discovery

```java
private Map<String, List<GraphRuleDescriptor>> scanStandaloneGraphRules(IndexView index) {
    Map<String, List<GraphRuleDescriptor>> byGraphPattern = new HashMap<>();
    for (AnnotationInstance grAnn : index.getAnnotations(GRAPH_RULE)) {
        if (grAnn.target().kind() != AnnotationTarget.Kind.CLASS) continue;
        ClassInfo classInfo = grAnn.target().asClass();
        String graphPattern = grAnn.value("graph").asString();

        for (MethodInfo method : classInfo.methods()) {
            if (method.hasAnnotation(GRAPH_RULE) && isPublicMethod(method)) {
                byGraphPattern
                    .computeIfAbsent(graphPattern, k -> new ArrayList<>())
                    .add(buildGraphRuleDescriptor(method, index));
            }
        }
    }
    return byGraphPattern;
}
```

### Build-time validation

| Check | Error message |
|-------|---------------|
| @GraphRule on non-static interface method | `@GraphRule on 'ensureValidator' must be a static method` |
| Return type not `List<GraphMutation>` | `@GraphRule 'ensureValidator' must return List<GraphMutation>` |
| Imperative param not DesiredStateGraph | `@GraphRule 'rebalance' imperative method first parameter must be DesiredStateGraph` |
| @NotExists with `of` but default direction | Warning: `@NotExists on parameter 'guard' specifies 'of' without explicit direction — defaults to DEPENDENCIES; consider specifying direction for clarity` |
| `of` references unknown parameter name | `@DirectDep 'of' references 'foo' — no parameter named 'foo' in ensureValidator` |
| @GraphRule on non-public standalone method | `@GraphRule on 'ensureMonitor' in PipelineRules must be public` |
| Standalone class not instantiable | `@GraphRule class PipelineRules must be concrete with a no-arg constructor` |
| Standalone class missing `graph` attribute | `@GraphRule on class PipelineRules requires graph attribute` |
| @Match type not known | Warning: `@Match type 'unknown-type' not found in any @Node or @DeclareNode declaration` |
| Standalone graph key no match | Warning: `@GraphRule class PipelineRules graph 'foo:bar' does not match any declared graph` |

### Recorder changes

`DesiredStateGraphRecorder.createGoalCompiler()` — after building the graph (with
customizers and goal method applied):

```java
if (!descriptor.graphRules().isEmpty()) {
    List<ResolvedGraphRule> resolved = resolveRules(
        descriptor.graphRules(), implClass, instance, classLoader);
    GraphRuleEngine engine = new GraphRuleEngine();
    graph = engine.evaluate(graph, resolved);
}
```

For the @GoalMethod path, rules are evaluated inside the GoalCompiler lambda, after the
goal method produces its graph — ensuring rules fire on every compile() call.

---

## Part 7: Testing Strategy

### Unit tests (`annotations/runtime/`)

`GraphRuleEngineTest`:
- Single imperative rule modifies graph
- Single parameterized rule with @Match adds nodes
- Fixed-point convergence: rule fires, adds node, @NotExists guard prevents re-firing (2 iterations)
- @DirectDep with DEPENDENCIES and DEPENDENTS directions
- @Reaches transitive reachability (3+ hop chain)
- @NotExists global guard: rule skipped when type exists
- @NotExists relational guard with `of` and `direction`
- Multiple rules producing non-conflicting mutations in same iteration
- ConflictingMutationException: two rules add same NodeId with different specs
- Non-convergence: rule always produces mutations → exception at 100 iterations
- Duplicate identical mutations deduplicated silently
- Empty rule list → graph returned unchanged
- Named binding via `of` (non-sequential chaining)

### Build extension tests (`annotations/deployment/`)

`GraphRuleProcessorTest`:
- Imperative @GraphRule on interface → graph modified
- Parameterized @GraphRule with @Match → nodes added
- Fixed-point convergence in Quarkus app context
- Standalone @GraphRule class with exact graph match
- Standalone @GraphRule class with wildcard graph match
- Interface + standalone rules merged and both fire

`GraphRuleValidationTest`:
- Non-static interface method → error
- Wrong return type → error
- `of` references non-existent parameter → error
- @NotExists with `of` but no direction → error
- Non-public standalone method → error
- Non-instantiable standalone class → error
- Missing graph attribute on standalone class → error

### Integration test (`examples/pipeline-annotated/`)

Add @GraphRule methods to `MedallionPipeline`:
- Auto-add monitoring node for every sink (parameterized with @NotExists guard)
- Verify compiled graph contains rule-generated nodes and edges

---

## References

- [DesiredStateAnnotationsProcessor.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java) — existing build extension (extended by this design)
- [AnnotationValidationStep.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java) — existing validation (extended by this design)
- [DesiredStateGraphRecorder.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java) — existing recorder (extended by this design)
- [GraphMutation.java](/Users/mdproctor/claude/casehub/desiredstate/api/src/main/java/io/casehub/desiredstate/api/GraphMutation.java) — sealed mutation types (reused by rules)
- [GraphMutations.java](/Users/mdproctor/claude/casehub/desiredstate/api/src/main/java/io/casehub/desiredstate/api/GraphMutations.java) — mutation utilities (used in rule implementations)
- [ImmutableDesiredStateGraph.java](/Users/mdproctor/claude/casehub/desiredstate/runtime/src/main/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraph.java) — withMutation(), dependenciesOf(), dependentsOf()
- [#102 annotations design spec](/Users/mdproctor/claude/casehub/desiredstate/docs/specs/issue-102-desiredstate-annotations/2026-08-20-desiredstate-annotations-design.md) — foundation module design
- [#112 TypedFaultPolicy](/Users/mdproctor/claude/casehub/desiredstate/docs/specs/issue-112-reviewnodepolicy-tier/2026-08-24-typedfaultpolicy-design.md) — FaultPolicy boundary (D10)
- decisions.md — 10 design decisions with rationale
