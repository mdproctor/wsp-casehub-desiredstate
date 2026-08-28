# Phase 2: YAML Language Extensions — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #116 — operator-first declaration language
**Issue group:** #116

**Goal:** Deliver Phase 2 of the YAML language extensions — declarative
graph rules (structural rewriting) and lifecycle phases (build-then-operate)
— making YAML the full operator surface for CaseHub desired-state.

**Architecture:** Graph rules reuse the sealed interface hierarchy and
`PatternEvaluator` from Phase 1. A new `actionEvaluator` function on
`DeclarativeRule` bridges pattern bindings to `GraphMutation` instances
— the engine stays YAML-agnostic; the YAML layer captures
`NodeSpecRegistry` and `ObjectMapper` in the closure. Lifecycle phases
compile sequentially with carry-forward injection: each phase's output
nodes are injected into the next phase's input so `dependsOn` references
resolve across phases.

**Tech Stack:** Java 21, Quarkus 3.x, Jackson YAML, Jandex, Maven

**Design spec:** `specs/issue-116-yaml-language-design/2026-08-27-yaml-language-extensions-design.md`

## Global Constraints

- All `${}` references use explicit prefixes: `${var.}`, `${match.}`, `${fault.}`, `${each.}`
- User-declared node IDs must not contain `.` (reserved separator)
- Rule-generated node IDs must not contain `.` — validated at rule evaluation time
- `${match.*}` supports `.id`, `.type`, `.flatId` accessors only — no `.spec.<field>` access (D3)
- `NodeSpecRegistry` types resolved via `@NodeTypeId` Jandex scan
- Jackson `ObjectMapper` for spec deserialization uses dedicated coercion-enabled instance
- Build-time validation produces errors, not warnings, for safety violations
- All YAML model records use compact constructors defaulting nulls to empty collections
- Test scope: unit tests for each component, integration test via pipeline-yaml example
- Cross-surface rule resolution (§8.4) is deferred — standalone annotation rules do not fire against YAML graphs in this phase
- Graph scoping for YAML rules (`graph:` field / `GraphPatternMatcher`) is deferred — YAML rules fire against their enclosing graph only in this phase; multi-graph scoping requires cross-surface infrastructure
- Last-phase `allPresent` completionCondition produces a build-time warning (lifecycle will terminate)

---

## Batch 1: Declarative Rule Engine Support

Safe wrap point: after this batch, `GraphRuleEngine` can evaluate
`DeclarativeRule` instances with pattern matching and action evaluation.
All existing annotation tests pass unchanged.

### Task 1: DeclarativeRule actionEvaluator + evaluateDeclarative + MatchTemplateResolver

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/ResolvedRule.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngine.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/MatchTemplateResolver.java`
- Modify: `annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngineTest.java`
- Create: `annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/MatchTemplateResolverTest.java`

**Interfaces:**
- Consumes: `PatternEvaluator.evaluate()`, `PatternParameterDescriptor`, `ResolvedRule` sealed interface
- Produces: `DeclarativeRule(name, patterns, bindingNames, actionEvaluator)` — the `actionEvaluator`
  is `Function<Map<String, DesiredNode>, List<GraphMutation>>`, called once per binding map.
  `MatchTemplateResolver.resolve(template, bindings)` — resolves `${match.binding.id}`,
  `${match.binding.type}`, `${match.binding.flatId}` in template strings. Returns the resolved
  string. Throws `IllegalArgumentException` if the resolved value contains `.` and the caller
  signals this is a node ID context (via `resolveNodeId` variant).

- [ ] **Step 1: Write failing tests for DeclarativeRule evaluation**

Add to `GraphRuleEngineTest.java`:

```java
@Test
void declarativeRuleAddsNodeViaActionEvaluator() {
    var graph = factory.of(List.of(
            new DesiredNode(NodeId.of("sink-1"), new Spec("sink-1", "sink"), HumanGating.NONE)),
            List.of());

    Function<Map<String, DesiredNode>, List<GraphMutation>> evaluator = bindings -> {
        DesiredNode sink = bindings.get("sink");
        DesiredNode monitor = new DesiredNode(
                NodeId.of("monitor-" + sink.id().value()),
                new Spec("monitor-" + sink.id().value(), "monitor"), HumanGating.NONE);
        return List.of(
                new GraphMutation.AddNode(monitor),
                new GraphMutation.AddDependency(new Dependency(monitor.id(), sink.id())));
    };

    var rule = new ResolvedRule.DeclarativeRule("ensure-monitoring",
            List.of(
                new PatternParameterDescriptor(PatternKind.MATCH, "sink", "", Direction.DEPENDENCIES),
                new PatternParameterDescriptor(PatternKind.NOT_EXISTS, "monitor", "sink", Direction.DEPENDENTS)),
            new String[]{"sink", "guard"},
            evaluator);

    var result = engine.evaluate(graph, List.of(rule));
    assertThat(result.nodes()).hasSize(2);
    assertThat(result.nodes()).containsKey(NodeId.of("monitor-sink-1"));
    assertThat(result.dependencies()).contains(
            new Dependency(NodeId.of("monitor-sink-1"), NodeId.of("sink-1")));
}

