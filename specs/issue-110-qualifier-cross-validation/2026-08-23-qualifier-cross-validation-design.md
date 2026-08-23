# GoalCompiler Qualifier + Cross-Model Validation — Design Spec

**Date:** 2026-08-23
**Issues:** casehubio/casehub-desiredstate#110, casehubio/casehub-desiredstate#111
**Status:** Draft

## Motivation

Issue #105 implemented `@DeclareNode` class-based nodes, graph merge, and `@DependsOn(nodes=...)`.
Two gaps remain:

1. **#110 — GoalCompiler qualifier:** `@DesiredStateQualifier` exists but is never applied to
   synthetic GoalCompiler beans. Multiple graphs produce multiple unqualified beans → CDI
   `AmbiguousResolutionException`. The qualifier must be applied without breaking single-graph
   apps.

2. **#111 — Cross-model validation:** The validator runs per-interface and per-class independently.
   No merged node set exists — duplicate node IDs across models, cross-model string ref
   resolution, and cross-model cycle detection are all missing.

Additionally, `@DependsOn(nodes=...)` on interface `@Node` methods is specified in the #105 spec
(Part 2 table) but never implemented — the processor crashes with NPE when only the `nodes`
attribute is specified.

---

## Part 1: Processor Changes (#110 + bug fixes)

### 1.1 GoalCompiler qualifier application

Always register both `@Default` and `@DesiredStateQualifier(namespace, name)` on every
GoalCompiler synthetic bean:

```java
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
                    .addQualifier(Default.class)
                    .addQualifier()
                        .annotation(DesiredStateQualifier.class)
                        .addValue("namespace", namespace)
                        .addValue("name", name)
                        .done()
                    .done());
}
```

**CDI behavior:**

| Scenario | `@Inject GoalCompiler` | `@Inject @DesiredStateQualifier(...) GoalCompiler` |
|----------|------------------------|---------------------------------------------------|
| 1 graph | Resolves via `@Default` | Resolves via qualifier |
| 2+ graphs | `AmbiguousResolutionException` (lists beans with qualifier values) | Resolves the specific graph |

Both call sites in `generateDesiredStateGraphs()` pass namespace and name:
1. Interface-sourced graphs: `descriptor.namespace()`, `descriptor.name()`
2. Class-only graphs: `ns`, `nm` from the graph key

FaultPolicy beans remain unqualified — they are additive and filtered by `nodeTypes` at runtime.

### 1.2 @DependsOn(nodes=...) on interface methods — processor fix

In `buildGraphDescriptor()`, handle both `value` and `nodes` attributes for interface methods:

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

This mirrors the existing `resolveClassDependencies()` logic for `@DeclareNode` classes.

### 1.3 Fault policy merge fix

When `@DeclareNode` classes merge into an interface-sourced graph, class-level fault policies
are currently dropped. The merge path builds a new `GraphDescriptor` with only the interface's
`descriptor.faultPolicies()`, ignoring class-sourced policies.

Fix in `generateDesiredStateGraphs()` merge path:

```java
List<FaultPolicyDescriptor> mergedPolicies = new ArrayList<>(descriptor.faultPolicies());
mergedPolicies.addAll(collectClassFaultPolicies(classNodes, index));

descriptor = new GraphDescriptor(descriptor.namespace(), descriptor.name(),
        descriptor.interfaceName(), descriptor.implClassName(),
        mergedNodes, mergedDeps, mergedPolicies, descriptor.goalMethod());
```

---

## Part 2: Validation Restructuring (#111)

### Current state

Validation runs in two independent loops with no shared state:
1. Per `@DesiredState` interface: `validateNodeMethod()`, `validateDependsOnRefs()`,
   `detectCycles()` — scoped to one interface's `nodeIds`
2. Per `@DeclareNode` class: `validateDeclareNodes()` — per-class constraints only

Problems:
- `validateDependsOnRefs()` rejects legitimate cross-model string refs (e.g.,
  `@DependsOn("class-node-id")` on an interface method) before cross-model validation can
  verify them
- No duplicate ID detection across models
- No cross-model cycle detection
- Index scanned twice if both phases scan independently

### Architecture — single scan, two validation passes

A single scan builds `MergedGraph` per graph key. Per-element structural checks run during the
scan. Reference resolution, duplicate detection, and cycle detection run after the scan
completes, operating on the merged structure.

