# Decisions — YAML Language Extensions (#116)

## D1: Explicit interpolation namespaces

**Choice:** All `${}` references use explicit prefixes: `${var.}`, `${match.}`, `${fault.}`, `${each.}`
**Alternatives:**
- Bare names with implicit resolution order — simpler syntax but ambiguous (`${sink}` = variable or binding?)
**Rationale:** A variable named `sink` and a pattern binding named `sink` are indistinguishable without prefixes. Namespace collision was identified as fatal in adversarial review.
**Trade-offs:** More verbose syntax. Operators must learn the prefix convention.
**Sources:** Adversarial review (semantics agent)
**Exploration:** quick
**Status:** captured

## D2: `.` as generated-ID separator

**Choice:** Single `.` separator for all system-generated node IDs (module scoping and forEach stamping)
**Alternatives:**
- Hyphen `-` — natural but ambiguous (hyphens appear in both node IDs and forEach values; `source-us-east` could be template `source-us` + value `east` or template `source` + value `us-east`)
- `~` or `::` — unambiguous but less readable
**Rationale:** `.` is unambiguous (user-declared IDs cannot contain it), visually clean, and a single convention covers both modules and forEach.
**Trade-offs:** User-declared node IDs and forEach values cannot contain `.` — validated at build time.
**Sources:** Adversarial review (edge cases agent)
**Exploration:** quick
**Status:** captured

## D3: No spec field access in rule interpolation

**Choice:** Rule action interpolation supports `${match.binding.id}`, `${match.binding.type}`, and `${match.binding.flatId}` only. No `${match.binding.spec.field}`. `.flatId` (added via cross-cutting review R1-05) replaces `.` with `-` for safe rule-generated IDs.
**Alternatives:**
- Closed set of spec accessors (`.spec.<field>`) — more powerful but opens path to property traversal
- Full spec access with arbitrary nesting — Helm trap
**Rationale:** YAML rules are structural (add/remove nodes and edges based on graph topology). Rules that propagate configuration between node specs are domain-logic — they belong in Java `@GraphRule` where the full spec type system is available.
**Trade-offs:** Common patterns like "copy the region field from matched node to new node" require Java. This is the intended boundary.
**Sources:** Adversarial review (semantics agent, prior art agent — Helm trap analysis)
**Exploration:** quick
**Status:** captured

## D4: Build-time error for unconditional→conditional dependencies

**Choice:** An unconditional node depending on a conditional node is a build-time error, not a warning.
**Alternatives:**
- Warning only — operators deploy despite warnings, causing runtime failures
- Silent dependency removal — worst option, already rejected
**Rationale:** Silent removal of a dependency when a `when:` condition evaluates to false causes runtime crashes (e.g., app deployed without its database). Errors are harder to ignore than warnings.
**Trade-offs:** Operators must add `when:` to all transitive dependents, or use `optional: true` dependency syntax. More verbose but explicitly safe.
**Sources:** Adversarial review (edge cases agent)
**Exploration:** quick
**Status:** captured

## D5: forEach zero-values error when dependents exist

**Choice:** forEach expanding to zero nodes is a compile-time error if any unconditional node depends on the template.
**Alternatives:**
- Allow empty expansion silently — causes dangling dependency references
- Always error on empty — blocks legitimate "no regions configured" scenarios
**Rationale:** Zero-expansion with dependents is silent graph corruption. Zero-expansion without dependents (no node references the template) is harmless and allowed.
**Trade-offs:** Operators must guard dependents with `when:` or ensure forEach always has at least one value.
**Sources:** Adversarial review (edge cases agent)
**Exploration:** quick
**Status:** captured

## D6: Implicit carry-forward across lifecycle phases

