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
