# TypeScript DSL Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #122 — TypeScript DSL: type-safe graph declarations
**Issue group:** #124, #121, #122 (branch covers all three)

**Goal:** Add a TypeScript declaration surface for desired-state graphs,
enabling programmatic graph construction with type-safe specs via a
`TsExecutor` SPI (TSJ + Node.js fallback) and a Quarkus build extension
that processes TS-produced JSON into GoalCompiler beans.

**Architecture:** TS source files produce JSON envelopes (single or
lifecycle) via `TsExecutor` SPI. A `TsDesiredStateProcessor` Quarkus
build extension discovers `.ts`/`.ds.json` files on classpath, evaluates
them, validates the output, and registers GoalCompiler beans — same
pattern as `YamlDesiredStateProcessor`. Rules/invariants reach TS graphs
via `CrossSurfaceRuleResolutionStep` (source filter broadened). Platform
`ts-core` owns the executor SPI; desiredstate `ts-dsl/` owns the
domain-specific build extension and SDK.

**Tech Stack:** Java 21, Quarkus (build extension + recorder), TypeScript,
TSJ (ts2jvm, experimental), Node.js 20+ (fallback), Jackson, Maven,
npm

## Global Constraints

- Pre-release platform — breaking changes cost nothing
- `ts-core` goes to `casehub-platform` (separate branch + issue needed)
- `ts-dsl/` goes to `casehub-desiredstate` (this branch)
- All Java code uses IntelliJ MCP for editing — never bash Edit/Write
- TDD: failing test → implement → pass → commit
- Cross-repo: update HANDOFF.md when making changes to platform repo

---

## Batch 1: ts-core — TsExecutor SPI + Node.js implementation (casehub-platform)

### Task 1: TsExecutor SPI and NodeTsExecutor

**Repo:** casehub-platform (new branch `issue-122-ts-core`)
**Prerequisite:** Create GitHub issue on casehubio/casehub-platform for ts-core

**Files:**
- Create: `ts-core/pom.xml`
- Create: `ts-core/src/main/java/io/casehub/ts/core/TsExecutor.java`
- Create: `ts-core/src/main/java/io/casehub/ts/core/TsEvalResult.java`
- Create: `ts-core/src/main/java/io/casehub/ts/core/TsError.java`
- Create: `ts-core/src/main/java/io/casehub/ts/core/NodeTsExecutor.java`
- Create: `ts-core/src/main/resources/io/casehub/ts/core/ts-runner.js`
- Test: `ts-core/src/test/java/io/casehub/ts/core/NodeTsExecutorTest.java`
- Test: `ts-core/src/test/resources/valid-graph.ts`
- Test: `ts-core/src/test/resources/invalid-syntax.ts`
- Test: `ts-core/src/test/resources/runtime-error.ts`
- Modify: `pom.xml` (add `ts-core` module)

**Interfaces:**
- Produces: `TsExecutor.evaluate(String tsSource) → TsEvalResult`
- Produces: `TsExecutor.evaluate(Path tsFile) → TsEvalResult`
- Produces: `TsEvalResult(String json, List<TsError> errors)`, `TsEvalResult.success()`
- Produces: `TsError(String message, String file, int line, int column)`
- Produces: `NodeTsExecutor implements TsExecutor`

- [ ] **Step 1: Create ts-core Maven module**

Create `ts-core/pom.xml` with parent `casehub-platform`, Java 21,
test dependencies (JUnit 5, AssertJ). Add `<module>ts-core</module>`
to the parent pom.

- [ ] **Step 2: Write failing test — evaluate valid TS returns JSON**

```java
@Test
void evaluateValidTsReturnsJson() {
    var executor = new NodeTsExecutor();
    var result = executor.evaluate(
        Path.of("src/test/resources/valid-graph.ts"));
    assertThat(result.success()).isTrue();
    assertThat(result.json()).contains("\"kind\":\"single\"");
    assertThat(result.json()).contains("\"namespace\":\"test\"");
}
```

Create `valid-graph.ts`:
```typescript
export default {
    kind: 'single',
    namespace: 'test',
    name: 'simple',
    nodes: [
        { id: 'a', type: 'mock-type', spec: { value: 'hello' } }
    ],
    dependencies: []
};
```

- [ ] **Step 3: Create SPI types**

`TsExecutor` interface, `TsEvalResult` record, `TsError` record —
exactly as specified in §4.1 of the design spec.

- [ ] **Step 4: Implement NodeTsExecutor**

Spawns `node --import tsx <runner.js> <input.ts>` subprocess.

