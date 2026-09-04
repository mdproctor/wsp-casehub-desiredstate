# Adopt yaml-core Module System + Parameter Validation

**Date:** 2026-09-04
**Issue:** casehubio/casehub-desiredstate#126
**Branch:** issue-128-migrate-yaml-core (covers #128, #126)
**Depends on:** platform#270 (ModuleBridge<T>), platform#269 (module extension) — both landed

---

## 1. Goal

Replace desiredstate's local module system (`ModuleExpander`, `YamlModule`, `YamlModuleParameter`,
`YamlImport`, `YamlModuleFile`) with yaml-core's shared module primitives. Wire parameter constraint
validation and module extension. Delete local copies.

This is the third and final migration batch on this branch — #128 migrated VariableResolver and
ForEachExpander; #126 completes the yaml-core adoption by migrating the module layer.

---

## 2. Platform API Surface (yaml-core)

All prerequisites landed:

| API | Purpose |
|-----|---------|
| `ModuleBridge<T>` | Domain-typed content conversion — `fromSections()`, `toSections()`, `rewriter()`, `deriveOutputs()` |
| `TypedExpandedModule<T>` | Typed expansion result — `T content`, moduleScopes, importConditions, moduleOutputs |
| `ModuleExpander.expand(imports, modules, content, bridge)` | Typed expansion overload |
| `ModuleExpander.resolveExtensions(List<YamlModuleFile>)` | Module inheritance resolution |
| `YamlModuleFileBuilder` + `@JsonAnySetter` | Dynamic section capture — top-level YAML keys as sections |
| `YamlCoreJacksonModule` | Jackson module registering `YamlModuleFileMixin` |
| `ParameterValidator` | Constraint validation (minLength, maxLength, pattern, minimum, maximum, allowedValues) |
| `ParameterType` enum | Typed parameters with case-insensitive deserialization |
| `YamlModuleOutput` | Typed outputs with template resolution |
| `YamlModuleHeader.extendsModule()` | Module extension declaration |

---

## 3. New Types in Desiredstate

### 3.1 DesiredStateModuleContent

```java
package io.casehub.desiredstate.yaml;

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

Domain-typed container for desiredstate's module content. Replaces the typed fields that were
previously on the local `YamlModule` record.

### 3.2 DesiredStateModuleBridge

```java
package io.casehub.desiredstate.yaml;

public final class DesiredStateModuleBridge implements ModuleBridge<DesiredStateModuleContent> {

    private final ObjectMapper mapper;

    public DesiredStateModuleBridge(ObjectMapper mapper) {
        this.mapper = mapper;
    }

    @Override
    public DesiredStateModuleContent fromSections(Map<String, Map<String, Object>> sections) {
        // Deserialize each section's entries via mapper.convertValue()
        // "nodes" → Map<String, YamlNode>
        // "rules" → Map<String, YamlRule>
        // "invariants" → Map<String, YamlInvariant>
    }

    @Override
    public Map<String, Map<String, Object>> toSections(DesiredStateModuleContent content) {
        // Convert typed maps back to raw section maps via mapper.convertValue()
        // Each typed entry (YamlNode, YamlRule, YamlInvariant) → Map<String, Object>
    }

    @Override
    public SectionContentRewriter rewriter() {
        // Dependency alias-prefixing for the "nodes" section:
        // - If section is "nodes", rewrite dependsOn entries
        // - Internal module references get alias prefix
        // - External references preserved as-is
    }
}
```

Uses Jackson `ObjectMapper.convertValue()` for typed conversion (D3). The mapper is already
available at both call sites (YamlGraphRecorder has it, YamlDesiredStateProcessor has it).

---

## 4. Changes to Existing Code

### 4.1 YamlGraphRecorder

**Current flow (lines ~88-97):**
```java
ModuleExpander.ExpandedGraph moduleExpanded = ModuleExpander.expand(
        yamlGraph.imports(), availableModules, effectiveNodes);
effectiveNodes = moduleExpanded.expandedNodes();
moduleScopes = moduleExpanded.moduleScopes();
promotedRules = moduleExpanded.promotedRules();
promotedInvariants = moduleExpanded.promotedInvariants();
```

**New flow:**
```java
DesiredStateModuleBridge bridge = new DesiredStateModuleBridge(mapper);
DesiredStateModuleContent existingContent = new DesiredStateModuleContent(
        effectiveNodes, yamlGraph.rules(), yamlGraph.invariants());

