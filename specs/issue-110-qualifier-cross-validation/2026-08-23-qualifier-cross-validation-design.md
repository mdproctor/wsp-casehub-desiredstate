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

## Part 1: GoalCompiler Qualifier Application (#110)

### Current state

`registerGoalCompilerBean()` registers synthetic beans with no qualifier:

```java
SyntheticBeanBuildItem.configure(GoalCompiler.class)
    .scope(ApplicationScoped.class)
    .unremovable()
    .setRuntimeInit()
    .runtimeValue(runtimeValue)
    .done()
```

CDI implicitly adds `@Default` and `@Any`. Multiple graphs → multiple beans with `@Default` →
`AmbiguousResolutionException`.

### Change

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

### CDI behavior

| Scenario | `@Inject GoalCompiler` | `@Inject @DesiredStateQualifier(...) GoalCompiler` |
|----------|------------------------|---------------------------------------------------|
| 1 graph | Resolves via `@Default` | Resolves via qualifier |
| 2+ graphs | `AmbiguousResolutionException` (lists both beans with qualifier values) | Resolves the specific graph |

Both are build-time errors in Quarkus ArC. `AmbiguousResolutionException` tells the developer
"multiple GoalCompiler beans found" and lists the qualifier values — directly showing how to
fix by adding `@DesiredStateQualifier(namespace = "...", name = "...")`.

### Call site updates

Both call sites in `generateDesiredStateGraphs()` pass namespace and name:

1. Interface-sourced graphs (line 101): pass `descriptor.namespace()`, `descriptor.name()`
2. Class-only graphs (line 133): pass `ns`, `nm` from the graph key

### FaultPolicy beans — no change

FaultPolicy beans remain unqualified. They are additive (all discovered via
`Instance<FaultPolicy>`) and filtered by `nodeTypes` at runtime. Multiple FaultPolicy beans
are expected and correct. No disambiguation needed.

---

## Part 2: @DependsOn(nodes=...) on Interface Methods (Bug Fix)

### Current bug

The processor and validator assume `@DependsOn` always has a string `value()`:

```java
// Processor line 307 — NPE when only nodes is specified
for (String dep : dependsOnAnn.value().asStringArray()) { ... }

// Validator line 178 — same NPE
for (String dep : dependsOnAnn.value().asStringArray()) { ... }
```

When `@DependsOn(nodes = SomeClass.class)` is used on an interface `@Node` method (without
`value`), `dependsOnAnn.value()` returns null → NPE. The `nodes` class ref resolution is
implemented for `@DeclareNode` classes only (processor `resolveClassDependencies()`), not for
interface `@Node` methods.

### Fix — Processor

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

### Fix — Validator

In `validateNodeMethod()`, null-safe access to `value()` and add `nodes` to adjacency:

```java
AnnotationInstance dependsOnAnn = method.annotation(DEPENDS_ON);
if (dependsOnAnn != null) {
    List<String> deps = new ArrayList<>();
    AnnotationValue stringDeps = dependsOnAnn.value();
    if (stringDeps != null) {
        for (String dep : stringDeps.asStringArray()) {
            deps.add(dep);
        }
    }
    // nodes attribute — resolve class refs to IDs for adjacency
    AnnotationValue classDeps = dependsOnAnn.value("nodes");
    if (classDeps != null) {
        for (var classRef : classDeps.asClassArray()) {
            ClassInfo targetClass = index.getClassByName(classRef.name());
            if (targetClass != null) {
                AnnotationInstance targetAnn = targetClass.declaredAnnotation(DECLARE_NODE);
                if (targetAnn != null) {
                    deps.add(targetAnn.value("id").asString());
                }
            }
        }
    }
    adjacency.put(nodeId, deps);
}
```

Same null-safe pattern in `validateDependsOnRefs()`.

---

## Part 3: Cross-Model Validation Phase 2 (#111)

### Current state

Validation runs in two independent loops:
1. Per `@DesiredState` interface: `validateNodeMethod()`, `validateDependsOnRefs()`,
   `detectCycles()` — scoped to one interface's `nodeIds`
2. Per `@DeclareNode` class: `validateDeclareNodes()` — per-class constraints only

No merged view exists. Cross-model issues (duplicate IDs, dangling refs, cycles spanning
both models) pass validation silently.

### Restructuring

Add a Phase 2 after the existing per-model validation. Phase 1 (existing) catches per-model
errors first — Phase 2 only runs if Phase 1 produces no errors.

```java
void validate(CombinedIndexBuildItem indexBuildItem) {
    // Phase 1 — per-model validation (existing, unchanged)
    for (AnnotationInstance dsAnn : index.getAnnotations(DESIRED_STATE)) { ... }
    validateDeclareNodes(index, errors);

    if (!errors.isEmpty()) {
        // Phase 1 errors — skip Phase 2 (merged set may be inconsistent)
        throw ...;
    }

    // Phase 2 — cross-model validation
    validateCrossModel(index, errors, warnings);

    // Report
    for (String warning : warnings) { LOG.warn(warning); }
    if (!errors.isEmpty()) { throw ...; }
}
```

