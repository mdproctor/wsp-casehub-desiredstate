# Migrate yaml/runtime to casehub-yaml-core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #128 — chore: migrate yaml/runtime to casehub-yaml-core
**Issue group:** #128

**Goal:** Replace desiredstate's local VariableResolver, ForEachExpander,
IterationGroup, and Truthiness with the shared versions from
`casehub-platform-yaml-core`.

**Architecture:** Add `casehub-platform-yaml-core` dependency. Create
`VariableSource` adapters for desiredstate's sources (inline variables,
MicroProfile Config, module scope). Implement `ForEachAdapter<YamlNode>`
for the generic expander. Restructure `YamlGraphRecorder` from single-pass
(expand + resolve) to two-pass (expand, then resolve). Delete local copies.

**Tech Stack:** Java 21, Quarkus, casehub-platform-yaml-core

## Global Constraints

- Foundation tier — yaml/runtime depends on api/ and platform, no upward deps
- Pre-release — no backward compatibility concerns
- `casehub-platform-yaml-core` artifact: `io.casehub:casehub-platform-yaml-core`
- `VariableSource` chain order for `var` prefix: moduleScope → inlineVariables → Config
- Deferred prefixes: `match`, `fault`

---

## Batch 1: Dependency + VariableResolver Migration

### Task 1: Add yaml-core dependency and migrate VariableResolver

**Files:**
- Modify: `yaml/runtime/pom.xml`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlRuleConverter.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/HookResolver.java`
- Delete: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/resolver/VariableResolver.java` (use `ide_refactor_safe_delete`)
- Delete: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/resolver/UnresolvedVariableException.java` (use `ide_refactor_safe_delete`)
- Modify: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/resolver/VariableResolverTest.java`
- Modify: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlRuleConverterTest.java`
- Test: all existing tests in `yaml/runtime`

**Interfaces:**
- Produces: yaml-core `VariableResolver` used throughout yaml/runtime. `buildResolver(Map<String,String>, Config)` factory method on `YamlGraphRecorder`. Module-scoped resolver via `VariableSource.chain()`.

- [ ] **Step 1: Add `casehub-platform-yaml-core` dependency to `yaml/runtime/pom.xml`**

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-yaml-core</artifactId>
</dependency>
```

- [ ] **Step 2: Add resolver factory method to YamlGraphRecorder**

Add a private static method that builds a yaml-core `VariableResolver`:

```java
private static io.casehub.yaml.core.resolver.VariableResolver buildResolver(
        Map<String, String> inlineVariables, org.eclipse.microprofile.config.Config config) {
    io.casehub.yaml.core.resolver.VariableSource varSource =
            io.casehub.yaml.core.resolver.VariableSource.chain(
                    inlineVariables::get,
                    name -> config != null
                            ? config.getOptionalValue(name, String.class).orElse(null)
                            : null);
    return new io.casehub.yaml.core.resolver.VariableResolver(
            Map.of("var", varSource), Set.of("match", "fault"));
}
```

Add a module-scoped resolver factory:

```java
private static io.casehub.yaml.core.resolver.VariableResolver resolverWithModuleScope(
        Map<String, String> inlineVariables, org.eclipse.microprofile.config.Config config,
        Map<String, String> moduleScope) {
    io.casehub.yaml.core.resolver.VariableSource varSource =
            io.casehub.yaml.core.resolver.VariableSource.chain(
                    moduleScope::get,
                    inlineVariables::get,
                    name -> config != null
                            ? config.getOptionalValue(name, String.class).orElse(null)
                            : null);
    return new io.casehub.yaml.core.resolver.VariableResolver(
            Map.of("var", varSource), Set.of("match", "fault"));
}
```

- [ ] **Step 3: Update all `new VariableResolver(...)` call sites in YamlGraphRecorder**

Replace `new VariableResolver(inlineVariables, null, null)` at lines 79 and 248 with
`buildResolver(inlineVariables, null)`.

- [ ] **Step 4: Update HookResolver import**

Change `import io.casehub.desiredstate.yaml.resolver.VariableResolver` to
`import io.casehub.yaml.core.resolver.VariableResolver`. Method signatures don't
change — `resolveString()` and `resolveMap()` have the same names and signatures.

- [ ] **Step 5: Update YamlRuleConverter import**