```java
void validate(CombinedIndexBuildItem indexBuildItem) {
    IndexView index = indexBuildItem.getIndex();
    List<String> errors = new ArrayList<>();
    List<String> warnings = new ArrayList<>();

    // --- Single scan: collect + per-element validation ---
    Map<String, MergedGraph> graphsByKey = new LinkedHashMap<>();
    Set<String> interfacesByKey = new HashMap<>();  // graphKey → interface names

    for (AnnotationInstance dsAnn : index.getAnnotations(DESIRED_STATE)) {
        ClassInfo dsClass = dsAnn.target().asClass();
        String graphKey = resolveGraphKey(dsAnn, index);

        // Duplicate @DesiredState interface detection
        if (interfacesByKey.containsKey(graphKey)) {
            errors.add("Multiple @DesiredState interfaces with graph key '"
                + graphKey + "': " + interfacesByKey.get(graphKey)
                + " and " + dsClass.name().local()
                + " — use a single interface per graph,"
                + " with @DeclareNode classes for extension nodes");
            continue;
        }
        interfacesByKey.put(graphKey, dsClass.name().local());

        MergedGraph mg = graphsByKey.computeIfAbsent(graphKey, k -> new MergedGraph(graphKey));

        // Per-element structural checks (unchanged)
        if (!isInterface(dsClass)) { errors.add(...); continue; }

        for (MethodInfo method : dsClass.methods()) {
            validateNodeMethod(method, index, mg, errors, warnings);
            validateFaultPolicyOnMethod(method, dsClass, index, errors);
        }
        validateFaultPolicyFaultTypes(dsClass, index, errors);
        validateTierReviewMethods(dsClass, index, errors);
        validateGoalMethod(dsClass, index, errors);

        if (mg.nodeCount() == 0 from this interface) {
            warnings.add("@DesiredState '" + dsClass.name().local()
                + "' has no @Node methods — graph will be empty");
        }
    }

    validateDeclareNodes(index, graphsByKey, errors, warnings);

    // --- Cross-model validation on merged structure ---
    for (MergedGraph mg : graphsByKey.values()) {
        mg.validateDuplicateIds(errors);
        mg.validateDependencyRefs(errors);
        mg.detectCycles(errors);
    }

    for (String warning : warnings) { LOG.warn(warning); }
    if (!errors.isEmpty()) {
        throw new RuntimeException(
            "Annotation validation failed:\n- " + String.join("\n- ", errors));
    }
}
```

### Key change: validateNodeMethod() collects into MergedGraph

`validateNodeMethod()` gains a `MergedGraph` parameter. It performs per-element structural
checks (default method, return type implements NodeSpec) and collects into the merged
structure — but does NOT validate ref resolution or detect cycles. Those move to
`MergedGraph.validateDependencyRefs()` and `MergedGraph.detectCycles()`.

```java
private void validateNodeMethod(MethodInfo method, IndexView index,
        MergedGraph mg, List<String> errors, List<String> warnings) {
    AnnotationInstance nodeAnn = method.annotation(NODE);
    if (nodeAnn == null) return;

    String nodeId = nodeAnn.value().asString();
    String source = "interface method " + method.declaringClass().name().local()
                    + "#" + method.name();

    // Per-element checks (unchanged)
    if (!isDefaultMethod(method)) {
        errors.add("@Node on '" + method.name()
            + "' must be a default method returning NodeSpec");
    }
    if (!implementsNodeSpec(method.returnType().name(), index)) {
        errors.add("@Node '" + method.name() + "' return type "
            + method.returnType().name().local() + " does not implement NodeSpec");
    }

    // Collect into MergedGraph
    mg.addNode(nodeId, source);

    // Collect dependencies — null-safe, handles both value and nodes
    collectDeps(method, index, nodeId, mg, errors, warnings);
}
```

### validateDependsOnRefs() and detectCycles() — REMOVED from Phase 1

These methods are removed from the per-interface loop. Their functionality is replaced by
`MergedGraph.validateDependencyRefs()` and `MergedGraph.detectCycles()`, which operate on
the merged node set and correctly handle cross-model references.

### validateDeclareNodes() — collects into MergedGraph

Gains `graphsByKey` parameter. After per-class structural checks, collects nodes and
dependencies into the appropriate `MergedGraph`:

