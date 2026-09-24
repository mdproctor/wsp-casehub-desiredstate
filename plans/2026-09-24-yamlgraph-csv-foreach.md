# YamlGraph CSV forEach + ObjectVariableSource Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #149 — YamlGraph model change: CSV forEach + ObjectVariableSource typed resolution
**Issue group:** #149

**Goal:** Add CSV-backed forEach data sources and typed variable resolution to the YAML surface, with deployment-time prefix normalization.

**Architecture:** Shared model prerequisite (YamlGraph record + constructor call sites), then three independent items: CSV data wiring, typed variable resolution, prefix normalization. All building blocks exist in yaml-core 0.2-SNAPSHOT (platform#419 + #426).

**Tech Stack:** Java 22, Quarkus (recorder + deployment processor), Jackson YAML, casehub-platform-yaml-core 0.2-SNAPSHOT

## Global Constraints

- yaml-core 0.2-SNAPSHOT must be in local .m2 with platform#419 + #426 changes
- All `new YamlGraph(...)` calls in tests use positional constructor args — new field appends `, null`
- `@Recorder` methods in `YamlGraphRecorder` must use serializable types (primitives, String, Map, List)
- `YamlGraph` compact constructor defaults null fields to empty collections

---

## Batch 1: Model Change + Constructor Call Sites

### Task 1: YamlGraph record — widen variables, add data field

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlGraph.java`
- Test: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/model/YamlGraphDeserializationTest.java`

**Interfaces:**
- Produces: `YamlGraph.variables()` returns `Map<String, Object>`, `YamlGraph.data()` returns `Map<String, Object>`

- [ ] **Step 1: Write failing deserialization test for typed variables**

```java
@Test
void deserialize_typedVariables_preservesTypes() throws Exception {
    String yaml = """
            desiredState:
              namespace: test
              name: typed-vars
            variables:
              batch_size: 500
              enabled: true
              ratio: 3.14
              bucket: prod
            nodes: {}
            """;
    YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);
    assertThat(graph.variables().get("batch_size")).isEqualTo(500);
    assertThat(graph.variables().get("enabled")).isEqualTo(true);
    assertThat(graph.variables().get("ratio")).isEqualTo(3.14);
    assertThat(graph.variables().get("bucket")).isEqualTo("prod");
}
```

- [ ] **Step 2: Write failing deserialization test for data field**

```java
@Test
void deserialize_dataBlock_parsesToMap() throws Exception {
    String yaml = """
            desiredState:
              namespace: test
              name: csv-data
            data:
              regions:
                inline: |
                  name,tier:integer
                  us-east,1
                  eu-west,2
            nodes: {}
            """;
    YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);
    assertThat(graph.data()).containsKey("regions");
    assertThat(graph.data().get("regions")).isInstanceOf(Map.class);
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlGraphDeserializationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation error — `variables` is `Map<String, String>`, `data` field doesn't exist

- [ ] **Step 4: Modify YamlGraph record**

Change `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlGraph.java`:

```java
public record YamlGraph(
        YamlDesiredState desiredState,
        Map<String, Object> variables,
        Map<String, YamlNode> nodes,
        List<YamlFaultPolicy> faultPolicy,
        Map<String, YamlInvariant> invariants,
        Map<String, YamlRule> rules,
        YamlLifecycle lifecycle,
        Map<String, IterationGroup> iterations,
        List<YamlImport> imports,
        Map<String, Object> data) {

    public YamlGraph {
        if (variables == null) {variables = Map.of();}
        if (nodes == null) {nodes = Map.of();}
        if (faultPolicy == null) {faultPolicy = List.of();}
        if (invariants == null) {invariants = Map.of();}
        if (rules == null) {rules = Map.of();}
        if (iterations == null) {iterations = Map.of();}
        if (imports == null) {imports = List.of();}
        if (data == null) {data = Map.of();}
    }
}
```

- [ ] **Step 5: Fix all 12 constructor call sites**

Each `new YamlGraph(...)` call appends `, null` for the `data` parameter:

**`YamlConditionalEvaluationTest.java:123`** — change `buildGraph` method:
```java
private YamlGraph buildGraph(Map<String, String> variables, Map<String, YamlNode> nodes) {
    return new YamlGraph(
            new io.casehub.desiredstate.yaml.model.YamlDesiredState("test", "cond"),
            variables, nodes, List.of(), Map.of(), Map.of(), null, null, null, null);}
```
Note: `variables` parameter type stays `Map<String, String>` here — Java auto-widens to `Map<String, Object>`.

**`YamlLifecycleCompilerTest.java`** — 4 calls (lines 38, 76, 104, 136): append `, null` after the last `null` in each constructor call.

**`YamlLifecycleValidationTest.java`** — 7 calls (lines 24, 40, 57, 69, 84, 101, 113): append `, null` after the last `null` in each constructor call. For line 101 and 105 which end with `null, null, null)`, change to `null, null, null, null)`.

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl yaml/runtime,yaml/deployment`
Expected: All tests pass including new deserialization tests

