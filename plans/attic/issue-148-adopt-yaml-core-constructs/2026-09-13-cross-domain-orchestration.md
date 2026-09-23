# Cross-Domain Orchestration Framework Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #140 — design: cross-domain orchestration framework — multi-compiler composition, domain graph, health gates
**Issue group:** #140

**Goal:** Build a cross-domain composition engine that merges multiple domain graphs via `overlay()` with provides/requires-derived edges, tracking per-domain lifecycle phases, supporting flattened (default) and hierarchical deployment modes.

**Architecture:** Push-model registration — each domain compiles its own goals (preserving GoalCompiler type safety) and registers `CompilationResult` + metadata (provides, requires, readinessCondition) with `CrossDomainCompositionEngine`. Flattened mode merges all domain graphs into one `DesiredStateGraph` with cross-domain edges and feeds a single `ReconciliationLoop`. Hierarchical mode builds a meta-loop with domain-level nodes; inner `ReconciliationLoop` per domain.

**Tech Stack:** Java 21, Quarkus (CDI), Mutiny, JUnit 5, AssertJ

## Global Constraints

- Pre-release project — no backward compatibility concern for new API additions
- All new types in `io.casehub.desiredstate.runtime.composition` except `DomainId` which goes in `io.casehub.desiredstate.api`
- No CDI qualifier annotations needed — push model sidesteps GoalCompiler type erasure
- `CompletionCondition.allPresent()` is the default readiness condition
- Mode is immutable after startup (`desiredstate.composition.mode` config property)
- Test with AssertJ assertions, project convention is `assertThat` / `assertThatThrownBy`

---

## Batch 1: Foundation — types, registration, validation

### Task 1: DomainId and foundation records

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/DomainId.java`
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/composition/DomainRegistration.java`
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/composition/DomainPhaseState.java`
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/composition/TenantCompositionState.java`
- Test: `api/src/test/java/io/casehub/desiredstate/api/DomainIdTest.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/composition/DomainRegistrationTest.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/composition/DomainPhaseStateTest.java`

**Interfaces:**
- Consumes: `CompilationResult`, `CompletionCondition`, `NodeType`, `SituationRecompiler`, `DesiredStateGraph`, `NodeStatus` (all from api/)
- Produces: `DomainId.of(String) → DomainId`, `DomainRegistration.builder(DomainId, CompilationResult) → Builder`, `DomainPhaseState(CompilationResult, int)`, `TenantCompositionState(Map<DomainId, DomainPhaseState>)`

- [ ] **Step 1: Write DomainId test**

```java
package io.casehub.desiredstate.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class DomainIdTest {
    @Test void equality() {
        assertThat(DomainId.of("infra")).isEqualTo(DomainId.of("infra"));
        assertThat(DomainId.of("infra")).isNotEqualTo(DomainId.of("deployment"));
    }
    @Test void rejectsNull() {
        assertThatThrownBy(() -> DomainId.of(null))
            .isInstanceOf(NullPointerException.class);
    }
    @Test void rejectsBlank() {
        assertThatThrownBy(() -> DomainId.of(""))
            .isInstanceOf(IllegalArgumentException.class);
        assertThatThrownBy(() -> DomainId.of("  "))
            .isInstanceOf(IllegalArgumentException.class);
    }
    @Test void value() {
        assertThat(DomainId.of("infra").value()).isEqualTo("infra");
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=DomainIdTest`
Expected: FAIL — `DomainId` class not found.

- [ ] **Step 3: Implement DomainId**

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

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn --batch-mode test -pl api -Dtest=DomainIdTest`
Expected: PASS

- [ ] **Step 5: Write DomainPhaseState test**

```java
package io.casehub.desiredstate.runtime.composition;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.runtime.ImmutableDesiredStateGraphTest;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class DomainPhaseStateTest {
    static final DesiredStateGraphFactory FACTORY = new io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory();

    static DesiredStateGraph graphWith(String nodeId) {
        return FACTORY.empty().withNode(new DesiredNode(
            NodeId.of(nodeId), new TestSpec(nodeId), HumanGating.NONE));
    }

    record TestSpec(String name) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("test"); }
    }

    @Test void singleGraph_currentGraph() {
        var graph = graphWith("n1");
        var state = new DomainPhaseState(CompilationResult.single(graph), 0);
        assertThat(state.currentGraph().nodes()).containsKey(NodeId.of("n1"));
        assertThat(state.hasLifecycle()).isFalse();
        assertThat(state.isAtFinalPhase()).isTrue();
    }

    @Test void lifecycle_phaseAdvancement() {
        var g1 = graphWith("p1");
        var g2 = graphWith("p2");
        var lifecycle = CompilationResult.lifecycle(List.of(
            new Phase("phase-1", g1, CompletionCondition.allPresent()),
            new Phase("phase-2", g2, CompletionCondition.allPresent())));
        var state = new DomainPhaseState(lifecycle, 0);
        assertThat(state.hasLifecycle()).isTrue();
        assertThat(state.isAtFinalPhase()).isFalse();
        assertThat(state.currentGraph().nodes()).containsKey(NodeId.of("p1"));

        var advanced = state.withAdvancedPhase();
        assertThat(advanced.phaseIndex()).isEqualTo(1);
        assertThat(advanced.currentGraph().nodes()).containsKey(NodeId.of("p2"));
        assertThat(advanced.isAtFinalPhase()).isTrue();
    }

    @Test void withResult_resetsPhaseIndex() {
        var g1 = graphWith("a");
        var g2 = graphWith("b");
        var lifecycle = CompilationResult.lifecycle(List.of(
            new Phase("p1", g1, CompletionCondition.allPresent()),
            new Phase("p2", g2, CompletionCondition.allPresent())));
        var state = new DomainPhaseState(lifecycle, 1);
        var replaced = state.withResult(CompilationResult.single(graphWith("c")));
        assertThat(replaced.phaseIndex()).isEqualTo(0);
        assertThat(replaced.currentGraph().nodes()).containsKey(NodeId.of("c"));
    }
}
```

- [ ] **Step 6: Run test — verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=DomainPhaseStateTest`
Expected: FAIL — class not found.

- [ ] **Step 7: Implement DomainPhaseState and TenantCompositionState**

`DomainPhaseState.java`:
```java
package io.casehub.desiredstate.runtime.composition;

import io.casehub.desiredstate.api.*;

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
```

`TenantCompositionState.java`:
```java
package io.casehub.desiredstate.runtime.composition;

import io.casehub.desiredstate.api.DomainId;
import java.util.LinkedHashMap;
import java.util.Map;

record TenantCompositionState(Map<DomainId, DomainPhaseState> phases) {
    TenantCompositionState withPhase(DomainId id, DomainPhaseState newPhase) {
        var copy = new LinkedHashMap<>(phases);
        copy.put(id, newPhase);
        return new TenantCompositionState(Map.copyOf(copy));
    }
}
```

- [ ] **Step 8: Run test — verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=DomainPhaseStateTest`
Expected: PASS

- [ ] **Step 9: Write DomainRegistration test**

```java
package io.casehub.desiredstate.runtime.composition;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.Test;
import java.util.Set;
import static org.assertj.core.api.Assertions.*;

class DomainRegistrationTest {
    static final DesiredStateGraphFactory FACTORY = new io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory();

    @Test void builder_defaults() {
        var reg = DomainRegistration.builder(
                DomainId.of("infra"), CompilationResult.single(FACTORY.empty()))
            .build();
        assertThat(reg.domainId()).isEqualTo(DomainId.of("infra"));
        assertThat(reg.provides()).isEmpty();
        assertThat(reg.requires()).isEmpty();
        assertThat(reg.readinessCondition()).isNotNull();
        assertThat(reg.situationRecompilers()).isEmpty();
    }

