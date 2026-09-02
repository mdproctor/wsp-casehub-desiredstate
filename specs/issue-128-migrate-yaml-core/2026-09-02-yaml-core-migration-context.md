# yaml-core Migration — Design Context and Decisions

**Date:** 2026-09-02
**Issues:** #128 (desiredstate migration), platform#252/253/255/256/257/259/266/267
**Status:** Reference — context for implementation

This document captures the design rationale, regression analysis, and architectural
decisions from the brainstorming sessions that shaped the yaml-core migration and
graph-core extraction. It is the authoritative reference for WHY decisions were made.

---

## 1. Why yaml-core exists

yaml-core (`casehub-platform-yaml-core`) was extracted from desiredstate's `yaml/runtime`
to provide shared YAML primitives. Desiredstate never migrated — it still has local
copies of `VariableResolver`, `ForEachExpander`, `Truthiness`, and `IterationGroup`.
New infrastructure (#126 ParameterValidator, #252 module system) went into yaml-core,
compounding the divergence. Migration issue #128 was filed to close this gap.

A follow-up migration issue should have been filed when yaml-core was originally
created. This was a process gap.

---

## 2. Migration regression analysis — 12 concerns

A systematic before/after comparison identified 12 potential regressions from migrating
to yaml-core. Each was addressed through platform API improvements before migration
proceeds.

### VariableResolver (4 concerns)

| # | Concern | Resolution | Platform issue |
|---|---------|-----------|----------------|
| 1 | Construction verbosity | Acceptable — explicit `VariableSource.chain()` is one-time factory | N/A |
| 2 | Module scope ergonomics | `withChainedScope("var", scope::get)` — one call, self-contained | #255 |
| 3 | Deferred prefix error context | `DeferredPrefixHandler` — IMPROVED over local (context-dependent) | #253 |
| 4 | resolveTemplateString elimination | Deferred prefixes produce same "leave as-is" behavior | #253 |

### ForEachExpander (5 concerns)

| # | Concern | Resolution | Platform issue |
|---|---------|-----------|----------------|
| 5 | Lost stamped IDs | `LinkedHashMap<String, E>` — IDs preserved in map keys | #253 |
| 6 | JSON array iteration parsing | `IterationValueExpander` callback — consumer provides parsing | #255 |
| 7 | Two-pass restructure | Design choice — better separation of concerns | N/A |
| 8 | Hook resolution displaced | Neutral — adapter's `stamp()` can invoke hooks | N/A |
| 9 | Dependency wiring displaced | Generic reference rewriting via `getReferences()`/`withReferences()` | #259 |

### ModuleExpander (3 concerns)

| # | Concern | Resolution | Platform issue |
|---|---------|-----------|----------------|
| 10 | Type safety loss | `SectionDeserializer` + `typedSection()` accessor | #255, #266 |
| 11 | Dependency rewriting | `SectionContentRewriter` callback | #255 |
| 12 | Pre/post conversion overhead | One conversion each direction — acceptable | N/A |

**Final verdict:** All 12 concerns resolved. 9 at parity, 3 improved beyond local.
Zero regressions in capability, type safety, or ergonomics.

---

## 3. Prior art — YAML programming languages

A search for existing YAML module/templating systems identified significant overlap:

| System | Overlap with yaml-core |
|--------|----------------------|
| **ytt (Carvel/VMware)** | Variable resolution, modules, schema validation, overlays. Go-based, not embeddable in Java. |
| **CUE** | Constraint-based validation (types ARE constraints), module/package system, standard library. Go-based. |
| **CloudFormation** | Parameter constraints (MinLength, MaxLength, AllowedPattern, MinValue, MaxValue) — nearly identical to our model. |
| **Terraform HCL** | Modules with parameters + outputs, validation blocks, for_each. |
| **Ansible** | Jinja2 templating, roles (modules), variables, conditionals, loops. |

**What's genuinely novel in our system:** Graph rules with pattern matching (fixed-point
loop), graph invariants with cardinality constraints, dependency-aware module expansion,
integration with a continuous reconciliation loop.

**Why we built our own:** The system is embedded in a Java/Quarkus build pipeline. YAML
compiles to domain graphs via Quarkus build extensions at augmentation time. Standalone
tools (Helm, Terraform) can't embed this way. CUE and ytt are Go-based with no Java
implementation. The graph-aware features have no equivalent in any existing system.

---

## 4. Module outputs — design decisions

Module outputs enable cross-module composition: one module's computed values feed
another's parameters. Terraform's `module.<NAME>.<OUTPUT>` pattern.

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Reference syntax | `${module.alias.outputName}` — dedicated prefix | Distinguishes computed outputs from provided parameters. Matches Terraform. |
| Resolution timing | During expansion, import-order-dependent | Enables chaining. Import order = resolution order. No circular refs possible. |
| API surface | `moduleOutputs` raw map + `outputSource()` helper on ExpandedModule | Data for inspection/testing, behaviour for resolver wiring. |
| Typed outputs | `YamlModuleOutput(ParameterType type, String value)` | Design-time validation — build step catches type mismatches between output declarations and consumer parameter types. Runtime ParameterValidator is the safety net. |

---

## 5. Graph-core extraction — architectural decisions

### Reader/Adapter pattern (not interface conformance)

