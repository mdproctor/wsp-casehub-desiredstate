## D1: Parameter binding model

**Choice:** Sequential chaining
**Alternatives:**
- Explicit reference via `of` attribute — more flexible but verbose
- All relative to first @Match — simpler but limits multi-hop patterns
**Rationale:** Each @DirectDep/@Reaches is relative to the PREVIOUS parameter. Left-to-right reading order matches rule-engine intuition (Drools pattern chaining). @Match binds independently, subsequent annotations navigate from the previous binding.
**Trade-offs:** Cannot reference an arbitrary earlier binding by name — must chain linearly. Multi-hop patterns with branching require multiple rules.
**Sources:** Issue #106 body (parameterized signature examples), Drools sequential pattern matching
**Exploration:** quick
**Status:** captured

## D2: Standalone rule scoping

**Choice:** @GraphRule on class with `graph` attribute
**Alternatives:**
- CDI qualifier matching — moves discovery to runtime, less build-time validation
- No standalone classes — limits reuse across graphs
**Rationale:** `@GraphRule(graph = "pipeline:medallion")` or wildcard `@GraphRule(graph = "pipeline:*")`. Processor discovers via Jandex scan at build time. Matches GraphDescriptor's namespace:name key. Consistent with @DeclareNode's namespace/name pattern.
**Trade-offs:** Wildcard matching adds complexity to the processor's rule collection logic.
**Sources:** Issue #106 body (standalone class example), @DeclareNode namespace/name pattern
**Exploration:** quick
**Status:** captured

## D3: @NotExists semantics

**Choice:** Both modes via attribute
**Alternatives:**
- Guard on type existence only — simpler but less expressive
- Guard on specific relationship only — misses global absence checks
- Sequential chaining only — inconsistent when global guard is needed
**Rationale:** `@NotExists(type = "validator")` defaults to global guard (fires only if NO node of that type exists). `@NotExists(type = "validator", of = "transformer")` checks relative to the named binding (fires for each transformer that has no validator dependency). Both modes serve distinct use cases.
**Trade-offs:** Two modes increase annotation complexity and engine branching. The `of` attribute introduces non-sequential reference — a departure from D1's chaining model, but justified because @NotExists is fundamentally different (guard vs binding).
**Sources:** Drools 'not' pattern (global), Rete negative join (relationship-specific)
**Exploration:** quick
**Status:** captured

## D4: Fixed-point loop semantics

**Choice:** Collect-then-apply
**Alternatives:**
- Apply-per-rule — faster convergence but rule ordering matters, non-deterministic
- Stratified — handles priority but adds complexity
**Rationale:** Run ALL rules against the current graph snapshot, collect ALL mutations, apply them all at once, re-run on the new graph. Rules see a consistent snapshot per iteration. Deterministic — rule ordering within an iteration doesn't affect the result.
**Trade-offs:** May require more iterations to converge than apply-per-rule (a rule can't see another rule's mutations until next iteration). Conflicting mutations within one iteration need a resolution strategy (dedup, last-wins, or error).
**Sources:** Standard production system semantics, Rete algorithm
**Exploration:** quick
**Status:** captured

## D5: Module placement

**Choice:** annotations/runtime
**Alternatives:**
- api/ — more reusable but pollutes api with annotation-specific concepts
- New annotations/engine module — clean separation but unnecessary module for one engine class
**Rationale:** The pattern matching engine is specific to the annotation compilation model. @Match/@Reaches/@DirectDep/@NotExists are annotation concepts. Programmatic GoalCompiler users write Java matching code directly. Recorder calls the engine during createGoalCompiler.
**Trade-offs:** If a future frontend (YAML, TypeScript) wants pattern-based rules, the engine would need extraction. Acceptable — extract when needed, not now.
**Sources:** Current annotations/runtime module structure, DesiredStateGraphRecorder.java
**Exploration:** quick
**Status:** captured

## D6: @GraphRule methods — static only

**Choice:** Static methods on @DesiredState interfaces
**Alternatives:**
- Both static and default — muddies the 'pure function of graph state' principle
- Default only — rules don't need instance state
**Rationale:** Rules are pure functions of graph state — no instance state needed. Consistent with @Customize on graph which is also static. On standalone classes, the conventional method is an instance method (natural for a class).
**Trade-offs:** Cannot access @Node method results directly in a rule. Rules must query the graph parameter instead. This is the correct design — rules operate on the graph, not on the interface's factory methods.
**Sources:** Issue #106 body (static method examples), @Customize pattern
**Exploration:** quick
**Status:** captured

## D7: Iteration cap and non-convergence

**Choice:** 100 iterations, RuntimeException
**Alternatives:**
- Configurable via @DesiredState attribute — flexibility most users won't need
- 50 iterations with warning + truncate — risks silently broken graphs
**Rationale:** Cap at 100 iterations. If exceeded, throw RuntimeException naming the rules that produced mutations in the last iteration. Real convergence happens in 2-5 iterations; 100 is a generous safety net.
**Trade-offs:** Hard failure on non-convergence means a bad rule set prevents startup entirely. This is correct — a non-converging graph is never safe to deploy.
**Sources:** Drools iteration caps, production system convergence literature
**Exploration:** quick
**Status:** captured

## D8: Standalone class shape

**Choice:** Rule containers — convention over annotation
**Alternatives:**
- Single method per class — SRP, simple, one-class-one-rule
- Multiple annotated methods — explicit but more annotation burden
**Rationale:** A standalone class is a container. Every public method returning `List<GraphMutation>` that has valid rule parameters is automatically a rule. @GraphRule(graph = "...") on the class scopes all methods to that graph. Maximally concise, convention-driven.
**Trade-offs:** Implicit discovery — adding a public method that happens to return `List<GraphMutation>` accidentally makes it a rule. Mitigated by the parameter signature requirement (must have either DesiredStateGraph or annotated DesiredNode params).
**Depends on:** D2 (standalone scoping)
**Sources:** Issue #106 body (standalone class example)
**Exploration:** quick
**Status:** captured

## D9: @Reaches reachability direction

**Choice:** Follow dependency direction (forward edges toward roots)
**Alternatives:**
- Reverse direction — 'who consumes this?' but less intuitive
- Either direction — most permissive but surprising matches
**Rationale:** @Match(type="tx") @Reaches(type="source") walks the dependency chain from tx toward roots. "tx depends on ... depends on source." Natural reading — dependencies point toward prerequisites.
**Trade-offs:** Cannot express "find what depends on me" with @Reaches. Use @Match with reverse graph queries in imperative rules for that pattern.
**Sources:** ImmutableDesiredStateGraph forward/reverse edge model, issue #106 examples
**Exploration:** quick
**Status:** captured