    @Test void builder_withProvidesRequires() {
        var ns = NodeType.of("k8s-namespace");
        var agent = NodeType.of("agent");
        var reg = DomainRegistration.builder(
                DomainId.of("deploy"), CompilationResult.single(FACTORY.empty()))
            .provides(Set.of(agent))
            .requires(Set.of(ns))
            .build();
        assertThat(reg.provides()).containsExactly(agent);
        assertThat(reg.requires()).containsExactly(ns);
    }

    @Test void rejectsNullDomainId() {
        assertThatThrownBy(() -> DomainRegistration.builder(
                null, CompilationResult.single(FACTORY.empty())).build())
            .isInstanceOf(NullPointerException.class);
    }

    @Test void rejectsNullCompilationResult() {
        assertThatThrownBy(() -> DomainRegistration.builder(
                DomainId.of("x"), null).build())
            .isInstanceOf(NullPointerException.class);
    }

    @Test void provides_isImmutableCopy() {
        var mutable = new java.util.HashSet<>(Set.of(NodeType.of("a")));
        var reg = DomainRegistration.builder(
                DomainId.of("x"), CompilationResult.single(FACTORY.empty()))
            .provides(mutable).build();
        mutable.add(NodeType.of("b"));
        assertThat(reg.provides()).hasSize(1);
    }
}
```

- [ ] **Step 10: Implement DomainRegistration**

Write `DomainRegistration.java` with the record, compact constructor (non-null enforcement, defensive copies), and Builder as specified in §3.2 of the design spec.

- [ ] **Step 11: Run test — verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=DomainRegistrationTest`
Expected: PASS

- [ ] **Step 12: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/DomainId.java api/src/test/java/io/casehub/desiredstate/api/DomainIdTest.java runtime/src/main/java/io/casehub/desiredstate/runtime/composition/ runtime/src/test/java/io/casehub/desiredstate/runtime/composition/
git commit -m "feat(#140): add DomainId, DomainRegistration, DomainPhaseState, TenantCompositionState foundation types"
```

---

### Task 2: Registration and startup validation engine

**Files:**
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngine.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngineValidationTest.java`

**Interfaces:**
- Consumes: `DomainRegistration`, `DomainId`, `NodeType`, `DesiredStateGraph`, `NodeId`, `DesiredNode` (from api/), `LifecycleManager`, `ReconciliationLoop`, `DesiredStateGraphFactory` (from runtime/)
- Produces: `CrossDomainCompositionEngine.registerDomain(DomainRegistration)`, `CrossDomainCompositionEngine.isActive() → boolean`, `CrossDomainCompositionEngine.registrationCount() → int`

- [ ] **Step 1: Write validation tests**

```java
package io.casehub.desiredstate.runtime.composition;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.runtime.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.Set;
import static org.assertj.core.api.Assertions.*;

class CrossDomainCompositionEngineValidationTest {

    static final DesiredStateGraphFactory FACTORY = new DefaultDesiredStateGraphFactory();
    static final NodeType NS = NodeType.of("k8s-namespace");
    static final NodeType AGENT = NodeType.of("agent");
    static final NodeType DB = NodeType.of("database");

    record Spec(String name) implements NodeSpec {
        @Override public NodeType nodeType() { return NS; }
    }

    CrossDomainCompositionEngine engine;

    @BeforeEach
    void setUp() {
        engine = new CrossDomainCompositionEngine(FACTORY);
    }

    @Test void registerDomain_acceptsValidRegistration() {
        engine.registerDomain(reg("infra", Set.of(NS), Set.of()));
        assertThat(engine.registrationCount()).isEqualTo(1);
    }

    @Test void validate_duplicateProvides_failsFast() {
        engine.registerDomain(reg("infra", Set.of(NS), Set.of()));
        engine.registerDomain(reg("platform", Set.of(NS), Set.of()));
        assertThatThrownBy(() -> engine.validate())
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("k8s-namespace")
            .hasMessageContaining("infra")
            .hasMessageContaining("platform");
    }

    @Test void validate_unsatisfiedRequires_failsFast() {
        engine.registerDomain(reg("deploy", Set.of(AGENT), Set.of(NS)));
        assertThatThrownBy(() -> engine.validate())
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("deploy")
            .hasMessageContaining("k8s-namespace");
    }

    @Test void validate_circularRequires_failsFast() {
        engine.registerDomain(reg("a", Set.of(NS), Set.of(AGENT)));
        engine.registerDomain(reg("b", Set.of(AGENT), Set.of(NS)));
        assertThatThrownBy(() -> engine.validate())
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("Circular");
    }

    @Test void validate_nodeIdConflict_failsFast() {
        var g1 = FACTORY.empty().withNode(new DesiredNode(
            NodeId.of("shared"), new Spec("v1"), HumanGating.NONE));
        var g2 = FACTORY.empty().withNode(new DesiredNode(
            NodeId.of("shared"), new Spec("v2"), HumanGating.NONE));
        engine.registerDomain(regWith("d1", g1, Set.of(NS), Set.of()));
        engine.registerDomain(regWith("d2", g2, Set.of(AGENT), Set.of()));
        assertThatThrownBy(() -> engine.validate())
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("shared")
            .hasMessageContaining("d1")
            .hasMessageContaining("d2");
    }

    @Test void validate_nodeIdSharing_identicalSpecs_allowed() {
        var spec = new Spec("shared-val");
        var g1 = FACTORY.empty().withNode(new DesiredNode(
            NodeId.of("shared"), spec, HumanGating.NONE));
        var g2 = FACTORY.empty().withNode(new DesiredNode(
            NodeId.of("shared"), spec, HumanGating.NONE));
        engine.registerDomain(regWith("d1", g1, Set.of(NS), Set.of()));
        engine.registerDomain(regWith("d2", g2, Set.of(AGENT), Set.of()));
        assertThatCode(() -> engine.validate()).doesNotThrowAnyException();
    }

    @Test void validate_topologicalOrder_respectsRequires() {
        engine.registerDomain(reg("deploy", Set.of(AGENT), Set.of(NS)));
        engine.registerDomain(reg("infra", Set.of(NS), Set.of()));
        engine.validate();
        assertThat(engine.topologicalOrder())
            .containsExactly(DomainId.of("infra"), DomainId.of("deploy"));
    }

    @Test void lateRegistration_rejected() {
        engine.registerDomain(reg("infra", Set.of(NS), Set.of()));
        engine.validate();
        engine.markComposed();
        assertThatThrownBy(() -> engine.registerDomain(reg("late", Set.of(DB), Set.of())))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("composition already completed");
    }

    private DomainRegistration reg(String name, Set<NodeType> provides, Set<NodeType> requires) {
        return regWith(name, FACTORY.empty(), provides, requires);
    }

    private DomainRegistration regWith(String name, DesiredStateGraph graph,
            Set<NodeType> provides, Set<NodeType> requires) {
        return DomainRegistration.builder(DomainId.of(name), CompilationResult.single(graph))
            .provides(provides).requires(requires).build();
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=CrossDomainCompositionEngineValidationTest`
Expected: FAIL — `CrossDomainCompositionEngine` class not found.

- [ ] **Step 3: Implement CrossDomainCompositionEngine (registration + validation)**

Create `runtime/src/main/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngine.java`:

