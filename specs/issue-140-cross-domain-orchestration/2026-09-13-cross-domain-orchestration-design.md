# Cross-Domain Orchestration Framework

**Issue:** casehub-desiredstate#140
**Date:** 2026-09-13
**Status:** Draft

## 1. Problem

The desired-state runtime handles single-domain reconciliation well. Each domain module provides one `GoalCompiler<G>`, one `ActualStateAdapter`, one `NodeProvisioner`, one `FaultPolicy`, and one `EventSource` — all `@ApplicationScoped`. The routers (`NodeProvisionerRouter`, `ActualStateAdapterRouter`), `MergedEventSource`, and `FaultPolicyEngine` already dispatch across multiple implementations by `NodeType`, making the runtime multi-domain-ready for everything except compilation and orchestration.

Three gaps remain:

**1. GoalCompiler composition.** `GoalCompiler<G>` is parameterized by goal type. `InfraGoalCompiler` takes `InfraGoals`, `DeploymentGoalCompiler` takes `DeploymentGoals`. CDI cannot discover these as `Instance<GoalCompiler<?>>` due to type erasure. There is no composition layer that calls multiple compilers and merges their graphs. The ops `ApplicationGoalCompiler` works around this by hand-coding a single compiler that only handles infra resources — it doesn't implement `GoalCompiler<G>` at all and bypasses `LifecycleManager` entirely.

**2. Cross-domain dependency ordering.** `DesiredStateGraph` has `overlay()` and `connect()` for graph merging, and `TransitionPlanner` respects `Dependency` edges across node types. But nobody orchestrates these across domains. There is no way to express "don't provision deployment agents until infra namespaces exist."

**3. Cross-domain lifecycle management.** Each domain may return `CompilationResult.Lifecycle` with multiple phases. `LifecycleManager` handles phases for a single graph. When multiple domains each have phases, the composed graph must evolve as individual domains transition — a combinatorial problem that `LifecycleManager` cannot solve alone.