Create `ts-runner.js`:
```javascript
const path = require('path');
const file = process.argv[2];
import(path.resolve(file))
    .then(mod => {
        const result = mod.default ?? mod;
        process.stdout.write(JSON.stringify(result));
    })
    .catch(err => {
        process.stderr.write(JSON.stringify({
            error: err.message,
            stack: err.stack
        }));
        process.exit(1);
    });
```

`NodeTsExecutor`:
- Validates Node.js is on PATH (checked once, cached)
- Runs subprocess with 30s timeout
- Captures stdout as `json`, stderr as error context
- Parses exit code: 0 = success, non-zero = parse stderr for errors

- [ ] **Step 5: Run test — verify pass**

Run: `mvn -pl ts-core test -Dtest=NodeTsExecutorTest`
Expected: PASS (requires Node.js 20+ and tsx on PATH)

- [ ] **Step 6: Write test — TS syntax error produces TsError**

```java
@Test
void evaluateSyntaxErrorReturnsTsError() {
    var executor = new NodeTsExecutor();
    var result = executor.evaluate(
        Path.of("src/test/resources/invalid-syntax.ts"));
    assertThat(result.success()).isFalse();
    assertThat(result.errors()).isNotEmpty();
    assertThat(result.errors().get(0).message()).isNotBlank();
}
```

Create `invalid-syntax.ts`: `export default { unclosed: `

- [ ] **Step 7: Write test — runtime error produces TsError**

```java
@Test
void evaluateRuntimeErrorReturnsTsError() {
    var executor = new NodeTsExecutor();
    var result = executor.evaluate(
        Path.of("src/test/resources/runtime-error.ts"));
    assertThat(result.success()).isFalse();
    assertThat(result.errors()).isNotEmpty();
}
```

Create `runtime-error.ts`: `throw new Error('boom');`

- [ ] **Step 8: Run tests — all pass**

Run: `mvn -pl ts-core test`
Expected: 3 tests PASS

- [ ] **Step 9: Commit**

```bash
git add ts-core/ pom.xml
git commit -m "feat(#122): TsExecutor SPI + NodeTsExecutor implementation

Refs casehubio/casehub-desiredstate#122"
```

---

## Batch 2: TS SDK — npm package with defineGraph, node, types (casehub-desiredstate)

### Task 2: SDK package structure, types, and transformation functions

**Repo:** casehub-desiredstate (this branch)

**Files:**
- Create: `ts-dsl/sdk/package.json`
- Create: `ts-dsl/sdk/tsconfig.json`
- Create: `ts-dsl/sdk/src/index.ts`
- Create: `ts-dsl/sdk/src/types.ts`
- Create: `ts-dsl/sdk/src/transform.ts`
- Create: `ts-dsl/sdk/src/generated/node-type-map.d.ts` (placeholder for pipeline example)
- Test: `ts-dsl/sdk/src/__tests__/defineGraph.test.ts`
- Test: `ts-dsl/sdk/src/__tests__/defineLifecycle.test.ts`
- Test: `ts-dsl/sdk/src/__tests__/node.test.ts`

**Interfaces:**
- Produces: `defineGraph(GraphDef) → GraphEnvelope`
- Produces: `defineLifecycle(LifecycleDef) → LifecycleEnvelope`
- Produces: `node<T>(type: T, spec: NodeTypeMap[T], opts?) → NodeDef`
- Produces: `GraphDef`, `NodeDef`, `LifecycleDef`, `PhaseDef`, etc.
- Produces: `GraphEnvelope`, `LifecycleEnvelope` (wire format)
- Produces: `NodeTypeMap` (generated, extensible)

- [ ] **Step 1: Create npm package scaffold**

`package.json`:
```json
{
    "name": "@casehub/desiredstate",
    "version": "0.1.0",
    "type": "module",
    "main": "dist/index.js",
    "types": "dist/index.d.ts",
    "scripts": {
        "build": "tsc",
        "test": "vitest run"
    },
    "devDependencies": {
        "typescript": "^5.6.0",
        "vitest": "^3.0.0"
    }
}
```

`tsconfig.json`:
```json
{
    "compilerOptions": {
        "target": "ES2022",
        "module": "ES2022",
        "moduleResolution": "bundler",
        "declaration": true,
        "outDir": "dist",
        "rootDir": "src",
        "strict": true,
        "exactOptionalPropertyTypes": true
    },
    "include": ["src"]
}
```

- [ ] **Step 2: Write failing test — defineGraph transforms nodes map to array**

