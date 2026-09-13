## D1: Architectural model — flat composition with single merged graph

**Choice:** Single-process flat composition. Multiple domain graphs merged via `overlay()` into one `DesiredStateGraph` with cross-domain edges derived from provides/requires declarations. Single `ReconciliationLoop` per tenant reconciles the merged graph. No hierarchical meta-loop.
**Alternatives:**
- Hierarchical from day one — meta-loop with domain-level nodes, inner loops per domain. Subsumes single-process and multi-process but adds permanent dual-path maintenance. No concrete consumer for multi-process deployment exists.
- CompositionStrategy SPI — flat by default, hierarchy as an alternative strategy via CDI displacement. Clean extension point but premature: an SPI with one implementation is an abstraction without a consumer.
**Rationale:** Flat composition meets the known need (casehub-ops single-process infra→deployment→compliance→iot ordering). `overlay()` and `connect()` already exist. Cross-domain ordering is structural — type-based edges in the merged graph enforce sequencing via the existing `TransitionPlanner`. One code path to maintain and test. If multi-process hierarchy is needed in the future, it can be introduced as a separate composition engine or extension — the flat model's overlay-based merging doesn't preclude this.
**Trade-offs:** Cannot support multi-process domain separation without architectural extension. Acceptable: no consumer exists for this capability, and designing for it now would double the test surface (R1-07 critique of two code paths).
**Depends on:** D4 (type-level deps manifest as edges in merged graph), D8 (provides/requires declarations drive edge creation)
**Sources:** casehub-desiredstate#140, casehub-ops#23, DesiredStateGraph.overlay(), TransitionPlanner, ReconciliationLoop.java, ForceDistributionTest.java (existing overlay usage)
**Exploration:** quick
**Status:** revised — was "hierarchical from day one"; revised to flat-only after R1-02 (YAGNI — no multi-process consumer) and R1-07 (dual code path maintenance cost)

## D2: Domain registration — push model