### Phase 2: validateCrossModel()

Build a merged node set per graph key, then validate:

```java
private void validateCrossModel(IndexView index, List<String> errors, List<String> warnings) {
    // 1. Collect all nodes by graph key
    Map<String, MergedGraph> graphsByKey = new LinkedHashMap<>();

    // From @DesiredState interfaces
    for (AnnotationInstance dsAnn : index.getAnnotations(DESIRED_STATE)) {
        ClassInfo dsClass = dsAnn.target().asClass();
        String ns = stringValueOrDefault(dsAnn, index, "namespace", "");
        String nm = stringValueOrDefault(dsAnn, index, "name", "");
        String graphKey = ns + ":" + nm;
        MergedGraph mg = graphsByKey.computeIfAbsent(graphKey, k -> new MergedGraph());

        for (MethodInfo method : dsClass.methods()) {
            AnnotationInstance nodeAnn = method.annotation(NODE);
            if (nodeAnn == null) continue;
            String nodeId = nodeAnn.value().asString();
            mg.addNode(nodeId, "interface method " + dsClass.name().local() + "#" + method.name());
            // collect deps for adjacency
            collectDepsForAdjacency(method, index, nodeId, mg);
        }
    }

    // From @DeclareNode classes
    for (AnnotationInstance dnAnn : index.getAnnotations(DECLARE_NODE)) {
        ClassInfo classInfo = dnAnn.target().asClass();
        String ns = stringValueOrDefault(dnAnn, index, "namespace", "");
        String nm = stringValueOrDefault(dnAnn, index, "name", "");
        String graphKey = ns + ":" + nm;
        MergedGraph mg = graphsByKey.computeIfAbsent(graphKey, k -> new MergedGraph());
        String nodeId = dnAnn.value("id").asString();
        mg.addNode(nodeId, "@DeclareNode class " + classInfo.name().local());
        // collect deps for adjacency
        collectClassDepsForAdjacency(classInfo, index, nodeId, mg);
    }

    // 2. Validate each merged graph
    for (var entry : graphsByKey.entrySet()) {
        MergedGraph mg = entry.getValue();
        mg.validateDuplicateIds(errors);
        mg.validateDependencyRefs(errors);
        mg.detectCycles(errors);
    }
}
```

### MergedGraph (private inner class or record)

```java
private static class MergedGraph {
    final Map<String, String> nodeIdToSource = new LinkedHashMap<>();
    final Map<String, List<String>> adjacency = new HashMap<>();
    final Set<String> allNodeIds = new LinkedHashSet<>();

    void addNode(String nodeId, String source) { ... }
    void addDependency(String fromId, String toId) { ... }
    void validateDuplicateIds(List<String> errors) { ... }
    void validateDependencyRefs(List<String> errors) { ... }
    void detectCycles(List<String> errors) { ... }
}
```

### Cross-model validations

| Check | Error message |
|-------|---------------|
| Duplicate node ID across models | `Duplicate node id 'lb' in graph 'infra:zones' — declared on @DeclareNode class LoadBalancer and interface method BaseInfra#lbNode` |
| Cross-model string ref unresolved | `@DependsOn on 'lbNode' in graph 'infra:zones' references 'missing-id' which is not declared as @Node or @DeclareNode` |
| Cross-model circular dependency | `Circular dependency detected in graph 'infra:zones': lb → dns → lb` |
| @DependsOn(nodes) target not in Jandex | `@DependsOn(nodes) on 'Vpc' references 'DnsRecord' which has no @DeclareNode annotation (if the class is in an external JAR, ensure a Jandex index is generated)` — **warning**, not error |

### Validation of @DependsOn(nodes) target class

This validation runs in Phase 1 (per-class) since it doesn't require the merged set:

```java
// In validateDeclareNodes() — for @DependsOn(nodes) on @DeclareNode classes
AnnotationValue classDeps = dependsOnAnn.value("nodes");
if (classDeps != null) {
    for (var classRef : classDeps.asClassArray()) {
        ClassInfo targetClass = index.getClassByName(classRef.name());
        if (targetClass == null) {
            warnings.add("@DependsOn(nodes) on '" + className
                + "' references '" + classRef.name().local()
                + "' which is not in the Jandex index"
                + " (if the class is in an external JAR, ensure a Jandex index is generated)");
        } else if (targetClass.declaredAnnotation(DECLARE_NODE) == null) {
            errors.add("@DependsOn(nodes) on '" + className
                + "' references '" + classRef.name().local()
                + "' which has no @DeclareNode annotation");
        } else if (!implementsNodeSpec(classRef.name(), index)) {
            errors.add("@DependsOn(nodes) references '" + classRef.name().local()
                + "' which does not implement NodeSpec");
        }
    }
}
```