@Test
void declarativeRuleConvergesWithGuard() {
    var graph = factory.of(List.of(
            new DesiredNode(NodeId.of("sink-1"), new Spec("sink-1", "sink"), HumanGating.NONE),
            new DesiredNode(NodeId.of("sink-2"), new Spec("sink-2", "sink"), HumanGating.NONE)),
            List.of());

    Function<Map<String, DesiredNode>, List<GraphMutation>> evaluator = bindings -> {
        DesiredNode sink = bindings.get("sink");
        DesiredNode monitor = new DesiredNode(
                NodeId.of("monitor-" + sink.id().value()),
                new Spec("monitor", "monitor"), HumanGating.NONE);
        return List.of(
                new GraphMutation.AddNode(monitor),
                new GraphMutation.AddDependency(new Dependency(monitor.id(), sink.id())));
    };

    var rule = new ResolvedRule.DeclarativeRule("ensure-monitoring",
            List.of(
                new PatternParameterDescriptor(PatternKind.MATCH, "sink", "", Direction.DEPENDENCIES),
                new PatternParameterDescriptor(PatternKind.NOT_EXISTS, "monitor", "sink", Direction.DEPENDENTS)),
            new String[]{"sink", "guard"},
            evaluator);

    var result = engine.evaluate(graph, List.of(rule));
    assertThat(result.nodes()).hasSize(4);
    assertThat(result.nodes()).containsKey(NodeId.of("monitor-sink-1"));
    assertThat(result.nodes()).containsKey(NodeId.of("monitor-sink-2"));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=GraphRuleEngineTest`
Expected: compilation error — `DeclarativeRule` constructor doesn't accept 4 args.

- [ ] **Step 3: Extend DeclarativeRule with actionEvaluator**

Modify `ResolvedRule.java` — add `Function` field to `DeclarativeRule`:

```java
record DeclarativeRule(String name, List<PatternParameterDescriptor> patterns,
                       String[] bindingNames,
                       java.util.function.Function<java.util.Map<String, io.casehub.desiredstate.api.DesiredNode>,
                               java.util.List<io.casehub.desiredstate.api.GraphMutation>> actionEvaluator)
        implements ResolvedRule {
}
```

Add the `java.util.function.Function` import. The `actionEvaluator` takes a binding
map (`Map<String, DesiredNode>`) and returns the mutations for that binding.

- [ ] **Step 4: Implement evaluateDeclarative in GraphRuleEngine**

Replace the `List.of()` placeholder at line 64:

```java
case ResolvedRule.DeclarativeRule decl -> evaluateDeclarative(decl, graph);
```

Add the method:

```java
private List<GraphMutation> evaluateDeclarative(ResolvedRule.DeclarativeRule rule,
                                                 DesiredStateGraph graph) {
    List<GraphMutation> allMutations = new ArrayList<>();
    List<Map<String, DesiredNode>> allBindings =
            PatternEvaluator.evaluate(graph, rule.patterns(), rule.bindingNames());
    for (Map<String, DesiredNode> binding : allBindings) {
        List<GraphMutation> mutations = rule.actionEvaluator().apply(binding);
        if (mutations != null && !mutations.isEmpty()) {
            allMutations.addAll(mutations);
        }
    }
    return allMutations;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=GraphRuleEngineTest`
Expected: ALL PASS

- [ ] **Step 6: Write MatchTemplateResolver tests**

Create `MatchTemplateResolverTest.java`:

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class MatchTemplateResolverTest {

    record Spec(String name, String typeValue) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of(typeValue); }
    }

    private final Map<String, DesiredNode> bindings = Map.of(
            "sink", new DesiredNode(NodeId.of("warehouse-sink"),
                    new Spec("ws", "sink"), HumanGating.NONE),
            "src", new DesiredNode(NodeId.of("pipe-monitor.monitor"),
                    new Spec("mon", "monitor"), HumanGating.NONE));

    @Test
    void resolveId() {
        assertThat(MatchTemplateResolver.resolve("monitor-${match.sink.id}", bindings))
                .isEqualTo("monitor-warehouse-sink");
    }

    @Test
    void resolveType() {
        assertThat(MatchTemplateResolver.resolve("type-is-${match.sink.type}", bindings))
                .isEqualTo("type-is-sink");
    }

    @Test
    void resolveFlatId() {
        assertThat(MatchTemplateResolver.resolve("health-${match.src.flatId}", bindings))
                .isEqualTo("health-pipe-monitor-monitor");
    }

    @Test
    void resolveMultipleBindings() {
        assertThat(MatchTemplateResolver.resolve(
                "${match.sink.id}-to-${match.src.type}", bindings))
                .isEqualTo("warehouse-sink-to-monitor");
    }

    @Test
    void noTemplatesPassesThrough() {
        assertThat(MatchTemplateResolver.resolve("literal-id", bindings))
                .isEqualTo("literal-id");
    }

    @Test
    void resolveNodeId_rejectsDotInResult() {
        assertThatThrownBy(() -> MatchTemplateResolver.resolveNodeId(
                "health-${match.src.id}", bindings, "add-health-check"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining(".")
                .hasMessageContaining("add-health-check");
    }

    @Test
    void resolveNodeId_acceptsFlatId() {
        assertThat(MatchTemplateResolver.resolveNodeId(
                "health-${match.src.flatId}", bindings, "add-health-check"))
                .isEqualTo("health-pipe-monitor-monitor");
    }
}
```

- [ ] **Step 7: Create MatchTemplateResolver**

Create `MatchTemplateResolver.java`:

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.api.DesiredNode;

import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public final class MatchTemplateResolver {

    private static final Pattern MATCH_PATTERN =
            Pattern.compile("\\$\\{match\\.([^.]+)\\.(id|type|flatId)}");

    private MatchTemplateResolver() {}

    public static String resolve(String template, Map<String, DesiredNode> bindings) {
        Matcher matcher = MATCH_PATTERN.matcher(template);
        StringBuilder sb = new StringBuilder();
        while (matcher.find()) {
            String bindingName = matcher.group(1);
            String accessor = matcher.group(2);
            DesiredNode node = bindings.get(bindingName);
            if (node == null) {
                throw new IllegalArgumentException(
                        "Unknown binding '" + bindingName + "' in template '" + template
                        + "'. Available: " + bindings.keySet());
            }
            String value = switch (accessor) {
                case "id" -> node.id().value();
                case "type" -> node.type().value();
                case "flatId" -> node.id().value().replace('.', '-');
                default -> throw new IllegalStateException("Unexpected accessor: " + accessor);
            };
            matcher.appendReplacement(sb, Matcher.quoteReplacement(value));
        }
        matcher.appendTail(sb);
        return sb.toString();
    }

    public static String resolveNodeId(String template, Map<String, DesiredNode> bindings,
                                        String ruleName) {
        String resolved = resolve(template, bindings);
        if (resolved.contains(".")) {
            throw new IllegalArgumentException(
                    "Rule '" + ruleName + "' produces node ID '" + resolved
                    + "' which contains the reserved '.' separator. "
                    + "Rule-generated node IDs must not contain '.'. "
                    + "Use ${match.*.flatId} to replace '.' with '-'.");
        }
        return resolved;
    }
}
```

- [ ] **Step 8: Run MatchTemplateResolver tests**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=MatchTemplateResolverTest`
Expected: ALL PASS

- [ ] **Step 9: Run ALL existing tests to verify no regressions**

Run: `mvn --batch-mode test -pl annotations/runtime`
Run: `mvn --batch-mode test -pl examples/pipeline-annotated`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```
feat(#116): DeclarativeRule actionEvaluator and MatchTemplateResolver

GraphRuleEngine evaluates DeclarativeRule via actionEvaluator function.
MatchTemplateResolver resolves ${match.*.id}, ${match.*.type}, and
${match.*.flatId} with node ID validation for rule-generated IDs.

Refs #116
```

---

## Batch 2: YAML Declarative Rules

Safe wrap point: after this batch, YAML operators can declare structural
graph rules with pattern matching and action templates. The pipeline-yaml
example demonstrates the feature.

### Task 2: YAML rule model types + VariableResolver template mode

**Files:**
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlRule.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlGraph.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/resolver/VariableResolver.java`
- Create: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/model/YamlRuleDeserializationTest.java`
- Modify: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/resolver/VariableResolverTest.java`

**Interfaces:**
- Consumes: `YamlPattern` (reused from invariants), Jackson YAML deserialization
- Produces: `YamlRule` record with `graph`, `match`, `directDep`, `reaches`, `notExists`,
  `actions` (`List<Map<String, Object>>`). `YamlGraph` gains `rules` field (`Map<String, YamlRule>`).
  `VariableResolver.resolveTemplateString(template, context)` — resolves `${var.*}` and
  passes through `${match.*}` untouched. Used for rule action templates.

- [ ] **Step 1: Create YamlRule record**

```java
package io.casehub.desiredstate.yaml.model;

import java.util.List;
import java.util.Map;

public record YamlRule(
        List<String> graph,
        Map<String, YamlPattern> match,
        Map<String, YamlPattern> directDep,
        Map<String, YamlPattern> reaches,
        Map<String, YamlPattern> notExists,
        List<Map<String, Object>> actions) {

    public YamlRule {
        graph = graph != null ? graph : List.of();
        match = match != null ? match : Map.of();
        directDep = directDep != null ? directDep : Map.of();
        reaches = reaches != null ? reaches : Map.of();
        notExists = notExists != null ? notExists : Map.of();
        actions = actions != null ? actions : List.of();
    }
}
```

- [ ] **Step 2: Add `rules` field to YamlGraph**

Add `Map<String, YamlRule> rules` to `YamlGraph`. Default to `Map.of()` in
compact constructor. The constructor signature becomes:

```java
public record YamlGraph(
        YamlDesiredState desiredState,
        Map<String, String> variables,
        Map<String, YamlNode> nodes,
        List<YamlFaultPolicy> faultPolicy,
        Map<String, YamlInvariant> invariants,
        Map<String, YamlRule> rules) {
    public YamlGraph {
        // existing defaults...
        rules = rules != null ? rules : Map.of();
    }
}
```

- [ ] **Step 3: Write deserialization test**

Create `YamlRuleDeserializationTest.java`:

```java
package io.casehub.desiredstate.yaml.model;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.desiredstate.annotations.runtime.Direction;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class YamlRuleDeserializationTest {

    private final ObjectMapper mapper = new ObjectMapper(new YAMLFactory());

    @Test
    @SuppressWarnings("unchecked")
    void deserialize_ruleWithAddNodeAndAddDependency() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: rule-test
                nodes: {}
                rules:
                  ensure-monitoring:
                    match:
                      sink: { type: sink }
                    notExists:
                      guard: { type: monitor, of: sink, direction: DEPENDENTS }
                    actions:
                      - addNode:
                          id: "monitor-${match.sink.id}"
                          type: monitor
                          spec:
                            target: "${match.sink.id}"
                      - addDependency:
                          from: "monitor-${match.sink.id}"
                          to: "${match.sink.id}"
                """;
        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);
        assertThat(graph.rules()).hasSize(1);

        YamlRule rule = graph.rules().get("ensure-monitoring");
        assertThat(rule.match()).hasSize(1);
        assertThat(rule.match().get("sink").type()).isEqualTo("sink");
        assertThat(rule.notExists()).hasSize(1);
        assertThat(rule.notExists().get("guard").of()).isEqualTo("sink");
        assertThat(rule.notExists().get("guard").direction()).isEqualTo(Direction.DEPENDENTS);
        assertThat(rule.actions()).hasSize(2);

        java.util.Map<String, Object> addNodeAction = rule.actions().get(0);
        assertThat(addNodeAction).containsKey("addNode");
        java.util.Map<String, Object> addNodeParams =
                (java.util.Map<String, Object>) addNodeAction.get("addNode");
        assertThat(addNodeParams.get("id")).isEqualTo("monitor-${match.sink.id}");
        assertThat(addNodeParams.get("type")).isEqualTo("monitor");
    }

    @Test
    void deserialize_ruleWithGraphScope() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: scope-test
                nodes: {}
                rules:
                  scoped-rule:
                    graph: ["pipeline:*"]
                    match:
                      node: { type: sink }
                    actions:
                      - removeNode:
                          id: "${match.node.id}"
                """;
        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);
        YamlRule rule = graph.rules().get("scoped-rule");
        assertThat(rule.graph()).containsExactly("pipeline:*");
    }

    @Test
    void deserialize_emptyRulesDefaultsToEmptyMap() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: no-rules
                nodes: {}
                """;
        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);
        assertThat(graph.rules()).isEmpty();
    }
}
```

- [ ] **Step 4: Run deserialization tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlRuleDeserializationTest`
Expected: ALL PASS

- [ ] **Step 5: Write failing test for VariableResolver template mode**

Add to `VariableResolverTest.java`:

```java
@Test
void resolveTemplateString_resolvesVarPrefix_passesMatchThrough() {
    var resolver = new VariableResolver(Map.of("region", "us-east"), null, null);
    String result = resolver.resolveTemplateString(
            "monitor-${match.sink.id}-${var.region}", "test-rule");
    assertThat(result).isEqualTo("monitor-${match.sink.id}-us-east");
}

@Test
void resolveTemplateString_noVarReferences_passesAllThrough() {
    var resolver = new VariableResolver(Map.of(), null, null);
    String result = resolver.resolveTemplateString(
            "${match.sink.id}", "test-rule");
    assertThat(result).isEqualTo("${match.sink.id}");
}
```

- [ ] **Step 6: Run template mode test to verify it fails**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=VariableResolverTest`
Expected: compilation error — `resolveTemplateString` method doesn't exist.

- [ ] **Step 7: Implement resolveTemplateString in VariableResolver**

Add to `VariableResolver.java`:

```java
public String resolveTemplateString(String template, String nodeContext) {
    Matcher matcher = VAR_PATTERN.matcher(template);
    StringBuilder sb = new StringBuilder();
    while (matcher.find()) {
        String key = matcher.group(1);
        if (key.startsWith("var.")) {
            String resolved = lookupVarPrefixed(key.substring(4), nodeContext);
            matcher.appendReplacement(sb, Matcher.quoteReplacement(resolved));
        } else {
            matcher.appendReplacement(sb, Matcher.quoteReplacement(matcher.group()));
        }
    }
    matcher.appendTail(sb);
    return sb.toString();
}

public Map<String, Object> resolveTemplateMap(Map<?, ?> input, String nodeContext) {
    Map<String, Object> result = new LinkedHashMap<>();
    for (Map.Entry<?, ?> entry : input.entrySet()) {
        String key = entry.getKey().toString();
        Object val = entry.getValue();
        if (val instanceof String s && s.contains("${")) {
            result.put(key, resolveTemplateString(s, nodeContext));
        } else if (val instanceof Map<?, ?> nested) {
            result.put(key, resolveTemplateMap(nested, nodeContext));
        } else if (val instanceof List<?> list) {
            result.put(key, resolveTemplateList(list, nodeContext));
        } else {
            result.put(key, val);
        }
    }
    return result;
}

public List<?> resolveTemplateList(List<?> input, String nodeContext) {
    return input.stream()
            .map(item -> {
                if (item instanceof String s && s.contains("${")) {
                    return resolveTemplateString(s, nodeContext);
                }
                return item;
            })
            .toList();
}
```

- [ ] **Step 8: Run template mode tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=VariableResolverTest`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```
feat(#116): YAML rule model types and VariableResolver template mode

YamlRule record with pattern vocabulary and action list. YamlGraph
gains rules field. VariableResolver.resolveTemplateString() resolves
${var.*} and passes through ${match.*} for later resolution.

Refs #116
```

### Task 3: Build-time validation + YamlRuleConverter + GoalCompiler rule wrapping

**Files:**
- Modify: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlRuleConverter.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`
- Create: `yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/YamlRuleValidationTest.java`
- Create: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlRuleConverterTest.java`

**Interfaces:**
- Consumes: `YamlRule`, `YamlPattern`, `VariableResolver.resolveTemplateString()`,
  `MatchTemplateResolver.resolve()`, `NodeSpecRegistry`, `ObjectMapper`
- Produces: `YamlRuleConverter.toDeclarativeRule(name, yamlRule, resolver, registry)` →
  `ResolvedRule.DeclarativeRule` with action evaluator closure (ObjectMapper created internally
  via `createCoercionMapper()`).
  `YamlGraphRecorder.createYamlGoalCompiler()` applies rules when `yamlGraph.rules()` is non-empty.

- [ ] **Step 1: Write validation tests**

Create `YamlRuleValidationTest.java`:

```java
package io.casehub.desiredstate.yaml.deployment;

import io.casehub.desiredstate.annotations.runtime.Direction;
import io.casehub.desiredstate.yaml.model.YamlPattern;
import io.casehub.desiredstate.yaml.model.YamlRule;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

class YamlRuleValidationTest {

    private static final Map<String, String> TYPE_REGISTRY = Map.of(
            "sink", "com.example.SinkSpec",
            "monitor", "com.example.MonitorSpec");

    @Test
    void validate_emptyMatch_throwsBuildError() {
        YamlRule rule = new YamlRule(
                List.of(), Map.of(), Map.of(), Map.of(), Map.of(),
                List.of(Map.of("removeNode", Map.of("id", "x"))));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateRule(
                "empty-match", rule, TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("match");
    }

    @Test
    void validate_emptyActions_throwsBuildError() {
        YamlRule rule = new YamlRule(
                List.of(),
                Map.of("sink", new YamlPattern("sink", null, Direction.DEPENDENCIES)),
                Map.of(), Map.of(), Map.of(),
                List.of());
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateRule(
                "empty-actions", rule, TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("actions");
    }

    @Test
    void validate_unknownActionType_throwsBuildError() {
        YamlRule rule = new YamlRule(
                List.of(),
                Map.of("sink", new YamlPattern("sink", null, Direction.DEPENDENCIES)),
                Map.of(), Map.of(), Map.of(),
                List.of(Map.of("destroyNode", Map.of("id", "x"))));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateRule(
                "bad-action", rule, TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("destroyNode")
                .hasMessageContaining("addNode");
    }

    @Test
    void validate_addNodeUnknownType_throwsBuildError() {
        YamlRule rule = new YamlRule(
                List.of(),
                Map.of("sink", new YamlPattern("sink", null, Direction.DEPENDENCIES)),
                Map.of(), Map.of(), Map.of(),
                List.of(Map.of("addNode", Map.of(
                        "id", "x", "type", "nonexistent", "spec", Map.of()))));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateRule(
                "bad-type", rule, TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("nonexistent");
    }

    @Test
    void validate_addNodeMissingType_throwsBuildError() {
        YamlRule rule = new YamlRule(
                List.of(),
                Map.of("sink", new YamlPattern("sink", null, Direction.DEPENDENCIES)),
                Map.of(), Map.of(), Map.of(),
                List.of(Map.of("addNode", Map.of("id", "x", "spec", Map.of()))));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateRule(
                "no-type", rule, TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("type");
    }

    @Test
    void validate_ofReferencesUnknownBinding_throwsBuildError() {
        YamlRule rule = new YamlRule(
                List.of(),
                Map.of("sink", new YamlPattern("sink", null, Direction.DEPENDENCIES)),
                Map.of("dep", new YamlPattern("monitor", "nonexistent", Direction.DEPENDENCIES)),
                Map.of(), Map.of(),
                List.of(Map.of("removeNode", Map.of("id", "x"))));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateRule(
                "bad-of", rule, TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("nonexistent");
    }
}
```

- [ ] **Step 2: Implement validateRule in YamlDesiredStateProcessor**

Add a `static` method (so the test can call it directly):

```java
static void validateRule(String ruleName, YamlRule rule,
                         Map<String, String> typeRegistry, String fileName) {
    String ctx = fileName + ": rules." + ruleName;

    if (rule.match().isEmpty()) {
        throw new RuntimeException(ctx + ": at least one 'match' binding is required");
    }

    if (rule.actions().isEmpty()) {
        throw new RuntimeException(ctx + ": at least one action is required");
    }

    Set<String> validActions = Set.of(
            "addNode", "removeNode", "updateNode", "addDependency", "removeDependency");

    Set<String> allBindings = new java.util.LinkedHashSet<>();
    for (String binding : rule.match().keySet()) {
        allBindings.add(binding);
        validatePatternType(rule.match().get(binding).type(), typeRegistry,
                ctx + ".match." + binding);
    }

    validatePatternSection(rule.directDep(), "directDep", allBindings, typeRegistry, ctx, true);
    validatePatternSection(rule.reaches(), "reaches", allBindings, typeRegistry, ctx, true);
    validatePatternSection(rule.notExists(), "notExists", allBindings, typeRegistry, ctx, false);

    for (int i = 0; i < rule.actions().size(); i++) {
        Map<String, Object> action = rule.actions().get(i);
        if (action.size() != 1) {
            throw new RuntimeException(ctx + ".actions[" + i
                    + "]: each action must have exactly one key (the action type)");
        }
        String actionType = action.keySet().iterator().next();
        if (!validActions.contains(actionType)) {
            throw new RuntimeException(ctx + ".actions[" + i + "]: unknown action type '"
                    + actionType + "'. Valid: " + validActions);
        }

        @SuppressWarnings("unchecked")
        Map<String, Object> params = (Map<String, Object>) action.get(actionType);
        if ("addNode".equals(actionType) || "updateNode".equals(actionType)) {
            if (!params.containsKey("type")) {
                throw new RuntimeException(ctx + ".actions[" + i
                        + "." + actionType + "]: 'type' is required");
            }
            Object typeVal = params.get("type");
            if (typeVal instanceof String typeStr
                    && !typeStr.contains("${")
                    && !typeRegistry.containsKey(typeStr)) {
                throw new RuntimeException(ctx + ".actions[" + i + "." + actionType
                        + "]: unknown type '" + typeStr + "'. Available: " + typeRegistry.keySet());
            }
        }
    }
}
```

Also add `validateRules()` that iterates over all rules and calls `validateRule`.

- [ ] **Step 3: Run validation tests**

Run: `mvn --batch-mode test -pl yaml/deployment -Dtest=YamlRuleValidationTest`
Expected: ALL PASS

- [ ] **Step 4: Write YamlRuleConverter test**

Create `YamlRuleConverterTest.java`:

```java
package io.casehub.desiredstate.yaml;

import io.casehub.desiredstate.annotations.runtime.Direction;
import io.casehub.desiredstate.annotations.runtime.PatternKind;
import io.casehub.desiredstate.annotations.runtime.ResolvedRule;
import io.casehub.desiredstate.api.Dependency;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.GraphMutation;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.annotations.runtime.GraphRuleEngine;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.casehub.desiredstate.yaml.model.YamlPattern;
import io.casehub.desiredstate.yaml.model.YamlRule;
import io.casehub.desiredstate.yaml.registry.NodeSpecRegistry;
import io.casehub.desiredstate.yaml.resolver.VariableResolver;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class YamlRuleConverterTest {

    record MonitorSpec(String target) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of("monitor"); }
    }

    private static final Map<String, String> TYPE_REGISTRY = Map.of(
            "sink", "io.casehub.desiredstate.yaml.YamlRuleConverterTest$MonitorSpec",
            "monitor", "io.casehub.desiredstate.yaml.YamlRuleConverterTest$MonitorSpec");

    private final DefaultDesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();

    @Test
    void toDeclarativeRule_addNodeAction_producesCorrectMutations() {
        YamlRule yamlRule = new YamlRule(
                List.of(),
                Map.of("sink", new YamlPattern("sink", null, Direction.DEPENDENCIES)),
                Map.of(), Map.of(),
                Map.of("guard", new YamlPattern("monitor", "sink", Direction.DEPENDENTS)),
                List.of(
                        Map.of("addNode", Map.of(
                                "id", "monitor-${match.sink.id}",
                                "type", "monitor",
                                "spec", Map.of("target", "${match.sink.id}"))),
                        Map.of("addDependency", Map.of(
                                "from", "monitor-${match.sink.id}",
                                "to", "${match.sink.id}"))));

        NodeSpecRegistry registry = NodeSpecRegistry.of(TYPE_REGISTRY);
        VariableResolver resolver = new VariableResolver(Map.of(), null, null);

        ResolvedRule.DeclarativeRule rule = YamlRuleConverter.toDeclarativeRule(
                "ensure-monitoring", yamlRule, resolver, registry);

        assertThat(rule.name()).isEqualTo("ensure-monitoring");
        assertThat(rule.patterns()).hasSize(2);
        assertThat(rule.patterns().get(0).kind()).isEqualTo(PatternKind.MATCH);
        assertThat(rule.patterns().get(1).kind()).isEqualTo(PatternKind.NOT_EXISTS);

        // Verify the action evaluator works
        DesiredNode sinkNode = new DesiredNode(NodeId.of("warehouse-sink"),
                new MonitorSpec("ws"), HumanGating.NONE);
        DesiredStateGraph graph = factory.of(List.of(sinkNode), List.of());

        var result = new GraphRuleEngine().evaluate(graph, List.of(rule));
        assertThat(result.nodes()).hasSize(2);
        assertThat(result.nodes()).containsKey(NodeId.of("monitor-warehouse-sink"));

        MonitorSpec monSpec = (MonitorSpec) result.nodes()
                .get(NodeId.of("monitor-warehouse-sink")).spec();
        assertThat(monSpec.target()).isEqualTo("warehouse-sink");

        assertThat(result.dependencies()).contains(
                new Dependency(NodeId.of("monitor-warehouse-sink"),
                        NodeId.of("warehouse-sink")));
    }
}
```

- [ ] **Step 5: Implement YamlRuleConverter**

Create `YamlRuleConverter.java`:

```java
package io.casehub.desiredstate.yaml;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.cfg.CoercionAction;
import com.fasterxml.jackson.databind.cfg.CoercionInputShape;
import io.casehub.desiredstate.annotations.runtime.MatchTemplateResolver;
import io.casehub.desiredstate.annotations.runtime.PatternKind;
import io.casehub.desiredstate.annotations.runtime.PatternParameterDescriptor;
import io.casehub.desiredstate.annotations.runtime.ResolvedRule;
import io.casehub.desiredstate.api.Dependency;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.GraphMutation;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.yaml.model.YamlPattern;
import io.casehub.desiredstate.yaml.model.YamlRule;
import io.casehub.desiredstate.yaml.registry.NodeSpecRegistry;
import io.casehub.desiredstate.yaml.resolver.VariableResolver;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public final class YamlRuleConverter {

    private YamlRuleConverter() {}

    public static ResolvedRule.DeclarativeRule toDeclarativeRule(
            String name, YamlRule yamlRule,
            VariableResolver resolver, NodeSpecRegistry registry) {

        List<PatternParameterDescriptor> patterns = new ArrayList<>();
        List<String> bindingNamesList = new ArrayList<>();

        for (Map.Entry<String, YamlPattern> entry : yamlRule.match().entrySet()) {
            YamlPattern p = entry.getValue();
            patterns.add(new PatternParameterDescriptor(
                    PatternKind.MATCH, p.type(),
                    p.of() != null ? p.of() : "", p.direction()));
            bindingNamesList.add(entry.getKey());
        }

        addPatterns(yamlRule.directDep(), PatternKind.DIRECT_DEP, patterns, bindingNamesList);
        addPatterns(yamlRule.reaches(), PatternKind.REACHES, patterns, bindingNamesList);
        addPatterns(yamlRule.notExists(), PatternKind.NOT_EXISTS, patterns, bindingNamesList);

        String[] bindingNames = bindingNamesList.toArray(String[]::new);

        List<Map<String, Object>> resolvedActions = new ArrayList<>();
        for (Map<String, Object> action : yamlRule.actions()) {
            resolvedActions.add(resolveVarInAction(action, resolver, name));
        }

        ObjectMapper coercionMapper = createCoercionMapper();

        return new ResolvedRule.DeclarativeRule(name, patterns, bindingNames,
                bindings -> evaluateActions(resolvedActions, bindings, registry,
                        coercionMapper, name));
    }

    @SuppressWarnings("unchecked")
    private static List<GraphMutation> evaluateActions(
            List<Map<String, Object>> actions,
            Map<String, DesiredNode> bindings,
            NodeSpecRegistry registry,
            ObjectMapper mapper,
            String ruleName) {

        List<GraphMutation> mutations = new ArrayList<>();

        for (Map<String, Object> action : actions) {
            String actionType = action.keySet().iterator().next();
            Map<String, Object> params = (Map<String, Object>) action.get(actionType);

            switch (actionType) {
                case "addNode" -> {
                    String id = MatchTemplateResolver.resolveNodeId(
                            (String) params.get("id"), bindings, ruleName);
                    String type = MatchTemplateResolver.resolve(
                            (String) params.get("type"), bindings);
                    Map<String, Object> specMap = params.containsKey("spec")
                            ? resolveMatchInMap((Map<String, Object>) params.get("spec"), bindings)
                            : Map.of();
                    Class<? extends NodeSpec> specClass = registry.resolve(type);
                    NodeSpec spec = mapper.convertValue(specMap, specClass);
                    HumanGating gating = params.containsKey("humanGating")
                            ? HumanGating.valueOf((String) params.get("humanGating"))
                            : HumanGating.NONE;
                    mutations.add(new GraphMutation.AddNode(
                            new DesiredNode(NodeId.of(id), spec, gating)));
                }
                case "removeNode" -> {
                    String id = MatchTemplateResolver.resolveNodeId(
                            (String) params.get("id"), bindings, ruleName);
                    mutations.add(new GraphMutation.RemoveNode(NodeId.of(id)));
                }
                case "updateNode" -> {
                    String id = MatchTemplateResolver.resolveNodeId(
                            (String) params.get("id"), bindings, ruleName);
                    String type = MatchTemplateResolver.resolve(
                            (String) params.get("type"), bindings);
                    Map<String, Object> specMap = params.containsKey("spec")
                            ? resolveMatchInMap((Map<String, Object>) params.get("spec"), bindings)
                            : Map.of();
                    Class<? extends NodeSpec> specClass = registry.resolve(type);
                    NodeSpec spec = mapper.convertValue(specMap, specClass);
                    HumanGating gating = params.containsKey("humanGating")
                            ? HumanGating.valueOf((String) params.get("humanGating"))
                            : HumanGating.NONE;
                    mutations.add(new GraphMutation.UpdateNode(NodeId.of(id),
                            new DesiredNode(NodeId.of(id), spec, gating)));
                }
                case "addDependency" -> {
                    String from = MatchTemplateResolver.resolve(
                            (String) params.get("from"), bindings);
                    String to = MatchTemplateResolver.resolve(
                            (String) params.get("to"), bindings);
                    mutations.add(new GraphMutation.AddDependency(
                            new Dependency(NodeId.of(from), NodeId.of(to))));
                }
                case "removeDependency" -> {
                    String from = MatchTemplateResolver.resolve(
                            (String) params.get("from"), bindings);
                    String to = MatchTemplateResolver.resolve(
                            (String) params.get("to"), bindings);
                    mutations.add(new GraphMutation.RemoveDependency(
                            new Dependency(NodeId.of(from), NodeId.of(to))));
                }
                default -> throw new IllegalArgumentException(
                        "Unknown action type: " + actionType);
            }
        }
        return mutations;
    }

    @SuppressWarnings("unchecked")
    private static Map<String, Object> resolveMatchInMap(
            Map<String, Object> input, Map<String, DesiredNode> bindings) {
        Map<String, Object> result = new LinkedHashMap<>();
        for (Map.Entry<String, Object> entry : input.entrySet()) {
            Object val = entry.getValue();
            if (val instanceof String s) {
                result.put(entry.getKey(), MatchTemplateResolver.resolve(s, bindings));
            } else if (val instanceof Map<?, ?> nested) {
                result.put(entry.getKey(),
                        resolveMatchInMap((Map<String, Object>) nested, bindings));
            } else {
                result.put(entry.getKey(), val);
            }
        }
        return result;
    }

    @SuppressWarnings("unchecked")
    private static Map<String, Object> resolveVarInAction(
            Map<String, Object> action, VariableResolver resolver, String context) {
        Map<String, Object> result = new LinkedHashMap<>();
        for (Map.Entry<String, Object> entry : action.entrySet()) {
            Object val = entry.getValue();
            if (val instanceof Map<?, ?> nested) {
                result.put(entry.getKey(),
                        resolveVarInActionParams((Map<String, Object>) nested, resolver, context));
            } else {
                result.put(entry.getKey(), val);
            }
        }
        return result;
    }

    @SuppressWarnings("unchecked")
    private static Map<String, Object> resolveVarInActionParams(
            Map<String, Object> params, VariableResolver resolver, String context) {
        Map<String, Object> result = new LinkedHashMap<>();
        for (Map.Entry<String, Object> entry : params.entrySet()) {
            Object val = entry.getValue();
            if (val instanceof String s && s.contains("${var.")) {
                result.put(entry.getKey(), resolver.resolveTemplateString(s, context));
            } else if (val instanceof Map<?, ?> nested) {
                result.put(entry.getKey(),
                        resolveVarInActionParams((Map<String, Object>) nested, resolver, context));
            } else {
                result.put(entry.getKey(), val);
            }
        }
        return result;
    }

    private static void addPatterns(Map<String, YamlPattern> section, PatternKind kind,
            List<PatternParameterDescriptor> patterns, List<String> bindingNames) {
        for (Map.Entry<String, YamlPattern> entry : section.entrySet()) {
            YamlPattern p = entry.getValue();
            patterns.add(new PatternParameterDescriptor(
                    kind, p.type(),
                    p.of() != null ? p.of() : "", p.direction()));
            bindingNames.add(entry.getKey());
        }
    }

    private static ObjectMapper createCoercionMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.coercionConfigDefaults()
                .setCoercion(CoercionInputShape.String, CoercionAction.TryConvert);
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        return mapper;
    }
}
```

- [ ] **Step 6: Run YamlRuleConverter tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlRuleConverterTest`
Expected: ALL PASS

- [ ] **Step 7: Wire rules into YamlGraphRecorder.createYamlGoalCompiler**

Add rules parameter and apply `GraphRuleEngine.evaluate()` after graph construction
but before invariant validation. In the compile lambda:

```java
// After building the DesiredStateGraph graph...

// Apply declarative rules
if (yamlGraph != null && !yamlGraph.rules().isEmpty()) {
    List<ResolvedRule> resolvedRules = new ArrayList<>();
    for (Map.Entry<String, YamlRule> entry : yamlGraph.rules().entrySet()) {
        resolvedRules.add(YamlRuleConverter.toDeclarativeRule(
                entry.getKey(), entry.getValue(), resolver, registry));
    }
    graph = new GraphRuleEngine().evaluate(graph, resolvedRules);
}

// Then invariant validation
if (!invariants.isEmpty()) {
    new GraphInvariantEngine().validate(graph, invariants);
}
```

- [ ] **Step 8: Wire validation in YamlDesiredStateProcessor**

Add rule validation call alongside invariant validation:

```java
if (!yamlGraph.rules().isEmpty()) {
    validateRules(yamlGraph.rules(), typeRegistry, fileName);
}
```

- [ ] **Step 9: Run all tests**

Run: `mvn --batch-mode test -pl yaml/runtime`
Run: `mvn --batch-mode test -pl yaml/deployment`
Run: `mvn --batch-mode test -pl examples/pipeline-yaml`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```
feat(#116): YamlRuleConverter and GoalCompiler rule wrapping

Build-time validation for YAML rules. YamlRuleConverter creates
DeclarativeRule instances with ${match.*} action evaluation closure.
GoalCompiler applies GraphRuleEngine after graph construction.

Refs #116
```

### Task 4: Pipeline-yaml rule integration test

**Files:**
- Create: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/MonitorSpec.java`
- Modify: `examples/pipeline-yaml/src/main/resources/META-INF/desiredstate/medallion-pipeline.yaml`
- Modify: `examples/pipeline-yaml/src/test/java/io/casehub/desiredstate/example/pipeline/yaml/PipelineYamlTest.java`

**Interfaces:**
- Consumes: YAML rule feature (Tasks 1-3), `MonitorSpec` NodeSpec
- Produces: Working integration test proving rule evaluation from YAML

- [ ] **Step 1: Create MonitorSpec in shared pipeline module**

```java
package io.casehub.desiredstate.example.pipeline;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.NodeTypeId;

@NodeTypeId("monitor")
public record MonitorSpec(String target) implements NodeSpec {
    @Override
    public NodeType nodeType() { return NodeType.of("monitor"); }
}
```

- [ ] **Step 2: Add rule to medallion-pipeline.yaml**

Append after the `faultPolicy` section:

```yaml
rules:
  ensure-monitoring:
    match:
      sink: { type: sink }
    notExists:
      guard: { type: monitor, of: sink, direction: DEPENDENTS }
    actions:
      - addNode:
          id: "monitor-${match.sink.id}"
          type: monitor
          spec:
            target: "${match.sink.id}"
      - addDependency:
          from: "monitor-${match.sink.id}"
          to: "${match.sink.id}"
```

- [ ] **Step 3: Update TYPE_REGISTRY in PipelineYamlTest**

Add the monitor entry:

```java
Map.entry("monitor", "io.casehub.desiredstate.example.pipeline.MonitorSpec")
```

- [ ] **Step 4: Update buildFromYaml to handle rules**

In the `@BeforeAll` method, after building invariants, convert rules:

```java
List<io.casehub.desiredstate.annotations.runtime.ResolvedRule> rules = new ArrayList<>();
io.casehub.desiredstate.yaml.resolver.VariableResolver resolver =
        new io.casehub.desiredstate.yaml.resolver.VariableResolver(
                yamlGraph.variables() != null ? yamlGraph.variables() : Map.of(),
                null, null);
io.casehub.desiredstate.yaml.registry.NodeSpecRegistry registry =
        io.casehub.desiredstate.yaml.registry.NodeSpecRegistry.of(TYPE_REGISTRY);
for (Map.Entry<String, io.casehub.desiredstate.yaml.model.YamlRule> ruleEntry :
        yamlGraph.rules().entrySet()) {
    rules.add(io.casehub.desiredstate.yaml.YamlRuleConverter.toDeclarativeRule(
            ruleEntry.getKey(), ruleEntry.getValue(), resolver, registry));
}
```

Then update the `compileSingleGraph()` to apply rules, or verify that the
`YamlGraphRecorder` now handles rules automatically when `yamlGraph.rules()`
is non-empty (from Task 3 wiring).

- [ ] **Step 5: Write rule integration tests**

```java
@Test
void yamlRule_ensureMonitoring_addMonitorForSink() {
    DesiredStateGraph graph = compileSingleGraph();
    // Rule should add monitor-warehouse-sink node
    assertThat(graph.nodes()).containsKey(NodeId.of("monitor-warehouse-sink"));

    // Verify monitor spec
    MonitorSpec monSpec = (MonitorSpec) graph.nodes()
            .get(NodeId.of("monitor-warehouse-sink")).spec();
    assertThat(monSpec.target()).isEqualTo("warehouse-sink");

    // Verify dependency: monitor depends on sink
    assertThat(graph.dependenciesOf(NodeId.of("monitor-warehouse-sink")))
            .contains(NodeId.of("warehouse-sink"));
}

@Test
void yamlRule_ensureMonitoring_convergesWithOneMonitor() {
    DesiredStateGraph graph = compileSingleGraph();
    // Only one sink → only one monitor added
    long monitorCount = graph.nodes().values().stream()
            .filter(n -> n.type().equals(io.casehub.desiredstate.api.NodeType.of("monitor")))
            .count();
    assertThat(monitorCount).isEqualTo(1);
}

@Test
void yamlRule_totalNodeCount_includesRuleGenerated() {
    DesiredStateGraph graph = compileSingleGraph();
    // 8 declared + 1 rule-generated monitor = 9
    // (debug-validator excluded by when: false)
    assertThat(graph.nodes()).hasSize(9);
}
```

- [ ] **Step 6: Run integration tests**

Run: `mvn --batch-mode test -pl examples/pipeline-yaml`
Expected: ALL PASS

- [ ] **Step 7: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```
feat(#116): pipeline-yaml example with declarative graph rule

Demonstrates ensure-monitoring rule: for every sink without a monitor,
add a monitor node with dependency. MonitorSpec added to shared pipeline
module. GraphRuleEngine fixed-point loop converges after one iteration.

Refs #116
```

---

## Batch 3: Lifecycle Phases

Safe wrap point: after this batch, YAML operators can declare lifecycle
phases with completion conditions and cross-phase node references. The
pipeline-yaml example demonstrates multi-phase compilation.

### Task 5: YAML lifecycle model types + validation

**Files:**
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlLifecycle.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlPhase.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlGraph.java`
- Modify: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java`
- Create: `yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/YamlLifecycleValidationTest.java`
- Create: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/model/YamlLifecycleDeserializationTest.java`

**Interfaces:**
- Consumes: `YamlNode`, `YamlGraph`, Jackson YAML deserialization
- Produces: `YamlLifecycle(phases: List<YamlPhase>)`, `YamlPhase(id, completionCondition, nodes)`.
  `YamlGraph` gains `lifecycle` field. Build-time error when both `lifecycle` and top-level
  `nodes` are present. Completion condition validated: `allPresent`, `never`, or
  `{ bean: "beanName" }`.

- [ ] **Step 1: Create YamlPhase and YamlLifecycle records**

```java
// YamlPhase.java
package io.casehub.desiredstate.yaml.model;

import java.util.Map;

public record YamlPhase(
        String id,
        Object completionCondition,
        Map<String, YamlNode> nodes) {
    public YamlPhase {
        nodes = nodes != null ? nodes : Map.of();
    }
}

// YamlLifecycle.java
package io.casehub.desiredstate.yaml.model;

import java.util.List;

public record YamlLifecycle(List<YamlPhase> phases) {
    public YamlLifecycle {
        phases = phases != null ? phases : List.of();
    }
}
```

Note: `completionCondition` is `Object` to support both `String` ("allPresent", "never")
and `Map<String, String>` (`{ bean: "beanName" }`) from Jackson deserialization.

- [ ] **Step 2: Add lifecycle field to YamlGraph**

```java
public record YamlGraph(
        YamlDesiredState desiredState,
        Map<String, String> variables,
        Map<String, YamlNode> nodes,
        List<YamlFaultPolicy> faultPolicy,
        Map<String, YamlInvariant> invariants,
        Map<String, YamlRule> rules,
        YamlLifecycle lifecycle) {
    public YamlGraph {
        // existing defaults...
        rules = rules != null ? rules : Map.of();
    }
}
```

- [ ] **Step 3: Write deserialization test**

Create `YamlLifecycleDeserializationTest.java`:

```java
package io.casehub.desiredstate.yaml.model;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class YamlLifecycleDeserializationTest {

    private final ObjectMapper mapper = new ObjectMapper(new YAMLFactory());

    @Test
    void deserialize_lifecycleWithPhases() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: lifecycle-test
                lifecycle:
                  phases:
                    - id: infrastructure
                      completionCondition: allPresent
                      nodes:
                        database:
                          type: db
                          spec:
                            engine: postgres
                    - id: application
                      completionCondition: allPresent
                      nodes:
                        api-server:
                          type: app
                          dependsOn: [database]
                          spec:
                            image: "api:latest"
                    - id: observability
                      completionCondition: never
                      nodes:
                        monitor:
                          type: monitor
                          dependsOn: [api-server]
                          spec:
                            target: api-server
                """;
        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);

        assertThat(graph.lifecycle()).isNotNull();
        assertThat(graph.lifecycle().phases()).hasSize(3);
        assertThat(graph.nodes()).isEmpty();

        YamlPhase infra = graph.lifecycle().phases().get(0);
        assertThat(infra.id()).isEqualTo("infrastructure");
        assertThat(infra.completionCondition()).isEqualTo("allPresent");
        assertThat(infra.nodes()).hasSize(1);
        assertThat(infra.nodes()).containsKey("database");

        YamlPhase obs = graph.lifecycle().phases().get(2);
        assertThat(obs.completionCondition()).isEqualTo("never");
    }

    @Test
    @SuppressWarnings("unchecked")
    void deserialize_beanCompletionCondition() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: bean-test
                lifecycle:
                  phases:
                    - id: custom
                      completionCondition:
                        bean: myCustomCondition
                      nodes:
                        node1:
                          type: app
                          spec: {}
                """;
        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);
        YamlPhase phase = graph.lifecycle().phases().get(0);
        assertThat(phase.completionCondition()).isInstanceOf(java.util.Map.class);
        java.util.Map<String, Object> condition =
                (java.util.Map<String, Object>) phase.completionCondition();
        assertThat(condition).containsEntry("bean", "myCustomCondition");
    }

    @Test
    void deserialize_noLifecycle_defaultsToNull() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: no-lifecycle
                nodes:
                  app:
                    type: app
                    spec: {}
                """;
        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);
        assertThat(graph.lifecycle()).isNull();
    }
}
```

- [ ] **Step 4: Run deserialization tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlLifecycleDeserializationTest`
Expected: ALL PASS