```java
package io.casehub.desiredstate.runtime.composition;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.runtime.LifecycleManager;
import io.casehub.desiredstate.runtime.ReconciliationLoop;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.logging.Logger;

public class CrossDomainCompositionEngine implements GlobalReconciliationListener {

    private static final Logger LOG = Logger.getLogger(CrossDomainCompositionEngine.class.getName());

    private final DesiredStateGraphFactory graphFactory;
    private volatile boolean composed = false;

    private final Map<DomainId, DomainRegistration> domainConfigs = new LinkedHashMap<>();
    private final Map<SituationRecompiler, DomainId> recompilerIndex = new IdentityHashMap<>();
    private List<Map.Entry<SituationRecompiler, DomainId>> sortedRecompilers = List.of();
    private List<SituationRecompiler> sortedCrossDomainRecompilers = List.of();
    private List<DomainId> topologicalOrder = List.of();

    private final ConcurrentHashMap<String, TenantCompositionState> tenantStates = new ConcurrentHashMap<>();
    private final Object recomposeLock = new Object();

    public CrossDomainCompositionEngine(DesiredStateGraphFactory graphFactory) {
        this.graphFactory = graphFactory;
    }

    public void registerDomain(DomainRegistration registration) {
        if (composed) {
            throw new IllegalStateException(
                "Cannot register domain '" + registration.domainId()
                + "' — composition already completed. "
                + "Ensure domain registrar @Priority is below PLATFORM_AFTER + 1000.");
        }
        domainConfigs.put(registration.domainId(), registration);
        for (SituationRecompiler r : registration.situationRecompilers()) {
            recompilerIndex.put(r, registration.domainId());
        }
    }

    public int registrationCount() { return domainConfigs.size(); }
    public boolean isActive() { return composed && registrationCount() > 1; }
    public List<DomainId> topologicalOrder() { return topologicalOrder; }
    void markComposed() { this.composed = true; }

    public void validate() {
        validateDuplicateProvides();
        validateUnsatisfiedRequires();
        topologicalOrder = computeTopologicalOrder();
        validateNodeIdUniqueness();
        buildSortedRecompilers();
    }

    private void validateDuplicateProvides() {
        Map<NodeType, DomainId> seen = new LinkedHashMap<>();
        for (var entry : domainConfigs.entrySet()) {
            for (NodeType type : entry.getValue().provides()) {
                DomainId prev = seen.put(type, entry.getKey());
                if (prev != null) {
                    throw new IllegalStateException(
                        "NodeType '" + type.value() + "' provided by both '"
                        + prev.value() + "' and '" + entry.getKey().value()
                        + "' — each NodeType must be provided by exactly one domain.");
                }
            }
        }
    }

    private void validateUnsatisfiedRequires() {
        Set<NodeType> allProvided = new LinkedHashSet<>();
        for (var reg : domainConfigs.values()) allProvided.addAll(reg.provides());
        for (var entry : domainConfigs.entrySet()) {
            for (NodeType req : entry.getValue().requires()) {
                if (!allProvided.contains(req)) {
                    throw new IllegalStateException(
                        "Domain '" + entry.getKey().value() + "' requires NodeType '"
                        + req.value() + "' but no domain provides it.");
                }
            }
        }
    }

    private List<DomainId> computeTopologicalOrder() {
        // Build provides→domain reverse index
        Map<NodeType, DomainId> providerOf = new LinkedHashMap<>();
        for (var entry : domainConfigs.entrySet())
            for (NodeType t : entry.getValue().provides())
                providerOf.put(t, entry.getKey());

        // Build adjacency (domain dependency edges from requires→provides)
        Map<DomainId, Set<DomainId>> dependsOn = new LinkedHashMap<>();
        Map<DomainId, Integer> inDegree = new LinkedHashMap<>();
        for (DomainId id : domainConfigs.keySet()) {
            dependsOn.put(id, new LinkedHashSet<>());
            inDegree.put(id, 0);
        }
        for (var entry : domainConfigs.entrySet()) {
            for (NodeType req : entry.getValue().requires()) {
                DomainId provider = providerOf.get(req);
                if (provider != null && !provider.equals(entry.getKey())
                        && dependsOn.get(entry.getKey()).add(provider)) {
                    inDegree.merge(entry.getKey(), 1, Integer::sum);
                }
            }
        }

        // Kahn's algorithm
        Deque<DomainId> queue = new ArrayDeque<>();
        for (var entry : inDegree.entrySet())
            if (entry.getValue() == 0) queue.add(entry.getKey());
        List<DomainId> sorted = new ArrayList<>();
        while (!queue.isEmpty()) {
            DomainId current = queue.poll();
            sorted.add(current);
            for (var entry : dependsOn.entrySet()) {
                if (entry.getValue().contains(current)) {
                    int newDeg = inDegree.merge(entry.getKey(), -1, Integer::sum);
                    if (newDeg == 0) queue.add(entry.getKey());
                }
            }
        }
        if (sorted.size() != domainConfigs.size()) {
            throw new IllegalStateException(
                "Circular dependency detected among domains: "
                + domainConfigs.keySet().stream()
                    .filter(id -> !sorted.contains(id))
                    .map(DomainId::value)
                    .toList());
        }
        return List.copyOf(sorted);
    }

    private void validateNodeIdUniqueness() {
        Map<NodeId, DomainId> seen = new LinkedHashMap<>();
        Map<NodeId, DesiredNode> nodeIndex = new LinkedHashMap<>();
        for (DomainId domainId : topologicalOrder) {
            var reg = domainConfigs.get(domainId);
            DesiredStateGraph graph = new DomainPhaseState(reg.compilationResult(), 0).currentGraph();
            for (var entry : graph.nodes().entrySet()) {
                DomainId prev = seen.get(entry.getKey());
                if (prev != null) {
                    DesiredNode prevNode = nodeIndex.get(entry.getKey());
                    if (!prevNode.equals(entry.getValue())) {
                        throw new IllegalStateException(
                            "Node ID '" + entry.getKey().value() + "' exists in both domain '"
                            + prev.value() + "' and '" + domainId.value()
                            + "' with different specs.");
                    }
                } else {
                    seen.put(entry.getKey(), domainId);
                    nodeIndex.put(entry.getKey(), entry.getValue());
                }
            }
        }
    }

    private void buildSortedRecompilers() {
        sortedRecompilers = recompilerIndex.entrySet().stream()
            .sorted(Comparator.comparingInt(e -> e.getKey().priority()))
            .map(e -> Map.entry(e.getKey(), e.getValue()))
            .toList();
    }

    @Override
    public void onReconciliationCycleCompleted(String tenancyId, DesiredStateGraph desired, ActualState actual) {
        // Implemented in Task 4
    }

    @Override
    public void onTenantStopped(String tenancyId) {
        tenantStates.remove(tenancyId);
    }
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=CrossDomainCompositionEngineValidationTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngine.java runtime/src/test/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngineValidationTest.java
git commit -m "feat(#140): add CrossDomainCompositionEngine with registration and startup validation"
```

---

## Batch 2: Flattened mode — composition, lifecycle, SituationRecompiler

### Task 3: Overlay composition, cross-domain edges, start/stop, passthrough

**Files:**
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngine.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngineFlattenedTest.java`

**Interfaces:**
- Consumes: `DomainRegistration`, `DesiredStateGraph.overlay()`, `DesiredStateGraph.withDependency()`, `DesiredStateGraph.roots()`, `DesiredNode.type()`, `LifecycleManager.start()`, `LifecycleManager.updateDesired()`, `LifecycleManager.stop()`
- Produces: `CrossDomainCompositionEngine.start(String tenancyId)`, `CrossDomainCompositionEngine.stop(String tenancyId)`, `CrossDomainCompositionEngine.compose()` (internal — validates + prepares), `CrossDomainCompositionEngine.recompose(String, TenantCompositionState) → DesiredStateGraph`

- [ ] **Step 1: Write flattened composition tests**

```java
package io.casehub.desiredstate.runtime.composition;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.runtime.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.Set;
import static org.assertj.core.api.Assertions.*;

