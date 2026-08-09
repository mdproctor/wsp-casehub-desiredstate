# Mutation Helpers + Multi-Tier Escalation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #87 — dependency-aware graph mutation helpers
**Issue group:** #87, #86

**Goal:** Add `GraphMutations` utility for composing node+dependency mutations,
update `addReviewNode` to include dependency edges, and extend `ThresholdFaultPolicy`
with multi-tier escalation using graph-presence detection.

**Architecture:** Three layered changes in `api/` and `runtime/`: (1) a stateless
`GraphMutations` utility, (2) updated `addReviewNode` that uses it, (3) multi-tier
`ThresholdFaultPolicy` that leverages dependency edges for escalation detection via
`dependentsOf()`. Pipeline example migrates from hand-rolled `ProvisionEscalationFaultPolicy`
to a `ThresholdFaultPolicy` configuration.

**Tech Stack:** Java 21, JUnit 5, AssertJ, Quarkus (runtime module), Maven

## Global Constraints

- Pre-release project — no backward compatibility required
- All code in `api/` must be pure Java (no CDI, no Quarkus runtime deps)
- Tests for `api/` types live in `runtime/src/test/` (follows existing convention)
- `mvn --batch-mode install` must pass after each task's commit
- Use `ide-tooling` (`ide_insert_member`, `ide_replace_member`, `ide_edit_member`)
  for structural code edits; use `mcp__intellij-index__*` for code navigation

---

### Task 1: GraphMutations Utility

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/GraphMutations.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/api/GraphMutationsTest.java`

**Interfaces:**
- Consumes: `GraphMutation.AddNode`, `GraphMutation.AddDependency`, `Dependency`, `DesiredNode`, `NodeId`
- Produces: `GraphMutations.addNodeDependingOn(DesiredNode, NodeId) → List<GraphMutation>`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.desiredstate.api;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class GraphMutationsTest {

    private static final NodeType TARGET = NodeType.of("target");
    private static final NodeType REVIEW = NodeType.of("review");

    record TestSpec(String detail) implements NodeSpec {}

    @Test
    void addNodeDependingOn_returnsAddNodeAndAddDependency() {
        DesiredNode node = new DesiredNode(NodeId.of("review-n1"), REVIEW,
                new TestSpec("test"), HumanGating.NONE);
        NodeId dependsOn = NodeId.of("n1");

        List<GraphMutation> mutations = GraphMutations.addNodeDependingOn(node, dependsOn);

        assertThat(mutations).hasSize(2);
        assertThat(mutations.get(0)).isInstanceOf(GraphMutation.AddNode.class);
        GraphMutation.AddNode addNode = (GraphMutation.AddNode) mutations.get(0);
        assertThat(addNode.node()).isEqualTo(node);

        assertThat(mutations.get(1)).isInstanceOf(GraphMutation.AddDependency.class);
        GraphMutation.AddDependency addDep = (GraphMutation.AddDependency) mutations.get(1);
        assertThat(addDep.dependency().from()).isEqualTo(NodeId.of("review-n1"));
        assertThat(addDep.dependency().to()).isEqualTo(NodeId.of("n1"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=GraphMutationsTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `GraphMutations` class does not exist

- [ ] **Step 3: Write minimal implementation**

```java
package io.casehub.desiredstate.api;

import java.util.List;

public final class GraphMutations {
    private GraphMutations() {}