```typescript
import { describe, it, expect } from 'vitest';
import { defineGraph } from '../index';

describe('defineGraph', () => {
    it('transforms nodes map to array with id field', () => {
        const result = defineGraph({
            namespace: 'test',
            name: 'simple',
            nodes: {
                'node-a': { type: 'mock' as const, spec: { value: 'hello' } },
                'node-b': { type: 'mock' as const, spec: { value: 'world' } },
            },
        });

        expect(result.kind).toBe('single');
        expect(result.nodes).toHaveLength(2);
        expect(result.nodes[0]).toEqual({
            id: 'node-a', type: 'mock', spec: { value: 'hello' },
        });
        expect(result.nodes[1]).toEqual({
            id: 'node-b', type: 'mock', spec: { value: 'world' },
        });
        expect(result.dependencies).toEqual([]);
    });

    it('extracts dependsOn into dependencies array', () => {
        const result = defineGraph({
            namespace: 'test',
            name: 'deps',
            nodes: {
                'source': { type: 'mock' as const, spec: {} },
                'sink': {
                    type: 'mock' as const,
                    spec: {},
                    dependsOn: ['source'],
                },
            },
        });

        expect(result.dependencies).toEqual([
            { from: 'sink', to: 'source' },
        ]);
        expect(result.nodes.find(n => n.id === 'sink'))
            .not.toHaveProperty('dependsOn');
    });

    it('merges inline dependsOn with top-level dependencies', () => {
        const result = defineGraph({
            namespace: 'test',
            name: 'merged',
            nodes: {
                'a': { type: 'mock' as const, spec: {} },
                'b': { type: 'mock' as const, spec: {}, dependsOn: ['a'] },
            },
            dependencies: [{ from: 'a', to: 'external' }],
        });

        expect(result.dependencies).toEqual([
            { from: 'b', to: 'a' },
            { from: 'a', to: 'external' },
        ]);
    });
});
```

- [ ] **Step 3: Write types.ts**

All type definitions from spec §5.3 and §5.4: `GraphDef`, `NodeDef`,
`DependencyRef`, `HumanGating`, `NodeHooks`, `HookBlock`, `HookStep`,
`VerifyStep`, `NotifyStep`, `WaitStep`, `LifecycleDef`, `PhaseDef`,
`CompletionCondition`, `DependencyDef`. Plus envelope types:
`GraphEnvelope`, `LifecycleEnvelope`, `EnvelopeNode`, `EnvelopePhase`.

Create placeholder `NodeTypeMap`:
```typescript
// generated/node-type-map.d.ts
export interface NodeTypeMap {
    [type: string]: Record<string, unknown>;
}
```

- [ ] **Step 4: Write transform.ts + index.ts**

`transform.ts`:
```typescript
export function transformNodes(nodeMap: Record<string, NodeDef>): {
    nodes: EnvelopeNode[]; dependencies: DependencyDef[];
} {
    const nodes: EnvelopeNode[] = [];
    const dependencies: DependencyDef[] = [];
    for (const [id, def] of Object.entries(nodeMap)) {
        const { dependsOn, ...rest } = def as NodeDef & { dependsOn?: string[] };
        nodes.push({ id, ...rest });
        if (dependsOn) {
            for (const dep of dependsOn) {
                dependencies.push({ from: id, to: dep });
            }
        }
    }
    return { nodes, dependencies };
}
```

`index.ts`: `defineGraph()`, `defineLifecycle()`, `node()` — exactly as
spec §5.2. Re-exports all types from `types.ts`.

- [ ] **Step 5: Run tests — verify pass**

Run: `npm test` (in ts-dsl/sdk/)
Expected: All tests pass

- [ ] **Step 6: Write test — node() helper infers literal type**

```typescript
describe('node', () => {
    it('returns NodeDef with correct type and spec', () => {
        const n = node('mock', { value: 'hello' });
        expect(n.type).toBe('mock');
        expect(n.spec).toEqual({ value: 'hello' });
    });

    it('passes through opts', () => {
        const n = node('mock', { value: 'x' }, {
            dependsOn: ['other'],
            humanGating: 'PROVISION_ONLY',
        });
        expect(n.dependsOn).toEqual(['other']);
        expect(n.humanGating).toBe('PROVISION_ONLY');
    });
});
```

- [ ] **Step 7: Write test — defineLifecycle transforms phases**