- [ ] **Step 5: Write validation tests**

Create `YamlLifecycleValidationTest.java`:

```java
package io.casehub.desiredstate.yaml.deployment;

import io.casehub.desiredstate.yaml.model.YamlGraph;
import io.casehub.desiredstate.yaml.model.YamlLifecycle;
import io.casehub.desiredstate.yaml.model.YamlNode;
import io.casehub.desiredstate.yaml.model.YamlPhase;
import io.casehub.desiredstate.yaml.model.YamlDesiredState;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

class YamlLifecycleValidationTest {

    private static final Map<String, String> TYPE_REGISTRY = Map.of(
            "db", "com.example.DbSpec",
            "app", "com.example.AppSpec");

    @Test
    void validate_lifecycleAndTopLevelNodes_throwsBuildError() {
        var graph = new YamlGraph(
                new YamlDesiredState("test", "conflict"),
                Map.of(),
                Map.of("app", new YamlNode("app", Map.of(), List.of(), null, null)),
                List.of(), Map.of(), Map.of(),
                new YamlLifecycle(List.of(
                        new YamlPhase("infra", "allPresent",
                                Map.of("db", new YamlNode("db", Map.of(), List.of(), null, null))))));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateLifecycle(
                graph, TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("nodes")
                .hasMessageContaining("lifecycle");
    }

    @Test
    void validate_emptyPhases_throwsBuildError() {
        var graph = new YamlGraph(
                new YamlDesiredState("test", "empty"),
                Map.of(), Map.of(), List.of(), Map.of(), Map.of(),
                new YamlLifecycle(List.of()));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateLifecycle(
                graph, TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("phase");
    }

    @Test
    void validate_unknownCompletionCondition_throwsBuildError() {
        var graph = new YamlGraph(
                new YamlDesiredState("test", "bad-cc"),
                Map.of(), Map.of(), List.of(), Map.of(), Map.of(),
                new YamlLifecycle(List.of(
                        new YamlPhase("infra", "whenReady",
                                Map.of("db", new YamlNode("db", Map.of(), List.of(), null, null))))));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateLifecycle(
                graph, TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("whenReady")
                .hasMessageContaining("allPresent");
    }

    @Test
    void validate_duplicatePhaseIds_throwsBuildError() {
        var graph = new YamlGraph(
                new YamlDesiredState("test", "dup-phase"),
                Map.of(), Map.of(), List.of(), Map.of(), Map.of(),
                new YamlLifecycle(List.of(
                        new YamlPhase("infra", "allPresent",
                                Map.of("db", new YamlNode("db", Map.of(), List.of(), null, null))),
                        new YamlPhase("infra", "never",
                                Map.of("app", new YamlNode("app", Map.of(), List.of(), null, null))))));
        assertThatThrownBy(() -> YamlDesiredStateProcessor.validateLifecycle(
                graph, TYPE_REGISTRY, "test.yaml"))
                .hasMessageContaining("infra")
                .hasMessageContaining("duplicate");
    }
}
```

