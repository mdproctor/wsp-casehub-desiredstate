# Module Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #126 — Adopt yaml-core module system + parameter validation
**Issue group:** #128, #126

**Goal:** Replace desiredstate's local module system with yaml-core's shared
primitives, using `ModuleBridge<T>` for typed module expansion.

**Architecture:** Create `DesiredStateModuleContent` record as the domain-typed
content container and `DesiredStateModuleBridge` implementing `ModuleBridge<T>`.
Wire yaml-core's typed `ModuleExpander.expand()` into YamlGraphRecorder.
Update the deployment processor to use yaml-core types with `YamlCoreJacksonModule`
for dynamic section capture and `resolveExtensions()` for module inheritance.
Delete all local module types.

**Tech Stack:** Java 21, Quarkus, Jackson, yaml-core (platform), yaml-jackson (platform)

## Global Constraints

- Pre-release — backward compatibility not required
- YAML format unchanged — existing module files continue to work via dynamic section capture
- `casehub-yaml-core` already on classpath (added in #128)
- `casehub-platform-yaml-jackson` is the new dependency (for `YamlCoreJacksonModule`)
- All 108 yaml/runtime + 28 yaml/deployment tests must pass after migration
- Full 28-module build must be green

---

## Batch 1: Bridge foundation

### Task 1: Create DesiredStateModuleBridge with tests

**Files:**
- Modify: `yaml/runtime/pom.xml` — add yaml-jackson dependency
- Modify: `yaml/deployment/pom.xml` — add yaml-jackson dependency
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/DesiredStateModuleContent.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/DesiredStateModuleBridge.java`
- Test: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/DesiredStateModuleBridgeTest.java`

**Interfaces:**
- Produces: `DesiredStateModuleContent(Map<String, YamlNode>, Map<String, YamlRule>, Map<String, YamlInvariant>)` — domain content record
- Produces: `DesiredStateModuleBridge(ObjectMapper)` — implements `ModuleBridge<DesiredStateModuleContent>` with `fromSections()`, `toSections()`, `rewriter()`
- Consumes: `io.casehub.yaml.core.module.ModuleBridge<T>` — platform interface
- Consumes: `io.casehub.yaml.core.module.ModuleExpander.expand(imports, modules, content, bridge)` — typed overload
- Consumes: `io.casehub.yaml.core.module.TypedExpandedModule<T>` — typed result

- [ ] **Step 1: Add yaml-jackson dependency to yaml/runtime pom.xml**

Add after the existing `casehub-platform-yaml-core` dependency:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-yaml-jackson</artifactId>
</dependency>
```

- [ ] **Step 2: Add yaml-jackson dependency to yaml/deployment pom.xml**

Add after the existing `casehub-platform-yaml-core` dependency:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-yaml-jackson</artifactId>
</dependency>
```

- [ ] **Step 3: Create DesiredStateModuleContent record**

Use `ide_create_file` to create `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/DesiredStateModuleContent.java`:

```java
package io.casehub.desiredstate.yaml;

import io.casehub.desiredstate.yaml.model.YamlInvariant;
import io.casehub.desiredstate.yaml.model.YamlNode;
import io.casehub.desiredstate.yaml.model.YamlRule;

import java.util.Map;

public record DesiredStateModuleContent(
        Map<String, YamlNode> nodes,
        Map<String, YamlRule> rules,
        Map<String, YamlInvariant> invariants) {

    public DesiredStateModuleContent {
        if (nodes == null) { nodes = Map.of(); }
        if (rules == null) { rules = Map.of(); }
        if (invariants == null) { invariants = Map.of(); }
    }

    public static DesiredStateModuleContent empty() {
        return new DesiredStateModuleContent(Map.of(), Map.of(), Map.of());
    }
}
```

- [ ] **Step 4: Write failing tests for DesiredStateModuleBridge**

Create `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/DesiredStateModuleBridgeTest.java`:

```java
package io.casehub.desiredstate.yaml;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.desiredstate.yaml.model.YamlInvariant;
import io.casehub.desiredstate.yaml.model.YamlNode;
import io.casehub.desiredstate.yaml.model.YamlRule;
import io.casehub.yaml.core.module.ModuleExpander;
import io.casehub.yaml.core.module.SectionContentRewriter;
import io.casehub.yaml.core.module.TypedExpandedModule;
import io.casehub.yaml.core.module.YamlImport;
import io.casehub.yaml.core.module.YamlModule;
import io.casehub.yaml.core.module.YamlModuleParameter;
import org.junit.jupiter.api.Test;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class DesiredStateModuleBridgeTest {

    private final ObjectMapper mapper = new ObjectMapper();
    private final DesiredStateModuleBridge bridge = new DesiredStateModuleBridge(mapper);

    @Test
    void roundTrip_nodesPreserved() {
        var content = new DesiredStateModuleContent(
                Map.of("monitor", new YamlNode("monitor",
                        Map.of("target", "sink"), List.of("sink"),
                        null, null, null, null, null)),
                Map.of(), Map.of());

        var sections = bridge.toSections(content);
        var restored = bridge.fromSections(sections);

        assertThat(restored.nodes()).containsKey("monitor");
        assertThat(restored.nodes().get("monitor").type()).isEqualTo("monitor");
    }

    @Test
    void roundTrip_rulesAndInvariantsPreserved() {
        var rule = new YamlRule(
                Map.of("mon", new YamlRule.MatchEntry("monitor", null, null)),
                null, null, null,
                new YamlRule.ActionBlock(
                        Map.of("alert", Map.of("type", "alerter")),
                        null, null));

        var invariant = new YamlInvariant(
                Map.of("mon", new YamlInvariant.MatchEntry("monitor", null)),
                null, null, null);

        var content = new DesiredStateModuleContent(
                Map.of(), Map.of("auto-alert", rule),
                Map.of("monitor-check", invariant));

        var sections = bridge.toSections(content);
        var restored = bridge.fromSections(sections);

        assertThat(restored.rules()).containsKey("auto-alert");
        assertThat(restored.invariants()).containsKey("monitor-check");
    }

    @Test
    void roundTrip_emptySections() {
        var content = DesiredStateModuleContent.empty();
        var sections = bridge.toSections(content);
        var restored = bridge.fromSections(sections);

        assertThat(restored.nodes()).isEmpty();
        assertThat(restored.rules()).isEmpty();
        assertThat(restored.invariants()).isEmpty();
    }

    @Test
    void rewriter_aliasesDependencyInNodesSection() {
        SectionContentRewriter rewriter = bridge.rewriter();
        assertThat(rewriter).isNotNull();

        var nodeMap = new LinkedHashMap<String, Object>();
        nodeMap.put("type", "alerter");
        nodeMap.put("dependsOn", List.of("monitor"));

        Object rewritten = rewriter.rewrite("nodes", "alerter", nodeMap,
                "pipe-monitor", java.util.Set.of("monitor", "alerter"));

        @SuppressWarnings("unchecked")
        var result = (Map<String, Object>) rewritten;
        @SuppressWarnings("unchecked")
        var deps = (List<Object>) result.get("dependsOn");
        assertThat(deps).contains("pipe-monitor.monitor");
    }

    @Test
    void rewriter_preservesExternalDependency() {
        SectionContentRewriter rewriter = bridge.rewriter();

        var nodeMap = new LinkedHashMap<String, Object>();
        nodeMap.put("type", "monitor");
        nodeMap.put("dependsOn", List.of("${var.watched_node_id}"));

        Object rewritten = rewriter.rewrite("nodes", "monitor", nodeMap,
                "pipe-monitor", java.util.Set.of("monitor", "alerter"));

        @SuppressWarnings("unchecked")
        var result = (Map<String, Object>) rewritten;
        @SuppressWarnings("unchecked")
        var deps = (List<Object>) result.get("dependsOn");
        assertThat(deps).contains("${var.watched_node_id}");
    }

    @Test
    void rewriter_handlesOptionalDependency() {
        SectionContentRewriter rewriter = bridge.rewriter();

        var nodeMap = new LinkedHashMap<String, Object>();
        nodeMap.put("type", "alerter");
        nodeMap.put("dependsOn", List.of(Map.of("node", "monitor", "optional", true)));

        Object rewritten = rewriter.rewrite("nodes", "alerter", nodeMap,
                "pipe-monitor", java.util.Set.of("monitor", "alerter"));

        @SuppressWarnings("unchecked")
        var result = (Map<String, Object>) rewritten;
        @SuppressWarnings("unchecked")
        var deps = (List<Object>) result.get("dependsOn");
        @SuppressWarnings("unchecked")
        var optDep = (Map<String, Object>) deps.get(0);
        assertThat(optDep.get("node")).isEqualTo("pipe-monitor.monitor");
        assertThat(optDep.get("optional")).isEqualTo(true);
    }

    @Test
    void rewriter_doesNotTouchNonNodesSections() {
        SectionContentRewriter rewriter = bridge.rewriter();

        var ruleMap = new LinkedHashMap<String, Object>();
        ruleMap.put("match", Map.of("mon", Map.of("type", "monitor")));

        Object rewritten = rewriter.rewrite("rules", "auto-alert", ruleMap,
                "pipe-monitor", java.util.Set.of("monitor", "alerter"));

        assertThat(rewritten).isSameAs(ruleMap);
    }

    @Test
    void fullExpand_aliasedNodesInTypedResult() {
        var monitoringModule = new YamlModule("monitoring",
                Map.of("watched_node_id",
                        YamlModuleParameter.builder().type(
                                io.casehub.yaml.core.module.ParameterType.STRING)
                                .required().build()),
                Map.of(),
                Map.of("nodes", Map.of(
                        "monitor", Map.<String, Object>of(
                                "type", "monitor",
                                "dependsOn", List.of("${var.watched_node_id}"),
                                "spec", Map.of("target", "${var.watched_node_id}")),
                        "alerter", Map.<String, Object>of(
                                "type", "alerter",
                                "dependsOn", List.of("monitor"),
                                "spec", Map.of("email", "${var.alert_email}")))));

        var imports = List.of(new YamlImport("monitoring", "pipe-monitor", null,
                Map.of("watched_node_id", "warehouse-sink")));

        TypedExpandedModule<DesiredStateModuleContent> result =
                ModuleExpander.expand(imports,
                        Map.of("monitoring", monitoringModule),
                        DesiredStateModuleContent.empty(), bridge);

        assertThat(result.content().nodes()).containsKey("pipe-monitor.monitor");
        assertThat(result.content().nodes()).containsKey("pipe-monitor.alerter");
        assertThat(result.moduleScopes()).containsKey("pipe-monitor");
    }
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=DesiredStateModuleBridgeTest`
Expected: Compilation failure — `DesiredStateModuleBridge` does not exist yet.

- [ ] **Step 6: Implement DesiredStateModuleBridge**

Create `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/DesiredStateModuleBridge.java`:

```java
package io.casehub.desiredstate.yaml;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.type.MapType;
import io.casehub.desiredstate.yaml.model.YamlInvariant;
import io.casehub.desiredstate.yaml.model.YamlNode;
import io.casehub.desiredstate.yaml.model.YamlRule;
import io.casehub.yaml.core.module.ModuleBridge;
import io.casehub.yaml.core.module.SectionContentRewriter;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;

public final class DesiredStateModuleBridge implements ModuleBridge<DesiredStateModuleContent> {

    private static final String NODES = "nodes";
    private static final String RULES = "rules";
    private static final String INVARIANTS = "invariants";

    private final ObjectMapper mapper;

    public DesiredStateModuleBridge(ObjectMapper mapper) {
        this.mapper = mapper;
    }

    @Override
    public DesiredStateModuleContent fromSections(Map<String, Map<String, Object>> sections) {
        return new DesiredStateModuleContent(
                convertSection(sections.getOrDefault(NODES, Map.of()), YamlNode.class),
                convertSection(sections.getOrDefault(RULES, Map.of()), YamlRule.class),
                convertSection(sections.getOrDefault(INVARIANTS, Map.of()), YamlInvariant.class));
    }

    @Override
    public Map<String, Map<String, Object>> toSections(DesiredStateModuleContent content) {
        Map<String, Map<String, Object>> sections = new LinkedHashMap<>();
        if (!content.nodes().isEmpty()) {
            sections.put(NODES, toRawSection(content.nodes()));
        }
        if (!content.rules().isEmpty()) {
            sections.put(RULES, toRawSection(content.rules()));
        }
        if (!content.invariants().isEmpty()) {
            sections.put(INVARIANTS, toRawSection(content.invariants()));
        }
        return sections;
    }

    @Override
    public SectionContentRewriter rewriter() {
        return (sectionName, entryKey, entryValue, alias, moduleKeys) -> {
            if (!NODES.equals(sectionName)) { return entryValue; }
            if (!(entryValue instanceof Map<?, ?> rawMap)) { return entryValue; }

            @SuppressWarnings("unchecked")
            Map<String, Object> nodeMap = (Map<String, Object>) rawMap;
            Object depsObj = nodeMap.get("dependsOn");
            if (!(depsObj instanceof List<?> deps)) { return entryValue; }

            List<Object> rewritten = new ArrayList<>(deps.size());
            for (Object dep : deps) {
                String depId = depNodeId(dep);
                boolean isOptional = isOptionalDep(dep);

                if (moduleKeys.contains(depId)) {
                    String aliased = alias + "." + depId;
                    rewritten.add(isOptional
                            ? Map.of("node", aliased, "optional", true)
                            : aliased);
                } else {
                    rewritten.add(dep);
                }
            }

            Map<String, Object> result = new LinkedHashMap<>(nodeMap);
            result.put("dependsOn", rewritten);
            return result;
        };
    }

    private <V> Map<String, V> convertSection(Map<String, Object> raw, Class<V> type) {
        Map<String, V> result = new LinkedHashMap<>();
        for (Map.Entry<String, Object> entry : raw.entrySet()) {
            result.put(entry.getKey(), mapper.convertValue(entry.getValue(), type));
        }
        return result;
    }

    @SuppressWarnings("unchecked")
    private <V> Map<String, Object> toRawSection(Map<String, V> typed) {
        Map<String, Object> raw = new LinkedHashMap<>();
        for (Map.Entry<String, V> entry : typed.entrySet()) {
            raw.put(entry.getKey(), mapper.convertValue(entry.getValue(), Map.class));
        }
        return raw;
    }

    private static String depNodeId(Object dep) {
        if (dep instanceof String s) { return s; }
        if (dep instanceof Map<?, ?> m) { return (String) m.get("node"); }
        return dep.toString();
    }

    private static boolean isOptionalDep(Object dep) {
        if (dep instanceof Map<?, ?> m) {
            return Boolean.TRUE.equals(m.get("optional"));
        }
        return false;
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=DesiredStateModuleBridgeTest`
Expected: All 7 tests PASS.

- [ ] **Step 8: Run full yaml/runtime test suite for regression**

Run: `mvn --batch-mode test -pl yaml/runtime`
Expected: All 108 tests PASS.

- [ ] **Step 9: Commit**

```bash
git add yaml/runtime/pom.xml yaml/deployment/pom.xml \
  yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/DesiredStateModuleContent.java \
  yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/DesiredStateModuleBridge.java \
  yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/DesiredStateModuleBridgeTest.java
git commit -m "feat(#126): add DesiredStateModuleBridge — ModuleBridge<T> for typed module expansion"
```

---

## Batch 2: Migration — wire yaml-core, delete local types

### Task 2: Migrate YamlGraphRecorder + processor to yaml-core module types

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlGraph.java` — import yaml-core YamlImport
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java` — use typed expand
- Modify: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java` — yaml-core types, YamlCoreJacksonModule, resolveExtensions, remove validateImports
- Delete: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/ModuleExpander.java` (use `ide_refactor_safe_delete`)
- Delete: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlModule.java` (use `ide_refactor_safe_delete`)
- Delete: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlModuleParameter.java` (use `ide_refactor_safe_delete`)
- Delete: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlImport.java` (use `ide_refactor_safe_delete`)
- Delete: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlModuleFile.java` (use `ide_refactor_safe_delete`)
- Delete: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/ModuleExpanderTest.java` (use `ide_refactor_safe_delete`)
- Delete: `yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/YamlModuleValidationTest.java` (use `ide_refactor_safe_delete`)
- Modify: `examples/webapp-yaml/src/test/java/io/casehub/desiredstate/example/webapp/yaml/Tutorial3ScaleAndComposeTest.java` — yaml-core types + YamlCoreJacksonModule

**Interfaces:**
- Consumes: `DesiredStateModuleBridge` and `DesiredStateModuleContent` from Task 1
- Consumes: `io.casehub.yaml.core.module.*` — YamlModule, YamlImport, YamlModuleFile, ModuleExpander, YamlCoreJacksonModule

- [ ] **Step 1: Update YamlGraph to use yaml-core YamlImport**

In `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlGraph.java`:

Add import: `import io.casehub.yaml.core.module.YamlImport;`

The record field `List<YamlImport> imports` now resolves to yaml-core's type.
Jackson deserialization is unaffected — the records are structurally identical.

- [ ] **Step 2: Update YamlGraphRecorder — module expansion call site**

In `YamlGraphRecorder.createYamlGoalCompiler()` (the 4-arg overload that accepts
`availableModules`), change the module expansion block:

**Change parameter type** of `availableModules` from
`Map<String, io.casehub.desiredstate.yaml.model.YamlModule>` to
`Map<String, io.casehub.yaml.core.module.YamlModule>`.

**Replace the module expansion block** (~lines 88-97):

```java
// Old:
ModuleExpander.ExpandedGraph moduleExpanded = ModuleExpander.expand(
        yamlGraph.imports(), availableModules, effectiveNodes);
effectiveNodes = moduleExpanded.expandedNodes();
moduleScopes = moduleExpanded.moduleScopes();
promotedRules = moduleExpanded.promotedRules();
promotedInvariants = moduleExpanded.promotedInvariants();

// New:
DesiredStateModuleBridge bridge = new DesiredStateModuleBridge(mapper);
DesiredStateModuleContent existingContent = new DesiredStateModuleContent(
        effectiveNodes, Map.of(), Map.of());
io.casehub.yaml.core.module.TypedExpandedModule<DesiredStateModuleContent> moduleExpanded =
        io.casehub.yaml.core.module.ModuleExpander.expand(
                yamlGraph.imports(), availableModules, existingContent, bridge);
effectiveNodes = moduleExpanded.content().nodes();
moduleScopes = moduleExpanded.moduleScopes();
promotedRules = moduleExpanded.content().rules();
promotedInvariants = moduleExpanded.content().invariants();
```

Note: existing content passes empty rules/invariants — graph-level rules and
invariants are merged AFTER module expansion in the recorder, not through the
expander. The expander's `content().rules()` / `content().invariants()` return
only module-promoted rules/invariants (matching the old `promotedRules` / `promotedInvariants`).

- [ ] **Step 3: Update YamlDesiredStateProcessor — discoverModules()**

Replace `discoverModules()` to use yaml-core types + YamlCoreJacksonModule + resolveExtensions:

```java
private Map<String, io.casehub.yaml.core.module.YamlModule> discoverModules(
        ObjectMapper mapper) throws IOException, java.net.URISyntaxException {

    ObjectMapper moduleMapper = mapper.copy();
    moduleMapper.registerModule(new io.casehub.yaml.jackson.YamlCoreJacksonModule());

    String prefix = "META-INF/desiredstate/modules/";
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    java.util.Enumeration<java.net.URL> resources = cl.getResources(prefix);

    List<io.casehub.yaml.core.module.YamlModuleFile> moduleFiles = new ArrayList<>();
    Set<String> seen = new HashSet<>();

    while (resources.hasMoreElements()) {
        java.net.URL dirUrl = resources.nextElement();
        if ("file".equals(dirUrl.getProtocol())) {
            java.io.File dir = new java.io.File(dirUrl.toURI().getPath());
            if (dir.isDirectory()) {
                java.io.File[] yamlFiles = dir.listFiles((d, name) ->
                        name.endsWith(".yaml") || name.endsWith(".yml"));
                if (yamlFiles != null) {
                    for (java.io.File f : yamlFiles) {
                        if (seen.add(f.getName())) {
                            try (InputStream is = f.toURI().toURL().openStream()) {
                                io.casehub.yaml.core.module.YamlModuleFile moduleFile =
                                        moduleMapper.readValue(is,
                                                io.casehub.yaml.core.module.YamlModuleFile.class);
                                moduleFiles.add(moduleFile);
                            }
                        }
                    }
                }
            }
        }
    }

    return io.casehub.yaml.core.module.ModuleExpander.resolveExtensions(moduleFiles);
}
```

- [ ] **Step 4: Remove validateImports() from YamlDesiredStateProcessor**

Delete the `validateImports()` static method (~lines 692-733) and its call site
(~line 91: `if (!yamlGraph.imports().isEmpty()) { validateImports(...); }`).

yaml-core's `ModuleExpander.expand()` handles all import validation internally.

- [ ] **Step 5: Update processor's recorder call to use yaml-core YamlModule type**

The `createYamlGoalCompiler()` calls in the processor already pass `availableModules` —
update the type from local `YamlModule` to `io.casehub.yaml.core.module.YamlModule`.

- [ ] **Step 6: Update Tutorial3ScaleAndComposeTest**

Change module loading to use yaml-core types + `YamlCoreJacksonModule`:

```java
// Old:
ObjectMapper yamlMapper = new ObjectMapper(new YAMLFactory());
YamlModuleFile moduleFile = yamlMapper.readValue(modIs, YamlModuleFile.class);
YamlModule module = moduleFile.toModule();

// New:
ObjectMapper yamlMapper = new ObjectMapper(new YAMLFactory()).findAndRegisterModules();
yamlMapper.registerModule(new io.casehub.yaml.jackson.YamlCoreJacksonModule());
io.casehub.yaml.core.module.YamlModuleFile moduleFile =
        yamlMapper.readValue(modIs, io.casehub.yaml.core.module.YamlModuleFile.class);
io.casehub.yaml.core.module.YamlModule module = moduleFile.toModule();
```

Update the `modules` map type from `Map<String, YamlModule>` (local) to
`Map<String, io.casehub.yaml.core.module.YamlModule>`.

- [ ] **Step 7: Delete local module types and tests**

Use `ide_refactor_safe_delete` for each file:
1. `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/ModuleExpander.java`
2. `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlModule.java`
3. `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlModuleParameter.java`
4. `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlImport.java`
5. `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlModuleFile.java`
6. `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/ModuleExpanderTest.java`
7. `yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/YamlModuleValidationTest.java`

If `ide_refactor_safe_delete` reports remaining usages, resolve them first.

- [ ] **Step 8: Run full build to verify**

Run: `mvn --batch-mode install`
Expected: All 28 modules build successfully. All tests pass.

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "chore(#126): migrate ModuleExpander to casehub-yaml-core + delete local module types"
```

---

## Batch 3: Example updates

### Task 3: Update YAML module examples with constraint declarations

**Files:**
- Modify: `examples/pipeline-yaml/src/main/resources/META-INF/desiredstate/modules/monitoring.yaml`
- Modify: `examples/webapp-yaml/src/main/resources/META-INF/desiredstate/modules/order-notifications.yaml`

**Interfaces:**
- Consumes: yaml-core `ParameterType`, `ParameterValidator` constraints (minLength, pattern, etc.)

- [ ] **Step 1: Update monitoring.yaml with constraints**

```yaml
module:
  name: monitoring
  parameters:
    watched_node_id:
      type: string
      required: true
      minLength: 1
      pattern: "^[a-z][a-z0-9-]*$"
      constraintDescription: "Node ID: lowercase alphanumeric with hyphens, starting with a letter"
    alert_email:
      type: string
      default: "ops@example.com"
      pattern: "^[^@]+@[^@]+\\.[^@]+$"
      constraintDescription: "Must be a valid email address"

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

- [ ] **Step 2: Update order-notifications.yaml with constraints**

```yaml
module:
  name: order-notifications
  parameters:
    watched_step:
      type: string
      required: true
      minLength: 1
      pattern: "^[a-z][a-z0-9-]*$"
      constraintDescription: "Node ID of the order step to watch"
    email_template:
      type: string
      default: "standard"
      allowedValues: ["standard", "premium", "minimal"]
      constraintDescription: "Email template style"

nodes:
  email:
    type: notification
    dependsOn: ["${var.watched_step}"]
    spec:
      channel: email
      target: "${var.watched_step}"

  sms:
    type: notification
    dependsOn: ["${var.watched_step}"]
    spec:
      channel: sms
      target: "${var.watched_step}"

invariants:
  notification-has-target:
    match:
      notif: { type: notification }
    directDep:
      target: { type: "*", of: notif, direction: DEPENDENCIES }
```

- [ ] **Step 3: Run example tests**

Run: `mvn --batch-mode test -pl examples/pipeline-yaml,examples/webapp-yaml`
Expected: All example tests PASS.

- [ ] **Step 4: Run full build for final verification**

Run: `mvn --batch-mode install`
Expected: All 28 modules build. All tests pass.

- [ ] **Step 5: Commit**

```bash
git add examples/pipeline-yaml/src/main/resources/META-INF/desiredstate/modules/monitoring.yaml \
  examples/webapp-yaml/src/main/resources/META-INF/desiredstate/modules/order-notifications.yaml
git commit -m "feat(#126): add parameter constraints to YAML module examples"
```

---

## References

- [2026-09-04-module-migration-design.md] — design spec this plan implements
- [2026-09-02-yaml-core-migration-context.md] — regression analysis (concerns 10-12)
- [decisions.md] — D2 (full bridge), D3 (ObjectMapper.convertValue)
- `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/ModuleExpander.java` — local expander (deletion target)
- `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java:92` — primary call site
- `yaml/deployment/.../YamlDesiredStateProcessor.java:252` — module discovery
- `io.casehub.yaml.core.module.ModuleBridge` — platform interface
- `io.casehub.yaml.core.module.ModuleExpander` — platform expander
- `io.casehub.yaml.jackson.YamlCoreJacksonModule` — dynamic section capture
- GE-20260602-a4d290 — ObjectMapper needs findAndRegisterModules()
- platform#270 — ModuleBridge<T>
- platform#269 — module extension
- casehubio/casehub-desiredstate#126
- casehubio/casehub-desiredstate#128