```typescript
describe('defineLifecycle', () => {
    it('transforms phases with carry-forward-compatible structure', () => {
        const result = defineLifecycle({
            namespace: 'test',
            name: 'lifecycle',
            phases: [
                {
                    id: 'infra',
                    completionCondition: 'allPresent',
                    nodes: {
                        'db': { type: 'mock' as const, spec: {} },
                    },
                },
                {
                    id: 'app',
                    completionCondition: 'never',
                    nodes: {
                        'api': {
                            type: 'mock' as const,
                            spec: {},
                            dependsOn: ['db'],
                        },
                    },
                },
            ],
        });

        expect(result.kind).toBe('lifecycle');
        expect(result.phases).toHaveLength(2);
        expect(result.phases[0].nodes).toHaveLength(1);
        expect(result.phases[1].dependencies).toEqual([
            { from: 'api', to: 'db' },
        ]);
    });
});
```

- [ ] **Step 8: Run all tests — verify pass**

Run: `npm test` (in ts-dsl/sdk/)
Expected: All tests pass

- [ ] **Step 9: Commit**

```bash
git add ts-dsl/sdk/
git commit -m "feat(#122): TS SDK — defineGraph, defineLifecycle, node helper

@casehub/desiredstate npm package with typed graph construction
API. NodeTypeMap discriminated union for spec autocomplete.

Refs #122"
```

**Note: Type generation (spec §8) is deferred.** The Jandex → `.d.ts`
generator is an authoring-time tool that produces `NodeTypeMap` for IDE
autocomplete. The build extension pipeline works without it — the Java
side validates specs via Jackson deserialization. The placeholder
`NodeTypeMap` in the SDK allows the pipeline-ts example to use
pre-compiled JSON. Type generation becomes a follow-up task once the
core pipeline is validated end-to-end.

---

## Batch 3: Build Extension — TsDesiredStateProcessor + TsGraphRecorder (casehub-desiredstate)

### Task 3: TsGraphRecorder — JSON envelope to GoalCompiler

**Files:**
- Create: `ts-dsl/runtime/pom.xml`
- Create: `ts-dsl/runtime/src/main/java/io/casehub/desiredstate/ts/TsGraphRecorder.java`
- Create: `ts-dsl/runtime/src/main/java/io/casehub/desiredstate/ts/TsEnvelope.java`
- Create: `ts-dsl/runtime/src/main/java/io/casehub/desiredstate/ts/TsLifecycleEnvelope.java`
- Create: `ts-dsl/runtime/src/main/java/io/casehub/desiredstate/ts/TsEnvelopeNode.java`
- Create: `ts-dsl/runtime/src/main/java/io/casehub/desiredstate/ts/TsEnvelopePhase.java`
- Create: `ts-dsl/pom.xml` (parent for runtime + deployment)
- Test: `ts-dsl/runtime/src/test/java/io/casehub/desiredstate/ts/TsGraphRecorderTest.java`
- Modify: `pom.xml` (root — add ts-dsl module)

**Interfaces:**
- Consumes: `GraphDescriptor` (from annotations/runtime)
- Consumes: `NodeDescriptor.InlineNode` (sealed variant)
- Consumes: `GraphRuleDescriptor`, `GraphInvariantDescriptor` (cross-surface)
- Consumes: `ResolvedInvariant` (from annotations/runtime)
- Produces: `TsGraphRecorder.createTsGoalCompiler(...) → RuntimeValue<GoalCompiler>`
- Produces: `TsGraphRecorder.createTsLifecycleGoalCompiler(...) → RuntimeValue<GoalCompiler>`
- Produces: `TsEnvelope`, `TsLifecycleEnvelope` (Jackson-deserializable records)

- [ ] **Step 1: Create ts-dsl parent + runtime Maven modules**

`ts-dsl/pom.xml`: parent with modules `runtime`, `deployment`.
Dependencies: `casehub-desiredstate-api`, `casehub-desiredstate-annotations`.
`ts-dsl/runtime/pom.xml`: Quarkus runtime library, Jackson.
Add `<module>ts-dsl</module>` to root pom.

- [ ] **Step 2: Write envelope model records**

Jackson-deserializable records for the JSON envelope:

