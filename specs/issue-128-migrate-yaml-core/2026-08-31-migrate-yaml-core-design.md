# Migrate yaml/runtime to casehub-yaml-core

**Date:** 2026-08-31
**Issue:** casehubio/casehub-desiredstate#128
**Status:** Draft

## Motivation

`casehub-platform/yaml-core` was extracted from desiredstate to provide shared
YAML primitives. Desiredstate still uses local copies of `VariableResolver`,
`ForEachExpander`, `Truthiness`, `IterationGroup`, and `UnresolvedVariableException`.
New YAML infrastructure (#126 ParameterValidator, platform#252 module system)
should build on yaml-core, not local copies. Migration first.

---

## Part 1: Dependency

Add `casehub-yaml-core` to `yaml/runtime/pom.xml`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-yaml-core</artifactId>
</dependency>
```

Version managed by the platform BOM.

---

## Part 2: VariableResolver Migration

### API delta

| Feature | Local | yaml-core |
|---------|-------|-----------|
| Source model | Hardcoded: `inlineVariables`, `Config`, `moduleScope` | Generic: `Map<String, VariableSource>` |
| Prefix dispatch | Hardcoded in `lookupVariable()` | Prefix map + `deferredPrefixes` |
| Deferred prefixes | `resolveTemplateString()` — only resolves `var.` | `Set<String> deferredPrefixes` — left as-is |
| Each context | `withEachContext(Map<String,String>)` | `withEachContext(Map<String,String>)` (same) |
| Module scope | `withModuleScope(Map<String,String>)` | `withScope(prefix, VariableSource)` |
| Exception | `resolver.UnresolvedVariableException(variableName, nodeContext, detail)` | `core.resolver.UnresolvedVariableException(variableName, elementContext, detail)` |

### VariableSource adapter

Create a factory method in `YamlGraphRecorder` (or a small utility class) that
builds a yaml-core `VariableResolver` from desiredstate's YAML context:

```java
private VariableResolver buildResolver(Map<String, String> inlineVariables,
                                        Config config) {
    VariableSource varSource = VariableSource.chain(
            inlineVariables::get,
            name -> config != null
                    ? config.getOptionalValue(name, String.class).orElse(null)
                    : null
    );
    return new VariableResolver(
            Map.of("var", varSource),
            Set.of("match", "fault"));
}
```

Module scope is added per-node via `resolver.withScope("var", chainedSource)`
where the chained source prepends the module scope:

```java
VariableResolver resolverForModule(VariableResolver base,
                                    Map<String, String> moduleScope) {
    VariableSource moduleSource = VariableSource.chain(
            moduleScope::get,
            base.prefixSources().get("var")  // fall through to base
    );
    return base.withScope("var", moduleSource);
}
```

Note: `VariableResolver.prefixSources()` is not exposed. Alternative: capture
the base `VariableSource` at construction time and pass it through.

Simpler approach — construct the full chain at each module-scoped call site:

```java
VariableSource moduleChainedSource = VariableSource.chain(
        moduleScope::get,
        inlineVariables::get,
        name -> config != null
                ? config.getOptionalValue(name, String.class).orElse(null)
                : null
);
VariableResolver moduleResolver = new VariableResolver(
        Map.of("var", moduleChainedSource),
        Set.of("match", "fault"));
```

### `resolveTemplateString/Map/List` elimination

The local `resolveTemplateString()` only resolves `var.` prefixed variables,
leaving `match.`, `fault.`, and `each.` as-is. With yaml-core, `match` and
`fault` are in `deferredPrefixes` — they are left as-is by the standard
`resolveString()`. So `resolveTemplateString()` callers can use the standard
`resolveString()` with the deferred-prefix resolver.

`each.` is handled internally by yaml-core when an each context is set, and
left as-is when no each context is active (since there's no matching prefix
source). Actually, yaml-core throws on unknown prefixes when they're not in
`deferredPrefixes`. So `each` must also be deferred when no forEach context
is active:

```java
Set.of("match", "fault")  // deferred — each is handled internally by the resolver
```

yaml-core's `VariableResolver` handles `each.` as a special prefix internally
(not via the prefix map), so it doesn't need to be in the deferred set.

### Callers to update

| File | Usage | Change |
|------|-------|--------|
| `YamlGraphRecorder` | Constructs resolver, calls `resolveString/Map/List`, `resolveTemplateString/Map/List` | Construct yaml-core resolver, use standard resolve methods |
| `YamlRuleConverter` | Calls `resolveTemplateString/Map/List` | Use standard `resolveString/Map/List` with deferred resolver |
| `HookResolver` | Calls `resolveTemplateString/Map/List` | Same |
| `ForEachExpander` | Calls `resolveString/Map`, `withEachContext` | Deleted (Part 3) |

### Files deleted

- `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/resolver/VariableResolver.java`
- `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/resolver/UnresolvedVariableException.java`
- `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/resolver/VariableResolverTest.java` (replaced by yaml-core tests)

---

## Part 3: ForEachExpander Migration (D1)

### Current structure

The local `ForEachExpander.expand()` does everything in one pass:
1. Resolve forEach groups → stamp IDs
2. Evaluate `when` conditions
3. Resolve specs via VariableResolver
4. Convert specs to `NodeSpec` via `ObjectMapper` + `NodeSpecRegistry`
5. Create `DesiredNode` instances
6. Wire dependencies with forEach-aware logic

Returns `ExpandedNodes(List<DesiredNode>, List<Dependency>, Set<String>)`.

### New structure — two-pass

**Pass 1 — Generic expansion (yaml-core):**

Implement `ForEachAdapter<YamlNode>`:

```java
public class YamlNodeForEachAdapter implements ForEachAdapter<YamlNode> {
    @Override
    public YamlNode stamp(YamlNode template, String stampedId, VariableResolver scopedResolver) {
        // Return a new YamlNode with resolved spec (variable substitution only)
        Map<String, Object> resolvedSpec = scopedResolver.resolveMap(template.spec(), stampedId);
        return new YamlNode(template.type(), resolvedSpec, template.dependsOn(),
                template.humanGating(), template.when(), null,
                template.provision(), template.deprovision());
    }

    @Override
    public Object getForEach(YamlNode element) { return element.forEach(); }

    @Override
    public String getId(YamlNode element) { /* not used by ForEachExpander */ return null; }

    @Override
    public String getWhen(YamlNode element) { return element.when(); }
}
```

Call yaml-core's `ForEachExpander.expand()`:

```java
ExpansionResult<YamlNode> expanded = ForEachExpander.expand(
        yamlNodes, iterationGroups, resolver, adapter, maxExpansion);
```

Returns `ExpansionResult<YamlNode>` — stamped YamlNodes with resolved specs,
forEach templates expanded, conditional nodes excluded.

**Pass 2 — Domain transformation (YamlGraphRecorder):**

After expansion, iterate the expanded elements to:
1. Convert each `YamlNode` spec to `NodeSpec` via `NodeSpecRegistry` + `ObjectMapper`
2. Create `DesiredNode` instances
3. Wire dependencies with forEach-aware ID rewriting

The dependency wiring logic from the local `ForEachExpander` moves into
`YamlGraphRecorder` as a private method. This is the only non-trivial code
move — the forEach-aware dependency stamping (`nodeId.value` → `nodeId.value.stampedValue`).

### IterationGroup mapping

`YamlIterationGroup` → `io.casehub.yaml.core.foreach.IterationGroup`. Same
record shape (`as`, `in`, `inAsList()`). Replace imports and delete the local copy.

### Files deleted

- `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/ForEachExpander.java`
- `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlIterationGroup.java`
- `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/ForEachExpanderTest.java` (rewritten against new structure)

---

## Part 4: Truthiness Migration

Replace `YamlGraphRecorder.isTruthy()` and `ForEachExpander.isTruthy()` with
`io.casehub.yaml.core.condition.Truthiness.isTruthy()`.

Delete the local `isTruthy()` static method from `YamlGraphRecorder`.

---

## Part 5: Testing Strategy

### Preserved test coverage

All existing behavior must pass through the migration. The tests change
in construction (new resolver construction, two-pass expansion) but assert
the same outcomes.

| Test class | Change |
|-----------|--------|
| `VariableResolverTest` | Rewrite to construct yaml-core resolver with VariableSource chain |
| `ForEachExpanderTest` | Rewrite to use yaml-core expander + YamlNodeForEachAdapter + domain pass |
| `YamlRuleConverterTest` | Update resolver construction |
| `ModuleExpanderTest` | No change — ModuleExpander doesn't use VariableResolver directly |
| `YamlGraphRecorderTest` (if exists) | Update resolver construction |
| All `examples/*` integration tests | Must pass unchanged — they test end-to-end behavior |

### New tests

- `YamlNodeForEachAdapterTest` — unit tests for the adapter: stamp produces resolved YamlNode, getForEach/getWhen delegate correctly

---

## References

- `io.casehub.yaml.core.resolver.VariableResolver` (platform/yaml-core) — target resolver
- `io.casehub.yaml.core.resolver.VariableSource` (platform/yaml-core) — source abstraction
- `io.casehub.yaml.core.foreach.ForEachExpander` (platform/yaml-core) — target expander
- `io.casehub.yaml.core.foreach.ForEachAdapter` (platform/yaml-core) — adapter interface
- `io.casehub.yaml.core.foreach.IterationGroup` (platform/yaml-core) — iteration group
- `io.casehub.yaml.core.condition.Truthiness` (platform/yaml-core) — truthiness utility
- `io.casehub.desiredstate.yaml.resolver.VariableResolver` — local copy being replaced
- `io.casehub.desiredstate.yaml.ForEachExpander` — local copy being replaced
- `io.casehub.desiredstate.yaml.YamlGraphRecorder` — primary caller, restructured
- `io.casehub.desiredstate.yaml.YamlRuleConverter` — caller, resolver construction updated
- `io.casehub.desiredstate.yaml.HookResolver` — caller, resolver construction updated
- decisions.md — D1 ForEachExpander restructure strategy
- GitHub #128 — this migration issue
- GitHub #126 — downstream consumer (parameter validation)
- GitHub platform#252 — downstream consumer (yaml-core module system)