**Choice:** Each domain compiles its own goals (type-safe, domain's concern) and registers the compiled `CompilationResult` plus metadata (`provides: Set<NodeType>`, `requires: Set<NodeType>`) with the `CrossDomainCompositionEngine` at startup. Registration via CDI startup observers — each domain JAR includes a `@ApplicationScoped` bean that `@Observes StartupEvent` and calls `engine.registerDomain()`. No DomainDescriptor SPI.
**Alternatives:**
- DomainDescriptor pull model — composition engine discovers descriptors via CDI, calls `descriptor.compile()`. Introduces the `GoalCompiler<G>` type erasure problem: the composition layer must call `compile()` without knowing `G`. DomainDescriptor bundles four concerns (compiler, goal loader, metadata, completion condition) in one SPI — coupling that isn't necessary when domains drive their own compilation.
- Raw `GoalCompiler<Object>` erasure with separate `GoalProvider` SPI — splits registration across two SPIs, requires matching key, uses unchecked casts. Worst alternative.
**Rationale:** The type erasure problem exists only when the composition layer calls `compile()`. With push, domains compile themselves — `GoalCompiler<AttackBlueprint>`, `GoalCompiler<DefenseBlueprint>` etc. are called by domain code that knows the type parameter. The composition engine receives `CompilationResult` (no generic parameter). The spatial example already demonstrates this pattern: `AttackGoalCompiler`, `DefenseGoalCompiler`, `DistributionGoalCompiler` each compile independently with their own blueprint type, and results merge via `overlay()`. Registration via `@Observes StartupEvent` follows the platform's established CDI startup pattern.
**Trade-offs:** The composition engine cannot trigger initial compilation — domains must compile at startup and register. For recompilation, see D10 (SituationRecompiler interaction). Registration ordering relies on CDI `@Priority` or lifecycle phasing.
**Sources:** GoalCompiler.java, AttackGoalCompiler.java, DefenseGoalCompiler.java, DistributionGoalCompiler.java, ForceDistributionTest.java (overlay pattern), CdiNodeProvisionerRouter (CDI collection pattern)
**Exploration:** quick
**Status:** revised — was "DomainDescriptor pattern"; revised to push model after R1-03 (type erasure only exists if composition layer calls compile; push eliminates DomainDescriptor entirely)

## D3: Steady-state definition — CompletionCondition for lifecycle phases only

**Choice:** `CompletionCondition` retains its existing role — lifecycle phase transitions evaluated by `LifecycleManager`. Cross-domain ordering in the flat model is handled by type-based edges in the merged graph, not by CompletionCondition. No cross-domain "readiness" concept needed.
**Alternatives:**
- Separate ReadinessCondition SPI for cross-domain readiness, naturally scoped by provides/requires types. Unnecessary in the flat model: graph edges handle ordering structurally. Would be needed if hierarchical model were adopted (deferred — see D1).
- CompletionCondition overloaded for both lifecycle phases and cross-domain readiness — conflates two concepts with different semantics (a domain can be "ready" for downstream while not yet "complete")
**Rationale:** In the flat model, dependency ordering is structural. If domain B's root nodes depend on domain A's NAMESPACE-type nodes via cross-domain edges, B's nodes are not plannable by `TransitionPlanner` until A's NAMESPACE nodes are PRESENT. No separate readiness check is needed — the graph's dependency edges enforce it. `CompletionCondition` stays scoped to its current role: "has this lifecycle phase reached its terminal state?"
**Trade-offs:** If hierarchical orchestration is added later, a ReadinessCondition concept will be needed to represent "domain-level readiness" distinct from phase completion. This is a future cost accepted by the flat-only architecture (D1).
**Sources:** CompletionCondition.java, Phase.java, LifecycleManager.java, TransitionPlanner.java
**Exploration:** quick
**Status:** revised — was "reuse CompletionCondition for cross-domain readiness"; revised to lifecycle-only scope after R1-04 (CompletionCondition overloading conflates two distinct concepts) and D1 revision (flat model handles ordering via graph edges)

## D4: Cross-domain dependency granularity — type-level

**Choice:** Type-level cross-domain dependencies. "Deployment roots depend on infra's namespace-type nodes." Expressed declaratively via provides/requires, runtime converts to graph edges in the merged graph.
**Alternatives:**
- Coarse (domain-level only) — simple but wasteful; domain B waits for ALL of domain A, even slow nodes it doesn't need (e.g. database clusters blocking deployment that only needs namespaces)
- Fine-grained (node-level) — most precise but creates naming coupling between independent domain compilers; fragile if a domain renames node IDs
- Type-level with predicate refinement — "deployment depends on infra's NAMESPACE nodes matching `namespace.name == deployment.targetNamespace`". Adds structural selectivity but couples the wiring layer to node spec internals. Types are the API boundary; content is opaque.
**Rationale:** Type-level is the right abstraction boundary — more precise than domain-level (avoids unnecessary blocking), no naming coupling (domains don't reference each other's node IDs). If a domain has semantically distinct resources with different provisioning timelines, they SHOULD be distinct NodeType values — NAMESPACE and DATABASE_CLUSTER are semantically different, not just "fast" and "slow". This reflects domain semantics, not orchestration concerns. Predicate refinement can be added later if the type-level model proves insufficient, but adds coupling (wiring layer must understand node spec structure) that should be avoided without evidence of need.
**Trade-offs:** Less precise than node-level deps. A slow node of a depended-upon type blocks even if the dependent only needs a fast one. Mitigated by finer NodeType modelling — which reflects semantic distinctions in the domain, not orchestration speed.
**Sources:** DesiredStateGraph.overlay(), DesiredStateGraph.connect(), TransitionPlanner ordering, NodeType value type, first-principles analysis of flattened vs hierarchical modes
**Exploration:** deep-analysis
**Status:** captured

## D5: Goal loading — domain responsibility (superseded)

**Choice:** Goal loading is the domain's responsibility. Each domain loads its own configuration, compiles using its own `GoalCompiler<G>`, and registers the compiled `CompilationResult` with the composition engine. The composition layer never sees domain-specific configuration or goal types.
**Alternatives:**
- Configuration-driven via DomainDescriptor — composition layer passes shared config source, each descriptor extracts domain-specific goals. Requires D2's DomainDescriptor model, which is superseded.
- Programmatic — consumer constructs each domain's goals explicitly. This is the current working pattern.
**Rationale:** Superseded by D2 revision (push model). With push, domains compile themselves — goal loading is inherently the domain's concern. The composition engine receives `CompilationResult`, not goals. No shared configuration contract needed. Each domain can use whatever configuration mechanism suits it (YAML, annotations, programmatic, Preferences).
**Trade-offs:** None beyond D2's trade-offs. Domain autonomy over configuration is a feature, not a trade-off.
**Sources:** InfraGoalCompiler.java, DeploymentGoalCompiler.java, YAML surface (YamlGraphRecorder), annotation surface
**Exploration:** quick
**Status:** revised — was "configuration-driven via DomainDescriptor"; superseded by D2 push model revision (R1-06)

## D6: Composition strategy — flat-only, single code path

**Choice:** One composition strategy: merge domain graphs via `overlay()`, add type-based cross-domain edges derived from provides/requires, feed the merged graph to a single `ReconciliationLoop`. No conditional flattening, no hierarchical mode, no mode detection.
**Alternatives:**
- Transparent flattening — composition layer detects single-process and automatically flattens. Creates two code paths (flat and hierarchical) with different fault propagation, CAS behavior, and SituationRecompiler interactions. Permanent maintenance multiplier.
- Always hierarchical — even in single-process, meta-loop runs with domain-level nodes and inner loops. Consistent but adds overhead (multiple loop instances, domain-level node synthesis) for the only deployment model that currently exists.
**Rationale:** Superseded by D1 revision (flat-only architecture). One code path, one test surface, one mental model. The overlay()-based merging is already validated (spatial example, ImmutableDesiredStateGraphTest). Cross-domain edges extend the graph's existing dependency infrastructure — `TransitionPlanner` respects them like any other dependency.
**Trade-offs:** Cannot support hierarchical reconciliation. Accepted: no consumer exists (D1 rationale).
**Depends on:** D1 (flat-only architecture), D4 (type-level deps as edges)
**Sources:** DesiredStateGraph.overlay(), TransitionPlanner, ReconciliationLoop
**Exploration:** quick
**Status:** revised — was "transparent flattening for single-process"; revised to flat-only single code path after D1 revision eliminates the need for mode switching (R1-07)

## D7: CDI discovery — automatic via startup registration

**Choice:** `CrossDomainCompositionEngine` is `@ApplicationScoped`. Domains register via CDI startup observers (`@Observes StartupEvent` with `@Priority`). The engine auto-composes after all registrations. One registration = single-domain passthrough (engine passes `CompilationResult` directly to `LifecycleManager`, no merging). Multiple registrations = auto-composition via `overlay()` + cross-domain edges.
**Alternatives:**
- Explicit registration — consumer creates composition bean manually, listing domains and ordering. More control but requires boilerplate in every multi-domain app.
- Instance<DomainDescriptor> injection — CDI auto-discovery of descriptor beans. Requires D2's DomainDescriptor SPI, which is superseded.
**Rationale:** Preserves auto-activation from D1's original intent. Adding a second domain JAR to the classpath activates composition automatically — each JAR's startup observer registers with the engine. Single-domain passthrough means existing single-domain deployments have no behavioral change in reconciliation outcomes.
**Trade-offs:** Entry-point API does change — domains now register with the composition engine rather than calling LifecycleManager directly. This is an API migration, not a behavioral change. Registration ordering depends on CDI `@Priority` or lifecycle phasing. Mitigated by the provides/requires declarations (D8) which make ordering declarative.
**Depends on:** D2 (push model registration), D8 (provides/requires for ordering)
**Sources:** CdiNodeProvisionerRouter, CdiActualStateAdapterRouter, CdiMergedEventSource (existing CDI compositor pattern)
**Exploration:** quick
**Status:** revised — was "Instance<DomainDescriptor> injection"; revised for push model. Narrowed "no behavioral change" claim per R1-08: reconciliation outcomes unchanged, but entry-point API changes.

## D8: Cross-domain dependency declaration — provides/requires with validation

**Choice:** Each domain declares `provides: Set<NodeType>` (types this domain creates) and `requires: Set<NodeType>` (types needed before starting) at registration. Composition engine matches requires→provides automatically. Domains reference NodeTypes, not other domains. Explicit validation at startup:
1. **Duplicate provides** — two domains claiming the same NodeType → fail fast. Consistent with `NodeProvisionerRouter` which already fails fast on overlapping `handledTypes()`.
2. **Circular requires** — topological sort of domain ordering at startup; cycle detected → fail fast with descriptive error.
3. **Partial readiness** — if domain B requires NodeType X from domain A's provides {X, Y, Z}, cross-domain edges connect B's root nodes to A's X-type nodes only. B's nodes become plannable as soon as A's X nodes are PRESENT, regardless of A's Y and Z nodes.
**Alternatives:**
- In dependent domain's descriptor only — descriptor references another domain's node types by name. Soft compile-time awareness, but workable.
- Separate orchestration config — cross-domain manifest (YAML/annotation). Clean separation but another artifact to maintain.
**Rationale:** Fully decoupled — domains don't reference each other by name, only by abstract NodeType. Wiring is automatic (requires/provides matching). Misconfiguration caught at startup. Type-scoped edge creation (point 3) provides fine-grained ordering: downstream domains start consuming each provided type as soon as it's available, not after the entire upstream domain completes. This interacts naturally with D4's type-level granularity.
**Trade-offs:** Requires that cross-domain dependencies are expressible via NodeType. Domains with untyped or single-typed nodes may need to introduce finer types to express partial dependencies.
**Depends on:** D4 (type-level granularity)
**Sources:** NodeType value type, NodeProvisionerRouter.handledTypes() pattern, first-principles analysis
**Exploration:** quick
**Status:** revised — was missing validation details; added duplicate-provides fail-fast, cycle detection, and partial readiness semantics per R1-09

## D9: Module placement — composition engine in runtime/

**Choice:** `CrossDomainCompositionEngine` in runtime/ (dedicated package: `io.casehub.desiredstate.runtime.composition`). No new SPI in api/ — with the push model (D2), there is no DomainDescriptor interface to place. The composition engine exposes a runtime registration API (`registerDomain()`), not an SPI.
**Alternatives:**
- DomainDescriptor SPI in api/, engine in runtime/ — original design. Superseded by D2 push model revision: no SPI needed.
- New orchestration/ module — unnecessary. The composition engine is tightly coupled to runtime APIs (`ReconciliationLoop`, `LifecycleManager`, `TransitionPlanner`). A separate module would not enable independent versioning.
**Rationale:** runtime/ already hosts CDI compositors (`FaultPolicyEngine`, `SituationRecompilerEngine`, routers, `MergedEventSource`). The composition engine follows the same pattern. Dedicated package keeps composition code isolated within runtime/. No new module means no extra dependency for multi-domain apps — consistent with D7 (automatic CDI discovery).
**Trade-offs:** runtime/ grows. Mitigated by dedicated package for composition code.
**Sources:** api/ module (existing routers), runtime/ module (existing compositors), module-tier-structure protocol
**Exploration:** deep-analysis
**Status:** revised — updated for D2 push model: no DomainDescriptor SPI in api/; composition engine's registerDomain() is a runtime API

## D10: LifecycleManager interaction — composition engine wraps with SituationRecompiler integration

**Choice:** Composition engine is the top-level entry point. It wraps `LifecycleManager` — same layering (composition above lifecycle above reconciliation), but with explicit SituationRecompiler integration:

1. **Initial composition:** Engine merges all registered domain graphs (overlay + cross-domain edges) and calls `LifecycleManager.start(tenancyId, composedResult)`.
2. **SituationRecompiler flow:** `SituationRecompiler` returns `CompilationResult` scoped to one domain. The composition engine intercepts: replaces that domain's contribution, re-merges with other domains' current graphs + cross-domain edges, calls `LifecycleManager.updateDesired(tenancyId, newComposedResult)`.
3. **Cascade detection:** If a domain's recompilation removes a NodeType from its provides set, the engine detects that downstream domains' requires are now unsatisfied. Initial behavior: fail fast with descriptive error. Cascade recompilation is a future evolution.
4. **Per-domain lifecycle tracking:** See D13.

**Alternatives:**
- Replace LifecycleManager — composition engine subsumes phase transition logic. Simpler call stack but conflates domain ordering and phase transitions, requires reimplementing phase CAS logic that already works.
- LifecycleManager directly receives SituationRecompiler results — stale composition engine view; engine and LifecycleManager fight over desired state.
**Rationale:** LifecycleManager's CAS-based phase transitions work well and shouldn't change. The composition engine adds domain orchestration above it. SituationRecompiler results must flow through the composition engine so it can re-compose — otherwise the composition engine's view of the merged graph diverges from LifecycleManager's.
**Trade-offs:** The composition engine must track per-domain CompilationResult to support re-composition. SituationRecompilers need domain scoping (which domain does a recompiled result belong to). This is a new requirement on the SituationRecompiler SPI — it may need a `domainId()` method or the composition engine needs to match recompilers to domains via registration.
**Depends on:** D1 (flat architecture), D2 (push model), D13 (per-domain lifecycle state)
**Sources:** LifecycleManager.java (CAS phase transitions), ReconciliationLoop.java, SituationRecompilerEngine.java
**Exploration:** quick
**Status:** revised — was "two layers, each with a single concern" without addressing SituationRecompiler interaction; revised to explicitly design the recompilation flow per R1-11

## D11: Tenancy model — same tenant for composed domains

**Choice:** Composed domains share the same `tenancyId`. The composition engine produces one merged graph per tenant. All domain nodes within a composition belong to the same tenant.
**Alternatives:**
- Per-domain tenant mapping — each domain has its own tenancyId. Would require cross-tenant edges and cross-tenant reconciliation coordination. Significantly more complex with no known consumer.
- Shared resource pool — a domain like infra provisions resources shared across tenants. This is a separate ReconciliationLoop, not part of multi-domain composition for any single tenant.
**Rationale:** `ReconciliationLoop` is per-tenant. The merged graph is per-tenant. Cross-domain dependencies (infra namespaces → deployments) are within the same tenant. If a domain provisions shared infrastructure across tenants, it operates as an independent reconciliation loop, not as a composed domain within a tenant's graph.
**Trade-offs:** Cannot model cross-tenant dependencies within the composition engine. Acceptable: cross-tenant coordination is an orchestration concern above the desired-state runtime.
**Sources:** ReconciliationLoop.java (per-tenant TenantLoop), ProvisionContext (carries tenancyId)
**Exploration:** surfaced-by-review
**Status:** captured

## D12: Fault propagation — via existing merged graph infrastructure

**Choice:** In the flat model, faults propagate through the existing infrastructure on the merged graph. A node in domain A fails → `FaultPolicyEngine` evaluates → mutations applied to the merged graph. Domain B's nodes are in the same graph — cross-domain edges carry dependency semantics, and `TransitionPlanner`/`FaultPolicyEngine`/`ReconciliationLoop` operate on the unified graph without needing cross-domain mediation.
**Alternatives:**
- Faults propagate through composed graph edges only (flat mode) — this IS the chosen approach.
- CloudEvents from domain A consumed by domain B's fault policies — adds coupling between domain fault policies and requires domain B to understand domain A's fault semantics.
- Composition engine mediates fault signals — adds a new fault propagation layer with different semantics from the existing per-graph FaultPolicyEngine.
**Rationale:** The flat model's chief advantage: cross-domain faults are just faults in a single graph. The existing `FaultPolicyEngine` evaluates all fault policies against the merged graph. Policies from different domains naturally compose (they handle different NodeType/FaultType combinations, just as they do today). No new fault propagation mechanism needed.
**Trade-offs:** A fault policy from domain A could mutate nodes from domain B if it has visibility into B's node types. This is a feature (cross-domain fault responses) but requires domain authors to be aware that the graph is composed. Mitigated by NodeType-scoped fault policies — existing convention is to scope policies by the node types they handle.
**Sources:** FaultPolicyEngine.java, ReconciliationLoop.reconcile() (drift detection + fault feedback), ThresholdFaultPolicy
**Exploration:** surfaced-by-review
**Status:** captured

## D13: Per-domain lifecycle state — composition engine manages internally

**Choice:** The composition engine manages per-domain lifecycle state. Each domain registers a `CompilationResult` — either `SingleGraph` or `Lifecycle(List<Phase>)`. The engine tracks which phase each domain is at. The composed graph at any moment is the overlay of each domain's current-phase graph plus cross-domain edges.

When a domain's phase completes (its `CompletionCondition` is satisfied for its nodes in the actual state), the engine:
1. Advances that domain to its next phase
2. Re-composes: overlay all domains' current-phase graphs + cross-domain edges
3. Calls `LifecycleManager.updateDesired()` with the new composed graph

**Alternatives:**
- Single composed Lifecycle — combinatorial explosion: A:2phases × B:3phases = 6 composed phases. Each composed phase would need its own CompletionCondition. Unworkable beyond two domains.
- LifecycleManager manages per-domain phases — would require LifecycleManager to understand domain composition, breaking its single-responsibility as a phase transition orchestrator.
**Rationale:** The composition engine is already tracking per-domain contributions (D10). Extending it to track per-domain phase state is natural. `LifecycleManager` continues to manage the single composed graph's lifecycle — it doesn't need to know about domains. The composition engine evaluates per-domain `CompletionCondition`s via a `ReconciliationListener` that filters actual state to each domain's current-phase node types.
**Trade-offs:** The composition engine grows in responsibility: domain registration, graph merging, cross-domain edges, per-domain lifecycle tracking, and re-composition on phase transitions. This is the cost of flat composition — one component manages the composed view. Mitigated by clear internal separation (dedicated package, distinct methods for each concern).
**Sources:** LifecycleManager.java, CompilationResult.Lifecycle, Phase.java, CompletionCondition.java, ReconciliationListener.java
**Exploration:** surfaced-by-review
**Status:** captured

## D14: Graph versioning — single composed graph, single CAS

**Choice:** In the flat model, the composed graph is a single `DesiredStateGraph` with its own version counter. All CAS operations (`compareAndSetDesired()`) operate on this single graph. Per-domain "subgraphs" don't have independent versions — they are merged into one graph at composition time.

When the composition engine re-composes (due to domain recompilation, phase transition, or SituationRecompiler), it:
1. Builds a new composed graph (overlay + edges)
2. Calls `LifecycleManager.updateDesired()` or `compareAndSetDesired()` with the new graph
3. The new graph gets a new version from `ImmutableDesiredStateGraph` construction

**Alternatives:**
- Dual versioning — composed graph version + per-domain subgraph versions. Requires reconciling two version spaces when CAS conflicts arise. Complex and error-prone.
- Per-domain CAS — each domain's subgraph has its own CAS reference. The composition engine must coordinate multiple CAS operations atomically. Requires distributed locking or a transaction protocol.
**Rationale:** The flat model's key simplification: one graph, one version, one CAS. The composition engine is the single writer to `LifecycleManager`/`ReconciliationLoop` — no concurrent domain-level CAS operations. `SituationRecompiler` results flow through the composition engine (D10), which serializes re-composition. The existing CAS semantics in `ReconciliationLoop` are unchanged.
**Trade-offs:** Concurrent SituationRecompiler triggers for different domains must be serialized through the composition engine. This is acceptable: situation-triggered recompilation is infrequent, and serialization through a single composition engine avoids split-brain scenarios.
**Sources:** ImmutableDesiredStateGraph (version counter), ReconciliationLoop.compareAndSetDesired(), LifecycleManager.java
**Exploration:** surfaced-by-review
**Status:** captured
