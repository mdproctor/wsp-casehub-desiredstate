# YamlGraph Model Change: CSV forEach + ObjectVariableSource Typed Resolution

**Issue:** casehubio/casehub-desiredstate#149
**Scope:** `yaml/runtime/`, `yaml/deployment/`, test fixtures
**Prerequisite:** platform#419 + platform#426 (both landed) — VariableResolver full objectPrefixSources integration

## Overview

Two items sharing a model prerequisite: both touch the `YamlGraph` record constructor. The model change is done once — add `data` field, widen `variables` type, update all call sites — then wire the runtime and deployment sides.

- **Item 1 (S):** CSV-backed forEach via `data` field
- **Item 2 (M):** ObjectVariableSource typed resolution via `variables` type widening

## 1. Model Change — YamlGraph Record

`YamlGraph` gains one new field and one type change:

```java
public record YamlGraph(
    YamlDesiredState desiredState,
    Map<String, Object> variables,       // widened from Map<String, String>
    Map<String, YamlNode> nodes,
    List<YamlFaultPolicy> faultPolicy,
    Map<String, YamlInvariant> invariants,
    Map<String, YamlRule> rules,
    YamlLifecycle lifecycle,
    Map<String, IterationGroup> iterations,
    List<YamlImport> imports,
    Map<String, Object> data) {          // new field (10th position)

    public YamlGraph {
        if (variables == null) {variables = Map.of();}
        if (nodes == null) {nodes = Map.of();}
        if (faultPolicy == null) {faultPolicy = List.of();}
        if (invariants == null) {invariants = Map.of();}
        if (rules == null) {rules = Map.of();}
        if (iterations == null) {iterations = Map.of();}
        if (imports == null) {imports = List.of();}
        if (data == null) {data = Map.of();}
    }
}
```

**`variables` widening:** `Map<String, String>` → `Map<String, Object>`. Jackson deserializes YAML `batch_size: 500` as Integer, `enabled: true` as Boolean, preserving native types through the chain. String values remain strings — backwards compatible.

**`data` field:** `Map<String, Object>` holding named data source declarations. Each entry contains a map with an `inline` key holding CSV text. Parsed at runtime via `CsvDataSource.fromDataBlock()`. Added at the end to minimise constructor call-site churn (12 test sites append `, null`).

**Deserialization sites** (`mapper.readValue(is, YamlGraph.class)`) are unaffected — Jackson is name-based, not position-based. New fields default to null/empty via the compact constructor.

## 2. Runtime Wiring — YamlGraphRecorder

### 2.1 Parameter Type Widening

All four `createYamlGoalCompiler` overloads change `Map<String, String> inlineVariables` → `Map<String, Object> inlineVariables`. The lifecycle compiler (`createYamlLifecycleGoalCompiler`) changes the same way.

### 2.2 Resolver Construction (D2)

With platform#419 + #426 landed, single `withObjectScope` registration is sufficient:

```java
VariableResolver resolver = new VariableResolver(Map.of(), Set.of("match", "fault"))
    .withObjectScope("var", inlineVariables::get);
```

Platform#426 ensures this works for all resolution paths:
- **String interpolation:** `"s3://${var.bucket}/data"` → `lookupVariable` → falls through to objectPrefixSources → `toString()` → `"s3://prod/data"`
- **Type-preserving sole reference:** `${var.batch_size}` → `resolveMap` calls `resolveTyped` → objectPrefixSources → `Integer 500`
- **Module chaining:** `withChainedScope("var", moduleOutput)` derives from objectPrefixSources when no prefixSources entry exists — chaining works correctly

The `withScope("each", ...)` calls in forEach expansion remain unchanged (VariableSource path).

### 2.3 CSV Data Source Wiring (D5)

Before the forEach expansion block, parse the data field:

```java
Map<String, CsvDataSource> dataSources = Map.of();
if (yamlGraph != null && !yamlGraph.data().isEmpty()) {
    dataSources = CsvDataSource.fromDataBlock(yamlGraph.data());
}
```

### 2.4 ForEachExpander Dispatch (D5)

Conditional on data sources being present:

```java
if (hasForEach || hasModules) {
    ExpansionResult<YamlNode> expanded;
    if (!dataSources.isEmpty()) {
        expanded = ForEachExpander.expand(
            effectiveNodes, iterations, dataSources, resolver, adapter, 1000);
    } else {
        expanded = ForEachExpander.expand(
            effectiveNodes, iterations, resolver, adapter, 1000, jsonArrayExpander(mapper));
    }
    // ... rest unchanged
}
```

The CSV-aware overload uses `CsvDataSource` rows for data-driven expansion with `${each.row.fieldName}` access. Non-CSV iteration groups fall back to the standard `IterationGroup` path within the same overload. The non-CSV overload preserves `jsonArrayExpander` for backwards compatibility.

### 2.5 Lifecycle Compiler

`createYamlLifecycleGoalCompiler` gets the parameter type widening and resolver construction change (2.1, 2.2). No CSV data support — forEach in lifecycle phases is a separate concern and the lifecycle compiler doesn't use `ForEachExpander`.

## 3. Deployment Processor — YamlDesiredStateProcessor

### 3.1 Variable Passing

Lines where the processor passes variables to the recorder change from `Map<String, String>` to `Map<String, Object>`:

```java
// Single-graph path (line ~110)
yamlGraph.variables() != null ? yamlGraph.variables() : Map.of()
// Lifecycle path (line ~103)
yamlGraph.variables() != null ? yamlGraph.variables() : Map.of()
```

No coercion needed — `yamlGraph.variables()` already returns `Map<String, Object>` after the model change. The null-coalescing with `Map.of()` still works.

