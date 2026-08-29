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
| Fault policies | ✗ (via cross-surface) | ✓ (`faultPolicy:`) | ✓ (`@FaultPolicyDef`) |
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

### 4.4 CDI Integration

```java
@DefaultBean @ApplicationScoped
public class DefaultTsExecutor implements TsExecutor {
    // Tries TsjTsExecutor first. If TSJ is not on classpath,
    // falls back to NodeTsExecutor.
}
```

`TsjTsExecutor` is classpath-activated — present only when the TSJ
dependency is included. `NodeTsExecutor` is always available as fallback.
CDI priority ensures TSJ wins when both are present.

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

### 5.2 Core API — `defineGraph()`

```typescript
export function defineGraph(def: GraphDef): GraphEnvelope {
    return { kind: 'single', ...def };
}

export function defineLifecycle(def: LifecycleDef): LifecycleEnvelope {
    return { kind: 'lifecycle', ...def };
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

export type DependencyRef = string | { node: string; optional: boolean };

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
    condition: string;
    timeout?: string;
    message?: string;
}

export interface NotifyStep {
    sink: string;
    event: string;
    data?: Record<string, unknown>;
}

export interface WaitStep {
    duration: string;
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

### 5.6 Dependencies

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

The `dependsOn` shorthand on `NodeDef` is syntactic sugar — the SDK expands
`dependsOn: ['csv-source']` on node `transformer` into
`{ from: 'transformer', to: 'csv-source' }` in the envelope.

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
| `nodes[].type` | Resolved via `NodeSpecRegistry` | Type string → spec class |
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

### 7.4 Build Items

The processor produces:
- `DesiredStateGraphBuildItem` with `source: "ts:<fileName>"` — feeds
  `CrossSurfaceRuleResolutionStep` and cross-surface duplicate detection
- `SyntheticBeanBuildItem` for `GoalCompiler<Void>` — same pattern as YAML

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
import { defineGraph } from '@casehub/desiredstate';
import type { DataSourceSpec, TransformerSpec, SinkSpec } from '@casehub/desiredstate/generated';

const regions = ['us-east', 'eu-west', 'ap-south'];

export default defineGraph({
    namespace: 'pipeline',
    name: 'medallion',
    nodes: {
        // Bronze tier — one source per region (programmatic stamping)
        ...Object.fromEntries(regions.map(region => [
            `source-${region}`,
            {
                type: 'data-source' as const,
                spec: {
                    uri: `s3://${region}/customers.csv`,
                    format: 'CSV' as const,
                    batchSize: 1000,
                },
            },
        ])),

        // Silver tier — schema validation
        'csv-ingest': {
            type: 'transformer' as const,
            dependsOn: regions.map(r => `source-${r}`),
            spec: {
                operation: 'VALIDATE' as const,
            },
        },

        // Gold tier — warehouse sink
        'warehouse-sink': {
            type: 'sink' as const,
            dependsOn: ['csv-ingest'],
            humanGating: 'PROVISION_ONLY' as const,
            spec: {
                target: 'warehouse',
                writeMode: 'APPEND' as const,
            },
        },
    },
});
```

**What this demonstrates:**
- `regions.map()` replaces YAML's `forEach:` — native TS iteration
- `Object.fromEntries()` + spread builds the node map programmatically
- Type narrowing on `'data-source' as const` → spec autocompletes correctly
- `dependsOn: regions.map(r => ...)` generates dynamic dependency lists
- `humanGating` on the sink — same feature as YAML/annotations

**What it does NOT include:**
- No `rules:` or `invariants:` — these come from a companion YAML file
  or Java `@GraphRule` class via cross-surface resolution
- No `faultPolicy:` — same, declared in YAML or annotations

The companion YAML file `ensure-monitoring.yaml` provides the monitoring
rule that fires against the TS-declared graph:

```yaml
desiredState:
  namespace: pipeline
  name: monitoring-rules

rules:
  ensure-monitoring:
    graph: ["pipeline:*"]
    match:
      sink: { type: sink }
    notExists:
      guard: { type: monitor, of: sink, direction: dependents }
    actions:
      - addNode:
          id: "monitor-${match.sink.flatId}"
          type: monitor
          spec:
            target: "${match.sink.id}"
      - addDependency:
          from: "monitor-${match.sink.flatId}"
          to: "${match.sink.id}"
```

The `graph: ["pipeline:*"]` pattern matches the TS-declared graph
(`pipeline:medallion`) via `GraphPatternMatcher`. The rule fires at
`GoalCompiler.compile()` time after `CrossSurfaceRuleResolutionStep`
delivers it to the TS graph's `GoalCompiler`.

## 10. Lifecycle Phase Example

```typescript
// multi-phase-deployment.ts
import { defineLifecycle } from '@casehub/desiredstate';

export default defineLifecycle({
    namespace: 'webapp',
    name: 'full-stack',
    phases: [
        {
            id: 'infrastructure',
            completionCondition: 'allPresent',
            nodes: {
                'database': {
                    type: 'db' as const,
                    spec: { engine: 'postgres', version: '15' },
                },
                'cache': {
                    type: 'cache' as const,
                    spec: { engine: 'redis' },
                },
            },
        },
        {
            id: 'application',
            completionCondition: 'allPresent',
            nodes: {
                'api-server': {
                    type: 'app' as const,
                    dependsOn: ['database'],
                    spec: { image: 'api:latest' },
                },
            },
        },
        {
            id: 'observability',
            completionCondition: 'never',
            nodes: {
                'monitor': {
                    type: 'monitor' as const,
                    dependsOn: ['api-server'],
                    spec: { target: 'api-server' },
                },
            },
        },
    ],
});
```

Cross-phase references (`dependsOn: ['database']` in the application phase)
are handled the same way as YAML lifecycle phases — earlier phase nodes are
carried forward into later phases' graphs. The `TsDesiredStateProcessor`
implements the same carry-forward injection as
`YamlGraphRecorder.createYamlLifecycleGoalCompiler()`.

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

**`as const` ergonomics:** The example shows `'data-source' as const`
to trigger discriminated union narrowing. This is a TypeScript ergonomic
wart — without `as const`, the `type` field is widened to `string` and
spec narrowing doesn't fire. Potential solutions: a TypeScript 5.x
`satisfies` pattern, or a helper function that infers literal types.
Investigate during SDK development.

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
- YAML language extensions spec `specs/issue-124-cross-surface-rules/2026-08-27-yaml-language-extensions-design.md`
- #116 — operator-first declaration language
- #122 — this issue
- #124 — cross-surface rule resolution
- #121 — lifecycle hooks
- #119 — Drools backend (deferred)