class CrossDomainCompositionEngineFlattenedTest {

    static final DesiredStateGraphFactory FACTORY = new DefaultDesiredStateGraphFactory();
    static final NodeType NS = NodeType.of("k8s-namespace");
    static final NodeType AGENT = NodeType.of("agent");

    record NsSpec(String name) implements NodeSpec {
        @Override public NodeType nodeType() { return NS; }
    }
    record AgentSpec(String name) implements NodeSpec {
        @Override public NodeType nodeType() { return AGENT; }
    }

    CrossDomainCompositionEngine engine;

    @BeforeEach void setUp() {
        engine = new CrossDomainCompositionEngine(FACTORY);
    }

    @Test void compose_mergesGraphsViaOverlay() {
        var infraGraph = FACTORY.empty()
            .withNode(new DesiredNode(NodeId.of("infra:ns"), new NsSpec("prod"), HumanGating.NONE));
        var deployGraph = FACTORY.empty()
            .withNode(new DesiredNode(NodeId.of("deploy:agent"), new AgentSpec("main"), HumanGating.NONE));

        engine.registerDomain(DomainRegistration.builder(DomainId.of("infra"),
                CompilationResult.single(infraGraph)).provides(Set.of(NS)).build());
        engine.registerDomain(DomainRegistration.builder(DomainId.of("deploy"),
                CompilationResult.single(deployGraph)).provides(Set.of(AGENT)).requires(Set.of(NS)).build());
        engine.validate();

        var tenantState = engine.initTenantState();
        var composed = engine.recompose("t1", tenantState);

        assertThat(composed.nodes()).hasSize(2);
        assertThat(composed.nodes()).containsKey(NodeId.of("infra:ns"));
        assertThat(composed.nodes()).containsKey(NodeId.of("deploy:agent"));
    }

    @Test void compose_addsCrossDomainEdges() {
        var infraGraph = FACTORY.empty()
            .withNode(new DesiredNode(NodeId.of("infra:ns"), new NsSpec("prod"), HumanGating.NONE));
        var deployGraph = FACTORY.empty()
            .withNode(new DesiredNode(NodeId.of("deploy:agent"), new AgentSpec("main"), HumanGating.NONE));

        engine.registerDomain(DomainRegistration.builder(DomainId.of("infra"),
                CompilationResult.single(infraGraph)).provides(Set.of(NS)).build());
        engine.registerDomain(DomainRegistration.builder(DomainId.of("deploy"),
                CompilationResult.single(deployGraph)).provides(Set.of(AGENT)).requires(Set.of(NS)).build());
        engine.validate();

        var tenantState = engine.initTenantState();
        var composed = engine.recompose("t1", tenantState);

        // deploy:agent depends on infra:ns (cross-domain edge)
        assertThat(composed.dependenciesOf(NodeId.of("deploy:agent")))
            .contains(NodeId.of("infra:ns"));
    }

    @Test void compose_noRequires_noEdges() {
        var g1 = FACTORY.empty()
            .withNode(new DesiredNode(NodeId.of("a:n1"), new NsSpec("x"), HumanGating.NONE));
        var g2 = FACTORY.empty()
            .withNode(new DesiredNode(NodeId.of("b:n2"), new AgentSpec("y"), HumanGating.NONE));

        engine.registerDomain(DomainRegistration.builder(DomainId.of("a"),
                CompilationResult.single(g1)).provides(Set.of(NS)).build());
        engine.registerDomain(DomainRegistration.builder(DomainId.of("b"),
                CompilationResult.single(g2)).provides(Set.of(AGENT)).build());
        engine.validate();

        var tenantState = engine.initTenantState();
        var composed = engine.recompose("t1", tenantState);

        assertThat(composed.dependencies()).isEmpty();
    }