Change the VariableResolver import. `resolveTemplateString/Map/List` callers:
replace with standard `resolveString/Map/List` — the deferred prefixes (`match`,
`fault`) handle the "leave it as-is" behavior that `resolveTemplateString` was
providing.

- [ ] **Step 6: Update ForEachExpander's resolverForNode**

The local `ForEachExpander.resolverForNode()` calls `base.withModuleScope(scope)`.
Replace with yaml-core's `withScope("var", chainedSource)`:

```java
private static io.casehub.yaml.core.resolver.VariableResolver resolverForNode(
        io.casehub.yaml.core.resolver.VariableResolver base, String nodeId,
        Map<String, Map<String, String>> moduleScopes) {
    if (moduleScopes.isEmpty()) return base;
    int dot = nodeId.indexOf('.');
    if (dot < 0) return base;
    String prefix = nodeId.substring(0, dot);
    Map<String, String> scope = moduleScopes.get(prefix);
    if (scope == null) return base;
    return base.withScope("var",
            io.casehub.yaml.core.resolver.VariableSource.chain(scope::get, base));
}
```

Note: `base` itself is not a `VariableSource` — we need the underlying source.
Simpler: pass `inlineVariables` and `config` through to reconstruct:

```java
private static io.casehub.yaml.core.resolver.VariableResolver resolverForNode(
        io.casehub.yaml.core.resolver.VariableResolver base,
        Map<String, String> inlineVariables, Config config,
        String nodeId, Map<String, Map<String, String>> moduleScopes) {
    if (moduleScopes.isEmpty()) return base;
    int dot = nodeId.indexOf('.');
    if (dot < 0) return base;
    String prefix = nodeId.substring(0, dot);
    Map<String, String> scope = moduleScopes.get(prefix);
    if (scope == null) return base;
    return resolverWithModuleScope(inlineVariables, config, scope);
}
```

This method stays in `ForEachExpander` until Task 2 deletes it — at which point it
moves to `YamlGraphRecorder`.

- [ ] **Step 7: Replace `isTruthy` calls with `Truthiness.isTruthy()`**

In `YamlGraphRecorder`: replace `isTruthy(resolved)` with
`io.casehub.yaml.core.condition.Truthiness.isTruthy(resolved)`. Delete the local
`static boolean isTruthy(String)` method.

In `ForEachExpander`: replace `isTruthy(resolvedWhen)` with
`Truthiness.isTruthy(resolvedWhen)`. Delete the local `isTruthy()` delegation.

- [ ] **Step 8: Update VariableResolverTest**

Rewrite test construction to use yaml-core's `VariableResolver` with `VariableSource`:

```java
var resolver = new io.casehub.yaml.core.resolver.VariableResolver(
        Map.of("var", (io.casehub.yaml.core.resolver.VariableSource)
                Map.of("batch_size", "1000", "source_uri", "s3://data")::get),
        Set.of("match", "fault"));
```

Update exception type assertions from local `UnresolvedVariableException` to
`io.casehub.yaml.core.resolver.UnresolvedVariableException`.

- [ ] **Step 9: Delete local VariableResolver and UnresolvedVariableException**

Use `ide_refactor_safe_delete` for both files. All references should already be
updated in Steps 3-8.

- [ ] **Step 10: Run all yaml/runtime tests**

Run: `mvn --batch-mode install -pl api,runtime,testing,annotations/runtime -DskipTests && mvn --batch-mode test -pl yaml/runtime`
Expected: All PASS

- [ ] **Step 11: Commit**

```bash
git add yaml/runtime/
git commit -m "chore(#128): migrate VariableResolver to casehub-yaml-core"
```

---

## Batch 2: ForEachExpander Migration + Restructure

### Task 2: Implement ForEachAdapter and restructure YamlGraphRecorder

**Files:**
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlNodeForEachAdapter.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java` (restructure expand call sites)
- Delete: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/ForEachExpander.java` (use `ide_refactor_safe_delete`)
- Delete: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlIterationGroup.java` (use `ide_refactor_safe_delete`)
- Modify: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/ForEachExpanderTest.java`
- Test: all existing tests in `yaml/runtime`

**Interfaces:**
- Consumes: yaml-core `VariableResolver` from Task 1; `ForEachAdapter<YamlNode>`, `ForEachExpander.expand()`, `IterationGroup`, `ExpansionResult<YamlNode>` from yaml-core
- Produces: `YamlNodeForEachAdapter implements ForEachAdapter<YamlNode>`; restructured `YamlGraphRecorder` with two-pass expansion