TypedExpandedModule<DesiredStateModuleContent> moduleExpanded =
        ModuleExpander.expand(yamlGraph.imports(), availableModules,
                              existingContent, bridge);

effectiveNodes = moduleExpanded.content().nodes();
moduleScopes = moduleExpanded.moduleScopes();
promotedRules = moduleExpanded.content().rules();
promotedInvariants = moduleExpanded.content().invariants();
```

Key changes:
- `ModuleExpander.expand()` → typed overload with bridge
- `ExpandedGraph.expandedNodes()` → `content().nodes()`
- `ExpandedGraph.promotedRules()` → `content().rules()`
- `ExpandedGraph.promotedInvariants()` → `content().invariants()`
- Import conditions available via `moduleExpanded.importConditions()`

The rest of YamlGraphRecorder (forEach expansion, spec resolution, rule/invariant evaluation)
is unchanged — it already works with `Map<String, YamlNode>` etc.

### 4.2 YamlDesiredStateProcessor (deployment)

**Module discovery — `discoverModules()`:**
- Register `YamlCoreJacksonModule` on the ObjectMapper for dynamic section capture
- Deserialize module files as yaml-core `YamlModuleFile` (not local)
- Call `ModuleExpander.resolveExtensions(moduleFiles)` to resolve `extends` inheritance
- Returns `Map<String, YamlModule>` (yaml-core type)

**Import validation — `validateImports()`:**
- The deployment processor's manual validation (unknown modules, missing required params,
  duplicate aliases, reserved separators) becomes **redundant** — yaml-core's `ModuleExpander`
  validates all of this internally via `validateImports()` and `ParameterValidator.validateOrThrow()`.
- Remove the local `validateImports()` method.
- Build-time validation is still provided — the recorder's `createYamlGoalCompiler()` runs at
  `RUNTIME_INIT`, and expansion failures surface as build errors.

**Recorder call site:**
- Pass `Map<String, YamlModule>` (yaml-core type) to `createYamlGoalCompiler()`.
- The recorder signature changes: `availableModules` parameter type changes from
  `Map<String, io.casehub.desiredstate.yaml.model.YamlModule>` to
  `Map<String, io.casehub.yaml.core.module.YamlModule>`.

### 4.3 YamlModuleFile changes (local model)

**Delete entirely.** Replaced by:
- `io.casehub.yaml.core.module.YamlModuleFile` for deserialization (with `YamlCoreJacksonModule`)
- `io.casehub.yaml.core.module.YamlModule` for the module model

### 4.4 YAML format

**No changes to existing module YAML files.** The `YamlCoreJacksonModule` registers
`YamlModuleFileBuilder` with `@JsonAnySetter`, which captures top-level `nodes:`, `rules:`,
`invariants:` keys as sections automatically. Existing files like:

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

...continue to work unchanged. The `type: string` lowercase form is accepted via
case-insensitive `ParameterType` deserialization.

**New capabilities available** (optional — examples can demonstrate):
- Parameter constraints: `minLength`, `maxLength`, `pattern`, `minimum`, `maximum`, `allowedValues`
- Module outputs: `outputs:` section in module header
- Module extension: `extends: parent-module-name` in module header
- Cross-module references: `${module.alias.outputName}` in parameter values

---

## 5. Deletions

| File | Reason |
|------|--------|
| `yaml/runtime/.../ModuleExpander.java` | Replaced by yaml-core `ModuleExpander` with `ModuleBridge<T>` |
| `yaml/runtime/.../model/YamlModule.java` | Replaced by `io.casehub.yaml.core.module.YamlModule` |
| `yaml/runtime/.../model/YamlModuleParameter.java` | Replaced by `io.casehub.yaml.core.module.YamlModuleParameter` |
| `yaml/runtime/.../model/YamlModuleFile.java` | Replaced by `io.casehub.yaml.core.module.YamlModuleFile` |
| `yaml/runtime/.../model/YamlImport.java` | Replaced by `io.casehub.yaml.core.module.YamlImport` |
| `yaml/runtime/...test.../ModuleExpanderTest.java` | Replaced by tests using yaml-core's expand with bridge |
| `yaml/deployment/...test.../YamlModuleValidationTest.java` | Validation now in yaml-core |
| Processor `validateImports()` method | Validation now in yaml-core |

---

## 6. Dependency Changes

### yaml/runtime pom.xml

- Add `casehub-yaml-jackson` dependency (for `YamlCoreJacksonModule`)
- `casehub-yaml-core` already present (added in #128)

### yaml/deployment pom.xml

- Add `casehub-yaml-jackson` dependency (for module file deserialization)

---

## 7. Test Strategy

### 7.1 DesiredStateModuleBridge tests

- `fromSections()` round-trips with `toSections()` — typed → raw → typed is identity
- `rewriter()` alias-prefixes internal dependsOn references, preserves external references
- `rewriter()` handles optional dependencies (Map form with `node` + `optional` keys)
- Empty sections produce empty typed content

### 7.2 Module expansion integration tests (replace ModuleExpanderTest)

- Alias-prefixed node IDs
- Internal dependency aliasing
- Cross-boundary dependency preservation
- Module scope parameter values
- Default parameter application
- Conditional import (when field) propagation
- Multiple imports — independent instances
- Existing nodes preserved alongside module nodes
- Promoted rules and invariants
- Parameter constraint validation failures surface correctly

### 7.3 Module extension tests

- Child inherits parent's nodes, rules, invariants
- Child overrides parent entries on key conflict
- Child adds new parameters alongside inherited ones
- Extension of unknown module fails
- Extension chain (A extends B extends C) fails
- Self-extension fails

### 7.4 Deployment processor tests

- Module discovery with `YamlCoreJacksonModule` registered
- Dynamic section capture (top-level keys, not `sections:` wrapper)
- `resolveExtensions()` integration
- Build-time validation via yaml-core (not local validateImports)
- Cross-surface namespace collision detection still works

### 7.5 Existing test suites (regression)

- All 108 yaml/runtime tests pass
- All 28 yaml/deployment tests pass
- Full 28-module build green
- Pipeline-yaml example compiles and tests pass
- Webapp-yaml example compiles and tests pass

---

## 8. Parity Verification

| Dimension | Before (local) | After (yaml-core) | Status |
|-----------|---------------|-------------------|--------|
| Node alias-prefixing | Local ModuleExpander | Bridge rewriter + yaml-core expand | Parity |
| Dependency rewriting | Inline in expand loop | SectionContentRewriter via bridge | Parity |
| Parameter defaults | Manual in expand | yaml-core resolveParameters | Parity |
| Parameter validation | None | ParameterValidator (minLength, maxLength, pattern, min, max, allowedValues) | **Improved** |
| Module outputs | None | YamlModuleOutput + cross-module refs | **Improved** |
| Module extension | None | resolveExtensions() with extends keyword | **Improved** |
| Import validation | Local validateImports | yaml-core validateImports + validateModuleRefs | **Improved** |
| YAML format | Top-level nodes/rules/invariants | Same — dynamic section capture | Parity |
| Type safety | Typed fields on local YamlModule | ModuleBridge<T> + DesiredStateModuleContent | Parity |
| Build-time validation | Deployment processor | yaml-core internal (RUNTIME_INIT surfaces errors) | Parity |

Zero regressions. Four improvements.

---

## References

- `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/ModuleExpander.java` — local expander (98 lines, deletion target)
- `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java:92` — primary call site
- `yaml/deployment/.../YamlDesiredStateProcessor.java:252` — module discovery
- `yaml/deployment/.../YamlDesiredStateProcessor.java:692` — local validateImports
- `io.casehub.yaml.core.module.ModuleBridge` — platform bridge interface
- `io.casehub.yaml.core.module.TypedExpandedModule` — platform typed result
- `io.casehub.yaml.core.module.ModuleExpander.resolveExtensions()` — extension resolution
- `io.casehub.yaml.jackson.YamlCoreJacksonModule` — dynamic section capture
- `specs/issue-128-migrate-yaml-core/2026-09-02-yaml-core-migration-context.md` — regression analysis (§2, concerns 10-12)
- `specs/issue-128-migrate-yaml-core/decisions.md` — D2 (full bridge), D3 (ObjectMapper.convertValue)
- platform#270 — ModuleBridge<T> implementation
- platform#269 — module extension implementation
- GE-20260602-a4d290 — ObjectMapper + YAMLFactory needs findAndRegisterModules()
