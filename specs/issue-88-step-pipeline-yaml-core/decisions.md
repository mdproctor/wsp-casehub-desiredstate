## D1: Module placement — new yaml-step-core

**Choice:** Create a new `casehub-platform-yaml-step-core` module in casehub-platform, sibling to `yaml-core`.
**Alternatives:**
- Add to existing `yaml-core` — smaller footprint but mixes compile-time YAML data transformation with runtime step execution. Consumers needing only VariableResolver would transitively pull HTTP client and expression evaluation dependencies.
**Rationale:** yaml-core is compile-time YAML data transformation (VariableResolver, ForEachExpander, ModuleExpander). Step pipelines are runtime sequential execution with retry, conditional skip, expression evaluation, and a primitive registry. Different abstraction level, different dependency profile. Separation communicates the architectural distinction and keeps dependency graphs clean.
**Trade-offs:** One more module to maintain. Consumers needing both must declare two dependencies.
**Sources:** `plugin/api/` (StepPrimitive, StepContext, StepParameters, StepResult — all pure Java), `plugin/runtime/` (StepPipelineExecutor, PrimitiveRegistry, CompoundPrimitiveExpander), `io.casehub.yaml.core.resolver.VariableResolver` (yaml-core), blog entry 2026-09-09 (confirms duplication rationale)
**Exploration:** quick
**Status:** captured

## D2: PluginInterpolator → VariableResolver integration — StepContext as VariableSource provider

**Choice:** StepContext provides per-prefix `VariableSource` factory methods (`specSource()`, `authSource()`, `resultSource()`, `paramSource()`) and a `toResolver()` convenience that builds a `VariableResolver`. `PluginInterpolator` is deleted. All interpolation flows through `VariableResolver`. Domain layers extend via `resolver.withScope("var", ...)`.
**Alternatives:**
- StepContext wraps VariableResolver internally — smaller diff in consumers but keeps a pass-through wrapper layer. Doesn't achieve full consolidation; two interpolation APIs coexist.
- StepContext implements VariableSource directly — not possible; VariableSource is a `@FunctionalInterface` handling one prefix, StepContext handles four.
**Rationale:** Same pattern that worked for #128 (desiredstate's domain context → VariableSource for the generic resolver). PluginInterpolator disappears entirely — single implementation for all `${...}` resolution across the platform. `VariableResolver.resolveMap()` replaces `PluginInterpolator.interpolateMap()`. Null handling becomes stricter (throws vs silent "null") — acceptable since build-time validates all references.
**Trade-offs:** StepContext gains a yaml-core dependency (VariableSource, VariableResolver). Acceptable — yaml-step-core already depends on yaml-core (D1). Stricter null handling is a behavioral change but improves error detection.
**Depends on:** D1 (yaml-step-core depends on yaml-core)
**Sources:** `plugin/api/PluginInterpolator.java` (81 lines, deletion target), `plugin/api/StepContext.java` (138 lines, refactor target), `io.casehub.yaml.core.resolver.VariableResolver` (193 lines, prefix routing + regex + resolveMap/List), `io.casehub.yaml.core.resolver.VariableSource` (`@FunctionalInterface`, `String resolve(String name)`), #128 migration pattern (VariableResolver integration in YamlGraphRecorder)
**Exploration:** quick
**Status:** captured

## D3: Step model types — generic StepDef/CompoundStepDef in yaml-step-core

