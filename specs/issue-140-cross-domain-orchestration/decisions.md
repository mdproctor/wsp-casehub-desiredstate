## D1: Architectural model — hierarchical from day one

**Choice:** Hierarchical architecture with good defaults so single-process deployments remain simple. Meta-loop with domain-level nodes, inner loops per domain. The hierarchical model subsumes single-process and multi-process as deployment configurations. Single-process simplicity is achieved via transparent flattening (D6), not by avoiding the architecture.
**Alternatives:**
- Flat composition only — single merged graph, no meta-loop. Meets the known single-process need but cannot support multi-process domain separation without architectural extension. The review (R1-02) argued YAGNI; the user overrode this — the project builds for the future, not just current consumers.
- CompositionStrategy SPI — flat by default, hierarchy as an alternative strategy via CDI displacement. Clean extension point but premature: an SPI with one implementation is an abstraction without a consumer. Can be introduced later if multiple composition strategies emerge.
**Rationale:** The hierarchical model is the correct long-term architecture. It subsumes single-process (via flattening) and multi-process (via inner loops) as deployment configurations of one design. Building it from day one avoids the cost of bolting on hierarchy later — which would require rearchitecting the composition engine, adding domain-level node synthesis, and introducing inner loop lifecycle management retroactively.
**Trade-offs:** More framework complexity upfront. Two code paths (flattened and hierarchical) require separate testing. Mitigated by D6 (transparent flattening) which ensures the common single-process case is simple and well-tested.
**Depends on:** D6 (transparent flattening makes hierarchical scale down)
**Sources:** casehub-desiredstate#140, casehub-ops#23, ReconciliationLoop.java, LifecycleManager.java, DesiredStateGraph.overlay()
**Exploration:** quick
**Status:** restored — user override of review YAGNI cut. "We don't do this based on our consumers now, we build for the future."

## D2: Domain registration — push model