### 3.2 Build-Time Validation for `data`

Add validation in `validateYamlGraph`:

1. **Parse validation:** Call `CsvDataSource.fromDataBlock(graph.data())` — catches malformed CSV at build time.
2. **Cross-reference check:** For nodes with `forEach` referencing a group name, verify the group exists in either `iterations` or the parsed `dataSources`. The existing `validateForEach` method already checks `iterations` — extend it to also accept parsed data source names.

### 3.3 Variable Prefix Normalization (Bonus)

Wire `VariablePrefixRewriter` from yaml-core into the deployment processor for authoring ergonomics. YAML authors can write `${batch_size}` or `${region}` and the processor normalizes to `${var.batch_size}` or `${each.region}` before validation.

Apply rewriting in `discoverYamlGraphs` after parsing, before validation — rewrite string values in node specs, forEach directives, and fault policy templates. The rewriter needs:
- `defaultPrefix`: `"var"` — bare references default to variables
- `knownPrefixes`: `{"var", "each", "match", "fault", "module", "params"}` — already-prefixed references pass through
- `forEachVars`: collected from forEach `as` declarations — bare references matching forEach variable names rewrite to `${each.<name>}`

This is a deployment-time normalization pass, not a runtime concern — the recorder sees already-normalized YAML.

### 3.4 GraphDescriptor

`toGraphDescriptor` is unaffected — it reads `nodes`, `desiredState`, and dependency info. The `data` and widened `variables` fields are consumed at runtime via the recorder, not at build time via the descriptor.

## 4. Test Updates

### 4.1 Constructor Call Sites (12)

All direct `new YamlGraph(...)` calls append `, null` for the `data` parameter:

| File | Count |
|------|-------|
| `YamlLifecycleCompilerTest` | 4 |
| `YamlLifecycleValidationTest` | 7 |
| `YamlConditionalEvaluationTest` | 1 |

### 4.2 New Tests — Item 1 (CSV forEach)

- **Deserialization:** YAML with `data` block deserializes into `YamlGraph.data()`
- **Expansion:** ForEachExpander with CSV data source produces correctly stamped nodes with `${each.row.fieldName}` resolution
- **Deployment validation:** Malformed CSV caught at build time; cross-reference between forEach group and data source validated
- **Integration:** End-to-end from YAML with `data` + `forEach` → compiled `DesiredStateGraph` with expanded nodes

### 4.3 New Tests — Item 2 (Typed Variables)

- **Type preservation:** `variables: {batch_size: 500}` resolves `${var.batch_size}` to `Integer 500` in spec fields (sole reference)
- **String interpolation:** `variables: {bucket: prod}` resolves `"s3://${var.bucket}/data"` to `"s3://prod/data"` (embedded reference)
- **Mixed:** Typed and string variables in the same graph
- **Deployment:** `Map<String, Object>` variables pass through recorder serialization correctly

### 4.4 New Tests — Item 3 (Prefix Normalization)

- **Bare variable rewrite:** `${batch_size}` in spec → normalized to `${var.batch_size}` before compilation
- **ForEach variable rewrite:** `${region}` where `region` is a forEach `as` name → normalized to `${each.region}`
- **Already-prefixed passthrough:** `${var.batch_size}`, `${match.sink.id}`, `${fault.nodeId}` unchanged
- **Mixed:** Bare and prefixed references in the same spec value
- **Integration:** End-to-end from YAML with bare references → compiled graph with correct values

### 4.5 Existing Test Stability

The `VariableResolverTest` in this repo currently constructs resolvers with `Map<String, String>` via the `resolver(Map<String, String>)` helper. These tests remain valid — the old construction pattern still works (VariableSource path). New tests for typed resolution use `withObjectScope`.

## 5. YAML Surface Example

After this change, YAML authors can write:

```yaml
variables:
  batch_size: 500        # preserved as Integer
  bucket: prod           # preserved as String
  enabled: true          # preserved as Boolean

data:
  regions:
    inline: |
      name,tier:integer,endpoint
      us-east,1,https://us-east.example.com
      eu-west,2,https://eu-west.example.com

nodes:
  ingest-${each.region.name}:
    type: data-source
    forEach:
      group: regions
      as: region
    spec:
      name: "${each.region.name}-ingest"
      tier: ${each.region.tier}          # resolves to Integer via CSV typed column
      uri: "s3://${var.bucket}/${each.region.name}/data"
      batchSize: ${batch_size}            # bare ref → normalized to ${var.batch_size} → Integer 500
```

Authors can also write `${var.batch_size}` explicitly — already-prefixed references pass through unchanged.

## References

- `io.casehub.yaml.core.resolver.VariableResolver` — lookupVariable fallthrough (platform#419)
- `io.casehub.yaml.core.resolver.ObjectVariableSource` — typed resolution interface
- `io.casehub.yaml.core.data.CsvDataSource` — `fromDataBlock()` CSV parsing
- `io.casehub.yaml.core.data.CsvColumnType` — typed column parsing (INTEGER, BOOLEAN, NUMBER)
- `io.casehub.yaml.core.foreach.ForEachExpander` — CSV-aware `expand()` overload
- `YamlGraphRecorder.java:68-241` — main goal compiler method
- `io.casehub.yaml.core.resolver.VariablePrefixRewriter` — bare reference normalization
- `YamlDesiredStateProcessor.java:52-162` — discovery and validation
- casehubio/casehub-desiredstate#148 — predecessor (forEach + ConditionEvaluator)
- casehubio/platform#419, #426 — VariableResolver objectPrefixSources integration
