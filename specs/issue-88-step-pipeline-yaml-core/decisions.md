## D1: Module placement — new yaml-step-core

**Choice:** Create a new `casehub-platform-yaml-step-core` module in casehub-platform, sibling to `yaml-core`.
**Alternatives:**
- Add to existing `yaml-core` — smaller footprint but mixes compile-time YAML data transformation with runtime step execution. Consumers needing only VariableResolver would transitively pull HTTP client and expression evaluation dependencies.
**Rationale:** yaml-core is compile-time YAML data transformation (VariableResolver, ForEachExpander, ModuleExpander). Step pipelines are runtime sequential execution with retry, conditional skip, expression evaluation, and a primitive registry. Different abstraction level, different dependency profile. Separation communicates the architectural distinction and keeps dependency graphs clean.
**Trade-offs:** One more module to maintain. Consumers needing both must declare two dependencies.
**Sources:** `plugin/api/` (StepPrimitive, StepContext, StepParameters, StepResult — all pure Java), `plugin/runtime/` (StepPipelineExecutor, PrimitiveRegistry, CompoundPrimitiveExpander), `io.casehub.yaml.core.resolver.VariableResolver` (yaml-core), blog entry 2026-09-09 (confirms duplication rationale)
**Exploration:** quick
**Status:** captured