- [ ] **Step 6: Implement lifecycle validation**

Add `static void validateLifecycle(YamlGraph, typeRegistry, fileName)`:

```java
static void validateLifecycle(YamlGraph graph, Map<String, String> typeRegistry,
                               String fileName) {
    if (graph.lifecycle() == null) return;

    if (!graph.nodes().isEmpty()) {
        throw new RuntimeException(fileName
                + ": cannot have both top-level 'nodes' and 'lifecycle'. "
                + "When lifecycle is present, nodes live inside phases.");
    }

    List<YamlPhase> phases = graph.lifecycle().phases();
    if (phases.isEmpty()) {
        throw new RuntimeException(fileName
                + ": lifecycle must have at least one phase");
    }

    Set<String> phaseIds = new HashSet<>();
    Set<String> allNodeIds = new HashSet<>();
    for (int i = 0; i < phases.size(); i++) {
        YamlPhase phase = phases.get(i);
        String ctx = fileName + ": lifecycle.phases[" + i + "]";

        if (phase.id() == null || phase.id().isBlank()) {
            throw new RuntimeException(ctx + ": phase id is required");
        }
        if (!phaseIds.add(phase.id())) {
            throw new RuntimeException(ctx + ": duplicate phase id '" + phase.id() + "'");
        }

        validateCompletionCondition(phase.completionCondition(), ctx);

        for (Map.Entry<String, YamlNode> nodeEntry : phase.nodes().entrySet()) {
            String nodeId = nodeEntry.getKey();
            YamlNode node = nodeEntry.getValue();
            allNodeIds.add(nodeId);

            if (!typeRegistry.containsKey(node.type())) {
                throw new RuntimeException(ctx + ": unknown node type '"
                        + node.type() + "' for node '" + nodeId + "'");
            }
        }
    }

    // Cross-phase dependency references are validated at compile time
    // (not build time) because carry-forward injects earlier phase nodes

    // Warn if last phase uses allPresent (lifecycle will terminate)
    YamlPhase lastPhase = phases.get(phases.size() - 1);
    if ("allPresent".equals(lastPhase.completionCondition())) {
        LOG.warnf("%s: last phase '%s' uses completionCondition 'allPresent' — "
                + "the lifecycle will terminate and reconciliation will stop. "
                + "Use 'never' for steady-state operation.", fileName, lastPhase.id());
    }
}

private static void validateCompletionCondition(Object condition, String ctx) {
    if (condition == null) {
        throw new RuntimeException(ctx + ": completionCondition is required");
    }
    if (condition instanceof String s) {
        if (!"allPresent".equals(s) && !"never".equals(s)) {
            throw new RuntimeException(ctx + ": unknown completionCondition '"
                    + s + "'. Valid: allPresent, never, or { bean: \"name\" }");
        }
    } else if (condition instanceof Map<?, ?> m) {
        if (!m.containsKey("bean")) {
            throw new RuntimeException(ctx
                    + ": completionCondition map must have 'bean' key");
        }
    } else {
        throw new RuntimeException(ctx + ": completionCondition must be a string "
                + "(allPresent, never) or a map ({ bean: \"name\" })");
    }
}
```