**Choice:** Nodes from earlier lifecycle phases are implicitly carried forward into later phases — injected into the later phase's `DesiredStateGraph` at compile time so `dependsOn` references resolve. Re-declaration is only needed to override a carried-forward node's spec.
**Alternatives:**
- Explicit re-declaration (earlier revision) — operators must repeat every node declaration in every phase that references it. DRY violation for large graphs.
**Rationale:** Carry-forward is a compile-time concern, not a runtime concern. The compile-time injection adds carried-forward nodes as already-PRESENT in the later phase's graph — the planner skips them. `LifecycleManager` semantics are unchanged: each phase still produces a separate `DesiredStateGraph`, and `compareAndSetDesired` works as before.
**Trade-offs:** Phases must be compiled sequentially (Phase N depends on Phase N-1's output). This sequential dependency is inherent.
**Sources:** Adversarial review (lifecycle agent, cross-cutting review R1-03)
**Exploration:** deep-analysis
**Status:** captured (revised — see spec §6.5)

## D7: Completion vocabulary — allPresent, never, bean reference

**Choice:** Three built-in completion conditions. Complex conditions use a named Java bean reference.
**Alternatives:**
- Extensible predicate DSL with composable `and:`/`or:` — unbounded complexity
- Only `allPresent` and `never` — insufficient for real deployments
**Rationale:** The runtime has exactly two built-in conditions (`allPresent()`, `never()`). A bean reference (`{ bean: "name" }`) provides a clean escape hatch to the `CompletionCondition` SPI without building a predicate language in YAML.
**Trade-offs:** Complex conditions require writing a Java `CompletionCondition` bean. This is deliberate — complex predicates belong in code.
**Sources:** Adversarial review (lifecycle agent), `CompletionCondition` SPI
**Exploration:** quick
**Status:** captured

## D8: Rules/invariants fire per-phase

**Choice:** In lifecycle mode, rules and invariants evaluate independently per phase's graph.
**Alternatives:**
- Cross-phase evaluation against a merged graph — requires a merged-graph concept that doesn't exist in the runtime
**Rationale:** The runtime reconciles one phase at a time. Each phase is an independent `DesiredStateGraph`. Cross-phase invariants would need a concept of "the full lifecycle graph" which the runtime doesn't have and the engines aren't designed for.
**Trade-offs:** Invariants like "every app must have a monitor" spanning phases 2 and 3 cannot be expressed declaratively. Use convention or lifecycle-level Java `CompletionCondition` instead.
**Sources:** Adversarial review (lifecycle agent), `LifecycleManager` implementation
**Exploration:** quick
**Status:** captured

## D9: String-only module parameters

**Choice:** Module parameters are strings only. No typed `nodeRef` parameter type.
**Alternatives:**
- Typed `nodeRef` parameters with cross-boundary validation — more powerful but builds a type system in YAML
**Rationale:** `nodeRef` requires a new validation layer (does this parameter value match a valid node ID in the importing graph?), introduces typed parameters into YAML, and is the "crack in the dam" toward Helm-style complexity. String parameters with build-time dependency validation (dangling reference check) achieve the same practical outcome.
**Trade-offs:** No compile-time distinction between "this parameter is a node reference" and "this parameter is a configuration value." Build-time dependency validation still catches invalid references.
**Sources:** Adversarial review (YAGNI agent — "kill nodeRef")
**Exploration:** quick
**Status:** captured

## D10: Module nesting capped at 2 levels

**Choice:** A module can import modules, but imported modules cannot import further modules.
**Alternatives:**
- Unlimited nesting — unbounded debugging complexity
- No nesting (flat modules only) — too restrictive for practical composition
**Rationale:** Crossplane's experience shows deep composition nesting is a major pain point for operators debugging reference chains. Two levels covers practical use cases (a monitoring module that imports a common alerting module). Deeper composition signals that the use case needs a Java `GoalCompiler`.
**Trade-offs:** Some composition patterns require flattening. This is a deliberate simplicity constraint.
**Sources:** Adversarial review (prior art agent — Crossplane nesting bugs)
**Exploration:** quick
**Status:** captured

## D11: GraphDescriptor becomes surface-agnostic via sealed descriptor interfaces

**Choice:** `GraphDescriptor` evolves from annotation-centric to surface-agnostic IR. Rule and invariant descriptors become sealed interfaces (`ReflectiveRuleDescriptor` / `DeclarativeRuleDescriptor`). YAML-specific expansion metadata (forEach, when) lives in a separate `ExpansionContext`.
**Alternatives:**
- Keep `GraphDescriptor` unchanged with all YAML data in `YamlExpansionContext` (earlier revision) — dual-path architecture where consumers handle two shapes for the same concept
- Extend `GraphDescriptor` with YAML fields — polymorphic descriptors, potential annotation processor breakage
**Rationale:** Rules and invariants are the same concept regardless of declaration surface. Sealed interfaces make both surfaces populate the same IR. Only forEach/when metadata — genuinely surface-specific expansion logic — stays in a separate `ExpansionContext`.
**Trade-offs:** Descriptor records become sealed interfaces — cascading type changes. Clean break worth making.
**Sources:** Adversarial review (architecture agent, structure review STR-R1-04), cross-cutting review R1-09
**Exploration:** deep-analysis
**Status:** captured (revised — see spec §8.2)

## D12: Phase implementation order

**Choice:** Phase 1: fault policy + invariants + when:. Phase 2: rules + lifecycle. Phase 3: forEach + modules.
**Alternatives:**
- All at once — 7 features × pairwise interactions = unmanageable risk
- Differentiators only (rules, invariants, fault policy) then everything else — puts rules (highest complexity) in Phase 1
**Rationale:** Phase 1 ships three low-to-medium complexity features that include two differentiators (fault policy, invariants) and one table-stakes feature (when:). Phase 2 tackles the highest-value differentiator (rules) alongside lifecycle phases. Phase 3 defers the highest-complexity table-stakes features (forEach, modules) until real operator demand validates the need.
**Trade-offs:** forEach and modules are deferred. Operators needing these features must use Java `GoalCompiler` or multiple YAML files until Phase 3.
**Sources:** Adversarial review (YAGNI agent), complexity analysis
**Exploration:** deep-analysis
**Status:** captured
