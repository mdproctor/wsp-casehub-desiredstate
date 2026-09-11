# Consolidate Step Pipeline Infrastructure into yaml-step-core

**Issue:** casehubio/casehub-ops#88 — Consolidate step pipeline infrastructure into yaml-core
**Date:** 2026-09-11
**Status:** Draft
**Depends on:** #87 (YAML plugin architecture — source of the step pipeline types)

---

## 1. Summary

Extract the generic step pipeline infrastructure from `casehub-desiredstate-plugin`
into a new platform module `casehub-platform-yaml-step-core`. Refactor
`PluginInterpolator` to delegate to yaml-core's `VariableResolver` via `StepContext`
implementing `VariableSource`. Move generic built-in primitives (`RestCallPrimitive`,
`JsonExtractPrimitive`, `AssertPrimitive`) to the new module. Keep domain-specific
types (`CompareStatePrimitive`, `YamlNodeSpec`, `YamlPluginProvisioner`,
`YamlPluginActualStateAdapter`, `PluginDescriptor`, RAS/CBR registrars) in
desiredstate.

## 2. Background

The YAML plugin architecture (#87) introduced a step pipeline system in
`casehub-desiredstate-plugin` that reimplements patterns already present in
`casehub-platform-yaml-core`:

| yaml-core | plugin (duplicate) |
|---|---|
| `VariableResolver` — prefix-based `${var.*}`, `${match.*}` with `VariableSource` | `PluginInterpolator` — prefix-based `${spec.*}`, `${auth.*}`, `${result.*}`, `${param.*}` |
| `ModuleBridge` — parameterized YAML expansion | `CompoundPrimitiveExpander` — parameterized YAML step expansion |
| `VariableSource.withScope()` — scoped resolution | `StepContext` — scoped binding accumulation |

The step pipeline SPI types (`StepPrimitive`, `StepResult`, `StepParameters`,
`StepContext`) are pure Java with zero domain imports. The executor, registry,
expander, and expression evaluator are similarly domain-free. Only the
domain-specific wiring (`YamlPluginProvisioner`, `CompareStatePrimitive`,
`executeActualState()`) couples to desiredstate's API.

Consolidating into a shared platform module ensures that new domains (compliance,
IoT, deployment) building step pipelines discover and reuse the infrastructure
rather than reimplementing it.

## 3. Design Principles

1. **Platform-first, then consumer migration.** Create yaml-step-core in
   casehub-platform and land it before modifying desiredstate. Same sequencing
   as #128 (yaml-core migration).

2. **VariableSource integration, not wrapper.** `StepContext` implements
   `VariableSource` directly — `PluginInterpolator` is deleted, not wrapped.
   Single interpolation implementation across the platform.

3. **Domain adapter for domain concerns.** `executeActualState()` returns
   `NodeStatus` (domain type). It stays in desiredstate as a thin adapter,
   not in the generic executor.

4. **YAGNI on module splits.** yaml-step-core ships as one module with the
   SPI, executor, registry, expander, expression evaluator, and generic
   primitives. Split into api/runtime when a second consumer needs it.

## 4. New Platform Module — yaml-step-core

**Artifact:** `casehub-platform-yaml-step-core`
**Package:** `io.casehub.yaml.step`
**Depends on:** `casehub-platform-yaml-core` (for `VariableResolver`, `VariableSource`)

### 4.1 Types

| Type | Subpackage | Origin | Notes |
|------|-----------|--------|-------|
| `StepPrimitive` | (root) | plugin/api | Interface, unchanged |
| `StepResult` | (root) | plugin/api | Value type, unchanged |
| `StepParameters` | (root) | plugin/api | Value type, unchanged |
| `StepContext` | (root) | plugin/api | Refactored — implements `VariableSource` |
| `StepDef` | (root) | plugin/model `PluginStepDef` | Renamed, pure record |
| `CompoundStepDef` | (root) | plugin/model `CompoundPrimitiveDef` | Renamed, pure record |
| `StepPipelineExecutor` | (root) | plugin/runtime | `executeActualState()` removed |
| `PrimitiveRegistry` | (root) | plugin/runtime | Unchanged |
| `CompoundStepExpander` | (root) | plugin/runtime `CompoundPrimitiveExpander` | Renamed |
| `StepExecutionException` | (root) | plugin/runtime | Unchanged |
| `ExpressionEvaluator` | expr | plugin/api/expr | Unchanged |
| `ExpressionParseException` | expr | plugin/api/expr | Unchanged |
| `InterpolationException` | (root) | plugin/api | Unchanged |
| `RestCallPrimitive` | primitives | plugin/runtime/primitives | Generic — HTTP call |
| `JsonExtractPrimitive` | primitives | plugin/runtime/primitives | Generic — JSONPath extract |
| `AssertPrimitive` | primitives | plugin/runtime/primitives | Generic — condition assert |

### 4.2 StepContext as VariableSource Provider

yaml-core's `VariableSource` is a `@FunctionalInterface`:
```java
@FunctionalInterface
public interface VariableSource {
    String resolve(String name);  // name = key after prefix dot
}
```

`VariableResolver` maps `${prefix.name}` → finds `VariableSource` for prefix →
calls `source.resolve(name)`. The resolver handles regex matching, recursive
map/list resolution, deferred prefixes, and default values (`${prefix.key:-default}`).

StepContext does NOT implement `VariableSource` (it handles four prefixes, not
one). Instead, it provides factory methods returning per-prefix `VariableSource`
instances backed by its internal state:

```java
package io.casehub.yaml.step;

import io.casehub.yaml.core.resolver.VariableResolver;
import io.casehub.yaml.core.resolver.VariableSource;

public final class StepContext {

    private final Map<String, Object> spec;
    private final Map<String, Map<String, String>> auth;
    private final Map<String, StepResult> results;  // mutable
    private final Map<String, Object> params;

    public VariableSource specSource() {
        return name -> {
            Object value = resolveDeep(spec, name);
            return value != null ? value.toString() : null;
        };
    }

    public VariableSource authSource() {
        return name -> {
            Object value = resolveAuth(name);
            return value != null ? value.toString() : null;
        };
    }

    public VariableSource resultSource() {
        // Closure over mutable results map — subsequent addResult()
        // calls are visible through this source
        return name -> {
            Object value = resolveResult(name);
            return value != null ? value.toString() : null;
        };
    }

    public VariableSource paramSource() {
        return name -> {
            Object value = params.get(name);
            return value != null ? value.toString() : null;
        };
    }

    public VariableResolver toResolver() {
        return new VariableResolver(
            Map.of("spec", specSource(), "auth", authSource(),
                   "result", resultSource(), "param", paramSource()),
            Set.of());
    }

    public void addResult(String name, StepResult result) {
        results.put(name, result);
    }

    // resolveDeep, resolveAuth, resolveResult — same logic as current
    // Builder — same pattern as current
}
```

`toResolver()` builds a base `VariableResolver` with the four step-pipeline
prefixes. Domain layers extend it via `withScope()`:

```java
// In desiredstate's YamlPluginProvisioner/YamlPluginActualStateAdapter
VariableResolver resolver = context.toResolver()
    .withScope("var", variableSource)   // Domain: graph YAML variables
    .withScope("fault", faultSource);   // Domain: fault context
```

**Mutable result accumulation:** The `resultSource()` closure captures the
mutable `results` map. When `StepPipelineExecutor` calls `context.addResult()`
after each step, subsequent `${result.name.*}` references resolve correctly
through the already-constructed `VariableResolver` — no rebuild needed.

**Null handling:** `PluginInterpolator` resolves null to the string `"null"`.
`VariableResolver` throws `UnresolvedVariableException`. Step pipeline sources
return null for missing keys (not throw), which lets `VariableResolver` apply
its default value syntax (`${spec.field:-fallback}`) or throw with context.
This is stricter than the current behavior — missing references are caught
rather than silently becoming `"null"`. Build-time validation already ensures
all `${spec.*}` references exist, so runtime misses indicate genuine errors.

### 4.3 StepPipelineExecutor

The generic executor takes a `PrimitiveRegistry` (stateless, reusable). The
`VariableResolver` is per-invocation (contains per-node state via StepContext
sources), so it's passed to `execute()`:

```java
package io.casehub.yaml.step;

import io.casehub.yaml.core.resolver.VariableResolver;

public class StepPipelineExecutor {

    private final PrimitiveRegistry registry;

    public StepPipelineExecutor(PrimitiveRegistry registry) {
        this.registry = registry;
    }

    public StepResult execute(List<StepDef> steps, StepContext context,
                              VariableResolver resolver) {
        // Same sequential execution with when/retry/skip
        // resolver.resolveString() replaces PluginInterpolator.interpolate()
        // resolver.resolveMap() replaces PluginInterpolator.interpolateMap()
        // context.addResult() accumulates step results (visible via resolver)
    }
}
```

The domain-specific `executeActualState()` method is removed (D3).

**Condition evaluation change:** `evaluateCondition(condition, context)` becomes
`resolver.resolveString(condition, context) → ExpressionEvaluator.evaluate()`.
The two-step flow (interpolate → evaluate) is unchanged; only the interpolation
implementation changes. `VariableResolver.resolveMap()` replaces
`PluginInterpolator.interpolateMap()` for step parameter resolution.

### 4.4 CompoundStepExpander

Renamed from `CompoundPrimitiveExpander`. Uses `StepDef` and `CompoundStepDef`
instead of `PluginStepDef` and `CompoundPrimitiveDef`. Parameter binding
(`${param.*}` substitution) uses `VariableResolver` instead of manual string
replacement.

### 4.5 Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-platform-yaml-core</artifactId>
    </dependency>
    <!-- java.net.http for RestCallPrimitive -->
    <!-- jackson-databind for JsonExtractPrimitive -->
</dependencies>
```

## 5. Changes to Desiredstate

### 5.1 plugin/api (casehub-desiredstate-plugin-api)

**Dependency change:** Replace ownership of step SPI types with dependency on
yaml-step-core. Transitive re-export gives downstream consumers the same API
surface.

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-yaml-step-core</artifactId>
</dependency>
```

**Remaining types:**
- `YamlNodeSpec` — implements `NodeSpec`, domain-coupled

**Deletions:**
- `StepPrimitive.java` — now in yaml-step-core
- `StepResult.java` — now in yaml-step-core
- `StepParameters.java` — now in yaml-step-core
- `StepContext.java` — now in yaml-step-core
- `PluginInterpolator.java` — deleted (D2)
- `InterpolationException.java` — now in yaml-step-core
- `expr/ExpressionEvaluator.java` — now in yaml-step-core
- `expr/ExpressionParseException.java` — now in yaml-step-core

### 5.2 plugin/runtime (casehub-desiredstate-plugin)

**Dependency change:** Gains yaml-step-core transitively via plugin-api.

**Domain adapter — actual state execution:**

```java
package io.casehub.desiredstate.plugin.runtime;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.yaml.step.StepContext;
import io.casehub.yaml.step.StepExecutionException;
import io.casehub.yaml.step.StepPipelineExecutor;
import io.casehub.yaml.step.StepResult;

final class ActualStateStepExecutor {

    private final StepPipelineExecutor executor;

    ActualStateStepExecutor(StepPipelineExecutor executor) {
        this.executor = executor;
    }

    NodeStatus execute(List<StepDef> steps, StepContext context) {
        try {
            StepResult result = executor.execute(steps, context);
            Object status = result.get("nodeStatus");
            if (status instanceof String s) {
                return NodeStatus.valueOf(s);
            }
            return NodeStatus.UNKNOWN;
        } catch (StepExecutionException e) {
            return NodeStatus.UNKNOWN;
        }
    }
}
```

`YamlPluginActualStateAdapter` uses `ActualStateStepExecutor` instead of
calling `StepPipelineExecutor.executeActualState()` directly.

**Remaining types (unchanged in function):**
- `YamlPluginProvisioner` — constructs `VariableResolver` with StepContext + domain sources
- `YamlPluginActualStateAdapter` — uses `ActualStateStepExecutor`
- `CompareStatePrimitive` — domain primitive, maps to `NodeStatus`
- `PluginDescriptor` — domain record
- `CbrPluginMetadata` — CBR metadata
- `YamlPluginRasRegistrar` — RAS situation registration

**Deletions:**
- `StepPipelineExecutor.java` — now in yaml-step-core
- `PrimitiveRegistry.java` — now in yaml-step-core
- `CompoundPrimitiveExpander.java` — now in yaml-step-core as `CompoundStepExpander`
- `StepExecutionException.java` — now in yaml-step-core
- `primitives/RestCallPrimitive.java` — now in yaml-step-core
- `primitives/JsonExtractPrimitive.java` — now in yaml-step-core
- `primitives/AssertPrimitive.java` — now in yaml-step-core

**Renames (import changes):**
- `PluginStepDef` → `io.casehub.yaml.step.StepDef`
- `CompoundPrimitiveDef` → `io.casehub.yaml.step.CompoundStepDef`

### 5.3 plugin/model

**Deletions:**
- `PluginStepDef.java` — now `StepDef` in yaml-step-core
- `CompoundPrimitiveDef.java` — now `CompoundStepDef` in yaml-step-core

**Remaining types (unchanged):**
- `PluginModel`, `PluginParser`, `PluginHeader`, `PluginSpecSchema`,
  `PluginFieldDef`, `PluginProvisionerDef`, `PluginFaultPolicyDef`,
  `PluginCbrDef`, `PluginRasDef`, `PluginAuthStanza`, `PluginParseException`

`PluginParser` produces `StepDef` (from yaml-step-core) instead of
`PluginStepDef`. The YAML deserialization output type changes but the
parsing logic is unchanged — the record fields are identical.

### 5.4 plugin/deployment (casehub-desiredstate-plugin-deployment)

`YamlPluginProcessor` references `StepDef` instead of `PluginStepDef` and
`CompoundStepDef` instead of `CompoundPrimitiveDef`. Build-time validation
logic is unchanged — it validates plugin YAML structure, not step pipeline
mechanics.

## 6. Implementation Sequence

Following the #128 pattern (platform-first, then consumer migration):

### Phase 1 — Platform (casehub-platform)

1. Create `yaml-step-core` module with pom.xml, package structure
2. Add generic types: `StepPrimitive`, `StepResult`, `StepParameters`, `StepDef`,
   `CompoundStepDef`, `StepExecutionException`, `InterpolationException`
3. Add `StepContext` with `VariableSource` factory methods and `toResolver()`
4. Add `ExpressionEvaluator`, `ExpressionParseException`
5. Add `StepPipelineExecutor` (without `executeActualState`)
6. Add `PrimitiveRegistry`, `CompoundStepExpander`
7. Add generic primitives: `RestCallPrimitive`, `JsonExtractPrimitive`, `AssertPrimitive`
8. Tests for all types (ported from desiredstate plugin tests)

**Platform issue(s):** File one issue covering the full module creation.

### Phase 2 — Desiredstate migration

1. Add yaml-step-core dependency to `plugin/api/pom.xml`
2. Delete local copies of migrated types from plugin/api and plugin/runtime
3. Update all imports: `io.casehub.desiredstate.plugin.api.*` →
   `io.casehub.yaml.step.*` for migrated types
4. Rename usages: `PluginStepDef` → `StepDef`, `CompoundPrimitiveDef` →
   `CompoundStepDef`, `CompoundPrimitiveExpander` → `CompoundStepExpander`
5. Create `ActualStateStepExecutor` domain adapter
6. Refactor `YamlPluginProvisioner` and `YamlPluginActualStateAdapter` to
   construct `VariableResolver` with StepContext-backed sources + domain sources
7. Delete `PluginInterpolator`
8. Update `YamlPluginProcessor` references
9. Update all tests

## 7. Dependency Graph (After)

```
casehub-platform-yaml-step-core
    └── casehub-platform-yaml-core

casehub-desiredstate-plugin-api
    ├── casehub-platform-yaml-step-core (transitive re-export)
    └── casehub-desiredstate-api (for NodeSpec → YamlNodeSpec)

casehub-desiredstate-plugin (runtime)
    ├── casehub-desiredstate-plugin-api
    ├── casehub-desiredstate (runtime — NodeStatus, ProvisionResult, etc.)
    ├── casehub-platform (CredentialResolver)
    └── casehub-ras-api (SituationDefinition)

casehub-desiredstate-plugin-deployment
    ├── casehub-desiredstate-plugin (runtime)
    └── yaml deployment infrastructure
```

## 8. Test Strategy

### 8.1 yaml-step-core tests (Phase 1)

Port existing tests from desiredstate plugin modules:
- `ExpressionEvaluatorTest` — all expression parsing and evaluation
- `StepPipelineExecutorTest` — sequential execution, when/retry/skip
- `CompoundStepExpanderTest` — expansion, cycle detection, depth limits
- `PrimitiveRegistryTest` — lookup, unknown primitive error
- `StepContextTest` — prefix resolution, builder
- `RestCallPrimitiveTest`, `JsonExtractPrimitiveTest`, `AssertPrimitiveTest`
- New: `StepContext` as `VariableSource` integration with `VariableResolver`

### 8.2 Desiredstate regression tests (Phase 2)

- All existing plugin tests pass with updated imports
- `YamlPluginProvisionerTest` — VariableResolver wiring
- `YamlPluginActualStateAdapterTest` — ActualStateStepExecutor integration
- `CompareStatePrimitiveTest` — domain primitive unchanged
- `YamlPluginProcessorTest` — build-time validation with renamed types
- End-to-end plugin integration test — full plugin lifecycle

### 8.3 Parity verification

| Dimension | Before | After | Status |
|-----------|--------|-------|--------|
| Step execution (when/retry/skip) | StepPipelineExecutor | StepPipelineExecutor (yaml-step-core) | Parity |
| Interpolation | PluginInterpolator | VariableResolver + StepContext as VariableSource | Parity (single impl) |
| Expression evaluation | ExpressionEvaluator | ExpressionEvaluator (yaml-step-core) | Parity |
| Compound expansion | CompoundPrimitiveExpander | CompoundStepExpander (yaml-step-core) | Parity |
| Actual state mapping | executeActualState() | ActualStateStepExecutor (domain adapter) | Parity |
| Generic primitives | In plugin/runtime | In yaml-step-core | Parity |
| Domain primitives | In plugin/runtime | In plugin/runtime | Unchanged |
| Build-time validation | YamlPluginProcessor | YamlPluginProcessor (updated refs) | Parity |
| Plugin author dependency | plugin-api | plugin-api → yaml-step-core (transitive) | Parity |

Zero regressions. One improvement: single interpolation implementation.

## 9. Deferred Items

| Item | Rationale |
|------|-----------|
| yaml-step-core api/runtime split | YAGNI — one consumer today. Split when a second consumer needs SPI without primitives. |
| Build-time validation in yaml-step-core | Compound step validation (cycles, depth) currently runs at build time in YamlPluginProcessor. Could be extracted to yaml-step-core as a generic validator. Not required for consolidation — desiredstate's processor continues to work. |
| Additional generic primitives (`graphql-call`, `poll-until`, `paginate`) | Listed in #87 spec §14 as deferred. Can be added to yaml-step-core later. |

## 10. Decisions

See `decisions.md` in this spec directory for the full decision log (D1–D5).

## 11. References

- `plugin/api/src/.../StepPrimitive.java` — SPI interface (8 lines)
- `plugin/api/src/.../StepContext.java` — prefix-based context (138 lines)
- `plugin/api/src/.../PluginInterpolator.java` — interpolation (81 lines, deletion target)
- `plugin/api/src/.../expr/ExpressionEvaluator.java` — expression parser (301 lines)
- `plugin/runtime/src/.../StepPipelineExecutor.java` — executor (133 lines)
- `plugin/runtime/src/.../PrimitiveRegistry.java` — registry (30 lines)
- `plugin/runtime/src/.../CompoundPrimitiveExpander.java` — expander (145 lines)
- `plugin/model/PluginStepDef.java` — step record (13 lines)
- `plugin/model/CompoundPrimitiveDef.java` — compound record (11 lines)
- `io.casehub.yaml.core.resolver.VariableResolver` — yaml-core resolver
- `io.casehub.yaml.core.resolver.VariableSource` — yaml-core source interface
- `docs/specs/issue-87-yaml-plugin-architecture/2026-09-09-yaml-plugin-architecture-design.md` — plugin architecture spec
- `docs/specs/issue-128-migrate-yaml-core/2026-09-04-module-migration-design.md` — prior yaml-core migration
- `docs/specs/issue-128-migrate-yaml-core/2026-09-02-yaml-core-migration-context.md` — migration regression analysis
- `docs/blog/2026-09-09-mdp01-the-plugin-that-writes-itself.md` — confirms duplication rationale
- casehubio/casehub-ops#88 — source issue