**Choice:** Each domain compiles its own goals (type-safe, domain's concern) and registers the compiled `CompilationResult` plus metadata (`provides: Set<NodeType>`, `requires: Set<NodeType>`, optional `CompletionCondition`) with the `CrossDomainCompositionEngine` at startup. Registration via CDI startup observers — each domain JAR includes a `@ApplicationScoped` bean that `@Observes StartupEvent` and calls `engine.registerDomain()`. No DomainDescriptor SPI.
**Alternatives:**
- DomainDescriptor pull model — composition engine discovers descriptors via CDI, calls `descriptor.compile()`. Introduces the `GoalCompiler<G>` type erasure problem: the composition layer must call `compile()` without knowing `G`. DomainDescriptor bundles four concerns (compiler, goal loader, metadata, completion condition) in one SPI — coupling that isn't necessary when domains drive their own compilation.
- Raw `GoalCompiler<Object>` erasure with separate `GoalProvider` SPI — splits registration across two SPIs, requires matching key, uses unchecked casts. Worst alternative.
**Rationale:** The type erasure problem exists only when the composition layer calls `compile()`. With push, domains compile themselves — `GoalCompiler<AttackBlueprint>`, `GoalCompiler<DefenseBlueprint>` etc. are called by domain code that knows the type parameter. The composition engine receives `CompilationResult` (no generic parameter). The spatial example already demonstrates this pattern: `AttackGoalCompiler`, `DefenseGoalCompiler`, `DistributionGoalCompiler` each compile independently with their own blueprint type, and results merge via `overlay()`. Registration via `@Observes StartupEvent` follows the platform's established CDI startup pattern.
**Trade-offs:** The composition engine cannot trigger initial compilation — domains must compile at startup and register. For recompilation, see D10 (SituationRecompiler interaction). Registration ordering relies on CDI `@Priority` or lifecycle phasing.
**Sources:** GoalCompiler.java, AttackGoalCompiler.java, DefenseGoalCompiler.java, DistributionGoalCompiler.java, ForceDistributionTest.java (overlay pattern), CdiNodeProvisionerRouter (CDI collection pattern)
**Exploration:** quick
**Status:** kept — review revision (R1-03) confirmed by user. Push eliminates type erasure entirely.

## D3: Steady-state definition — pluggable CompletionCondition

**Choice:** Reuse the existing `CompletionCondition` SPI. Each domain provides a condition at registration (D2). Default: all nodes PRESENT, zero active faults. In hierarchical mode, the CompletionCondition determines when a domain-level node transitions to "ready" — the meta-loop advances to downstream domains. In flattened mode (D6), cross-domain ordering is handled structurally by type-based graph edges — CompletionCondition is not evaluated for cross-domain readiness (provides/requires edges enforce ordering via TransitionPlanner).
**Alternatives:**
- All nodes PRESENT, zero faults (hardcoded) — strictest, but a non-critical node failure blocks downstream domains indefinitely
- Sentinel nodes — domain nominates specific nodes as readiness markers; adds API surface to the compiler
- Separate ReadinessCondition SPI — unnecessary new concept; CompletionCondition already has the right signature
**Rationale:** `CompletionCondition` already exists in the codebase (`Phase` record uses it). The default is strict (safe). Domains that need softer semantics override explicitly. No new concept introduced. The dual role (lifecycle phases + cross-domain readiness in hierarchical mode) uses the same interface with the same semantics — "has this set of nodes reached a sufficient state?"
**Trade-offs:** Domains with mixed-criticality nodes must implement a custom condition or model criticality via finer NodeTypes. In flattened mode, CompletionCondition is unused for cross-domain ordering (graph edges handle it), so the condition is only exercised in hierarchical mode.
**Depends on:** D1 (hierarchical architecture needs CompletionCondition for domain readiness)
**Sources:** CompletionCondition.java, Phase.java, LifecycleManager.java, TransitionPlanner.java
**Exploration:** quick
**Status:** restored — user override of review YAGNI cut. Hierarchical mode (D1) requires CompletionCondition for cross-domain readiness.

## D4: Cross-domain dependency granularity — type-level

**Choice:** Type-level cross-domain dependencies. "Deployment roots depend on infra's namespace-type nodes." Expressed declaratively via provides/requires, runtime converts to graph edges (flattened mode) or type-scoped CompletionConditions (hierarchical mode).
**Alternatives:**
- Coarse (domain-level only) — simple but wasteful; domain B waits for ALL of domain A, even slow nodes it doesn't need (e.g. database clusters blocking deployment that only needs namespaces)
- Fine-grained (node-level) — most precise but creates naming coupling between independent domain compilers; fragile if a domain renames node IDs
- Type-level with predicate refinement — "deployment depends on infra's NAMESPACE nodes matching `namespace.name == deployment.targetNamespace`". Adds structural selectivity but couples the wiring layer to node spec internals. Types are the API boundary; content is opaque.
**Rationale:** Type-level is the right abstraction boundary — more precise than domain-level (avoids unnecessary blocking), no naming coupling (domains don't reference each other's node IDs). If a domain has semantically distinct resources with different provisioning timelines, they SHOULD be distinct NodeType values — NAMESPACE and DATABASE_CLUSTER are semantically different, not just "fast" and "slow". This reflects domain semantics, not orchestration concerns. Predicate refinement can be added later if the type-level model proves insufficient, but adds coupling (wiring layer must understand node spec structure) that should be avoided without evidence of need.
**Trade-offs:** Less precise than node-level deps. A slow node of a depended-upon type blocks even if the dependent only needs a fast one. Mitigated by finer NodeType modelling — which reflects semantic distinctions in the domain, not orchestration speed.
**Sources:** DesiredStateGraph.overlay(), DesiredStateGraph.connect(), TransitionPlanner ordering, NodeType value type, first-principles analysis of flattened vs hierarchical modes
**Exploration:** deep-analysis
**Status:** kept — unchanged from original. Valid in both flattened and hierarchical modes.

## D5: Goal loading — domain responsibility

**Choice:** Goal loading is the domain's responsibility. Each domain loads its own configuration, compiles using its own `GoalCompiler<G>`, and registers the compiled `CompilationResult` with the composition engine. The composition layer never sees domain-specific configuration or goal types.
**Alternatives:**
- Configuration-driven via DomainDescriptor — composition layer passes shared config source, each descriptor extracts domain-specific goals. Requires the superseded DomainDescriptor model.
- Programmatic — consumer constructs each domain's goals explicitly. This is the current working pattern.
**Rationale:** Follows from D2 (push model). With push, domains compile themselves — goal loading is inherently the domain's concern. The composition engine receives `CompilationResult`, not goals. No shared configuration contract needed. Each domain can use whatever configuration mechanism suits it (YAML, annotations, programmatic, Preferences).
**Trade-offs:** None beyond D2's trade-offs. Domain autonomy over configuration is a feature, not a trade-off.
**Sources:** InfraGoalCompiler.java, DeploymentGoalCompiler.java, YAML surface (YamlGraphRecorder), annotation surface
**Exploration:** quick
**Status:** kept — follows from D2 push model.

## D6: Flattening strategy — transparent for single-process

**Choice:** In single-process deployment (default), the composition layer detects all descriptors in one JVM, merges domain graphs into one via `overlay()`, adds type-based cross-domain edges derived from provides/requires, and feeds the merged graph to a single `ReconciliationLoop`. The meta-loop doesn't run. The consumer never sees domain-level nodes. Hierarchical mode is activated via explicit config property (`desiredstate.composition.mode=flattened|hierarchical`, default `flattened`).
**Alternatives:**
- Always hierarchical — even in single-process, meta-loop runs with domain-level nodes and inner loops. Consistent mental model but adds overhead (multiple loop instances, domain-level node synthesis) when a merged graph suffices.
- Flat-only, no hierarchical — eliminates the two-code-path concern but abandons the hierarchical architecture (rejected by user — see D1).
**Rationale:** This is how hierarchical "scales down" to single-process simplicity. The flattened path reuses existing primitives (`overlay()`, `TransitionPlanner`, single `ReconciliationLoop`). Single-process ops deployments get the same performance and debugging experience as today's single-domain case. The hierarchical machinery only activates when the consumer explicitly opts in.
**Trade-offs:** Two code paths (flattened and hierarchical) require separate testing. Behaviour differences between modes must be documented. The review (R1-07) identified this as a maintenance cost — mitigated by the flattened path being well-tested as the default and the hierarchical path being an explicit opt-in.
**Depends on:** D1 (hierarchical architecture), D4 (type-level deps manifest as edges in flattened mode)
**Sources:** DesiredStateGraph.overlay(), TransitionPlanner, ReconciliationLoop
**Exploration:** quick
**Status:** restored — user override of review YAGNI cut. Transparent flattening is how D1's hierarchical architecture scales down.

## D7: CDI discovery — automatic via startup registration

**Choice:** `CrossDomainCompositionEngine` is `@ApplicationScoped`. Domains register via CDI startup observers (`@Observes StartupEvent` at default or explicit `@Priority`). The engine composes in its own `@Observes @Priority(PLATFORM_AFTER + 1000) StartupEvent` observer — this fires after all domain registrations because CDI observers of the same event fire in `@Priority` order (lower value = earlier). One registration = single-domain passthrough (engine passes `CompilationResult` directly to `LifecycleManager`, no merging). Multiple registrations = auto-composition. Zero registrations = no-op. Mode selection (`flattened` or `hierarchical`) via config property (D6).
**Alternatives:**
- Explicit `compose()` call — consumer triggers composition after registering all domains. Explicit and robust but requires app code, breaking "just add JARs" auto-activation.
- Post-startup lifecycle event — engine observes a custom event fired after `StartupEvent` processing. Clean separation but requires a custom event definition.
- Instance<DomainDescriptor> injection — CDI auto-discovery of descriptor beans. Superseded by D2 push model.
**Rationale:** CDI `@Priority` ordering on `StartupEvent` observers is the standard Quarkus mechanism for sequencing startup work. The composition engine at `PLATFORM_AFTER + 1000` fires after all application-level observers (which use default or lower priority). The convention is validated at startup: if the engine has zero registrations and at least one domain JAR is on the classpath (detectable via CDI `Instance<NodeProvisioner>` being non-empty), it logs a warning about likely misconfiguration.
**Trade-offs:** Registration ordering depends on CDI `@Priority` convention — a domain observer with priority above `PLATFORM_AFTER + 1000` would register after composition. Mitigated by documenting the convention and the engine's startup validation.
**Depends on:** D2 (push model registration), D6 (mode selection via config), D8 (provides/requires for ordering)
**Sources:** CdiNodeProvisionerRouter, CdiActualStateAdapterRouter, CdiMergedEventSource (existing CDI compositor pattern)
**Exploration:** quick
**Status:** kept — review's CDI @Priority convention is sound.

## D8: Cross-domain dependency declaration — provides/requires with validation

**Choice:** Each domain declares `provides: Set<NodeType>` (types this domain creates) and `requires: Set<NodeType>` (types needed before starting) at registration. Composition engine matches requires→provides automatically. Domains reference NodeTypes, not other domains. Explicit validation at startup:
1. **Duplicate provides** — two domains claiming the same NodeType → fail fast. Consistent with `NodeProvisionerRouter` which already fails fast on overlapping `handledTypes()`.
2. **Circular requires** — topological sort of domain ordering at startup; cycle detected → fail fast with descriptive error.
3. **Partial readiness** — if domain B requires NodeType X from domain A's provides {X, Y, Z}, cross-domain edges (flattened) or CompletionCondition scoping (hierarchical) connect B to A's X-type nodes only. B starts as soon as A's X nodes are available, regardless of Y and Z.
**Alternatives:**
- In dependent domain's descriptor only — descriptor references another domain's node types by name. Soft compile-time awareness, but workable.
- Separate orchestration config — cross-domain manifest (YAML/annotation). Clean separation but another artifact to maintain.
**Rationale:** Fully decoupled — domains don't reference each other by name, only by abstract NodeType. Wiring is automatic (requires/provides matching). Misconfiguration caught at startup. Naturally scopes CompletionCondition (D3) in hierarchical mode and generates edges in flattened mode (D6).
**Trade-offs:** Requires that cross-domain dependencies are expressible via NodeType. Domains with untyped or single-typed nodes may need to introduce finer types to express partial dependencies.
**Depends on:** D4 (type-level granularity)
**Sources:** NodeType value type, NodeProvisionerRouter.handledTypes() pattern, first-principles analysis
**Exploration:** quick
**Status:** kept — review's validation additions (duplicate-provides, cycle detection, partial readiness) are valuable.

## D9: Module placement — composition engine in runtime/

**Choice:** `CrossDomainCompositionEngine` in runtime/ (dedicated package: `io.casehub.desiredstate.runtime.composition`). With the push model (D2), there is no DomainDescriptor SPI in api/. The composition engine exposes a runtime registration API (`registerDomain()`). `CompletionCondition` is already in api/ (D3) — no new api/ types needed.
**Alternatives:**
- New orchestration/ module — unnecessary. The composition engine is tightly coupled to runtime APIs (`ReconciliationLoop`, `LifecycleManager`, `TransitionPlanner`). A separate module would not enable independent versioning.
- DomainDescriptor SPI in api/ — superseded by D2 push model.
**Rationale:** runtime/ already hosts CDI compositors (`FaultPolicyEngine`, `SituationRecompilerEngine`, routers, `MergedEventSource`). The composition engine follows the same pattern. Dedicated package keeps composition code isolated within runtime/. No new module means no extra dependency for multi-domain apps.
**Trade-offs:** runtime/ grows. Mitigated by dedicated package for composition code.
**Sources:** api/ module (existing routers), runtime/ module (existing compositors)
**Exploration:** deep-analysis
**Status:** kept — valid regardless of flat vs hierarchical.

## D10: LifecycleManager interaction — composition engine wraps with SituationRecompiler integration

**Choice:** Composition engine is the top-level entry point. It wraps `LifecycleManager` — composition above lifecycle above reconciliation. Explicit SituationRecompiler integration:

1. **Initial composition:** Engine merges all registered domain graphs (overlay + cross-domain edges in flattened mode; meta-loop domain-level nodes in hierarchical mode) and calls `LifecycleManager.start(tenancyId, composedResult)`.
2. **SituationRecompiler flow:** `SituationRecompiler` returns `CompilationResult` scoped to one domain. The composition engine intercepts: replaces that domain's contribution, re-merges with other domains' current graphs + cross-domain edges, calls `LifecycleManager.updateDesired(tenancyId, newComposedResult)`.
3. **Domain matching:** Domains register their `SituationRecompiler`s alongside their graphs via `registerDomain()`. The composition engine maps each recompiler to its domain at registration time — no SPI change to `SituationRecompiler` in api/.
4. **Cascade detection:** If a domain's recompilation removes a NodeType from its provides set, the engine detects that downstream domains' requires are now unsatisfied. Initial behavior: fail fast with descriptive error. Cascade recompilation is a future evolution.
5. **Per-domain lifecycle tracking:** See D13.

**Alternatives:**
- Replace LifecycleManager — composition engine subsumes phase transition logic. Simpler call stack but conflates domain ordering and phase transitions, requires reimplementing phase CAS logic that already works.
- LifecycleManager directly receives SituationRecompiler results — stale composition engine view; engine and LifecycleManager fight over desired state.
- Add `domainId()` to SituationRecompiler SPI — makes the api/ SPI aware of cross-domain composition, which is a runtime concern.
**Rationale:** LifecycleManager's CAS-based phase transitions work well and shouldn't change. The composition engine adds domain orchestration above it. SituationRecompiler results must flow through the composition engine so it can re-compose. Domain matching happens at registration time (push model), keeping the `SituationRecompiler` SPI in api/ composition-agnostic.
**Trade-offs:** The composition engine must track per-domain CompilationResult and per-domain SituationRecompilers. The `registerDomain()` API includes optional SituationRecompilers.
**Depends on:** D1 (hierarchical architecture), D2 (push model), D13 (per-domain lifecycle state)
**Sources:** LifecycleManager.java, ReconciliationLoop.java, SituationRecompilerEngine.java, SituationRecompiler.java
**Exploration:** quick
**Status:** kept — review's registration-based SituationRecompiler matching is sound.

## D11: Tenancy model — same tenant for composed domains

**Choice:** Composed domains share the same `tenancyId`. The composition engine produces one merged graph per tenant. All domain nodes within a composition belong to the same tenant.
**Alternatives:**
- Per-domain tenant mapping — each domain has its own tenancyId. Would require cross-tenant edges and cross-tenant reconciliation coordination. Significantly more complex with no known consumer.
- Shared resource pool — a domain like infra provisions resources shared across tenants. This is a separate ReconciliationLoop, not part of multi-domain composition for any single tenant.
**Rationale:** `ReconciliationLoop` is per-tenant. The merged graph is per-tenant. Cross-domain dependencies (infra namespaces → deployments) are within the same tenant. If a domain provisions shared infrastructure across tenants, it operates as an independent reconciliation loop, not as a composed domain within a tenant's graph.
**Trade-offs:** Cannot model cross-tenant dependencies within the composition engine. Acceptable: cross-tenant coordination is an orchestration concern above the desired-state runtime.
**Sources:** ReconciliationLoop.java (per-tenant TenantLoop), ProvisionContext (carries tenancyId)
**Exploration:** surfaced-by-review
**Status:** kept — valid regardless of flat vs hierarchical.

## D12: Fault propagation — mode-dependent

**Choice:** In flattened mode, faults propagate through the existing infrastructure on the merged graph. A node in domain A fails → `FaultPolicyEngine` evaluates → mutations applied to the merged graph. Domain B's nodes are in the same graph — cross-domain edges carry dependency semantics, and `TransitionPlanner`/`FaultPolicyEngine`/`ReconciliationLoop` operate on the unified graph without needing cross-domain mediation.

In hierarchical mode, each domain has its own inner loop and fault handling. Cross-domain fault propagation would require the meta-loop to observe inner loop faults and trigger downstream domain responses. Initial implementation: inner loop faults are domain-local. Cross-domain fault escalation is a future evolution.

**Alternatives:**
- CloudEvents from domain A consumed by domain B's fault policies — adds coupling between domain fault policies.
- Composition engine mediates fault signals — adds a new fault propagation layer.
**Rationale:** The flat model's chief advantage: cross-domain faults are just faults in a single graph. The existing `FaultPolicyEngine` evaluates all fault policies against the merged graph. Policies from different domains naturally compose (they handle different NodeType/FaultType combinations).
**Trade-offs:** In flattened mode, a fault policy from domain A could mutate nodes from domain B if it has visibility into B's node types. This is a feature for trusted composition but an implicit trust extension.
**Trust assumption:** Composed domains are trusted — authored by the same team (casehub-ops is the first consumer, single team). For untrusted domain composition, fault policy scoping via NodeType-based filtering in `FaultPolicyEngine` would be needed — future evolution.
**Sources:** FaultPolicyEngine.java, ReconciliationLoop.reconcile(), ThresholdFaultPolicy
**Exploration:** surfaced-by-review
**Status:** kept — trust assumption made explicit; hierarchical mode noted as domain-local faults initially.

## D13: Per-domain lifecycle state — composition engine manages internally

**Choice:** The composition engine manages per-domain lifecycle state. Each domain registers a `CompilationResult` — either `SingleGraph` or `Lifecycle(List<Phase>)`. The engine tracks which phase each domain is at.

In flattened mode: the composed graph at any moment is the overlay of each domain's current-phase graph plus cross-domain edges. When a domain's phase completes (its `CompletionCondition` is satisfied for its nodes in the actual state), the engine advances that domain, re-composes, and calls `LifecycleManager.updateDesired()`. Phase completion is detected via `GlobalReconciliationListener` — the CDI-discovered, multi-instance listener that fires for all tenants after every full reconciliation cycle. This avoids competing with `LifecycleManager` for the per-tenant `ReconciliationListener` slot.

In hierarchical mode: per-domain lifecycle maps to the domain-level node's inner loop. Phase transitions are managed by `LifecycleManager` within the inner loop, as it works today for single-domain deployments.

**Alternatives:**
- Single composed Lifecycle — combinatorial explosion: A:2phases × B:3phases = 6 composed phases. Unworkable beyond two domains.
- LifecycleManager manages per-domain phases — would require LifecycleManager to understand domain composition, breaking its single-responsibility.
**Rationale:** The composition engine is already tracking per-domain contributions (D10). Extending it to track per-domain phase state is natural. `LifecycleManager` continues to manage the single composed graph's lifecycle — it doesn't need to know about domains.

Note: `GlobalReconciliationListener` fires only from full `reconcile()`, not from type-filtered `reconcileTypes()`. This is correct — `CompletionCondition` should evaluate against full actual state, not a type-filtered subset.

**Trade-offs:** The composition engine grows in responsibility: domain registration, graph merging, cross-domain edges, per-domain lifecycle tracking, and re-composition on phase transitions. Mitigated by clear internal separation (dedicated package, distinct methods for each concern).
**Sources:** LifecycleManager.java, CompilationResult.Lifecycle, Phase.java, CompletionCondition.java, GlobalReconciliationListener.java, ReconciliationLoop.java (TenantLoop.fireGlobalListeners line 573)
**Exploration:** surfaced-by-review
**Status:** kept — GlobalReconciliationListener mechanism is sound. Hierarchical mode interaction noted.

## D14: Graph versioning — single composed graph, single CAS

**Choice:** In flattened mode, the composed graph is a single `DesiredStateGraph` with its own version counter. All CAS operations operate on this single graph. Per-domain "subgraphs" don't have independent versions — they are merged into one graph at composition time.

In hierarchical mode, the meta-loop has its own graph (domain-level nodes) with its own CAS. Each inner loop has its own graph with its own CAS. No cross-loop CAS coordination needed — the meta-loop and inner loops are independent reconciliation loops.

When the composition engine re-composes (due to domain recompilation, phase transition, or SituationRecompiler):
1. Builds a new composed/meta-loop graph
2. Calls `LifecycleManager.updateDesired()` or `compareAndSetDesired()` with the new graph
3. The new graph gets a new version from `ImmutableDesiredStateGraph` construction

**Alternatives:**
- Dual versioning — composed graph version + per-domain subgraph versions. Complex and error-prone.
- Per-domain CAS — requires distributed locking.
**Rationale:** One graph per loop, one CAS per loop. The composition engine is the single writer to each loop. Existing CAS semantics unchanged.
**Trade-offs:** Concurrent SituationRecompiler triggers for different domains must be serialized through the composition engine. Acceptable: situation-triggered recompilation is infrequent.
**Sources:** ImmutableDesiredStateGraph (version counter), ReconciliationLoop.compareAndSetDesired(), LifecycleManager.java
**Exploration:** surfaced-by-review
**Status:** kept — extended to cover hierarchical mode (per-loop CAS independence).

## D15: Node ID uniqueness — convention-based with composition engine validation

**Choice:** Domains in cross-domain composition must use globally unique `NodeId` values. Enforced by convention (domain-specific ID prefixes) and validated by the composition engine at registration time. Collisions produce a domain-attributed error message identifying both domains and the conflicting ID.

Convention: domain-prefixed IDs (e.g., `infra:namespace-default`, `deploy:agent-main`). Consistent with existing examples — dungeon uses `room-*`, `goblin-*`; pipeline uses `metadata-*`, `ingestion-*`; spatial uses `cell-*`, `scout-*`, `unit-*`.

Note: in hierarchical mode with separate inner loops, node ID uniqueness is less critical (domains have separate graphs). The validation is primarily for flattened mode where all nodes share one graph. The engine validates regardless of mode — consistent behavior.

**Alternatives:**
- Automatic namespacing — engine prefixes domain ID to all node IDs. Breaks the opaque-ID contract (provisioners expect unprefixed IDs).
- No validation — poor developer experience.
- Shared nodes by convention — two domains intentionally share node IDs with identical specs (e.g., spatial example's cell nodes). Supported — only conflicting specs trigger validation error, consistent with `overlay()` semantics.
**Rationale:** Convention-based uniqueness preserves the opaque-ID contract. Domain-prefixed IDs are natural. Startup validation catches collisions with helpful attribution.
**Trade-offs:** Convention is not compiler-enforced. Mitigated by startup validation with clear error messages.
**Sources:** ImmutableDesiredStateGraph.overlay() (line 252), DungeonGoalCompiler, PipelineGoalCompiler, spatial compilers
**Exploration:** surfaced-by-review
**Status:** kept — valid in both modes.
