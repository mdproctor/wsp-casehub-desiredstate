# TypeScript DSL — Design Spec

**Issue:** #122 — TypeScript DSL: type-safe graph declarations
**Date:** 2026-08-29
**Status:** Draft

## 1. Summary

A TypeScript declaration surface for CaseHub desired-state graphs. DevOps
engineers who prefer programmatic control declare graphs in TypeScript with
full IDE autocomplete, type-safe spec construction, and native TS constructs
(loops, conditionals, functions) for dynamic graph generation. The TS DSL
compiles to the same `GraphDescriptor` IR as YAML and Java annotations.

**Primary value:** Programmatic graph construction with type safety — the
capability YAML can't express. Computed node topologies, environment-aware
configurations, and conditional composition using a real programming language.

**Secondary value:** TypeScript type definitions (`.d.ts`) serve as precise
LLM context for code generation. The consumer decides which surface fits
their use case — neither YAML nor TS is positioned as preferred.

**What TS does NOT do:** Rules, invariants, and fault policies remain in
YAML or Java annotations. They reach TS-declared graphs via cross-surface
resolution (#124). TS handles graph construction; declarative concerns stay
in declarative surfaces.

## 2. Background

The YAML surface (#117, #116) provides a full declarative vocabulary for
graph declarations — nodes, dependencies, rules, invariants, fault policies,
lifecycle phases, forEach, when, modules. Java annotations provide the same
with full type safety and programmatic power.

The YAML surface is limited to what can be expressed declaratively. The
`forEach:` directive handles simple stamping but cannot express conditional
composition, computed specs, or dynamic topology based on runtime data.
Java annotations provide full programmatic power but require Java expertise
and operate within the Quarkus build lifecycle.

TypeScript bridges this gap: full programming language capabilities for
graph construction, type-safe specs via generated `.d.ts` files, and
familiarity for DevOps engineers who already use Pulumi or CDK.

### 2.1 Convergence Architecture

All four surfaces compile to the same intermediate representation:

```
Visual editor ──→ YAML ──→ GraphDescriptor ──→ GoalCompiler ──→ ReconciliationLoop
YAML file            ──→ GraphDescriptor ──→ GoalCompiler ──→ ReconciliationLoop
TS DSL               ──→ GraphDescriptor ──→ GoalCompiler ──→ ReconciliationLoop
Java @annotations    ──→ GraphDescriptor ──→ GoalCompiler ──→ ReconciliationLoop
```

The runtime is surface-agnostic. `GraphDescriptor` carries nodes, dependencies,
rules, and invariants. `GoalCompiler` produces `CompilationResult` (single graph
or lifecycle phases). The reconciliation loop, fault policy engine, and
lifecycle manager operate identically regardless of source surface.

## 3. Architecture

### 3.1 Pipeline

```
TS source (.ts)
     │
     ▼
TsExecutor SPI              TS SDK (.d.ts) ──→ IDE autocomplete
├─ TsjTsExecutor (JVM)
└─ NodeTsExecutor (subprocess)
     │
     ▼
JSON envelope
{"kind":"single"|"lifecycle", ...}
     │
     ▼
TsDesiredStateProcessor (Quarkus build extension)
     │
     ▼
DesiredStateGraphBuildItem ──► CrossSurfaceRuleResolutionStep
     │                              │
     ▼                              ▼
GoalCompiler ◄── rules/invariants from YAML/annotation surfaces
     │
     ▼
ReconciliationLoop
```

### 3.2 Module Placement

Following the `yaml-core` pattern in `casehub-platform`:

| Module | Repo | Package | Purpose |
|--------|------|---------|---------|
| `ts-core/` | casehub-platform | `io.casehub.ts.core` | `TsExecutor` SPI, TSJ + Node.js implementations, type generation tooling |
| `ts-dsl/runtime/` | casehub-desiredstate | `io.casehub.desiredstate.ts` | `TsGraphRecorder` — JSON envelope → GoalCompiler bridge |
| `ts-dsl/deployment/` | casehub-desiredstate | `io.casehub.desiredstate.ts.deployment` | `TsDesiredStateProcessor` — Quarkus build extension |
| `ts-dsl/sdk/` | casehub-desiredstate | npm: `@casehub/desiredstate` | TS SDK — `defineGraph()`, base types, generated NodeTypeMap |

`ts-core` is shared infrastructure — any repo wanting TS-defined configurations
gets the executor SPI and builds its own domain-specific processor. The
desiredstate `ts-dsl/` layers domain logic on top, paralleling how
`desiredstate/yaml/` layers on `platform/yaml-core/`.

### 3.3 Scope Boundary

| Concern | TS surface | YAML surface | Java annotations |
|---------|-----------|--------------|------------------|
| Node declarations | ✓ (imperative) | ✓ (declarative) | ✓ (declarative) |
| Dependency edges | ✓ | ✓ | ✓ |
| Lifecycle phases | ✓ (native TS) | ✓ (`lifecycle:`) | ✓ (`GoalMethod`) |
| forEach / stamping | ✓ (native `for`/`map`) | ✓ (`forEach:`) | ✓ (Java loops) |
| Conditional inclusion | ✓ (native `if`) | ✓ (`when:`) | ✓ (Java conditionals) |
| Modules / composition | ✓ (native `import`) | ✓ (`imports:`) | ✓ (Java composition) |
| Graph rules | ✗ (via cross-surface) | ✓ (`rules:`) | ✓ (`@GraphRule`) |
| Graph invariants | ✗ (via cross-surface) | ✓ (`invariants:`) | ✓ (`@GraphInvariant`) |
| Fault policies | ✗ (via Java annotation or CDI bean) | ✓ (`faultPolicy:`) | ✓ (`@FaultPolicyDef`) |
| Human gating | ✓ | ✓ | ✓ |
| Lifecycle hooks | ✓ | ✓ | ✗ |

TS uses native language constructs where YAML uses declarative directives.
The output is the same: an expanded graph with concrete nodes and dependencies.

## 4. TsExecutor SPI (ts-core)

### 4.1 Interface

```java
package io.casehub.ts.core;

public interface TsExecutor {
    TsEvalResult evaluate(String tsSource);
    TsEvalResult evaluate(Path tsFile);
}

public record TsEvalResult(
    String json,
    List<TsError> errors
) {
    public boolean success() { return errors.isEmpty(); }
}

public record TsError(
    String message,
    String file,
    int line,
    int column
) {}
```

### 4.2 TsjTsExecutor

Default implementation using TSJ (ts2jvm). Compiles TypeScript to JVM
bytecode and executes it, capturing the default export as JSON.

The TS source file must have a default export that is a plain object (the
graph envelope). TSJ evaluates the TS module and serializes the default
export to JSON via `JSON.stringify()`.

**Evaluation model:** Each `.ts` file is evaluated independently. ES module
`import` statements resolve against the classpath (for SDK types) or
relative paths (for user modules). TSJ handles TypeScript compilation
internally — no external `tsc` step needed.

**Error handling:** TypeScript compilation errors are captured as `TsError`
entries with file/line/column context. Runtime errors (exceptions during
evaluation) are wrapped with stack trace context.

### 4.3 NodeTsExecutor

Fallback implementation. Spawns a Node.js subprocess with a runner script:

```
node --loader tsx <runner.js> <input.ts>
```

The runner script imports the TS file's default export and writes
`JSON.stringify(result)` to stdout. Parse errors and runtime exceptions
are captured via stderr.

**Prerequisite:** Node.js 20+ and `tsx` (or `ts-node`) on PATH. Validated
at Quarkus build time — clear error if missing.

### 4.4 Build-Time Instantiation

The `TsDesiredStateProcessor` instantiates the executor directly at
build time — CDI beans are not available during Quarkus augmentation.
This follows the same pattern as `YamlDesiredStateProcessor`, which
creates its `ObjectMapper` directly rather than via CDI injection.

```java
// In TsDesiredStateProcessor @BuildStep method:
TsExecutor executor;
try {
    executor = new TsjTsExecutor();
} catch (NoClassDefFoundError e) {
    executor = new NodeTsExecutor();
}
```

`TsjTsExecutor` is classpath-activated — present only when the TSJ
dependency is included. `NodeTsExecutor` is always available as fallback.
The classpath check determines which implementation is used.

## 5. TS SDK (`@casehub/desiredstate`)

### 5.1 Package Structure

```
@casehub/desiredstate/
├── src/
│   ├── index.ts          # defineGraph(), defineLifecycle()
│   ├── types.ts          # GraphDef, NodeDef, PhaseConfig, etc.
│   └── generated/        # Auto-generated NodeTypeMap + spec interfaces
├── package.json
└── tsconfig.json
```

### 5.2 Core API — `defineGraph()` and `node()`

```typescript
export function defineGraph(def: GraphDef): GraphEnvelope {
    return { kind: 'single', ...def };
}

export function defineLifecycle(def: LifecycleDef): LifecycleEnvelope {
    return { kind: 'lifecycle', ...def };
}

export function node<T extends keyof NodeTypeMap>(
    type: T,
    spec: NodeTypeMap[T],
    opts?: { dependsOn?: DependencyRef[]; humanGating?: HumanGating; hooks?: NodeHooks }
): NodeDef {
    return { type, spec, ...opts } as NodeDef;
}
```

### 5.3 Type Definitions

```typescript
export interface GraphDef {
    namespace: string;
    name: string;
    nodes: Record<string, NodeDef>;
    dependencies?: DependencyDef[];
}

export type NodeDef = {
    [T in keyof NodeTypeMap]: {
        type: T;
        spec: NodeTypeMap[T];
        dependsOn?: DependencyRef[];
        humanGating?: HumanGating;
        hooks?: NodeHooks;
    }
}[keyof NodeTypeMap];

export type DependencyRef = string;

export type HumanGating = 'NONE' | 'PROVISION_ONLY' | 'DEPROVISION_ONLY' | 'ALL';

export interface NodeHooks {
    provision?: HookBlock;
    deprovision?: HookBlock;
}

export interface HookBlock {
    pre?: HookStep[];
    post?: HookStep[];
}

export type HookStep =
    | { verify: VerifyStep }
    | { notify: NotifyStep }
    | { wait: WaitStep };

export interface VerifyStep {
    url: string;
    timeout?: number;
}

export interface NotifyStep {
    channel: string;
    message: string;
}

export interface WaitStep {
    seconds: number;
}
```

### 5.4 Lifecycle Types

```typescript
export interface LifecycleDef {
    namespace: string;
    name: string;
    phases: PhaseDef[];
}

export interface PhaseDef {
    id: string;
    completionCondition: CompletionCondition;
    nodes: Record<string, NodeDef>;
    dependencies?: DependencyDef[];
}

export type CompletionCondition = 'allPresent' | 'never' | { bean: string };
```

### 5.5 NodeTypeMap — Discriminated Union

The generated `NodeTypeMap` provides the type narrowing that makes spec
autocomplete work:

```typescript
// generated/node-type-map.d.ts (auto-generated from @NodeTypeId scan)
export interface NodeTypeMap {
    'data-source': DataSourceSpec;
    'transformer': TransformerSpec;
    'sink': SinkSpec;
    'monitor': MonitorSpec;
    'alerter': AlerterSpec;
    'ai-review': AiReviewSpec;
    'human-review': HumanReviewSpec;
}

export interface DataSourceSpec {
    uri: string;
    format: 'CSV' | 'JSON' | 'PARQUET';
    batchSize?: number;
}

export interface TransformerSpec {
    operation: 'VALIDATE' | 'TRANSFORM' | 'ENRICH';
    rules?: string[];
}

// ... one interface per @NodeTypeId-annotated NodeSpec
```

When the operator types `type: 'data-source'`, TypeScript narrows `spec`
to `DataSourceSpec` — autocomplete shows `uri`, `format`, `batchSize`.
Wrong fields produce compile errors.

### 5.6 Dependencies and Envelope Transformation

Dependencies can be declared inline on nodes via `dependsOn` (producing
`DependencyDef` entries in the envelope), or explicitly via the top-level
`dependencies` array for cases where the dependency doesn't belong to
either node's declaration:

```typescript
export interface DependencyDef {
    from: string;  // source node ID
    to: string;    // target node ID
}
```

**SDK → Envelope transformation:** `defineGraph()` transforms the authoring
format into the wire format:

1. **Nodes:** `Record<string, NodeDef>` (map keyed by ID) → array of objects
   with explicit `id` field. The map key becomes the `id`.
2. **Dependencies:** Collects `dependsOn` from each node and merges with
   top-level `dependencies`. `dependsOn: ['csv-source']` on node
   `transformer` becomes `{ from: 'transformer', to: 'csv-source' }`.
3. **Stripping:** `dependsOn` is removed from each node in the output
   (it's been promoted to the `dependencies` array).

This keeps the authoring format ergonomic (map keys as IDs, inline deps)
while the wire format is flat and easy to parse on the Java side.

## 6. JSON Envelope Schema

The TS SDK's `defineGraph()` and `defineLifecycle()` produce a JSON
envelope consumed by `TsDesiredStateProcessor`.

### 6.1 Single Graph

```json
{
    "kind": "single",
    "namespace": "pipeline",
    "name": "medallion",
    "nodes": [
        {
            "id": "csv-source",
            "type": "data-source",
            "spec": { "uri": "s3://data/customers.csv", "format": "CSV" },
            "humanGating": "NONE"
        },
        {
            "id": "transformer",
            "type": "transformer",
            "spec": { "operation": "VALIDATE" },
            "humanGating": "NONE"
        }
    ],
    "dependencies": [
        { "from": "transformer", "to": "csv-source" }
    ]
}
```

### 6.2 Lifecycle

```json
{
    "kind": "lifecycle",
    "namespace": "pipeline",
    "name": "medallion",
    "phases": [
        {
            "id": "infrastructure",
            "completionCondition": "allPresent",
            "nodes": [...],
            "dependencies": [...]
        },
        {
            "id": "application",
            "completionCondition": "allPresent",
            "nodes": [...],
            "dependencies": [...]
        },
        {
            "id": "observability",
            "completionCondition": "never",
            "nodes": [...],
            "dependencies": [...]
        }
    ]
}
```

### 6.3 Envelope → GraphDescriptor Mapping

| JSON envelope field | GraphDescriptor field | Notes |
|--------------------|-----------------------|-------|
| `namespace` | `namespace` | Direct |
| `name` | `name` | Direct |
| `nodes[].id` | `NodeDescriptor.InlineNode.id` | All TS nodes are inline |
| `nodes[].type` | `NodeDescriptor.InlineNode.specClassName` | Looked up via `typeRegistry.get(type)` — the Jandex-built `Map<String, String>` (type string → spec class name), same as `YamlDesiredStateProcessor.toGraphDescriptor()` |
| `nodes[].spec` | `NodeDescriptor.InlineNode.specValues` | Map<String, Object> |
| `nodes[].humanGating` | `NodeDescriptor.InlineNode.humanGating` | Enum mapping |
| `dependencies[]` | `DependencyDescriptor` | Direct |
| `nodes[].hooks` | Resolved via `HookResolver` | Same as YAML hooks (#121) |

`interfaceName` and `implClassName` are null for TS-sourced descriptors
(annotation-specific fields). `goalMethod` is null (TS has no Java method).
`graphRules` and `graphInvariants` are empty lists — these arrive via
`CrossSurfaceRuleResolutionStep`.

## 7. TsDesiredStateProcessor (Build Extension)

Parallel to `YamlDesiredStateProcessor`. Runs at Quarkus build time.

### 7.1 Discovery

Classpath scan for `META-INF/desiredstate/*.ts` files. Same convention as
YAML (`META-INF/desiredstate/*.yaml`).

The processor also discovers pre-compiled `META-INF/desiredstate/*.ds.json`
files — JSON envelopes produced by an external TS build step. This supports
CI/CD pipelines where TS compilation happens before the Maven build.

### 7.2 Evaluation

For `.ts` files: inject `TsExecutor` (from `ts-core`), call
`evaluate(tsSource)`, parse the resulting JSON envelope.

For `.ds.json` files: parse directly — no TS evaluation needed.

### 7.3 Validation

After parsing the JSON envelope:

1. **Node types** — validate each node's `type` against the `NodeSpecRegistry`
   (same Jandex scan as YAML)
2. **Dependency references** — validate `from`/`to` refer to declared nodes
3. **Cycle detection** — same algorithm as `YamlDesiredStateProcessor`
4. **Spec deserialization** — `ObjectMapper.convertValue()` each node's spec
   map into the typed `NodeSpec` class. Deserialization errors produce
   build-time failures with TS source context
5. **Cross-surface namespace collision** — via `DesiredStateGraphBuildItem`
   (existing infrastructure)
6. **Lifecycle validation** — phase IDs unique, completion conditions valid,
   cross-phase dependency references resolve (same rules as YAML §6.5)
7. **Hook validation** — same rules as YAML (#121)

### 7.4 Build Items and Cross-Surface Rule Consumption

The processor's `@BuildStep` method signature mirrors `YamlDesiredStateProcessor.discoverYamlGraphs()`:

```java
@BuildStep
@Record(ExecutionTime.RUNTIME_INIT)
void discoverTsGraphs(CombinedIndexBuildItem indexBuildItem,
                      TsGraphRecorder recorder,
                      BuildProducer<SyntheticBeanBuildItem> syntheticBeans,
                      BuildProducer<DesiredStateGraphBuildItem> graphItems,
                      List<AdditionalRulesBuildItem> additionalRuleItems) {
    // ...
}
```

The processor produces:
- `DesiredStateGraphBuildItem` with `source: "ts:<fileName>"` — feeds
  `CrossSurfaceRuleResolutionStep` and cross-surface duplicate detection
- `SyntheticBeanBuildItem` for `GoalCompiler<Void>` — same pattern as YAML

**Cross-surface rule consumption:** For each TS graph, the processor
finds matching `AdditionalRulesBuildItem` by namespace+name and passes
the matched rules/invariants to `TsGraphRecorder`:

```java
List<GraphRuleDescriptor> crossSurfaceRules = List.of();
List<GraphInvariantDescriptor> crossSurfaceInvariants = List.of();
for (var additional : additionalRuleItems) {
    if (additional.namespace().equals(ns) && additional.name().equals(name)) {
        crossSurfaceRules = additional.rules();
        crossSurfaceInvariants = additional.invariants();
        break;
    }
}

if (isLifecycle) {
    compiler = recorder.createTsLifecycleGoalCompiler(
            envelope, typeRegistry, invariants);
} else {
    compiler = recorder.createTsGoalCompiler(
            descriptor, typeRegistry, invariants,
            crossSurfaceRules, crossSurfaceInvariants);
}
```

Note: cross-surface rules/invariants flow into single-graph compilers
only. Lifecycle compilers do not currently receive them — this matches
the existing YAML behavior where `createYamlLifecycleGoalCompiler()`
also does not accept cross-surface parameters. See §7.6 for details.

### 7.5 Cross-Surface Rule Resolution

`CrossSurfaceRuleResolutionStep` currently filters:
```java
if (!graph.source().startsWith("yaml:")) { continue; }
```

This must change to:
```java
if (graph.source().startsWith("annotation:")) { continue; }
```

This broadens the filter to include both `yaml:*` and `ts:*` sources
(and any future surfaces). Annotation graphs are excluded because their
rules are already resolved inline by `DesiredStateAnnotationsProcessor`.

### 7.6 TsGraphRecorder

`TsGraphRecorder` is a Quarkus `@Recorder` (same pattern as
`YamlGraphRecorder`) that creates `GoalCompiler` closures.

#### Single-graph GoalCompiler — `createTsGoalCompiler()`

```java
@Recorder
public class TsGraphRecorder {

    @SuppressWarnings({"unchecked", "rawtypes"})
    public RuntimeValue<GoalCompiler> createTsGoalCompiler(
            GraphDescriptor descriptor,
            Map<String, String> typeRegistryMap,
            List<ResolvedInvariant> invariants,
            List<GraphRuleDescriptor> crossSurfaceRuleDescriptors,
            List<GraphInvariantDescriptor> crossSurfaceInvariantDescriptors) {

        ObjectMapper mapper = new ObjectMapper();

        return new RuntimeValue<>((GoalCompiler) (goals, factory) -> {
            // 1. Materialize nodes: resolve specClassName via typeRegistry,
            //    deserialize specValues via ObjectMapper.convertValue()
            // 2. Materialize dependencies: DependencyDescriptor → Dependency
            // 3. Build graph via factory.of(nodes, deps)
            // 4. Evaluate cross-surface rules (if any)
            // 5. Validate invariants (inline + cross-surface)
            // 6. Return CompilationResult.single(graph)
        });
    }
}
```

The GoalCompiler closure:
1. Converts `NodeDescriptor.InlineNode` → `DesiredNode` using
   `typeRegistryMap` for spec class resolution and `ObjectMapper` for
   spec deserialization — identical to `YamlGraphRecorder`
2. Converts `DependencyDescriptor` → `Dependency`
3. Evaluates cross-surface rules via `GraphRuleEngine` (if any
   `crossSurfaceRuleDescriptors` were matched)
4. Validates invariants via `GraphInvariantEngine` (inline invariants +
   cross-surface invariants merged)
5. Returns `CompilationResult.single(graph)`

TS does not need `VariableResolver` (no YAML `${var}` syntax), `when:`
conditional exclusion, `forEach:` expansion, or module expansion. These
are YAML-specific concerns handled by native TS constructs at graph
construction time.

#### Lifecycle GoalCompiler — `createTsLifecycleGoalCompiler()`

```java
public RuntimeValue<GoalCompiler> createTsLifecycleGoalCompiler(
        TsLifecycleEnvelope envelope,
        Map<String, String> typeRegistryMap,
        List<ResolvedInvariant> invariants) { ... }
```

Lifecycle carry-forward semantics (matching `YamlGraphRecorder.createYamlLifecycleGoalCompiler()`):

1. **Phase iteration:** Process phases in declared order
2. **Node materialization:** For each phase, materialize declared nodes
   from the envelope (type registry lookup + spec deserialization)
3. **Carry-forward merge:** Merge carry-forward nodes from previous
   phases into the current phase's graph. Phase-declared nodes override
   carry-forward nodes with the same ID
4. **Dependency carry-forward:** Carry forward dependencies from previous
   phases only when both endpoints (from, to) exist in the merged graph
   AND the `from` node is a carry-forward node (not re-declared in this
   phase)
5. **Per-phase invariant validation:** Validate invariants against the
   merged phase graph — `GraphInvariantEngine.validate(phaseGraph, invariants)`
6. **Completion condition resolution:** Map `allPresent`, `never`, or
   `{ bean: "name" }` to `CompletionCondition`
7. **State update:** After processing each phase, update carry-forward
   state: `carryForwardNodes = phaseGraph.nodes()`,
   `carryForwardDeps = phaseGraph.dependencies()`
8. **Result:** Return `CompilationResult.lifecycle(phases)`

The TS lifecycle compiler is simpler than YAML's because it does not
need `forEach:` expansion or `when:` conditional exclusion per phase —
these are handled by native TS at graph construction time.

**Cross-surface rules for lifecycle:** Currently not supported — matching
YAML's `createYamlLifecycleGoalCompiler()` which also does not receive
cross-surface rule/invariant parameters. Adding cross-surface support
for lifecycle graphs is tracked as a separate concern (applies to both
YAML and TS surfaces).

## 8. Type Generation Pipeline

### 8.1 Source Data

The Jandex index already scans `@NodeTypeId`-annotated classes and builds
the `NodeSpecRegistry` map (`type string → spec class name`). The type
generator reads the same Jandex data.

### 8.2 Java Record → TypeScript Interface

For each `@NodeTypeId`-annotated `NodeSpec` implementation:

1. Read the record components (or bean properties) from Jandex
2. Map Java types to TypeScript types:

| Java type | TypeScript type |
|-----------|----------------|
| `String` | `string` |
| `int`, `long`, `double`, `Integer`, `Long`, `Double` | `number` |
| `boolean`, `Boolean` | `boolean` |
| `List<T>` | `T[]` |
| `Map<String, T>` | `Record<string, T>` |
| `Optional<T>` | `T \| undefined` (optional property) |
| Enum | String union (`'A' \| 'B' \| 'C'`) |
| Nested record | Nested interface |

3. Generate a TypeScript interface with the same field names
4. Generate the `NodeTypeMap` entry mapping type string → interface

### 8.3 Generation Tool

A Maven plugin goal (`casehub-ts-typegen:generate`) or a Quarkus build step
that runs during the `generate-sources` phase. Reads Jandex index, writes
`.d.ts` files to a configurable output directory.

The generator lives in `ts-core` (platform) — it's not desiredstate-specific.
Any repo with `@NodeTypeId`-annotated types can generate `.d.ts` files.

### 8.4 Output

```
generated/
├── node-type-map.d.ts    # NodeTypeMap + all spec interfaces
└── index.d.ts            # Re-exports
```

The generated types are checked into the TS SDK package or published as
part of the npm package. They are NOT generated on-the-fly during IDE
usage — they are pre-generated artifacts that IDE TypeScript services
consume for autocomplete.

## 9. Example — Medallion Pipeline in TypeScript

Side-by-side companion to `examples/pipeline-yaml/`.

```typescript
// examples/pipeline-ts/src/medallion-pipeline.ts
import { defineGraph, node } from '@casehub/desiredstate';

const regions = ['us-east', 'eu-west', 'ap-south'];

export default defineGraph({
    namespace: 'pipeline',
    name: 'medallion',
    nodes: {
        // Bronze tier — one source per region (programmatic stamping)
        ...Object.fromEntries(regions.map(region => [
            `source-${region}`,
            node('data-source', {
                uri: `s3://${region}/customers.csv`,
                format: 'CSV',
                batchSize: 1000,
            }),
        ])),

        // Silver tier — schema validation
        'csv-ingest': node('transformer', {
            operation: 'VALIDATE',
        }, { dependsOn: regions.map(r => `source-${r}`) }),

        // Gold tier — warehouse sink
        'warehouse-sink': node('sink', {
            target: 'warehouse',
            writeMode: 'APPEND',
        }, { dependsOn: ['csv-ingest'], humanGating: 'PROVISION_ONLY' }),
    },
});
```

**What this demonstrates:**
- `node()` helper infers literal types — no `as const` needed
- `regions.map()` replaces YAML's `forEach:` — native TS iteration
- `Object.fromEntries()` + spread builds the node map programmatically
- `node('data-source', { ... })` → TypeScript infers `T = 'data-source'`,
  narrowing `spec` to `DataSourceSpec` with full autocomplete
- `dependsOn: regions.map(r => ...)` generates dynamic dependency lists
- `humanGating` on the sink — same feature as YAML/annotations

**What it does NOT include:**
- No `rules:` or `invariants:` — these come from a Java `@GraphRule`
  class via cross-surface resolution
- No `faultPolicy:` — declared via Java annotation classes or CDI beans

The companion Java `@GraphRule` class provides the monitoring rule that
fires against the TS-declared graph:

```java
@GraphRule(graph = {"pipeline:*"})
public class EnsureMonitoringRule {
    @RuleMatch       NodeHandle sink = NodeHandle.ofType("sink");
    @RuleNotExists   NodeHandle guard = NodeHandle.ofType("monitor")
                                           .of(sink).direction(DEPENDENTS);

