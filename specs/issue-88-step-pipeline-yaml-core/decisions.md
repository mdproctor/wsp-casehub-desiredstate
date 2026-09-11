## D1: Module placement — new yaml-step-core

**Choice:** Create a new `casehub-platform-yaml-step-core` module in casehub-platform, sibling to `yaml-core`.
**Alternatives:**
- Add to existing `yaml-core` — smaller footprint but mixes compile-time YAML data transformation with runtime step execution. Consumers needing only VariableResolver would transitively pull HTTP client and expression evaluation dependencies.
**Rationale:** yaml-core is compile-time YAML data transformation (VariableResolver, ForEachExpander, ModuleExpander). Step pipelines are runtime sequential execution with retry, conditional skip, expression evaluation, and a primitive registry. Different abstraction level, different dependency profile. Separation communicates the architectural distinction and keeps dependency graphs clean.
**Trade-offs:** One more module to maintain. Consumers needing both must declare two dependencies.
**Sources:** `plugin/api/` (StepPrimitive, StepContext, StepParameters, StepResult — all pure Java), `plugin/runtime/` (StepPipelineExecutor, PrimitiveRegistry, CompoundPrimitiveExpander), `io.casehub.yaml.core.resolver.VariableResolver` (yaml-core), blog entry 2026-09-09 (confirms duplication rationale)
**Exploration:** quick
**Status:** captured

## D2: PluginInterpolator → VariableResolver integration — StepContext as VariableSource

**Choice:** StepContext implements yaml-core's `VariableSource` interface. `PluginInterpolator` is deleted. All interpolation flows through `VariableResolver` constructed with a StepContext-backed source. StepContext becomes a thin data holder + source adapter.
**Alternatives:**
- StepContext wraps VariableResolver internally — smaller diff in consumers but keeps a pass-through wrapper layer. Doesn't achieve full consolidation; two interpolation APIs coexist.
**Rationale:** Same pattern that worked for #128 (desiredstate's domain context → VariableSource for the generic resolver). PluginInterpolator disappears entirely — single implementation for all `${...}` resolution across the platform. StepContext.resolve() can remain as a convenience but delegates to the VariableSource contract internally.
**Trade-offs:** StepContext gains a yaml-core dependency (VariableSource interface). Acceptable — yaml-step-core already depends on yaml-core (D1).
**Depends on:** D1 (yaml-step-core depends on yaml-core)
**Sources:** `plugin/api/PluginInterpolator.java` (81 lines, deletion target), `plugin/api/StepContext.java` (138 lines, refactor target), `io.casehub.yaml.core.resolver.VariableResolver`, `io.casehub.yaml.core.resolver.VariableSource`, #128 migration pattern (VariableResolver integration in YamlGraphRecorder)
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
