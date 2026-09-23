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

## D2: Variable resolution — single ObjectVariableSource registration

**Choice:** Register `"var"` only via `withObjectScope` — single registration. All three platform#426 fixes are landed:
1. `lookupVariable` falls through to objectPrefixSources on both absent prefix AND same-prefix null
2. `withChainedScope` derives from objectPrefixSources when prefixSources has no entry
3. `resolveMap`/`resolveList` call `resolveTyped` for sole references — typed values propagate end-to-end
**Alternatives:**
- D1 dual registration — no longer needed after platform#426
- VariableSource only with toString coercion — loses type preservation entirely
**Rationale:** Platform#426 makes `objectPrefixSources` a proper first-class fallback in all resolution paths. Single registration is clean, correct, and eliminates the dual-registration boilerplate.
**Trade-offs:** None — platform#419 + #426 resolved all gaps
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

## D6: VariablePrefixRewriter normalization at deployment time

**Choice:** Wire `VariablePrefixRewriter` into the deployment processor to normalize bare variable references before validation and compilation. `${batch_size}` → `${var.batch_size}`, `${region}` (forEach var) → `${each.region}`.
**Alternatives:**
- No normalization — authors must always use fully-prefixed references. Works but verbose, especially for simple graphs.
**Rationale:** `VariablePrefixRewriter` already exists in yaml-core. Wiring it at deployment time is a one-pass normalization before validation — zero runtime cost, better authoring ergonomics.
**Trade-offs:** YAML semantics become slightly less explicit (bare refs work). Mitigated by the rewriter being deterministic and the normalized form being visible in error messages.
**Sources:** `io.casehub.yaml.core.resolver.VariablePrefixRewriter` (yaml-core)
**Exploration:** quick
**Status:** captured