    @RuleAction
    List<GraphMutation> fire() {
        return List.of(
            GraphMutation.addNode("monitor-" + sink.flatId(),
                new MonitorSpec(sink.id())),
            GraphMutation.addDependency(
                "monitor-" + sink.flatId(), sink.id()));
    }
}
```

The `graph = {"pipeline:*"}` pattern matches the TS-declared graph
(`pipeline:medallion`) via `GraphPatternMatcher`. The annotation
processor produces a `StandaloneRuleBuildItem`, which
`CrossSurfaceRuleResolutionStep` delivers to the TS graph's
`GoalCompiler` via `AdditionalRulesBuildItem`.

**YAML standalone rules:** YAML rules with `graph:` patterns
(e.g., `graph: ["pipeline:*"]`) will also work as cross-surface rules
once the YAML processor produces `StandaloneRuleBuildItem` for them —
this is part of #116 YAML language extensions.

## 10. Lifecycle Phase Example

```typescript
// multi-phase-deployment.ts
import { defineLifecycle, node } from '@casehub/desiredstate';

export default defineLifecycle({
    namespace: 'webapp',
    name: 'full-stack',
    phases: [
        {
            id: 'infrastructure',
            completionCondition: 'allPresent',
            nodes: {
                'database': node('db', { engine: 'postgres', version: '15' }),
                'cache': node('cache', { engine: 'redis' }),
            },
        },
        {
            id: 'application',
            completionCondition: 'allPresent',
            nodes: {
                'api-server': node('app', { image: 'api:latest' },
                    { dependsOn: ['database'] }),
            },
        },
        {
            id: 'observability',
            completionCondition: 'never',
            nodes: {
                'monitor': node('monitor', { target: 'api-server' },
                    { dependsOn: ['api-server'] }),
            },
        },
    ],
});
```

**Cross-phase carry-forward:** `dependsOn: ['database']` in the
application phase references a node from the infrastructure phase.
The `TsGraphRecorder.createTsLifecycleGoalCompiler()` implements
carry-forward semantics (§7.6):
- Infrastructure phase nodes (`database`, `cache`) carry forward into
  the application phase graph
- Application phase nodes (`api-server` + carried `database`, `cache`)
  carry forward into the observability phase
- Phase-declared nodes override carry-forward nodes with the same ID
- Dependencies carry forward only when both endpoints exist in the
  merged graph

**Per-phase invariant validation:** Invariants (inline or cross-surface)
are validated against each phase's merged graph independently. An
invariant that requires a `monitor` dependent on every `sink` would pass
in the observability phase (where monitors exist) but could fail in
earlier phases if the invariant applies globally. This matches
`YamlGraphRecorder.createYamlLifecycleGoalCompiler()`'s per-phase
`GraphInvariantEngine.validate()` call.

## 11. Testing Strategy

### 11.1 ts-core (platform)

- **TsExecutor SPI:** Unit tests for both implementations
  - `TsjTsExecutor`: evaluate simple TS → JSON, type errors, runtime errors
  - `NodeTsExecutor`: same test suite via subprocess
  - Equivalence: both implementations produce identical JSON for the same input
- **Type generator:** Unit tests for Java→TS type mapping
  - Records, enums, nested types, optionals, generics
  - Edge cases: cyclic types, Java-specific types with no TS equivalent

### 11.2 ts-dsl (desiredstate)

- **TsDesiredStateProcessor:** Build-time validation tests
  - Unknown node types → build failure
  - Dangling dependency references → build failure
  - Cycle detection → build failure
  - Namespace collision with YAML/annotation graphs → build failure
  - Valid single graph → GoalCompiler bean registered
  - Valid lifecycle graph → GoalCompiler with phases
- **Cross-surface integration:** End-to-end tests
  - TS graph + YAML rules → rules fire against TS graph
  - TS graph + annotation rules → rules fire against TS graph
  - TS lifecycle + cross-surface invariants → per-phase validation
- **Pipeline-ts example:** Integration test
  - Medallion pipeline in TS produces the same expanded graph as
    the YAML and annotation companions
  - Cross-surface monitoring rule fires correctly

## 12. Decisions

| # | Decision | Rationale | Alternative | Why not |
|---|----------|-----------|-------------|---------|
| D1 | Scope: graph construction + lifecycle + LLM target; rules/invariants via cross-surface | TS's value is programmatic construction; declarative concerns stay in declarative surfaces | Full parity | Reimplements YAML in TS syntax without added value |
| D2 | TsExecutor SPI with TSJ (evaluate) + Node.js (fallback) | Pre-release evaluation behind an abstraction — if TSJ doesn't work, SPI delegates to Node.js | External compilation only | Overly conservative for pre-release; blocks runtime evaluation |
| D3 | Imperative DSL style — native TS constructs only | TS developers chose TS for programmatic control; declarative sub-DSL contradicts the persona | Hybrid imperative+declarative | Rules/invariants excluded from TS scope (D1) |
| D4 | Object literal / defineGraph() with NodeTypeMap discriminated unions | Mirrors YAML shape, plays to TS structural typing, LLMs generate object literals reliably | Functional composition, fluent builder | Less idiomatic TS (builders), harder LLM generation (functions) |
| D5 | Generated .d.ts from Jandex @NodeTypeId scan | Auto-synced with Java, NodeTypeMap enables spec autocomplete keyed on node type | Manual TS types | Drift risk |
| D6 | Programmatic power + type safety primary; LLM secondary; consumer decides | For static declarations YAML is simpler; TS enables declarations YAML can't express | Position TS as superior | Neither is universally better |
| D7 | JSON envelope (single/lifecycle) → TsDesiredStateProcessor | Converges at IR level; parallels YamlDesiredStateProcessor | TS emits YAML | YAML error messages for TS errors; loses TS source context |
| D8 | ts-core in platform, ts-dsl in desiredstate | yaml-core pattern — shared execution infra in platform, domain logic in consuming repos | Everything in desiredstate | Other repos can't reuse TS execution |

## 13. Deferred Items

**Drools integration (#119):** If the Drools backend is adopted for
YAML rules, TS-declared graphs benefit automatically via cross-surface
resolution — no TS-side changes needed.

**Runtime TS evaluation for LLM-generated graphs:** The `TsExecutor` SPI
supports runtime evaluation. The use case (how dynamic TS enters the
system, security boundaries, reconciliation loop interaction) needs its
own design spec when the need is validated.

**Visual editor integration:** The visual editor operates on YAML as its
serialisation format. A TS-authored graph could be rendered in the editor
by serializing the JSON envelope to YAML — but this is a visual editor
concern, not a TS DSL concern.

**Fault policy declaration for TS graphs:** TS-declared graphs acquire
fault policies via Java annotation classes (`@FaultPolicyDef`) or custom
CDI `FaultPolicy` beans. The annotation processor registers these
without `@DesiredStateQualifier`, making them globally available to all
reconciliation loops including TS-declared graphs. Declarative companion
fault policies (YAML file targeting a TS graph) are not currently
supported — the YAML processor requires a full graph declaration and
registers policies with a qualifier scoped to that graph. A cross-surface
fault policy resolution mechanism (analogous to
`CrossSurfaceRuleResolutionStep` for rules) would address this gap.

**Cross-surface rules for lifecycle phases:** Neither YAML nor TS
lifecycle compilers currently receive cross-surface rules/invariants.
`YamlGraphRecorder.createYamlLifecycleGoalCompiler()` only evaluates
inline rules and invariants per phase. Adding cross-surface rule support
to lifecycle compilers is a cross-surface concern that should be
addressed for both YAML and TS simultaneously.

**Pre-compiled JSON support:** The processor supports `.ds.json` files
for CI/CD pipelines where TS compilation happens before Maven. The
compilation tooling (npm scripts, CLI wrapper) needs its own design.

## 14. References

- `annotations/runtime/.../GraphDescriptor.java` — IR record
- `annotations/runtime/.../NodeDescriptor.java` — sealed node descriptor
- `yaml/runtime/.../YamlGraphRecorder.java` — YAML GoalCompiler factory (pattern for TsGraphRecorder)
- `yaml/deployment/.../YamlDesiredStateProcessor.java` — YAML build pipeline (pattern for TsDesiredStateProcessor)
- `yaml/runtime/.../model/YamlGraph.java` — YAML model (structural reference)
- `annotations/deployment/.../CrossSurfaceRuleResolutionStep.java` — rule delivery to non-annotation graphs
- `runtime/.../LifecycleManager.java` — phase orchestration
- `platform/yaml-core/` — shared YAML infrastructure (pattern for ts-core)
- Research doc `docs/research/2026-08-27-operator-declaration-language-research.md` — §5.2 TypeScript DSL
- YAML language extensions spec `docs/specs/issue-116-yaml-language-design/2026-08-27-yaml-language-extensions-design.md`
- #116 — operator-first declaration language
- #122 — this issue
- #124 — cross-surface rule resolution
- #121 — lifecycle hooks
- #119 — Drools backend (deferred)