```java
private void validateDeclareNodes(IndexView index,
        Map<String, MergedGraph> graphsByKey,
        List<String> errors, List<String> warnings) {
    for (AnnotationInstance dnAnn : index.getAnnotations(DECLARE_NODE)) {
        ClassInfo classInfo = dnAnn.target().asClass();
        String className = classInfo.name().local();

        // Per-class structural checks (unchanged)
        if (isInterface(classInfo)) { errors.add(...); continue; }
        if (isAbstract(classInfo))  { errors.add(...); continue; }
        if (!implementsNodeSpec(classInfo.name(), index)) { errors.add(...); continue; }
        if (classInfo.hasAnnotation(DESIRED_STATE)) { errors.add(...); }

        // Annotation misuse: @GoalMethod, @Node, @Customize on @DeclareNode
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
        if (classInfo.hasAnnotation(CUSTOMIZE)) {
            errors.add("@Customize on @DeclareNode class '" + className
                + "' — @Customize requires a @DesiredState interface");
        }

        // Collect into MergedGraph
        String graphKey = resolveGraphKey(dnAnn, index);
        MergedGraph mg = graphsByKey.computeIfAbsent(graphKey, k -> new MergedGraph(graphKey));
        String nodeId = dnAnn.value("id").asString();
        mg.addNode(nodeId, "@DeclareNode class " + className);

        // Collect dependencies + @DependsOn(nodes) target validation
        collectClassDeps(classInfo, index, nodeId, mg, errors, warnings);
    }
}
```

### collectDeps() — shared helper for @DependsOn handling

Handles both string and class ref dependencies, validates `nodes` targets:

```java
private void collectDeps(AnnotationTarget target, IndexView index,
        String sourceNodeId, MergedGraph mg,
        List<String> errors, List<String> warnings) {
    AnnotationInstance dependsOnAnn = target.annotation(DEPENDS_ON);
    if (dependsOnAnn == null) return;

    // String dependencies
    AnnotationValue stringDeps = dependsOnAnn.value();
    if (stringDeps != null) {
        for (String dep : stringDeps.asStringArray()) {
            mg.addDependency(sourceNodeId, dep);
        }
    }

    // Class ref dependencies + target validation
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
                String targetId = targetClass.declaredAnnotation(DECLARE_NODE)
                                             .value("id").asString();
                mg.addDependency(sourceNodeId, targetId);
            }
        }
    }
}
```

### MergedGraph

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

    // hasCycle() — same algorithm as existing detectCycles, unchanged
}
```

### Cross-model validations summary

| Check | Error message | Phase |
|-------|---------------|-------|
| Duplicate `@DesiredState` interfaces per graphKey | `Multiple @DesiredState interfaces with graph key 'infra:zones': BaseInfra and ExtInfra — use a single interface per graph, with @DeclareNode classes for extension nodes` | Scan |
| Duplicate node ID across models | `Duplicate node id 'lb' in graph 'infra:zones' — declared on @DeclareNode class LoadBalancer and interface method BaseInfra#lbNode` | Cross-model |
| Cross-model string ref unresolved | `@DependsOn on 'lbNode' in graph 'infra:zones' references 'missing-id' which is not declared as @Node or @DeclareNode` | Cross-model |
| Cross-model circular dependency | `Circular dependency detected in graph 'infra:zones': lb → dns → lb` | Cross-model |
| @DependsOn(nodes) target not in Jandex | `@DependsOn(nodes) on 'Vpc' references 'DnsRecord' which is not in the Jandex index (if the class is in an external JAR, ensure a Jandex index is generated)` — **warning** | Scan |
| @DependsOn(nodes) target lacks @DeclareNode | `@DependsOn(nodes) on 'Vpc' references 'DnsRecord' which has no @DeclareNode annotation` — **error** | Scan |
| @Customize on @DeclareNode class | `@Customize on @DeclareNode class 'Foo' — @Customize requires a @DesiredState interface` | Scan |

---

## Testing Strategy

### Part 1 tests — Qualifier + processor fixes

**QualifierSingleGraphTest:**
- Single `@DesiredState` → GoalCompiler has `@DesiredStateQualifier(namespace, name)`
- Unqualified `@Inject GoalCompiler` resolves
- Qualified `@Inject @DesiredStateQualifier(...) GoalCompiler` also resolves

**QualifierMultiGraphTest:**
- Two `@DesiredState` interfaces with different (namespace, name)
- Each injectable via `@DesiredStateQualifier(...)`
- Verify each produces correct graph nodes

**QualifierClassOnlyTest:**
- Class-only graph → GoalCompiler has `@DesiredStateQualifier` with correct namespace/name

**InterfaceNodesDependencyTest:**
- `@Node` method with `@DependsOn(nodes = SomeClass.class)` → dependency resolved
- `@Node` method with `@DependsOn(nodes = ...)` only (no `value`) → no NPE, dependency resolved
- Mixed `@DependsOn(value = "x", nodes = {A.class})` on interface method → both resolved