Same validation applies to `@DependsOn(nodes)` on interface `@Node` methods.

---

## Testing Strategy

### Part 1 tests — Qualifier

**QualifierSingleGraphTest:**
- Single `@DesiredState` → GoalCompiler bean has `@DesiredStateQualifier(namespace, name)`
- Unqualified `@Inject GoalCompiler` still resolves
- Qualified `@Inject @DesiredStateQualifier(...) GoalCompiler` also resolves

**QualifierMultiGraphTest:**
- Two `@DesiredState` interfaces with different (namespace, name)
- Each injectable via `@DesiredStateQualifier(...)`
- Verify both compile separate GoalCompiler beans with correct graphs

**QualifierClassOnlyTest:**
- Class-only graph → GoalCompiler bean has `@DesiredStateQualifier` with correct namespace/name

### Part 2 tests — @DependsOn(nodes) on interface methods

**InterfaceNodesDependencyTest:**
- `@Node` method with `@DependsOn(nodes = SomeClass.class)` → dependency resolved
- `@Node` method with `@DependsOn(nodes = ...)` only (no `value`) → no NPE, dependency resolved
- Mixed `@DependsOn(value = "x", nodes = {A.class})` on interface method → both resolved

### Part 3 tests — Cross-model validation

**CrossModelDuplicateIdTest:**
- Same node ID in `@Node` method and `@DeclareNode` class, same graph → build error

**CrossModelStringRefTest:**
- `@DeclareNode` class with `@DependsOn("interface-node-id")` → resolves across models
- `@DeclareNode` class with `@DependsOn("nonexistent")` → build error naming the unresolved ref

**CrossModelCycleTest:**
- Interface node depends on class node, class node depends back on interface node → build error with cycle path

**JandexWarningTest:**
- `@DependsOn(nodes = SomeClass.class)` where `SomeClass` is not in index → warning (not error)
- `@DependsOn(nodes = SomeClass.class)` where `SomeClass` lacks `@DeclareNode` → error

### Existing tests — no regression

All existing tests (`DesiredStateAnnotationsProcessorTest`, `ClassBasedNodeTest`,
`ClassBasedDependencyTest`, `ClassBasedFaultPolicyTest`, `ClassBasedValidationTest`,
`MergedGraphTest`, `FaultPolicyWiringTest`, `GoalMethodCompositionTest`, `ValidationErrorTest`)
must pass without modification.

---

## Migration Impact

| Component | Change |
|-----------|--------|
| `DesiredStateAnnotationsProcessor.registerGoalCompilerBean()` | Add `@Default` + `@DesiredStateQualifier` qualifiers. Accept namespace/name params. |
| `DesiredStateAnnotationsProcessor.buildGraphDescriptor()` | Handle `@DependsOn` `nodes` attribute on interface methods (null-safe `value()` + class ref resolution). |
| `AnnotationValidationStep.validate()` | Add Phase 2 call after Phase 1. |
| `AnnotationValidationStep.validateNodeMethod()` | Null-safe `@DependsOn` `value()` access. Handle `nodes` in adjacency. |
| `AnnotationValidationStep.validateDependsOnRefs()` | Null-safe `value()` access. Handle `nodes` refs. |
| `AnnotationValidationStep` (new) | `validateCrossModel()`, `MergedGraph` inner class, `collectDepsForAdjacency()`. |

No changes to: runtime module, api module, recorder, examples, or any module outside `annotations/`.

---

## References

- [DesiredStateAnnotationsProcessor.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java) — processor (qualifier + @DependsOn changes)
- [AnnotationValidationStep.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java) — validator (cross-model validation)
- [DesiredStateQualifier.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DesiredStateQualifier.java) — qualifier annotation
- [#105 spec](/Users/mdproctor/claude/casehub/desiredstate/docs/specs/issue-105-class-based-desirednode/2026-08-22-class-based-desirednode-design.md) — Part 2 (@DependsOn table), Part 5 (qualifier), Part 6 (validation)
- [#102 spec](/Users/mdproctor/claude/casehub/desiredstate/docs/specs/issue-102-desiredstate-annotations/2026-08-20-desiredstate-annotations-design.md) — §2.6 (GoalCompiler<Void> integration)
- [decisions.md](/Users/mdproctor/claude/public/casehub-desiredstate/specs/issue-110-qualifier-cross-validation/decisions.md) — D1 (revised after decision review R1-02, R1-03)
- [Decision review R1](/Users/mdproctor/reviews/casehub-desiredstate/issue-110-decision-20260823-042920/responses/reviewer-1.md) — non-local coupling analysis
- CDI 4.1 spec §2.3.5 — qualifier matching rules