- [ ] **Step 1: Write failing test for YamlNodeForEachAdapter**

Create test that constructs the adapter and verifies `stamp()` produces a
YamlNode with resolved spec values:

```java
@Test
void stamp_resolvesSpecVariables() {
    var resolver = new VariableResolver(
            Map.of("var", (VariableSource) Map.of("target", "warehouse-sink")::get),
            Set.of("match", "fault"));

    var adapter = new YamlNodeForEachAdapter();
    var template = new YamlNode("monitor",
            Map.of("target", "${var.target}"),
            List.of(), null, null, null, null, null);

    YamlNode stamped = adapter.stamp(template, "mon-1", resolver);
    assertThat(stamped.spec().get("target")).isEqualTo("warehouse-sink");
    assertThat(stamped.forEach()).isNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlNodeForEachAdapterTest`
Expected: Compilation error — class doesn't exist

- [ ] **Step 3: Implement YamlNodeForEachAdapter**

```java
package io.casehub.desiredstate.yaml;

import io.casehub.desiredstate.yaml.model.YamlNode;
import io.casehub.yaml.core.foreach.ForEachAdapter;
import io.casehub.yaml.core.resolver.VariableResolver;

import java.util.Map;

public class YamlNodeForEachAdapter implements ForEachAdapter<YamlNode> {

    @Override
    public YamlNode stamp(YamlNode template, String stampedId,
                          VariableResolver scopedResolver) {
        Map<String, Object> resolvedSpec = scopedResolver.resolveMap(
                template.spec(), stampedId);
        return new YamlNode(template.type(), resolvedSpec, template.dependsOn(),
                template.humanGating(), null, null,
                template.provision(), template.deprovision());
    }

    @Override
    public Object getForEach(YamlNode element) {
        return element.forEach();
    }

    @Override
    public String getId(YamlNode element) {
        return null;
    }

    @Override
    public String getWhen(YamlNode element) {
        return element.when();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlNodeForEachAdapterTest`
Expected: PASS

- [ ] **Step 5: Replace YamlIterationGroup with yaml-core IterationGroup**

Update all imports of `io.casehub.desiredstate.yaml.model.YamlIterationGroup` to
`io.casehub.yaml.core.foreach.IterationGroup`.

Key locations:
- `YamlGraphRecorder.java` — `yamlGraph.iterations()` returns `Map<String, YamlIterationGroup>`.
  The `YamlGraph` record type needs updating too.
- `YamlDesiredState.java` (or `YamlGraph.java`) — the `iterations` field type
- `ForEachExpanderTest.java` — test construction

Check the `YamlGraph` model:

```java
// YamlGraph.iterations() return type changes from
//   Map<String, YamlIterationGroup>
// to
//   Map<String, IterationGroup>
```

Use `ide_refactor_safe_delete` to remove `YamlIterationGroup.java` after all
references are updated.

- [ ] **Step 6: Restructure YamlGraphRecorder — single-graph path**

Replace the `ForEachExpander.expand()` call at line 102 with yaml-core's
`ForEachExpander.expand()` + post-processing:

```java
// Pass 1: generic expansion
var adapter = new YamlNodeForEachAdapter();
Map<String, io.casehub.yaml.core.foreach.IterationGroup> coreGroups =
        yamlGraph.iterations() != null ? yamlGraph.iterations() : Map.of();
io.casehub.yaml.core.foreach.ExpansionResult<YamlNode> expanded =
        io.casehub.yaml.core.foreach.ForEachExpander.expand(
                effectiveNodes, coreGroups, resolver, adapter, 1000);

// Pass 2: domain transformation
List<DesiredNode> nodes = new ArrayList<>();
for (var stampedEntry : /* iterate expanded.elements() with their IDs */) {
    // ... resolve NodeSpec via registry + mapper, create DesiredNode
}
```

The challenge: yaml-core's `ExpansionResult<YamlNode>` returns `List<YamlNode>` —
stamped elements with resolved specs, but without their IDs (IDs are map keys in
the input, not on the element). The `stamp()` method receives the `stampedId` but
doesn't store it on the YamlNode (YamlNode has no ID field).

Solution: add an `id` field to `YamlNodeForEachAdapter.stamp()` output. Either:
a. Add a transient `stampedId` field to the stamped YamlNode (record change)
b. Return a wrapper record from the adapter

