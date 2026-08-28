# Phase 1: YAML Language Extensions — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #116 — operator-first declaration language
**Issue group:** #116, #114 (PatternEvaluator extraction)

**Goal:** Deliver Phase 1 of the YAML language extensions — fault policy
declarations, graph invariants, and conditional inclusion (when:) — making
the three features available to YAML operators without Java.

**Architecture:** Each feature adds YAML model types, parser validation in
`YamlDesiredStateProcessor`, and runtime evaluation in `YamlGraphRecorder`.
The invariant feature requires engine infrastructure work: extracting a
shared `PatternEvaluator` from the duplicate code in `GraphRuleEngine` and
`GraphInvariantEngine`, and refactoring their resolved types from records
to sealed interfaces. A foundation batch first migrates `VariableResolver`
from bare `${name}` to prefixed `${var.name}` references.

**Tech Stack:** Java 21, Quarkus 3.x, Jackson YAML, Jandex, Maven

**Design spec:** `specs/issue-116-yaml-language-design/2026-08-27-yaml-language-extensions-design.md`

## Global Constraints

- All `${}` references use explicit prefixes: `${var.}`, `${match.}`, `${fault.}`, `${each.}`
- User-declared node IDs must not contain `.` (reserved separator)
- `NodeSpecRegistry` types resolved via `@NodeTypeId` Jandex scan
- Jackson `ObjectMapper` for spec deserialization uses dedicated coercion-enabled instance
- Build-time validation produces errors, not warnings, for safety violations
- All YAML model records use compact constructors defaulting nulls to empty collections
- Test scope: unit tests for each component, integration test via pipeline-yaml example
- D15 (Drools vs custom engine) is unresolved — this plan uses the existing custom engine; if Drools is chosen later, `DeclarativeInvariantAdapter` becomes a Drools adapter instead

---

## Batch 1: Foundation — VariableResolver Prefix Migration

Safe wrap point: after this batch, all existing tests pass with the new
`${var.}` prefix. The pipeline-yaml example uses the new syntax. No new
features yet.

### Task 1: VariableResolver prefix dispatch

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/resolver/VariableResolver.java`
- Modify: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/resolver/VariableResolverTest.java`
- Modify: `examples/pipeline-yaml/src/main/resources/META-INF/desiredstate/medallion-pipeline.yaml`
- Modify: `examples/pipeline-yaml/src/test/java/io/casehub/desiredstate/example/pipeline/yaml/PipelineYamlTest.java`

**Interfaces:**
- Consumes: `Config` (MicroProfile), `Map<String, String>` inline variables
- Produces: `VariableResolver` with prefix-dispatched `resolveString()` — strips
  `var.` prefix before lookup. Rejects bare `${name}` with `UnresolvedVariableException`
  guidance: `"Use ${var.name} instead of ${name}"`. Rejects `${match.*}` and `${fault.*}`
  at compile time with: `"${match.*} references are resolved at rule evaluation time, not
  during variable resolution"`.

- [ ] **Step 1: Write failing test for prefix dispatch**

Add to `VariableResolverTest`:

```java
@Test
void resolveString_withVarPrefix_resolvesFromInlineVariables() {
    var resolver = new VariableResolver(Map.of("source_uri", "s3://data/test.csv"), null, null);
    String result = resolver.resolveString("${var.source_uri}", "test-node");
    assertEquals("s3://data/test.csv", result);
}

@Test
void resolveString_bareName_throwsWithGuidance() {
    var resolver = new VariableResolver(Map.of("source_uri", "s3://data/test.csv"), null, null);
    var ex = assertThrows(UnresolvedVariableException.class,
            () -> resolver.resolveString("${source_uri}", "test-node"));
    assertTrue(ex.getMessage().contains("${var.source_uri}"));
}

@Test
void resolveString_matchPrefix_throwsWithGuidance() {
    var resolver = new VariableResolver(Map.of(), null, null);
    var ex = assertThrows(UnresolvedVariableException.class,
            () -> resolver.resolveString("${match.sink.id}", "test-node"));
    assertTrue(ex.getMessage().contains("rule evaluation time"));
}

@Test
void resolveString_faultPrefix_throwsWithGuidance() {
    var resolver = new VariableResolver(Map.of(), null, null);
    var ex = assertThrows(UnresolvedVariableException.class,
            () -> resolver.resolveString("${fault.nodeId}", "test-node"));
    assertTrue(ex.getMessage().contains("fault"));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=VariableResolverTest`
Expected: 4 failures — prefix dispatch not implemented yet.

- [ ] **Step 3: Implement prefix dispatch in VariableResolver**

Modify `lookupVariable` to handle prefixes:

```java
private String lookupVariable(String key, String nodeContext) {
    if (key.startsWith("var.")) {
        String varName = key.substring(4);
        return lookupVarPrefixed(varName, nodeContext);
    }
    if (key.startsWith("match.")) {
        throw new UnresolvedVariableException(key, nodeContext,
                "${match.*} references are resolved at rule evaluation time, "
                + "not during variable resolution. This placeholder will be "
                + "resolved by the DeclarativeRuleAdapter.");
    }
    if (key.startsWith("fault.")) {
        throw new UnresolvedVariableException(key, nodeContext,
                "${fault.*} references are resolved at fault time, "
                + "not during variable resolution. This placeholder will be "
                + "resolved by the fault policy template factory.");
    }
    if (key.startsWith("each.")) {
        throw new UnresolvedVariableException(key, nodeContext,
                "${each.*} references are resolved during forEach expansion, "
                + "not during variable resolution.");
    }
    // Bare name — no prefix
    throw new UnresolvedVariableException(key, nodeContext,
            "Bare variable references are no longer supported. "
            + "Use ${var." + key + "} instead of ${" + key + "}.");
}

private String lookupVarPrefixed(String varName, String nodeContext) {
    String value = inlineVariables.get(varName);
    if (value != null) return value;

    if (config != null) {
        Optional<String> configValue = config.getOptionalValue(varName, String.class);
        if (configValue.isPresent()) return configValue.get();
    }

    throw new UnresolvedVariableException("var." + varName, nodeContext,
            "Not found in: inline variables " + inlineVariables.keySet()
            + ", MicroProfile Config.");
}
```