**Choice:** Extract `PluginStepDef` and `CompoundPrimitiveDef` to yaml-step-core as `StepDef` and `CompoundStepDef` (drop "Plugin" prefix). Remove `executeActualState()` from the generic `StepPipelineExecutor` — it returns `NodeStatus` (domain type). Desiredstate keeps a thin domain adapter that wraps the generic executor and maps `StepResult` → `NodeStatus`.
**Alternatives:**
- Keep `executeActualState` on the generic executor with a generic return type (e.g., `<T> T executeAndMap(steps, context, Function<StepResult, T>)`) — over-engineers the API for one consumer. The domain adapter is simpler and keeps the generic executor clean.
- Leave step model types in desiredstate, have yaml-step-core define interfaces — adds indirection without benefit since the records are already pure Java.
**Rationale:** `PluginStepDef` and `CompoundPrimitiveDef` are pure Java records with zero domain imports. They belong with the executor that consumes them. `executeActualState()` is a 12-line method with a single domain import (`NodeStatus`) — it's the adapter, not the engine. Removing it from the generic executor is the same pattern as `DesiredStateGraphAdapter` wrapping `GraphView`.
**Trade-offs:** Desiredstate gains a thin adapter class. `PluginStepDef` → `StepDef` rename ripples through plugin/runtime and plugin/deployment.
**Depends on:** D1 (types move to yaml-step-core)
**Sources:** `plugin/model/PluginStepDef.java` (13 lines, pure record), `plugin/model/CompoundPrimitiveDef.java` (11 lines, pure record), `plugin/runtime/StepPipelineExecutor.java:47-59` (executeActualState — domain-coupled method)
**Exploration:** quick
**Status:** captured

## D4: Generic primitives — ship with yaml-step-core

**Choice:** `RestCallPrimitive`, `JsonExtractPrimitive`, `AssertPrimitive` move into yaml-step-core alongside the executor and registry. No separate primitives module. `CompareStatePrimitive` stays in desiredstate (domain-coupled via `NodeStatus`).
**Alternatives:**
- Separate `yaml-step-primitives` module — keeps yaml-step-core dependency-light but adds a module for three small classes that every consumer will need. YAGNI for one consumer.
- Split yaml-step-core into api/runtime (api has SPI, runtime has primitives + executor) — premature; split when a second consumer surfaces that doesn't want the HTTP dependency.
**Rationale:** A step pipeline without `rest-call` and `assert` is unusable. The generic primitives are the reason the module exists. One consumer today doesn't justify the split.
**Trade-offs:** yaml-step-core gains an HTTP client dependency (likely java.net.http or a thin wrapper). Acceptable — step pipelines are runtime execution, HTTP is expected.
**Depends on:** D1 (module placement)
**Sources:** `plugin/runtime/primitives/RestCallPrimitive.java`, `plugin/runtime/primitives/JsonExtractPrimitive.java`, `plugin/runtime/primitives/AssertPrimitive.java` (generic), `plugin/runtime/primitives/CompareStatePrimitive.java` (domain — stays)
**Exploration:** quick
**Status:** captured

## D5: plugin/api survives as thin re-export module

**Choice:** Keep `casehub-desiredstate-plugin-api` as a module. It depends on yaml-step-core (transitive re-export of `StepPrimitive`, `StepResult`, `StepParameters`, `StepContext`, `ExpressionEvaluator`) and owns `YamlNodeSpec` (domain-coupled via `NodeSpec`). Library JARs implementing custom primitives depend on `plugin-api` to get both the SPI and domain adapter types.
**Alternatives:**
- Delete `plugin/api/` — consumers depend on yaml-step-core directly + desiredstate-api for YamlNodeSpec. Fragments the dependency declaration for no benefit. Library authors need to know two artifacts instead of one.
**Rationale:** `plugin/api/` is the published contract for plugin authors. Changing its artifact ID would break downstream. Keeping it as a thin module that bridges yaml-step-core and desiredstate domain types preserves the contract while eliminating duplicated implementations.
**Trade-offs:** One extra module in the dependency chain (plugin-api → yaml-step-core). Negligible — it's already there.
**Depends on:** D1 (yaml-step-core exists), D2 (StepContext moves), D4 (primitives move)
**Sources:** `plugin/api/pom.xml`, `plugin/api/YamlNodeSpec.java` (sole remaining domain type)
**Exploration:** quick
**Status:** captured
