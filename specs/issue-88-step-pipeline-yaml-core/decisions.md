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
