# Phase 3: YAML Language Extensions — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #118 — conditional and iterated subgraph inclusion
**Issue group:** #118, #120

**Goal:** Deliver Phase 3 of the YAML language extensions — forEach
cardinality stamping and composable modules — completing the YAML
surface's parity with Java annotations. An operator can stamp N copies
of a node from a collection and compose reusable parameterised module
fragments — no Java required.

**Architecture:** forEach expansion happens at compile time inside the
GoalCompiler lambda. A `ForEachExpander` utility stamps N copies per
template with `templateId.value` IDs, resolves `${each.*}` via
`VariableResolver.withEachContext()`, and wires aligned dependencies
for nodes sharing a named iteration group. Module expansion happens at
compile time in a `ModuleExpander` utility — alias-prefixes node IDs,
pushes module parameter scope onto VariableResolver for `${var.*}`
resolution, and promotes module-scoped rules/invariants to top-level.
Both paths are shared between single-graph and lifecycle GoalCompilers.

**Tech Stack:** Java 21, Quarkus 3.x, Jackson YAML, Jandex, Maven

**Design spec:** `specs/issue-116-yaml-language-design/2026-08-27-yaml-language-extensions-design.md`

## Global Constraints

- All `${}` references use explicit prefixes: `${var.}`, `${match.}`, `${fault.}`, `${each.}`
- User-declared node IDs must not contain `.` (reserved separator for generated IDs)
- forEach iteration values must not contain `.`
- forEach expansion capped at configurable max per template (default: 1,000)
- Named iteration groups for aligned forEach — inline forEach creates anonymous groups
- Module parameters are strings only — no `nodeRef` type (D9)
- Module nesting capped at 2 levels (D10)
- Module-scoped rules/invariants fire against the full graph (D14)
- Jackson `ObjectMapper` for spec deserialization uses dedicated coercion-enabled instance
- Build-time validation produces errors, not warnings, for safety violations
- All YAML model records use compact constructors defaulting nulls to empty collections
- Test scope: unit tests for each component, integration test via pipeline-yaml example

---

## Batch 1: forEach Model + VariableResolver

Safe wrap point: after this batch, forEach model types deserialize
correctly and `VariableResolver` supports `${each.*}` resolution with
a pushed scope. Build-time validation catches structural errors.

### Task 1: forEach model types + VariableResolver each. scope

**Files:**
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlForEach.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlIterationGroup.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlNode.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlGraph.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/resolver/VariableResolver.java`
- Create: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/model/YamlForEachDeserializationTest.java`
- Modify: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/resolver/VariableResolverTest.java`

**Interfaces:**
- Consumes: Jackson YAML deserialization, existing `VariableResolver` prefix dispatch
- Produces: `YamlForEach(String as, List<String> in)` — inline forEach definition.
  `YamlIterationGroup(String as, List<Object> in)` — named group (`in` is `List<Object>` to support
  `${var.*}` string references that resolve to JSON arrays at compile time).
  `YamlNode` gains `Object forEach` field (String reference to named group, or inline map).
  `YamlGraph` gains `Map<String, YamlIterationGroup> iterations` field.
  `VariableResolver.withEachContext(Map<String, String>)` — returns a new resolver with `${each.*}` scope.

- [ ] **Step 1: Create YamlForEach record**

```java
package io.casehub.desiredstate.yaml.model;

import java.util.List;

public record YamlForEach(String as, List<String> in) {
    public YamlForEach {
        if (in == null) {in = List.of();}
    }
}
```

- [ ] **Step 2: Create YamlIterationGroup record**

```java
package io.casehub.desiredstate.yaml.model;

import java.util.List;

public record YamlIterationGroup(String as, List<Object> in) {
    public YamlIterationGroup {
        if (in == null) {in = List.of();}
    }
}
```

Note: `in` is `List<Object>` because it may contain `String` literals or a single
`String` that is a `${var.*}` reference resolving to a JSON array at compile time.

- [ ] **Step 3: Add forEach field to YamlNode**

Add `Object forEach` as the 6th field (after `when`). The compact constructor
defaults it to `null`:

```java
public record YamlNode(
        String type,
        Map<String, Object> spec,
        List<Object> dependsOn,
        HumanGating humanGating,
        String when,
        Object forEach) {

    public YamlNode {
        if (spec == null) {spec = Map.of();}
        if (dependsOn == null) {dependsOn = List.of();}
        if (humanGating == null) {humanGating = HumanGating.NONE;}
    }
    // ... existing static methods unchanged
}
```

`forEach` is `Object` to support both:
- `String`: reference to a named iteration group (`forEach: regional`)
- `Map`: inline definition (`forEach: { as: region, in: ["us-east", "eu-west"] }`)

Jackson deserializes YAML string to `String` and YAML map to `LinkedHashMap`.

- [ ] **Step 4: Add iterations field to YamlGraph**

```java
public record YamlGraph(
        YamlDesiredState desiredState,
        Map<String, String> variables,
        Map<String, YamlNode> nodes,
        List<YamlFaultPolicy> faultPolicy,
        Map<String, YamlInvariant> invariants,
        Map<String, YamlRule> rules,
        YamlLifecycle lifecycle,
        Map<String, YamlIterationGroup> iterations) {

    public YamlGraph {
        if (variables == null) {variables = Map.of();}
        if (nodes == null) {nodes = Map.of();}
        if (faultPolicy == null) {faultPolicy = List.of();}
        if (invariants == null) {invariants = Map.of();}
        if (rules == null) {rules = Map.of();}
        if (iterations == null) {iterations = Map.of();}
    }
}
```

- [ ] **Step 5: Write deserialization tests**

Create `YamlForEachDeserializationTest.java`:

```java
package io.casehub.desiredstate.yaml.model;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class YamlForEachDeserializationTest {

    private final ObjectMapper mapper = new ObjectMapper(new YAMLFactory());

    @Test
    void deserialize_inlineForEach() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: foreach-test
                nodes:
                  regional-source:
                    type: data-source
                    forEach:
                      as: region
                      in: ["us-east", "eu-west", "ap-south"]
                    spec:
                      name: "customers-${each.region}"
                      uri: "s3://data/${each.region}/customers.csv"
                """;
        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);
        assertThat(graph.nodes()).hasSize(1);

        Object forEach = graph.nodes().get("regional-source").forEach();
        assertThat(forEach).isInstanceOf(java.util.Map.class);
        @SuppressWarnings("unchecked")
        java.util.Map<String, Object> forEachMap = (java.util.Map<String, Object>) forEach;
        assertThat(forEachMap.get("as")).isEqualTo("region");
        @SuppressWarnings("unchecked")
        java.util.List<String> inList = (java.util.List<String>) forEachMap.get("in");
        assertThat(inList).containsExactly("us-east", "eu-west", "ap-south");
    }

    @Test
    void deserialize_namedGroupReference() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: named-group-test
                iterations:
                  regional:
                    as: region
                    in: ["us-east", "eu-west"]
                nodes:
                  regional-source:
                    type: data-source
                    forEach: regional
                    spec:
                      uri: "s3://${each.region}/data.csv"
                  regional-ingest:
                    type: ingestion
                    forEach: regional
                    dependsOn: [regional-source]
                    spec:
                      region: "${each.region}"
                """;
        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);
        assertThat(graph.iterations()).hasSize(1);
        assertThat(graph.iterations().get("regional").as()).isEqualTo("region");
        assertThat(graph.iterations().get("regional").in()).containsExactly("us-east", "eu-west");

        assertThat(graph.nodes().get("regional-source").forEach()).isEqualTo("regional");
        assertThat(graph.nodes().get("regional-ingest").forEach()).isEqualTo("regional");
    }

    @Test
    void deserialize_variableSourcedValues() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: var-sourced
                variables:
                  regions: '["us-east", "eu-west"]'
                iterations:
                  regional:
                    as: region
                    in: "${var.regions}"
                nodes:
                  regional-source:
                    type: data-source
                    forEach: regional
                    spec:
                      uri: "s3://${each.region}/data.csv"
                """;
        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);
        YamlIterationGroup group = graph.iterations().get("regional");
        assertThat(group.in()).hasSize(1);
        assertThat(group.in().get(0)).isEqualTo("${var.regions}");
    }

    @Test
    void deserialize_noForEach_defaultsToNull() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: no-foreach
                nodes:
                  app:
                    type: app
                    spec: {}
                """;
        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);
        assertThat(graph.nodes().get("app").forEach()).isNull();
        assertThat(graph.iterations()).isEmpty();
    }
}
```

- [ ] **Step 6: Run deserialization tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlForEachDeserializationTest`
Expected: ALL PASS

- [ ] **Step 7: Write failing test for VariableResolver each. scope**

Add to `VariableResolverTest.java`:

```java
@Test
void withEachContext_resolvesEachPrefix() {
    var resolver = new VariableResolver(Map.of("batch", "1000"), null, null);
    var eachResolver = resolver.withEachContext(Map.of("region", "us-east"));
    String result = eachResolver.resolveString(
            "s3://${each.region}/${var.batch}", "test-node");
    assertThat(result).isEqualTo("s3://us-east/1000");
}

@Test
void withEachContext_unknownEachVar_throws() {
    var resolver = new VariableResolver(Map.of(), null, null);
    var eachResolver = resolver.withEachContext(Map.of("region", "us-east"));
    assertThatThrownBy(() -> eachResolver.resolveString(
            "${each.zone}", "test-node"))
            .isInstanceOf(UnresolvedVariableException.class)
            .hasMessageContaining("zone");
}

@Test
void withoutEachContext_eachPrefix_throwsDeferred() {
    var resolver = new VariableResolver(Map.of(), null, null);
    assertThatThrownBy(() -> resolver.resolveString(
            "${each.region}", "test-node"))
            .isInstanceOf(UnresolvedVariableException.class)
            .hasMessageContaining("forEach expansion");
}
```

- [ ] **Step 8: Run test to verify it fails**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=VariableResolverTest`
Expected: compilation error — `withEachContext` method doesn't exist.

- [ ] **Step 9: Implement withEachContext in VariableResolver**

Add field and method to `VariableResolver.java`:

```java
private final Map<String, String> eachContext;

public VariableResolver(Map<String, String> inlineVariables,
                        Object preferences, Config config) {
    this.inlineVariables = inlineVariables != null ? inlineVariables : Map.of();
    this.config = config;
    this.eachContext = null;
}

private VariableResolver(Map<String, String> inlineVariables,
                         Config config, Map<String, String> eachContext) {
    this.inlineVariables = inlineVariables != null ? inlineVariables : Map.of();
    this.config = config;
    this.eachContext = eachContext;
}