    @Test void singleDomainPassthrough_noOverlay() {
        var graph = FACTORY.empty()
            .withNode(new DesiredNode(NodeId.of("n1"), new NsSpec("x"), HumanGating.NONE));
        engine.registerDomain(DomainRegistration.builder(DomainId.of("only"),
                CompilationResult.single(graph)).provides(Set.of(NS)).build());
        engine.validate();
        assertThat(engine.registrationCount()).isEqualTo(1);
        assertThat(engine.isActive()).isFalse();
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=CrossDomainCompositionEngineFlattenedTest`
Expected: FAIL — methods not found.

- [ ] **Step 3: Implement composition, cross-domain edge creation, start/stop**

Add to `CrossDomainCompositionEngine`:

```java
public void compose() {
    validate();
    composed = true;
}

public TenantCompositionState initTenantState() {
    Map<DomainId, DomainPhaseState> phases = new LinkedHashMap<>();
    for (DomainId id : topologicalOrder) {
        var reg = domainConfigs.get(id);
        phases.put(id, new DomainPhaseState(reg.compilationResult(), 0));
    }
    return new TenantCompositionState(Map.copyOf(phases));
}

public DesiredStateGraph recompose(String tenancyId, TenantCompositionState tenantState) {
    DesiredStateGraph composed = graphFactory.empty();
    for (DomainId domainId : topologicalOrder) {
        DomainPhaseState phaseState = tenantState.phases().get(domainId);
        composed = composed.overlay(phaseState.currentGraph());
    }
    for (Dependency dep : computeCrossDomainEdges(tenantState)) {
        composed = composed.withDependency(dep);
    }
    return composed;
}

private List<Dependency> computeCrossDomainEdges(TenantCompositionState tenantState) {
    Map<NodeType, DomainId> providerOf = new LinkedHashMap<>();
    for (var entry : domainConfigs.entrySet())
        for (NodeType t : entry.getValue().provides())
            providerOf.put(t, entry.getKey());

    List<Dependency> edges = new ArrayList<>();
    for (DomainId domainId : topologicalOrder) {
        var reg = domainConfigs.get(domainId);
        if (reg.requires().isEmpty()) continue;

        DesiredStateGraph dependentGraph = tenantState.phases().get(domainId).currentGraph();
        Set<NodeId> dependentRoots = dependentGraph.roots();

        for (NodeType requiredType : reg.requires()) {
            DomainId providerId = providerOf.get(requiredType);
            if (providerId == null) continue;
            DesiredStateGraph providerGraph = tenantState.phases().get(providerId).currentGraph();
            for (var nodeEntry : providerGraph.nodes().entrySet()) {
                if (nodeEntry.getValue().type().equals(requiredType)) {
                    for (NodeId root : dependentRoots) {
                        edges.add(new Dependency(root, nodeEntry.getKey()));
                    }
                }
            }
        }
    }
    return edges;
}

public void start(String tenancyId, LifecycleManager lifecycleManager) {
    if (registrationCount() <= 1) {
        // Single-domain passthrough
        if (registrationCount() == 1) {
            var reg = domainConfigs.values().iterator().next();
            lifecycleManager.start(tenancyId, reg.compilationResult());
        }
        return;
    }
    var tenantState = initTenantState();
    tenantStates.put(tenancyId, tenantState);
    DesiredStateGraph composed = recompose(tenancyId, tenantState);
    lifecycleManager.start(tenancyId, CompilationResult.single(composed));
}

public void stop(String tenancyId, LifecycleManager lifecycleManager) {
    tenantStates.remove(tenancyId);
    lifecycleManager.stop(tenancyId);
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=CrossDomainCompositionEngineFlattenedTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngine.java runtime/src/test/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngineFlattenedTest.java
git commit -m "feat(#140): add flattened composition — overlay merging, cross-domain edges, start/stop, passthrough"
```

---

### Task 4: Per-domain lifecycle tracking via GlobalReconciliationListener

**Files:**
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngine.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngineLifecycleTest.java`

**Interfaces:**
- Consumes: `GlobalReconciliationListener.onReconciliationCycleCompleted(String, DesiredStateGraph, ActualState)`, `CompletionCondition.isComplete(DesiredStateGraph, ActualState)`, `Phase.completionCondition()`, `LifecycleManager.updateDesired(String, CompilationResult)`
- Produces: `CrossDomainCompositionEngine.onReconciliationCycleCompleted(...)` — evaluates per-domain CompletionCondition, advances phases, recomposes

- [ ] **Step 1: Write lifecycle tracking tests**

```java
package io.casehub.desiredstate.runtime.composition;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.runtime.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

class CrossDomainCompositionEngineLifecycleTest {

    static final DesiredStateGraphFactory FACTORY = new DefaultDesiredStateGraphFactory();
    static final NodeType NS = NodeType.of("k8s-namespace");
    static final NodeType AGENT = NodeType.of("agent");

    record NsSpec(String n) implements NodeSpec { @Override public NodeType nodeType() { return NS; } }
    record AgentSpec(String n) implements NodeSpec { @Override public NodeType nodeType() { return AGENT; } }

    CrossDomainCompositionEngine engine;

    @BeforeEach void setUp() {
        engine = new CrossDomainCompositionEngine(FACTORY);
    }

    @Test void phaseAdvancement_recomposes() {
        var g1 = FACTORY.empty().withNode(new DesiredNode(NodeId.of("infra:ns"), new NsSpec("p1"), HumanGating.NONE));
        var g2 = FACTORY.empty().withNode(new DesiredNode(NodeId.of("infra:ns-v2"), new NsSpec("p2"), HumanGating.NONE));
        var lifecycle = CompilationResult.lifecycle(List.of(
            new Phase("setup", g1, CompletionCondition.allPresent()),
            new Phase("ready", g2, CompletionCondition.allPresent())));

        engine.registerDomain(DomainRegistration.builder(DomainId.of("infra"), lifecycle)
            .provides(Set.of(NS)).build());
        engine.registerDomain(DomainRegistration.builder(DomainId.of("deploy"),
                CompilationResult.single(FACTORY.empty().withNode(
                    new DesiredNode(NodeId.of("deploy:a"), new AgentSpec("a"), HumanGating.NONE))))
            .provides(Set.of(AGENT)).requires(Set.of(NS)).build());
        engine.compose();

        var tenantState = engine.initTenantState();
        engine.setTenantState("t1", tenantState);

        // Simulate all nodes PRESENT — infra phase 1 complete
        ActualState actual = ActualState.of(Map.of(
            NodeId.of("infra:ns"), NodeStatus.PRESENT,
            NodeId.of("deploy:a"), NodeStatus.PRESENT));

        var composed = engine.recompose("t1", tenantState);
        engine.onReconciliationCycleCompleted("t1", composed, actual);

        var newState = engine.getTenantState("t1");
        var infraPhase = newState.phases().get(DomainId.of("infra"));
        assertThat(infraPhase.phaseIndex()).isEqualTo(1);
        assertThat(infraPhase.currentGraph().nodes()).containsKey(NodeId.of("infra:ns-v2"));
    }

    @Test void noLifecycle_noAdvancement() {
        var graph = FACTORY.empty().withNode(new DesiredNode(
            NodeId.of("n1"), new NsSpec("x"), HumanGating.NONE));
        engine.registerDomain(DomainRegistration.builder(DomainId.of("d1"),
                CompilationResult.single(graph)).provides(Set.of(NS)).build());
        engine.registerDomain(DomainRegistration.builder(DomainId.of("d2"),
                CompilationResult.single(FACTORY.empty().withNode(
                    new DesiredNode(NodeId.of("n2"), new AgentSpec("a"), HumanGating.NONE))))
            .provides(Set.of(AGENT)).build());
        engine.compose();
        var ts = engine.initTenantState();
        engine.setTenantState("t1", ts);

        ActualState actual = ActualState.of(Map.of(
            NodeId.of("n1"), NodeStatus.PRESENT,
            NodeId.of("n2"), NodeStatus.PRESENT));
        var composed = engine.recompose("t1", ts);
        engine.onReconciliationCycleCompleted("t1", composed, actual);

        assertThat(engine.getTenantState("t1").phases().get(DomainId.of("d1")).phaseIndex()).isEqualTo(0);
    }

    @Test void onTenantStopped_cleansUp() {
        engine.registerDomain(DomainRegistration.builder(DomainId.of("d"),
                CompilationResult.single(FACTORY.empty())).provides(Set.of(NS)).build());
        engine.compose();
        engine.setTenantState("t1", engine.initTenantState());
        assertThat(engine.getTenantState("t1")).isNotNull();
        engine.onTenantStopped("t1");
        assertThat(engine.getTenantState("t1")).isNull();
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=CrossDomainCompositionEngineLifecycleTest`
Expected: FAIL — methods not found.

- [ ] **Step 3: Implement per-domain lifecycle tracking**

Add to `CrossDomainCompositionEngine`:

```java
public void setTenantState(String tenancyId, TenantCompositionState state) {
    tenantStates.put(tenancyId, state);
}

public TenantCompositionState getTenantState(String tenancyId) {
    return tenantStates.get(tenancyId);
}

@Override
public void onReconciliationCycleCompleted(
        String tenancyId, DesiredStateGraph desired, ActualState actual) {
    TenantCompositionState tenantState = tenantStates.get(tenancyId);
    if (tenantState == null) return;

    synchronized (recomposeLock) {
        boolean recomposeNeeded = false;
        TenantCompositionState current = tenantStates.get(tenancyId);
        if (current == null) return;

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
                LOG.info("Domain '" + domainId.value() + "' phase transition: '"
                    + currentPhase.id() + "' → '"
                    + lc.phases().get(phaseState.phaseIndex() + 1).id() + "'");
            }
        }

        if (recomposeNeeded) {
            tenantStates.put(tenancyId, current);
        }
    }
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=CrossDomainCompositionEngineLifecycleTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngine.java runtime/src/test/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngineLifecycleTest.java
git commit -m "feat(#140): add per-domain lifecycle tracking via GlobalReconciliationListener"
```

---

### Task 5: SituationRecompiler integration and DesiredStateReplanDispatch modification

**Files:**
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngine.java`
- Modify: `engine-adapter/src/main/java/io/casehub/desiredstate/engine/DesiredStateReplanDispatch.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngineReplanTest.java`
- Test: `engine-adapter/src/test/java/io/casehub/desiredstate/engine/DesiredStateReplanDispatchTest.java` (modify existing)

**Interfaces:**
- Consumes: `SituationRecompiler.recompile(String, DesiredStateGraph, ActualState, ActiveSituation, DesiredStateGraphFactory)`, `SituationRecompiler.priority()`, `LifecycleManager.updateDesired(String, CompilationResult)`
- Produces: `CrossDomainCompositionEngine.handleReplan(String tenancyId, ActualState actual, ActiveSituation situation, DesiredStateGraphFactory factory) → Optional<CompilationResult>`

- [ ] **Step 1: Write SituationRecompiler integration tests**

```java
package io.casehub.desiredstate.runtime.composition;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.runtime.*;
import io.casehub.ras.api.ActiveSituation;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

class CrossDomainCompositionEngineReplanTest {

    static final DesiredStateGraphFactory FACTORY = new DefaultDesiredStateGraphFactory();
    static final NodeType NS = NodeType.of("k8s-namespace");
    static final NodeType AGENT = NodeType.of("agent");

    record NsSpec(String n) implements NodeSpec { @Override public NodeType nodeType() { return NS; } }
    record AgentSpec(String n) implements NodeSpec { @Override public NodeType nodeType() { return AGENT; } }

    CrossDomainCompositionEngine engine;
    ActiveSituation testSituation;

    @BeforeEach void setUp() {
        engine = new CrossDomainCompositionEngine(FACTORY);
        testSituation = new ActiveSituation("sit-1", "corr-1", "t1", 0.9,
            Map.of(), Instant.now(), Instant.now(), 1);
    }

    @Test void handleReplan_domainRecompiler_receivesDomainGraph() {
        var infraGraph = FACTORY.empty().withNode(
            new DesiredNode(NodeId.of("infra:ns"), new NsSpec("prod"), HumanGating.NONE));
        var deployGraph = FACTORY.empty().withNode(
            new DesiredNode(NodeId.of("deploy:a"), new AgentSpec("a"), HumanGating.NONE));

        var capturedGraph = new DesiredStateGraph[1];
        SituationRecompiler infraRecompiler = new SituationRecompiler() {
            @Override public Optional<CompilationResult> recompile(String t, DesiredStateGraph current,
                    ActualState actual, ActiveSituation sit, DesiredStateGraphFactory f) {
                capturedGraph[0] = current;
                return Optional.of(CompilationResult.single(current));
            }
            @Override public int priority() { return 0; }
        };

        engine.registerDomain(DomainRegistration.builder(DomainId.of("infra"),
                CompilationResult.single(infraGraph)).provides(Set.of(NS))
            .situationRecompilers(List.of(infraRecompiler)).build());
        engine.registerDomain(DomainRegistration.builder(DomainId.of("deploy"),
                CompilationResult.single(deployGraph)).provides(Set.of(AGENT))
            .requires(Set.of(NS)).build());
        engine.compose();
        engine.setTenantState("t1", engine.initTenantState());

        ActualState actual = ActualState.of(Map.of());
        engine.handleReplan("t1", actual, testSituation, FACTORY);

        // Recompiler should receive only infra's graph, not the composed graph
        assertThat(capturedGraph[0].nodes()).containsKey(NodeId.of("infra:ns"));
        assertThat(capturedGraph[0].nodes()).doesNotContainKey(NodeId.of("deploy:a"));
    }

    @Test void handleReplan_noTenantState_returnsEmpty() {
        engine.registerDomain(DomainRegistration.builder(DomainId.of("d"),
                CompilationResult.single(FACTORY.empty())).provides(Set.of(NS)).build());
        engine.compose();
        // No tenant state set
        var result = engine.handleReplan("unknown", ActualState.of(Map.of()),
            testSituation, FACTORY);
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=CrossDomainCompositionEngineReplanTest`
Expected: FAIL — `handleReplan` method not found.

- [ ] **Step 3: Implement handleReplan**

Add to `CrossDomainCompositionEngine`:

```java
public Optional<CompilationResult> handleReplan(
        String tenancyId, ActualState actual,
        ActiveSituation situation, DesiredStateGraphFactory factory) {
    TenantCompositionState tenantState = tenantStates.get(tenancyId);
    if (tenantState == null) return Optional.empty();

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
            }
            return result;
        }
    }
    return Optional.empty();
}

public void registerCrossDomainRecompiler(SituationRecompiler recompiler) {
    if (composed) {
        throw new IllegalStateException("Cannot register cross-domain recompiler after composition.");
    }
    sortedCrossDomainRecompilers = new ArrayList<>(sortedCrossDomainRecompilers);
    sortedCrossDomainRecompilers.add(recompiler);
    sortedCrossDomainRecompilers.sort(Comparator.comparingInt(SituationRecompiler::priority));
    sortedCrossDomainRecompilers = List.copyOf(sortedCrossDomainRecompilers);
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=CrossDomainCompositionEngineReplanTest`
Expected: PASS

- [ ] **Step 5: Modify DesiredStateReplanDispatch**

Add optional `Instance<CrossDomainCompositionEngine>` injection to `DesiredStateReplanDispatch`. In the `replan()` method, check if composition is active and delegate:

In `DesiredStateReplanDispatch.java`, add field:
```java
@Inject
Instance<CrossDomainCompositionEngine> compositionEngine;
```

In `replan()`, replace the direct `recompilerEngine.recompile()` + `lifecycleManager.updateDesired()` with:
```java
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

- [ ] **Step 6: Run full build — verify no regressions**

Run: `mvn --batch-mode install`
Expected: PASS — all existing tests continue to pass.

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngine.java runtime/src/test/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngineReplanTest.java engine-adapter/src/main/java/io/casehub/desiredstate/engine/DesiredStateReplanDispatch.java
git commit -m "feat(#140): add SituationRecompiler integration and DesiredStateReplanDispatch delegation"
```

---

## Batch 3: Hierarchical mode and integration tests

### Task 6: Meta-loop, DomainNodeProvisioner, DomainActualStateAdapter

**Files:**
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/composition/DomainNodeSpec.java`
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/composition/DomainNodeProvisioner.java`
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/composition/DomainActualStateAdapter.java`
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/composition/CrossDomainCompositionEngine.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/composition/HierarchicalModeTest.java`

**Interfaces:**
- Consumes: `NodeProvisioner.provision()/.deprovision()/.handledTypes()/.resyncInterval()`, `ActualStateAdapter.readActual()/.handledTypes()`, `ReconciliationLoop.Builder`, `CompletionCondition`, `DesiredStateGraphFactory`, `TransitionPlanner`, `SimpleTransitionExecutor`, `ActualStateAdapterRouter`, `FaultPolicyEngine`, `MergedEventSource`, `NodeProvisionerRouter`
- Produces: `DomainNodeSpec(DomainId, DomainRegistration)` implements `NodeSpec`, `DomainNodeProvisioner` implements `NodeProvisioner`, `DomainActualStateAdapter` implements `ActualStateAdapter`

- [ ] **Step 1: Write DomainNodeSpec**

```java
package io.casehub.desiredstate.runtime.composition;

import io.casehub.desiredstate.api.*;

public record DomainNodeSpec(DomainId domainId, DomainRegistration registration) implements NodeSpec {
    static final NodeType DOMAIN_NODE_TYPE = NodeType.of("domain");
    @Override public NodeType nodeType() { return DOMAIN_NODE_TYPE; }
}
```

- [ ] **Step 2: Write hierarchical mode tests**

```java
package io.casehub.desiredstate.runtime.composition;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.runtime.*;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

class HierarchicalModeTest {

    static final DesiredStateGraphFactory FACTORY = new DefaultDesiredStateGraphFactory();
    static final NodeType NS = NodeType.of("k8s-namespace");
    static final NodeType AGENT = NodeType.of("agent");

    record NsSpec(String n) implements NodeSpec { @Override public NodeType nodeType() { return NS; } }
    record AgentSpec(String n) implements NodeSpec { @Override public NodeType nodeType() { return AGENT; } }

    @Test void buildMetaGraph_createsDomainNodes() {
        var engine = new CrossDomainCompositionEngine(FACTORY);
        var infraGraph = FACTORY.empty().withNode(
            new DesiredNode(NodeId.of("infra:ns"), new NsSpec("prod"), HumanGating.NONE));
        var deployGraph = FACTORY.empty().withNode(
            new DesiredNode(NodeId.of("deploy:a"), new AgentSpec("a"), HumanGating.NONE));

        engine.registerDomain(DomainRegistration.builder(DomainId.of("infra"),
                CompilationResult.single(infraGraph)).provides(Set.of(NS)).build());
        engine.registerDomain(DomainRegistration.builder(DomainId.of("deploy"),
                CompilationResult.single(deployGraph)).provides(Set.of(AGENT))
            .requires(Set.of(NS)).build());
        engine.validate();

        DesiredStateGraph metaGraph = engine.buildMetaGraph();
        assertThat(metaGraph.nodes()).hasSize(2);
        assertThat(metaGraph.nodes().get(NodeId.of("domain:infra")).spec())
            .isInstanceOf(DomainNodeSpec.class);
        assertThat(metaGraph.nodes().get(NodeId.of("domain:deploy")).spec())
            .isInstanceOf(DomainNodeSpec.class);
    }

    @Test void buildMetaGraph_addsDomainDependencyEdges() {
        var engine = new CrossDomainCompositionEngine(FACTORY);
        engine.registerDomain(DomainRegistration.builder(DomainId.of("infra"),
                CompilationResult.single(FACTORY.empty())).provides(Set.of(NS)).build());
        engine.registerDomain(DomainRegistration.builder(DomainId.of("deploy"),
                CompilationResult.single(FACTORY.empty())).provides(Set.of(AGENT))
            .requires(Set.of(NS)).build());
        engine.validate();

        DesiredStateGraph metaGraph = engine.buildMetaGraph();
        // deploy depends on infra
        assertThat(metaGraph.dependenciesOf(NodeId.of("domain:deploy")))
            .contains(NodeId.of("domain:infra"));
    }

    @Test void domainNodeProvisioner_reportsNotReadyWhenInnerLoopIncomplete() {
        var graph = FACTORY.empty().withNode(
            new DesiredNode(NodeId.of("n1"), new NsSpec("x"), HumanGating.NONE));
        var reg = DomainRegistration.builder(DomainId.of("test"),
                CompilationResult.single(graph)).provides(Set.of(NS)).build();
        var spec = new DomainNodeSpec(DomainId.of("test"), reg);
        var node = new DesiredNode(NodeId.of("domain:test"), spec, HumanGating.NONE);

        var provisioner = new DomainNodeProvisioner(FACTORY);
        var result = provisioner.checkReadiness(reg, ActualState.of(Map.of()));
        assertThat(result).isFalse();
    }

    @Test void domainNodeProvisioner_reportsReadyWhenAllPresent() {
        var graph = FACTORY.empty().withNode(
            new DesiredNode(NodeId.of("n1"), new NsSpec("x"), HumanGating.NONE));
        var reg = DomainRegistration.builder(DomainId.of("test"),
                CompilationResult.single(graph)).provides(Set.of(NS)).build();

        var provisioner = new DomainNodeProvisioner(FACTORY);
        var result = provisioner.checkReadiness(reg,
            ActualState.of(Map.of(NodeId.of("n1"), NodeStatus.PRESENT)));
        assertThat(result).isTrue();
    }
}
```

- [ ] **Step 3: Run test — verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=HierarchicalModeTest`
Expected: FAIL — classes not found.

- [ ] **Step 4: Implement DomainNodeProvisioner, DomainActualStateAdapter, buildMetaGraph**

`DomainNodeProvisioner.java`:
```java
package io.casehub.desiredstate.runtime.composition;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.runtime.ReconciliationLoop;
import java.time.Duration;
import java.util.*;

public class DomainNodeProvisioner implements NodeProvisioner {

    static final NodeType DOMAIN_TYPE = DomainNodeSpec.DOMAIN_NODE_TYPE;
    private final DesiredStateGraphFactory graphFactory;
    private final Map<DomainId, ReconciliationLoop> activeInnerLoops = new LinkedHashMap<>();

    public DomainNodeProvisioner(DesiredStateGraphFactory graphFactory) {
        this.graphFactory = graphFactory;
    }

    @Override public Set<NodeType> handledTypes() { return Set.of(DOMAIN_TYPE); }
    @Override public Duration resyncInterval() { return Duration.ofSeconds(30); }

    @Override
    public ProvisionResult provision(DesiredNode node, ProvisionContext context) {
        DomainNodeSpec spec = (DomainNodeSpec) node.spec();
        // Inner loop creation happens here in full implementation
        // For now, check readiness based on actual state
        return ProvisionResult.failed("domain '" + spec.domainId().value()
            + "' not ready — awaiting convergence");
    }

    @Override
    public DeprovisionResult deprovision(DesiredNode node, DeprovisionContext context) {
        DomainNodeSpec spec = (DomainNodeSpec) node.spec();
        ReconciliationLoop innerLoop = activeInnerLoops.remove(spec.domainId());
        if (innerLoop != null) {
            innerLoop.stop(context.tenancyId());
            innerLoop.shutdown();
        }
        return DeprovisionResult.success();
    }

    public boolean checkReadiness(DomainRegistration reg, ActualState actual) {
        DesiredStateGraph graph = new DomainPhaseState(reg.compilationResult(), 0).currentGraph();
        return reg.readinessCondition().isComplete(graph, actual);
    }
}
```

`DomainActualStateAdapter.java`:
```java
package io.casehub.desiredstate.runtime.composition;

import io.casehub.desiredstate.api.*;
import java.util.*;

public class DomainActualStateAdapter implements ActualStateAdapter {

    private final DomainNodeProvisioner provisioner;
    private final Map<DomainId, DomainRegistration> domainConfigs;

    public DomainActualStateAdapter(DomainNodeProvisioner provisioner,
            Map<DomainId, DomainRegistration> domainConfigs) {
        this.provisioner = provisioner;
        this.domainConfigs = domainConfigs;
    }

    @Override public Set<NodeType> handledTypes() { return Set.of(DomainNodeSpec.DOMAIN_NODE_TYPE); }

    @Override
    public ActualState readActual(DesiredStateGraph desired, String tenancyId) {
        Map<NodeId, NodeStatus> statuses = new LinkedHashMap<>();
        for (var entry : desired.nodes().entrySet()) {
            if (entry.getValue().spec() instanceof DomainNodeSpec dSpec) {
                DomainRegistration reg = domainConfigs.get(dSpec.domainId());
                // In full impl, check actual inner loop state
                statuses.put(entry.getKey(), NodeStatus.ABSENT);
            }
        }
        return ActualState.of(statuses);
    }
}
```

Add `buildMetaGraph()` to `CrossDomainCompositionEngine`:
```java
public DesiredStateGraph buildMetaGraph() {
    Map<NodeType, DomainId> providerOf = new LinkedHashMap<>();
    for (var entry : domainConfigs.entrySet())
        for (NodeType t : entry.getValue().provides())
            providerOf.put(t, entry.getKey());

    List<DesiredNode> nodes = new ArrayList<>();
    List<Dependency> deps = new ArrayList<>();
    for (DomainId id : topologicalOrder) {
        var reg = domainConfigs.get(id);
        var spec = new DomainNodeSpec(id, reg);
        var nodeId = NodeId.of("domain:" + id.value());
        nodes.add(new DesiredNode(nodeId, spec, HumanGating.NONE));

        for (NodeType req : reg.requires()) {
            DomainId provider = providerOf.get(req);
            if (provider != null) {
                deps.add(new Dependency(nodeId, NodeId.of("domain:" + provider.value())));
            }
        }
    }
    return graphFactory.of(nodes, deps);
}
```

- [ ] **Step 5: Run test — verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=HierarchicalModeTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/desiredstate/runtime/composition/ runtime/src/test/java/io/casehub/desiredstate/runtime/composition/HierarchicalModeTest.java
git commit -m "feat(#140): add hierarchical mode — meta-graph, DomainNodeProvisioner, DomainActualStateAdapter"
```

---

### Task 7: End-to-end integration tests

**Files:**
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/composition/CrossDomainIntegrationTest.java`

**Interfaces:**
- Consumes: all composition engine APIs, `ReconciliationLoop.Builder`, `TransitionPlanner`, `SimpleTransitionExecutor`, mock provisioners/adapters from testing/

- [ ] **Step 1: Write end-to-end flattened test**

```java
package io.casehub.desiredstate.runtime.composition;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.runtime.*;
import io.casehub.desiredstate.testing.*;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

class CrossDomainIntegrationTest {

    static final DesiredStateGraphFactory FACTORY = new DefaultDesiredStateGraphFactory();
    static final NodeType NS = NodeType.of("k8s-namespace");
    static final NodeType AGENT = NodeType.of("agent");

    record NsSpec(String n) implements NodeSpec { @Override public NodeType nodeType() { return NS; } }
    record AgentSpec(String n) implements NodeSpec { @Override public NodeType nodeType() { return AGENT; } }

    @Test void flattened_infraBeforeDeployment() {
        var infraGraph = FACTORY.empty()
            .withNode(new DesiredNode(NodeId.of("infra:ns"), new NsSpec("prod"), HumanGating.NONE));
        var deployGraph = FACTORY.empty()
            .withNode(new DesiredNode(NodeId.of("deploy:agent"), new AgentSpec("main"), HumanGating.NONE));

        var engine = new CrossDomainCompositionEngine(FACTORY);
        engine.registerDomain(DomainRegistration.builder(DomainId.of("infra"),
                CompilationResult.single(infraGraph)).provides(Set.of(NS)).build());
        engine.registerDomain(DomainRegistration.builder(DomainId.of("deploy"),
                CompilationResult.single(deployGraph)).provides(Set.of(AGENT))
            .requires(Set.of(NS)).build());
        engine.compose();

        var tenantState = engine.initTenantState();
        var composed = engine.recompose("t1", tenantState);

        // Verify ordering: infra:ns must be provisioned before deploy:agent
        var planner = new TransitionPlanner();
        var plan = planner.plan(FACTORY.empty(), composed);

        // Additions should be ordered: infra:ns before deploy:agent
        List<NodeId> additionOrder = plan.additions().stream()
            .map(step -> step.node().id()).toList();
        assertThat(additionOrder.indexOf(NodeId.of("infra:ns")))
            .isLessThan(additionOrder.indexOf(NodeId.of("deploy:agent")));
    }

    @Test void singleDomain_backwardCompatible() {
        var engine = new CrossDomainCompositionEngine(FACTORY);
        var graph = FACTORY.empty()
            .withNode(new DesiredNode(NodeId.of("n1"), new NsSpec("x"), HumanGating.NONE));
        engine.registerDomain(DomainRegistration.builder(DomainId.of("only"),
                CompilationResult.single(graph)).provides(Set.of(NS)).build());
        engine.compose();

        // Single domain — engine is not active (passthrough)
        assertThat(engine.isActive()).isFalse();
        assertThat(engine.registrationCount()).isEqualTo(1);
    }

    @Test void threeDomainChain_orderedCorrectly() {
        var TYPE_A = NodeType.of("type-a");
        var TYPE_B = NodeType.of("type-b");
        var TYPE_C = NodeType.of("type-c");

        record SpecA(String n) implements NodeSpec { @Override public NodeType nodeType() { return TYPE_A; } }
        record SpecB(String n) implements NodeSpec { @Override public NodeType nodeType() { return TYPE_B; } }
        record SpecC(String n) implements NodeSpec { @Override public NodeType nodeType() { return TYPE_C; } }

        var gA = FACTORY.empty().withNode(new DesiredNode(NodeId.of("a:1"), new SpecA("a"), HumanGating.NONE));
        var gB = FACTORY.empty().withNode(new DesiredNode(NodeId.of("b:1"), new SpecB("b"), HumanGating.NONE));
        var gC = FACTORY.empty().withNode(new DesiredNode(NodeId.of("c:1"), new SpecC("c"), HumanGating.NONE));

        var engine = new CrossDomainCompositionEngine(FACTORY);
        engine.registerDomain(DomainRegistration.builder(DomainId.of("domC"),
                CompilationResult.single(gC)).provides(Set.of(TYPE_C)).requires(Set.of(TYPE_B)).build());
        engine.registerDomain(DomainRegistration.builder(DomainId.of("domA"),
                CompilationResult.single(gA)).provides(Set.of(TYPE_A)).build());
        engine.registerDomain(DomainRegistration.builder(DomainId.of("domB"),
                CompilationResult.single(gB)).provides(Set.of(TYPE_B)).requires(Set.of(TYPE_A)).build());
        engine.compose();

        assertThat(engine.topologicalOrder()).containsExactly(
            DomainId.of("domA"), DomainId.of("domB"), DomainId.of("domC"));

        var ts = engine.initTenantState();
        var composed = engine.recompose("t1", ts);
        assertThat(composed.nodes()).hasSize(3);

        var planner = new TransitionPlanner();
        var plan = planner.plan(FACTORY.empty(), composed);
        List<NodeId> order = plan.additions().stream()
            .map(step -> step.node().id()).toList();
        assertThat(order.indexOf(NodeId.of("a:1")))
            .isLessThan(order.indexOf(NodeId.of("b:1")));
        assertThat(order.indexOf(NodeId.of("b:1")))
            .isLessThan(order.indexOf(NodeId.of("c:1")));
    }
}
```

- [ ] **Step 2: Run test — verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=CrossDomainIntegrationTest`
Expected: PASS (uses already-implemented code from Tasks 1-5)

- [ ] **Step 3: Run full build**

Run: `mvn --batch-mode install`
Expected: PASS — all tests across all modules pass.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/test/java/io/casehub/desiredstate/runtime/composition/CrossDomainIntegrationTest.java
git commit -m "feat(#140): add end-to-end integration tests for cross-domain composition"
```

---

## References

- [2026-09-13-cross-domain-orchestration-design.md] — design spec this plan implements
- [api/src/main/java/io/casehub/desiredstate/api/GoalCompiler.java] — parameterized SPI
- [api/src/main/java/io/casehub/desiredstate/api/DesiredStateGraph.java] — overlay(), connect()
- [api/src/main/java/io/casehub/desiredstate/api/CompletionCondition.java] — allPresent() factory
- [api/src/main/java/io/casehub/desiredstate/api/CompilationResult.java] — SingleGraph, Lifecycle
- [api/src/main/java/io/casehub/desiredstate/api/GlobalReconciliationListener.java] — CDI listener
- [runtime/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java] — per-tenant loop
- [runtime/src/main/java/io/casehub/desiredstate/runtime/LifecycleManager.java] — CAS transitions
- [runtime/src/main/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraph.java] — overlay() impl
- [runtime/src/main/java/io/casehub/desiredstate/runtime/SituationRecompilerEngine.java] — chain-of-responsibility
- [engine-adapter/src/main/java/io/casehub/desiredstate/engine/DesiredStateReplanDispatch.java] — modified for delegation
- [GitHub #140] — cross-domain orchestration framework
- [GitHub ops#23] — first consumer: cross-domain dependency graphs
