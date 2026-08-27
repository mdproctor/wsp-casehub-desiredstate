# YAML Language Extensions — Design Spec

**Issue:** #116 — operator-first declaration language
**Date:** 2026-08-27
**Status:** Draft

## 1. Summary

Seven language extensions to the YAML desired-state surface, bringing it to
full parity with Java annotations. Three expose the runtime's differentiating
capabilities (graph rules, invariants, fault policies). One enables lifecycle
phase declarations. Three provide structural composition (conditional inclusion,
cardinality stamping, modules). Together they make YAML the universal operator
surface — no Java required for graph declarations, structural rules, continuous
invariants, or multi-tier fault escalation.

## 2. Background

The YAML surface (#117) currently supports: node declarations with typed specs,
dependency edges, variable substitution via `VariableResolver`, and human gating.
It compiles to `GraphDescriptor` → `GoalCompiler<Void>` → `CompilationResult.single()`.

Missing: graph rules, invariants, fault policies, lifecycle phases, conditional
inclusion, cardinality stamping, and composable modules. These are the features
that make CaseHub desired-state worth choosing over Terraform, Helm, or Ansible.

## 3. Design Principles

1. **YAML is data, not code.** Declarative vocabulary for patterns and actions.
   No expression language, no Turing-completeness. Complex logic uses Java
   annotations — YAML is the operator surface, not a general-purpose DSL.

2. **Same IR.** Everything compiles to `GraphDescriptor` → `GoalCompiler` →
   `DesiredStateGraph` → runtime. YAML rules use the same `PatternMatchingSupport`
   primitives and `GraphRuleEngine` fixed-point loop as annotations.

3. **Build-time validation, runtime expansion.** Structure validated at Quarkus
   build time. `forEach`, `when:`, and variables resolved at `GoalCompiler.compile()`
   time via `VariableResolver`.

4. **Explicit interpolation namespaces.** All `${}` references use prefixes to
   avoid ambiguity (see §4).

5. **Declarative boundary defended.** The interpolation vocabulary is closed.
   Features that push toward expression evaluation belong in Java, not YAML.

## 4. Interpolation Model

All `${}` references use explicit prefixes. No bare variable names. This prevents
collision between variables, pattern bindings, fault context, and iteration variables.

**Breaking change from #117 YAML surface.** The existing `medallion-pipeline.yaml`
uses bare variable names (`${source_uri}`). This spec requires prefixed names
(`${var.source_uri}`). Existing YAML files must be updated. The `VariableResolver`
pattern changes from flat key lookup to prefix-dispatched resolution. This break
is intentional — bare names collide with `${match.*}`, `${each.*}`, and `${fault.*}`
namespaces. The migration is mechanical: prepend `var.` to every existing `${}`
reference.

| Prefix | Scope | Available in | Examples |
|--------|-------|-------------|----------|
| `${var.name}` | Variables block, module parameters, Config fallthrough | Everywhere | `${var.source_uri}`, `${var.batch_size}` |
| `${each.name}` | forEach iteration variable | forEach node specs, dependsOn, when: (within forEach) | `${each.region}` |
| `${match.binding.prop}` | Pattern bindings | Rule actions (including spec values), invariant messages | `${match.sink.id}`, `${match.sink.type}` |
| `${fault.prop}` | Fault event context | Fault policy tier specs | `${fault.nodeId}`, `${fault.detail}`, `${fault.type}` |

**Accessible properties on `${match.binding}`:**

- `.id` — the node's `NodeId` value (string)
- `.type` — the node's `NodeType` value (string)

Spec field access (`.spec.<field>`) is deliberately excluded. YAML rules that
need spec-level data require Java `@GraphRule`. This boundary prevents the
interpolation vocabulary from growing into a property-traversal language — the
path Helm's Go templates took.

**Resolution precedence for `${var.}`:**

1. Module parameters (when inside a module scope — highest priority)
2. Inline `variables:` block
3. MicroProfile Config (`VariableResolver` existing fallthrough)

**Resolution architecture:**

The four prefixes are resolved by different components at different points
in the evaluation pipeline — this is not a scope stack but a delegation
architecture with prefix-based dispatch:

| Prefix | Resolver | Resolution point | Component |
|--------|----------|------------------|-----------|
| `${var.*}` | `VariableResolver` | Compile time (step 7) | `YamlGraphRecorder` |
| `${each.*}` | `VariableResolver` | forEach expansion (step 8) | `YamlGraphRecorder` |
| `${match.*}` | `DeclarativeRuleAdapter` | Rule evaluation (step 12) | `GraphRuleEngine` |
| `${fault.*}` | Template factory closure | Fault time (runtime) | `ThresholdFaultPolicy` |

`VariableResolver` gains prefix routing and a module scope stack:
- Strips the `var.` or `each.` prefix before lookup
- `${var.*}` → module parameters → inline variables → MicroProfile Config
- `${each.*}` → forEach iteration context (bound during expansion)
- Rejects `${match.*}` and `${fault.*}` at compile time with a clear error:
  these prefixes are placeholders validated syntactically but resolved later
  by their respective components
- Unrecognized prefixes are build-time errors

`DeclarativeRuleAdapter` resolves `${match.*}` references against pattern
bindings at rule evaluation time. It receives the binding map from
`PatternEvaluator` and performs string interpolation on action templates.

The fault policy template factory resolves `${fault.*}` references against
`FaultEvent` properties at fault time. The factory lambda captures the
template map and resolves at invocation (same pattern as §8.5).

**`${fault.*}` property mapping:**

| YAML reference | `FaultEvent` accessor | Java type | String representation |
|----------------|----------------------|-----------|---------------------|
| `${fault.nodeId}` | `event.node().value()` | `String` | The `NodeId`'s string value |
| `${fault.type}` | `event.type().name()` | `String` | Enum constant name: `PROVISION_FAILED` (uppercase) |
| `${fault.detail}` | `event.detail()` | `String` | Verbatim fault detail message |

The naming divergence (`${fault.nodeId}` vs `FaultEvent.node()`) is
intentional — `nodeId` is the operator-facing name (what the YAML author
cares about), while `node()` returns a `NodeId` wrapper. The template
resolver calls `.value()` to extract the raw string. `${fault.type}` uses
`FaultType.name()` (uppercase enum constant) because Jackson enum
deserialization expects uppercase by default — the resolved value flows
through `convertValue` into the spec class.

**Breaking change:** The existing `VariableResolver` uses bare `${name}`
references (e.g., `${source_uri}` in the pipeline-yaml example). This spec
mandates `${var.name}` prefixed references. All existing YAML graph files
must be updated. This is a clean break — no deprecation period, no backward
compatibility shim. The platform has no external consumers; the migration is
mechanical (add `var.` prefix to every `${...}` reference in YAML files).
The `VariableResolver` regex changes from `\$\{([^}]+)}` to prefix-aware
dispatch: `var.` → variable lookup, `each.` → forEach scope, `match.` →
rule bindings, `fault.` → fault context. Bare references produce
`UnresolvedVariableException` with guidance: "Use `${var.name}` instead
of `${name}`."

## 5. Node ID Conventions

The `.` character is reserved as the system separator for generated node IDs.
User-declared node IDs (in `nodes:`, forEach template IDs, module `as:` aliases)
must not contain `.`. Validated at build time.

- **Module scoping:** `<alias>.<nodeId>` → `pipe-monitor.monitor`
- **forEach stamping:** `<templateId>.<value>` → `regional-source.us-east`

forEach iteration values also must not contain `.`. Validated at expansion time.

This single-separator convention makes generated IDs unambiguous — the provenance
(module vs forEach) doesn't matter at runtime; the node ID is opaque to the
reconciliation loop.

---

## 6. Features

Ordered by implementation phase. Phase 1 features are low-to-medium complexity
and deliver immediate value. Phase 2 features are high complexity and deliver
the primary differentiators. Phase 3 features are table-stakes catch-up with
known design risks.

---

### Phase 1

#### 6.1 Fault Policy Declarations

**Complexity:** Low
**Differentiator:** Fault-driven adaptation

Maps directly to `FaultPolicyDescriptor` → `ThresholdFaultPolicy.builder()`.
Each tier's review node is a template expanded at fault time.

```yaml
faultPolicy:
  - faultTypes: [PROVISION_FAILED]
    nodeTypes: [transformer, sink]
    ignoreTypes: [ai-review, human-review]
    namespace: pipeline-escalation
    tiers:
      - threshold: 3
        reviewNode:
          type: ai-review
          spec:
            target: "${fault.nodeId}"
            detail: "${fault.detail}"
      - threshold: 5
        reviewNode:
          type: human-review
          humanGating: ALL
          spec:
            target: "${fault.nodeId}"
            detail: "${fault.detail}"
            instruction: "Requires manual review"
```

**Semantics:**

- `faultTypes`, `nodeTypes`, `ignoreTypes`, `namespace` map directly to
  `ThresholdFaultPolicy.builder()` parameters.
- Each tier's `reviewNode` defines the `ReviewSpecFactory` as a template.
  The spec class is resolved via `NodeSpecRegistry` from the `type:` field.
  At fault time, `${fault.*}` variables are resolved against the `FaultEvent`,
  and Jackson `convertValue` deserializes the spec map into the typed
  `NodeSpec` subclass.
- The tier's `type:` field provides `nodeType()` directly — bypassing the
  `ReviewSpecFactory.nodeType()` probe mechanism (same pattern as
  `@Tier(nodeType=...)` in annotations).
- `ignoreTypes` is auto-extended with each tier's output node type (existing
  `ThresholdFaultPolicy` behavior).

