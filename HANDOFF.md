# HANDOFF — casehub-desiredstate

## Last Session

Completed #128 (yaml-core migration). Three commits:

1. **VariableResolver migration** — replaced local `VariableResolver` and `UnresolvedVariableException` with yaml-core versions. `VariableSource.chain()` for source composition, `Set.of("match", "fault")` deferred prefixes replace `resolveTemplateString`. `withChainedScope("var", scope::get)` replaces `withModuleScope`. `Truthiness.isTruthy()` replaces local `isTruthy`. All callers updated: `YamlGraphRecorder`, `YamlRuleConverter`, `HookResolver`, `ForEachExpander`. Local resolver package deleted.

2. **ForEachExpander migration** — created `YamlNodeForEachAdapter` implementing `ForEachAdapter<YamlNode>` with `ForEachDirective` conversion (sealed interface: `GroupRef`/`InlineIteration`), module-scope-aware stamping, dependency variable resolution in `stamp()`, and generic reference rewriting via `getReferences()`/`withReferences()` (platform #259). Restructured `YamlGraphRecorder` from single-pass to two-pass (yaml-core expand, then domain transform). `IterationValueExpander` replaces local `parseJsonArray`. `YamlGraph.iterations()` type changed to yaml-core `IterationGroup`. Deployment processor updated. Local `ForEachExpander` and `YamlIterationGroup` deleted.

3. Net: -308 lines (541 deleted, 233 added). Full 28-module build green, 108 yaml/runtime + 28 yaml/deployment tests pass.

Advanced queue to #126.

## Immediate Next Step

Brainstorm #126 (adopt yaml-core module system + parameter validation). The module layer migration is different from #128 — `ModuleExpander` needs to use yaml-core's `ModuleExpander` with `SectionDeserializer`, `SectionContentRewriter`, typed `YamlModuleOutput`, and `ParameterValidator`. Read the yaml-core module API surface before designing.

Key platform APIs to study:
- `io.casehub.yaml.core.module.ModuleExpander` — generic module expansion
- `io.casehub.yaml.core.module.SectionDeserializer` / `SectionContentRewriter` — typed section handling
- `io.casehub.yaml.core.module.ParameterValidator` / `ParameterValidationException` — validation
- `io.casehub.yaml.core.module.YamlModuleOutput` — typed outputs for cross-module composition
- `io.casehub.yaml.core.module.ExpansionOptions` — expansion configuration

## Cross-Module

- Platform #267 (graph-core extraction) — future, not blocking. Desiredstate #129 (in-place refactor) is the prerequisite.
- Platform #257 (allowedValues + constraintDescription) — open, would add enum constraints to parameter validation
- Platform #266 (API polish — typedSection, output param validation, commaSplit, dead getId) — open, final quality pass

## References

- `specs/issue-128-migrate-yaml-core/2026-08-31-migrate-yaml-core-design.md` — migration spec (executed, historical)
- `specs/issue-128-migrate-yaml-core/2026-09-02-yaml-core-migration-context.md` — full context: regression analysis, graph-core architecture
- `plans/2026-08-31-migrate-yaml-core.md` — implementation plan (executed, historical)
- `docs/adr/0002-custom-yaml-surface-over-existing-tools.md` — ADR: why custom over CUE/ytt/HCL
- `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlNodeForEachAdapter.java` — new adapter (92 lines)
- `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/ModuleExpander.java` — local module expander (#126 migration target)