```java
// TsEnvelope.java
@JsonIgnoreProperties(ignoreUnknown = true)
public record TsEnvelope(
    String kind,
    String namespace,
    String name,
    List<TsEnvelopeNode> nodes,
    List<DependencyDescriptor> dependencies
) {}

// TsEnvelopeNode.java
public record TsEnvelopeNode(
    String id,
    String type,
    Map<String, Object> spec,
    HumanGating humanGating
) {
    public TsEnvelopeNode {
        if (humanGating == null) humanGating = HumanGating.NONE;
        if (spec == null) spec = Map.of();
    }
}

// TsLifecycleEnvelope.java
public record TsLifecycleEnvelope(
    String kind,
    String namespace,
    String name,
    List<TsEnvelopePhase> phases
) {}

// TsEnvelopePhase.java
public record TsEnvelopePhase(
    String id,
    Object completionCondition,
    List<TsEnvelopeNode> nodes,
    List<DependencyDescriptor> dependencies
) {}
```

- [ ] **Step 3: Write failing test — single graph GoalCompiler**

```java
@Test
void createTsGoalCompilerProducesSingleGraph() {
    var recorder = new TsGraphRecorder();
    var descriptor = new GraphDescriptor(
        "test", "simple", null, null,
        List.of(new NodeDescriptor.InlineNode("a", MockSpec.class.getName(),
            Map.of("value", "hello"), HumanGating.NONE)),
        List.of(), List.of(), null, List.of(), List.of());

    var compiler = recorder.createTsGoalCompiler(
        descriptor, Map.of("mock-type", MockSpec.class.getName()),
        List.of(), List.of(), List.of()).getValue();

    var result = compiler.compile(null, new ImmutableDesiredStateGraphFactory());
    assertThat(result).isInstanceOf(CompilationResult.SingleGraph.class);
    var graph = ((CompilationResult.SingleGraph) result).graph();
    assertThat(graph.nodes()).hasSize(1);
    assertThat(graph.node(NodeId.of("a"))).isNotNull();
}
```

- [ ] **Step 4: Implement TsGraphRecorder.createTsGoalCompiler()**

Follow `YamlGraphRecorder.createYamlGoalCompiler()` pattern:
1. Materialize `InlineNode` → `DesiredNode` (type registry + ObjectMapper)
2. Build graph via factory
3. Evaluate cross-surface rules (GraphRuleEngine)
4. Validate invariants (GraphInvariantEngine)
5. Return `CompilationResult.single(graph)`

- [ ] **Step 5: Run test — verify pass**

Run: `mvn -pl ts-dsl/runtime test`

- [ ] **Step 6: Write failing test — lifecycle GoalCompiler with carry-forward**

```java
@Test
void createTsLifecycleGoalCompilerCarriesForwardNodes() {
    var recorder = new TsGraphRecorder();
    var envelope = new TsLifecycleEnvelope("lifecycle", "test", "lc",
        List.of(
            new TsEnvelopePhase("infra", "allPresent",
                List.of(new TsEnvelopeNode("db", "mock-type",
                    Map.of("value", "pg"), HumanGating.NONE)),
                List.of()),
            new TsEnvelopePhase("app", "allPresent",
                List.of(new TsEnvelopeNode("api", "mock-type",
                    Map.of("value", "svc"), HumanGating.NONE)),
                List.of(new DependencyDescriptor("api", "db")))
        ));

    var compiler = recorder.createTsLifecycleGoalCompiler(
        envelope, Map.of("mock-type", MockSpec.class.getName()),
        List.of()).getValue();

    var result = compiler.compile(null, new ImmutableDesiredStateGraphFactory());
    assertThat(result).isInstanceOf(CompilationResult.Lifecycle.class);
    var phases = ((CompilationResult.Lifecycle) result).phases();
    assertThat(phases).hasSize(2);

    // Phase 2 should have 'api' + carried-forward 'db'
    var phase2Graph = phases.get(1).graph();
    assertThat(phase2Graph.nodes()).hasSize(2);
    assertThat(phase2Graph.node(NodeId.of("db"))).isNotNull();
    assertThat(phase2Graph.node(NodeId.of("api"))).isNotNull();
}
```

- [ ] **Step 7: Implement createTsLifecycleGoalCompiler()**

Follow spec §7.6 — phase iteration, node materialization, carry-forward
merge, dependency carry-forward, per-phase invariant validation,
completion condition resolution.

- [ ] **Step 8: Run all tests — verify pass**

Run: `mvn -pl ts-dsl/runtime test`

- [ ] **Step 9: Commit**

```bash
git add ts-dsl/
git commit -m "feat(#122): TsGraphRecorder — JSON envelope to GoalCompiler

Single-graph and lifecycle compilers with carry-forward semantics.
Envelope model records for Jackson deserialization.

Refs #122"
```

### Task 4: TsDesiredStateProcessor + CrossSurface filter fix

