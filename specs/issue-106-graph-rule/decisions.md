## D1: Parameter binding model

**Choice:** Sequential chaining with optional named binding via `of`
**Alternatives:**
- Sequential chaining only — simple but can't express branching patterns without splitting into multiple rules
- All relative to first @Match — simpler but limits multi-hop patterns
- Mandatory named bindings — flexible but verbose for simple chains
**Rationale:** Each @DirectDep/@Reaches is relative to the PREVIOUS parameter by default (sequential chaining). All traversal annotations (@DirectDep, @Reaches) and guards (@NotExists) accept an optional `of` attribute to reference an earlier binding by parameter name. When `of` is omitted, sequential chaining applies. Simple linear chains use implicit sequential chaining for ergonomics; branching patterns use `of` for explicit reference. Bindings are named by their Java parameter name (Jandex MethodParameterInfo, standard for Quarkus projects compiled with `-parameters`).
**Trade-offs:** `of` adds optionality — rule authors choose between implicit sequential and explicit named reference. Default (sequential) handles most patterns; `of` is opt-in for branching.
**Sources:** Issue #106 body (parameterized signature examples), Drools sequential pattern matching, review R1-02
**Exploration:** quick
**Status:** revised (R1-02: unified sequential + named binding; `of` available on all annotations, not just @NotExists)

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

