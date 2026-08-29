# Decisions — #122 TypeScript DSL

## D1: Scope

**Choice:** Programmatic graph construction + typed specs + lifecycle phases + LLM generation target. Rules, invariants, and fault policies are NOT in scope — they remain in YAML/Java surfaces and are delivered to TS-declared graphs via cross-surface resolution (#124) after the `CrossSurfaceRuleResolutionStep` source filter is broadened (currently YAML-only, see note).
**Alternatives:**
- Full parity — reimplements all YAML declarative features (forEach, when, rules, invariants, fault policies, lifecycle phases, modules) in TS syntax
- Programmatic generation only — no rules/invariants even via cross-surface
- LLM generation only — .d.ts as context, SDK as generation target
- TS as meta-generator — emits YAML fed into existing YAML pipeline
**Rationale:** TS's value proposition is programmatic graph construction with type-safe specs — the capability YAML can't express (D6). forEach/when/modules have natural TS equivalents (loops, conditionals, functions/imports) that are more powerful than DSL-level reimplementations. Rules and invariants are declarative concerns — they belong in YAML or Java annotations. `CrossSurfaceRuleResolutionStep` delivers standalone `@GraphRule`/`@GraphInvariant` classes to non-annotation graphs via `GraphPatternMatcher`; extending it to TS sources requires changing the source filter from `graph.source().startsWith("yaml:")` to exclude annotation sources (`!graph.source().startsWith("annotation:")`), a trivial but required change. Lifecycle phases are in scope because they are structural graph construction, not declarative rules — TS handles carry-forward naturally via shared variable references between phase graphs, and multi-phase deployments are a core value proposition (research doc §4.3).
**Trade-offs:** TS authors who need graph rules must declare them in YAML or Java annotations. This is the intended boundary — declarative concerns in declarative surfaces. Lifecycle phases ARE in scope — excluding them would limit TS to `CompilationResult.single()` graphs, a significant limitation for the multi-phase deployment use case.
**Sources:** Issue #122, research doc §5.2, CrossSurfaceRuleResolutionStep (line 31 — source filter), GraphPatternMatcher, CompilationResult sealed interface
**Exploration:** quick → revised via review (R1-03, R2-01, R2-02)
**Status:** revised

## D2: Integration path

**Choice:** External compilation — TS executes in its native runtime (Node.js, Deno, or Bun), emits GraphDescriptor-compatible JSON, consumed at Quarkus build time via classpath scan at `META-INF/desiredstate/`.
**Alternatives:**
- TSJ (ts2jvm) — TypeScript compiled to JVM bytecode. Experimental, single-maintainer, no visible community. Original choice — revised.
- GraalJS + esbuild transpile — Oracle-maintained JS engine, mature. Indirection is a single fast transpilation step, not a fundamental limitation. Valid for JVM-embedded use case if that need materialises.
- Javet (V8 binding) — full Node.js but heavyweight JNI dependency
- REST endpoint — TS compiles offline, posts JSON to Quarkus app
- Build-time classpath only with JVM-embedded runtime — requires a validated JVM-embedded TS engine
- Kotlin Script as DSL host — native JVM, type-safe builders (Gradle pattern), IntelliJ support. Valid alternative but serves the JVM developer persona, which already has the Java annotation surface. TS targets DevOps engineers who know Pulumi/CDK and benefits from LLM training data coverage.
**Rationale:** The YAML surface demonstrates the pattern: files on classpath processed at build time by `YamlDesiredStateProcessor`. A `TsDesiredStateProcessor` follows the identical pattern — reads JSON, produces `GraphDescriptor`, feeds `GoalCompiler`. TS in its native runtime has full ecosystem access (npm packages, native TS toolchain, IDE support). No experimental JVM dependency. The compilation boundary is clean — the TS toolchain is a build prerequisite, not a runtime dependency.
**Trade-offs:** No runtime TS evaluation. LLM-generated TS requires a build cycle. Runtime evaluation was the only justification for JVM-embedded execution — but that use case is undesigned (no specification for how dynamic TS enters the JVM, what security boundary applies, or how it interacts with the reconciliation loop). Deferring runtime evaluation until the use case is validated is the right call.
**Sources:** User input (TSJ identified), YamlDesiredStateProcessor pattern, research doc §5.2
**Exploration:** quick → revised via review (R1-02, R1-04, R1-09)
**Status:** revised

## D3: DSL style

**Choice:** Imperative graph construction only — native TS loops, conditionals, functions, and the typed graph builder API. No declarative sub-DSL for rules or invariants.
**Alternatives:**
- Hybrid — imperative graph construction with declarative pattern-matching API for rules/invariants (original choice — revised)
- Fully imperative — rules/invariants as typed TS functions with `defineRule()` wrapper
- Fully declarative — mirror YAML's forEach/when/modules as TS DSL concepts
**Rationale:** D1 revision narrows TS scope to programmatic construction — rules and invariants are excluded from the TS surface. The TS audience is programmers who chose TS for programmatic control. A declarative sub-DSL within their programmatic environment contradicts their preference (research doc §2.3: "the moment you hand someone a `.ts` file with `import` statements, the 'configuration, not programming' psychological contract breaks"). Cross-surface resolution (#124) applies rules and invariants from YAML/Java annotations to TS-declared graphs — no reimplementation needed.
**Trade-offs:** If a future use case validates rules-in-TS, the `defineRule()` imperative approach (typed functions returning `GraphMutation[]`) would be the right style — not a declarative pattern vocabulary.
**Sources:** Research doc §2.3, CrossSurfaceRuleResolutionStep, D1 revision
**Exploration:** quick → revised via review (R1-05)
**Status:** revised

## D4: SDK API design

**Choice:** Object literal / `defineGraph()` with NodeTypeMap discriminated unions
**Alternatives:**
- Functional composition — nodes as first-class values, composed with helper functions
- Fluent builder — method chaining (Java-style)
**Rationale:** Object literal shape mirrors YAML (reduces cognitive load across surfaces). TypeScript structural typing provides deep validation via discriminated unions on the `type` field — spec autocomplete narrows automatically. LLMs generate object literals more reliably than function chains. Native TS (spread, map, conditionals) provides programmatic power without DSL-specific constructs.
**Trade-offs:** Less composable than functional approach for very complex programmatic patterns. Node ID cross-references validated at build time, not compile time.
**Sources:** Zod, tRPC, Drizzle (TS ecosystem precedent for object literal APIs)
**Exploration:** deep-analysis
**Status:** captured

## D5: Type safety mechanism

**Choice:** Generated .d.ts from Jandex-scanned `@NodeTypeId`-annotated NodeSpec types, producing a NodeTypeMap discriminated union. Generation source is the same Jandex scan data that populates `NodeSpecRegistry` at build time. .d.ts files provide IDE autocomplete and type checking during TS authoring — decoupled from D2 (works regardless of where TS executes).
**Alternatives:**
- Manual TS types — hand-written, drift risk
- Untyped spec maps — Map<string, unknown>, weakest safety
- JSON Schema intermediate → TypeScript types via `json-schema-to-typescript` — adds a multi-consumer format (also usable for YAML spec validation, visual editor forms) at the cost of an extra intermediate and toolchain dependency
- OpenAPI spec generation → TypeScript client — only relevant if GraphDescriptor submission is via REST (not the case with external compilation)
- NodeSpecRegistry as generation source — functionally identical to Jandex/@NodeTypeId since NodeSpecRegistry is populated from the same Jandex scan
**Rationale:** Auto-generated types stay in sync with Java. The `NodeTypeMap` enables spec-level autocomplete keyed on node type — the primary DX advantage over YAML. Generation from Jandex data aligns with how the YAML surface discovers types. JSON Schema as an intermediate is valid for multi-consumer scenarios but adds a layer without clear benefit for Phase 4 scope where .d.ts is the only consumer.
**Trade-offs:** Requires a Quarkus build step that emits .d.ts from Jandex data. JSON Schema could be added as a parallel output of the same generation step in a future phase if YAML spec validation or visual editor form generation needs it.
**Sources:** GraphDescriptor.java, NodeDescriptor.java, @NodeTypeId, NodeSpecRegistry
**Exploration:** quick → revised via review (R1-06)
**Status:** revised

## D6: Positioning relative to YAML

**Choice:** Programmatic power and type safety are the primary value propositions. LLM generation is a secondary benefit. Consumer decides which surface.
**Alternatives:**
- Position TS as the primary LLM target, YAML for humans
- Position TS as strictly superior to YAML
**Rationale:** For static declarations, YAML is simpler and sufficient. TS enables declarations YAML can't express (programmatic graph construction). The .d.ts files add value as LLM context regardless of output format.
**Trade-offs:** Neither surface is positioned as "preferred" — consumer must understand when each is appropriate
**Sources:** Research doc §2.3 (Ansible observation about psychological contract), §5.2
**Exploration:** quick
**Status:** captured

## D7: Compilation target

**Choice:** TS compiles to a JSON envelope containing GraphDescriptor-compatible data, consumed by a `TsDesiredStateProcessor` build step. Not to YAML. The envelope supports two variants: `"kind":"single"` (wrapping one GraphDescriptor) and `"kind":"lifecycle"` (wrapping multiple phase entries, each with an id, completionCondition, and GraphDescriptor). This parallels the YAML surface's branching between `createYamlGoalCompiler` and `createYamlLifecycleGoalCompiler`.
**Alternatives:**
- TS emits YAML → feeds existing `YamlDesiredStateProcessor` — reuses YAML infrastructure but produces YAML error messages for TS authoring errors, loses line-number context, serialises programmatic constructs to YAML strings awkwardly
- TS emits directly to `DesiredStateGraph` API — bypasses the IR, loses cross-surface rule resolution
- TS emits bare GraphDescriptor JSON (no envelope) — simpler but cannot represent `CompilationResult.Lifecycle`; limits TS to single-graph deployments
**Rationale:** TS → JSON envelope converges at the IR level while keeping source-level concerns separate. A `TsDesiredStateProcessor` build step (parallel to `YamlDesiredStateProcessor`) reads JSON from `META-INF/desiredstate/`, produces `DesiredStateGraphBuildItem` entries. For lifecycle variants, the processor produces a GoalCompiler that returns `CompilationResult.lifecycle()` — the same downstream `LifecycleManager` orchestration used by YAML lifecycle graphs. `CrossSurfaceRuleResolutionStep` then matches standalone rules/invariants to TS-declared graphs via `GraphPatternMatcher` — requiring the source filter broadening noted in D1. The lifecycle data (phase id, completionCondition) lives in the envelope, not in `GraphDescriptor` itself — consistent with how the YAML surface handles lifecycle outside `GraphDescriptor` via `YamlGraph.lifecycle()`.
**Trade-offs:** A new build step is needed. The step parses JSON into `GraphDescriptor` (for single graphs) or into per-phase `GraphDescriptor` + phase metadata (for lifecycle). All downstream infrastructure (GoalCompiler, rule engines, invariant engines, LifecycleManager) is reused. The envelope format is TS-specific but structurally simple.
**Sources:** YamlDesiredStateProcessor, YamlGraphRecorder.createYamlLifecycleGoalCompiler(), CrossSurfaceRuleResolutionStep (#124), GraphDescriptor sealed evolution (D11 YAML decisions), CompilationResult sealed interface
**Exploration:** surfaced via review (R1-10), refined (R2-02)
**Status:** revised
