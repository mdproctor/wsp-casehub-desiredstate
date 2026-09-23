# Decisions — #149 YamlGraph Model Change

## D1: Variable resolution — dual VariableSource registration

**Choice:** Register both `VariableSource` (string interpolation) and `ObjectVariableSource` (type-preserving) for the `"var"` prefix
**Alternatives:**
- ObjectVariableSource only — simpler wiring but breaks string interpolation (`resolveString()` only checks `prefixSources`)
- VariableSource only with toString coercion — loses type preservation, defeating the purpose of item 2
**Rationale:** VariableResolver was designed for this dual path. `resolveTyped()` checks `objectPrefixSources` first for sole references like `${var.batch_size}`, `lookupVariable()` checks `prefixSources` for embedded interpolation. Both must be present.
**Trade-offs:** Two registrations instead of one — trivial complexity cost
**Sources:** `io.casehub.yaml.core.resolver.VariableResolver` (yaml-core), `VariableResolverTest` line 82-87 (non-string passthrough)
**Exploration:** quick
**Status:** revised — superseded by D2

## D2: Variable resolution — single ObjectVariableSource registration via yaml-core fix

**Choice:** Fix `VariableResolver.lookupVariable()` in yaml-core (platform#419) to fall through to `objectPrefixSources` for string interpolation (toString). Then register only `ObjectVariableSource` for `"var"` — no dual registration needed.
**Alternatives:**
- D1 dual registration (without yaml-core fix) — works but forces boilerplate on every consumer, error-prone
- VariableSource only with toString coercion — loses type preservation entirely
**Rationale:** `ObjectVariableSource` subsumes `VariableSource` — every Object can be toString'd for interpolation. The gap in `lookupVariable` is an implementation oversight, not a conceptual need. Fixing it in yaml-core (~5 lines, non-breaking) benefits all consumers.
**Trade-offs:** Cross-repo prerequisite (platform#419 must land before desiredstate#149)
**Depends on:** platform#419
**Sources:** `io.casehub.yaml.core.resolver.VariableResolver` lines 220-268 (lookupVariable only checks prefixSources), `io.casehub.yaml.core.resolver.ObjectVariableSource`
**Exploration:** deep-analysis
**Status:** captured

## D3: `data` field placement in YamlGraph record

**Choice:** Add `Map<String, Object> data` as the last field (10th position)
**Alternatives:**
- After `iterations` (semantic grouping) — forces reordering of trailing fields at all 12 call sites for no functional benefit
**Rationale:** Jackson deserialization is name-based — field order doesn't affect YAML parsing. Appending minimises call-site churn (all 12 sites just append `, null`).
**Trade-offs:** None meaningful
**Sources:** `YamlGraph.java` (current 9-field record), 12 constructor call sites across tests
**Exploration:** quick
**Status:** captured

## D4: Recorder parameter type widening

**Choice:** Widen `inlineVariables` from `Map<String, String>` to `Map<String, Object>` throughout the `YamlGraphRecorder` method chain
**Alternatives:**
- Keep `Map<String, String>` in recorder, coerce at call site — loses type info before it reaches the resolver, defeating item 2
**Rationale:** Quarkus recorder serialization handles primitive-typed Object values (String, Integer, Boolean, Double) which is exactly what YAML produces. The deployment processor passes `yamlGraph.variables()` directly — type widening is a clean pass-through.
**Trade-offs:** All 4 `createYamlGoalCompiler` overloads need the signature change
**Sources:** `YamlGraphRecorder` lines 37-76 (4 overloads), `YamlDesiredStateProcessor` lines 103, 110 (call sites)
**Exploration:** quick
**Status:** captured

## D5: ForEachExpander overload selection

**Choice:** Conditional dispatch — when `yamlGraph.data()` is non-null and non-empty, parse via `CsvDataSource.fromDataBlock()` and call the CSV-aware `ForEachExpander.expand(elements, iterationGroups, dataSources, resolver, adapter, maxExpansion)`. Otherwise use the existing overload with `jsonArrayExpander`.
**Alternatives:**
- Always use CSV overload (pass empty dataSources) — CSV overload uses `resolveValuesStatic` which doesn't support `IterationValueExpander`, losing JSON array expansion for non-CSV graphs
**Rationale:** The two overloads serve different use cases. CSV overload handles structured tabular data with row-level field access. Existing overload handles simple value lists with optional JSON array expansion. Mixing is unlikely and YAGNI.
**Trade-offs:** Two code paths in `YamlGraphRecorder` — acceptable given the distinct semantics
**Sources:** `ForEachExpander` (yaml-core) — two `expand()` overloads, `CsvDataSource.fromDataBlock()`
**Exploration:** quick
**Status:** captured