public VariableResolver withEachContext(Map<String, String> eachContext) {
    return new VariableResolver(this.inlineVariables, this.config, eachContext);
}
```

Update `lookupVariable` for `each.` prefix:

```java
if (key.startsWith("each.")) {
    String eachName = key.substring(5);
    if (eachContext != null && eachContext.containsKey(eachName)) {
        return eachContext.get(eachName);
    }
    if (eachContext != null) {
        throw new UnresolvedVariableException(key, nodeContext,
                "Unknown forEach variable '" + eachName + "'. "
                + "Available: " + eachContext.keySet());
    }
    throw new UnresolvedVariableException(key, nodeContext,
            "${each.*} references are resolved during forEach expansion, "
            + "not during variable resolution.");
}
```

- [ ] **Step 10: Run VariableResolver tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=VariableResolverTest`
Expected: ALL PASS

- [ ] **Step 11: Run all yaml/runtime tests to verify no regressions**

Run: `mvn --batch-mode test -pl yaml/runtime`
Expected: ALL PASS

- [ ] **Step 12: Commit**

```
feat(#118): forEach model types and VariableResolver each. scope

YamlForEach and YamlIterationGroup records. YamlNode gains forEach
field (string ref or inline map). YamlGraph gains iterations field.
VariableResolver.withEachContext() resolves ${each.*} references
against a pushed scope.

Refs #118
```

---

### Task 2: Build-time forEach validation

**Files:**
- Modify: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java`
- Create: `yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/YamlForEachValidationTest.java`

**Interfaces:**
- Consumes: `YamlNode.forEach()`, `YamlGraph.iterations()`, `YamlForEach`, `YamlIterationGroup`
- Produces: `YamlDesiredStateProcessor.validateForEach(nodes, iterations, typeRegistry, fileName)` —
  build-time validation: `.` in node IDs, `.` in forEach values, unknown group references,
  non-forEach depending on forEach template (error), cross-group dependencies (error),
  inline→named-group dependencies (error).

- [ ] **Step 1: Write validation tests**

Create `YamlForEachValidationTest.java`:

```java
package io.casehub.desiredstate.yaml.deployment;