**Template deserialization safety:**

The template `ReviewSpecFactory` lambda must catch deserialization failures
and produce a clear error rather than propagating an uncaught exception into
the fault feedback loop. Specifically:

- Catch `IllegalArgumentException` and Jackson `MappingException` from
  `convertValue`. Wrap with context: template node type, fault event details,
  resolved template values, and target spec class name.
- On failure: log the error at WARN and return `List.of()` — the tier is
  skipped for this fault event, allowing fallthrough to higher tiers or
  natural fault re-evaluation on the next cycle. The original fault is NOT
  swallowed — it remains in the reconciliation loop's fault tracking.
- All `${fault.*}` values are guaranteed to be `String` type. The template
  resolver never produces non-string values from fault context.
- The `ObjectMapper` used for template spec deserialization is a dedicated
  instance with scalar coercion explicitly enabled (`CoercionConfig` for
  String-to-number and String-to-boolean). This mapper is captured in the
  `ReviewSpecFactory` lambda closure — it is NOT the application's default
  `ObjectMapper`.

**FaultCountStore injection:**

YAML-originated fault policies inherit the CDI-provided `FaultCountStore`
bean, NOT the inline `InMemoryFaultCountStore` default. Both the YAML
recorder and the annotation recorder pass the CDI-resolved `FaultCountStore`
to `ThresholdFaultPolicy.Builder.faultCountStore()`. This ensures:

- If `JpaFaultCountStore` is on the classpath (displaces `DefaultFaultCountStore`
  via `@DefaultBean`), both YAML and annotation policies use durable storage.
- If no custom store is provided, both surfaces use the CDI `DefaultFaultCountStore`
  (`InMemoryFaultCountStore` subclass) — identical behavior, no silent divergence.
- The annotation recorder's `@Customize` escape hatch can still override the
  store per-policy if needed.

**Architecture changes:**

- `YamlGraphRecorder`: accept `List<YamlFaultPolicy>` and build
  `ThresholdFaultPolicy` beans directly via the builder API. Each tier's
  `ReviewSpecFactory` is a template-based lambda that captures
  `NodeSpecRegistry`, a dedicated coercion-enabled `ObjectMapper`, and the
  CDI `FaultCountStore` in its closure. The YAML path bypasses
  `FaultPolicyDescriptor`/`TierDescriptor` entirely — those types carry
  `reviewMethodName` which is annotation-specific (names a Java method).
  YAML tiers use template-based `ReviewSpecFactory` lambdas instead.
- `YamlDesiredStateProcessor`: parse `faultPolicy:` YAML, build descriptors,
  validate tier types exist in the registry.
- YAML model: add `List<YamlFaultPolicy>` to `YamlGraph`.
- `DesiredStateGraphRecorder.createFaultPolicy()`: inject CDI `FaultCountStore`
  into the builder (currently defaults to inline `InMemoryFaultCountStore`).

---

#### 6.2 Graph Invariants

**Complexity:** Medium
**Differentiator:** Live invariants

Same pattern vocabulary as `@GraphInvariant` annotations. Purely structural
assertions — no custom validation logic.

```yaml
invariants:
  every-sink-has-upstream:
    graph: ["pipeline:*"]
    match:
      sink: { type: sink }
    directDep:
      upstream: { type: transformer, of: sink, direction: dependencies }

  unmonitored-sinks-need-alerter:
    graph: ["pipeline:*"]
    match:
      sink: { type: sink }
    notExists:
      monitor: { type: monitor, of: sink, direction: dependents }
    directDep:
      alert: { type: alerter, of: sink, direction: dependents }
```

**Pattern vocabulary:**

| YAML key | Annotation | Semantics |
|----------|-----------|-----------|
| `match:` | `@Match` | Bind nodes by type (anchor — cross-product enumeration) |
| `directDep:` | `@DirectDep` | Direct neighbor of a bound node |
| `reaches:` | `@Reaches` | Transitively reachable from a bound node |
| `notExists:` | `@NotExists` | Absence guard — narrows scope |

Each entry: `name: { type: <nodeType>, of: <bindingRef>, direction: dependencies|dependents }`