**Files:**
- Create: `ts-dsl/deployment/pom.xml`
- Create: `ts-dsl/deployment/src/main/java/io/casehub/desiredstate/ts/deployment/TsDesiredStateProcessor.java`
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/CrossSurfaceRuleResolutionStep.java:31` — source filter
- Test: `ts-dsl/deployment/src/test/java/io/casehub/desiredstate/ts/deployment/TsDesiredStateProcessorTest.java`
- Test: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/CrossSurfaceRuleResolutionStepTest.java` (extend)

**Interfaces:**
- Consumes: `TsExecutor` (from ts-core, instantiated directly — §4.4)
- Consumes: `TsGraphRecorder` (from ts-dsl/runtime)
- Consumes: `AdditionalRulesBuildItem` (cross-surface rules)
- Produces: `DesiredStateGraphBuildItem` (source: `"ts:<fileName>"`)
- Produces: `SyntheticBeanBuildItem` for `GoalCompiler<Void>`

- [ ] **Step 1: Create deployment Maven module**

`ts-dsl/deployment/pom.xml`: Quarkus deployment extension dependencies.
Add to `ts-dsl/pom.xml` modules.

- [ ] **Step 2: Write failing test — processor discovers .ds.json on classpath**

```java
@Test
void discoversPrecompiledJsonEnvelope() {
    // Place a valid .ds.json in test resources at
    // META-INF/desiredstate/test-graph.ds.json
    // Verify TsDesiredStateProcessor produces a
    // DesiredStateGraphBuildItem with source "ts:test-graph"
}
```

- [ ] **Step 3: Implement TsDesiredStateProcessor.discoverTsGraphs()**

```java
@BuildStep
@Record(ExecutionTime.RUNTIME_INIT)
void discoverTsGraphs(CombinedIndexBuildItem indexBuildItem,
                      TsGraphRecorder recorder,
                      BuildProducer<SyntheticBeanBuildItem> syntheticBeans,
                      BuildProducer<DesiredStateGraphBuildItem> graphItems,
                      List<AdditionalRulesBuildItem> additionalRuleItems) {
    // 1. Scan classpath for META-INF/desiredstate/*.ts and *.ds.json
    // 2. For .ts: instantiate TsExecutor (§4.4 pattern), evaluate
    // 3. For .ds.json: parse directly
    // 4. Validate (type registry, deps, cycles, specs)
    // 5. Build GraphDescriptor with InlineNode entries
    // 6. Produce DesiredStateGraphBuildItem(source: "ts:<name>")
    // 7. Match AdditionalRulesBuildItem by namespace+name
    // 8. Register GoalCompiler bean via recorder
}
```

Type registry: reuse same `scanNodeTypes()` Jandex pattern from
`YamlDesiredStateProcessor`.

- [ ] **Step 4: Run test — verify pass**

- [ ] **Step 5: Fix CrossSurfaceRuleResolutionStep source filter**

Change line 31 in `CrossSurfaceRuleResolutionStep.java`:

From: `if (!graph.source().startsWith("yaml:")) { continue; }`
To: `if (graph.source().startsWith("annotation:")) { continue; }`

- [ ] **Step 6: Write test — cross-surface rule reaches ts: source graph**

Extend `CrossSurfaceRuleResolutionStepTest`:

```java
@Test
void standaloneRuleMatchesTsGraph() {
    var tsGraph = new DesiredStateGraphBuildItem(
        "ts:medallion", "pipeline", "medallion",
        new GraphDescriptor(...), List.of(), List.of());

    var rule = new StandaloneRuleBuildItem(...); // graph: ["pipeline:*"]

    // Run step, verify AdditionalRulesBuildItem produced for ts:medallion
}
```

- [ ] **Step 7: Run all tests in annotations/deployment**

Run: `mvn -pl annotations/deployment test`
Expected: All existing + new tests pass

- [ ] **Step 8: Commit**

```bash
git add ts-dsl/deployment/ annotations/deployment/
git commit -m "feat(#122): TsDesiredStateProcessor + cross-surface filter broadening

Quarkus build extension discovers .ts/.ds.json on classpath.
CrossSurfaceRuleResolutionStep now includes ts: sources.

Refs #122"
```

---

## Batch 4: Integration — pipeline-ts example + end-to-end tests

### Task 5: Pipeline-ts example with cross-surface rule integration

