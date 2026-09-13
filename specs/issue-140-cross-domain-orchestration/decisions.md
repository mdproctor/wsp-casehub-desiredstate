## D1: Architectural model — hierarchical from day one

**Choice:** Hierarchical architecture with good defaults so single-process deployments remain simple
**Alternatives:**
- Single-process merged-graph only — simpler but doesn't support multi-process or distributed deployments without rearchitecting
- Hierarchical added later — risks bolting on what should be foundational
**Rationale:** The hierarchical model (meta-loop with domain-level nodes, inner loops per domain) subsumes single-process and multi-process as deployment configurations. Good defaults (auto-discovery, automatic flattening for single-process) keep the consumer experience simple.
**Trade-offs:** More framework complexity upfront. The flattening optimisation must be designed correctly to avoid exposing hierarchical concepts to single-process consumers.
**Sources:** casehub-desiredstate#140, casehub-ops#23, ReconciliationLoop.java, LifecycleManager.java
**Exploration:** quick
**Status:** captured

## D2: Domain registration — DomainDescriptor pattern

**Choice:** Each domain provides a `DomainDescriptor` (non-parameterized) that bundles its compiler, goal loader, and metadata. The composition layer discovers descriptors via CDI.
**Alternatives:**
- Raw `GoalCompiler<Object>` erasure with separate `GoalProvider` SPI — splits registration across two SPIs, requires matching key, uses unchecked casts
**Rationale:** Type-safe and self-contained. Each domain is a single registration point. The descriptor hides the `GoalCompiler<G>` type parameter internally, solving the type erasure problem without unchecked casts.
**Trade-offs:** Domains must implement DomainDescriptor rather than just GoalCompiler — slightly more boilerplate per domain.
**Sources:** GoalCompiler.java, InfraGoalCompiler.java, DeploymentGoalCompiler.java, ApplicationGoalCompiler.java (the hand-coded workaround)
**Exploration:** quick
**Status:** captured

## D3: Steady-state definition — pluggable CompletionCondition

**Choice:** Reuse the existing `CompletionCondition` SPI. Each domain provides a condition via its `DomainDescriptor`. Default: all nodes PRESENT, zero active faults.
**Alternatives:**
- All nodes PRESENT, zero faults (hardcoded) — strictest, but a non-critical node failure blocks downstream domains indefinitely
- Sentinel nodes — domain nominates specific nodes as readiness markers; adds API surface to the compiler
**Rationale:** `CompletionCondition` already exists in the codebase (`Phase` record uses it). The default is strict (safe). Domains that need softer semantics override explicitly. No new concept introduced.
**Trade-offs:** Domains with mixed-criticality nodes must implement a custom condition or model criticality via finer NodeTypes.
**Sources:** CompletionCondition.java, Phase.java, LifecycleManager.java
**Exploration:** quick
**Status:** captured

## D4: Cross-domain dependency granularity — type-level

**Choice:** Type-level cross-domain dependencies. "Deployment roots depend on infra's namespace-type nodes." Expressed declaratively, runtime converts to graph edges (flattened mode) or type-scoped CompletionConditions (hierarchical mode).
**Alternatives:**
- Coarse (domain-level only) — simple but wasteful; domain B waits for ALL of domain A, even slow nodes it doesn't need (e.g. database clusters blocking deployment that only needs namespaces)
- Fine-grained (node-level) — most precise but creates naming coupling between independent domain compilers; fragile if a domain renames node IDs
**Rationale:** Type-level is the right abstraction boundary — more precise than domain-level (avoids unnecessary blocking), no naming coupling (domains don't reference each other's node IDs). If a domain needs finer distinction within a type, it should model that as finer NodeType values — the type system carries the semantic distinction, not the wiring layer.
**Trade-offs:** Less precise than node-level deps. A slow node of a depended-upon type blocks even if the dependent only needs a fast one. Mitigated by encouraging finer NodeType modelling.
**Sources:** DesiredStateGraph.overlay(), DesiredStateGraph.connect(), TransitionPlanner ordering, first-principles analysis of flattened vs hierarchical modes
**Exploration:** deep-analysis
**Status:** captured

## D5: Goal loading — configuration-driven

**Choice:** DomainDescriptor declares a configuration key/path. The composition layer passes a shared configuration source. Each descriptor extracts its domain-specific goals and compiles internally.
**Alternatives:**
- Programmatic — consumer constructs each domain's goals explicitly, keyed by domain ID. More control but requires the consumer to know every domain's goal type, defeating the descriptor abstraction.
**Rationale:** Keeps the consumer decoupled — one config document describes the whole system, domains self-serve from it. Aligns with YAML and annotation surfaces (declarative config → compiled graph).
**Trade-offs:** Domains must agree on a configuration source format. The shared config must be flexible enough for diverse domain goal structures.
**Sources:** InfraGoalCompiler.java, DeploymentGoalCompiler.java, YAML surface (YamlGraphRecorder), annotation surface
**Exploration:** quick
**Status:** captured