**Choice:** Both modes via attribute, with explicit direction
**Alternatives:**
- Guard on type existence only — simpler but less expressive
- Guard on specific relationship only — misses global absence checks
- Direction as separate annotations (@NotExistsUpstream/@NotExistsDownstream) — proliferates annotations
- Implicit direction (always same as @Reaches) — wrong for the primary use case
**Rationale:** `@NotExists(type = "validator")` defaults to global guard (fires only if NO node of that type exists in the graph). `@NotExists(type = "validator", of = "transformer", direction = DEPENDENTS)` checks relative to the named binding in the specified direction. Direction is REQUIRED when `of` is specified — no implicit default, forcing rule authors to be explicit about traversal semantics. `DEPENDENCIES` checks forward edges (what the binding depends on, via `dependenciesOf()`). `DEPENDENTS` checks reverse edges (what depends on the binding, via `dependentsOf()`). The `of` attribute references a binding by parameter name (unified with D1's named binding model).
**Trade-offs:** Three attributes (type, of, direction) for relational guards. Necessary complexity — the motivating use case ("every transformer needs a downstream validator") requires DEPENDENTS direction, which cannot be inferred from context.
**Sources:** Drools 'not' pattern (global), Rete negative join (relationship-specific), DesiredStateGraph.dependenciesOf/dependentsOf, review R1-03
**Exploration:** quick
**Status:** revised (R1-03: added explicit direction attribute, required when `of` is specified; eliminated ambiguity in traversal semantics)

## D4: Fixed-point loop semantics

**Choice:** Collect-then-apply with conflict detection and idempotency requirement
**Alternatives:**
- Apply-per-rule — faster convergence but rule ordering matters, non-deterministic
- Stratified — handles priority but adds complexity
- Silent last-wins on conflict — non-deterministic, violates collect-then-apply invariant
**Rationale:** Run ALL rules against the current graph snapshot, collect ALL mutations, apply them all at once, re-run on the new graph. Rules see a consistent snapshot per iteration. Deterministic — rule ordering within an iteration doesn't affect the result. Conflict resolution: adopt FaultPolicyEngine's existing pattern — group mutations by target NodeId, throw ConflictingMutationException when multiple distinct mutations target the same node in the same iteration. Additionally, validate the composed mutation set for cycle introduction before applying. Idempotency requirement: rules must be deterministic functions of graph state — given the same graph, a rule must produce the same mutations (or no mutations). Non-deterministic specs (timestamps, random seeds) prevent convergence. The annotation model handles idempotency by construction: @NotExists only matches when the pattern is absent, so after the first iteration adds the missing node, the guard no longer fires. Imperative rules bear this responsibility explicitly.
**Trade-offs:** Error on conflict is strict — no "last-wins" flexibility. This is correct: if two rules disagree about a node's state, that's a rule design error, not a conflict to resolve silently. May require more iterations to converge than apply-per-rule.
**Sources:** Standard production system semantics, Rete algorithm, FaultPolicyEngine.evaluate(), ConflictingMutationException, review R1-04, R1-08
**Exploration:** quick
**Status:** revised (R1-04: explicit conflict resolution via ConflictingMutationException; R1-08: explicit idempotency requirement)

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

**Choice:** 100 iterations, RuntimeException with diagnostic guidance
**Alternatives:**
- Configurable via @DesiredState attribute — flexibility most users won't need
- 50 iterations with warning + truncate — risks silently broken graphs
**Rationale:** Cap at 100 iterations. If exceeded, throw RuntimeException naming the rules that produced mutations in the last iteration. Error message should suggest non-idempotent rules as a likely cause: rules that produce non-deterministic specs (timestamps, random seeds) or fail to check existing graph state before producing mutations. Real convergence happens in 2-5 iterations; 100 is a generous safety net.
**Trade-offs:** Hard failure on non-convergence means a bad rule set prevents startup entirely. This is correct — a non-converging graph is never safe to deploy.
**Sources:** Drools iteration caps, production system convergence literature, review R1-08
**Exploration:** quick
**Status:** revised (R1-08: diagnostic guidance for non-idempotent rules in error message)

## D8: Standalone class shape

**Choice:** Explicit @GraphRule per method
**Alternatives:**
- Convention over annotation (every matching public method is a rule) — accidental discovery hazard
- Single method per class — SRP, simple, one-class-one-rule
**Rationale:** A standalone class is scoped to a graph by class-level `@GraphRule(graph = "...")`. Each rule method within the class is explicitly marked with `@GraphRule`. Consistent with @Node method marking on @DesiredState interfaces (#102) and with D6's interface-level @GraphRule. Eliminates the accidental discovery hazard: utility methods like `rebalanceIfNeeded(DesiredStateGraph graph)` returning `List<GraphMutation>` are NOT accidentally treated as rules. One annotation per method — trivial cost, significant safety gain.
**Trade-offs:** Slightly more verbose than convention-based discovery. Worth it for safety and consistency with the established @Node pattern.
**Depends on:** D2 (standalone scoping)
**Sources:** Issue #106 body (standalone class example), @Node method marking pattern from #102, review R1-05
**Exploration:** quick
**Status:** revised (R1-05: require explicit @GraphRule per method to prevent accidental discovery)

## D9: Traversal direction — @Reaches and @DirectDep

**Choice:** Follow dependency direction by default, with optional direction override on both traversal annotations
**Alternatives:**
- Forward only (no override) — can't express "find what depends on me" in annotations
- Reverse direction as default — 'who consumes this?' but less intuitive for the common case
- Either direction (bidirectional search) — most permissive but surprising matches
- Direction on @Reaches only — creates asymmetry where direct-edge reverse traversal requires transitive @Reaches
**Rationale:** Both traversal annotations accept an optional `direction` attribute, default DEPENDENCIES. `@Reaches(type="source")` walks the dependency chain transitively toward roots. `@DirectDep(type="validator")` matches a direct dependency only. Both accept `direction=DEPENDENTS` for reverse traversal: `@Reaches(type="consumer", direction=DEPENDENTS)` finds transitive dependents; `@DirectDep(type="validator", direction=DEPENDENTS)` binds a node that directly depends on the binding (from `dependentsOf()`). Direction values match DesiredStateGraph API: DEPENDENCIES maps to `dependenciesOf()`, DEPENDENTS maps to `dependentsOf()`. Without direction on @DirectDep, "bind the validator that directly depends on this transformer" forces @Reaches with DEPENDENTS — which overmatches transitively.
**Trade-offs:** Direction attribute adds one more option on two annotations. Default (DEPENDENCIES) handles most patterns; override is opt-in for reverse traversal.
**Sources:** ImmutableDesiredStateGraph forward/reverse edge model, issue #106 examples, review R1-03, R2-01
**Exploration:** quick
**Status:** revised (R1-03: added direction to @Reaches; R2-01: extended direction to @DirectDep for consistency)

## D10: @GraphRule vs FaultPolicy boundary

**Choice:** Separate mechanisms, no shared pattern matching infrastructure
**Alternatives:**
- Shared pattern matching engine in runtime/ usable by both — architecturally clean but premature
- Unified rule model replacing both — radical scope change beyond issue #106
**Rationale:** @GraphRule and FaultPolicy serve architecturally distinct roles with different inputs and execution contexts. @GraphRule operates at compile-time on desired graph structure only, with fixed-point convergence to quiescence. FaultPolicy operates at runtime, responding to FaultEvents with access to desired graph + actual state + fault context. The input signatures differ (`DesiredStateGraph` only vs `FaultEvent + DesiredStateGraph + ActualState`), the execution contexts differ (GoalCompiler phase vs reconciliation cycle), and the matching semantics differ (structural pattern matching vs event-type + node-type filtering). ThresholdFaultPolicy's type filtering is well-served by its builder API and the TypedFaultPolicy abstraction (#112) — it doesn't need a general-purpose pattern matching engine. If a future issue surfaces a need for FaultPolicy to do structural graph matching, the engine can be extracted from annotations/runtime/ to runtime/ at that point — the extraction is a class move, not a redesign.
**Trade-offs:** Two parallel graph mutation mechanisms exist in the platform. Acceptable because they serve fundamentally different triggers and contexts. Premature unification would force FaultPolicy's event-driven model into a pattern-matching paradigm that doesn't fit its primary use case.
**Sources:** FaultPolicy.java, ThresholdFaultPolicy.java, TypedFaultPolicy (#112), FaultPolicyEngine.java, review R1-07
**Exploration:** quick (surfaced by reviewer)
**Status:** captured