**First consumer:** casehub-ops (casehubio/casehub-ops#23) — wiring infra → deployment → compliance → iot ordering across four domain modules that share one classpath.

## 2. Design Overview

The design introduces a `CrossDomainCompositionEngine` that sits above `LifecycleManager` in the call stack. It discovers domain contributions via a **push model** — each domain compiles its own goals (preserving type safety) and registers the compiled `CompilationResult` plus metadata with the engine at startup.

**Architecture:** Hierarchical from day one. The conceptual model is a meta-loop with domain-level nodes and inner reconciliation loops per domain. This architecture subsumes three deployment configurations:

| Mode | Activation | Behavior |
|------|-----------|----------|
| **Flattened** (default) | `desiredstate.composition.mode=flattened` | All domain graphs merged via `overlay()` into one `DesiredStateGraph` with cross-domain edges. Single `ReconciliationLoop`. No meta-loop. |
| **Hierarchical** | `desiredstate.composition.mode=hierarchical` | Meta-loop with domain-level nodes. Inner `ReconciliationLoop` per domain. `CompletionCondition` determines domain readiness. |
| **Single-domain** | One registration (or zero) | Passthrough — `CompilationResult` goes directly to `LifecycleManager`. No composition overhead. |

The flattened mode is the default because it reuses existing primitives (`overlay()`, `TransitionPlanner`, single `ReconciliationLoop`) and delivers the same performance and debugging experience as single-domain. Hierarchical mode activates when the consumer explicitly opts in via configuration.

**Push model rationale:** The `GoalCompiler<G>` type erasure problem exists only when the composition layer calls `compile()`. With push, domains compile themselves — `GoalCompiler<InfraGoals>`, `GoalCompiler<DeploymentGoals>` etc. are called by domain code that knows the type parameter. The composition engine receives `CompilationResult` (no generic parameter). The spatial example validates the underlying `overlay()` primitive: `ForceDistributionTest.zoneSplitStructuralChange()` merges graphs from independent compilations of the same compiler type via `overlay()`. Cross-compiler overlay (different compilers with different node types) is a new composition pattern that requires dedicated test coverage (§10).

**Mode immutability:** The composition mode (`flattened` or `hierarchical`) is determined at startup and cannot be changed at runtime. Switching modes would require tearing down and rebuilding the entire reconciliation infrastructure — inner loops, meta-graph, domain state tracking. The mode is read once during the composition engine's startup observer and is immutable thereafter.

## 3. New Types

### 3.1 DomainId (api/)

Value type for domain identity. Placed in api/ alongside `NodeId` and `NodeType`.

```java
package io.casehub.desiredstate.api;

public record DomainId(String value) {
    public DomainId {
        java.util.Objects.requireNonNull(value, "DomainId value must not be null");
        if (value.isBlank()) throw new IllegalArgumentException("DomainId must not be blank");
    }

    public static DomainId of(String value) { return new DomainId(value); }
}
```

### 3.2 DomainRegistration (runtime/)

Registration record passed by domains to the composition engine. Lives in `io.casehub.desiredstate.runtime.composition`.

```java
package io.casehub.desiredstate.runtime.composition;

import io.casehub.desiredstate.api.*;

import java.util.List;
import java.util.Set;

public record DomainRegistration(
    DomainId domainId,
    CompilationResult compilationResult,
    Set<NodeType> provides,
    Set<NodeType> requires,
    CompletionCondition readinessCondition,
    List<SituationRecompiler> situationRecompilers
) {
    public DomainRegistration {
        java.util.Objects.requireNonNull(domainId);
        java.util.Objects.requireNonNull(compilationResult);
        java.util.Objects.requireNonNull(readinessCondition);
        provides = Set.copyOf(provides);
        requires = Set.copyOf(requires);
        situationRecompilers = situationRecompilers != null
            ? List.copyOf(situationRecompilers) : List.of();
    }

    public static Builder builder(DomainId domainId, CompilationResult result) {
        return new Builder(domainId, result);
    }

    public static class Builder {
        private final DomainId domainId;
        private final CompilationResult compilationResult;
        private Set<NodeType> provides = Set.of();
        private Set<NodeType> requires = Set.of();
        private CompletionCondition readinessCondition = CompletionCondition.allPresent();
        private List<SituationRecompiler> situationRecompilers = List.of();

        private Builder(DomainId domainId, CompilationResult compilationResult) {
            this.domainId = domainId;
            this.compilationResult = compilationResult;
        }

        public Builder provides(Set<NodeType> provides) {
            this.provides = provides; return this;
        }

        public Builder requires(Set<NodeType> requires) {
            this.requires = requires; return this;
        }

        public Builder readinessCondition(CompletionCondition condition) {
            this.readinessCondition = condition; return this;
        }

        public Builder situationRecompilers(List<SituationRecompiler> recompilers) {
            this.situationRecompilers = recompilers; return this;
        }

        public DomainRegistration build() {
            return new DomainRegistration(domainId, compilationResult,
                provides, requires, readinessCondition, situationRecompilers);
        }
    }
}
```

**Default readiness condition:** The Builder defaults `readinessCondition` to `CompletionCondition.allPresent()`. The compact constructor enforces non-null — every `DomainRegistration` has an explicit condition, eliminating null checks in the engine and all downstream consumers. Domains that need softer semantics (e.g., tolerating a degraded monitoring sidecar) provide an explicit condition via the Builder.

**Invariant:** Any node with an active fault has a non-PRESENT status in ActualState. `PROVISION_FAILED` leaves the node `ABSENT`; `NODE_DEGRADED` and `NODE_DRIFTED` set `DRIFTED`. `allPresent()` implicitly enforces zero faults through this invariant — it checks `actual.statuses()`, not `ReconciliationLoop.activeProblems`. These are different tracking mechanisms with different semantics: `statuses()` is the ground truth from `ActualStateAdapter`, while `activeProblems` is the reconciliation loop's internal fault tracking set.

## 4. CrossDomainCompositionEngine

Package: `io.casehub.desiredstate.runtime.composition`

```java
@ApplicationScoped
public class CrossDomainCompositionEngine implements GlobalReconciliationListener {

    private final LifecycleManager lifecycleManager;
    private final ReconciliationLoop reconciliationLoop;
    private final DesiredStateGraphFactory graphFactory;
    private final String mode; // "flattened" or "hierarchical"
    private volatile boolean composed = false;

    // --- Domain configuration (global, immutable after compose()) ---

    // Registration metadata — provides, requires, original CompilationResult
    private final Map<DomainId, DomainRegistration> domainConfigs = new LinkedHashMap<>();

    // Recompiler → domain reverse index. IdentityHashMap because
    // SituationRecompiler identity is instance-based.
    private final Map<SituationRecompiler, DomainId> recompilerIndex = new IdentityHashMap<>();

    // Priority-sorted recompiler list — preserves chain-of-responsibility
    // semantics from SituationRecompilerEngine. Built at compose() from
    // all domains' situationRecompilers, sorted by priority() ascending.
    private List<Map.Entry<SituationRecompiler, DomainId>> sortedRecompilers;

    // Cross-domain recompilers — priority-sorted, tried after domain-specific
    private List<SituationRecompiler> sortedCrossDomainRecompilers;

    // Provides/requires topology — immutable after compose()
    private List<DomainId> topologicalOrder;

    // --- Per-tenant runtime state ---

    // Phase tracking per tenant, per domain. Follows the per-tenant keying
    // pattern of ReconciliationLoop (ConcurrentHashMap<String, TenantLoop>)
    // and LifecycleManager (ConcurrentHashMap<String, TenantLifecycle>).
    private final ConcurrentHashMap<String, TenantCompositionState> tenantStates
        = new ConcurrentHashMap<>();

    // Serializes recompose paths — see Thread Safety Model below
    private final Object recomposeLock = new Object();

    public void registerDomain(DomainRegistration registration) { ... }
    void compose(String tenancyId) { ... }

    // GlobalReconciliationListener — per-domain phase advancement
    @Override
    public void onReconciliationCycleCompleted(
        String tenancyId, DesiredStateGraph desired, ActualState actual) { ... }
}
```

**`TenantCompositionState`** holds per-tenant per-domain phase tracking:

```java
record DomainPhaseState(CompilationResult currentResult, int phaseIndex) {
    DesiredStateGraph currentGraph() {
        return switch (currentResult) {
            case CompilationResult.SingleGraph sg -> sg.graph();
            case CompilationResult.Lifecycle lc -> lc.phases().get(phaseIndex).graph();
        };
    }
    boolean hasLifecycle() { return currentResult instanceof CompilationResult.Lifecycle; }
    boolean isAtFinalPhase() {
        return !(currentResult instanceof CompilationResult.Lifecycle lc)
            || phaseIndex >= lc.phases().size() - 1;
    }
    DomainPhaseState withAdvancedPhase() { return new DomainPhaseState(currentResult, phaseIndex + 1); }
    DomainPhaseState withResult(CompilationResult r) { return new DomainPhaseState(r, 0); }
}

// Per-tenant: one DomainPhaseState per domain
record TenantCompositionState(Map<DomainId, DomainPhaseState> phases) {
    TenantCompositionState withPhase(DomainId id, DomainPhaseState newPhase) {
        var copy = new LinkedHashMap<>(phases);
        copy.put(id, newPhase);
        return new TenantCompositionState(Map.copyOf(copy));
    }
}
```

`DomainPhaseState` is an immutable record; phase advancement creates a new instance. `TenantCompositionState` is also immutable — updates create a new instance with the modified domain entry.

**Thread safety model:** After `compose()` fires, the engine's runtime state (per-tenant phase tracking and recompose) is accessed from two threads:

1. **Scheduler thread:** `onReconciliationCycleCompleted` → advance phases → `recompose()`
2. **Workflow executor thread:** `handleReplan` → replace domain result → `recompose()`

Both paths must be serialized because `recompose()` reads ALL domain states, builds a composed graph, and calls `LifecycleManager.updateDesired()`. Concurrent recompose calls produce a last-writer-wins race. The engine serializes the entire advance-then-recompose and replace-then-recompose paths with `synchronized(recomposeLock)`. This is correct for the expected contention level — recompilation is rare; cycle completion is periodic but fast.

### 4.1 Registration (startup)

Domains register via CDI startup observers:

```java
@ApplicationScoped
public class InfraDomainRegistrar {

    @Inject InfraGoalCompiler compiler;
    @Inject CrossDomainCompositionEngine engine;

    void onStartup(@Observes StartupEvent event) {
        InfraGoals goals = loadFromConfig();
        CompilationResult result = compiler.compile(goals, graphFactory);

        engine.registerDomain(DomainRegistration.builder(
                DomainId.of("infra"), result)
            .provides(Set.of(NodeTypes.K8S_NAMESPACE, NodeTypes.K8S_DEPLOYMENT,
                             NodeTypes.DATABASE_CLUSTER))
            .build());
    }
}
```

```java
@ApplicationScoped
public class DeploymentDomainRegistrar {

    @Inject DeploymentGoalCompiler compiler;
    @Inject CrossDomainCompositionEngine engine;

    void onStartup(@Observes StartupEvent event) {
        DeploymentGoals goals = loadFromConfig();
        CompilationResult result = compiler.compile(goals, graphFactory);

        engine.registerDomain(DomainRegistration.builder(
                DomainId.of("deployment"), result)
            .provides(Set.of(NodeTypes.AGENT, NodeTypes.CHANNEL, NodeTypes.CASE_TYPE))
            .requires(Set.of(NodeTypes.K8S_NAMESPACE))
            .build());
    }
}
```

Each domain's startup observer uses default CDI priority (or explicit `@Priority` below `PLATFORM_AFTER + 1000`). The composition engine's own startup observer fires at `@Priority(PLATFORM_AFTER + 1000)` — after all domain registrations — and calls `compose()`.

**Late registration guard:** `registerDomain()` checks the `composed` flag (set by `compose()`) and rejects late registrations:

```java
public void registerDomain(DomainRegistration registration) {
    if (composed) {
        throw new IllegalStateException(
            "Cannot register domain '" + registration.domainId()
            + "' — composition already completed. "
            + "Ensure domain registrar @Priority is below PLATFORM_AFTER + 1000.");
    }
    domainConfigs.put(registration.domainId(), registration);
}
```

This is defense-in-depth against misconfigured `@Priority` values — a domain observer firing after the composition engine's observer would silently register without being composed.

### 4.2 Startup validation (D8)

When `compose()` fires, the engine validates all registrations before building the composed graph:

1. **Duplicate provides.** If two domains declare the same `NodeType` in their `provides` set → fail fast with: `"NodeType 'K8S_NAMESPACE' provided by both 'infra' and 'platform' — each NodeType must be provided by exactly one domain."` Consistent with `NodeProvisionerRouter`, which fails fast on overlapping `handledTypes()`.

2. **Unsatisfied requires.** If a domain requires a `NodeType` that no other domain provides → fail fast with: `"Domain 'deployment' requires NodeType 'K8S_NAMESPACE' but no domain provides it."` Catches missing domain JARs on the classpath.

3. **Circular requires.** Topological sort of the domain dependency graph (derived from requires→provides matching). Cycle detected → fail fast with: `"Circular dependency: infra → deployment → infra."` Uses Kahn's algorithm — the same approach as `TransitionPlanner`'s dependency ordering.

4. **Node ID uniqueness (D15).** For each registered domain's graph nodes, check all node IDs against previously registered domains. If a collision is found with differing specs → fail fast with: `"Node ID 'ns-default' exists in both domain 'infra' and domain 'platform' with different specs."` Intentional sharing (identical specs) is allowed — consistent with `overlay()` semantics. Convention: domain-prefixed IDs (e.g., `infra:namespace-default`, `deploy:agent-main`).

5. **Misconfiguration warning.** If the engine has zero registrations but `Instance<NodeProvisioner>` is non-empty (domain JARs are on the classpath), log a warning: `"CrossDomainCompositionEngine has zero registrations but NodeProvisioner beans exist — domains may be missing startup observers."`

### 4.3 Cross-domain edge creation

For each domain in topological order, the engine creates cross-domain edges based on requires/provides matching:

```
For each domain D with non-empty requires:
  For each required NodeType T in D.requires:
    Find the provider domain P where T ∈ P.provides
    Find all nodes in P's graph with type T → providerNodes
    Find all root nodes in D's graph → dependentRoots
    For each (root, provider) pair:
      Add Dependency(root, provider) to the merged graph
```

This ensures that domain D's root nodes cannot be planned by `TransitionPlanner` until all of domain P's T-typed nodes are provisioned. The ordering is structural — no runtime readiness polling needed in flattened mode.

**Intentional coarseness:** The edge creation is deliberately coarse — ALL root nodes of the dependent domain, regardless of their individual type-level needs, wait for ALL provider nodes of the required type. This means a webhook root node in the Deployment domain would also wait for all namespace nodes, even if webhooks don't need namespaces. This is correct: domain-level ordering expresses "domain D cannot start until domain P's required types exist." Fine-grained per-node cross-domain dependencies (e.g., "this specific agent needs this specific namespace") are expressed within a single domain by placing both nodes in the same graph. If a subset of a domain's nodes genuinely has no cross-domain dependency, that subset should be a separate domain. The provides/requires vocabulary is intentionally at the domain granularity, not the node granularity.

**Example:** Infra provides `{K8S_NAMESPACE, K8S_DEPLOYMENT, DATABASE_CLUSTER}`. Deployment requires `{K8S_NAMESPACE}`. The engine finds infra's namespace nodes (e.g., `infra:ns-prod`, `infra:ns-staging`) and deployment's root nodes (e.g., `deploy:agent-main`). It adds edges: `deploy:agent-main → infra:ns-prod`, `deploy:agent-main → infra:ns-staging`. Deployment's agent node now depends on infra's namespace nodes — `TransitionPlanner` provisions namespaces first.

### 4.4 Single-domain passthrough

When only one domain registers (or zero), the engine skips composition:

- **One registration:** Pass the `CompilationResult` directly to `LifecycleManager.start()`. No overlay, no edge creation. Identical behavior to today's single-domain case.
- **Zero registrations, no provisioners:** No-op. The engine is inert.
- **Zero registrations, provisioners exist:** Log misconfiguration warning (§4.2 point 5).

## 5. Flattened Mode

Default mode. Activated by `desiredstate.composition.mode=flattened` (or by omission — flattened is the default).

### 5.1 Graph merging

The engine builds the composed graph by overlaying all domain graphs in topological order:

```java
DesiredStateGraph recompose(String tenancyId, TenantCompositionState tenantState) {
    DesiredStateGraph composed = graphFactory.empty();
    for (DomainId domainId : topologicalOrder) {
        DomainPhaseState phaseState = tenantState.phases().get(domainId);
        composed = composed.overlay(phaseState.currentGraph());
    }
    // Add cross-domain edges (§4.3)
    for (CrossDomainEdge edge : computeCrossDomainEdges()) {
        composed = composed.withDependency(edge.dependency());
    }
    return composed;
}
```

`overlay()` adds nodes and edges from each domain's graph. It rejects conflicting nodes (same `NodeId`, different spec) — caught earlier by the node ID uniqueness validation (§4.2 point 4).

### 5.2 TransitionPlanner handles ordering

No special ordering logic is needed. `TransitionPlanner` already produces dependency-aware transition plans:

- **Additions:** roots before leaves. Cross-domain edges make provider nodes into roots relative to dependent nodes. Provider nodes are planned first.
- **Removals:** leaves before roots. Dependent nodes are removed before their provider dependencies.

The cross-domain edges are indistinguishable from intra-domain edges to the planner.

### 5.3 Single CAS (D14)

The composed graph is a single `DesiredStateGraph` with its own version counter. All CAS operations (`compareAndSetDesired()`) operate on this one graph. The composition engine is the single writer — `SituationRecompiler` results and phase transitions flow through the engine (D10), which serializes re-composition.

### 5.4 Fault propagation (D12)

Cross-domain faults are faults in a single graph. `FaultPolicyEngine` evaluates all fault policies against the merged graph. Policies from different domains compose naturally — they handle different `NodeType`/`FaultType` combinations. No new fault propagation mechanism needed.

**Trust assumption:** Composed domains are trusted — a fault policy from domain A has visibility into domain B's nodes. This is a feature for trusted composition (same-team domains, e.g., casehub-ops). For untrusted domain composition, fault policy scoping via `NodeType`-based filtering in `FaultPolicyEngine` would be needed — tracked as #143.

### 5.5 Per-domain lifecycle tracking (D13)

Each domain registers a `CompilationResult` — either `SingleGraph` or `Lifecycle(List<Phase>)`. The engine tracks each domain's current phase index. The composed graph at any moment is the overlay of each domain's current-phase graph plus cross-domain edges.

**CompilationResult type for LifecycleManager:** In composed mode, the engine always passes `CompilationResult.single(composedGraph)` to `LifecycleManager` — never `Lifecycle`. Per-domain lifecycle tracking is the composition engine's responsibility, not LifecycleManager's. LifecycleManager becomes a pass-through to `ReconciliationLoop` — it receives a single flat graph and delegates `start()` / `updateDesired()` directly. This avoids dual-listener conflict: only the composition engine's `GlobalReconciliationListener` tracks per-domain phase advancement. LifecycleManager's per-tenant `ReconciliationListener` is never set in composed mode because no `Lifecycle` CompilationResult reaches it.

The engine implements `GlobalReconciliationListener`. After each full reconciliation cycle, it checks each domain's phase completion for the given tenant:

```java
@Override
public void onReconciliationCycleCompleted(
        String tenancyId, DesiredStateGraph desired, ActualState actual) {
    TenantCompositionState tenantState = tenantStates.get(tenancyId);
    if (tenantState == null) return;

    synchronized (recomposeLock) {
        boolean recomposeNeeded = false;
        TenantCompositionState current = tenantStates.get(tenancyId);

        for (var entry : current.phases().entrySet()) {
            DomainId domainId = entry.getKey();
            DomainPhaseState phaseState = entry.getValue();
            if (!phaseState.hasLifecycle() || phaseState.isAtFinalPhase()) continue;

            CompilationResult.Lifecycle lc = (CompilationResult.Lifecycle) phaseState.currentResult();
            Phase currentPhase = lc.phases().get(phaseState.phaseIndex());
            CompletionCondition condition = currentPhase.completionCondition();

            DesiredStateGraph domainGraph = phaseState.currentGraph();
            if (condition.isComplete(domainGraph, actual)) {
                current = current.withPhase(domainId, phaseState.withAdvancedPhase());
                recomposeNeeded = true;
            }
        }

        if (recomposeNeeded) {
            tenantStates.put(tenancyId, current);
            recompose(tenancyId, current);
        }
    }
}
```

All phase advances for a tenant are batched into a single immutable `TenantCompositionState` update, followed by one `recompose()` call, all within `synchronized(recomposeLock)`. This eliminates the concurrent-recompose race between the scheduler thread and the workflow executor thread.

`GlobalReconciliationListener` fires from full `reconcile()` only (not type-filtered `reconcileTypes()`). This is correct — `CompletionCondition` evaluates against full actual state.

The engine uses `GlobalReconciliationListener` (CDI-discovered, multi-instance) rather than the per-tenant `ReconciliationListener` slot. The per-tenant slot is owned by `LifecycleManager` (a single `volatile` field in `TenantLoop`, always set by `LifecycleManager`).

### 5.6 SituationRecompiler integration (D10)

Domains register their `SituationRecompiler`s alongside their graphs:

```java
engine.registerDomain(DomainRegistration.builder(
        DomainId.of("infra"), result)
    .provides(Set.of(NodeTypes.K8S_NAMESPACE))
    .situationRecompilers(List.of(infraSituationRecompiler))
    .build());
```

The composition engine maps each recompiler to its domain at registration time via the `recompilerIndex` (`Map<SituationRecompiler, DomainId>`). When a `SituationRecompiler` fires and returns a `CompilationResult`:

1. The engine identifies which domain the recompiler belongs to via `recompilerIndex`
2. Replaces that domain's `DomainState` with the new `CompilationResult` via `domains.compute()`
3. Re-merges all domains' current graphs + cross-domain edges → new composed graph
4. Calls `LifecycleManager.updateDesired(tenancyId, CompilationResult.single(newComposedGraph))`

#### 5.6.1 DesiredStateReplanDispatch interception

`DesiredStateReplanDispatch` (engine-adapter/) is the only production caller of `SituationRecompilerEngine.recompile()`. In single-domain mode, it calls `recompilerEngine.recompile()` and then `lifecycleManager.updateDesired()`. In composed mode, this flow must be intercepted to prevent a domain-specific CompilationResult from replacing the entire composed graph.

The composition engine provides a `handleReplan()` method:

```java
public Optional<CompilationResult> handleReplan(
        String tenancyId, ActualState actual,
        ActiveSituation situation, DesiredStateGraphFactory factory) {
    TenantCompositionState tenantState = tenantStates.get(tenancyId);
    if (tenantState == null) return Optional.empty();

    // Iterate in priority order — preserves SituationRecompilerEngine's
    // chain-of-responsibility semantics (lower priority fires first)
    for (var entry : sortedRecompilers) {
        SituationRecompiler recompiler = entry.getKey();
        DomainId domainId = entry.getValue();
        DomainPhaseState phaseState = tenantState.phases().get(domainId);
        DesiredStateGraph domainGraph = phaseState.currentGraph();

        Optional<CompilationResult> result = recompiler.recompile(
            tenancyId, domainGraph, actual, situation, factory);
        if (result.isPresent()) {
            synchronized (recomposeLock) {
                TenantCompositionState current = tenantStates.get(tenancyId);
                current = current.withPhase(domainId,
                    current.phases().get(domainId).withResult(result.get()));
                tenantStates.put(tenancyId, current);
                DesiredStateGraph composed = recompose(tenancyId, current);
                lifecycleManager.updateDesired(tenancyId,
                    CompilationResult.single(composed));
            }
            return Optional.of(CompilationResult.single(
                tenantStates.get(tenancyId).phases().get(domainId).currentGraph()));
        }
    }
    // Try cross-domain recompilers (§5.6.3)
    return handleCrossDomainReplan(tenancyId, actual, situation, factory);
}
```

Key differences from `SituationRecompilerEngine`:
- Domain-specific recompilers receive their **domain graph** (not the composed graph), so they only see and modify their own nodes
- Iteration follows `sortedRecompilers` — a priority-sorted list built at `compose()` time, preserving the chain-of-responsibility contract that existing `SituationRecompiler` implementations rely on (lower `priority()` fires first, first match wins)
- State update and recompose are serialized within `synchronized(recomposeLock)` to prevent concurrent-recompose races

`DesiredStateReplanDispatch` injects `CrossDomainCompositionEngine` (optional — `Instance<CrossDomainCompositionEngine>`). When composition is active (engine has registrations), it delegates to `engine.handleReplan()`. Otherwise, it falls through to the existing `SituationRecompilerEngine` + `LifecycleManager.updateDesired()` path:

```java
// In DesiredStateReplanDispatch.replan():
Optional<CompilationResult> newResult;
if (compositionEngine.isResolvable() && compositionEngine.get().isActive()) {
    newResult = compositionEngine.get().handleReplan(
        tenancyId, actual, situation, graphFactory);
} else {
    newResult = recompilerEngine.recompile(
        tenancyId, current, actual, situation, graphFactory);
    newResult.ifPresent(r -> lifecycleManager.updateDesired(tenancyId, r));
}
```

**Impact:** `DesiredStateReplanDispatch` gains a new optional `Instance<CrossDomainCompositionEngine>` injection. This is the only change to existing code outside the new `composition` package. `SituationRecompilerEngine` itself is unchanged.

#### 5.6.2 Cascade detection

If a domain's recompilation changes its provides set (e.g., removes a `NodeType`), the engine checks whether downstream domains' requires are still satisfied. Initial behavior: fail fast with descriptive error. Cascade recompilation is tracked in #141.

#### 5.6.3 Cross-domain recompilers

Cross-domain recompilers (spanning multiple domains' types) are registered directly with the engine, not via a domain registration:

```java
public void registerCrossDomainRecompiler(SituationRecompiler recompiler) { ... }
```

**API contract:**
- Cross-domain recompilers receive the **composed graph** (all domains merged) as the `current` parameter
- They return a `CompilationResult` that replaces the **entire** composed graph
- Cross-domain recompilers are tried after all domain-specific recompilers return `Optional.empty()` — they are the fallback in the chain, not a separate chain. Within the cross-domain set, iteration follows `sortedCrossDomainRecompilers` (priority-sorted, first match wins)

**Phase reset semantics:** When a cross-domain recompiler fires, per-domain lifecycle phase tracking is abandoned for the affected tenant. Specifically:
1. The returned graph replaces the composed graph directly via `LifecycleManager.updateDesired()`
2. Each domain's `DomainPhaseState` is replaced with `DomainPhaseState(CompilationResult.single(partition), 0)` where `partition` is the subset of the returned graph matching the domain's `provides` NodeTypes
3. Lifecycle phases are NOT re-entered — each domain is treated as SingleGraph until a subsequent domain-specific recompilation restores Lifecycle phases
4. This is equivalent to a full re-composition from scratch with each domain holding a SingleGraph derived from its partition of the returned graph

**Interaction with `SituationRecompilerEngine`:** The composition engine takes over the entire recompiler dispatch in composed mode (§5.6.1). `SituationRecompilerEngine` is not called directly — domain-specific recompilers are iterated by the composition engine (with domain graph scoping), and cross-domain recompilers are iterated as a fallback. `SituationRecompilerEngine`'s CDI-discovered recompilers are mapped to domains at registration time; any CDI-discovered recompiler not registered with a domain is treated as a cross-domain recompiler.

The `SituationRecompiler` SPI in api/ is unchanged — no `domainId()` method, no awareness of cross-domain composition. The composition engine in runtime/ knows which recompilers belong to which domain because domains push them at registration.

## 6. Hierarchical Mode

Activated via `desiredstate.composition.mode=hierarchical`.

### 6.1 Meta-loop

The composition engine creates a meta-graph with one domain-level node per registered domain. Domain-level nodes are `DesiredNode` instances with a synthetic `NodeType` (e.g., `NodeType.of("domain")`) and a `DomainNodeSpec` containing the domain's registration metadata.

Domain-level dependency edges are derived from provides/requires matching:

```
deployment requires K8S_NAMESPACE
infra provides K8S_NAMESPACE
→ Dependency(deployment-domain-node, infra-domain-node)
```

The meta-graph is fed to a dedicated `ReconciliationLoop` instance — the meta-loop. The meta-loop has its own `DomainNodeProvisioner` that provisions domain-level nodes by starting inner loops.

### 6.2 Inner loops

When the meta-loop provisions a domain-level node, the `DomainNodeProvisioner`:

1. Extracts the domain's `CompilationResult` from the `DomainNodeSpec`
2. Creates an inner `ReconciliationLoop` via `ReconciliationLoop.Builder` (NOT CDI)
3. If the CompilationResult is `Lifecycle`, manages phase transitions directly (§6.5)
4. Starts the inner loop with the domain's current-phase graph and the shared `tenancyId`

**CDI wiring for inner loops:** Inner `ReconciliationLoop` instances are created via the existing `ReconciliationLoop.Builder`, not CDI injection. The Builder requires `TransitionPlanner`, `TransitionExecutor`, `ActualStateAdapterRouter`, `FaultPolicyEngine`, and `MergedEventSource` — all shared CDI singletons that the `DomainNodeProvisioner` receives via constructor injection and passes to each Builder. The `NodeProvisionerRouter` is set via `Builder.router()`. `GlobalReconciliationListener` list is set via `Builder.globalListeners()` — inner loops get an empty list (or a domain-specific listener for phase advancement). Inner loops create their own `ScheduledExecutorService` (via the Builder's build path), independent of the meta-loop's scheduler.

```java
ReconciliationLoop innerLoop = ReconciliationLoop.builder(
        planner, executor, actualStateAdapterRouter, faultPolicyEngine, mergedEventSource)
    .router(provisionerRouter)
    .globalListeners(List.of(domainPhaseListener))
    .build();
innerLoop.start(tenancyId, domainGraph);
```

Each inner loop uses the same `ActualStateAdapterRouter`, `NodeProvisionerRouter`, `MergedEventSource`, and `FaultPolicyEngine` as the meta-loop — these are already multi-domain-ready. The routers dispatch by `NodeType`, so inner loops only trigger adapters/provisioners for their domain's node types.

**Inner loop cleanup (deprovision):** When the meta-loop deprovisions a domain-level node, `DomainNodeProvisioner.deprovision()` stops the inner loop and reclaims its resources:

```java
public DeprovisionResult deprovision(DesiredNode node, DeprovisionContext context) {
    DomainNodeSpec spec = (DomainNodeSpec) node.spec();
    ReconciliationLoop innerLoop = activeInnerLoops.remove(spec.domainId());
    if (innerLoop != null) {
        innerLoop.stop(context.tenancyId());
        innerLoop.shutdown();
    }
    return DeprovisionResult.success();
}
```

`ReconciliationLoop.stop(tenancyId)` cancels all scheduled futures (resync timers, debounce timers), cancels the `MergedEventSource` subscription, and removes the `TenantLoop` entry. `ReconciliationLoop.shutdown()` shuts down the `ScheduledExecutorService`. Together these reclaim all resources that the Builder-created inner loop allocated.

The `DomainNodeProvisioner` maintains a `Map<DomainId, ReconciliationLoop> activeInnerLoops` to track inner loops it has provisioned. Inner loops created via Builder are NOT registered in the CDI singleton `ReconciliationLoop`'s `loops` map, so the CDI `@PreDestroy` shutdown hook does not reach them — explicit cleanup via deprovision is required.

### 6.3 Domain readiness (D3)

The `DomainNodeProvisioner` reports a domain-level node as `PRESENT` when the domain's `CompletionCondition` is satisfied. The condition evaluates against the inner loop's desired graph and the actual state:

```java
// readinessCondition is guaranteed non-null (§3.2 — Builder defaults to allPresent())
CompletionCondition condition = registration.readinessCondition();

if (condition.isComplete(innerDesiredGraph, actualState)) {
    return ProvisionResult.success();
}
// Not ready yet — return Failed so the meta-loop retries on next cycle.
// The meta-loop's fault policy is configured to suppress escalation for
// domain-level nodes, treating "not ready" as a transient condition.
return ProvisionResult.failed("domain '" + domainId + "' not ready — awaiting convergence");
```

The default condition (`allPresent()`) means a domain-level node transitions to PRESENT when all of its inner nodes are PRESENT. Domains with softer readiness semantics override via `readinessCondition` in their `DomainRegistration`.

In hierarchical mode, `CompletionCondition` scoping by `requires` types means the readiness check only considers the types that downstream domains actually need. If domain B requires `{K8S_NAMESPACE}` from domain A, domain A's readiness (from B's perspective) is determined by A's namespace nodes only — not A's database cluster nodes.

### 6.4 Fault isolation

In hierarchical mode, each inner loop has its own fault handling cycle. A fault in domain A's inner loop is handled by domain A's fault policies. Cross-domain fault propagation (domain A's fault affecting domain B) is not supported in the initial implementation — inner loops are independent. Cross-domain fault escalation (e.g., domain A's persistent failure triggers domain B's degradation) via meta-loop fault policies is tracked as #142.

### 6.5 Per-domain lifecycle in hierarchical mode

If a domain returns `CompilationResult.Lifecycle`, the `DomainNodeProvisioner` manages phase transitions directly — not `LifecycleManager`. `LifecycleManager` is an `@ApplicationScoped` singleton bound to the CDI `ReconciliationLoop` instance; it cannot manage phases for Builder-created inner loops.

The `DomainNodeProvisioner` implements lifecycle phase management for inner loops by:

1. Extracting the phases from `CompilationResult.Lifecycle`
2. Starting the inner `ReconciliationLoop` with the first phase's graph
3. Setting a `ReconciliationListener` on the inner loop that evaluates each phase's `CompletionCondition`
4. On phase completion, calling `innerLoop.compareAndSetDesired()` to advance to the next phase's graph
5. Reporting the domain-level node as `PRESENT` when the final phase's `CompletionCondition` is satisfied

This is equivalent to LifecycleManager's `onCycleCompleted()` logic — the same CAS-based phase transition, the same `CompletionCondition` evaluation — but integrated into the provisioner rather than requiring a separate LifecycleManager instance per inner loop.

For domains returning `CompilationResult.SingleGraph`, no lifecycle management is needed — the inner loop starts with the single graph and the domain-level node is PRESENT when the domain's `readinessCondition` is satisfied.

## 7. Same-Tenant Composition (D11)

Composed domains share the same `tenancyId`. All domain nodes within a composition belong to the same tenant.

**Per-tenant state model:** The engine stores per-domain phase tracking per tenant via `ConcurrentHashMap<String, TenantCompositionState>` — the same per-tenant keying pattern used by `ReconciliationLoop` (`ConcurrentHashMap<String, TenantLoop>`) and `LifecycleManager` (`ConcurrentHashMap<String, TenantLifecycle>`).

`compose(tenancyId)` initializes a `TenantCompositionState` for the given tenant with each domain's initial `DomainPhaseState` (phase index 0). Domain configuration (provides, requires, topology) is global and tenant-independent. Phase progression is tenant-scoped — tenant A advancing Infra to phase 2 does not affect tenant B's Infra phase.

Multiple tenants can share one engine instance. Each tenant's composed graph evolves independently.

If a domain provisions shared infrastructure across tenants, it operates as an independent `ReconciliationLoop` — not as a composed domain within a tenant's graph. Cross-tenant coordination is an orchestration concern above the desired-state runtime.

## 8. CDI Wiring

### 8.1 Startup sequence

```
StartupEvent fires (CDI @Priority ordering):
  1. Domain startup observers (default priority)
     → InfraDomainRegistrar.onStartup() → engine.registerDomain(...)
     → DeploymentDomainRegistrar.onStartup() → engine.registerDomain(...)
     → ComplianceDomainRegistrar.onStartup() → engine.registerDomain(...)
     → IoTDomainRegistrar.onStartup() → engine.registerDomain(...)
  2. Composition engine observer (@Priority(PLATFORM_AFTER + 1000))
     → validate registrations
     → compose and start reconciliation
```

### 8.2 Backward compatibility

Existing single-domain apps are unchanged. If no domain registers with the composition engine, it is inert. If exactly one domain registers, it passes the `CompilationResult` directly to `LifecycleManager` — zero composition overhead.

Apps that currently call `ReconciliationLoop.start()` or `LifecycleManager.start()` directly continue to work. The composition engine does not intercept these calls. It is an additive capability.

### 8.3 Misconfiguration detection

At startup, if the engine has zero registrations but `Instance<NodeProvisioner>` is non-empty, it logs a warning. This catches the case where domain JARs are on the classpath but missing startup observers.

## 9. Impact on Existing Code

| Component | Change |
|-----------|--------|
| `ReconciliationLoop` | None. Receives composed or single-domain graphs as before. |
| `LifecycleManager` | None. Receives `CompilationResult.single()` from the composition engine in composed mode. Phase transition CAS logic unchanged but not exercised in composed mode. |
| `TransitionPlanner` | None. Cross-domain edges are standard `Dependency` instances. |
| `FaultPolicyEngine` | None. `List<FaultPolicy>` already collects from all domains via CDI. No CDI ambiguity — multi-domain-ready as-is. |
| `ActualStateAdapterRouter` | None. Already dispatches by `NodeType` across domains. |
| `NodeProvisionerRouter` | None. Already dispatches by `NodeType` across domains. |
| `MergedEventSource` | None. Already merges multiple `EventSource` streams via `Multi.merge()`. No CDI ambiguity — multi-domain-ready as-is. |
| `SituationRecompilerEngine` | None. The composition engine takes over recompiler dispatch in composed mode (§5.6.1). `SituationRecompilerEngine` class itself is unchanged. |
| `DesiredStateReplanDispatch` | Gains optional `Instance<CrossDomainCompositionEngine>` injection. When composition is active, delegates replan to the engine instead of directly to `SituationRecompilerEngine` + `LifecycleManager` (§5.6.1). |
| api/ module | Gains `DomainId` value type only. |
| runtime/ module | Gains `io.casehub.desiredstate.runtime.composition` package. |
| Example modules | No changes required. Examples remain single-domain. A new cross-domain example could demonstrate the composition pattern. |

**CDI qualification (issue #140 requirement):** FaultPolicy and EventSource CDI ambiguity is already resolved by existing infrastructure — `FaultPolicyEngine` takes `List<FaultPolicy>` via CDI collection injection, and `MergedEventSource` composes multiple `EventSource` streams via `Multi.merge()`. `GoalCompiler` CDI ambiguity is sidestepped by the push model (§2). No CDI qualifier annotations are needed for any of these SPIs.

### 9.1 ops/app migration path

Replace `ApplicationGoalCompiler` (the hand-coded workaround) with per-domain registrars:

**Before:**
```java
// ApplicationGoalCompiler — hand-coded, only handles infra, bypasses LifecycleManager
var graph = goalCompiler.compileForCluster(services, clusterId, namespace, graphFactory);
reconciliationLoop.start(key, graph);
```

**After:**
```java
// InfraDomainRegistrar — registers infra domain with provides/requires
engine.registerDomain(DomainRegistration.builder(DomainId.of("infra"), infraResult)
    .provides(Set.of(NodeTypes.K8S_NAMESPACE, NodeTypes.K8S_DEPLOYMENT))
    .build());

// DeploymentDomainRegistrar — registers deployment domain
engine.registerDomain(DomainRegistration.builder(DomainId.of("deployment"), deployResult)
    .provides(Set.of(NodeTypes.AGENT, NodeTypes.CHANNEL))
    .requires(Set.of(NodeTypes.K8S_NAMESPACE))
    .build());

// Engine composes and starts reconciliation automatically
```

## 10. Testing Strategy

| Component | Approach |
|-----------|----------|
| `DomainRegistration` validation | Unit: null checks, empty provides/requires, builder defaults, non-null readinessCondition |
| Provides/requires graph | Unit: duplicate provides detection, cycle detection (Kahn's), unsatisfied requires |
| Cross-domain edge creation | Unit: edges from requiring domain's roots to provider's typed nodes; empty requires = no edges |
| Node ID uniqueness | Unit: collision with different specs → error; collision with same specs → allowed |
| Multi-compiler overlay | Unit: overlay graphs from different compiler types with different NodeTypes; verify merged graph contains all node types and correct dependency edges. Validates the cross-compiler overlay pattern claimed in §2. |
| Flattened composition | Unit: overlay + edges produce correct merged graph; single-domain passthrough |
| Per-domain lifecycle tracking | Unit: phase advancement via `GlobalReconciliationListener`; re-composition on phase change |
| SituationRecompiler re-merge | Unit: domain recompilation triggers correct re-merge; cascade detection. Verify domain-specific recompilers receive domain graph, not composed graph. |
| DesiredStateReplanDispatch delegation | Unit: verify replan delegates to composition engine when active; verify fallback to SituationRecompilerEngine when composition is inactive |
| Hierarchical meta-loop | Integration: domain-level nodes provisioned in topological order; inner loops started |
| Hierarchical readiness | Integration: `CompletionCondition` gates downstream domain activation |
| Hierarchical lifecycle | Integration: domain with `CompilationResult.Lifecycle` — inner loop phases advance via `DomainNodeProvisioner`; domain-level node PRESENT after final phase |
| End-to-end flattened | Integration: multi-domain app with infra→deployment ordering; provision namespaces before agents |
| End-to-end hierarchical | Integration: same multi-domain app in hierarchical mode |
| Backward compat | Integration: existing single-domain examples unchanged with composition engine on classpath |

## 11. Migration Notes

- **Existing single-domain apps:** Zero changes. The composition engine is inert with zero or one registrations.
- **Pre-release project:** No backward compatibility concern for new API additions.
- **ops/app migration:** Replace `ApplicationGoalCompiler` with per-domain registrars. Remove direct `ReconciliationLoop.start()` calls. Let the composition engine manage composition and lifecycle.
- **Example modules:** No changes required. They remain single-domain demonstrations.
- **Multi-process deployment:** Out of scope for this design. The hierarchical architecture provides the logical foundation — domain-level abstraction, readiness gates, independent fault isolation — that naturally extends to multi-process deployment with distributed coordination (event bus, shared state store). Tracked as #144.

## 12. References

### Decisions

| Decision | Summary |
|----------|---------|
| D1 | Hierarchical architecture from day one — builds for the future, not just current consumers |
| D2 | Push model — domains compile and register CompilationResult, eliminating type erasure |
| D3 | Pluggable CompletionCondition — default all-PRESENT-zero-faults, domains override |
| D4 | Type-level cross-domain deps — no naming coupling, right abstraction boundary |
| D5 | Goal loading is domain responsibility — follows from push model |
| D6 | Transparent flattening — single-process default, hierarchical via config |
| D7 | Automatic CDI discovery — @Priority convention for startup ordering |
| D8 | Provides/requires with validation — fail-fast on duplicate provides, cycles, unsatisfied requires |
| D9 | Composition engine in runtime/ — follows existing compositor pattern |
| D10 | Composition wraps LifecycleManager — SituationRecompiler integration via registration-based matching |
| D11 | Same-tenant composition — composed domains share tenancyId |
| D12 | Fault propagation — mode-dependent (merged graph in flat, domain-local in hierarchical) |
| D13 | Per-domain lifecycle — composition engine tracks phase state, re-composes on transitions |
| D14 | Single composed graph, single CAS per loop |
| D15 | Convention-based node ID uniqueness with startup validation |

### Code files

- `api/src/main/java/io/casehub/desiredstate/api/GoalCompiler.java` — parameterized SPI, root of the type erasure problem
- `api/src/main/java/io/casehub/desiredstate/api/DesiredStateGraph.java` — `overlay()`, `connect()`, `filterByTypes()`
- `api/src/main/java/io/casehub/desiredstate/api/CompletionCondition.java` — `allPresent()` factory, `isComplete()` contract
- `api/src/main/java/io/casehub/desiredstate/api/CompilationResult.java` — `SingleGraph`, `Lifecycle` sealed types
- `api/src/main/java/io/casehub/desiredstate/api/GlobalReconciliationListener.java` — CDI-discovered multi-instance listener
- `runtime/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java` — per-tenant loop, CAS, interval groups
- `runtime/src/main/java/io/casehub/desiredstate/runtime/LifecycleManager.java` — CAS phase transitions
- `runtime/src/main/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraph.java` — `overlay()` implementation with conflict detection
- `runtime/src/main/java/io/casehub/desiredstate/runtime/SituationRecompilerEngine.java` — chain-of-responsibility recompiler
- `runtime/src/main/java/io/casehub/desiredstate/runtime/FaultPolicyEngine.java` — `List<FaultPolicy>` multi-domain ready

### Issues

- casehub-desiredstate#140 — this design
- casehub-ops#23 — first consumer: cross-domain dependency graphs
- casehub-desiredstate#51, #52 — predecessor: multi-domain SPI routing (ActualStateAdapterRouter, MergedEventSource)
- casehub-desiredstate#141 — deferred: cascade recompilation when provides set changes
- casehub-desiredstate#142 — deferred: cross-domain fault escalation in hierarchical mode
- casehub-desiredstate#143 — deferred: fault policy scoping for untrusted domain composition
- casehub-desiredstate#144 — deferred: multi-process deployment model