**MergedFaultPolicyTest:**
- `@DeclareNode` class with `@FaultPolicyDef` merges into interface graph → policy preserved

### Part 2 tests — Cross-model validation

**CrossModelDuplicateIdTest:**
- Same node ID in `@Node` method and `@DeclareNode` class, same graph → build error

**CrossModelStringRefTest:**
- Interface `@DependsOn("class-node-id")` → resolves across models (no error)
- `@DeclareNode` `@DependsOn("interface-node-id")` → resolves across models
- `@DependsOn("nonexistent")` → build error naming the unresolved ref and graph

**CrossModelCycleTest:**
- Interface node depends on class node, class node depends back → build error with cycle path

**DuplicateDesiredStateInterfaceTest:**
- Two `@DesiredState` interfaces with same (namespace, name) → build error

**JandexWarningTest:**
- `@DependsOn(nodes = SomeClass.class)` where `SomeClass` not in index → warning
- `@DependsOn(nodes = SomeClass.class)` where `SomeClass` lacks `@DeclareNode` → error

**CustomizeOnDeclareNodeTest:**
- `@Customize` on `@DeclareNode` class → build error

### Existing tests — no regression

All existing tests (`DesiredStateAnnotationsProcessorTest`, `ClassBasedNodeTest`,
`ClassBasedDependencyTest`, `ClassBasedFaultPolicyTest`, `ClassBasedValidationTest`,
`MergedGraphTest`, `FaultPolicyWiringTest`, `GoalMethodCompositionTest`, `ValidationErrorTest`)
must pass without modification.

---

## Migration Impact

| Component | Change |
|-----------|--------|
| `DesiredStateAnnotationsProcessor.registerGoalCompilerBean()` | Add `@Default` + `@DesiredStateQualifier`. Accept namespace/name. |
| `DesiredStateAnnotationsProcessor.buildGraphDescriptor()` | Null-safe `@DependsOn` `value()` + `nodes` class ref resolution on interface methods. |
| `DesiredStateAnnotationsProcessor.generateDesiredStateGraphs()` | Merge class fault policies in the interface+class merge path. |
| `AnnotationValidationStep.validate()` | Single-scan architecture: build MergedGraph per graphKey, cross-model validation after scan. |
| `AnnotationValidationStep.validateNodeMethod()` | Collect into MergedGraph. Null-safe @DependsOn. No ref validation (moved to MergedGraph). |
| `AnnotationValidationStep.validateDependsOnRefs()` | **Removed** — replaced by `MergedGraph.validateDependencyRefs()`. |
| `AnnotationValidationStep.detectCycles()` | **Removed** — replaced by `MergedGraph.detectCycles()`. |
| `AnnotationValidationStep.validateDeclareNodes()` | Collect into MergedGraph. Add `@Customize` check. `@DependsOn(nodes)` target validation. |
| `AnnotationValidationStep` (new) | `MergedGraph` inner class, `collectDeps()` shared helper, `resolveGraphKey()`. |

No changes to: runtime module, api module, recorder, examples, or any module outside `annotations/`.

---

## References

- [DesiredStateAnnotationsProcessor.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java) — processor (qualifier + @DependsOn + fault policy merge)
- [AnnotationValidationStep.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java) — validator (single-scan restructure)
- [DesiredStateQualifier.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DesiredStateQualifier.java) — qualifier annotation
- [DependsOn.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DependsOn.java) — `value` + `nodes` attributes
- [#105 spec](/Users/mdproctor/claude/casehub/desiredstate/docs/specs/issue-105-class-based-desirednode/2026-08-22-class-based-desirednode-design.md) — Part 2 (@DependsOn table), Part 5 (qualifier), Part 6 (validation architecture)
- [#102 spec](/Users/mdproctor/claude/casehub/desiredstate/docs/specs/issue-102-desiredstate-annotations/2026-08-20-desiredstate-annotations-design.md) — §2.6 (GoalCompiler<Void> integration)
- [decisions.md](/Users/mdproctor/claude/public/casehub-desiredstate/specs/issue-110-qualifier-cross-validation/decisions.md) — D1 (revised after decision review R1-02, R1-03)
- [Decision review](/Users/mdproctor/reviews/casehub-desiredstate/issue-110-decision-20260823-042920/responses/reviewer-1.md) — non-local coupling analysis
- [Spec review](/Users/mdproctor/reviews/casehub-desiredstate/issue-110-spec-20260823-044439/responses/reviewer-1.md) — validation restructure, fault policy merge gap, @DependsOn(nodes) target validation
- CDI 4.1 spec §2.3.5 — qualifier matching rules