- [ ] **Step 7: Commit**

```bash
git add yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlGraph.java
git add yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/model/YamlGraphDeserializationTest.java
git add yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlConditionalEvaluationTest.java
git add yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlLifecycleCompilerTest.java
git add yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/YamlLifecycleValidationTest.java
git commit -m "feat(#149): widen YamlGraph.variables to Map<String,Object>, add data field"
```

---

## Batch 2: Typed Variable Resolution + CSV ForEach Wiring

### Task 2: YamlGraphRecorder — ObjectVariableSource + CSV dispatch

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java:37-280`
- Test: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlGraphRecorderTest.java`

**Interfaces:**
- Consumes: `YamlGraph.variables()` → `Map<String, Object>`, `YamlGraph.data()` → `Map<String, Object>`
- Produces: `createYamlGoalCompiler(GraphDescriptor, Map<String,String>, Map<String,Object>, ...)` — widened `inlineVariables` parameter

- [ ] **Step 1: Write failing test for typed variable resolution**

Add to `YamlGraphRecorderTest`:

```java
@Test
void typedVariables_soleReference_preservesIntegerType() {
    Map<String, Object> variables = Map.of("batch_size", 500, "bucket", "prod");
    VariableResolver resolver = new VariableResolver(Map.of(), Set.of("match", "fault"))
            .withObjectScope("var", variables::get);
    Map<String, Object> specValues = Map.of("batchSize", "${var.batch_size}");
    Map<String, Object> resolved = resolver.resolveMap(specValues, "test-node");
    assertThat(resolved.get("batchSize")).isEqualTo(500);
    assertThat(resolved.get("batchSize")).isInstanceOf(Integer.class);
}

@Test
void typedVariables_embeddedReference_resolvesToString() {
    Map<String, Object> variables = Map.of("bucket", "prod");
    VariableResolver resolver = new VariableResolver(Map.of(), Set.of("match", "fault"))
            .withObjectScope("var", variables::get);
    Map<String, Object> specValues = Map.of("uri", "s3://${var.bucket}/data");
    Map<String, Object> resolved = resolver.resolveMap(specValues, "test-node");
    assertThat(resolved.get("uri")).isEqualTo("s3://prod/data");
}
```