**Decision:** Engines work through `GraphReader<G, N>` adapters, not by requiring
domain types to implement `Graph<N>`. Domain types are untouched — zero coupling.

**Rationale:** Interface conformance (`DesiredNode implements TypedNode`) forces
foundation-tier modules to depend on platform's graph-api. The reader approach has
the same precedent as `ForEachAdapter<E>` in yaml-core.

### GraphView — eliminates parameter threading

**Decision:** `GraphView<N>` wraps graph + reader at the boundary. Engine internals
operate on `GraphView<N>` — single type parameter, single parameter, no reader threading.

**Rationale:** Raw reader threading means every internal method gains `<G, N>` type
parameters plus separate `graph` and `reader` parameters (~25 methods). `GraphView`
encapsulates the reader so engine code is `(GraphView<N> graph)` — nearly identical
to the current `(DesiredStateGraph graph)` signatures. Method bodies are actually
simpler — no `NodeType.of()`/`NodeId.of()` wrapping.

### Two consumer patterns supported

The reader pattern is agnostic to how the consumer structures their graph:

**Pattern 1 — Separated (generic graph + domain payload):**
```java
record GraphNode<T>(String id, String type, T payload) {}
```
Graph structure and domain content are separate. The domain payload is opaque to
the graph layer. A future `platform-graph` module with concrete node/edge/graph
implementations would use this pattern.

**Pattern 2 — Unified (domain class IS the graph node):**
```java
record DesiredNode(NodeId id, NodeSpec spec, HumanGating gating) {}
```
The domain object carries graph identity directly. No separate graph node type.
This is desiredstate's current pattern.

The `GraphReader` adapter absorbs the structural difference. The engine calls
`graph.nodeType(n)` and gets a string regardless of whether it came from a generic
`type` property or a domain-specific `spec.nodeType().value()` chain.

### Desiredstate's current graph implementation

- **Node:** `DesiredNode` — record in `api/` with `NodeId`, `NodeSpec` (domain payload),
  `HumanGating`, `HookDescriptor`. Domain-specific — carries provisioning instructions.
- **Graph:** `ImmutableDesiredStateGraph` — in `runtime/`. Stores `Map<NodeId, DesiredNode>`
  with adjacency sets for dependencies/dependents. Immutable — every mutation returns
  a new versioned graph.
- **Pattern:** Unified — `DesiredNode` IS the graph node. No separate generic graph layer.

After graph-core extraction, `DesiredStateGraphAdapter` maps these to `GraphView<DesiredNode>`.
The domain types don't change.

### Migration path — refactor in place first

1. **Refactor in desiredstate:** Introduce `GraphView<N>`, `GraphReader`, `GraphWriter`
   in `annotations/runtime`. Make engines work through views. Write adapter. All tests pass.
2. **Verify boundary:** Engines have zero imports of `io.casehub.desiredstate.api.*`.
3. **Extract to platform:** Package move to `graph-core`. Mechanical.

Step 1 is where the work and risk are. Step 3 is a package move.

### What moves where

| Component | To | Notes |
|-----------|------|-------|
| `GraphView`, `MutableGraphView`, `GraphReader`, `GraphWriter` | graph-core | View + adapter interfaces |
| `PatternEvaluator`, `PatternMatchingSupport` | graph-core | Generified via view |
| `GraphRuleEngine`, `GraphInvariantEngine` | graph-core | Generified via view |
| `PatternParameterDescriptor`, `PatternKind`, `Direction` | graph-core | Already type-agnostic |
| `GraphViolation` | graph-core | `List<NodeId>` → `List<String>` |
| `GraphMutation<N>` | graph-core | Parameterized on node type |
| `YamlRule`, `YamlInvariant`, converters | yaml-core (graph DSL layer) | YAML → engine IR bridge |
| `@GraphRule`, `@GraphInvariant`, processors | stays in desiredstate | Quarkus-specific |
| Imperative rule/invariant evaluation | stays in desiredstate | Reflection-based, annotation-specific |
| `DesiredStateGraphAdapter` | stays in desiredstate | Reader/writer for domain graph |

### Future: platform-graph

A potential `platform-graph` module in `casehub/platform` with concrete graph data
structure implementations (nodes, edges, directed/undirected graph types). Independent
of `graph-core` — both consumed through the same reader interface. The reader pattern
means this decision can be deferred without affecting the engine extraction.

---

## 6. Platform issue tracker

| # | Title | Scale | Status | What it provides |
|---|-------|-------|--------|-----------------|
| ~~252~~ | Module system | L | Landed | Modules, ParameterValidator, ModuleExpander |
| ~~253~~ | ExpansionResult Map, DeferredPrefixHandler | S | Landed | Clean expansion API |
| ~~255~~ | sourceFor, IterationValueExpander, SectionDeserializer/Rewriter | S | Landed | Zero-regression migration |
| 256 | Module outputs | M | Landed | `${module.alias.output}` chaining |
| 257 | allowedValues + constraintDescription | S | Open | Enum constraints, author error messages |
| ~~259~~ | Generic reference rewriting in ForEachExpander | S | Landed | Eliminates dependency wiring locality regression |
| 266 | API polish — typedSection, output param validation, commaSplit, dead getId | XS | Open | Final quality pass |
| 267 | graph-core — GraphView + reader/adapter pattern | L | Open | Generic graph engines for all repos |