**Files:**
- Create: `examples/pipeline-ts/pom.xml`
- Create: `examples/pipeline-ts/src/main/resources/META-INF/desiredstate/medallion-pipeline.ds.json`
- Create: `examples/pipeline-ts/src/main/java/io/casehub/desiredstate/example/pipeline/ts/EnsureMonitoringRule.java`
- Test: `examples/pipeline-ts/src/test/java/io/casehub/desiredstate/example/pipeline/ts/PipelineTsTest.java`
- Create: `examples/pipeline-ts/src/main/java/io/casehub/desiredstate/example/pipeline/ts/MockProvisioner.java`
- Modify: `pom.xml` (root — add examples/pipeline-ts module)

**Interfaces:**
- Consumes: `defineGraph()` output (pre-compiled as .ds.json)
- Consumes: `@GraphRule(graph = {"pipeline:*"})` cross-surface matching
- Consumes: NodeSpec types from `examples/pipeline/` module

- [ ] **Step 1: Create pipeline-ts Maven module**

Dependencies: `casehub-desiredstate-ts-dsl-deployment` (build extension),
`casehub-desiredstate` (runtime), `casehub-desiredstate-testing`,
`casehub-desiredstate-example-pipeline` (reuses NodeSpec types).

- [ ] **Step 2: Create pre-compiled .ds.json**

Generate `medallion-pipeline.ds.json` — the JSON envelope equivalent
of the spec §9 example. Pre-compiled because TSJ/Node.js availability
isn't guaranteed in CI.

```json
{
    "kind": "single",
    "namespace": "pipeline",
    "name": "medallion",
    "nodes": [
        { "id": "source-us-east", "type": "data-source",
          "spec": { "uri": "s3://us-east/customers.csv", "format": "CSV", "batchSize": 1000 } },
        { "id": "source-eu-west", "type": "data-source",
          "spec": { "uri": "s3://eu-west/customers.csv", "format": "CSV", "batchSize": 1000 } },
        { "id": "source-ap-south", "type": "data-source",
          "spec": { "uri": "s3://ap-south/customers.csv", "format": "CSV", "batchSize": 1000 } },
        { "id": "csv-ingest", "type": "transformer",
          "spec": { "operation": "VALIDATE" } },
        { "id": "warehouse-sink", "type": "sink",
          "spec": { "target": "warehouse", "writeMode": "APPEND" },
          "humanGating": "PROVISION_ONLY" }
    ],
    "dependencies": [
        { "from": "csv-ingest", "to": "source-us-east" },
        { "from": "csv-ingest", "to": "source-eu-west" },
        { "from": "csv-ingest", "to": "source-ap-south" },
        { "from": "warehouse-sink", "to": "csv-ingest" }
    ]
}
```

- [ ] **Step 3: Write @GraphRule for cross-surface monitoring**

```java
@GraphRule(graph = {"pipeline:*"})
public class EnsureMonitoringRule {

    @Match(type = "sink")
    DesiredNode sink;

    @NotExists(type = "monitor", of = "sink", direction = Direction.DEPENDENTS)
    DesiredNode guard;

    public List<GraphMutation> fire(DesiredStateGraph graph) {
        String monitorId = "monitor-" + sink.id().value().replace('.', '-');
        return GraphMutations.addNodeDependingOn(
            new DesiredNode(NodeId.of(monitorId), NodeType.of("monitor"),
                new MonitorSpec(sink.id().value()), HumanGating.NONE),
            sink.id());
    }
}
```

- [ ] **Step 4: Write failing integration test**

```java
@QuarkusTest
class PipelineTsTest {

    @Inject
    GoalCompiler<Void> compiler;

    @Inject
    DesiredStateGraphFactory graphFactory;

    @Test
    void tsGraphCompilesWithCrossSurfaceMonitoringRule() {
        var result = compiler.compile(null, graphFactory);
        assertThat(result).isInstanceOf(CompilationResult.SingleGraph.class);
        var graph = ((CompilationResult.SingleGraph) result).graph();

        // Original nodes from TS
        assertThat(graph.node(NodeId.of("warehouse-sink"))).isNotNull();
        assertThat(graph.node(NodeId.of("csv-ingest"))).isNotNull();

        // Monitor added by cross-surface @GraphRule
        var monitorNode = graph.nodes().stream()
            .filter(n -> n.type().equals(NodeType.of("monitor")))
            .findFirst();
        assertThat(monitorNode).isPresent();
    }
}
```

- [ ] **Step 5: Create MockProvisioner**

Standard mock provisioner returning `ProvisionResult.success()` for
all node types used in the example (data-source, transformer, sink,
monitor). Follows existing examples' pattern.

- [ ] **Step 6: Run integration test — verify pass**