- [ ] **Step 2: Run tests to verify they pass** (these test yaml-core directly — should pass with platform#426)

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlGraphRecorderTest#typedVariables*`
Expected: PASS — validates platform#426 is working

- [ ] **Step 3: Write failing test for CSV forEach expansion**

Add to `ForEachExpanderTest`:

```java
@Test
void csvDataSource_expandsWithTypedRowFields() {
    Map<String, YamlNode> nodes = new LinkedHashMap<>();
    nodes.put("ingest", new YamlNode("data-source",
            Map.of("name", "${each.region.name}", "uri", "s3://${each.region.name}/data"),
            List.of(), null, null, "regions", "region", null));

    Map<String, Object> data = Map.of("regions", Map.of("inline",
            "name,tier:integer\nus-east,1\neu-west,2"));
    Map<String, CsvDataSource> dataSources = CsvDataSource.fromDataBlock(data);

    var adapter = new YamlNodeForEachAdapter();
    ExpansionResult<YamlNode> expanded = ForEachExpander.expand(
            nodes, Map.of(), dataSources, resolver, adapter, 1000);

    assertThat(expanded.elements()).hasSize(2);
    assertThat(expanded.elements()).containsKey("ingest.us-east");
    assertThat(expanded.elements()).containsKey("ingest.eu-west");
}
```

- [ ] **Step 4: Run test to verify it passes** (tests yaml-core ForEachExpander directly — should pass)

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=ForEachExpanderTest#csvDataSource*`
Expected: PASS

- [ ] **Step 5: Widen inlineVariables parameter in all YamlGraphRecorder overloads**

Change all 4 `createYamlGoalCompiler` overloads (lines 37-65) and `createYamlLifecycleGoalCompiler` (line 268):

`Map<String, String> inlineVariables` → `Map<String, Object> inlineVariables`

- [ ] **Step 6: Update resolver construction in main createYamlGoalCompiler (line 82-84)**

Replace:
```java
VariableResolver resolver = new VariableResolver(
        Map.of("var", (VariableSource) inlineVariables::get),
        Set.of("match", "fault"));
```

With:
```java
VariableResolver resolver = new VariableResolver(Map.of(), Set.of("match", "fault"))
        .withObjectScope("var", inlineVariables::get);
```

- [ ] **Step 7: Add CSV data source wiring before forEach expansion (after line 107)**

Insert before the `boolean hasForEach` line:
```java
Map<String, io.casehub.yaml.core.data.CsvDataSource> dataSources = Map.of();
if (yamlGraph != null && !yamlGraph.data().isEmpty()) {
    dataSources = io.casehub.yaml.core.data.CsvDataSource.fromDataBlock(yamlGraph.data());
}
```

- [ ] **Step 8: Update ForEachExpander dispatch (lines 113-119)**

Replace the existing expansion call:
```java
io.casehub.yaml.core.foreach.ExpansionResult<io.casehub.desiredstate.yaml.model.YamlNode> expanded =
        io.casehub.yaml.core.foreach.ForEachExpander.expand(
                effectiveNodes,
                yamlGraph != null && yamlGraph.iterations() != null ? yamlGraph.iterations() : Map.of(),
                resolver, adapter, 1000, jsonArrayExpander(mapper));
```

With conditional dispatch:
```java
Map<String, io.casehub.yaml.core.foreach.IterationGroup> iterations =
        yamlGraph != null && yamlGraph.iterations() != null ? yamlGraph.iterations() : Map.of();
io.casehub.yaml.core.foreach.ExpansionResult<io.casehub.desiredstate.yaml.model.YamlNode> expanded;
if (!dataSources.isEmpty()) {
    expanded = io.casehub.yaml.core.foreach.ForEachExpander.expand(
            effectiveNodes, iterations, dataSources, resolver, adapter, 1000);
} else {
    expanded = io.casehub.yaml.core.foreach.ForEachExpander.expand(
            effectiveNodes, iterations, resolver, adapter, 1000, jsonArrayExpander(mapper));
}
```

- [ ] **Step 9: Update resolver construction in createYamlLifecycleGoalCompiler (lines 278-280)**

Same change as step 6:
```java
VariableResolver resolver = new VariableResolver(Map.of(), Set.of("match", "fault"))
        .withObjectScope("var", inlineVariables::get);
```

- [ ] **Step 10: Run full test suite**

Run: `mvn --batch-mode test -pl yaml/runtime`
Expected: All tests pass

- [ ] **Step 11: Commit**

```bash
git add yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java
git add yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlGraphRecorderTest.java
git add yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/ForEachExpanderTest.java
git commit -m "feat(#149): wire ObjectVariableSource + CSV ForEachExpander dispatch"
```

### Task 3: Deployment processor — validation + variable passing

**Files:**
- Modify: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java:52-162,307-343,707-780`
- Test: `yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessorTest.java` (or existing validation tests)

**Interfaces:**
- Consumes: `YamlGraph.variables()` → `Map<String, Object>`, `YamlGraph.data()` → `Map<String, Object>`
- Consumes: `YamlGraphRecorder.createYamlGoalCompiler(...)` with widened `Map<String, Object>` parameter
- Produces: Build-time validation for `data` field, cross-reference with forEach groups

- [ ] **Step 1: Write failing test for data source validation**

Add to deployment test class:

```java
@Test
void validate_forEachReferencesDataSource_passes() {
    Map<String, Object> data = Map.of("regions", Map.of("inline",
            "name,tier:integer\nus-east,1\neu-west,2"));
    Map<String, YamlNode> nodes = Map.of(
            "ingest", new YamlNode("db", Map.of(), List.of(), null, null, "regions", "region", null));
    var graph = new YamlGraph(
            new YamlDesiredState("test", "csv-foreach"),
            Map.of(), nodes, List.of(), Map.of(), Map.of(), null, null, null, data);
    assertThatCode(() -> YamlDesiredStateProcessor.validateForEach(
            graph.nodes(), graph.iterations(), graph.data(), TYPE_REGISTRY, "test.yaml"))
            .doesNotThrowAnyException();
}

@Test
void validate_forEachReferencesUnknownGroupOrDataSource_throws() {
    Map<String, YamlNode> nodes = Map.of(
            "ingest", new YamlNode("db", Map.of(), List.of(), null, null, "missing", "item", null));
    var graph = new YamlGraph(
            new YamlDesiredState("test", "bad-ref"),
            Map.of(), nodes, List.of(), Map.of(), Map.of(), null, null, null, Map.of());
    assertThatThrownBy(() -> YamlDesiredStateProcessor.validateForEach(
            graph.nodes(), graph.iterations(), graph.data(), TYPE_REGISTRY, "test.yaml"))
            .hasMessageContaining("missing");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl yaml/deployment`
Expected: Compilation error — `validateForEach` doesn't accept `data` parameter

- [ ] **Step 3: Update validateForEach signature**

Add `Map<String, Object> data` parameter to `validateForEach` (line 707):

```java
static void validateForEach(Map<String, YamlNode> nodes,
                            Map<String, IterationGroup> iterations,
                            Map<String, Object> data,
                            Map<String, String> typeRegistry, String fileName) {
```

At the start of the method, parse data sources and merge with iteration groups for validation:

```java
Map<String, io.casehub.yaml.core.data.CsvDataSource> dataSources = Map.of();
if (data != null && !data.isEmpty()) {
    dataSources = io.casehub.yaml.core.data.CsvDataSource.fromDataBlock(data);
}
```

Update the group reference check (line 727) to also accept data source names:

```java
} else if (forEach instanceof String groupRef) {
    if (!iterations.containsKey(groupRef) && !dataSources.containsKey(groupRef)) {
        throw new RuntimeException(fileName + ": node '" + nodeId
                + "' references unknown iteration group or data source '" + groupRef
                + "'. Available groups: " + iterations.keySet()
                + ", data sources: " + dataSources.keySet());
    }
```

- [ ] **Step 4: Update validateYamlGraph call site (line 342)**

Pass `graph.data()`:
```java
validateForEach(graph.nodes(), graph.iterations(), graph.data(), typeRegistry, fileName);
```

- [ ] **Step 5: Update variable passing in discoverYamlGraphs (lines 103, 110)**

The type widening is automatic — `yamlGraph.variables()` now returns `Map<String, Object>`, and the recorder accepts `Map<String, Object>`. No code change needed — the existing `yamlGraph.variables() != null ? yamlGraph.variables() : Map.of()` still works.

Verify by reading the call sites.

- [ ] **Step 6: Run full deployment test suite**

Run: `mvn --batch-mode test -pl yaml/deployment`
Expected: All tests pass

- [ ] **Step 7: Commit**

```bash
git add yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java
git add yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/
git commit -m "feat(#149): deployment validation for data sources, widen variable passing"
```

---

## Batch 3: Prefix Normalization + Full Build

### Task 4: VariablePrefixRewriter deployment-time normalization

**Files:**
- Modify: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java`
- Test: `yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/YamlPrefixNormalizationTest.java`

**Interfaces:**
- Consumes: `io.casehub.yaml.core.resolver.VariablePrefixRewriter.rewrite(input, defaultPrefix, knownPrefixes, forEachVars)`
- Produces: Normalized YAML spec values before compilation

- [ ] **Step 1: Write failing test for bare variable normalization**

Create `YamlPrefixNormalizationTest.java`:

```java
package io.casehub.desiredstate.yaml.deployment;

import io.casehub.yaml.core.resolver.VariablePrefixRewriter;
import org.junit.jupiter.api.Test;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;

class YamlPrefixNormalizationTest {

    private static final Set<String> KNOWN = Set.of("var", "each", "match", "fault", "module", "params");

    @Test
    void bareVariable_rewritesToVarPrefix() {
        String result = VariablePrefixRewriter.rewrite(
                "${batch_size}", "var", KNOWN, Set.of());
        assertThat(result).isEqualTo("${var.batch_size}");
    }

    @Test
    void alreadyPrefixed_passesThrough() {
        String result = VariablePrefixRewriter.rewrite(
                "${var.batch_size}", "var", KNOWN, Set.of());
        assertThat(result).isEqualTo("${var.batch_size}");
    }

    @Test
    void forEachVariable_rewritesToEachPrefix() {
        String result = VariablePrefixRewriter.rewrite(
                "${region}", "var", KNOWN, Set.of("region"));
        assertThat(result).isEqualTo("${each.region}");
    }

    @Test
    void mixed_rewritesCorrectly() {
        String result = VariablePrefixRewriter.rewrite(
                "s3://${bucket}/${region}/data", "var", KNOWN, Set.of("region"));
        assertThat(result).isEqualTo("s3://${var.bucket}/${each.region}/data");
    }

    @Test
    void deferredPrefixes_passThrough() {
        String result = VariablePrefixRewriter.rewrite(
                "${match.sink.id}", "var", KNOWN, Set.of());
        assertThat(result).isEqualTo("${match.sink.id}");
    }
}
```

- [ ] **Step 2: Run tests to verify they pass** (these test yaml-core directly)

Run: `mvn --batch-mode test -pl yaml/deployment -Dtest=YamlPrefixNormalizationTest`
Expected: PASS — validates VariablePrefixRewriter works as expected

- [ ] **Step 3: Wire normalization into discoverYamlGraphs**

In `YamlDesiredStateProcessor.discoverYamlGraphs`, after parsing and before validation, add a normalization pass. Add a private method:

```java
private static void normalizeVariableReferences(YamlGraph graph) {
    Set<String> knownPrefixes = Set.of("var", "each", "match", "fault", "module", "params");
    Set<String> forEachVars = new HashSet<>();
    for (Map.Entry<String, YamlNode> entry : graph.nodes().entrySet()) {
        Object forEach = entry.getValue().forEach();
        if (forEach instanceof Map<?, ?> inlineForEach) {
            Object as = inlineForEach.get("as");
            if (as instanceof String asStr) { forEachVars.add(asStr); }
        } else if (forEach instanceof String groupRef) {
            IterationGroup group = graph.iterations().get(groupRef);
            if (group != null && group.as() != null) { forEachVars.add(group.as()); }
        }
    }
    for (Map.Entry<String, YamlNode> entry : graph.nodes().entrySet()) {
        YamlNode node = entry.getValue();
        if (node.spec() != null) {
            Map<String, Object> rewritten = rewriteMap(node.spec(), knownPrefixes, forEachVars);
            if (!rewritten.equals(node.spec())) {
                entry.setValue(node.withSpec(rewritten));
            }
        }
    }
}

private static Map<String, Object> rewriteMap(Map<String, Object> map,
        Set<String> knownPrefixes, Set<String> forEachVars) {
    Map<String, Object> result = new LinkedHashMap<>();
    for (Map.Entry<String, Object> e : map.entrySet()) {
        Object val = e.getValue();
        if (val instanceof String s && s.contains("${")) {
            result.put(e.getKey(),
                    VariablePrefixRewriter.rewrite(s, "var", knownPrefixes, forEachVars));
        } else if (val instanceof Map<?, ?> nested) {
            result.put(e.getKey(), rewriteMap((Map<String, Object>) nested, knownPrefixes, forEachVars));
        } else {
            result.put(e.getKey(), val);
        }
    }
    return result;
}
```

Note: This requires `YamlNode.withSpec(Map<String,Object>)` — check if it exists. If `YamlNode` is a record, use a wither or construct a new instance. The implementer should check `YamlNode`'s structure and use `ide_file_structure` to determine the approach.

Call `normalizeVariableReferences(yamlGraph)` in the `for (NamedYamlGraph named : yamlGraphs)` loop, after `YamlGraph yamlGraph = named.graph()` and before any validation.

- [ ] **Step 4: Write integration test — bare refs through full compilation**

Add to an appropriate test class:

```java
@Test
void endToEnd_bareVariableRef_resolvedAfterNormalization() {
    // YAML with bare ${batch_size} instead of ${var.batch_size}
    // Verify the compiled graph's NodeSpec has the resolved integer value
}
```

The implementer should construct the full test with actual YAML parsing → normalization → compilation, verifying that bare references produce the same result as fully-prefixed ones.

- [ ] **Step 5: Run full test suite**

Run: `mvn --batch-mode test -pl yaml/deployment`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java
git add yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/YamlPrefixNormalizationTest.java
git commit -m "feat(#149): wire VariablePrefixRewriter for bare reference normalization"
```

### Task 5: Full build verification

**Files:**
- No new files — verification only

- [ ] **Step 1: Full project build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules compile and pass tests

- [ ] **Step 2: Verify example projects still compile**

The example projects (`examples/pipeline-yaml/`, `examples/pipeline-annotated/`, `examples/pipeline-ts/`, `examples/webapp-yaml/`) use YAML deserialization — verify they pick up the widened `variables` type without issues.

Run: `mvn --batch-mode test -pl examples/pipeline-yaml,examples/webapp-yaml`
Expected: PASS

- [ ] **Step 3: Commit any fixes**

If any example or downstream module needed adjustment, commit the fixes.

```bash
git commit -m "fix(#149): adjust examples for YamlGraph model change"
```

## References

- `specs/issue-149-yamlgraph-csv-foreach/2026-09-23-yamlgraph-csv-foreach-design.md` — design spec
- `specs/issue-149-yamlgraph-csv-foreach/decisions.md` — 6 design decisions
- `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlGraph.java` — model record
- `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java:37-280` — recorder methods
- `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java:52-162,307-343,707-780` — deployment processor
- `io.casehub.yaml.core.resolver.VariableResolver` — platform#419 + #426
- `io.casehub.yaml.core.resolver.VariablePrefixRewriter` — bare reference normalization
- `io.casehub.yaml.core.data.CsvDataSource` — CSV parsing
- casehubio/casehub-desiredstate#148 — predecessor
- casehubio/platform#419, #426 — yaml-core prerequisites