    public static List<GraphMutation> addNodeDependingOn(DesiredNode node, NodeId dependsOn) {
        return List.of(
            new GraphMutation.AddNode(node),
            new GraphMutation.AddDependency(new Dependency(node.id(), dependsOn))
        );
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=GraphMutationsTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/GraphMutations.java \
        runtime/src/test/java/io/casehub/desiredstate/api/GraphMutationsTest.java
git commit -m "feat(#87): add GraphMutations.addNodeDependingOn utility"
```

---

### Task 2: withDependency CAS Safety

**Files:**
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraph.java:185-195`
- Test: `runtime/src/test/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraphTest.java`

**Interfaces:**
- Consumes: `DesiredStateGraph.withDependency(Dependency)`
- Produces: No new interfaces — changes existing behavior from throwing to no-op on missing nodes

- [ ] **Step 1: Write the failing tests**

Add two new tests and modify three existing "throws" tests to assert no-op instead.

New tests:
```java
@Test void withDependency_missingFromNode_returnsUnchanged() {
    DesiredStateGraph g = factory.of(List.of(node("A")), List.of());
    DesiredStateGraph result = g.withDependency(dep("phantom", "A"));
    assertThat(result).isSameAs(g);
}

@Test void withDependency_missingToNode_returnsUnchanged() {
    DesiredStateGraph g = factory.of(List.of(node("A")), List.of());
    DesiredStateGraph result = g.withDependency(dep("A", "phantom"));
    assertThat(result).isSameAs(g);
}
```

Change existing tests:
- `withDependency_throws_when_from_node_missing` → rename to `withDependency_missingFromNode_returnsUnchanged` (covered by new test above — delete the old one)
- `withDependency_throws_when_to_node_missing` → same
- `withDependency_throws_when_both_nodes_missing` → change to assert `isSameAs(g)` instead of `assertThatThrownBy`

- [ ] **Step 2: Run tests to verify failures**

Run: `mvn --batch-mode test -pl runtime -Dtest=ImmutableDesiredStateGraphTest`
Expected: New no-op tests FAIL (still throws DanglingDependencyException)

- [ ] **Step 3: Modify withDependency implementation**

In `ImmutableDesiredStateGraph.withDependency()`, replace the two missing-node checks
(lines 190-195) with:

```java
if (!nodes.containsKey(from) || !nodes.containsKey(to)) {
    return this;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl runtime -Dtest=ImmutableDesiredStateGraphTest`
Expected: ALL PASS (including cycle detection, self-loop — unchanged)

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraph.java \
        runtime/src/test/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraphTest.java
git commit -m "feat(#87): withDependency tolerates missing nodes for CAS safety"
```

---

### Task 3: Updated addReviewNode

**Files:**
- Modify: `api/src/main/java/io/casehub/desiredstate/api/FaultPolicy.java:8-17`
- Test: `runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java`

**Interfaces:**
- Consumes: `GraphMutations.addNodeDependingOn()` (from Task 1)
- Produces: `FaultPolicy.addReviewNode(NodeType, ReviewSpecFactory)` — now returns
  `[AddNode, AddDependency]` instead of `[AddNode]`; ID prefix from `reviewType.value()`

- [ ] **Step 1: Write failing tests for new behavior**

Add a new test to ThresholdFaultPolicyTest:

```java
@Test
void addReviewNode_returnsDependencyEdge() {
    var policy = policyWithThreshold(1);
    var graph = graphWith("n1", TARGET);
    var event = new FaultEvent(NodeId.of("n1"), FaultType.PROVISION_FAILED, "fail");

    var mutations = policy.onFault("t1", event, graph, EMPTY_ACTUAL);

    assertThat(mutations).hasSize(2);
    assertThat(mutations.get(0)).isInstanceOf(GraphMutation.AddNode.class);
    assertThat(mutations.get(1)).isInstanceOf(GraphMutation.AddDependency.class);
    var addDep = (GraphMutation.AddDependency) mutations.get(1);
    assertThat(addDep.dependency().from()).isEqualTo(NodeId.of("review-n1"));
    assertThat(addDep.dependency().to()).isEqualTo(NodeId.of("n1"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=ThresholdFaultPolicyTest#addReviewNode_returnsDependencyEdge`
Expected: FAIL — mutations has size 1, not 2

- [ ] **Step 3: Update addReviewNode implementation**

Replace `FaultPolicy.addReviewNode()` body in `FaultPolicy.java`:

```java
static FaultPolicy addReviewNode(NodeType reviewType, ReviewSpecFactory specFactory) {
    return (tenancyId, event, current, actual) -> {
        NodeId reviewId = NodeId.of(reviewType.value() + "-" + event.node().value());
        if (current.nodes().containsKey(reviewId)) {
            return List.of();
        }
        DesiredNode node = new DesiredNode(reviewId, reviewType,
                specFactory.create(event, current), HumanGating.ALL);
        return GraphMutations.addNodeDependingOn(node, event.node());
    };
}
```

- [ ] **Step 4: Fix existing test assertions**

In `ThresholdFaultPolicyTest`, update mutation count assertions:

- `atThreshold_delegatesToAction`: change `assertThat(mutations).hasSize(1)` → `hasSize(2)`
- `aboveThreshold_delegatesAgain`: change both `hasSize(1)` → `hasSize(2)`
- `emptyNodeTypes_matchesAll`: change `hasSize(1)` → `hasSize(2)`
- `multipleNodesTrackedIndependently`: change `hasSize(1)` → `hasSize(2)`
- `faultCountPersistsAcrossRecovery`: change `hasSize(1)` → `hasSize(2)`
- `resetCount_resetsAndNextFaultsStartFromOne`: change `hasSize(1)` → `hasSize(2)`

- [ ] **Step 5: Run all ThresholdFaultPolicyTest tests**

Run: `mvn --batch-mode test -pl runtime -Dtest=ThresholdFaultPolicyTest`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/FaultPolicy.java \
        runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java
git commit -m "feat(#87): addReviewNode includes dependency edge and NodeType-based ID"
```

---

### Task 4: Multi-Tier ThresholdFaultPolicy

**Files:**
- Modify: `api/src/main/java/io/casehub/desiredstate/api/ThresholdFaultPolicy.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java`

**Interfaces:**
- Consumes: `DesiredStateGraph.dependentsOf(NodeId)`, `FaultPolicy`, `FaultCountStore`
- Produces: `ThresholdFaultPolicy.Tier(int, FaultPolicy, NodeType)` record,
  `Builder.tier(int, FaultPolicy, NodeType)` method

- [ ] **Step 1: Write failing multi-tier tests**

Add new tests to ThresholdFaultPolicyTest:

```java
private static final NodeType AI_REVIEW = NodeType.of("ai-review");
private static final NodeType HUMAN_REVIEW = NodeType.of("human-review");

record AiReviewSpec(NodeId faultedNode) implements NodeSpec {}
record HumanReviewSpec(NodeId faultedNode) implements NodeSpec {}

private ThresholdFaultPolicy twoTierPolicy() {
    return ThresholdFaultPolicy.builder()
            .faultTypes(Set.of(FaultType.PROVISION_FAILED))
            .nodeTypes(Set.of(TARGET))
            .tier(3, FaultPolicy.addReviewNode(AI_REVIEW,
                    (event, current) -> new AiReviewSpec(event.node())), AI_REVIEW)
            .tier(6, FaultPolicy.addReviewNode(HUMAN_REVIEW,
                    (event, current) -> new HumanReviewSpec(event.node())), HUMAN_REVIEW)
            .build();
}

@Test
void multiTier_belowAllThresholds_returnsEmpty() {
    var policy = twoTierPolicy();
    var graph = graphWith("n1", TARGET);
    var event = new FaultEvent(NodeId.of("n1"), FaultType.PROVISION_FAILED, "fail");

    assertThat(policy.onFault("t1", event, graph, EMPTY_ACTUAL)).isEmpty();
    assertThat(policy.onFault("t1", event, graph, EMPTY_ACTUAL)).isEmpty();
}

@Test
void multiTier_atTier1Threshold_firesTier1() {
    var policy = twoTierPolicy();
    var graph = graphWith("n1", TARGET);
    var event = new FaultEvent(NodeId.of("n1"), FaultType.PROVISION_FAILED, "fail");

    policy.onFault("t1", event, graph, EMPTY_ACTUAL);
    policy.onFault("t1", event, graph, EMPTY_ACTUAL);
    var mutations = policy.onFault("t1", event, graph, EMPTY_ACTUAL);

    assertThat(mutations).hasSize(2);
    var addNode = (GraphMutation.AddNode) mutations.get(0);
    assertThat(addNode.node().type()).isEqualTo(AI_REVIEW);
}

@Test
void multiTier_atTier2Threshold_tier1Present_firesTier2() {
    var policy = twoTierPolicy();
    var graph = graphWith("n1", TARGET);
    var event = new FaultEvent(NodeId.of("n1"), FaultType.PROVISION_FAILED, "fail");

    // Reach tier 2 threshold
    for (int i = 0; i < 5; i++) {
        policy.onFault("t1", event, graph, EMPTY_ACTUAL);
    }

    // Add AI review node with dependency edge so dependentsOf finds it
    var aiNode = new DesiredNode(NodeId.of("ai-review-n1"), AI_REVIEW,
            new AiReviewSpec(NodeId.of("n1")), HumanGating.ALL);
    var graphWithAi = graph.withNode(aiNode)
            .withDependency(new Dependency(NodeId.of("ai-review-n1"), NodeId.of("n1")));

    var mutations = policy.onFault("t1", event, graphWithAi, EMPTY_ACTUAL);

    assertThat(mutations).hasSize(2);
    var addNode = (GraphMutation.AddNode) mutations.get(0);
    assertThat(addNode.node().type()).isEqualTo(HUMAN_REVIEW);
}

@Test
void multiTier_atTier2Threshold_tier1Absent_firesTier1() {
    var policy = twoTierPolicy();
    var graph = graphWith("n1", TARGET);
    var event = new FaultEvent(NodeId.of("n1"), FaultType.PROVISION_FAILED, "fail");

    // Reach tier 2 threshold without adding AI review node
    for (int i = 0; i < 5; i++) {
        policy.onFault("t1", event, graph, EMPTY_ACTUAL);
    }
    var mutations = policy.onFault("t1", event, graph, EMPTY_ACTUAL);

    assertThat(mutations).hasSize(2);
    var addNode = (GraphMutation.AddNode) mutations.get(0);
    assertThat(addNode.node().type()).isEqualTo(AI_REVIEW);
}

@Test
void multiTier_firstMatchWins_emptyResultNotFallenThrough() {
    var policy = twoTierPolicy();
    var graph = graphWith("n1", TARGET);
    var event = new FaultEvent(NodeId.of("n1"), FaultType.PROVISION_FAILED, "fail");

    // Build up to tier 1 and let it fire (count=3)
    for (int i = 0; i < 3; i++) {
        policy.onFault("t1", event, graph, EMPTY_ACTUAL);
    }

    // Now the AI review node exists — tier 1 duplicate guard → empty
    var aiNode = new DesiredNode(NodeId.of("ai-review-n1"), AI_REVIEW,
            new AiReviewSpec(NodeId.of("n1")), HumanGating.ALL);
    var graphWithAi = graph.withNode(aiNode)
            .withDependency(new Dependency(NodeId.of("ai-review-n1"), NodeId.of("n1")));

    // Count is 4 (above tier 1), but below tier 2 — tier 1 fires, duplicate guard → empty
    var mutations = policy.onFault("t1", event, graphWithAi, EMPTY_ACTUAL);
    assertThat(mutations).isEmpty();
}

@Test
void multiTier_autoIgnore_faultOnTierNodeType_returnsEmpty() {
    var policy = twoTierPolicy();
    var aiNode = new DesiredNode(NodeId.of("ai-review-n1"), AI_REVIEW,
            new AiReviewSpec(NodeId.of("n1")), HumanGating.NONE);
    var graph = graphFactory.of(List.of(aiNode), List.of());
    var event = new FaultEvent(NodeId.of("ai-review-n1"), FaultType.PROVISION_FAILED, "fail");

    assertThat(policy.onFault("t1", event, graph, EMPTY_ACTUAL)).isEmpty();
}

@Test
void multiTier_builderRejectsNoTiers() {
    assertThatThrownBy(() -> ThresholdFaultPolicy.builder()
            .faultTypes(Set.of(FaultType.PROVISION_FAILED))
            .build())
            .isInstanceOf(IllegalArgumentException.class);
}

@Test
void multiTier_builderRejectsNonAscendingThresholds() {
    assertThatThrownBy(() -> ThresholdFaultPolicy.builder()
            .faultTypes(Set.of(FaultType.PROVISION_FAILED))
            .tier(5, (t, e, g, a) -> List.of(), AI_REVIEW)
            .tier(3, (t, e, g, a) -> List.of(), HUMAN_REVIEW)
            .build())
            .isInstanceOf(IllegalArgumentException.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=ThresholdFaultPolicyTest#multiTier_belowAllThresholds_returnsEmpty`
Expected: FAIL — `.tier()` method does not exist on Builder

- [ ] **Step 3: Implement Tier record, Builder changes, and evaluation logic**

Replace the full `ThresholdFaultPolicy` class body. Key changes:

1. Add nested `Tier` record
2. Replace `threshold`/`action` fields with `List<Tier> tiers`
3. Builder: drop `.threshold()`, `.action()`, add `.tier()`
4. Auto-merge tier nodeTypes into ignoreTypes at build time
5. New evaluation loop: iterate tiers highest-first, first-match-wins
6. Graph-presence guard: `dependentsOf` + type check

```java
package io.casehub.desiredstate.api;

import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Objects;
import java.util.Set;
import java.util.stream.Collectors;

public class ThresholdFaultPolicy implements FaultPolicy {

    public record Tier(int threshold, FaultPolicy action, NodeType nodeType) {
        public Tier {
            if (threshold < 1) throw new IllegalArgumentException("threshold must be >= 1, got " + threshold);
            Objects.requireNonNull(action, "action is required");
            Objects.requireNonNull(nodeType, "nodeType is required");
        }
    }

    private final Set<FaultType>  faultTypes;
    private final Set<NodeType>   nodeTypes;
    private final Set<NodeType>   ignoreTypes;
    private final List<Tier>      tiers;
    private final FaultCountStore store;
    private final String          namespace;

    private ThresholdFaultPolicy(Builder builder) {
        this.faultTypes  = Set.copyOf(builder.faultTypes);
        this.nodeTypes   = builder.nodeTypes == null ? Set.of() : Set.copyOf(builder.nodeTypes);

        Set<NodeType> merged = new HashSet<>();
        if (builder.ignoreTypes != null) merged.addAll(builder.ignoreTypes);
        for (Tier tier : builder.tiers) {
            merged.add(tier.nodeType());
        }
        this.ignoreTypes = Set.copyOf(merged);

        this.tiers       = List.copyOf(builder.tiers);
        this.store       = builder.store != null ? builder.store : new InMemoryFaultCountStore();
        this.namespace   = builder.namespace != null ? builder.namespace : deriveNamespace(this.faultTypes);
    }

    private static String deriveNamespace(Set<FaultType> faultTypes) {
        return faultTypes.stream()
                         .map(FaultType::name)
                         .sorted()
                         .collect(Collectors.joining(","));
    }

    public static Builder builder() {
        return new Builder();
    }

    @Override
    public List<GraphMutation> onFault(String tenancyId, FaultEvent event,
                                       DesiredStateGraph current, ActualState actual) {
        DesiredNode node = current.nodes().get(event.node());

        if (node != null && ignoreTypes.contains(node.type())) {
            return List.of();
        }

        if (node == null) {
            store.remove(namespace, tenancyId, event.node());
            return List.of();
        }

        if (!faultTypes.contains(event.type())) {
            return List.of();
        }

        if (!nodeTypes.isEmpty() && !nodeTypes.contains(node.type())) {
            return List.of();
        }

        int count = store.incrementAndGet(namespace, tenancyId, event.node());

        for (int i = tiers.size() - 1; i >= 0; i--) {
            Tier tier = tiers.get(i);
            if (count < tier.threshold()) {
                continue;
            }

            if (i > 0) {
                NodeType previousNodeType = tiers.get(i - 1).nodeType();
                boolean previousTierPresent = current.dependentsOf(event.node()).stream()
                        .map(depId -> current.nodes().get(depId))
                        .filter(Objects::nonNull)
                        .anyMatch(n -> n.type().equals(previousNodeType));
                if (!previousTierPresent) {
                    continue;
                }
            }

            return tier.action().onFault(tenancyId, event, current, actual);
        }

        return List.of();
    }

    public void resetCount(String tenancyId, NodeId nodeId) {
        store.reset(namespace, tenancyId, nodeId);
    }

    public static class Builder {
        private Set<FaultType>  faultTypes;
        private Set<NodeType>   nodeTypes;
        private Set<NodeType>   ignoreTypes;
        private final List<Tier> tiers = new ArrayList<>();
        private FaultCountStore store;
        private String          namespace;

        public Builder faultTypes(Set<FaultType> faultTypes) { this.faultTypes = faultTypes; return this; }
        public Builder nodeTypes(Set<NodeType> nodeTypes) { this.nodeTypes = nodeTypes; return this; }
        public Builder ignoreTypes(Set<NodeType> ignoreTypes) { this.ignoreTypes = ignoreTypes; return this; }
        public Builder tier(int threshold, FaultPolicy action, NodeType nodeType) {
            this.tiers.add(new Tier(threshold, action, nodeType));
            return this;
        }
        public Builder faultCountStore(FaultCountStore store) { this.store = store; return this; }
        public Builder namespace(String namespace) { this.namespace = namespace; return this; }

        public ThresholdFaultPolicy build() {
            Objects.requireNonNull(faultTypes, "faultTypes is required");
            if (tiers.isEmpty()) {
                throw new IllegalArgumentException("at least one tier is required");
            }
            for (int i = 1; i < tiers.size(); i++) {
                if (tiers.get(i).threshold() <= tiers.get(i - 1).threshold()) {
                    throw new IllegalArgumentException(
                        "tier thresholds must be strictly ascending: " +
                        tiers.get(i - 1).threshold() + " >= " + tiers.get(i).threshold());
                }
            }
            if (store != null && namespace == null) {
                throw new IllegalArgumentException("namespace is required when a custom faultCountStore is provided");
            }
            return new ThresholdFaultPolicy(this);
        }
    }
}
```

- [ ] **Step 4: Migrate existing ThresholdFaultPolicyTest builder calls**

Replace `policyWithThreshold` helper:
```java
private ThresholdFaultPolicy policyWithThreshold(int threshold) {
    return ThresholdFaultPolicy.builder()
            .faultTypes(Set.of(FaultType.PROVISION_FAILED))
            .nodeTypes(Set.of(TARGET))
            .ignoreTypes(Set.of(REVIEW))
            .tier(threshold, FaultPolicy.addReviewNode(REVIEW,
                    (event, current) -> new TestReviewSpec(event.node(), event.detail())), REVIEW)
            .build();
}
```

Update `emptyNodeTypes_matchesAll`:
```java
var policy = ThresholdFaultPolicy.builder()
        .faultTypes(Set.of(FaultType.PROVISION_FAILED))
        .tier(1, FaultPolicy.addReviewNode(REVIEW,
                (event, current) -> new TestReviewSpec(event.node(), event.detail())), REVIEW)
        .build();
```

Update `customStore_receivesIncrementCalls`:
```java
var policy = ThresholdFaultPolicy.builder()
        .faultTypes(Set.of(FaultType.PROVISION_FAILED))
        .tier(2, FaultPolicy.addReviewNode(REVIEW,
                (event, current) -> new TestReviewSpec(event.node(), event.detail())), REVIEW)
        .faultCountStore(store)
        .namespace("test-policy")
        .build();
```

Update `lazyEviction_matchingFaultType_removesCount`:
```java
var policy = ThresholdFaultPolicy.builder()
        .faultTypes(Set.of(FaultType.PROVISION_FAILED))
        .tier(3, (t, e, g, a) -> List.of(), NodeType.of("escalation"))
        .faultCountStore(store)
        .namespace("test")
        .build();
```

Update `lazyEviction_nonMatchingFaultType_stillRemovesCount` — same pattern as above.

Update `builderRequiresFaultTypes`:
```java
assertThatThrownBy(() -> ThresholdFaultPolicy.builder()
        .tier(1, (t, e, g, a) -> List.of(), NodeType.of("x"))
        .build())
        .isInstanceOf(NullPointerException.class);
```

Update `builderRequiresAction` → rename to `builderRejectsNoTiers`:
```java
assertThatThrownBy(() -> ThresholdFaultPolicy.builder()
        .faultTypes(Set.of(FaultType.PROVISION_FAILED))
        .build())
        .isInstanceOf(IllegalArgumentException.class);
```

Update `builderRejectsZeroThreshold`:
```java
assertThatThrownBy(() -> ThresholdFaultPolicy.builder()
        .faultTypes(Set.of(FaultType.PROVISION_FAILED))
        .tier(0, (t, e, g, a) -> List.of(), NodeType.of("x"))
        .build())
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("threshold");
```

Update `builder_requiresNamespaceForCustomStore`:
```java
assertThatThrownBy(() -> ThresholdFaultPolicy.builder()
        .faultTypes(Set.of(FaultType.PROVISION_FAILED))
        .tier(1, (t, e, g, a) -> List.of(), NodeType.of("x"))
        .faultCountStore(new InMemoryFaultCountStore())
        .build())
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("namespace");
```

- [ ] **Step 5: Run all tests**

Run: `mvn --batch-mode test -pl runtime -Dtest=ThresholdFaultPolicyTest`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/ThresholdFaultPolicy.java \
        runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java
git commit -m "feat(#86): multi-tier escalation in ThresholdFaultPolicy with graph-presence guards"
```

---

### Task 5: Pipeline Example Migration

**Files:**
- Delete: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/ProvisionEscalationFaultPolicy.java` (use `ide_refactor_safe_delete`)
- Modify: `examples/pipeline/src/test/java/io/casehub/desiredstate/example/pipeline/ProvisionEscalationFaultPolicyTest.java`
- Modify: `examples/pipeline/src/test/java/io/casehub/desiredstate/example/pipeline/PipelineTest.java`

**Interfaces:**
- Consumes: `ThresholdFaultPolicy.builder().tier()`, `FaultPolicy.addReviewNode()`
- Produces: No new interfaces — migration only

- [ ] **Step 1: Rewrite ProvisionEscalationFaultPolicyTest**

Replace the test class to use `ThresholdFaultPolicy` configuration:

```java
package io.casehub.desiredstate.example.pipeline;

import io.casehub.desiredstate.api.ActualState;
import io.casehub.desiredstate.api.Dependency;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.FaultCountStore;
import io.casehub.desiredstate.api.FaultEvent;
import io.casehub.desiredstate.api.FaultPolicy;
import io.casehub.desiredstate.api.FaultType;
import io.casehub.desiredstate.api.GraphMutation;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.InMemoryFaultCountStore;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.ThresholdFaultPolicy;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class PipelineEscalationPolicyTest {

    private FaultCountStore store;
    private ThresholdFaultPolicy policy;
    private DefaultDesiredStateGraphFactory graphFactory;

    @BeforeEach
    void setUp() {
        store = new InMemoryFaultCountStore();
        graphFactory = new DefaultDesiredStateGraphFactory();
        policy = ThresholdFaultPolicy.builder()
                .faultTypes(Set.of(FaultType.PROVISION_FAILED))
                .tier(4, FaultPolicy.addReviewNode(PipelineNodeTypes.AI_REVIEW,
                        (event, current) -> new AiReviewSpec(event.node(), event.detail())), PipelineNodeTypes.AI_REVIEW)
                .tier(7, FaultPolicy.addReviewNode(PipelineNodeTypes.HUMAN_REVIEW,
                        (event, current) -> new HumanReviewSpec(event.node(), event.detail(), "Escalated")), PipelineNodeTypes.HUMAN_REVIEW)
                .faultCountStore(store)
                .namespace("pipeline-escalation")
                .build();
    }

    @Test
    void belowThreshold_returnsEmpty() {
        DesiredNode node = new DesiredNode(NodeId.of("ingest"), PipelineNodeTypes.INGESTION,
                new IngestionSpec("data", 100, "json"), HumanGating.NONE);
        DesiredStateGraph graph = graphFactory.of(List.of(node), List.of());
        FaultEvent fault = new FaultEvent(NodeId.of("ingest"), FaultType.PROVISION_FAILED, "down");

        for (int i = 0; i < 3; i++) {
            assertThat(policy.onFault("t1", fault, graph, new ActualState(Map.of()))).isEmpty();
        }
    }

    @Test
    void atThreshold_firesAiReview() {
        DesiredNode node = new DesiredNode(NodeId.of("ingest"), PipelineNodeTypes.INGESTION,
                new IngestionSpec("data", 100, "json"), HumanGating.NONE);
        DesiredStateGraph graph = graphFactory.of(List.of(node), List.of());
        FaultEvent fault = new FaultEvent(NodeId.of("ingest"), FaultType.PROVISION_FAILED, "down");

        for (int i = 0; i < 3; i++) {
            policy.onFault("t1", fault, graph, new ActualState(Map.of()));
        }
        var mutations = policy.onFault("t1", fault, graph, new ActualState(Map.of()));

        assertThat(mutations).hasSize(2);
        var addNode = (GraphMutation.AddNode) mutations.get(0);
        assertThat(addNode.node().type()).isEqualTo(PipelineNodeTypes.AI_REVIEW);
        assertThat(addNode.node().id()).isEqualTo(NodeId.of("ai-review-ingest"));
    }

    @Test
    void escalation_aiPresentAndThresholdReached_firesHumanReview() {
        DesiredNode node = new DesiredNode(NodeId.of("ingest"), PipelineNodeTypes.INGESTION,
                new IngestionSpec("data", 100, "json"), HumanGating.NONE);
        DesiredNode aiNode = new DesiredNode(NodeId.of("ai-review-ingest"), PipelineNodeTypes.AI_REVIEW,
                new AiReviewSpec(NodeId.of("ingest"), "fail"), HumanGating.NONE);
        DesiredStateGraph graph = graphFactory.of(List.of(node, aiNode),
                List.of(new Dependency(NodeId.of("ai-review-ingest"), NodeId.of("ingest"))));
        FaultEvent fault = new FaultEvent(NodeId.of("ingest"), FaultType.PROVISION_FAILED, "down");

        for (int i = 0; i < 6; i++) {
            policy.onFault("t1", fault, graph, new ActualState(Map.of()));
        }
        var mutations = policy.onFault("t1", fault, graph, new ActualState(Map.of()));

        assertThat(mutations).hasSize(2);
        var addNode = (GraphMutation.AddNode) mutations.get(0);
        assertThat(addNode.node().type()).isEqualTo(PipelineNodeTypes.HUMAN_REVIEW);
    }

    @Test
    void tenantIsolation() {
        DesiredNode node = new DesiredNode(NodeId.of("ingest"), PipelineNodeTypes.INGESTION,
                new IngestionSpec("data", 100, "json"), HumanGating.NONE);
        DesiredStateGraph graph = graphFactory.of(List.of(node), List.of());
        FaultEvent fault = new FaultEvent(NodeId.of("ingest"), FaultType.PROVISION_FAILED, "down");

        for (int i = 0; i < 3; i++) {
            policy.onFault("tenant-a", fault, graph, new ActualState(Map.of()));
        }
        policy.onFault("tenant-b", fault, graph, new ActualState(Map.of()));

        assertThat(store.getCount("pipeline-escalation", "tenant-a", NodeId.of("ingest"))).isEqualTo(3);
        assertThat(store.getCount("pipeline-escalation", "tenant-b", NodeId.of("ingest"))).isEqualTo(1);
    }
}
```

- [ ] **Step 2: Update PipelineTest references**

In `PipelineTest.java`, replace `ProvisionEscalationFaultPolicy` usages at lines 318
and 369 with `ThresholdFaultPolicy` configurations using the same builder pattern as
the test above. Import `ThresholdFaultPolicy`, `FaultPolicy`, `Set`, `FaultType`.

- [ ] **Step 3: Delete ProvisionEscalationFaultPolicy**

Use `ide_refactor_safe_delete` on `ProvisionEscalationFaultPolicy.java`.
Delete the old `ProvisionEscalationFaultPolicyTest.java`.

- [ ] **Step 4: Run all pipeline tests**

Run: `mvn --batch-mode test -pl examples/pipeline`
Expected: ALL PASS

- [ ] **Step 5: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add -A examples/pipeline/
git commit -m "refactor(#86): replace ProvisionEscalationFaultPolicy with ThresholdFaultPolicy config"
```

---

### Task 6: Follow-Up Issue and Documentation

**Files:**
- None (GitHub issue creation + CLAUDE.md update)

**Interfaces:**
- None

- [ ] **Step 1: File follow-up issue**

```bash
gh issue create --repo casehubio/casehub-desiredstate \
  --title "refactor: migrate QuarantineFaultPolicy and SchemaDriftFaultPolicy to addReviewNode" \
  --body "Both policies hand-roll AddNode without addReviewNode — they create review nodes
without dependency edges, which is now structurally inconsistent with the platform
convention established in #87.

Migrate both to use FaultPolicy.addReviewNode() so review nodes get dependency edges
and consistent ID derivation.

Surfaced during design review of #87/#86."
```

- [ ] **Step 2: Update CLAUDE.md ThresholdFaultPolicy description**

Update the Core Runtime Types table entry for `ThresholdFaultPolicy` to reflect
the new multi-tier builder API: replace "Builder: faultTypes, nodeTypes, ignoreTypes,
threshold, action, faultCountStore, namespace" with "Builder: faultTypes, nodeTypes,
ignoreTypes, tier(threshold, action, nodeType), faultCountStore, namespace. Multi-tier
escalation with graph-presence guards. Auto-ignore tier nodeTypes."

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#86): update CLAUDE.md for multi-tier ThresholdFaultPolicy"
```