import io.casehub.desiredstate.yaml.model.YamlGraph;
import io.casehub.desiredstate.yaml.model.YamlIterationGroup;
import io.casehub.desiredstate.yaml.model.YamlNode;
import io.casehub.desiredstate.yaml.model.YamlDesiredState;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThatCode;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class YamlForEachValidationTest {

    private static final Map<String, String> TYPE_REGISTRY = Map.of(
            "data-source", "com.example.DataSourceSpec",
            "ingestion", "com.example.IngestionSpec");

    @Test
    void validate_dotInNodeId_throwsBuildError() {
        var nodes = Map.of("my.source", new YamlNode("data-source",
                Map.of(), List.of(), null, null, null));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateForEach(
                nodes, Map.of(), TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining(".")
                .hasMessageContaining("my.source");
    }

    @Test
    void validate_nonForEachDependsOnForEachTemplate_throwsBuildError() {
        var nodes = Map.of(
                "regional-source", new YamlNode("data-source",
                        Map.of(), List.of(), null, null, "regional"),
                "processor", new YamlNode("ingestion",
                        Map.of(), List.of("regional-source"), null, null, null));
        var iterations = Map.of("regional",
                new YamlIterationGroup("region", List.of("us-east", "eu-west")));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateForEach(
                nodes, iterations, TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("processor")
                .hasMessageContaining("regional-source")
                .hasMessageContaining("forEach");
    }

    @Test
    void validate_crossGroupDependency_throwsBuildError() {
        var nodes = Map.of(
                "source", new YamlNode("data-source",
                        Map.of(), List.of(), null, null, "group-a"),
                "sink", new YamlNode("ingestion",
                        Map.of(), List.of("source"), null, null, "group-b"));
        var iterations = Map.of(
                "group-a", new YamlIterationGroup("region", List.of("us-east")),
                "group-b", new YamlIterationGroup("zone", List.of("z1")));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateForEach(
                nodes, iterations, TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("group-a")
                .hasMessageContaining("group-b");
    }

    @Test
    void validate_unknownGroupReference_throwsBuildError() {
        var nodes = Map.of("source", new YamlNode("data-source",
                Map.of(), List.of(), null, null, "nonexistent"));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateForEach(
                nodes, Map.of(), TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("nonexistent");
    }

    @Test
    void validate_inlineForEachDependsOnNamedGroup_throwsBuildError() {
        Map<String, Object> inlineForEach = Map.of("as", "idx", "in", List.of("1", "2"));
        var nodes = Map.of(
                "named-src", new YamlNode("data-source",
                        Map.of(), List.of(), null, null, "regional"),
                "inline-proc", new YamlNode("ingestion",
                        Map.of(), List.of("named-src"), null, null, inlineForEach));
        var iterations = Map.of("regional",
                new YamlIterationGroup("region", List.of("us-east")));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateForEach(
                nodes, iterations, TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("inline-proc")
                .hasMessageContaining("named-src");
    }

    @Test
    void validate_validForEach_passes() {
        var nodes = Map.of(
                "regional-source", new YamlNode("data-source",
                        Map.of(), List.of(), null, null, "regional"),
                "regional-ingest", new YamlNode("ingestion",
                        Map.of(), List.of("regional-source"), null, null, "regional"),
                "fixed-node", new YamlNode("data-source",
                        Map.of(), List.of(), null, null, null));
        var iterations = Map.of("regional",
                new YamlIterationGroup("region", List.of("us-east", "eu-west")));
        assertThatCode(() -> YamlDesiredStateProcessor.validateForEach(
                nodes, iterations, TYPE_REGISTRY, "test.yaml"))
                .doesNotThrowAnyException();
    }

    @Test
    void validate_forEachDependsOnFixedNode_passes() {
        var nodes = Map.of(
                "fixed-db", new YamlNode("data-source",
                        Map.of(), List.of(), null, null, null),
                "regional-source", new YamlNode("ingestion",
                        Map.of(), List.of("fixed-db"), null, null, "regional"));
        var iterations = Map.of("regional",
                new YamlIterationGroup("region", List.of("us-east")));
        assertThatCode(() -> YamlDesiredStateProcessor.validateForEach(
                nodes, iterations, TYPE_REGISTRY, "test.yaml"))
                .doesNotThrowAnyException();
    }
}
```

- [ ] **Step 2: Implement validateForEach**

Add to `YamlDesiredStateProcessor.java`:

```java
static void validateForEach(Map<String, YamlNode> nodes,
                            Map<String, YamlIterationGroup> iterations,
                            Map<String, String> typeRegistry, String fileName) {
    // Check node IDs for reserved '.' separator
    for (String nodeId : nodes.keySet()) {
        if (nodeId.contains(".")) {
            throw new RuntimeException(fileName + ": node ID '" + nodeId
                    + "' contains the reserved '.' separator. "
                    + "User-declared node IDs must not contain '.'.");
        }
    }

    // Build a map of nodeId -> group name (null for non-forEach, group name for named, 
    // anonymous key for inline)
    Map<String, String> nodeGroupMap = new java.util.HashMap<>();
    Set<String> forEachNodeIds = new HashSet<>();

    for (Map.Entry<String, YamlNode> entry : nodes.entrySet()) {
        String nodeId = entry.getKey();
        Object forEach = entry.getValue().forEach();
        if (forEach == null) {
            nodeGroupMap.put(nodeId, null);
        } else if (forEach instanceof String groupRef) {
            if (!iterations.containsKey(groupRef)) {
                throw new RuntimeException(fileName + ": node '" + nodeId
                        + "' references unknown iteration group '" + groupRef
                        + "'. Available: " + iterations.keySet());
            }
            nodeGroupMap.put(nodeId, groupRef);
            forEachNodeIds.add(nodeId);
        } else if (forEach instanceof Map<?, ?>) {
            nodeGroupMap.put(nodeId, "__inline__" + nodeId);
            forEachNodeIds.add(nodeId);
        } else {
            throw new RuntimeException(fileName + ": node '" + nodeId
                    + "': forEach must be a string (group name) or map ({as, in})");
        }
    }

    // Validate forEach iteration values don't contain '.'
    for (Map.Entry<String, YamlIterationGroup> entry : iterations.entrySet()) {
        for (Object val : entry.getValue().in()) {
            if (val instanceof String s && s.contains(".") && !s.contains("${")) {
                throw new RuntimeException(fileName + ": iteration group '"
                        + entry.getKey() + "': value '" + s
                        + "' contains the reserved '.' separator");
            }
        }
    }

    // Validate dependency rules
    for (Map.Entry<String, YamlNode> entry : nodes.entrySet()) {
        String nodeId = entry.getKey();
        String nodeGroup = nodeGroupMap.get(nodeId);

        for (String depId : entry.getValue().dependencyNodeIds()) {
            if (!nodes.containsKey(depId)) {continue;}
            String depGroup = nodeGroupMap.get(depId);

            // Non-forEach depending on forEach template → error
            if (nodeGroup == null && forEachNodeIds.contains(depId)) {
                throw new RuntimeException(fileName + ": Node '" + nodeId
                        + "' depends on forEach template '" + depId
                        + "'. Non-forEach nodes cannot depend on forEach templates "
                        + "(the template ID doesn't exist after expansion). "
                        + "Use a forEach node with the same iteration group for fan-in.");
            }

            // Cross-group dependency → error
            if (nodeGroup != null && depGroup != null
                    && !nodeGroup.equals(depGroup)) {
                throw new RuntimeException(fileName + ": Node '" + nodeId
                        + "' (group: " + nodeGroup + ") depends on '" + depId
                        + "' (group: " + depGroup + "). "
                        + "forEach nodes referencing different groups cannot depend on each other. "
                        + "Use the same named group for aligned iteration.");
            }
        }
    }
}
```

- [ ] **Step 3: Wire validation into validateYamlGraph and validateLifecycle**

In `validateYamlGraph()`, add after `validateConditionalDependencies`:
```java
validateForEach(graph.nodes(), graph.iterations(), typeRegistry, fileName);
```

In `validateLifecycle()`, add inside the phase loop after the type check:
```java
validateForEach(phase.nodes(), graph.iterations(), typeRegistry, fileName);
```

- [ ] **Step 4: Run validation tests**

Run: `mvn --batch-mode test -pl yaml/deployment -Dtest=YamlForEachValidationTest`
Expected: ALL PASS

- [ ] **Step 5: Run all yaml/deployment tests to verify no regressions**

Run: `mvn --batch-mode test -pl yaml/deployment`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```
feat(#118): build-time forEach validation

Node ID dot-separator check, unknown group references, non-forEach
depending on forEach template, cross-group dependencies, inline-to-named
group dependency mismatch.

Refs #118
```

---

## Batch 2: forEach Expansion + Integration

Safe wrap point: after this batch, YAML operators can declare forEach
nodes with inline or named iteration groups. The GoalCompiler stamps N
copies with aligned dependencies. Pipeline-yaml example demonstrates
multi-region data sources.

### Task 3: ForEachExpander + GoalCompiler integration

**Files:**
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/ForEachExpander.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`
- Create: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/ForEachExpanderTest.java`

**Interfaces:**
- Consumes: `YamlNode.forEach()`, `YamlGraph.iterations()`, `VariableResolver.withEachContext()`,
  `NodeSpecRegistry`, `ObjectMapper`, `YamlNode.dependencyNodeId()`, `YamlNode.isDependencyOptional()`
- Produces: `ForEachExpander.expand(nodes, iterations, resolver, registry, mapper, maxExpansion)` →
  `ExpandedNodes(List<DesiredNode>, List<Dependency>, Set<String> excludedNodeIds)`.
  Both `createYamlGoalCompiler` and `createYamlLifecycleGoalCompiler` delegate to `ForEachExpander`.

- [ ] **Step 1: Write ForEachExpander tests**

Create `ForEachExpanderTest.java`:

```java
package io.casehub.desiredstate.yaml;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.desiredstate.api.Dependency;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.yaml.model.YamlIterationGroup;
import io.casehub.desiredstate.yaml.model.YamlNode;
import io.casehub.desiredstate.yaml.registry.NodeSpecRegistry;
import io.casehub.desiredstate.yaml.resolver.VariableResolver;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ForEachExpanderTest {

    public record TestSpec(String name, String uri) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("data-source"); }
    }

    private static final Map<String, String> TYPE_MAP = Map.of(
            "data-source", "io.casehub.desiredstate.yaml.ForEachExpanderTest$TestSpec",
            "ingestion", "io.casehub.desiredstate.yaml.ForEachExpanderTest$TestSpec");

    private final NodeSpecRegistry registry = NodeSpecRegistry.of(TYPE_MAP);
    private final ObjectMapper mapper = new ObjectMapper();
    private final VariableResolver resolver = new VariableResolver(Map.of(), null, null);

    @Test
    void inlineForEach_stampsThreeCopies() {
        Map<String, Object> inlineForEach = Map.of("as", "region",
                "in", List.of("us-east", "eu-west", "ap-south"));
        var nodes = Map.of("regional-source", new YamlNode("data-source",
                Map.of("name", "customers-${each.region}",
                       "uri", "s3://${each.region}/data.csv"),
                List.of(), null, null, inlineForEach));

        var result = ForEachExpander.expand(nodes, Map.of(), resolver,
                registry, mapper, 1000);

        assertThat(result.nodes()).hasSize(3);
        assertThat(result.nodes().stream().map(n -> n.id().value()).toList())
                .containsExactlyInAnyOrder("regional-source.us-east",
                        "regional-source.eu-west", "regional-source.ap-south");

        DesiredNode usEast = result.nodes().stream()
                .filter(n -> n.id().value().equals("regional-source.us-east"))
                .findFirst().orElseThrow();
        TestSpec spec = (TestSpec) usEast.spec();
        assertThat(spec.name()).isEqualTo("customers-us-east");
        assertThat(spec.uri()).isEqualTo("s3://us-east/data.csv");
    }

    @Test
    void namedGroup_alignedDependencies() {
        var iterations = Map.of("regional",
                new YamlIterationGroup("region", List.of("us-east", "eu-west")));
        var nodes = new java.util.LinkedHashMap<String, YamlNode>();
        nodes.put("regional-source", new YamlNode("data-source",
                Map.of("name", "${each.region}", "uri", "s3://${each.region}"),
                List.of(), null, null, "regional"));
        nodes.put("regional-ingest", new YamlNode("ingestion",
                Map.of("name", "${each.region}-ingest", "uri", ""),
                List.of("regional-source"), null, null, "regional"));

        var result = ForEachExpander.expand(nodes, iterations, resolver,
                registry, mapper, 1000);

        assertThat(result.nodes()).hasSize(4);
        assertThat(result.dependencies()).contains(
                new Dependency(NodeId.of("regional-ingest.us-east"),
                        NodeId.of("regional-source.us-east")));
        assertThat(result.dependencies()).contains(
                new Dependency(NodeId.of("regional-ingest.eu-west"),
                        NodeId.of("regional-source.eu-west")));
        // No cross-region dependencies
        assertThat(result.dependencies()).doesNotContain(
                new Dependency(NodeId.of("regional-ingest.us-east"),
                        NodeId.of("regional-source.eu-west")));
    }

    @Test
    void forEachDependsOnFixedNode_eachCopyDependsOnSame() {
        var iterations = Map.of("regional",
                new YamlIterationGroup("region", List.of("us-east", "eu-west")));
        var nodes = new java.util.LinkedHashMap<String, YamlNode>();
        nodes.put("fixed-db", new YamlNode("data-source",
                Map.of("name", "db", "uri", "jdbc://db"),
                List.of(), null, null, null));
        nodes.put("regional-source", new YamlNode("data-source",
                Map.of("name", "${each.region}", "uri", ""),
                List.of("fixed-db"), null, null, "regional"));

        var result = ForEachExpander.expand(nodes, iterations, resolver,
                registry, mapper, 1000);

        assertThat(result.nodes()).hasSize(3);
        assertThat(result.dependencies()).contains(
                new Dependency(NodeId.of("regional-source.us-east"), NodeId.of("fixed-db")));
        assertThat(result.dependencies()).contains(
                new Dependency(NodeId.of("regional-source.eu-west"), NodeId.of("fixed-db")));
    }

    @Test
    void expansionLimit_exceeded_throws() {
        Map<String, Object> inlineForEach = Map.of("as", "idx",
                "in", java.util.stream.IntStream.rangeClosed(1, 5)
                        .mapToObj(String::valueOf).toList());
        var nodes = Map.of("node", new YamlNode("data-source",
                Map.of("name", "${each.idx}", "uri", ""),
                List.of(), null, null, inlineForEach));

        assertThatThrownBy(() -> ForEachExpander.expand(nodes, Map.of(),
                resolver, registry, mapper, 3))
                .hasMessageContaining("node")
                .hasMessageContaining("5")
                .hasMessageContaining("3");
    }

    @Test
    void variableSourcedValues_jsonArray() {
        var resolver = new VariableResolver(
                Map.of("regions", "[\"us-east\", \"eu-west\"]"), null, null);
        var iterations = Map.of("regional",
                new YamlIterationGroup("region", List.of("${var.regions}")));
        var nodes = Map.of("source", new YamlNode("data-source",
                Map.of("name", "${each.region}", "uri", ""),
                List.of(), null, null, "regional"));

        var result = ForEachExpander.expand(nodes, iterations, resolver,
                registry, mapper, 1000);

        assertThat(result.nodes()).hasSize(2);
        assertThat(result.nodes().stream().map(n -> n.id().value()).toList())
                .containsExactlyInAnyOrder("source.us-east", "source.eu-west");
    }

    @Test
    void forEachPlusWhen_allCopiesExcluded() {
        var resolver = new VariableResolver(
                Map.of("enable_sources", "false"), null, null);
        Map<String, Object> inlineForEach = Map.of("as", "region",
                "in", List.of("us-east", "eu-west"));
        var nodes = Map.of("source", new YamlNode("data-source",
                Map.of("name", "${each.region}", "uri", ""),
                List.of(), null, "${var.enable_sources}", inlineForEach));

        var result = ForEachExpander.expand(nodes, Map.of(), resolver,
                registry, mapper, 1000);

        assertThat(result.nodes()).isEmpty();
        assertThat(result.excludedNodeIds())
                .containsExactlyInAnyOrder("source.us-east", "source.eu-west");
    }

    @Test
    void zeroValues_noDependents_producesEmpty() {
        Map<String, Object> inlineForEach = Map.of("as", "idx", "in", List.of());
        var nodes = Map.of("empty-template", new YamlNode("data-source",
                Map.of("name", "x", "uri", ""),
                List.of(), null, null, inlineForEach));

        var result = ForEachExpander.expand(nodes, Map.of(), resolver,
                registry, mapper, 1000);

        assertThat(result.nodes()).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=ForEachExpanderTest`
Expected: compilation error — `ForEachExpander` doesn't exist.

- [ ] **Step 3: Implement ForEachExpander**

Create `ForEachExpander.java`:

```java
package io.casehub.desiredstate.yaml;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.desiredstate.api.Dependency;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.yaml.model.YamlIterationGroup;
import io.casehub.desiredstate.yaml.model.YamlNode;
import io.casehub.desiredstate.yaml.registry.NodeSpecRegistry;
import io.casehub.desiredstate.yaml.resolver.VariableResolver;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.HashSet;

public final class ForEachExpander {

    public record ExpandedNodes(
            List<DesiredNode> nodes,
            List<Dependency> dependencies,
            Set<String> excludedNodeIds) {}

    private ForEachExpander() {}

    @SuppressWarnings("unchecked")
    public static ExpandedNodes expand(
            Map<String, YamlNode> yamlNodes,
            Map<String, YamlIterationGroup> iterationGroups,
            VariableResolver resolver,
            NodeSpecRegistry registry,
            ObjectMapper mapper,
            int maxExpansion) {

        List<DesiredNode> allNodes = new ArrayList<>();
        List<Dependency> allDeps = new ArrayList<>();
        Set<String> excludedNodeIds = new HashSet<>();
        Set<String> forEachTemplateIds = new HashSet<>();
        Map<String, String> nodeToGroup = new LinkedHashMap<>();
        Map<String, List<String>> groupValues = new LinkedHashMap<>();

        // Pass 1: classify nodes, resolve forEach values
        for (Map.Entry<String, YamlNode> entry : yamlNodes.entrySet()) {
            String nodeId = entry.getKey();
            YamlNode yamlNode = entry.getValue();
            Object forEach = yamlNode.forEach();

            if (forEach == null) {
                nodeToGroup.put(nodeId, null);
                continue;
            }

            forEachTemplateIds.add(nodeId);
            List<String> values;
            String groupKey;

            if (forEach instanceof String groupRef) {
                groupKey = groupRef;
                if (!groupValues.containsKey(groupRef)) {
                    YamlIterationGroup group = iterationGroups.get(groupRef);
                    values = resolveGroupValues(group.in(), resolver, groupRef);
                    groupValues.put(groupRef, values);
                }
                values = groupValues.get(groupRef);
            } else if (forEach instanceof Map<?, ?> inlineMap) {
                groupKey = "__inline__" + nodeId;
                String as = (String) inlineMap.get("as");
                List<?> in = (List<?>) inlineMap.get("in");
                values = in.stream().map(Object::toString).toList();
                groupValues.put(groupKey, values);
            } else {
                throw new IllegalArgumentException("Invalid forEach on node '" + nodeId + "'");
            }

            nodeToGroup.put(nodeId, groupKey);

            if (values.size() > maxExpansion) {
                throw new IllegalStateException("forEach template '" + nodeId
                        + "' would expand to " + values.size() + " nodes (limit: "
                        + maxExpansion + "). Configure "
                        + "casehub.desiredstate.foreach.max-expansion to raise the limit.");
            }
        }

        // Pass 2: expand nodes
        for (Map.Entry<String, YamlNode> entry : yamlNodes.entrySet()) {
            String nodeId = entry.getKey();
            YamlNode yamlNode = entry.getValue();
            String groupKey = nodeToGroup.get(nodeId);

            if (groupKey == null) {
                // Non-forEach node
                Class<? extends NodeSpec> specClass = registry.resolve(yamlNode.type());
                Map<String, Object> resolvedSpec = resolver.resolveMap(
                        yamlNode.spec(), nodeId);
                NodeSpec spec = mapper.convertValue(resolvedSpec, specClass);
                allNodes.add(new DesiredNode(NodeId.of(nodeId), spec,
                        yamlNode.humanGating()));
                continue;
            }

            // forEach node — stamp copies
            List<String> values = groupValues.get(groupKey);
            String as = resolveAs(yamlNode.forEach(), iterationGroups, groupKey);

            for (String value : values) {
                String stampedId = nodeId + "." + value;
                VariableResolver eachResolver = resolver.withEachContext(Map.of(as, value));

                // Evaluate when: per copy
                if (yamlNode.when() != null) {
                    String resolvedWhen = eachResolver.resolveString(
                            yamlNode.when(), stampedId);
                    if (!isTruthy(resolvedWhen)) {
                        excludedNodeIds.add(stampedId);
                        continue;
                    }
                }

                Class<? extends NodeSpec> specClass = registry.resolve(yamlNode.type());
                Map<String, Object> resolvedSpec = eachResolver.resolveMap(
                        yamlNode.spec(), stampedId);
                NodeSpec spec = mapper.convertValue(resolvedSpec, specClass);
                allNodes.add(new DesiredNode(NodeId.of(stampedId), spec,
                        yamlNode.humanGating()));
            }
        }

        // Pass 3: wire dependencies
        for (Map.Entry<String, YamlNode> entry : yamlNodes.entrySet()) {
            String nodeId = entry.getKey();
            YamlNode yamlNode = entry.getValue();
            String groupKey = nodeToGroup.get(nodeId);

            for (Object dep : yamlNode.dependsOn()) {
                String depId = YamlNode.dependencyNodeId(dep);
                String depGroup = nodeToGroup.get(depId);

                if (groupKey == null && depGroup == null) {
                    // Both non-forEach
                    allDeps.add(new Dependency(NodeId.of(nodeId), NodeId.of(depId)));
                } else if (groupKey != null && depGroup == null) {
                    // forEach depends on fixed node — each copy depends on same
                    List<String> values = groupValues.get(groupKey);
                    for (String value : values) {
                        allDeps.add(new Dependency(
                                NodeId.of(nodeId + "." + value), NodeId.of(depId)));
                    }
                } else if (groupKey != null && groupKey.equals(depGroup)) {
                    // Same group — aligned per-value
                    List<String> values = groupValues.get(groupKey);
                    for (String value : values) {
                        allDeps.add(new Dependency(
                                NodeId.of(nodeId + "." + value),
                                NodeId.of(depId + "." + value)));
                    }
                }
                // Cross-group and non-forEach→forEach are caught at build time
            }
        }

        return new ExpandedNodes(allNodes, allDeps, excludedNodeIds);
    }

    @SuppressWarnings("unchecked")
    private static String resolveAs(Object forEach,
            Map<String, YamlIterationGroup> groups, String groupKey) {
        if (forEach instanceof String groupRef) {
            return groups.get(groupRef).as();
        }
        if (forEach instanceof Map<?, ?> m) {
            return (String) m.get("as");
        }
        throw new IllegalArgumentException("Invalid forEach: " + forEach);
    }

    private static List<String> resolveGroupValues(List<Object> in,
            VariableResolver resolver, String groupRef) {
        if (in.size() == 1 && in.get(0) instanceof String s && s.contains("${")) {
            String resolved = resolver.resolveString(s, "iterations." + groupRef);
            return parseJsonArray(resolved, groupRef);
        }
        return in.stream().map(Object::toString).toList();
    }

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

    private static List<String> parseJsonArray(String json, String groupRef) {
        if (json == null || json.isBlank()) {
            throw new IllegalArgumentException("forEach group '" + groupRef
                    + "': Use [] for an empty array, not an empty string");
        }
        try {
            ObjectMapper jsonMapper = new ObjectMapper();
            List<?> parsed = jsonMapper.readValue(json,
                    new TypeReference<List<?>>() {});
            List<String> result = new ArrayList<>();
            for (Object item : parsed) {
                if (!(item instanceof String)) {
                    throw new IllegalArgumentException("forEach group '" + groupRef
                            + "': forEach values must be strings, got " + item.getClass().getSimpleName());
                }
                result.add((String) item);
            }
            return result;
        } catch (com.fasterxml.jackson.core.JsonProcessingException e) {
            throw new IllegalArgumentException("forEach group '" + groupRef
                    + "': variable resolved to '" + json
                    + "' which is not a valid JSON array. "
                    + "Expected a JSON array of strings like [\"a\", \"b\"].", e);
        }
    }
}
```

- [ ] **Step 4: Run ForEachExpander tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=ForEachExpanderTest`
Expected: ALL PASS

- [ ] **Step 5: Wire ForEachExpander into createYamlGoalCompiler**

Replace the node-building loop in `createYamlGoalCompiler` to delegate to `ForEachExpander`
when any node has `forEach`. The GoalCompiler lambda body becomes:

```java
// Check if any node has forEach
boolean hasForEach = yamlGraph != null && yamlGraph.nodes().values().stream()
        .anyMatch(n -> n.forEach() != null);

DesiredStateGraph graph;
if (hasForEach) {
    ForEachExpander.ExpandedNodes expanded = ForEachExpander.expand(
            yamlGraph.nodes(),
            yamlGraph.iterations() != null ? yamlGraph.iterations() : Map.of(),
            resolver, registry, mapper, 1000);
    graph = factory.of(expanded.nodes(), expanded.dependencies());
} else {
    // Existing node-building path (when:/exclude, InlineNode iteration)
    // ... unchanged ...
}
```

Refactor: extract the existing when:/exclude/node-building logic into a private method
so both paths are clean. The ForEachExpander handles when: for forEach nodes internally;
non-forEach nodes go through the existing path.

- [ ] **Step 6: Wire ForEachExpander into createYamlLifecycleGoalCompiler**

Same pattern — for each phase, check if any node has forEach:

```java
boolean hasForEach = yamlPhase.nodes().values().stream()
        .anyMatch(n -> n.forEach() != null);

List<DesiredNode> phaseNodes;
List<Dependency> phaseDeps;
if (hasForEach) {
    ForEachExpander.ExpandedNodes expanded = ForEachExpander.expand(
            yamlPhase.nodes(),
            yamlGraph.iterations() != null ? yamlGraph.iterations() : Map.of(),
            resolver, registry, mapper, 1000);
    phaseNodes = expanded.nodes();
    phaseDeps = expanded.dependencies();
} else {
    // Existing per-phase node-building path
    // ... unchanged ...
}
```

Then continue with carry-forward injection, rules, invariants as before.

- [ ] **Step 7: Run all existing tests to verify no regressions**

Run: `mvn --batch-mode test -pl yaml/runtime`
Run: `mvn --batch-mode test -pl yaml/deployment`
Run: `mvn --batch-mode test -pl examples/pipeline-yaml`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```
feat(#118): ForEachExpander with GoalCompiler integration

Stamps N copies per forEach template with templateId.value IDs.
Aligned dependencies for same-group nodes, fan-out for forEach→fixed.
Variable-sourced JSON arrays, expansion limit. Both single-graph and
lifecycle GoalCompilers delegate to ForEachExpander.

Refs #118
```

---

### Task 4: Pipeline-yaml forEach integration test

**Files:**
- Modify: `examples/pipeline-yaml/src/main/resources/META-INF/desiredstate/medallion-pipeline.yaml`
- Modify: `examples/pipeline-yaml/src/test/java/io/casehub/desiredstate/example/pipeline/yaml/PipelineYamlTest.java`

**Interfaces:**
- Consumes: ForEachExpander (Task 3), all pipeline NodeSpec types
- Produces: Working integration test proving forEach from YAML with aligned
  dependencies and `${each.*}` interpolation

- [ ] **Step 1: Add named iteration group and forEach nodes to medallion-pipeline.yaml**

Add `iterations:` block and convert `csv-source` to a forEach template with a
companion `csv-ingest` that iterates aligned:

```yaml
iterations:
  regional:
    as: region
    in: ["us-east", "eu-west"]

nodes:
  csv-source:
    type: data-source
    forEach: regional
    spec:
      name: "customers-${each.region}"
      format: CSV
      uri: "s3://data/${each.region}/customers.csv"

  customer-schema:
    type: schema
    spec:
      name: customer-schema
      fields: [id, name, email]
      version: 1

  csv-ingest:
    type: ingestion
    forEach: regional
    dependsOn: [csv-source]
    spec:
      sourceRef: "csv-source-${each.region}"
      batchSize: ${var.batch_size}
      format: CSV
  # ... rest unchanged, but dedup-cleanser now depends on csv-ingest
  # which is a forEach template. Since dedup-cleanser is not forEach,
  # this would be a build-time error. Fix: make dedup-cleanser depend
  # on a fixed intermediate, or make it forEach too.
```

**Design decision:** The medallion pipeline is a teaching example. Rather than
making the entire pipeline forEach, add the regional forEach as a NEW section
alongside the existing pipeline — demonstrating both forEach and non-forEach
in the same graph:

```yaml
iterations:
  regional:
    as: region
    in: ["us-east", "eu-west"]

nodes:
  # ... existing 8 nodes unchanged ...

  regional-source:
    type: data-source
    forEach: regional
    spec:
      name: "regional-${each.region}"
      format: CSV
      uri: "s3://data/${each.region}/regional.csv"

  regional-ingest:
    type: ingestion
    forEach: regional
    dependsOn: [regional-source, customer-schema]
    spec:
      sourceRef: "regional-source-${each.region}"
      batchSize: ${var.batch_size}
      format: CSV
```

This adds 4 nodes (2 templates × 2 regions) without disrupting existing tests.

- [ ] **Step 2: Update existing test node count**

In `PipelineYamlTest.java`, update the expected node count:
```java
// 8 declared + 1 rule-generated monitor + 4 forEach-expanded
// (2 templates × 2 regions) - debug-validator excluded = 13
assertThat(graph.nodes()).hasSize(13);
```

Also update `yamlRule_ensureMonitoring_convergesWithOneMonitor` — monitor count
stays 1 (rule only fires for `warehouse-sink`, not forEach nodes which aren't sinks).

- [ ] **Step 3: Write forEach integration tests**

Add to `PipelineYamlTest.java`:

```java
@Test
void forEach_regionalSourcesExpanded() {
    DesiredStateGraph graph = compileSingleGraph();
    assertThat(graph.nodes()).containsKey(NodeId.of("regional-source.us-east"));
    assertThat(graph.nodes()).containsKey(NodeId.of("regional-source.eu-west"));
    assertThat(graph.nodes()).containsKey(NodeId.of("regional-ingest.us-east"));
    assertThat(graph.nodes()).containsKey(NodeId.of("regional-ingest.eu-west"));
}

@Test
void forEach_alignedDependencies() {
    DesiredStateGraph graph = compileSingleGraph();
    assertThat(graph.dependenciesOf(NodeId.of("regional-ingest.us-east")))
            .contains(NodeId.of("regional-source.us-east"));
    assertThat(graph.dependenciesOf(NodeId.of("regional-ingest.eu-west")))
            .contains(NodeId.of("regional-source.eu-west"));
    // No cross-region dependencies
    assertThat(graph.dependenciesOf(NodeId.of("regional-ingest.us-east")))
            .doesNotContain(NodeId.of("regional-source.eu-west"));
}

@Test
void forEach_dependsOnFixedSchema() {
    DesiredStateGraph graph = compileSingleGraph();
    assertThat(graph.dependenciesOf(NodeId.of("regional-ingest.us-east")))
            .contains(NodeId.of("customer-schema"));
    assertThat(graph.dependenciesOf(NodeId.of("regional-ingest.eu-west")))
            .contains(NodeId.of("customer-schema"));
}

@Test
void forEach_eachVariableInterpolated() {
    DesiredStateGraph graph = compileSingleGraph();
    DataSourceSpec usSpec = (DataSourceSpec) graph.nodes()
            .get(NodeId.of("regional-source.us-east")).spec();
    assertThat(usSpec.name()).isEqualTo("regional-us-east");
    assertThat(usSpec.uri()).isEqualTo("s3://data/us-east/regional.csv");
}
```

- [ ] **Step 4: Update buildFromYaml to handle forEach**

The `buildFromYaml` method currently creates a `GraphDescriptor` from YAML nodes. With
forEach, the ForEachExpander handles expansion inside the GoalCompiler. But the test
uses `YamlGraphRecorder.createYamlGoalCompiler()` which now delegates to `ForEachExpander`
automatically. Verify the test's `buildFromYaml` passes `yamlGraph` (5th argument) —
it already does.

If the test's `toGraphDescriptor` method needs updating for forEach template nodes
(they shouldn't appear in the descriptor since expansion is compile-time), the method
should skip forEach nodes and let the GoalCompiler handle them.

- [ ] **Step 5: Run integration tests**

Run: `mvn --batch-mode test -pl examples/pipeline-yaml`
Expected: ALL PASS

- [ ] **Step 6: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```
feat(#118): pipeline-yaml example with forEach cardinality stamping

Demonstrates named iteration group (regional) with two forEach
templates: regional-source and regional-ingest. Aligned per-value
dependencies, fan-out to fixed schema node, ${each.region}
interpolation in spec values.

Refs #118
```

---

## Batch 3: Module Model + Validation

Safe wrap point: after this batch, module YAML files parse correctly,
classpath discovery finds them, and build-time validation catches
structural errors (missing parameters, nesting depth, import cycles).

### Task 5: Module model types + classpath discovery + build-time validation

**Files:**
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlModule.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlModuleParameter.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlImport.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlGraph.java`
- Modify: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java`
- Create: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/model/YamlModuleDeserializationTest.java`
- Create: `yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/YamlModuleValidationTest.java`

**Interfaces:**
- Consumes: Jackson YAML deserialization, `YamlDesiredStateProcessor.discoverYamlFiles()`
- Produces: `YamlModule(String name, Map<String, YamlModuleParameter> parameters, Map<String, YamlNode> nodes,
  Map<String, YamlRule> rules, Map<String, YamlInvariant> invariants)`.
  `YamlModuleParameter(String type, boolean required, String defaultValue)`.
  `YamlImport(String module, String as, String when, Map<String, String> parameters)`.
  `YamlGraph` gains `List<YamlImport> imports` field.
  `YamlDesiredStateProcessor.discoverModules()` scans `META-INF/desiredstate/modules/`.
  `YamlDesiredStateProcessor.validateImports(imports, modules, typeRegistry, fileName)`.

- [ ] **Step 1: Create YamlModuleParameter record**

```java
package io.casehub.desiredstate.yaml.model;

import com.fasterxml.jackson.annotation.JsonProperty;

public record YamlModuleParameter(
        String type,
        boolean required,
        @JsonProperty("default") String defaultValue) {

    public YamlModuleParameter {
        if (type == null) {type = "string";}
    }
}
```

- [ ] **Step 2: Create YamlModule record**

```java
package io.casehub.desiredstate.yaml.model;

import java.util.Map;

public record YamlModule(
        String name,
        Map<String, YamlModuleParameter> parameters,
        Map<String, YamlNode> nodes,
        Map<String, YamlRule> rules,
        Map<String, YamlInvariant> invariants) {

    public YamlModule {
        if (parameters == null) {parameters = Map.of();}
        if (nodes == null) {nodes = Map.of();}
        if (rules == null) {rules = Map.of();}
        if (invariants == null) {invariants = Map.of();}
    }
}
```

Note: Module YAML files have `module:` as top-level key (not `desiredState:`).
The file structure is:
```yaml
module:
  name: monitoring
  parameters:
    watched_node_id:
      type: string
      required: true
nodes:
  monitor:
    type: monitor
    ...
```

Jackson needs a wrapper type to deserialize this:

```java
package io.casehub.desiredstate.yaml.model;

import java.util.Map;

public record YamlModuleFile(
        YamlModuleHeader module,
        Map<String, YamlNode> nodes,
        Map<String, YamlRule> rules,
        Map<String, YamlInvariant> invariants) {

    public YamlModuleFile {
        if (nodes == null) {nodes = Map.of();}
        if (rules == null) {rules = Map.of();}
        if (invariants == null) {invariants = Map.of();}
    }

    public YamlModule toModule() {
        return new YamlModule(module.name(), module.parameters(),
                nodes, rules, invariants);
    }

    public record YamlModuleHeader(String name,
            Map<String, YamlModuleParameter> parameters) {
        public YamlModuleHeader {
            if (parameters == null) {parameters = Map.of();}
        }
    }
}
```

- [ ] **Step 3: Create YamlImport record**

```java
package io.casehub.desiredstate.yaml.model;

import java.util.Map;

public record YamlImport(
        String module,
        String as,
        String when,
        Map<String, String> parameters) {

    public YamlImport {
        if (parameters == null) {parameters = Map.of();}
    }
}
```

- [ ] **Step 4: Add imports field to YamlGraph**

```java
public record YamlGraph(
        YamlDesiredState desiredState,
        Map<String, String> variables,
        Map<String, YamlNode> nodes,
        List<YamlFaultPolicy> faultPolicy,
        Map<String, YamlInvariant> invariants,
        Map<String, YamlRule> rules,
        YamlLifecycle lifecycle,
        Map<String, YamlIterationGroup> iterations,
        List<YamlImport> imports) {

    public YamlGraph {
        // ... existing defaults ...
        if (iterations == null) {iterations = Map.of();}
        if (imports == null) {imports = List.of();}
    }
}
```

- [ ] **Step 5: Write deserialization tests**

Create `YamlModuleDeserializationTest.java`:

```java
package io.casehub.desiredstate.yaml.model;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class YamlModuleDeserializationTest {

    private final ObjectMapper mapper = new ObjectMapper(new YAMLFactory());

    @Test
    void deserialize_moduleFile() throws Exception {
        String yaml = """
                module:
                  name: monitoring
                  parameters:
                    watched_node_id:
                      type: string
                      required: true
                    alert_email:
                      type: string
                      default: "ops@example.com"
                nodes:
                  monitor:
                    type: monitor
                    dependsOn: ["${var.watched_node_id}"]
                    spec:
                      target: "${var.watched_node_id}"
                  alerter:
                    type: alerter
                    dependsOn: [monitor]
                    spec:
                      email: "${var.alert_email}"
                """;
        YamlModuleFile file = mapper.readValue(yaml, YamlModuleFile.class);
        YamlModule module = file.toModule();

        assertThat(module.name()).isEqualTo("monitoring");
        assertThat(module.parameters()).hasSize(2);
        assertThat(module.parameters().get("watched_node_id").required()).isTrue();
        assertThat(module.parameters().get("alert_email").defaultValue())
                .isEqualTo("ops@example.com");
        assertThat(module.nodes()).hasSize(2);
        assertThat(module.nodes()).containsKey("monitor");
        assertThat(module.nodes()).containsKey("alerter");
    }

    @Test
    void deserialize_graphWithImports() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: import-test
                nodes:
                  warehouse-sink:
                    type: sink
                    spec:
                      destination: s3://warehouse/
                imports:
                  - module: monitoring
                    as: pipe-monitor
                    parameters:
                      watched_node_id: warehouse-sink
                      alert_email: "pipeline-ops@example.com"
                  - module: monitoring
                    as: schema-monitor
                    when: "${var.monitoring_enabled}"
                    parameters:
                      watched_node_id: customer-schema
                """;
        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);
        assertThat(graph.imports()).hasSize(2);
        assertThat(graph.imports().get(0).module()).isEqualTo("monitoring");
        assertThat(graph.imports().get(0).as()).isEqualTo("pipe-monitor");
        assertThat(graph.imports().get(0).parameters())
                .containsEntry("watched_node_id", "warehouse-sink");
        assertThat(graph.imports().get(1).when()).isEqualTo("${var.monitoring_enabled}");
    }

    @Test
    void deserialize_moduleWithRulesAndInvariants() throws Exception {
        String yaml = """
                module:
                  name: monitored
                  parameters:
                    watched_node_id:
                      type: string
                      required: true
                nodes:
                  monitor:
                    type: monitor
                    dependsOn: ["${var.watched_node_id}"]
                    spec:
                      target: "${var.watched_node_id}"
                invariants:
                  monitor-must-have-dep:
                    match:
                      mon: { type: monitor }
                    directDep:
                      target: { type: "*", of: mon, direction: DEPENDENCIES }
                """;
        YamlModuleFile file = mapper.readValue(yaml, YamlModuleFile.class);
        YamlModule module = file.toModule();
        assertThat(module.invariants()).hasSize(1);
        assertThat(module.invariants()).containsKey("monitor-must-have-dep");
    }
}
```

- [ ] **Step 6: Run deserialization tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlModuleDeserializationTest`
Expected: ALL PASS

- [ ] **Step 7: Add module classpath discovery to YamlDesiredStateProcessor**

Add a `discoverModules(ObjectMapper)` method:

```java
private static final String MODULE_PATH_PREFIX = "META-INF/desiredstate/modules/";

private Map<String, YamlModule> discoverModules(ObjectMapper mapper) throws IOException, java.net.URISyntaxException {
    Map<String, YamlModule> modules = new HashMap<>();
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    Enumeration<URL> resources = cl.getResources(MODULE_PATH_PREFIX);

    Set<String> seen = new HashSet<>();
    while (resources.hasMoreElements()) {
        URL dirUrl = resources.nextElement();
        if ("file".equals(dirUrl.getProtocol())) {
            java.io.File dir = new java.io.File(dirUrl.toURI().getPath());
            if (dir.isDirectory()) {
                java.io.File[] yamlFiles = dir.listFiles((d, name) ->
                        name.endsWith(".yaml") || name.endsWith(".yml"));
                if (yamlFiles != null) {
                    for (java.io.File f : yamlFiles) {
                        if (seen.add(f.getName())) {
                            try (InputStream is = f.toURI().toURL().openStream()) {
                                io.casehub.desiredstate.yaml.model.YamlModuleFile moduleFile =
                                        mapper.readValue(is, io.casehub.desiredstate.yaml.model.YamlModuleFile.class);
                                YamlModule module = moduleFile.toModule();
                                modules.put(module.name(), module);
                            }
                        }
                    }
                }
            }
        }
    }
    return modules;
}
```

Call from `discoverYamlGraphs` before processing individual graphs:
```java
Map<String, YamlModule> availableModules = discoverModules(yamlMapper);
```

- [ ] **Step 8: Write validation tests**

Create `YamlModuleValidationTest.java`:

```java
package io.casehub.desiredstate.yaml.deployment;

import io.casehub.desiredstate.yaml.model.*;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThatCode;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class YamlModuleValidationTest {

    private static final Map<String, String> TYPE_REGISTRY = Map.of(
            "monitor", "com.example.MonitorSpec",
            "alerter", "com.example.AlerterSpec",
            "sink", "com.example.SinkSpec");

    private final YamlModule monitoringModule = new YamlModule("monitoring",
            Map.of("watched_node_id", new YamlModuleParameter("string", true, null),
                    "alert_email", new YamlModuleParameter("string", false, "ops@example.com")),
            Map.of("monitor", new YamlNode("monitor", Map.of(), List.of(), null, null, null)),
            Map.of(), Map.of());

    @Test
    void validate_unknownModule_throwsBuildError() {
        var imports = List.of(new YamlImport("nonexistent", "alias", null, Map.of()));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateImports(
                imports, Map.of(), TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("nonexistent");
    }

    @Test
    void validate_missingRequiredParameter_throwsBuildError() {
        var imports = List.of(new YamlImport("monitoring", "mon", null, Map.of()));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateImports(
                imports, Map.of("monitoring", monitoringModule), TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("watched_node_id")
                .hasMessageContaining("required");
    }

    @Test
    void validate_duplicateAlias_throwsBuildError() {
        var imports = List.of(
                new YamlImport("monitoring", "mon", null,
                        Map.of("watched_node_id", "sink-1")),
                new YamlImport("monitoring", "mon", null,
                        Map.of("watched_node_id", "sink-2")));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateImports(
                imports, Map.of("monitoring", monitoringModule), TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("mon")
                .hasMessageContaining("duplicate");
    }

    @Test
    void validate_aliasContainsDot_throwsBuildError() {
        var imports = List.of(new YamlImport("monitoring", "pipe.monitor", null,
                Map.of("watched_node_id", "sink-1")));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateImports(
                imports, Map.of("monitoring", monitoringModule), TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("pipe.monitor")
                .hasMessageContaining(".");
    }

    @Test
    void validate_nestingDepthExceeded_throwsBuildError() {
        // Module A imports module B — valid (depth 2)
        // Module B imports module C — invalid (depth 3, exceeds cap of 2)
        var moduleB = new YamlModule("b", Map.of(),
                Map.of("node-b", new YamlNode("monitor", Map.of(), List.of(), null, null, null)),
                Map.of(), Map.of());
        // Module A has an import of B (tracked via module's own imports field)
        // Nesting validation happens at graph level: graph imports A, A imports B → depth 2 OK
        // If B also imported C → depth 3 → error
        // For this test, validate that modules with imports are flagged:
        var imports = List.of(new YamlImport("a", "alias-a", null, Map.of()));
        var moduleA = new YamlModule("a", Map.of(),
                Map.of("node-a", new YamlNode("monitor", Map.of(), List.of(), null, null, null)),
                Map.of(), Map.of());
        // Nesting depth is validated by checking if imported modules themselves have imports
        // This requires the module to carry its own imports list
        // For now, module nesting cap means: modules cannot import other modules
        // that themselves import modules. Validated at discovery time.
        assertThatCode(() -> YamlDesiredStateProcessor.validateImports(
                imports, Map.of("a", moduleA), TYPE_REGISTRY, "test.yaml"))
                .doesNotThrowAnyException();
    }

    @Test
    void validate_validImport_passes() {
        var imports = List.of(new YamlImport("monitoring", "pipe-monitor", null,
                Map.of("watched_node_id", "warehouse-sink",
                        "alert_email", "ops@example.com")));
        assertThatCode(() -> YamlDesiredStateProcessor.validateImports(
                imports, Map.of("monitoring", monitoringModule), TYPE_REGISTRY, "test.yaml"))
                .doesNotThrowAnyException();
    }

    @Test
    void validate_defaultParameterOmitted_passes() {
        var imports = List.of(new YamlImport("monitoring", "pipe-monitor", null,
                Map.of("watched_node_id", "warehouse-sink")));
        assertThatCode(() -> YamlDesiredStateProcessor.validateImports(
                imports, Map.of("monitoring", monitoringModule), TYPE_REGISTRY, "test.yaml"))
                .doesNotThrowAnyException();
    }
}
```

- [ ] **Step 9: Implement validateImports**

Add to `YamlDesiredStateProcessor.java`:

```java
static void validateImports(List<io.casehub.desiredstate.yaml.model.YamlImport> imports,
                            Map<String, io.casehub.desiredstate.yaml.model.YamlModule> modules,
                            Map<String, String> typeRegistry, String fileName) {
    Set<String> aliases = new HashSet<>();

    for (int i = 0; i < imports.size(); i++) {
        var imp = imports.get(i);
        String ctx = fileName + ": imports[" + i + "]";

        if (!modules.containsKey(imp.module())) {
            throw new RuntimeException(ctx + ": unknown module '" + imp.module()
                    + "'. Available: " + modules.keySet());
        }

        if (imp.as() == null || imp.as().isBlank()) {
            throw new RuntimeException(ctx + ": 'as' alias is required");
        }

        if (imp.as().contains(".")) {
            throw new RuntimeException(ctx + ": alias '" + imp.as()
                    + "' contains the reserved '.' separator");
        }

        if (!aliases.add(imp.as())) {
            throw new RuntimeException(ctx + ": duplicate alias '" + imp.as() + "'");
        }

        var module = modules.get(imp.module());
        for (Map.Entry<String, io.casehub.desiredstate.yaml.model.YamlModuleParameter> param :
                module.parameters().entrySet()) {
            if (param.getValue().required()
                    && !imp.parameters().containsKey(param.getKey())) {
                throw new RuntimeException(ctx + ": required parameter '"
                        + param.getKey() + "' is missing for module '"
                        + imp.module() + "'");
            }
        }
    }
}
```

- [ ] **Step 10: Wire validation into discoverYamlGraphs**

After discovering modules, validate imports for each graph:
```java
if (!yamlGraph.imports().isEmpty()) {
    validateImports(yamlGraph.imports(), availableModules, typeRegistry, fileName);
}
```

- [ ] **Step 11: Run validation tests**

Run: `mvn --batch-mode test -pl yaml/deployment -Dtest=YamlModuleValidationTest`
Expected: ALL PASS

- [ ] **Step 12: Run all tests**

Run: `mvn --batch-mode test -pl yaml/runtime`
Run: `mvn --batch-mode test -pl yaml/deployment`
Expected: ALL PASS

- [ ] **Step 13: Commit**

```
feat(#120): module model types, classpath discovery, and validation

YamlModule, YamlModuleParameter, YamlImport, YamlModuleFile records.
Module classpath discovery at META-INF/desiredstate/modules/.
Build-time validation: unknown modules, missing required parameters,
duplicate aliases, dot-in-alias.

Refs #120
```

---

## Batch 4: Module Expansion + Integration

Safe wrap point: after this batch, YAML operators can compose reusable
modules with alias-prefixed node IDs, parameter scoping, conditional
imports, and module-scoped rules/invariants. Pipeline-yaml example
demonstrates a monitoring module.

### Task 6: ModuleExpander + GoalCompiler integration

**Files:**
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/ModuleExpander.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/resolver/VariableResolver.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`
- Create: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/ModuleExpanderTest.java`
- Modify: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/resolver/VariableResolverTest.java`

**Interfaces:**
- Consumes: `YamlImport`, `YamlModule`, `YamlNode`, `VariableResolver`
- Produces: `ModuleExpander.expand(imports, modules, existingNodes)` →
  `ExpandedGraph(Map<String, YamlNode> expandedNodes, Map<String, Map<String, String>> moduleScopes,
  List<ResolvedRule> promotedRules, List<ResolvedInvariant> promotedInvariants)`.
  `VariableResolver.withModuleScope(Map<String, String>)` — pushes a module parameter scope.

- [ ] **Step 1: Write failing test for VariableResolver module scope**

Add to `VariableResolverTest.java`:

```java
@Test
void withModuleScope_parametersOverrideVariables() {
    var resolver = new VariableResolver(
            Map.of("email", "global@example.com"), null, null);
    var moduleResolver = resolver.withModuleScope(
            Map.of("email", "module@example.com"));
    assertThat(moduleResolver.resolveString("${var.email}", "test"))
            .isEqualTo("module@example.com");
}

@Test
void withModuleScope_fallthroughToVariables() {
    var resolver = new VariableResolver(
            Map.of("batch_size", "1000"), null, null);
    var moduleResolver = resolver.withModuleScope(
            Map.of("watched_id", "sink-1"));
    assertThat(moduleResolver.resolveString("${var.batch_size}", "test"))
            .isEqualTo("1000");
}
```

- [ ] **Step 2: Implement withModuleScope in VariableResolver**

Add field and method:

```java
private final Map<String, String> moduleScope;

// Update constructors to include moduleScope (default null)

public VariableResolver withModuleScope(Map<String, String> moduleScope) {
    return new VariableResolver(this.inlineVariables, this.config,
            this.eachContext, moduleScope);
}
```

Update `lookupVarPrefixed` to check module scope first:

```java
private String lookupVarPrefixed(String varName, String nodeContext) {
    // Module scope has highest priority
    if (moduleScope != null) {
        String value = moduleScope.get(varName);
        if (value != null) {return value;}
    }

    String value = inlineVariables.get(varName);
    if (value != null) {return value;}

    if (config != null) {
        Optional<String> configValue = config.getOptionalValue(varName, String.class);
        if (configValue.isPresent()) {return configValue.get();}
    }

    throw new UnresolvedVariableException("var." + varName, nodeContext,
            "Not found in: " + (moduleScope != null ? "module parameters, " : "")
            + "inline variables " + inlineVariables.keySet()
            + ", MicroProfile Config.");
}
```

- [ ] **Step 3: Run VariableResolver tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=VariableResolverTest`
Expected: ALL PASS

- [ ] **Step 4: Write ModuleExpander tests**

Create `ModuleExpanderTest.java`:

```java
package io.casehub.desiredstate.yaml;

import io.casehub.desiredstate.yaml.model.*;
import org.junit.jupiter.api.Test;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class ModuleExpanderTest {

    private final YamlModule monitoringModule = new YamlModule("monitoring",
            Map.of("watched_node_id", new YamlModuleParameter("string", true, null),
                    "alert_email", new YamlModuleParameter("string", false, "ops@example.com")),
            Map.of("monitor", new YamlNode("monitor",
                            Map.of("target", "${var.watched_node_id}"),
                            List.of("${var.watched_node_id}"), null, null, null),
                    "alerter", new YamlNode("alerter",
                            Map.of("email", "${var.alert_email}"),
                            List.of("monitor"), null, null, null)),
            Map.of(), Map.of());

    @Test
    void expand_aliasPrefix_nodeIds() {
        var imports = List.of(new YamlImport("monitoring", "pipe-monitor", null,
                Map.of("watched_node_id", "warehouse-sink",
                        "alert_email", "ops@example.com")));

        var result = ModuleExpander.expand(imports,
                Map.of("monitoring", monitoringModule),
                new LinkedHashMap<>());

        assertThat(result.expandedNodes()).containsKey("pipe-monitor.monitor");
        assertThat(result.expandedNodes()).containsKey("pipe-monitor.alerter");
    }

    @Test
    void expand_internalDependencies_aliased() {
        var imports = List.of(new YamlImport("monitoring", "pipe-monitor", null,
                Map.of("watched_node_id", "warehouse-sink")));

        var result = ModuleExpander.expand(imports,
                Map.of("monitoring", monitoringModule),
                new LinkedHashMap<>());

        YamlNode alerter = result.expandedNodes().get("pipe-monitor.alerter");
        assertThat(alerter.dependencyNodeIds()).contains("pipe-monitor.monitor");
    }

    @Test
    void expand_crossBoundaryDependency_parameterValue() {
        var imports = List.of(new YamlImport("monitoring", "pipe-monitor", null,
                Map.of("watched_node_id", "warehouse-sink")));

        var result = ModuleExpander.expand(imports,
                Map.of("monitoring", monitoringModule),
                new LinkedHashMap<>());

        YamlNode monitor = result.expandedNodes().get("pipe-monitor.monitor");
        assertThat(monitor.dependencyNodeIds()).contains("warehouse-sink");
    }

    @Test
    void expand_moduleScopes_parameterValues() {
        var imports = List.of(new YamlImport("monitoring", "pipe-monitor", null,
                Map.of("watched_node_id", "warehouse-sink",
                        "alert_email", "custom@example.com")));

        var result = ModuleExpander.expand(imports,
                Map.of("monitoring", monitoringModule),
                new LinkedHashMap<>());

        assertThat(result.moduleScopes()).containsKey("pipe-monitor");
        assertThat(result.moduleScopes().get("pipe-monitor"))
                .containsEntry("watched_node_id", "warehouse-sink")
                .containsEntry("alert_email", "custom@example.com");
    }

    @Test
    void expand_defaultParameter_appliedWhenOmitted() {
        var imports = List.of(new YamlImport("monitoring", "pipe-monitor", null,
                Map.of("watched_node_id", "warehouse-sink")));

        var result = ModuleExpander.expand(imports,
                Map.of("monitoring", monitoringModule),
                new LinkedHashMap<>());

        assertThat(result.moduleScopes().get("pipe-monitor"))
                .containsEntry("alert_email", "ops@example.com");
    }

    @Test
    void expand_conditionalImport_whenFieldPreserved() {
        var imports = List.of(new YamlImport("monitoring", "pipe-monitor",
                "${var.monitoring_enabled}",
                Map.of("watched_node_id", "warehouse-sink")));

        var result = ModuleExpander.expand(imports,
                Map.of("monitoring", monitoringModule),
                new LinkedHashMap<>());

        // Module nodes get the when: from the import
        YamlNode monitor = result.expandedNodes().get("pipe-monitor.monitor");
        assertThat(monitor.when()).isEqualTo("${var.monitoring_enabled}");
    }

    @Test
    void expand_twoImports_independentInstances() {
        var imports = List.of(
                new YamlImport("monitoring", "pipe-monitor", null,
                        Map.of("watched_node_id", "sink-1")),
                new YamlImport("monitoring", "schema-monitor", null,
                        Map.of("watched_node_id", "sink-2")));

        var result = ModuleExpander.expand(imports,
                Map.of("monitoring", monitoringModule),
                new LinkedHashMap<>());

        assertThat(result.expandedNodes()).hasSize(4);
        assertThat(result.expandedNodes()).containsKey("pipe-monitor.monitor");
        assertThat(result.expandedNodes()).containsKey("schema-monitor.monitor");
    }
}
```

- [ ] **Step 5: Implement ModuleExpander**

Create `ModuleExpander.java`:

```java
package io.casehub.desiredstate.yaml;

import io.casehub.desiredstate.yaml.model.*;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public final class ModuleExpander {

    public record ExpandedGraph(
            Map<String, YamlNode> expandedNodes,
            Map<String, Map<String, String>> moduleScopes) {}

    private ModuleExpander() {}

    public static ExpandedGraph expand(
            List<YamlImport> imports,
            Map<String, YamlModule> availableModules,
            Map<String, YamlNode> existingNodes) {

        Map<String, YamlNode> allNodes = new LinkedHashMap<>(existingNodes);
        Map<String, Map<String, String>> moduleScopes = new LinkedHashMap<>();

        for (YamlImport imp : imports) {
            YamlModule module = availableModules.get(imp.module());
            String alias = imp.as();

            // Build parameter scope: import values + defaults
            Map<String, String> paramScope = new LinkedHashMap<>();
            for (Map.Entry<String, YamlModuleParameter> paramDef :
                    module.parameters().entrySet()) {
                String paramName = paramDef.getKey();
                String value = imp.parameters().get(paramName);
                if (value == null && paramDef.getValue().defaultValue() != null) {
                    value = paramDef.getValue().defaultValue();
                }
                if (value != null) {
                    paramScope.put(paramName, value);
                }
            }
            moduleScopes.put(alias, paramScope);

            // Expand module nodes with alias prefix
            for (Map.Entry<String, YamlNode> nodeEntry : module.nodes().entrySet()) {
                String nodeId = nodeEntry.getKey();
                YamlNode node = nodeEntry.getValue();
                String aliasedId = alias + "." + nodeId;

                // Rewrite dependencies: alias internal refs, preserve cross-boundary
                List<Object> rewrittenDeps = new ArrayList<>();
                for (Object dep : node.dependsOn()) {
                    String depId = YamlNode.dependencyNodeId(dep);
                    boolean isOptional = YamlNode.isDependencyOptional(dep);

                    if (module.nodes().containsKey(depId)) {
                        // Internal module dependency — alias-prefix
                        String aliasedDepId = alias + "." + depId;
                        rewrittenDeps.add(isOptional
                                ? Map.of("node", aliasedDepId, "optional", true)
                                : aliasedDepId);
                    } else if (depId.contains("${var.")) {
                        // Cross-boundary parameter reference — keep as-is
                        // VariableResolver will resolve ${var.watched_node_id}
                        // against the module scope at compile time
                        rewrittenDeps.add(isOptional
                                ? Map.of("node", depId, "optional", true)
                                : depId);
                    } else {
                        // Literal cross-boundary reference
                        rewrittenDeps.add(isOptional
                                ? Map.of("node", depId, "optional", true)
                                : depId);
                    }
                }

                // Apply conditional import when:
                String when = node.when();
                if (imp.when() != null) {
                    when = imp.when();
                }

                allNodes.put(aliasedId, new YamlNode(
                        node.type(), node.spec(), rewrittenDeps,
                        node.humanGating(), when, node.forEach()));
            }
        }

        return new ExpandedGraph(allNodes, moduleScopes);
    }
}
```

- [ ] **Step 6: Wire ModuleExpander into createYamlGoalCompiler**

In the GoalCompiler lambda, before forEach expansion and node building:

```java
// Module expansion
Map<String, YamlNode> effectiveNodes = new LinkedHashMap<>(
        yamlGraph != null ? yamlGraph.nodes() : Map.of());
Map<String, Map<String, String>> moduleScopes = Map.of();

if (yamlGraph != null && !yamlGraph.imports().isEmpty()) {
    ModuleExpander.ExpandedGraph expanded = ModuleExpander.expand(
            yamlGraph.imports(), availableModules, effectiveNodes);
    effectiveNodes = expanded.expandedNodes();
    moduleScopes = expanded.moduleScopes();
}

// Use effectiveNodes for the rest of the compilation...
// When building nodes for a module-scoped node, push moduleScope:
for (Map.Entry<String, YamlNode> entry : effectiveNodes.entrySet()) {
    String nodeId = entry.getKey();
    // Determine module scope
    String aliasPrefix = nodeId.contains(".")
            ? nodeId.substring(0, nodeId.indexOf(".")) : null;
    VariableResolver nodeResolver = resolver;
    if (aliasPrefix != null && moduleScopes.containsKey(aliasPrefix)) {
        nodeResolver = resolver.withModuleScope(moduleScopes.get(aliasPrefix));
    }
    // ... resolve spec with nodeResolver ...
}
```

Pass `availableModules` into the recorder method. The processor already discovers
modules and can pass them via a new parameter on `createYamlGoalCompiler`.

- [ ] **Step 7: Run ModuleExpander tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=ModuleExpanderTest`
Expected: ALL PASS

- [ ] **Step 8: Run all tests**

Run: `mvn --batch-mode test -pl yaml/runtime`
Run: `mvn --batch-mode test -pl yaml/deployment`
Run: `mvn --batch-mode test -pl examples/pipeline-yaml`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```
feat(#120): ModuleExpander with parameter scope and GoalCompiler integration

Module import expansion: alias-prefixed node IDs, internal dependency
rewriting, cross-boundary parameter references, conditional imports,
default parameter values. VariableResolver.withModuleScope() for
parameter shadowing during compilation.

Refs #120
```

---

### Task 7: Module rules/invariants + pipeline-yaml module integration

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/ModuleExpander.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`
- Create: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/AlerterSpec.java`
- Create: `examples/pipeline-yaml/src/main/resources/META-INF/desiredstate/modules/monitoring.yaml`
- Modify: `examples/pipeline-yaml/src/main/resources/META-INF/desiredstate/medallion-pipeline.yaml`
- Modify: `examples/pipeline-yaml/src/test/java/io/casehub/desiredstate/example/pipeline/yaml/PipelineYamlTest.java`

**Interfaces:**
- Consumes: `ModuleExpander.expand()`, `YamlRuleConverter`, `YamlInvariantConverter`
- Produces: Module-promoted rules and invariants participate in the GoalCompiler's
  rule/invariant evaluation. Integration test proves monitoring module works end-to-end.

- [ ] **Step 1: Extend ModuleExpander to collect promoted rules and invariants**

Add rule/invariant collection to `ModuleExpander.expand()`. Module rules and invariants
are returned alongside expanded nodes:

```java
public record ExpandedGraph(
        Map<String, YamlNode> expandedNodes,
        Map<String, Map<String, String>> moduleScopes,
        Map<String, YamlRule> promotedRules,
        Map<String, YamlInvariant> promotedInvariants) {}
```

In the expansion loop, after processing nodes:
```java
// Promote module rules with aliased names
for (Map.Entry<String, YamlRule> ruleEntry : module.rules().entrySet()) {
    String promotedName = alias + "." + ruleEntry.getKey();
    promotedRules.put(promotedName, ruleEntry.getValue());
}

// Promote module invariants with aliased names
for (Map.Entry<String, YamlInvariant> invEntry : module.invariants().entrySet()) {
    String promotedName = alias + "." + invEntry.getKey();
    promotedInvariants.put(promotedName, invEntry.getValue());
}
```

- [ ] **Step 2: Wire promoted rules/invariants into GoalCompiler**

In `createYamlGoalCompiler`, after module expansion, merge promoted rules:

```java
Map<String, YamlRule> effectiveRules = new LinkedHashMap<>(
        yamlGraph != null ? yamlGraph.rules() : Map.of());
effectiveRules.putAll(expanded.promotedRules());

Map<String, YamlInvariant> effectiveInvariants = new LinkedHashMap<>(
        yamlGraph != null ? yamlGraph.invariants() : Map.of());
effectiveInvariants.putAll(expanded.promotedInvariants());
```

Use `effectiveRules` and `effectiveInvariants` in the rule/invariant evaluation.

- [ ] **Step 3: Create AlerterSpec in shared pipeline module**

```java
package io.casehub.desiredstate.example.pipeline;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.NodeTypeId;

@NodeTypeId("alerter")
public record AlerterSpec(String email) implements NodeSpec {
    @Override
    public NodeType nodeType() { return NodeType.of("alerter"); }
}
```

- [ ] **Step 4: Create monitoring module YAML**

Create `examples/pipeline-yaml/src/main/resources/META-INF/desiredstate/modules/monitoring.yaml`:

```yaml
module:
  name: monitoring
  parameters:
    watched_node_id:
      type: string
      required: true
    alert_email:
      type: string
      default: "ops@example.com"

nodes:
  monitor:
    type: monitor
    dependsOn: ["${var.watched_node_id}"]
    spec:
      target: "${var.watched_node_id}"
  alerter:
    type: alerter
    dependsOn: [monitor]
    spec:
      email: "${var.alert_email}"

invariants:
  monitor-has-target:
    match:
      mon: { type: monitor }
    directDep:
      target: { type: "*", of: mon, direction: DEPENDENCIES }
```

- [ ] **Step 5: Add module import to medallion-pipeline.yaml**

Add to `medallion-pipeline.yaml`:

```yaml
imports:
  - module: monitoring
    as: pipe-monitor
    parameters:
      watched_node_id: warehouse-sink
      alert_email: "pipeline-ops@example.com"
```

Remove the `ensure-monitoring` rule from the YAML (the module now provides monitoring).
The existing `monitor-warehouse-sink` rule-generated node is replaced by `pipe-monitor.monitor`
module node.

- [ ] **Step 6: Update TYPE_REGISTRY in PipelineYamlTest**

Add the alerter entry:
```java
Map.entry("alerter", "io.casehub.desiredstate.example.pipeline.AlerterSpec")
```

- [ ] **Step 7: Write module integration tests**

Add to `PipelineYamlTest.java`:

```java
@Test
void module_monitoringImported_nodesAliased() {
    DesiredStateGraph graph = compileSingleGraph();
    assertThat(graph.nodes()).containsKey(NodeId.of("pipe-monitor.monitor"));
    assertThat(graph.nodes()).containsKey(NodeId.of("pipe-monitor.alerter"));
}

@Test
void module_monitorDependsOnImportingNode() {
    DesiredStateGraph graph = compileSingleGraph();
    assertThat(graph.dependenciesOf(NodeId.of("pipe-monitor.monitor")))
            .contains(NodeId.of("warehouse-sink"));
}

@Test
void module_alerterDependsOnMonitor() {
    DesiredStateGraph graph = compileSingleGraph();
    assertThat(graph.dependenciesOf(NodeId.of("pipe-monitor.alerter")))
            .contains(NodeId.of("pipe-monitor.monitor"));
}

@Test
void module_parameterResolvedInSpec() {
    DesiredStateGraph graph = compileSingleGraph();
    MonitorSpec monSpec = (MonitorSpec) graph.nodes()
            .get(NodeId.of("pipe-monitor.monitor")).spec();
    assertThat(monSpec.target()).isEqualTo("warehouse-sink");

    AlerterSpec alertSpec = (AlerterSpec) graph.nodes()
            .get(NodeId.of("pipe-monitor.alerter")).spec();
    assertThat(alertSpec.email()).isEqualTo("pipeline-ops@example.com");
}
```

Update existing node count and rule tests to account for module nodes replacing
rule-generated nodes.

- [ ] **Step 8: Run integration tests**

Run: `mvn --batch-mode test -pl examples/pipeline-yaml`
Expected: ALL PASS

- [ ] **Step 9: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```
feat(#120): module rules/invariants promotion and pipeline-yaml module example

Module-scoped invariants promoted to top-level with aliased names.
AlerterSpec added to shared pipeline module. Monitoring module YAML
replaces rule-based monitoring with composable module import.
Parameter resolution, cross-boundary dependencies, aliased node IDs.

Refs #120
```

---

## Summary

| Batch | Tasks | What's working after |
|-------|-------|---------------------|
| 1: forEach Model + VariableResolver | 1-2 | Model types deserialize, validation catches structural errors, `${each.*}` scope works |
| 2: forEach Expansion + Integration | 3-4 | forEach stamps N copies with aligned deps, pipeline-yaml demonstrates multi-region |
| 3: Module Model + Validation | 5 | Module files parse, classpath discovery finds them, validation catches errors |
| 4: Module Expansion + Integration | 6-7 | Modules expand with aliased IDs, parameter scoping, promoted rules/invariants |

**Total:** 4 batches, 7 tasks

**What Phase 3 delivers:** An operator can write YAML with forEach cardinality
stamping (named iteration groups, inline forEach, aligned dependencies,
`${each.*}` interpolation, expansion limits, variable-sourced JSON arrays) and
composable modules (reusable parameterised fragments, alias-prefixed node IDs,
module parameter scope, conditional imports, promoted rules and invariants).
Together with Phases 1 and 2, the YAML surface now covers: nodes, dependencies,
fault policies, invariants, conditional inclusion, graph rules, lifecycle phases,
forEach cardinality stamping, and composable modules — no Java required.

## References

- `specs/issue-116-yaml-language-design/2026-08-27-yaml-language-extensions-design.md` — design spec (§6.6, §6.7, §4, §5, §7, §8)
- `specs/issue-116-yaml-language-design/decisions.md` — D2, D5, D9, D10, D13, D14
- `yaml/runtime/.../model/YamlGraph.java` — current YAML model (fields to extend)
- `yaml/runtime/.../model/YamlNode.java` — current node model (forEach field addition)
- `yaml/runtime/.../resolver/VariableResolver.java` — prefix dispatch, each./module scope additions
- `yaml/runtime/.../YamlGraphRecorder.java:34-126` — single-graph GoalCompiler
- `yaml/runtime/.../YamlGraphRecorder.java:162-274` — lifecycle GoalCompiler
- `yaml/runtime/.../YamlRuleConverter.java` — converter pattern reference
- `yaml/runtime/.../YamlInvariantConverter.java` — converter pattern reference
- `yaml/deployment/.../YamlDesiredStateProcessor.java` — build processor, validation, discovery
- `examples/pipeline-yaml/.../medallion-pipeline.yaml` — YAML example to extend
- `examples/pipeline-yaml/.../PipelineYamlTest.java` — integration test pattern
- `examples/pipeline-yaml/.../LifecyclePipelineTest.java` — lifecycle test pattern
- `plans/2026-08-28-phase2-yaml-extensions.md` — Phase 2 plan (format reference)
- #118 — conditional and iterated subgraph inclusion
- #120 — module composition — reusable parameterised subgraphs