Run: `mvn -pl examples/pipeline-ts test`
Expected: Quarkus starts, discovers .ds.json, cross-surface rule fires,
test passes.

- [ ] **Step 7: Commit**

```bash
git add examples/pipeline-ts/ pom.xml
git commit -m "feat(#122): pipeline-ts example — TS graph + cross-surface rules

Medallion pipeline declared as pre-compiled JSON envelope.
@GraphRule with graph={\"pipeline:*\"} fires against TS graph.

Closes #122"
```

---

## Batch 5: TSJ Evaluation (casehub-platform — experimental)

### Task 6: TsjTsExecutor — TSJ implementation

**Repo:** casehub-platform (ts-core branch)

**Files:**
- Create: `ts-core/src/main/java/io/casehub/ts/core/TsjTsExecutor.java`
- Test: `ts-core/src/test/java/io/casehub/ts/core/TsjTsExecutorTest.java`
- Modify: `ts-core/pom.xml` (add TSJ dependency, optional)

**Interfaces:**
- Implements: `TsExecutor`
- Same test suite as `NodeTsExecutorTest` — equivalence verification

- [ ] **Step 1: Add TSJ dependency (optional)**

Add TSJ Maven dependency to `ts-core/pom.xml` with `<optional>true</optional>`.
The exact artifact coordinates depend on the library the user identified.

- [ ] **Step 2: Write failing test — same contract as NodeTsExecutor**

```java
@Test
void evaluateValidTsReturnsJson() {
    var executor = new TsjTsExecutor();
    var result = executor.evaluate(
        Path.of("src/test/resources/valid-graph.ts"));
    assertThat(result.success()).isTrue();
    assertThat(result.json()).contains("\"kind\":\"single\"");
}
```

- [ ] **Step 3: Implement TsjTsExecutor**

Uses TSJ API to:
1. Compile TS source to JVM bytecode
2. Execute the compiled module
3. Extract the default export
4. Serialize to JSON via the embedded JS `JSON.stringify()`
5. Capture compilation/runtime errors as `TsError`

Implementation details depend on TSJ's API surface — this task is
explicitly experimental and may require iteration.

- [ ] **Step 4: Write equivalence test**

```java
@Test
void tsjAndNodeProduceSameOutput() {
    var tsj = new TsjTsExecutor();
    var node = new NodeTsExecutor();
    var tsjResult = tsj.evaluate(
        Path.of("src/test/resources/valid-graph.ts"));
    var nodeResult = node.evaluate(
        Path.of("src/test/resources/valid-graph.ts"));

    assertThat(tsjResult.success()).isEqualTo(nodeResult.success());
    // Compare parsed JSON (not string equality — key order may differ)
    var mapper = new ObjectMapper();
    assertThat(mapper.readTree(tsjResult.json()))
        .isEqualTo(mapper.readTree(nodeResult.json()));
}
```

- [ ] **Step 5: Run tests — assess TSJ viability**

Run: `mvn -pl ts-core test -Dtest=TsjTsExecutorTest`

If TSJ works: commit and document findings.
If TSJ fails: document what doesn't work, keep NodeTsExecutor as the
working implementation, file a follow-up issue.

- [ ] **Step 6: Commit**

```bash
git add ts-core/
git commit -m "feat(#122): TsjTsExecutor — experimental TSJ evaluation

TSJ (ts2jvm) implementation of TsExecutor SPI.
Equivalence-tested against NodeTsExecutor.

Refs casehubio/casehub-desiredstate#122"
```

---

## References

- [2026-08-29-typescript-dsl-design.md] — design spec this plan implements
- [annotations/runtime/.../GraphDescriptor.java] — IR record
- [annotations/runtime/.../NodeDescriptor.java:7] — sealed node descriptor
- [yaml/runtime/.../YamlGraphRecorder.java:29] — pattern for TsGraphRecorder
- [yaml/deployment/.../YamlDesiredStateProcessor.java:43] — pattern for TsDesiredStateProcessor
- [annotations/deployment/.../CrossSurfaceRuleResolutionStep.java:31] — source filter to broaden
- [annotations/deployment/.../AdditionalRulesBuildItem.java:9] — cross-surface delivery
- [annotations/deployment/.../DesiredStateGraphBuildItem.java:5] — build item for cross-surface
- [platform/yaml-core/] — pattern for ts-core module placement
- [GitHub #122] — TypeScript DSL epic
- [GitHub #124] — cross-surface rule resolution
- [GitHub #121] — lifecycle hooks