## D6: Flattening strategy — transparent for single-process

**Choice:** Composition layer detects single-process deployment (all descriptors in one JVM), merges domain graphs into one via overlay(), adds type-based cross-domain edges, feeds merged graph to a single ReconciliationLoop. Meta-loop doesn't run. Consumer never sees domain-level nodes.
**Alternatives:**
- Always hierarchical — even in single-process, meta-loop runs with domain-level nodes and inner loops. Consistent mental model but adds overhead (multiple loop instances, domain-level node synthesis) when a merged graph suffices.
**Rationale:** Simpler for the common case. Single-process ops deployments get the same performance and debugging experience as today's single-domain case. Hierarchical machinery only activates when genuinely needed.
**Trade-offs:** Flattened path is a distinct code path that needs separate testing. Behaviour differences between modes must be documented.
**Depends on:** D1 (hierarchical architecture), D4 (type-level deps manifest as edges in flattened mode)
**Sources:** DesiredStateGraph.overlay(), TransitionPlanner, ReconciliationLoop
**Exploration:** quick
**Status:** captured

## D7: CDI discovery — automatic

**Choice:** CrossDomainCompositionEngine is @ApplicationScoped, injects Instance<DomainDescriptor>, wires at startup. One descriptor = single-domain passthrough (no change). Multiple descriptors = auto-composition. Zero config needed.
**Alternatives:**
- Explicit registration — consumer creates composition bean manually, listing descriptors and ordering. More control but requires boilerplate in every multi-domain app.
**Rationale:** Preserves "good defaults" from D1. Just adding a second domain JAR to the classpath activates cross-domain composition. Single-descriptor path is a no-op passthrough — no behavioral change for existing single-domain deployments.
**Trade-offs:** Less control over composition order. Mitigated by provides/requires declarations (D8) which make ordering declarative and automatic.
**Depends on:** D1 (good defaults), D2 (DomainDescriptor)
**Sources:** CdiNodeProvisionerRouter, CdiActualStateAdapterRouter, CdiMergedEventSource (existing CDI compositor pattern)
**Exploration:** quick
**Status:** captured

## D8: Cross-domain dependency declaration — provides/requires

**Choice:** Each DomainDescriptor declares provides: Set<NodeType> (types this domain creates) and requires: Set<NodeType> (types needed before starting). Composition layer matches requires→provides automatically. Domains reference NodeTypes, not other domains.
**Alternatives:**
- In dependent domain's descriptor only — descriptor references another domain's node types by name. Soft compile-time awareness, but workable.
- Separate orchestration config — cross-domain manifest (YAML/annotation). Clean separation but another artifact to maintain.
**Rationale:** Fully decoupled — domains don't reference each other by name, only by abstract NodeType. Wiring is automatic (requires/provides matching). Misconfiguration caught at startup (unsatisfied requires → fail fast). Naturally scopes the CompletionCondition from D3.
**Trade-offs:** Requires that cross-domain dependencies are expressible via NodeType. Domains with untyped or single-typed nodes may need to introduce finer types to express partial dependencies.
**Depends on:** D4 (type-level granularity)
**Sources:** NodeType value type, NodeProvisionerRouter.handledTypes() pattern, first-principles analysis
**Exploration:** quick
**Status:** captured

## D9: Module placement — api/ for SPI, runtime/ for engine

**Choice:** DomainDescriptor SPI in api/, CrossDomainCompositionEngine in runtime/ (dedicated package: io.casehub.desiredstate.runtime.composition).
**Alternatives:**
- New orchestration/ module — cleaner layering signal, but: api/ already contains multi-domain primitives (routers, overlay/connect), runtime/ already contains CDI compositors (FaultPolicyEngine, SituationRecompilerEngine, routers, MergedEventSource). A new module breaks "just add JARs" experience and the composition engine is tightly coupled to runtime APIs so independent versioning is illusory.
**Rationale:** api/ already hosts multi-domain SPIs (ActualStateAdapterRouter, NodeProvisionerRouter). runtime/ already hosts their CDI compositors. DomainDescriptor + composition engine follow the same pattern. No new module means no extra dependency for multi-domain apps — consistent with D1 (good defaults) and D7 (automatic CDI discovery).
**Trade-offs:** runtime/ grows. Mitigated by dedicated package for composition code.
**Sources:** api/ module (existing routers), runtime/ module (existing compositors), first-principles analysis of module boundary criteria
**Exploration:** deep-analysis
**Status:** captured