- `type:` specifies the node type to match. Use `"*"` as a wildcard to match
  **any** node type. This is useful for structural assertions that are
  type-agnostic (e.g., "every monitor must have at least one dependency,
  regardless of what it depends on"). The wildcard is supported in all
  pattern kinds: `match:`, `directDep:`, `reaches:`, `notExists:`.
  `PatternMatchingSupport` methods (`findDirectNeighbors`, `findReachable`,
  `existsRelational`, `existsGlobal`) must skip the type filter when
  `"*".equals(p.nodeType())`.
- `of:` references a previously bound name. Required for all patterns except
  the first `match:` entry.
- `direction:` defaults to `dependencies`.
- `graph:` is optional — same semantics as `@GraphInvariant(graph={...})`.
  Supports exact, namespace wildcard (`pipeline:*`), global wildcard (`*:*`),
  and `!`-prefixed exclusions. **When `graph:` is omitted**, the invariant
  applies to the enclosing YAML graph only (scoped to `source:<fileName>` —
  the same graph produced by this YAML file). This matches the annotation
  convention where in-class invariants are scoped to their enclosing
  `@DesiredState` graph.

**Violation details:**

Each `GraphViolation` produced by a YAML invariant carries:
- `invariantName` — the YAML key name (e.g., `every-sink-has-upstream`)
- `sourceClassName` — `"yaml:<fileName>"` (matching the `source` field in
  `DesiredStateGraphBuildItem`)
- `message` — auto-generated: `"<invariantName> violated for [<anchorDesc>]"`
  (matches `GraphInvariantEngine.validateParameterized()` existing format)
- `affectedNodes` — the anchor node IDs from the failing binding tuple

An optional `message:` field on the invariant provides a custom message
template with `${match.*}` interpolation:

```yaml
invariants:
  every-sink-has-upstream:
    message: "Sink ${match.sink.id} has no upstream transformer"
    match:
      sink: { type: sink }
    directDep:
      upstream: { type: transformer, of: sink, direction: dependencies }
```

When `message:` is present, the custom template replaces the auto-generated
message. `${match.*}` references resolve against the anchor bindings that
triggered the violation.

**Semantics:** For every anchor tuple (from `match:` cross-product), all
remaining patterns must be satisfiable. If any pattern fails to bind for an
anchor tuple → `GraphViolation`. This matches `GraphInvariantEngine`'s existing
behavior: empty `expandedArgs` = violation.

**Architecture changes:**

- New `DeclarativeInvariantDescriptor` record: `name`, `graphPatterns`,
  `List<PatternParameterDescriptor>` (reuses existing type).
- New `DeclarativeInvariantAdapter` implementing a `ResolvedInvariant` interface.
  Wraps pattern evaluation using `PatternMatchingSupport` (unchanged).
- `GraphInvariantEngine`: refactor `ResolvedGraphInvariant` from record to
  sealed interface with `ReflectiveInvariant` and `DeclarativeInvariant` variants.
  The invariant engine's `expandChain` and binding resolution are extracted from
  the reflection-specific code path. Pattern matching is already reflection-free
  via `PatternMatchingSupport`.
- Parallel surgery required with `GraphRuleEngine` (§6.4) — both engines share
  the same refactoring pattern.

---

#### 6.3 Conditional Inclusion (`when:`)

**Complexity:** Low
**Value:** Table-stakes (every IaC tool has this)

Node-level boolean condition evaluated at `GoalCompiler.compile()` time.

```yaml
nodes:
  monitoring-dashboard:
    type: dashboard
    when: "${var.monitoring_enabled}"
    spec:
      title: Pipeline Health

  debug-logger:
    type: logger
    when: "${var.debug_mode}"
    dependsOn: [csv-ingest]
    spec:
      level: TRACE
```

**Semantics:**

- `when:` takes a `${var.*}` reference (or `${each.*}` inside a forEach
  node — see §6.6) that resolves to a boolean string.
  Truthy values (case-insensitive): `true`, `yes`, `on`, `y`, `1`.
  Falsy values (case-insensitive): `false`, `no`, `off`, `n`, `0`.
  Any other value is a **build-time error** — not silently treated as false.
  This matches YAML's own boolean vocabulary, so `monitoring_enabled: yes`
  in a YAML variables block works as operators expect.
- Evaluation at `GoalCompiler.compile()` time, after variable resolution.
- When a conditional node is excluded, its outgoing and incoming dependency
  edges are removed from the graph.

**Dependency safety (adversarial fix):**

An unconditional node depending on a conditional node is a **build-time error**,
not a warning. If node A (no `when:`) has `dependsOn: [B]` and B has `when:`,
the build fails with:

> "Node 'A' depends on conditional node 'B' (when: ${var.monitoring_enabled}).
> Either make 'A' conditional with the same or stricter condition, or mark the
> dependency as optional."

Optional dependencies: `dependsOn: [{ node: B, optional: true }]`. When B is
excluded, the optional dependency is silently removed. When B is included, the
dependency is enforced normally.

**Architecture changes:**

- YAML model: add `String when` field to `YamlNode`.
- Node descriptor: add `when` to `NodeDescriptor.InlineNode` — or carry as
  YAML-specific metadata in a `YamlExpansionDescriptor` (see §8).
- `YamlGraphRecorder`: evaluate `when:` conditions during graph construction.
- `YamlDesiredStateProcessor`: validate conditional dependency safety.
- YAML model: extend `dependsOn` to support map form for optional deps:
  `dependsOn: [csv-source, { node: debug-logger, optional: true }]`.

---

### Phase 2

#### 6.4 Graph Rules

**Complexity:** High
**Differentiator:** Structural graph rewriting (#1 differentiator)

Same pattern vocabulary as `@GraphRule` annotations, with a declarative action
model constrained to `GraphMutation` sealed variants.

```yaml
rules:
  ensure-monitoring:
    graph: ["pipeline:*"]
    match:
      sink: { type: sink }
    notExists:
      guard: { type: monitor, of: sink, direction: dependents }
    actions:
      - addNode:
          id: "monitor-${match.sink.id}"
          type: monitor
          spec:
            target: "${match.sink.id}"
      - addDependency:
          from: "monitor-${match.sink.id}"
          to: "${match.sink.id}"
```

**Pattern vocabulary:** Same as invariants (§6.2) — `match:`, `directDep:`,
`reaches:`, `notExists:`. Patterns bind named variables used in actions.

**Action vocabulary** (maps to `GraphMutation` sealed variants):

| YAML key | GraphMutation | Parameters |
|----------|--------------|------------|
| `addNode:` | `AddNode` | `id`, `type`, `spec`, optional `humanGating` |
| `removeNode:` | `RemoveNode` | `id` |
| `updateNode:` | `UpdateNode` | `id`, `type`, complete `spec` (replaces existing) |
| `addDependency:` | `AddDependency` | `from`, `to` |
| `removeDependency:` | `RemoveDependency` | `from`, `to` |

Action parameters support `${match.*}` interpolation for node ID, type,
and spec values. `${var.*}` interpolation is also supported in spec values
for configuration data. Both namespaces are valid in `addNode:` and
`updateNode:` spec blocks — `${match.*}` for referencing matched nodes,
`${var.*}` for injecting configuration.

`updateNode:` requires a **complete** spec — it replaces the existing node
entirely, matching the runtime `GraphMutation.UpdateNode(NodeId, DesiredNode)`
semantics. There is no partial merge. Rules that need to read the current
node's spec and modify a single field require Java `@GraphRule` — this is
a deliberate boundary. Partial merge semantics in a fixed-point loop create
non-deterministic convergence: if multiple rules modify the same node, the
merge result depends on evaluation order and changes between iterations.
Complete replacement is idempotent and order-independent.

**Spec deserialization:** Rule action spec blocks (`addNode:` and
`updateNode:` `spec:` values) are deserialized using a dedicated
`ObjectMapper` with scalar coercion explicitly enabled (`CoercionConfig`
for String-to-number and String-to-boolean). After `${var.*}` interpolation,
all substituted values are strings; the dedicated mapper ensures `"10"` →
`int 10` and `"true"` → `boolean true` regardless of the application's
Jackson configuration. YAML-native scalars (`threshold: 5`) pass through
as their native types — only `${}`-interpolated values require coercion.

**Semantics:**

- Rules participate in the same `GraphRuleEngine` fixed-point loop as annotation
  rules. Declarative and reflective rules are evaluated together — mixed YAML and
  Java rules on the same graph work correctly.
- `graph:` scoping uses `GraphPatternMatcher` — same include/exclude semantics as
  `@GraphRule(graph={...})`. When `graph:` is omitted, the rule applies to the
  enclosing YAML graph only (same default as invariants — see §6.2).
- Conflict detection, cycle detection, convergence checking, and mutation ordering
  all apply unchanged.
- The `addNode` action's `type:` is validated against the `NodeSpecRegistry` at
  build time.

**Architecture changes:**

- New `DeclarativeRuleDescriptor` record: `name`, `graphPatterns`,
  `List<PatternParameterDescriptor>`, `List<ActionDescriptor>`.
- New `ActionDescriptor` sealed interface: `AddNode`, `RemoveNode`, `UpdateNode`,
  `AddDependency`, `RemoveDependency` — each carrying template strings for
  interpolation.
- New `DeclarativeRuleAdapter` implementing `ResolvedRule` interface. Given a
  set of pattern bindings, resolves `${match.*}` references in action templates
  and produces `List<GraphMutation>`.
- `GraphRuleEngine`: refactor `ResolvedGraphRule` from record to sealed interface
  with `ReflectiveRule` (existing — wraps `Method` + instance) and
  `DeclarativeRule` (new — wraps patterns + action descriptors).
  - Extract `getParameterNames()` from `Method.getParameters()` into a method on
    the interface — declarative rules carry their own binding names from YAML keys.
  - Extract the "invoke action" step from `invokeRule()` — reflective calls
    `Method.invoke()`, declarative evaluates action templates.
  - `PatternMatchingSupport` is unchanged — it already works with
    `PatternParameterDescriptor` and `DesiredStateGraph` without reflection.
  - Estimated surgery: ~80 lines in `GraphRuleEngine`, parallel ~80 lines in
    `GraphInvariantEngine`.

**Cross-surface rule composition (adversarial fix):**

A standalone `@GraphRule(graph = {"*:*"})` Java class must fire against YAML
graphs too. The current annotation processor only resolves standalone rules
against annotation graphs. Fix: a new build step
(`CrossSurfaceRuleResolutionStep`) runs after both processors, consuming all
`DesiredStateGraphBuildItem`s and all standalone `GraphRuleDescriptor`s /
`GraphInvariantDescriptor`s. It produces `AdditionalRulesBuildItem` entries
consumed by the YAML recorder when wrapping its `GoalCompiler`.

**Rule ordering:** When YAML declarative rules and cross-surface annotation
rules are combined, the ordering is: YAML declarative rules first, then
cross-surface annotation rules — matching the annotation processor's
convention where graph-local rules precede standalone rules
(`DesiredStateAnnotationsProcessor` line ~120). The `GraphRuleEngine`
fixed-point loop ensures that the final converged graph is the same
regardless of evaluation order for well-formed rules (monotonic, no mutual
conflicts). Ordering affects convergence speed, not correctness. If rules
truly conflict, `ConflictingMutationException` is thrown deterministically.

---

#### 6.5 Lifecycle Phases

**Complexity:** Medium
**Differentiator:** Lifecycle phases

Maps to `CompilationResult.Lifecycle(List<Phase>)` — the runtime already
supports this via `LifecycleManager`.

```yaml
lifecycle:
  phases:
    - id: infrastructure
      completionCondition: allPresent
      nodes:
        database:
          type: db
          spec:
            engine: postgres
            version: "15"
        cache:
          type: cache
          spec:
            engine: redis

    - id: application
      completionCondition: allPresent
      nodes:
        api-server:
          type: app
          dependsOn: [database]
          spec:
            image: "api:latest"

    - id: observability
      completionCondition: never
      nodes:
        monitor:
          type: monitor
          dependsOn: [api-server]
          spec:
            target: api-server
```

In this example, `database` and `cache` are declared in `infrastructure`.
The `application` phase references `database` in `dependsOn` without
re-declaring it — `database` is implicitly carried forward. Similarly,
`observability` references `api-server` without re-declaration.

**Cross-phase node references — implicit carry-forward:**

Each phase produces a **separate** `DesiredStateGraph`. The runtime reconciles
one phase at a time. Nodes from earlier phases are **implicitly carried forward**
into later phases — they are already present in actual state (the earlier phase
reconciled them) and later phases can reference them in `dependsOn` without
re-declaration.

At compile time, the build step injects carried-forward nodes from all earlier
phases into each later phase's `DesiredStateGraph`. The planner in the later
phase sees these as already PRESENT and skips them — no redundant reconciliation.
The only reason they appear in the later phase's graph is so that `dependsOn`
references resolve correctly.

**Explicit re-declaration for overrides:** An operator can re-declare a node
from an earlier phase in a later phase to override its spec. The re-declared
node's type must match; the spec is the override value. This is for cases where
a node's configuration evolves between phases (e.g., a database that gets
different connection pool settings in the application phase).

**Cross-phase forEach:** forEach-generated nodes from earlier phases are
carried forward as their expanded concrete nodes. A later phase that needs
to depend on forEach-generated nodes from an earlier phase references the
template ID in `dependsOn` — alignment rules (§6.6) apply across phases.

**Completion condition vocabulary:**

| YAML value | Maps to | Semantics |
|-----------|---------|-----------|
| `allPresent` | `CompletionCondition.allPresent()` | All nodes in the phase are PRESENT in actual state |
| `never` | `CompletionCondition.never()` | Phase never completes — steady-state reconciliation |
| `{ bean: "beanName" }` | CDI lookup | Named `CompletionCondition` bean — Java escape hatch |

CDI bean lookup for `completionCondition: { bean: "name" }` must happen at
Quarkus **build time**. The `YamlDesiredStateProcessor` validates that the
named bean exists in the Jandex index and implements `CompletionCondition`.
Build-time failure is the only acceptable mode — a typo in a bean name must
fail the build, not produce a runtime `NoSuchBeanException` or a hung
lifecycle during reconciliation.

The last phase should use `never` for steady-state operations or `allPresent`
for finite lifecycles. Build-time warning if the last phase uses `allPresent`
without an explicit acknowledgement (the lifecycle will terminate and
`LifecycleManager` will stop reconciliation for that tenant).

**Rules and invariants scope (adversarial fix):**

Rules and invariants fire **per-phase**. Each phase's graph is compiled
independently through the rule engine and invariant engine. Cross-phase
structural assertions (e.g., "every app must have a monitor" spanning phases)
are not supported declaratively — use a lifecycle-level Java `CompletionCondition`
or enforce by convention.

This is documented as a known limitation. Per-phase evaluation matches the
runtime contract: each phase is reconciled independently.

**When `lifecycle:` is present:** The top-level `nodes:` key is forbidden.
Nodes live inside phases. The YAML produces `CompilationResult.Lifecycle`
instead of `CompilationResult.single()`. All other features (`when:`, `forEach:`,
rules, invariants, imports) work within each phase independently — each
phase's graph is compiled and evaluated through the rule/invariant engines
separately.

**Imports in lifecycle phases:** Module `imports:` can appear at two levels:
- **Top-level `imports:`** (alongside `lifecycle:`) — the module's nodes are
  included in **every phase**. This is for shared infrastructure modules
  (e.g., a monitoring module that should exist in all phases).
- **Per-phase `imports:`** (inside a phase's block) — the module's nodes
  are included only in that phase.

Top-level imports are expanded first, then per-phase imports. If the same
module is imported at both levels (different `as:` aliases), they produce
independent instances. Same `as:` alias at both levels is a build-time error.

Fault policies are `@ApplicationScoped` CDI beans — they are NOT scoped per-phase.
However, fault policy isolation works in practice because each phase reconciles
independently: Phase 1's reconciliation cycle produces fault events only for
Phase 1's nodes, and Phase 2's cycle produces events only for Phase 2's nodes.
The `nodeTypes`/`faultTypes` filtering in `ThresholdFaultPolicy` provides
content-based scoping. Carried-forward nodes that appear in multiple phases may
trigger the same fault policy in each phase's reconciliation cycle — this is
correct behavior (the carried-forward node is part of that phase's desired graph).

---

### Phase 3

#### 6.6 Cardinality Stamping (`forEach:`)

**Complexity:** High
**Value:** Table-stakes

Stamps N copies of a node from a collection.

```yaml
nodes:
  regional-source:
    type: data-source
    forEach:
      as: region
      in: ["us-east", "eu-west", "ap-south"]
    spec:
      name: "customers-${each.region}"
      uri: "s3://data/${each.region}/customers.csv"
```

**Expansion** produces three concrete nodes:
- `regional-source.us-east` → `{name: "customers-us-east", ...}`
- `regional-source.eu-west` → `{name: "customers-eu-west", ...}`
- `regional-source.ap-south` → `{name: "customers-ap-south", ...}`

**ID derivation:** `${templateId}.${value}` using the reserved `.` separator.

**Expansion limit:** forEach expansion is capped at a configurable maximum
per template (default: 1,000). If the `in` array resolves to more entries
than the limit, `GoalCompiler.compile()` fails with:

> "forEach template 'regional-source' would expand to 5,000 nodes (limit:
> 1,000). Configure `casehub.desiredstate.foreach.max-expansion` to raise
> the limit, or reduce the input array."

The cap prevents OOM during graph compilation and protects against
`PatternMatchingSupport.crossProduct()` combinatorial explosion in
downstream rule evaluation. The limit is configurable via MicroProfile
Config: `casehub.desiredstate.foreach.max-expansion=2000`.

**Named iteration groups (aligned iteration):**

When multiple forEach nodes must iterate together (aligned per-value), they
reference a named iteration group instead of duplicating `as`/`in` inline.
This makes alignment structural — both nodes reference the same group, so
values cannot diverge and renaming is atomic.

```yaml
iterations:
  regional:
    as: region
    in: ["us-east", "eu-west"]

nodes:
  regional-source:
    type: data-source
    forEach: regional
    spec: { uri: "s3://${each.region}/data.csv" }

  regional-ingest:
    type: ingestion
    forEach: regional
    dependsOn: [regional-source]
    spec: { region: "${each.region}" }
```

→ `regional-ingest.us-east` depends on `regional-source.us-east` (aligned).

**Inline forEach** remains supported for standalone iteration (no alignment):

```yaml
nodes:
  worker:
    type: processor
    forEach: { as: idx, in: ["1", "2", "3"] }
    spec: { instance: "${each.idx}" }
```

Inline forEach creates an anonymous iteration group — two nodes with inline
forEach are never aligned, even if they happen to share the same `as`/`in`.
Alignment requires a named group reference. This removes the fragile naming
convention (same `as` + identical `in`) in favor of explicit structural linkage.

If a forEach node depends on a non-forEach node, each copy depends on the
same fixed node. If a **non-forEach node depends on a forEach node** →
build-time error: the template ID no longer exists after expansion (only
stamped copies exist), so the dependency cannot resolve. This is analogous
to the unconditional→conditional error (§6.3). If the intent is fan-in
(depend on all copies), use a forEach node with the same iteration group.
If two forEach nodes reference **different** named groups and depend on
each other → build-time error. If an inline forEach node depends on a
named-group forEach node (or vice versa) → build-time error with guidance
to use the same named group.

**Variable-sourced values:**

```yaml
variables:
  regions: '["us-east", "eu-west"]'

nodes:
  regional-source:
    type: data-source
    forEach: { as: region, in: "${var.regions}" }
```

Resolves to a JSON array string, parsed into a list. Allows
environment-specific configuration via MicroProfile Config.

**JSON parse error handling:** If the resolved value is not valid JSON or
is not a JSON array of strings, `GoalCompiler.compile()` fails with a
contextual error:

> "forEach template 'regional-source': variable '${var.regions}' resolved
> to '["us-east", "eu-west"' which is not a valid JSON array. Expected a
> JSON array of strings like `["a", "b"]`."

Specific validations:
- Non-JSON values (e.g., `us-east,eu-west`) → error with JSON format guidance
- Empty string `""` → error: "Use `[]` for an empty array, not an empty string"
- JSON array of non-strings (e.g., `[1, 2, 3]`) → error: "forEach values
  must be strings"
- Valid JSON non-array (e.g., `{"key": "value"}`) → error: "Expected a JSON
  array, got object"

**Edge cases (adversarial fixes):**

- **Zero values:** If forEach `in` resolves to an empty array and any
  unconditional node depends on the template → compile-time error. An empty
  forEach with no dependents produces zero nodes silently (valid).
- **ID collisions:** The `.` separator cannot appear in user-declared node IDs
  or forEach values. Validated at build time. This prevents the ambiguity that
  hyphen separators would create.
- **forEach + when:** Each stamped copy evaluates its own `when:` condition
  independently. The `when:` value can reference `${each.region}` but NOT
  nested interpolation (`${var.enable_${each.region}}` is invalid). To
  conditionally include some iterations, pre-filter the `in` list.
- **Shrinkage:** When a forEach value is removed between deployments (config
  change), the corresponding stamped nodes vanish from the desired graph.
  The reconciliation loop deprovisions them. This is documented behavior —
  consistent with Terraform `for_each` removal semantics.
- **Interaction with rules:** Rules see the fully expanded graph (post-forEach).
  A rule matching `type: data-source` fires once per stamped copy. Rules
  cannot produce forEach-expandable nodes — they operate on the expanded graph.
  Documented limitation.

---

#### 6.7 Composable Modules

**Complexity:** Very high
**Value:** Table-stakes (composition)

Reusable, parameterized YAML graph fragments. String parameters only — no
typed `nodeRef` parameter in this phase (avoids building a type system in YAML).

**Module definition** (`META-INF/desiredstate/modules/monitoring.yaml`):

```yaml
module:
  name: monitoring
  parameters:
    watched_node_id:
      type: string
      required: true
    alert_email:
      type: string
      default: "ops@example.com"

nodes:
  monitor:
    type: monitor
    dependsOn: ["${var.watched_node_id}"]
    spec:
      target: "${var.watched_node_id}"
  alerter:
    type: alerter
    dependsOn: [monitor]
    spec:
      email: "${var.alert_email}"
```

**Module import:**

```yaml
imports:
  - module: monitoring
    as: pipe-monitor
    when: "${var.monitoring_enabled}"
    parameters:
      watched_node_id: warehouse-sink
      alert_email: "pipeline-ops@example.com"
```

**Scoping:** Module node IDs are prefixed with the `as:` alias using `.`:
- `pipe-monitor.monitor` depends on `warehouse-sink`
- `pipe-monitor.alerter` depends on `pipe-monitor.monitor`

**Parameter resolution order:** Parameter values specified in the `parameters:`
block of an import are resolved in the **importing context** BEFORE the module
scope is pushed. This means `${var.target}` in a parameter value resolves
against the importing graph's variables, not the module's parameter scope.
This prevents circular shadowing: a module parameter named `target` does not
shadow the importing variable `target` during its own value resolution.

The sequence:
1. Resolve parameter values in the importing scope (variables + Config)
2. Push module scope: parameters shadow variables within the module
3. Resolve `${var.*}` inside the module against the parameter scope first,
   then fall through to importing variables, then Config

**Cross-boundary dependencies:** Module parameters carry string values. A
parameter value that matches a node ID in the importing graph creates a
dependency naturally through `dependsOn: ["${var.watched_node_id}"]`. The
importer references module nodes with the alias prefix:
`dependsOn: [pipe-monitor.monitor]`.

No `nodeRef` type — all parameters are strings. This keeps the module system
simple. If the parameter value doesn't match any node ID, the dependency
validation catches it at build time (dangling reference). The error message
is clear: the parameter value is treated as a literal node ID.

**Module nesting:** Modules can import other modules. Nesting capped at 2
levels — a module can import modules, but those inner modules cannot import
further. Build-time validation enforces this and detects import cycles.

**Diamond imports:** Each import path produces distinct instances (different
`as:` prefixes → different node IDs). Module D imported via both B and C
produces `B.D.monitor` and `C.D.monitor` — two independent instances. If a
shared instance is needed, import D once at the top level and pass its node
IDs to B and C as string parameters.

**Conditional imports:** `when:` on an import excludes all module nodes when
the condition is false (same semantics as node-level `when:`).

**Module-scoped rules and invariants:**

Modules can carry their own `rules:` and `invariants:` sections:

```yaml
module:
  name: monitoring
  parameters:
    watched_node_id:
      type: string
      required: true

nodes:
  monitor:
    type: monitor
    dependsOn: ["${var.watched_node_id}"]
    spec:
      target: "${var.watched_node_id}"

invariants:
  monitor-must-have-dependency:
    match:
      mon: { type: monitor }
    directDep:
      target: { type: "*", of: mon, direction: dependencies }
```

Module rules and invariants are promoted to the top-level rule/invariant lists
after module expansion and alias prefixing. They fire against the **full graph**,
not just the module's own nodes. Pattern-based matching provides natural scoping:
a rule matching `type: monitor` only fires against monitor nodes, which are
typically the module's own nodes. If a module rule needs to match broader node
types, it has full graph visibility — this is intentional, as structural rules
like "every node this module monitors must have a health check" require
cross-boundary access.

Module rules participate in the same fixed-point loop and invariant validation
as top-level rules.

**Module discovery:** Classpath scan at `META-INF/desiredstate/modules/`.
Modules can come from library JARs — a monitoring module published as a Maven
artifact is usable in any project.

**Module + forEach:** A module can be imported multiple times with different
`as:` aliases and parameters. forEach inside a module's nodes is supported.
forEach on an import itself is not supported in Phase 3 — use multiple
explicit imports.

---

## 7. Evaluation Pipeline

```
BUILD TIME                                COMPILE TIME (GoalCompiler.compile())
────────────                              ────────────────────────────────────
1. Discover YAML files + modules          7.  Resolve ${var.*} via VariableResolver
2. Parse YAML model                       7b. Validate resolved dependency references
3. Resolve imports (expand modules,            (post-variable-resolution — catches
   apply alias prefixes, validate              templated refs that were deferred at
   nesting depth, detect import cycles)        step 4)
                                          8.  Expand forEach (stamp N copies per template)
                                          9.  Evaluate when: (include/exclude nodes)
                                         10.  Remove orphaned optional dependencies
                                         11.  Build DesiredNodes + Dependencies
4. Validate structure:                   12. GraphRuleEngine (YAML + annotation rules,
   - node types in registry                  fixed-point loop)
   - static dependency references valid  13. GraphInvariantEngine (YAML + annotation
     (literal node IDs only — templated       invariants, single pass)
     refs like ${var.*} are deferred)    14. Return CompilationResult
   - conditional dependency safety            (single or lifecycle)
   - forEach template consistency
   - rule action types in registry
   - cross-surface namespace:name
     duplicate check
5. Build descriptors (GraphDescriptor
   + YAML expansion metadata)
6. Register GoalCompiler + FaultPolicy
   synthetic beans
```

Rules and invariants see the fully expanded, post-forEach/when graph. They
never deal with templates or conditions.

For lifecycle phases, steps 7–14 run **sequentially in declaration order** —
each phase's compilation output (including rule-generated nodes from step 12)
feeds the next phase's carry-forward injection. "Independently" means each
phase has its own expansion/rule/invariant pipeline — not that phases can be
compiled in parallel. Phase 2 cannot begin compilation until Phase 1's fully
expanded graph (post-rules, post-invariants) is available, because carry-forward
injects Phase 1's output nodes into Phase 2's input graph.

## 8. Architecture Changes

### 8.1 YAML Model Extensions

| New type | Fields | Purpose |
|----------|--------|---------|
| `YamlFaultPolicy` | faultTypes, nodeTypes, ignoreTypes, namespace, tiers | Fault policy declaration |
| `YamlFaultTier` | threshold, reviewNode (type, spec, humanGating) | Escalation tier |
| `YamlRule` | name, graph, patterns, actions | Declarative graph rule |
| `YamlInvariant` | name, graph, patterns, message (optional) | Declarative invariant |
| `YamlPattern` | type, of, direction, kind (match/directDep/reaches/notExists) | Pattern entry |
| `YamlAction` | kind (addNode/removeNode/etc), parameters | Rule action |
| `YamlLifecycle` | phases (list) | Lifecycle structure |
| `YamlPhase` | id, completionCondition, nodes | Lifecycle phase |
| `YamlForEach` | as, in | forEach metadata on nodes (inline) |
| `YamlIterationGroup` | name, as, in | Named iteration group (top-level `iterations:` key) |
| `YamlImport` | module, as, when, parameters | Module import |

`YamlGraph` gains: `faultPolicy`, `rules`, `invariants`, `lifecycle`, `imports`,
`iterations`.
`YamlNode` gains: `when`, `forEach` (string reference to named group, or inline
`YamlForEach` map).

### 8.2 GraphDescriptor — Surface-Agnostic Descriptors

`GraphDescriptor` evolves from annotation-centric to surface-agnostic IR.
The descriptor types for rules and invariants become sealed interfaces
supporting both reflective (annotation) and declarative (YAML/TS) variants:

```
GraphRuleDescriptor (record) → GraphRuleDescriptor (sealed interface)
  ├── ReflectiveRuleDescriptor(methodName, imperative, patterns,
  │                            sourceClassName)
  └── DeclarativeRuleDescriptor(name, graphPatterns, patterns, actions)

GraphInvariantDescriptor (record) → GraphInvariantDescriptor (sealed interface)
  ├── ReflectiveInvariantDescriptor(methodName, imperative, patterns,
  │                                 sourceClassName)
  └── DeclarativeInvariantDescriptor(name, graphPatterns, patterns)
```

Both variants carry `String[] graphPatterns` for graph scoping. For
annotation rules defined within a `@DesiredState` class, `graphPatterns`
is empty (meaning "this graph only" — scoped by the build step that
populates `GraphDescriptor`). For standalone `@GraphRule` classes and
YAML declarative rules, `graphPatterns` carries the include/exclude
patterns. This eliminates the asymmetry where annotation rules carried
graph scoping externally (`List<Entry<String[], List<GraphRuleDescriptor>>>`)
while declarative rules carried it internally.

`GraphDescriptor.graphRules()` and `graphInvariants()` now carry both
reflective and declarative variants via the sealed interface. Both surfaces
populate the same IR. The annotation processor populates
`ReflectiveRuleDescriptor`; the YAML processor populates
`DeclarativeRuleDescriptor`. Downstream consumers (`DesiredStateGraphRecorder`,
`YamlGraphRecorder`) work with the sealed interface.

`FaultPolicyDescriptor` and `TierDescriptor` are annotation-specific —
`TierDescriptor.reviewMethodName` names a Java method. The YAML path
builds `ThresholdFaultPolicy` beans directly from `YamlFaultPolicy` via
the builder API (§6.1), bypassing `FaultPolicyDescriptor` entirely. Both
paths converge at `ThresholdFaultPolicy` — the runtime type is shared,
only the construction differs.

**ExpansionContext** (narrowed from `YamlExpansionContext`):

Expansion-time metadata that doesn't belong in `GraphDescriptor`:
- `Map<String, ForEachDescriptor>` — forEach metadata keyed by node ID
- `Map<String, String>` — when conditions keyed by node ID
- `List<ModuleImportDescriptor>` — module imports

These are compile-time expansion directives resolved during
`GoalCompiler.compile()` — they don't exist at runtime. Named
`ExpansionContext` (not `YamlExpansionContext`) because the TS DSL
may also support `forEach` and `when:`. Rules, invariants, and fault
policies live in `GraphDescriptor` — not in the expansion context.

### 8.3 Engine Refactoring

**Three-variant sealed hierarchy:**

The `boolean imperative` field on `ResolvedGraphRule` conflates two
structurally different evaluation strategies (imperative: whole graph →
mutations; parameterized: pattern bindings → method invocation). The
declarative variant adds a third. Making the dispatch explicit in the
type system:

```
ResolvedRule (sealed interface)
  ├── ImperativeRule(String name, Method method, Object instance)
  ├── ParameterizedReflectiveRule(String name, Method method, Object instance,
  │                               List<PatternParameterDescriptor> patterns)
  └── DeclarativeRule(String name, List<PatternParameterDescriptor> patterns,
                      List<ActionDescriptor> actions)
```

`ImperativeRule` takes the whole `DesiredStateGraph` and returns
`List<GraphMutation>`. `ParameterizedReflectiveRule` does pattern matching
and calls a Java method with bindings. `DeclarativeRule` does pattern
matching and evaluates action templates.

Same three-variant structure for invariants:
```
ResolvedInvariant (sealed interface)
  ├── ImperativeInvariant(String name, Method method, Object instance)
  ├── ParameterizedReflectiveInvariant(String name, Method method, Object instance,
  │                                    List<PatternParameterDescriptor> patterns)
  └── DeclarativeInvariant(String name, List<PatternParameterDescriptor> patterns)
```

**PatternEvaluator extraction (#114):**

`GraphRuleEngine.expandChain` (~30 lines) and `GraphInvariantEngine.expandChain`
(~35 lines) are structurally identical — same cross-product expansion, same
recursive binding resolution, same pattern dispatch (DIRECT_DEP, REACHES,
NOT_EXISTS). The only difference is the terminal action.

Extract a shared `PatternEvaluator` that takes:
- `DesiredStateGraph` — the graph to match against
- `List<PatternParameterDescriptor>` — the pattern chain
- `String[]` binding names — from `getParameterNames()` on the sealed interface

And returns `List<Map<String, DesiredNode>>` — all valid binding maps. The
terminal action (rule: produce mutations; invariant: validate) is the caller's
responsibility. This eliminates the ~80-line parallel surgery and unifies
pattern evaluation into one well-tested component.

This aligns with issue #114 (shared pattern matching infrastructure) — the
`PatternEvaluator` is the next step after the existing `PatternMatchingSupport`
utility methods.

**Binding name extraction:** `getParameterNames()` moves from
`PatternMatchingSupport.getParameterNames(Method)` to a method on the sealed
interface — `ImperativeRule` returns empty (no bindings),
`ParameterizedReflectiveRule` derives names from `Method.getParameters()`,
`DeclarativeRule` carries names from YAML keys.

**AddNode construction pipeline:**

`DeclarativeRule`'s action evaluation requires runtime access to
`NodeSpecRegistry` and `ObjectMapper` for constructing `DesiredNode` from
raw YAML spec maps. These dependencies are captured in the
`DeclarativeRuleAdapter`'s closure — same pattern as the fault policy
template factory (§8.5):

1. `YamlGraphRecorder` creates `DeclarativeRuleAdapter` instances, capturing
   `NodeSpecRegistry` and a coercion-enabled `ObjectMapper` in the closure
2. For `addNode:` actions: resolve `${match.*}` templates → look up spec
   class from `NodeSpecRegistry` using the `type:` field → `ObjectMapper
   .convertValue()` into the typed `NodeSpec` → construct `DesiredNode`
3. For `updateNode:` actions: same pipeline, produces
   `GraphMutation.UpdateNode`
4. For `removeNode:`/`addDependency:`/`removeDependency:`: string
   interpolation only — no spec construction needed

The `DeclarativeRuleAdapter` lives in the YAML recorder layer (not in the
domain-agnostic `GraphRuleEngine`), preserving the engine's independence
from YAML-specific services.

### 8.4 Cross-Surface Rule Resolution

New build step: `CrossSurfaceRuleResolutionStep`

Runs after both `DesiredStateAnnotationsProcessor` and
`YamlDesiredStateProcessor`. Consumes:
- All `DesiredStateGraphBuildItem`s (from both surfaces)
- All standalone `@GraphRule` and `@GraphInvariant` descriptors

Produces: `AdditionalRulesBuildItem` entries — standalone rules matched to
YAML graphs via `GraphPatternMatcher`. The YAML recorder consumes these
alongside its own declarative rules.

### 8.5 FaultPolicy Template Factory

The YAML recorder creates `ThresholdFaultPolicy` instances where each tier's
`TypedFaultPolicy` uses a template-based `ReviewSpecFactory`. The factory
lambda captures:
- `NodeSpecRegistry` — to resolve the review node's spec class from `type:`
- `ObjectMapper` — to deserialize spec values via `convertValue`
- Template map — the `spec:` block with `${fault.*}` placeholders

At fault time: resolve `${fault.*}` against the `FaultEvent`, deserialize
into the `NodeSpec` subclass, return. The `nodeType()` is provided directly
from the tier's `type:` field — no probe needed.

## 9. Cross-Cutting Concerns

### 9.1 Dry-Run / Preview

Every mature IaC tool has a preview mode (Terraform `plan`, Pulumi `preview`).
An operator writing YAML with forEach, when:, rules, and invariants needs to
see the expanded graph before it hits the reconciliation loop.

Proposed: a Quarkus dev-mode endpoint and/or Maven goal that renders the
fully expanded graph (post-module, post-forEach, post-when, post-rules,
post-invariants) as YAML or JSON. Implementation will be tracked as a
dedicated GitHub issue once the features in this spec are built — dry-run
depends on the expansion pipeline being complete.

### 9.2 Error Messages

YAML-originated errors must reference the source YAML file and line number,
not generated class names or descriptor indices. The `source` field in
`DesiredStateGraphBuildItem` already tracks provenance (`yaml:<fileName>`).
Extend this to carry line numbers for individual nodes, rules, and invariants.

### 9.3 Testing

Each feature needs:
- Unit tests for YAML parsing (model types)
- Unit tests for build-time validation (deployment processor)
- Unit tests for runtime expansion (GoalCompiler behavior)
- Integration tests via the pipeline-yaml example (extended with new features)

The pipeline-yaml example grows incrementally — each phase adds rules,
invariants, fault policies, etc. to the existing medallion pipeline.

### 9.4 Documentation

- Consumer guide: YAML syntax reference for each feature
- Contributor guide: architecture of the YAML deployment processor
- Pipeline-yaml example: annotated companion showing all features

## 10. Decisions

| # | Decision | Rationale | Alternative | Why not |
|---|----------|-----------|-------------|---------|
| D1 | Explicit interpolation namespaces (`${var.}`, `${match.}`, etc.) | Prevents collision between variables, bindings, and fault context | Bare names with implicit resolution order | Ambiguous — `${sink}` could be variable or binding |
| D2 | `.` as sole generated-ID separator | Single convention, unambiguous, can't appear in user IDs | Hyphen (natural) | Ambiguous — hyphens appear in IDs and values |
| D3 | No spec field access in rule interpolation | Defends declarative boundary; spec-aware rules use Java | `${match.sink.spec.field}` | Opens path to property-traversal language (Helm trap) |
| D4 | Build-time error for unconditional→conditional deps | Silent removal causes runtime failures | Warning only | Operators deploy despite warnings |
| D5 | forEach zero-values is error when dependents exist | Prevents dangling references | Allow empty expansion | Silent graph corruption |
| D6 | Implicit carry-forward across lifecycle phases | Avoids DRY violation — nodes reconciled in earlier phases are injected into later graphs so `dependsOn` resolves; re-declaration only for spec overrides | Explicit re-declaration | Redundant duplication, error-prone for large graphs |
| D7 | Completion vocabulary: `allPresent`, `never`, `{ bean: "name" }` | Minimal built-in set with Java escape hatch | Extensible predicate DSL | Predicate language in YAML is unbounded complexity |
| D8 | Rules/invariants fire per-phase | Matches runtime reconciliation scope | Cross-phase invariants | Requires merged graph concept that doesn't exist |
| D9 | String-only module parameters (no nodeRef) | Avoids type system in YAML | Typed `nodeRef` parameter | Typed cross-boundary resolution is the Helm trap crack |
| D10 | Module nesting capped at 2 levels | Bounds debugging complexity | Unlimited nesting | Crossplane experience shows deep nesting is a pain point |
| D11 | GraphDescriptor becomes surface-agnostic via sealed descriptor interfaces | Rules and invariants use sealed hierarchies (Reflective/Declarative variants) so both surfaces populate the same IR | Keep YAML data in separate context | Dual-path architecture — consumers must handle two shapes for the same concept |
| D12 | Phase: differentiators first (1: fault+invariant+when, 2: rules+lifecycle, 3: forEach+modules) | Ship value before catch-up | All at once | 7 features × pairwise interactions = unmanageable risk |
| D13 | Named iteration groups for aligned forEach | Structural linkage — two nodes referencing the same group are guaranteed to iterate over the same values; inline forEach for standalone cases | Naming convention (same `as` + identical `in`) | Fragile — values can diverge silently, renaming is non-atomic |
| D14 | Module-scoped rules and invariants fire against the full graph | Pattern-based matching provides natural scoping; cross-boundary rules (e.g., "every monitored node needs health check") require full visibility | Module-local scope only | Prevents useful structural assertions that span module boundaries |
| D15 | Custom pattern matching engine (not Drools) for YAML rules | Research doc (§10) recommends Drools; this spec extends the existing `GraphRuleEngine` instead — requires explicit human decision (see §11 references) | Drools backend per #119 | Deferred — decision escalated to human |
| D16 | Three-variant sealed hierarchy for resolved rules/invariants | Makes dispatch explicit in the type system — `ImperativeRule`, `ParameterizedReflectiveRule`, `DeclarativeRule` instead of boolean flag | Boolean `imperative` field | Conflates two structurally different evaluation strategies |

## 11. Deferred Items

**Lifecycle hooks (#121):** Issue #121 ("YAML lifecycle hooks — imperative steps
within transitions") is explicitly deferred from this spec. Lifecycle hooks
require imperative step execution within phase transitions, which is structurally
different from the declarative features specified here. The current spec provides
the lifecycle phase structure (§6.5) and completion condition vocabulary needed
as a foundation. Issue #121 remains open for a separate design spec.

**Drools backend (#119):** The research document (§10) recommends Drools as the
rule engine backend for YAML and operator-facing rules. Issue #119 is titled
"YAML rules and invariants — Drools backend." This spec instead extends the
existing custom `GraphRuleEngine`/`GraphInvariantEngine` with sealed interface
hierarchies and a `PatternEvaluator` extraction. The Drools decision is captured
in D15 and escalated to the human for explicit confirmation or reversal. The
spec's design is compatible with either direction — if Drools is chosen, the
`DeclarativeRule`/`DeclarativeInvariant` variants become Drools rule adapters
instead of template evaluators. The sealed interface hierarchy works regardless.

## 12. References

- `api/.../ThresholdFaultPolicy.java` — fault policy builder API
- `annotations/runtime/.../GraphRuleEngine.java` — fixed-point rule evaluation
- `annotations/runtime/.../GraphInvariantEngine.java` — invariant validation
- `annotations/runtime/.../PatternMatchingSupport.java` — shared pattern primitives
- `annotations/runtime/.../GraphDescriptor.java` — IR record
- `annotations/runtime/.../GraphPatternMatcher.java` — include/exclude matching
- `annotations/deployment/.../DesiredStateAnnotationsProcessor.java` — annotation build pipeline
- `annotations/deployment/.../DesiredStateGraphBuildItem.java` — cross-surface validation
- `yaml/runtime/.../YamlGraphRecorder.java` — YAML GoalCompiler factory
- `yaml/deployment/.../YamlDesiredStateProcessor.java` — YAML build pipeline
- `examples/pipeline-annotated/.../MedallionPipeline.java` — annotation reference example
- `examples/pipeline-yaml/.../medallion-pipeline.yaml` — YAML reference example
- `runtime/.../LifecycleManager.java` — phase orchestration
- `docs/research/2026-08-27-operator-declaration-language-research.md` — research document (Drools recommendation at §10)
- #116 — operator-first declaration language vision
- #117 — YAML surface foundation (closed)
- #114 — shared pattern matching infrastructure (open)
- #108 — referenced by ARC42STORIES C10 (related epic scope — not directly
  addressed by this spec; requirements to be triaged separately)
- #109 — referenced by ARC42STORIES C10 (related epic scope — not directly
  addressed by this spec; requirements to be triaged separately)
- #119 — YAML rules and invariants — Drools backend (open, decision escalated)
- #121 — YAML lifecycle hooks — imperative steps within transitions (open, deferred)