- [ ] **Step 4: Update existing VariableResolverTest tests for ${var.} prefix**

All existing tests that use `${source_uri}` must change to `${var.source_uri}`.
Update every test method that constructs template strings.

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=VariableResolverTest`
Expected: ALL PASS

- [ ] **Step 6: Migrate pipeline-yaml example**

Update `medallion-pipeline.yaml`: change `${source_uri}` to `${var.source_uri}`
and `${batch_size}` to `${var.batch_size}`.

Update `PipelineYamlTest.java` if it references variable syntax in assertions.

- [ ] **Step 7: Run full pipeline-yaml integration test**

Run: `mvn --batch-mode test -pl examples/pipeline-yaml`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```
feat(#116): migrate VariableResolver to prefix-dispatched ${var.} references

Breaking change: bare ${name} references no longer supported.
Use ${var.name} instead. Rejects ${match.*}, ${fault.*}, ${each.*}
at compile time with guidance messages.

Refs #116
```

### Task 2: YAML 1.2 Core Schema boolean configuration

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`
  (or wherever the ObjectMapper/YAMLFactory is configured)
- Create: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlBooleanResolutionTest.java`

**Interfaces:**
- Consumes: Jackson `YAMLFactory` configuration
- Produces: YAML parser that treats only `true`/`false` (case variants) as boolean —
  `yes`, `no`, `on`, `off` remain strings.

- [ ] **Step 1: Write failing test**

```java
@Test
void yamlParsing_yesValue_remainsString() throws Exception {
    String yaml = """
            desiredState:
              namespace: test
              name: bool-test
            variables:
              monitoring_enabled: yes
              debug: true
            nodes: {}
            """;
    YamlGraph graph = parseYaml(yaml);
    assertEquals("yes", graph.variables().get("monitoring_enabled"));
    // 'true' is boolean in YAML 1.2 — but stored as String in Map<String,String>
}
```

- [ ] **Step 2: Run test to verify behavior**

Run the test. If SnakeYAML auto-converts `yes` to boolean `true`, the test
fails — confirming the fix is needed. If it passes, YAML 1.2 Core Schema
is already the default (check Jackson/SnakeYAML version).

- [ ] **Step 3: Configure YAMLFactory if needed**

If the test fails, configure the `YAMLFactory` to use YAML 1.2 Core Schema
boolean resolution. The exact configuration depends on the Jackson YAML
version — may require a custom SnakeYAML `Resolver` or Jackson's
`YAMLGenerator.Feature` settings.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlBooleanResolutionTest`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#116): configure YAML 1.2 Core Schema boolean resolution

Only true/false are boolean literals. yes/no/on/off remain strings.
Prevents automatic boolean coercion from corrupting variable values.

Refs #116
```

---

## Batch 2: Fault Policy Declarations in YAML

Safe wrap point: after this batch, YAML operators can declare fault policies
with template-based review node specs. The pipeline-yaml example demonstrates
the feature.

### Task 3: YAML fault policy model types + parser + validation

**Files:**
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlFaultPolicy.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlFaultTier.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlReviewNode.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlGraph.java`
- Modify: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java`
- Create: `yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/YamlFaultPolicyValidationTest.java`

**Interfaces:**
- Consumes: `YamlGraph` (extended with `faultPolicy` list), `NodeSpecRegistry` type map
- Produces: Validated `List<YamlFaultPolicy>` passed to `YamlGraphRecorder`

- [ ] **Step 1: Write YAML model records**

```java
// YamlFaultPolicy.java
public record YamlFaultPolicy(
        List<String> faultTypes,
        List<String> nodeTypes,
        List<String> ignoreTypes,
        String namespace,
        List<YamlFaultTier> tiers) {
    public YamlFaultPolicy {
        faultTypes = faultTypes != null ? faultTypes : List.of();
        nodeTypes = nodeTypes != null ? nodeTypes : List.of();
        ignoreTypes = ignoreTypes != null ? ignoreTypes : List.of();
        tiers = tiers != null ? tiers : List.of();
    }
}

// YamlFaultTier.java
public record YamlFaultTier(int threshold, YamlReviewNode reviewNode) {}

// YamlReviewNode.java
public record YamlReviewNode(
        String type,
        Map<String, Object> spec,
        HumanGating humanGating) {
    public YamlReviewNode {
        spec = spec != null ? spec : Map.of();
        humanGating = humanGating != null ? humanGating : HumanGating.NONE;
    }
}
```

- [ ] **Step 2: Add `faultPolicy` field to `YamlGraph`**

Add `List<YamlFaultPolicy> faultPolicy` to the `YamlGraph` record. Default
to `List.of()` in compact constructor.

- [ ] **Step 3: Write validation test**

```java
@Test
void validateFaultPolicy_unknownTierType_throwsBuildException() {
    YamlFaultPolicy policy = new YamlFaultPolicy(
            List.of("PROVISION_FAILED"),
            List.of("transformer"),
            List.of(),
            "test",
            List.of(new YamlFaultTier(3,
                    new YamlReviewNode("nonexistent-type", Map.of(), HumanGating.NONE))));
    // Validation should throw because "nonexistent-type" is not in the registry
    assertThrows(/* build exception */);
}

@Test
void validateFaultPolicy_emptyFaultTypes_throwsBuildException() {
    YamlFaultPolicy policy = new YamlFaultPolicy(
            List.of(), // empty — invalid
            List.of("transformer"),
            List.of(),
            "test",
            List.of(new YamlFaultTier(3,
                    new YamlReviewNode("ai-review", Map.of(), HumanGating.NONE))));
    assertThrows(/* build exception */);
}
```

- [ ] **Step 4: Implement validation in `YamlDesiredStateProcessor`**

Add `validateFaultPolicies(List<YamlFaultPolicy>, Map<String, String> typeRegistry)`
method. Validate:
- `faultTypes` is non-empty
- At least one tier
- Tier thresholds are ascending and >= 1
- Each tier's `reviewNode.type` exists in the type registry
- `namespace` is non-empty (or auto-derive from faultTypes)

- [ ] **Step 5: Run tests**

Run: `mvn --batch-mode test -pl yaml/deployment`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```
feat(#116): YAML fault policy model types and build-time validation

Refs #116
```

### Task 4: Template ReviewSpecFactory + FaultCountStore CDI injection

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`
- Create: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlFaultPolicyRecorderTest.java`
- Modify: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java`

**Interfaces:**
- Consumes: `List<YamlFaultPolicy>`, `NodeSpecRegistry`, `FaultCountStore` (CDI), `ObjectMapper`
- Produces: `ThresholdFaultPolicy` beans registered as `SyntheticBeanBuildItem`

- [ ] **Step 1: Write failing test for template ReviewSpecFactory**

```java
@Test
void templateReviewSpecFactory_resolvesFaultVariables() {
    Map<String, Object> specTemplate = Map.of(
            "target", "${fault.nodeId}",
            "detail", "${fault.detail}");

    // Create a template factory
    ReviewSpecFactory factory = createTemplateFactory(
            "ai-review", specTemplate, registry, objectMapper);

    FaultEvent event = new FaultEvent(
            NodeId.of("my-sink"), FaultType.PROVISION_FAILED, "connection timeout");

    NodeSpec result = factory.create(event, null);
    assertInstanceOf(AiReviewSpec.class, result);
    AiReviewSpec spec = (AiReviewSpec) result;
    assertEquals("my-sink", spec.target());
    assertEquals("connection timeout", spec.detail());
}
```

- [ ] **Step 2: Run test — expected FAIL**

- [ ] **Step 3: Implement template ReviewSpecFactory in YamlGraphRecorder**

Add method `createTemplateReviewSpecFactory`:

```java
private ReviewSpecFactory createTemplateReviewSpecFactory(
        YamlReviewNode reviewNode,
        NodeSpecRegistry registry,
        ObjectMapper coercionMapper) {

    String typeName = reviewNode.type();
    Class<? extends NodeSpec> specClass = registry.resolve(typeName);
    Map<String, Object> specTemplate = reviewNode.spec();

    return (faultEvent, graph) -> {
        Map<String, Object> resolved = resolveFaultTemplate(specTemplate, faultEvent);
        try {
            return coercionMapper.convertValue(resolved, specClass);
        } catch (IllegalArgumentException e) {
            LOG.warn("Fault policy template deserialization failed for type '{}': {}",
                    typeName, e.getMessage());
            return null;
        }
    };
}

private Map<String, Object> resolveFaultTemplate(
        Map<String, Object> template, FaultEvent event) {
    Map<String, Object> resolved = new LinkedHashMap<>();
    for (var entry : template.entrySet()) {
        Object val = entry.getValue();
        if (val instanceof String s) {
            resolved.put(entry.getKey(), resolveFaultString(s, event));
        } else {
            resolved.put(entry.getKey(), val);
        }
    }
    return resolved;
}

private String resolveFaultString(String template, FaultEvent event) {
    return template
            .replace("${fault.nodeId}", event.node().value())
            .replace("${fault.type}", event.type().name())
            .replace("${fault.detail}", event.detail() != null ? event.detail() : "");
}
```

- [ ] **Step 4: Add `buildYamlFaultPolicy` method to YamlGraphRecorder**

```java
public RuntimeValue<ThresholdFaultPolicy> buildYamlFaultPolicy(
        YamlFaultPolicy yamlPolicy,
        Map<String, String> typeRegistryMap,
        RuntimeValue<FaultCountStore> faultCountStore) {

    NodeSpecRegistry registry = NodeSpecRegistry.of(typeRegistryMap);
    ObjectMapper coercionMapper = createCoercionMapper();

    return new RuntimeValue<>(() -> {
        var builder = ThresholdFaultPolicy.builder()
                .faultTypes(yamlPolicy.faultTypes().stream()
                        .map(FaultType::valueOf).collect(Collectors.toSet()))
                .nodeTypes(yamlPolicy.nodeTypes().stream()
                        .map(NodeType::of).collect(Collectors.toSet()))
                .ignoreTypes(yamlPolicy.ignoreTypes().stream()
                        .map(NodeType::of).collect(Collectors.toSet()))
                .namespace(yamlPolicy.namespace())
                .faultCountStore(faultCountStore.getValue());

        for (YamlFaultTier tier : yamlPolicy.tiers()) {
            NodeType outputType = NodeType.of(tier.reviewNode().type());
            ReviewSpecFactory factory = createTemplateReviewSpecFactory(
                    tier.reviewNode(), registry, coercionMapper);
            builder.tier(tier.threshold(),
                    FaultPolicy.addReviewNode(factory).withNodeType(outputType));
        }

        return builder.build();
    });
}
```

- [ ] **Step 5: Wire into YamlDesiredStateProcessor build step**

For each `YamlFaultPolicy` in the parsed graph, call
`recorder.buildYamlFaultPolicy()` and register the result as a
`SyntheticBeanBuildItem` with `@ApplicationScoped`.

- [ ] **Step 6: Run tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlFaultPolicyRecorderTest`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```
feat(#116): template ReviewSpecFactory for YAML fault policies

Builds ThresholdFaultPolicy from YAML declarations with ${fault.*}
template resolution. CDI FaultCountStore injected via build step.

Refs #116
```

### Task 5: Pipeline-yaml fault policy integration test

**Files:**
- Modify: `examples/pipeline-yaml/src/main/resources/META-INF/desiredstate/medallion-pipeline.yaml`
- Modify: `examples/pipeline-yaml/src/test/java/io/casehub/desiredstate/example/pipeline/yaml/PipelineYamlTest.java`
- Modify: `examples/pipeline-yaml/pom.xml` (if test dependencies needed)

**Interfaces:**
- Consumes: YAML fault policy feature (Task 3-4)
- Produces: Working integration test proving end-to-end fault policy from YAML

- [ ] **Step 1: Add fault policy to medallion-pipeline.yaml**

```yaml
faultPolicy:
  - faultTypes: [PROVISION_FAILED]
    nodeTypes: [transformer, sink]
    ignoreTypes: [ai-review, human-review]
    namespace: pipeline-escalation
    tiers:
      - threshold: 3
        reviewNode:
          type: ai-review
          spec:
            target: "${fault.nodeId}"
            detail: "${fault.detail}"
      - threshold: 5
        reviewNode:
          type: human-review
          humanGating: ALL
          spec:
            target: "${fault.nodeId}"
            detail: "${fault.detail}"
            instruction: "Requires manual review"
```

- [ ] **Step 2: Write integration test**

```java
@Test
void yamlFaultPolicy_registersThresholdFaultPolicy() {
    // Verify a ThresholdFaultPolicy bean is registered
    ThresholdFaultPolicy policy = Arc.container()
            .instance(ThresholdFaultPolicy.class).get();
    assertNotNull(policy);

    // Verify it responds to PROVISION_FAILED on a transformer node
    FaultEvent event = new FaultEvent(
            NodeId.of("aggregate-tx"),
            FaultType.PROVISION_FAILED,
            "connection timeout");
    // First 2 faults: below threshold
    // 3rd fault: triggers tier 1 (ai-review)
}
```

- [ ] **Step 3: Run integration test**

Run: `mvn --batch-mode test -pl examples/pipeline-yaml`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```
feat(#116): pipeline-yaml example with YAML fault policy declarations

Demonstrates two-tier escalation (ai-review at 3, human-review at 5)
declared entirely in YAML with ${fault.*} template resolution.

Refs #116
```

---

## Batch 3: Invariant Engine Infrastructure

Safe wrap point: after this batch, `PatternEvaluator` is extracted and
shared between both engines. The sealed interface hierarchy is in place.
All existing annotation-based tests pass unchanged.

### Task 6: PatternEvaluator extraction

**Files:**
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/PatternEvaluator.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngine.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphInvariantEngine.java`
- Create: `annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/PatternEvaluatorTest.java`

**Interfaces:**
- Consumes: `DesiredStateGraph`, `List<PatternParameterDescriptor>`, `String[] bindingNames`
- Produces: `List<Map<String, DesiredNode>>` — all valid binding maps for the pattern chain.
  Callers (rule engine, invariant engine) apply their own terminal action to each binding map.

- [ ] **Step 1: Write PatternEvaluator test**

```java
@Test
void evaluate_matchSingleType_returnsAllMatchingNodes() {
    DesiredStateGraph graph = buildGraphWith(
            node("sink-1", "sink"), node("sink-2", "sink"), node("db-1", "db"));
    List<PatternParameterDescriptor> patterns = List.of(
            new PatternParameterDescriptor(PatternKind.MATCH, "sink", "", Direction.DEPENDENCIES));
    String[] bindingNames = {"sink"};

    List<Map<String, DesiredNode>> bindings =
            PatternEvaluator.evaluate(graph, patterns, bindingNames);

    assertEquals(2, bindings.size());
}

@Test
void evaluate_matchWithNotExists_filtersCorrectly() {
    DesiredStateGraph graph = buildGraphWith(
            node("sink-1", "sink"), node("monitor-1", "monitor"),
            dep("monitor-1", "sink-1"));
    List<PatternParameterDescriptor> patterns = List.of(
            new PatternParameterDescriptor(PatternKind.MATCH, "sink", "", Direction.DEPENDENCIES),
            new PatternParameterDescriptor(PatternKind.NOT_EXISTS, "monitor", "sink", Direction.DEPENDENTS));
    String[] bindingNames = {"sink", "guard"};

    List<Map<String, DesiredNode>> bindings =
            PatternEvaluator.evaluate(graph, patterns, bindingNames);

    assertEquals(0, bindings.size()); // sink-1 has a monitor dependent — filtered out
}
```

- [ ] **Step 2: Run test — expected FAIL**

- [ ] **Step 3: Extract PatternEvaluator from GraphInvariantEngine.expandChain**

Create `PatternEvaluator` with a single static method:

```java
public class PatternEvaluator {
    public static List<Map<String, DesiredNode>> evaluate(
            DesiredStateGraph graph,
            List<PatternParameterDescriptor> patterns,
            String[] bindingNames) {
        // Extract cross-product from @Match patterns
        // Then expandChain for remaining patterns
        // Return all valid binding maps
    }
}
```

The implementation is the existing `expandChain` logic from both engines,
extracted into a shared utility. `PatternMatchingSupport` methods are
called unchanged.

- [ ] **Step 3b: Add wildcard type matching to PatternMatchingSupport**

Modify `findDirectNeighbors`, `findReachable`, `existsRelational`, and
`existsGlobal` to skip the type filter when `"*".equals(p.nodeType())`:

```java
// In findDirectNeighbors:
if (!"*".equals(p.nodeType()) && !node.type().equals(NodeType.of(p.nodeType()))) {
    continue;
}
```

Add test:
```java
@Test
void findDirectNeighbors_wildcardType_matchesAllTypes() {
    // Graph: A(sink) -> B(transformer), A -> C(monitor)
    // Pattern: directDep of A with type "*"
    // Expected: returns both B and C
}
```

- [ ] **Step 4: Refactor GraphInvariantEngine to use PatternEvaluator**

Replace `expandChain` call in `validateParameterized` with:
```java
List<Map<String, DesiredNode>> bindings =
        PatternEvaluator.evaluate(graph, invariant.patterns(), paramNames);
if (bindings.isEmpty()) {
    // violation — pattern not satisfiable for this anchor tuple
}
for (Map<String, DesiredNode> binding : bindings) {
    // invoke invariant method with binding values
}
```

- [ ] **Step 5: Refactor GraphRuleEngine to use PatternEvaluator**

Same pattern — replace `expandChain`/`expandBindings` with `PatternEvaluator.evaluate()`.

- [ ] **Step 6: Run ALL existing tests**

Run: `mvn --batch-mode test -pl annotations/runtime`
Expected: ALL PASS — behavior unchanged, only extraction.

Run: `mvn --batch-mode test -pl examples/pipeline-annotated`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```
feat(#114): extract PatternEvaluator from GraphRuleEngine and GraphInvariantEngine

Shared pattern evaluation for both engines. Eliminates ~80 lines of
duplicated expandChain logic. Foundation for declarative YAML rules
and invariants.

Refs #114, #116
```

### Task 7: Sealed interface refactoring for resolved types

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/ResolvedGraphInvariant.java`
  → refactor from record to sealed interface
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/ResolvedInvariant.java`
  (sealed interface with three variants)
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/ResolvedGraphRule.java`
  → refactor from record to sealed interface
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/ResolvedRule.java`
  (sealed interface with three variants)
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphInvariantEngine.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngine.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java`
- Modify: all test files that reference the old record types

**Interfaces:**
- Consumes: existing `ResolvedGraphRule` and `ResolvedGraphInvariant` records
- Produces: `ResolvedRule` sealed interface (`ImperativeRule` | `ParameterizedReflectiveRule` | `DeclarativeRule`),
  `ResolvedInvariant` sealed interface (same three variants).
  Each variant carries a `String name()` and `List<PatternParameterDescriptor> patterns()` accessor.

- [ ] **Step 1: Create `ResolvedInvariant` sealed interface**

```java
public sealed interface ResolvedInvariant {
    String name();
    List<PatternParameterDescriptor> patterns();
    String[] bindingNames();

    record ImperativeInvariant(String name, Method method, Object instance)
            implements ResolvedInvariant {
        public List<PatternParameterDescriptor> patterns() { return List.of(); }
        public String[] bindingNames() { return new String[0]; }
    }

    record ParameterizedReflectiveInvariant(String name, Method method,
            Object instance, List<PatternParameterDescriptor> patterns)
            implements ResolvedInvariant {
        public String[] bindingNames() {
            return PatternMatchingSupport.getParameterNames(method);
        }
    }

    record DeclarativeInvariant(String name,
            List<PatternParameterDescriptor> patterns, String[] bindingNames)
            implements ResolvedInvariant {}
}
```

- [ ] **Step 2: Create `ResolvedRule` sealed interface (same pattern)**

Same three-variant structure. `DeclarativeRule` adds `List<ActionDescriptor> actions`.

- [ ] **Step 3: Update GraphInvariantEngine to use `ResolvedInvariant`**

Replace `ResolvedGraphInvariant` with `ResolvedInvariant` in method signatures.
Use `switch` on the sealed interface for dispatch:

```java
public void validate(DesiredStateGraph graph, List<ResolvedInvariant> invariants) {
    for (ResolvedInvariant invariant : invariants) {
        switch (invariant) {
            case ResolvedInvariant.ImperativeInvariant imp ->
                    validateImperative(imp, graph, violations);
            case ResolvedInvariant.ParameterizedReflectiveInvariant param ->
                    validateParameterized(param, graph, violations);
            case ResolvedInvariant.DeclarativeInvariant decl ->
                    validateDeclarative(decl, graph, violations);
        }
    }
}
```

`validateDeclarative` calls `PatternEvaluator.evaluate()` and checks for
empty bindings (violation). No method invocation — the pattern IS the assertion.

- [ ] **Step 4: Update GraphRuleEngine to use `ResolvedRule` (same pattern)**

- [ ] **Step 5: Update DesiredStateGraphRecorder to construct the new variants**

`resolveInvariants` and `resolveRules` now produce `ImperativeInvariant` or
`ParameterizedReflectiveInvariant` based on the `imperative` flag in the
descriptor.

- [ ] **Step 6: Run ALL tests**

Run: `mvn --batch-mode test -pl annotations/runtime`
Run: `mvn --batch-mode test -pl examples/pipeline-annotated`
Run: `mvn --batch-mode test -pl examples/expansion`
Expected: ALL PASS — behavior unchanged.

- [ ] **Step 7: Remove old `ResolvedGraphRule` and `ResolvedGraphInvariant` records**

Use `ide_refactor_safe_delete` after all references are updated.

- [ ] **Step 8: Commit**

```
feat(#116): sealed interface hierarchy for ResolvedRule and ResolvedInvariant

Three variants: Imperative, ParameterizedReflective, Declarative.
Engines dispatch via sealed switch. Foundation for YAML declarative
rules and invariants.

Refs #116, #114
```

---

## Batch 4: Declarative Invariants in YAML

Safe wrap point: after this batch, YAML operators can declare structural
invariants. The pipeline-yaml example demonstrates the feature.

### Task 8: YAML invariant model types + parser + validation

**Files:**
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlInvariant.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlPattern.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlGraph.java`
- Modify: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java`
- Create: `yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/YamlInvariantValidationTest.java`

**Interfaces:**
- Consumes: `YamlGraph` (extended with `invariants` map)
- Produces: Validated `Map<String, YamlInvariant>` converted to
  `List<DeclarativeInvariantDescriptor>` for the recorder

- [ ] **Step 1: Write YAML model records**

```java
// YamlPattern.java
public record YamlPattern(String type, String of, Direction direction) {
    public YamlPattern {
        direction = direction != null ? direction : Direction.DEPENDENCIES;
    }
}

// YamlInvariant.java
public record YamlInvariant(
        List<String> graph,
        Map<String, YamlPattern> match,
        Map<String, YamlPattern> directDep,
        Map<String, YamlPattern> reaches,
        Map<String, YamlPattern> notExists,
        String message) {
    public YamlInvariant {
        graph = graph != null ? graph : List.of();
        match = match != null ? match : Map.of();
        directDep = directDep != null ? directDep : Map.of();
        reaches = reaches != null ? reaches : Map.of();
        notExists = notExists != null ? notExists : Map.of();
    }
}
```

- [ ] **Step 2: Add `invariants` field to `YamlGraph`**

Add `Map<String, YamlInvariant> invariants` to `YamlGraph`. Default to `Map.of()`.

- [ ] **Step 3: Write validation tests**

Test: pattern `of` references must name a previously declared pattern binding.
Test: `match` must have at least one entry. Test: `type` values validated
against `NodeSpecRegistry` (except `"*"` wildcard).

- [ ] **Step 4: Implement validation + conversion to descriptors**

Add `validateInvariants` and `toPatternDescriptors` methods in
`YamlDesiredStateProcessor`. Convert `YamlPattern` entries to
`PatternParameterDescriptor` instances with the correct `PatternKind`.

When `graph:` is omitted on an invariant, default to the enclosing YAML
graph's scope (`source:<fileName>` — same convention as annotation
in-class invariants scoped to their enclosing `@DesiredState` graph).
Populate `graphPatterns` with `[namespace:name]` from the parsed
`YamlDesiredState` header.

- [ ] **Step 5: Run tests**

Run: `mvn --batch-mode test -pl yaml/deployment`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```
feat(#116): YAML invariant model types and build-time validation

Refs #116
```

### Task 9: DeclarativeInvariantAdapter + GoalCompiler wrapping

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`
- Create: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlInvariantRecorderTest.java`
- Modify: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java`

**Interfaces:**
- Consumes: `List<DeclarativeInvariantDescriptor>` from deployment processor
- Produces: `GoalCompiler` wrapped with `GraphInvariantEngine.validate()` call
  after graph construction

- [ ] **Step 1: Write failing test**

```java
@Test
void yamlGoalCompiler_withInvariant_throwsOnViolation() {
    // Graph with a sink but no upstream transformer
    // Invariant: every sink must have a transformer dependency
    // Expected: GraphInvariantViolationsException
}

@Test
void yamlGoalCompiler_withInvariant_passesWhenSatisfied() {
    // Graph with a sink that has a transformer dependency
    // Expected: compilation succeeds
}
```

- [ ] **Step 2: Run tests — expected FAIL**

- [ ] **Step 3: Implement GoalCompiler invariant wrapping**

In `YamlGraphRecorder.createYamlGoalCompiler()`, after building the graph,
apply invariant validation:

```java
// After building the DesiredStateGraph...
if (!declarativeInvariants.isEmpty()) {
    List<ResolvedInvariant> resolved = declarativeInvariants.stream()
            .map(d -> new ResolvedInvariant.DeclarativeInvariant(
                    d.name(), d.patterns(), d.bindingNames(),
                    d.messageTemplate()))
            .toList();
    new GraphInvariantEngine().validate(graph, resolved);
}
```

- [ ] **Step 3b: Implement custom message template resolution**

In `DeclarativeInvariant`, when a `messageTemplate` is present, resolve
`${match.*}` references against the anchor bindings that triggered the
violation:

```java
// In GraphInvariantEngine.validateDeclarative:
String message = decl.messageTemplate() != null
        ? resolveMatchTemplate(decl.messageTemplate(), anchorBindings)
        : decl.name() + " violated for " + anchorBindings.keySet();
violations.add(new GraphViolation(decl.name(), "yaml:" + fileName,
        message, affectedNodes));
```

Add test:
```java
@Test
void declarativeInvariant_customMessage_resolvesMatchBindings() {
    // Invariant with message: "Sink ${match.sink.id} has no upstream transformer"
    // Expected: violation message = "Sink warehouse-sink has no upstream transformer"
}
```

- [ ] **Step 4: Wire in deployment processor**

Pass the `List<DeclarativeInvariantDescriptor>` from the deployment processor
to the recorder via the `createYamlGoalCompiler` method signature.

- [ ] **Step 5: Run tests**

Run: `mvn --batch-mode test -pl yaml/runtime`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```
feat(#116): declarative invariant evaluation in YAML GoalCompiler

YAML invariants use the same GraphInvariantEngine as annotations.
DeclarativeInvariant variant evaluates patterns via PatternEvaluator
and reports violations for unsatisfiable structural assertions.

Refs #116
```

### Task 10: Pipeline-yaml invariant integration test

**Files:**
- Modify: `examples/pipeline-yaml/src/main/resources/META-INF/desiredstate/medallion-pipeline.yaml`
- Modify: `examples/pipeline-yaml/src/test/java/io/casehub/desiredstate/example/pipeline/yaml/PipelineYamlTest.java`

**Interfaces:**
- Consumes: YAML invariant feature (Tasks 8-9)
- Produces: Working integration test proving invariant evaluation from YAML

- [ ] **Step 1: Add invariant to medallion-pipeline.yaml**

```yaml
invariants:
  every-sink-has-upstream:
    match:
      sink: { type: sink }
    directDep:
      upstream: { type: transformer, of: sink, direction: dependencies }
```

- [ ] **Step 2: Write integration test**

```java
@Test
void yamlInvariant_everySinkHasUpstream_passesForMedallionPipeline() {
    // The medallion pipeline's warehouse-sink depends on aggregate-tx (transformer)
    // This should pass without violation
    GoalCompiler<Void> compiler = /* CDI lookup */;
    CompilationResult result = compiler.compile(null, factory);
    assertNotNull(result);
    // No exception = invariant satisfied
}
```

- [ ] **Step 3: Run integration test**

Run: `mvn --batch-mode test -pl examples/pipeline-yaml`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```
feat(#116): pipeline-yaml example with YAML invariant declaration

Demonstrates structural invariant: every sink must have an upstream
transformer dependency. Evaluated at compile time by GraphInvariantEngine.

Refs #116
```

---

## Batch 5: Conditional Inclusion (when:)

Safe wrap point: after this batch, YAML operators can use `when:` on nodes
for conditional inclusion. The pipeline-yaml example demonstrates the feature.
Phase 1 is complete.

### Task 11: when: field + dependency safety validation

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlNode.java`
- Modify: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java`
- Create: `yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/YamlConditionalValidationTest.java`

**Interfaces:**
- Consumes: `YamlNode` with `when` field, `dependsOn` entries
- Produces: Build-time error when unconditional node depends on conditional node.
  `dependsOn` extended to support optional form: `{ node: "id", optional: true }`.

- [ ] **Step 1: Add `when` field to `YamlNode`**

```java
public record YamlNode(
        String type,
        Map<String, Object> spec,
        List<Object> dependsOn,  // String or Map with {node, optional}
        HumanGating humanGating,
        String when) {
    // compact constructor defaults...
}
```

Note: `dependsOn` type changes from `List<String>` to `List<Object>` to
support both `"nodeId"` and `{ node: "nodeId", optional: true }` forms.

- [ ] **Step 2: Write validation tests**

```java
@Test
void validate_unconditionalDependsOnConditional_throwsBuildError() {
    // Node A (no when) depends on Node B (when: ${var.x})
    // Expected: build-time error
}

@Test
void validate_unconditionalDependsOnConditional_optional_passes() {
    // Node A depends on Node B (when: ${var.x}) with optional: true
    // Expected: passes validation
}

@Test
void validate_conditionalDependsOnConditional_sameCondition_passes() {
    // Both nodes have the same when: condition
    // Expected: passes validation
}
```

- [ ] **Step 3: Implement validation**

In `YamlDesiredStateProcessor`, after parsing all nodes:
- Build a set of conditional node IDs (nodes with `when:` set)
- For each node's `dependsOn` entries:
  - If the dependency target is conditional AND the current node is NOT conditional
    AND the dependency is NOT optional → build error

- [ ] **Step 4: Run tests**

Run: `mvn --batch-mode test -pl yaml/deployment`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
feat(#116): when: field on YAML nodes with dependency safety validation

Build-time error when unconditional node depends on conditional node.
Optional dependency syntax: { node: "id", optional: true }.

Refs #116
```

### Task 12: Compile-time when: evaluation in GoalCompiler

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`
- Create: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlConditionalEvaluationTest.java`

**Interfaces:**
- Consumes: `YamlNode.when` values, `VariableResolver`
- Produces: Nodes excluded when `when:` evaluates to falsy. Optional dependencies
  to excluded nodes silently removed. Truthy: `true`, `yes`, `on`, `y`, `1`.
  Falsy: `false`, `no`, `off`, `n`, `0`. Other values → compile-time error.

- [ ] **Step 1: Write failing tests**

```java
@Test
void whenTrue_nodeIncluded() {
    // Node with when: "${var.enabled}", enabled=true
    // Expected: node appears in compiled graph
}

@Test
void whenFalse_nodeExcluded() {
    // Node with when: "${var.enabled}", enabled=false
    // Expected: node absent from compiled graph
}

@Test
void whenFalse_optionalDependencyRemoved() {
    // Node A depends optionally on B (when: false)
    // Expected: A in graph, dependency on B removed
}

@Test
void whenInvalidValue_throwsCompileError() {
    // Node with when: "${var.mode}", mode="production"
    // Expected: compile-time error — not a boolean
}
```

- [ ] **Step 2: Run tests — expected FAIL**

- [ ] **Step 3: Implement when: evaluation in GoalCompiler**

In the `GoalCompiler.compile()` lambda, after variable resolution:

```java
// Resolve when: conditions
Set<String> excludedNodeIds = new HashSet<>();
for (var entry : yamlGraph.nodes().entrySet()) {
    String nodeId = entry.getKey();
    YamlNode yamlNode = entry.getValue();
    if (yamlNode.when() != null) {
        String resolved = variableResolver.resolveString(yamlNode.when(), nodeId);
        if (!isTruthy(resolved)) {
            excludedNodeIds.add(nodeId);
        }
    }
}

// Filter nodes and dependencies
// Remove excluded nodes
// Remove dependencies TO excluded nodes (if optional)
// Error if non-optional dependency to excluded node survives
```

Add `isTruthy` method:
```java
private static boolean isTruthy(String value) {
    return switch (value.toLowerCase()) {
        case "true", "yes", "on", "y", "1" -> true;
        case "false", "no", "off", "n", "0" -> false;
        default -> throw new IllegalArgumentException(
                "when: condition resolved to '" + value
                + "' which is not a boolean value. "
                + "Expected: true/false/yes/no/on/off/y/n/1/0");
    };
}
```

- [ ] **Step 4: Run tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlConditionalEvaluationTest`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
feat(#116): compile-time when: evaluation in YAML GoalCompiler

Conditional nodes excluded when when: evaluates to falsy.
Truthy: true/yes/on/y/1. Falsy: false/no/off/n/0.
Other values are compile-time errors.

Refs #116
```

### Task 13: Pipeline-yaml when: integration test

**Files:**
- Modify: `examples/pipeline-yaml/src/main/resources/META-INF/desiredstate/medallion-pipeline.yaml`
- Modify: `examples/pipeline-yaml/src/test/java/io/casehub/desiredstate/example/pipeline/yaml/PipelineYamlTest.java`

**Interfaces:**
- Consumes: when: feature (Tasks 11-12)
- Produces: Working integration test with conditional nodes

- [ ] **Step 1: Add conditional node to medallion-pipeline.yaml**

```yaml
variables:
  batch_size: "1000"
  source_uri: s3://data/customers.csv
  debug_mode: "false"

nodes:
  # ... existing nodes ...

  debug-logger:
    type: logger
    when: "${var.debug_mode}"
    dependsOn:
      - { node: csv-ingest, optional: true }
    spec:
      level: TRACE
      target: csv-ingest
```

Note: requires a `LoggerSpec` NodeSpec class. If not available, use an
existing type with a `when:` condition for the test.

- [ ] **Step 2: Write integration test**

```java
@Test
void yamlWhen_debugModeFalse_loggerExcluded() {
    GoalCompiler<Void> compiler = /* CDI lookup */;
    CompilationResult result = compiler.compile(null, factory);
    DesiredStateGraph graph = ((CompilationResult.SingleGraph) result).graph();
    // debug-logger should NOT be in the graph (debug_mode = false)
    assertFalse(graph.nodeIds().contains(NodeId.of("debug-logger")));
    // All other nodes should still be present (8 nodes)
    assertEquals(8, graph.nodeIds().size());
}
```

- [ ] **Step 3: Run integration test**

Run: `mvn --batch-mode test -pl examples/pipeline-yaml`
Expected: ALL PASS

- [ ] **Step 4: Run full build to verify nothing is broken**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```
feat(#116): pipeline-yaml example with conditional inclusion (when:)

Demonstrates debug-logger node excluded when debug_mode=false.
Phase 1 of YAML language extensions complete.

Refs #116
```

---

## Summary

| Batch | Tasks | What's working after |
|-------|-------|---------------------|
| 1: Foundation | 1-2 | `${var.}` prefix, YAML 1.2 booleans, pipeline-yaml migrated |
| 2: Fault Policy | 3-5 | YAML fault policies with template `${fault.*}` resolution |
| 3: Engine Infrastructure | 6-7 | `PatternEvaluator` extracted, sealed interface hierarchy |
| 4: Invariants | 8-10 | YAML invariants with structural pattern assertions |
| 5: Conditional | 11-13 | `when:` on nodes with dependency safety |

**Total:** 5 batches, 13 tasks

**What Phase 1 delivers:** An operator can write a single YAML file with
nodes, dependencies, fault policies (with template-based escalation),
structural invariants (continuous enforcement), and conditional nodes
(environment-aware inclusion) — no Java required.

**What Phase 2 adds:** Graph rules (structural rewriting) and lifecycle
phases (build-then-operate).

## References

- `specs/issue-116-yaml-language-design/2026-08-27-yaml-language-extensions-design.md` — design spec (§6.1-§6.3, §4, §7, §8)
- `specs/issue-116-yaml-language-design/decisions.md` — D1-D5, D7, D11, D16
- `yaml/runtime/.../VariableResolver.java` — current bare-name resolver
- `yaml/runtime/.../YamlGraphRecorder.java:29-65` — GoalCompiler factory
- `yaml/deployment/.../YamlDesiredStateProcessor.java` — build step
- `annotations/runtime/.../GraphInvariantEngine.java` — invariant validation
- `annotations/runtime/.../GraphRuleEngine.java` — rule evaluation (refactored in parallel)
- `annotations/runtime/.../PatternMatchingSupport.java` — shared primitives
- `annotations/runtime/.../ResolvedGraphInvariant.java` — current record (replaced)
- `annotations/runtime/.../ResolvedGraphRule.java` — current record (replaced)
- `annotations/runtime/.../DesiredStateGraphRecorder.java:33-118` — GoalCompiler wrapping pattern
- `api/.../ThresholdFaultPolicy.java` — builder API for fault policies
- `examples/pipeline-yaml/.../medallion-pipeline.yaml` — integration test target
- #116 — operator-first declaration language
- #114 — shared pattern matching infrastructure (PatternEvaluator)
- #119 — Drools backend (deferred, D15 unresolved)