Simpler: keep a parallel map of stamped ID → YamlNode. The adapter's `stamp()`
is called with `stampedId` — capture it in a side-channel `Map<String, YamlNode>`
on the adapter (make it stateful for this expansion call):

```java
public class YamlNodeForEachAdapter implements ForEachAdapter<YamlNode> {
    private final Map<String, YamlNode> stampedById = new LinkedHashMap<>();

    // ... stamp() stores stampedById.put(stampedId, stamped) before returning

    public Map<String, YamlNode> stampedById() { return stampedById; }
}
```

After expansion, iterate `adapter.stampedById()` for domain transformation.

- [ ] **Step 7: Restructure YamlGraphRecorder — lifecycle path**

Same pattern for the lifecycle path at line 264. Replace `ForEachExpander.expand()`
with yaml-core expander + post-processing.

- [ ] **Step 8: Move dependency wiring logic from ForEachExpander to YamlGraphRecorder**

The local ForEachExpander's dependency wiring logic (lines 156-207) handles:
- Static-to-static deps
- ForEach-to-static deps (stamped copies all depend on same static target)
- ForEach-to-forEachSameGroup deps (paired by value)
- Static-to-forEach error (non-optional)

Move this as a private method on `YamlGraphRecorder`:

```java
private static List<Dependency> wireDependencies(
        Map<String, YamlNode> originalNodes,
        Map<String, String> nodeToGroup,
        Map<String, List<String>> groupValues,
        Set<String> excludedNodeIds,
        VariableResolver resolver) { ... }
```

- [ ] **Step 9: Delete local ForEachExpander**

Use `ide_refactor_safe_delete`. All call sites should now use yaml-core's expander.

- [ ] **Step 10: Rewrite ForEachExpanderTest**

Tests should construct yaml-core's `ForEachExpander` with `YamlNodeForEachAdapter`
and verify the same expansion outcomes. The test assertions stay the same — stamped
IDs, excluded nodes, expansion counts.

- [ ] **Step 11: Run all yaml/runtime tests**

Run: `mvn --batch-mode test -pl yaml/runtime`
Expected: All PASS

- [ ] **Step 12: Commit**

```bash
git add yaml/runtime/
git commit -m "chore(#128): migrate ForEachExpander to casehub-yaml-core + restructure YamlGraphRecorder"
```

---

## Batch 3: Downstream Verification

### Task 3: Full build + deployment module verification

**Files:** None — verification only

- [ ] **Step 1: Install yaml/runtime and run yaml/deployment tests**

Run: `mvn --batch-mode install -pl yaml/runtime -DskipTests && mvn --batch-mode test -pl yaml/deployment`
Expected: All PASS

- [ ] **Step 2: Run full project build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all 25 modules compile and pass

- [ ] **Step 3: Fix any downstream breakage**

If any module fails due to `YamlIterationGroup` or local `VariableResolver`
imports, update those imports. The ts-dsl modules and example projects may
reference these types transitively.

- [ ] **Step 4: Commit any fixes**

```bash
git add .
git commit -m "chore(#128): fix downstream imports after yaml-core migration"
```

---

## References

- [2026-08-31-migrate-yaml-core-design.md] — design spec this plan implements
- `io.casehub.yaml.core.resolver.VariableResolver` (platform/yaml-core) — target resolver
- `io.casehub.yaml.core.resolver.VariableSource` (platform/yaml-core) — source abstraction with `chain()` factory
- `io.casehub.yaml.core.foreach.ForEachExpander` (platform/yaml-core) — target expander
- `io.casehub.yaml.core.foreach.ForEachAdapter` (platform/yaml-core) — adapter interface
- `io.casehub.yaml.core.foreach.IterationGroup` (platform/yaml-core) — replaces `YamlIterationGroup`
- `io.casehub.yaml.core.condition.Truthiness` (platform/yaml-core) — replaces `YamlGraphRecorder.isTruthy()`
- `YamlGraphRecorder.java:65-202` — primary restructure target (single-graph)
- `YamlGraphRecorder.java:238-367` — secondary restructure target (lifecycle)
- `ForEachExpander.java:156-207` — dependency wiring logic to extract
- GitHub #128 — focal issue
- GitHub #126 — downstream consumer (parameter validation)
- GitHub platform#252 — downstream consumer (yaml-core module system)