- [ ] **Step 7: Run validation tests**

Run: `mvn --batch-mode test -pl yaml/deployment -Dtest=YamlLifecycleValidationTest`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```
feat(#116): YAML lifecycle model types and validation

YamlLifecycle and YamlPhase records. YamlGraph gains lifecycle field.
Build-time validation: mutual exclusivity with nodes, completion
condition vocabulary, duplicate phase IDs, node types.

Refs #116
```

### Task 6: Lifecycle GoalCompiler with carry-forward

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`
- Modify: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java`
- Create: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlLifecycleCompilerTest.java`

**Interfaces:**
- Consumes: `YamlLifecycle`, `YamlPhase`, `CompilationResult.lifecycle()`, `Phase`, `CompletionCondition`
- Produces: `YamlGraphRecorder.createYamlLifecycleGoalCompiler()` — compiles lifecycle phases
  sequentially with carry-forward. Each phase: resolve variables → evaluate when: → build nodes →
  inject carry-forward → apply rules → validate invariants → build Phase.

- [ ] **Step 1: Write failing lifecycle compiler tests**

Create `YamlLifecycleCompilerTest.java`:

```java
package io.casehub.desiredstate.yaml;

import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.Phase;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.casehub.desiredstate.yaml.model.YamlDesiredState;
import io.casehub.desiredstate.yaml.model.YamlGraph;
import io.casehub.desiredstate.yaml.model.YamlLifecycle;
import io.casehub.desiredstate.yaml.model.YamlNode;
import io.casehub.desiredstate.yaml.model.YamlPhase;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class YamlLifecycleCompilerTest {

    record Spec(String name, String typeValue) implements NodeSpec {
        @Override
        public NodeType nodeType() { return NodeType.of(typeValue); }
    }

    private static final Map<String, String> TYPE_REGISTRY = Map.of(
            "db", "io.casehub.desiredstate.yaml.YamlLifecycleCompilerTest$Spec",
            "app", "io.casehub.desiredstate.yaml.YamlLifecycleCompilerTest$Spec",
            "monitor", "io.casehub.desiredstate.yaml.YamlLifecycleCompilerTest$Spec");

    private final DefaultDesiredStateGraphFactory factory = new DefaultDesiredStateGraphFactory();

    @Test
    void lifecycle_twoPhases_producesLifecycleResult() {
        YamlGraph yamlGraph = new YamlGraph(
                new YamlDesiredState("test", "lifecycle"),
                Map.of(), Map.of(), List.of(), Map.of(), Map.of(),
                new YamlLifecycle(List.of(
                        new YamlPhase("infra", "allPresent",
                                Map.of("database", new YamlNode("db",
                                        Map.of("name", "pg", "typeValue", "db"),
                                        List.of(), null, null))),
                        new YamlPhase("app", "never",
                                Map.of("api-server", new YamlNode("app",
                                        Map.of("name", "api", "typeValue", "app"),
                                        List.of("database"), null, null))))));

        var recorder = new YamlGraphRecorder();
        var compiler = recorder.createYamlLifecycleGoalCompiler(
                yamlGraph, TYPE_REGISTRY, Map.of(), List.of()).getValue();

        CompilationResult result = compiler.compile(null, factory);
        assertThat(result).isInstanceOf(CompilationResult.Lifecycle.class);

        List<Phase> phases = ((CompilationResult.Lifecycle) result).phases();
        assertThat(phases).hasSize(2);

        // Phase 1: just database
        assertThat(phases.get(0).id()).isEqualTo("infra");
        assertThat(phases.get(0).graph().nodes()).containsKey(NodeId.of("database"));
        assertThat(phases.get(0).graph().nodes()).hasSize(1);
        assertThat(phases.get(0).isTerminal()).isFalse();

        // Phase 2: api-server + carried-forward database
        assertThat(phases.get(1).id()).isEqualTo("app");
        assertThat(phases.get(1).graph().nodes()).containsKey(NodeId.of("api-server"));
        assertThat(phases.get(1).graph().nodes()).containsKey(NodeId.of("database"));
        assertThat(phases.get(1).graph().nodes()).hasSize(2);
        assertThat(phases.get(1).isTerminal()).isTrue();
    }

    @Test
    void lifecycle_carryForward_dependenciesResolveAcrossPhases() {
        YamlGraph yamlGraph = new YamlGraph(
                new YamlDesiredState("test", "deps"),
                Map.of(), Map.of(), List.of(), Map.of(), Map.of(),
                new YamlLifecycle(List.of(
                        new YamlPhase("infra", "allPresent",
                                Map.of("database", new YamlNode("db",
                                        Map.of("name", "pg", "typeValue", "db"),
                                        List.of(), null, null))),
                        new YamlPhase("app", "never",
                                Map.of("api-server", new YamlNode("app",
                                        Map.of("name", "api", "typeValue", "app"),
                                        List.of("database"), null, null))))));

        var recorder = new YamlGraphRecorder();
        var compiler = recorder.createYamlLifecycleGoalCompiler(
                yamlGraph, TYPE_REGISTRY, Map.of(), List.of()).getValue();

        CompilationResult result = compiler.compile(null, factory);
        List<Phase> phases = ((CompilationResult.Lifecycle) result).phases();

        DesiredStateGraph appGraph = phases.get(1).graph();
        assertThat(appGraph.dependenciesOf(NodeId.of("api-server")))
                .contains(NodeId.of("database"));
    }

    @Test
    void lifecycle_overrideNodeInLaterPhase() {
        YamlGraph yamlGraph = new YamlGraph(
                new YamlDesiredState("test", "override"),
                Map.of(), Map.of(), List.of(), Map.of(), Map.of(),
                new YamlLifecycle(List.of(
                        new YamlPhase("infra", "allPresent",
                                Map.of("database", new YamlNode("db",
                                        Map.of("name", "pg-v1", "typeValue", "db"),
                                        List.of(), null, null))),
                        new YamlPhase("app", "never",
                                Map.of("database", new YamlNode("db",
                                        Map.of("name", "pg-v2", "typeValue", "db"),
                                        List.of(), null, null))))));

        var recorder = new YamlGraphRecorder();
        var compiler = recorder.createYamlLifecycleGoalCompiler(
                yamlGraph, TYPE_REGISTRY, Map.of(), List.of()).getValue();

        CompilationResult result = compiler.compile(null, factory);
        List<Phase> phases = ((CompilationResult.Lifecycle) result).phases();

        Spec infraSpec = (Spec) phases.get(0).graph().nodes()
                .get(NodeId.of("database")).spec();
        assertThat(infraSpec.name()).isEqualTo("pg-v1");

        Spec appSpec = (Spec) phases.get(1).graph().nodes()
                .get(NodeId.of("database")).spec();
        assertThat(appSpec.name()).isEqualTo("pg-v2");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlLifecycleCompilerTest`
Expected: compilation error — `createYamlLifecycleGoalCompiler` doesn't exist.

- [ ] **Step 3: Implement createYamlLifecycleGoalCompiler**

Add to `YamlGraphRecorder.java`:

```java
@SuppressWarnings({"unchecked", "rawtypes"})
public RuntimeValue<GoalCompiler> createYamlLifecycleGoalCompiler(
        io.casehub.desiredstate.yaml.model.YamlGraph yamlGraph,
        Map<String, String> typeRegistryMap,
        Map<String, String> inlineVariables,
        List<ResolvedInvariant> invariants) {

    ObjectMapper mapper = new ObjectMapper();
    NodeSpecRegistry registry = NodeSpecRegistry.of(typeRegistryMap);

    return new RuntimeValue<>((GoalCompiler) (goals, factory) -> {
        VariableResolver resolver = new VariableResolver(inlineVariables, null, null);
        List<Phase> phases = new ArrayList<>();
        List<DesiredNode> carryForwardNodes = new ArrayList<>();
        List<Dependency> carryForwardDeps = new ArrayList<>();

        for (io.casehub.desiredstate.yaml.model.YamlPhase yamlPhase :
                yamlGraph.lifecycle().phases()) {

            // Build this phase's own nodes
            Set<String> excludedNodeIds = new HashSet<>();
            List<DesiredNode> phaseNodes = new ArrayList<>();
            Set<String> phaseNodeIds = new HashSet<>();

            for (Map.Entry<String, io.casehub.desiredstate.yaml.model.YamlNode> entry :
                    yamlPhase.nodes().entrySet()) {
                String nodeId = entry.getKey();
                io.casehub.desiredstate.yaml.model.YamlNode yamlNode = entry.getValue();

                if (yamlNode.when() != null) {
                    String resolved = resolver.resolveString(yamlNode.when(), nodeId);
                    if (!isTruthy(resolved)) {
                        excludedNodeIds.add(nodeId);
                        continue;
                    }
                }

                Class<? extends NodeSpec> specClass = registry.resolve(yamlNode.type());
                Map<String, Object> resolvedSpec = resolver.resolveMap(
                        yamlNode.spec() != null ? yamlNode.spec() : Map.of(), nodeId);
                NodeSpec spec = mapper.convertValue(resolvedSpec, specClass);
                phaseNodes.add(new DesiredNode(NodeId.of(nodeId), spec, yamlNode.humanGating()));
                phaseNodeIds.add(nodeId);
            }

            // Inject carry-forward nodes (earlier phases' output)
            // Phase-declared nodes override carried-forward nodes
            List<DesiredNode> allNodes = new ArrayList<>();
            for (DesiredNode cf : carryForwardNodes) {
                if (!phaseNodeIds.contains(cf.id().value())) {
                    allNodes.add(cf);
                }
            }
            allNodes.addAll(phaseNodes);

            // Build dependencies — this phase's declared deps + carry-forward deps
            List<Dependency> phaseDeps = new ArrayList<>();
            for (Map.Entry<String, io.casehub.desiredstate.yaml.model.YamlNode> entry :
                    yamlPhase.nodes().entrySet()) {
                String nodeId = entry.getKey();
                if (excludedNodeIds.contains(nodeId)) continue;
                for (Object dep : entry.getValue().dependsOn()) {
                    String depId = io.casehub.desiredstate.yaml.model.YamlNode
                            .dependencyNodeId(dep);
                    if (excludedNodeIds.contains(depId)) {
                        boolean optional = io.casehub.desiredstate.yaml.model.YamlNode
                                .isDependencyOptional(dep);
                        if (!optional) {
                            throw new IllegalStateException("Node '" + nodeId
                                    + "' depends on excluded conditional node '" + depId + "'");
                        }
                        continue;
                    }
                    phaseDeps.add(new Dependency(NodeId.of(nodeId), NodeId.of(depId)));
                }
            }

            // Include carry-forward deps for carried-forward nodes
            for (Dependency cfDep : carryForwardDeps) {
                boolean fromInPhase = allNodes.stream()
                        .anyMatch(n -> n.id().equals(cfDep.from()));
                boolean toInPhase = allNodes.stream()
                        .anyMatch(n -> n.id().equals(cfDep.to()));
                if (fromInPhase && toInPhase
                        && !phaseNodeIds.contains(cfDep.from().value())) {
                    phaseDeps.add(cfDep);
                }
            }

            DesiredStateGraph phaseGraph = factory.of(allNodes, phaseDeps);

            // Apply rules per-phase
            if (!yamlGraph.rules().isEmpty()) {
                List<io.casehub.desiredstate.annotations.runtime.ResolvedRule> resolvedRules =
                        new ArrayList<>();
                for (Map.Entry<String, io.casehub.desiredstate.yaml.model.YamlRule> ruleEntry :
                        yamlGraph.rules().entrySet()) {
                    resolvedRules.add(YamlRuleConverter.toDeclarativeRule(
                            ruleEntry.getKey(), ruleEntry.getValue(), resolver, registry));
                }
                phaseGraph = new io.casehub.desiredstate.annotations.runtime.GraphRuleEngine()
                        .evaluate(phaseGraph, resolvedRules);
            }

            // Validate invariants per-phase
            if (!invariants.isEmpty()) {
                new io.casehub.desiredstate.annotations.runtime.GraphInvariantEngine()
                        .validate(phaseGraph, invariants);
            }

            // Build completion condition
            io.casehub.desiredstate.api.CompletionCondition cc =
                    resolveCompletionCondition(yamlPhase.completionCondition());

            phases.add(new Phase(yamlPhase.id(), phaseGraph, cc));

            // Update carry-forward: accumulate this phase's output
            carryForwardNodes = new ArrayList<>(phaseGraph.nodes().values());
            carryForwardDeps = new ArrayList<>(phaseGraph.dependencies());
        }

        return CompilationResult.lifecycle(phases);
    });
}

private static io.casehub.desiredstate.api.CompletionCondition resolveCompletionCondition(
        Object condition) {
    if (condition instanceof String s) {
        return switch (s) {
            case "allPresent" -> io.casehub.desiredstate.api.CompletionCondition.allPresent();
            case "never" -> io.casehub.desiredstate.api.CompletionCondition.never();
            default -> throw new IllegalArgumentException(
                    "Unknown completionCondition: " + s);
        };
    }
    if (condition instanceof Map<?, ?> m) {
        String beanName = (String) m.get("bean");
        return io.quarkus.arc.Arc.container()
                .instance(io.casehub.desiredstate.api.CompletionCondition.class,
                        new jakarta.enterprise.inject.literal.NamedLiteral(beanName))
                .get();
    }
    throw new IllegalArgumentException("Invalid completionCondition: " + condition);
}
```

- [ ] **Step 4: Wire lifecycle mode in YamlDesiredStateProcessor**

In `discoverYamlGraphs`, after parsing and validation, check lifecycle mode:

```java
if (yamlGraph.lifecycle() != null) {
    validateLifecycle(yamlGraph, typeRegistry, fileName);

    @SuppressWarnings("rawtypes")
    RuntimeValue<GoalCompiler> compiler = recorder.createYamlLifecycleGoalCompiler(
            yamlGraph, typeRegistry,
            yamlGraph.variables() != null ? yamlGraph.variables() : Map.of(),
            invariants);

    // Register as GoalCompiler bean (same pattern as single-graph mode)
    syntheticBeans.produce(SyntheticBeanBuildItem.configure(GoalCompiler.class)
            .scope(ApplicationScoped.class)
            .unremovable()
            .setRuntimeInit()
            .addQualifier()
                .annotation(DesiredStateQualifier.class)
                .addValue("namespace", ns)
                .addValue("name", name)
                .done()
            .runtimeValue(compiler)
            .done());
} else {
    // Existing single-graph compilation path...
}
```

- [ ] **Step 5: Run lifecycle compiler tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlLifecycleCompilerTest`
Expected: ALL PASS

- [ ] **Step 6: Run all tests**

Run: `mvn --batch-mode test -pl yaml/runtime`
Run: `mvn --batch-mode test -pl yaml/deployment`
Run: `mvn --batch-mode test -pl examples/pipeline-yaml`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```
feat(#116): lifecycle GoalCompiler with carry-forward injection

Sequential per-phase compilation. Each phase's output nodes are
injected into later phases so dependsOn resolves across phases.
Completion conditions: allPresent, never. Per-phase rule and
invariant evaluation.

Refs #116
```

### Task 7: Pipeline-yaml lifecycle integration test

**Files:**
- Create: `examples/pipeline-yaml/src/main/resources/META-INF/desiredstate/lifecycle-pipeline.yaml`
- Create: `examples/pipeline-yaml/src/test/java/io/casehub/desiredstate/example/pipeline/yaml/LifecyclePipelineTest.java`

**Interfaces:**
- Consumes: Lifecycle feature (Tasks 5-6), all pipeline NodeSpec types
- Produces: Working integration test proving multi-phase compilation with
  carry-forward and per-phase rule evaluation

- [ ] **Step 1: Create lifecycle-pipeline.yaml**

```yaml
desiredState:
  namespace: pipeline-yaml
  name: lifecycle-medallion

variables:
  source_uri: s3://data/customers.csv
  batch_size: "1000"

lifecycle:
  phases:
    - id: infrastructure
      completionCondition: allPresent
      nodes:
        csv-source:
          type: data-source
          spec:
            name: customers
            format: CSV
            uri: ${var.source_uri}

        customer-schema:
          type: schema
          spec:
            name: customer-schema
            fields: [id, name, email]
            version: 1

    - id: processing
      completionCondition: allPresent
      nodes:
        csv-ingest:
          type: ingestion
          dependsOn: [csv-source]
          spec:
            sourceRef: csv-source
            batchSize: ${var.batch_size}
            format: CSV

        dedup-cleanser:
          type: cleanser
          dependsOn: [csv-ingest, customer-schema]
          spec:
            rules: [dedup, nullcheck]
            deduplication: true
            nullHandling: DROP

        quality-validator:
          type: validator
          dependsOn: [dedup-cleanser, customer-schema]
          spec:
            schemaRef: customer-schema
            qualityThreshold: 0.95
            anomalyDetection: true

    - id: delivery
      completionCondition: never
      nodes:
        aggregate-tx:
          type: transformer
          dependsOn: [quality-validator]
          spec:
            aggregations: [sum, avg]
            reshapeRules: []
            outputFormat: parquet
            approvalRequired: true

        warehouse-sink:
          type: sink
          dependsOn: [aggregate-tx]
          spec:
            destination: s3://warehouse/gold/
            format: parquet
            partitionKeys: [date]
            approvalRequired: true

rules:
  ensure-monitoring:
    match:
      sink: { type: sink }
    notExists:
      guard: { type: monitor, of: sink, direction: DEPENDENTS }
    actions:
      - addNode:
          id: "monitor-${match.sink.id}"
          type: monitor
          spec:
            target: "${match.sink.id}"
      - addDependency:
          from: "monitor-${match.sink.id}"
          to: "${match.sink.id}"

invariants:
  every-sink-has-upstream:
    match:
      sink: { type: sink }
    directDep:
      upstream: { type: transformer, of: sink, direction: DEPENDENCIES }
    message: "Sink ${match.sink.id} has no upstream transformer"
```

- [ ] **Step 2: Write lifecycle integration test**

Create `LifecyclePipelineTest.java`:

```java
package io.casehub.desiredstate.example.pipeline.yaml;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.Phase;
import io.casehub.desiredstate.example.pipeline.DataSourceSpec;
import io.casehub.desiredstate.example.pipeline.MonitorSpec;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.casehub.desiredstate.yaml.YamlGraphRecorder;
import io.casehub.desiredstate.yaml.YamlInvariantConverter;
import io.casehub.desiredstate.yaml.model.YamlGraph;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import java.io.InputStream;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class LifecyclePipelineTest {

    private static GoalCompiler<Void> compiler;
    private static final DefaultDesiredStateGraphFactory factory =
            new DefaultDesiredStateGraphFactory();

    private static final Map<String, String> TYPE_REGISTRY = Map.ofEntries(
            Map.entry("data-source", "io.casehub.desiredstate.example.pipeline.DataSourceSpec"),
            Map.entry("schema", "io.casehub.desiredstate.example.pipeline.SchemaSpec"),
            Map.entry("ingestion", "io.casehub.desiredstate.example.pipeline.IngestionSpec"),
            Map.entry("cleanser", "io.casehub.desiredstate.example.pipeline.CleanserSpec"),
            Map.entry("enricher", "io.casehub.desiredstate.example.pipeline.EnricherSpec"),
            Map.entry("validator", "io.casehub.desiredstate.example.pipeline.ValidatorSpec"),
            Map.entry("transformer", "io.casehub.desiredstate.example.pipeline.TransformerSpec"),
            Map.entry("sink", "io.casehub.desiredstate.example.pipeline.SinkSpec"),
            Map.entry("ai-review", "io.casehub.desiredstate.example.pipeline.AiReviewSpec"),
            Map.entry("human-review", "io.casehub.desiredstate.example.pipeline.HumanReviewSpec"),
            Map.entry("monitor", "io.casehub.desiredstate.example.pipeline.MonitorSpec"));

    @BeforeAll
    @SuppressWarnings("unchecked")
    static void buildFromYaml() throws Exception {
        ObjectMapper yamlMapper = new ObjectMapper(new YAMLFactory());
        try (InputStream is = LifecyclePipelineTest.class.getClassLoader()
                .getResourceAsStream("META-INF/desiredstate/lifecycle-pipeline.yaml")) {
            assertThat(is).as("Lifecycle YAML must be on classpath").isNotNull();
            YamlGraph yamlGraph = yamlMapper.readValue(is, YamlGraph.class);

            List<io.casehub.desiredstate.annotations.runtime.ResolvedInvariant> invariants =
                    new ArrayList<>();
            for (var inv : yamlGraph.invariants().entrySet()) {
                invariants.add(YamlInvariantConverter.toDeclarativeInvariant(
                        inv.getKey(), inv.getValue()));
            }

            YamlGraphRecorder recorder = new YamlGraphRecorder();
            compiler = recorder.createYamlLifecycleGoalCompiler(
                    yamlGraph, TYPE_REGISTRY,
                    yamlGraph.variables() != null ? yamlGraph.variables() : Map.of(),
                    invariants).getValue();
        }
    }

    @Test
    void lifecycle_producesThreePhases() {
        CompilationResult result = compiler.compile(null, factory);
        assertThat(result).isInstanceOf(CompilationResult.Lifecycle.class);
        List<Phase> phases = ((CompilationResult.Lifecycle) result).phases();
        assertThat(phases).hasSize(3);
        assertThat(phases.get(0).id()).isEqualTo("infrastructure");
        assertThat(phases.get(1).id()).isEqualTo("processing");
        assertThat(phases.get(2).id()).isEqualTo("delivery");
    }

    @Test
    void infrastructure_hasTwoNodes() {
        List<Phase> phases = compilePhases();
        DesiredStateGraph infra = phases.get(0).graph();
        assertThat(infra.nodes()).hasSize(2);
        assertThat(infra.nodes()).containsKey(NodeId.of("csv-source"));
        assertThat(infra.nodes()).containsKey(NodeId.of("customer-schema"));
    }

    @Test
    void infrastructure_completionIsAllPresent() {
        List<Phase> phases = compilePhases();
        assertThat(phases.get(0).isTerminal()).isFalse();
    }

    @Test
    void processing_carryForwardInfrastructureNodes() {
        List<Phase> phases = compilePhases();
        DesiredStateGraph proc = phases.get(1).graph();
        // Phase 2's own nodes + carried-forward from phase 1
        assertThat(proc.nodes()).containsKey(NodeId.of("csv-source"));
        assertThat(proc.nodes()).containsKey(NodeId.of("customer-schema"));
        assertThat(proc.nodes()).containsKey(NodeId.of("csv-ingest"));
        assertThat(proc.nodes()).containsKey(NodeId.of("dedup-cleanser"));
        assertThat(proc.nodes()).containsKey(NodeId.of("quality-validator"));
    }

    @Test
    void processing_crossPhaseDependenciesResolve() {
        List<Phase> phases = compilePhases();
        DesiredStateGraph proc = phases.get(1).graph();
        assertThat(proc.dependenciesOf(NodeId.of("csv-ingest")))
                .contains(NodeId.of("csv-source"));
        assertThat(proc.dependenciesOf(NodeId.of("dedup-cleanser")))
                .contains(NodeId.of("customer-schema"));
    }

    @Test
    void delivery_carryForwardAllPriorNodes() {
        List<Phase> phases = compilePhases();
        DesiredStateGraph delivery = phases.get(2).graph();
        // All prior nodes + delivery's own + rule-generated
        assertThat(delivery.nodes()).containsKey(NodeId.of("csv-source"));
        assertThat(delivery.nodes()).containsKey(NodeId.of("aggregate-tx"));
        assertThat(delivery.nodes()).containsKey(NodeId.of("warehouse-sink"));
    }

    @Test
    void delivery_isTerminal() {
        List<Phase> phases = compilePhases();
        assertThat(phases.get(2).isTerminal()).isTrue();
    }

    @Test
    void delivery_ruleAddsMonitor() {
        List<Phase> phases = compilePhases();
        DesiredStateGraph delivery = phases.get(2).graph();
        assertThat(delivery.nodes()).containsKey(NodeId.of("monitor-warehouse-sink"));

        MonitorSpec monSpec = (MonitorSpec) delivery.nodes()
                .get(NodeId.of("monitor-warehouse-sink")).spec();
        assertThat(monSpec.target()).isEqualTo("warehouse-sink");
    }

    @Test
    void delivery_variableSubstitution() {
        List<Phase> phases = compilePhases();
        DesiredStateGraph infra = phases.get(0).graph();
        DataSourceSpec dsSpec = (DataSourceSpec) infra.nodes()
                .get(NodeId.of("csv-source")).spec();
        assertThat(dsSpec.uri()).isEqualTo("s3://data/customers.csv");
    }

    private List<Phase> compilePhases() {
        return ((CompilationResult.Lifecycle) compiler.compile(null, factory)).phases();
    }
}
```

- [ ] **Step 3: Run lifecycle integration test**

Run: `mvn --batch-mode test -pl examples/pipeline-yaml -Dtest=LifecyclePipelineTest`
Expected: ALL PASS

- [ ] **Step 4: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```
feat(#116): pipeline-yaml lifecycle example with three-phase compilation

Demonstrates infrastructure → processing → delivery lifecycle with
carry-forward, cross-phase dependencies, per-phase rule evaluation
(monitor added in delivery phase), and invariant validation.

Refs #116
```

---

## Summary

| Batch | Tasks | What's working after |
|-------|-------|---------------------|
| 1: Engine Support | 1 | `GraphRuleEngine` evaluates `DeclarativeRule` with action closures |
| 2: YAML Rules | 2-4 | YAML declarative rules with `${match.*}` action templates |
| 3: Lifecycle | 5-7 | Multi-phase lifecycle with carry-forward and per-phase rules |

**Total:** 3 batches, 7 tasks

**What Phase 2 delivers:** An operator can write YAML with structural graph
rules (pattern matching → node/edge mutations) and lifecycle phases
(sequential compilation with carry-forward injection). Together with Phase 1,
the YAML surface now covers: nodes, dependencies, fault policies, invariants,
conditional inclusion, graph rules, and lifecycle phases — no Java required.

**What Phase 3 adds:** forEach cardinality stamping and composable modules.

## References

- `specs/issue-116-yaml-language-design/2026-08-27-yaml-language-extensions-design.md` — design spec (§6.4, §6.5, §4, §7, §8)
- `specs/issue-116-yaml-language-design/decisions.md` — D3, D6, D7, D8, D11, D12, D16
- `annotations/runtime/.../ResolvedRule.java:30-32` — current DeclarativeRule (placeholder)
- `annotations/runtime/.../GraphRuleEngine.java:60-66` — evaluateRule dispatch (DeclarativeRule returns List.of())
- `annotations/runtime/.../PatternEvaluator.java` — shared pattern evaluation
- `annotations/runtime/.../GraphInvariantEngine.java:99-136` — DeclarativeInvariant evaluation (pattern for DeclarativeRule)
- `annotations/runtime/.../DesiredStateGraphRecorder.java:96-101` — annotation rule wrapping pattern
- `annotations/runtime/.../DesiredStateGraphRecorder.java:120-134` — applyGraphRulesToResult (per-phase rule application)
- `yaml/runtime/.../YamlGraphRecorder.java:43-114` — current GoalCompiler factory
- `yaml/runtime/.../YamlInvariantConverter.java` — pattern→descriptor conversion pattern
- `yaml/runtime/.../VariableResolver.java` — prefix-dispatched resolution
- `yaml/deployment/.../YamlDesiredStateProcessor.java` — build step pipeline
- `yaml/runtime/.../model/YamlInvariant.java` — YAML model pattern (for YamlRule)
- `api/.../CompilationResult.java` — SingleGraph | Lifecycle sealed interface
- `api/.../Phase.java:5-15` — phase record with CompletionCondition
- `api/.../CompletionCondition.java` — allPresent(), never(), FI interface
- `api/.../GraphMutation.java` — AddNode, RemoveNode, UpdateNode, AddDependency, RemoveDependency
- `examples/pipeline-yaml/.../medallion-pipeline.yaml` — current YAML example
- `examples/pipeline-yaml/.../PipelineYamlTest.java` — integration test pattern
- `examples/pipeline-annotated/.../MedallionPipeline.java:108-116` — annotation rule pattern (reference)
- `examples/pipeline-annotated/.../MonitorSpec.java` — MonitorSpec (in annotated, not shared)
- `plans/2026-08-28-phase1-yaml-extensions.md` — Phase 1 plan (format reference)
- #116 — operator-first declaration language
