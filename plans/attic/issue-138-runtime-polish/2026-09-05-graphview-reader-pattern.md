# GraphView/Reader Pattern Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #129 — GraphView reader pattern in annotations/runtime
**Issue group:** #138, #129, #132

**Goal:** Decouple graph engines from desiredstate API types so they are mechanically extractable to graph-core (platform#267).

**Architecture:** Introduce GraphView<N>/GraphReader<G,N>/GraphWriter<G,N> abstraction layer.
Parameterize GraphMutation<N> in place. Unify imperative/generic engine paths via function
closures in recorders (D1). All algorithms unchanged — structural type generification only.

**Tech Stack:** Java 21, Quarkus (build extensions + recorders), sealed interfaces, generics

## Global Constraints

- Zero imports of `DesiredNode`, `DesiredStateGraph`, `NodeId`, `NodeType`, `Dependency`
  from engine classes after refactor
- `GraphMutation`, `ConflictingMutationException` remain in `api/` during step 1 (move at
  platform#267)
- All existing tests pass — same logic, updated types
- New interfaces in `io.casehub.desiredstate.annotations.runtime.graph` subpackage
- Adapter + view in `io.casehub.desiredstate.annotations.runtime` (alongside engines)

---

## Batch 1: View stack foundation

### Task 1: Create graph view interfaces, adapter, and view implementation

**Files:**
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/graph/GraphView.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/graph/MutableGraphView.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/graph/GraphReader.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/graph/GraphWriter.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/graph/GraphCycleException.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphAdapter.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphView.java`
- Test: `annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphViewTest.java`

**Interfaces:**
- Produces: `GraphView<N>` (6 methods: `nodes()`, `node(id)`, `nodeId(n)`, `nodeType(n)`, `dependenciesOf(id)`, `dependentsOf(id)`), `MutableGraphView<N>` (adds `withMutation()`), `GraphReader<G,N>` (6 methods mirroring view but with graph parameter), `GraphWriter<G,N>` (1 method: `applyMutation()`), `GraphCycleException`, `DesiredStateGraphAdapter` (implements reader+writer), `DesiredStateGraphView` (implements MutableGraphView, exposes `graph()` accessor)

- [ ] **Step 1: Write the failing test — view delegates correctly**

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.annotations.runtime.graph.GraphView;
import io.casehub.desiredstate.annotations.runtime.graph.MutableGraphView;
import io.casehub.desiredstate.runtime.ImmutableDesiredStateGraph;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Set;
import static org.junit.jupiter.api.Assertions.*;

class DesiredStateGraphViewTest {

    private static final NodeSpec SPEC_A = () -> NodeType.of("type-a");
    private static final NodeSpec SPEC_B = () -> NodeType.of("type-b");

    private DesiredStateGraphView viewOf(DesiredStateGraph graph) {
        return new DesiredStateGraphView(graph, new DesiredStateGraphAdapter());
    }

    @Test
    void nodesReturnsStringKeyedMap() {
        DesiredNode a = new DesiredNode(NodeId.of("a"), SPEC_A, HumanGating.NONE);
        DesiredNode b = new DesiredNode(NodeId.of("b"), SPEC_B, HumanGating.NONE);
        var graph = new ImmutableDesiredStateGraph(List.of(a, b), List.of());
        var view = viewOf(graph);

        assertEquals(2, view.nodes().size());
        assertSame(a, view.nodes().get("a"));
        assertSame(b, view.nodes().get("b"));
    }

    @Test
    void nodeByIdReturnsCorrectNode() {
        DesiredNode a = new DesiredNode(NodeId.of("a"), SPEC_A, HumanGating.NONE);
        var graph = new ImmutableDesiredStateGraph(List.of(a), List.of());
        var view = viewOf(graph);

        assertSame(a, view.node("a"));
        assertNull(view.node("nonexistent"));
    }

    @Test
    void nodeIdExtractsStringId() {
        DesiredNode a = new DesiredNode(NodeId.of("my-node"), SPEC_A, HumanGating.NONE);
        var graph = new ImmutableDesiredStateGraph(List.of(a), List.of());
        var view = viewOf(graph);

        assertEquals("my-node", view.nodeId(a));
    }

    @Test
    void nodeTypeExtractsTypeString() {
        DesiredNode a = new DesiredNode(NodeId.of("a"), SPEC_A, HumanGating.NONE);
        var graph = new ImmutableDesiredStateGraph(List.of(a), List.of());
        var view = viewOf(graph);

        assertEquals("type-a", view.nodeType(a));
    }

    @Test
    void dependenciesOfReturnsStringIds() {
        DesiredNode a = new DesiredNode(NodeId.of("a"), SPEC_A, HumanGating.NONE);
        DesiredNode b = new DesiredNode(NodeId.of("b"), SPEC_B, HumanGating.NONE);
        var dep = new Dependency(NodeId.of("a"), NodeId.of("b"));
        var graph = new ImmutableDesiredStateGraph(List.of(a, b), List.of(dep));
        var view = viewOf(graph);

        assertEquals(Set.of("b"), view.dependenciesOf("a"));
        assertEquals(Set.of(), view.dependenciesOf("b"));
    }

    @Test
    void dependentsOfReturnsStringIds() {
        DesiredNode a = new DesiredNode(NodeId.of("a"), SPEC_A, HumanGating.NONE);
        DesiredNode b = new DesiredNode(NodeId.of("b"), SPEC_B, HumanGating.NONE);
        var dep = new Dependency(NodeId.of("a"), NodeId.of("b"));
        var graph = new ImmutableDesiredStateGraph(List.of(a, b), List.of(dep));
        var view = viewOf(graph);

        assertEquals(Set.of("a"), view.dependentsOf("b"));
        assertEquals(Set.of(), view.dependentsOf("a"));
    }

    @Test
    void withMutationReturnsNewViewWithUpdatedGraph() {
        DesiredNode a = new DesiredNode(NodeId.of("a"), SPEC_A, HumanGating.NONE);
        var graph = new ImmutableDesiredStateGraph(List.of(a), List.of());
        var view = viewOf(graph);

        DesiredNode b = new DesiredNode(NodeId.of("b"), SPEC_B, HumanGating.NONE);
        MutableGraphView<DesiredNode> updated = view.withMutation(new GraphMutation.AddNode<>(b));

        assertEquals(1, view.nodes().size());
        assertEquals(2, updated.nodes().size());
        assertNotNull(updated.node("b"));
    }

    @Test
    void withMutationPreservesGraphAccessor() {
        DesiredNode a = new DesiredNode(NodeId.of("a"), SPEC_A, HumanGating.NONE);
        var graph = new ImmutableDesiredStateGraph(List.of(a), List.of());
        var view = viewOf(graph);

        DesiredNode b = new DesiredNode(NodeId.of("b"), SPEC_B, HumanGating.NONE);
        var updated = (DesiredStateGraphView) view.withMutation(new GraphMutation.AddNode<>(b));

        assertNotSame(graph, updated.graph());
        assertTrue(updated.graph().nodes().containsKey(NodeId.of("b")));
    }

    @Test
    void writerCatchesCyclicDependencyAndWraps() {
        DesiredNode a = new DesiredNode(NodeId.of("a"), SPEC_A, HumanGating.NONE);
        DesiredNode b = new DesiredNode(NodeId.of("b"), SPEC_B, HumanGating.NONE);
        var dep = new Dependency(NodeId.of("a"), NodeId.of("b"));
        var graph = new ImmutableDesiredStateGraph(List.of(a, b), List.of(dep));
        var view = viewOf(graph);

        var ex = assertThrows(
            io.casehub.desiredstate.annotations.runtime.graph.GraphCycleException.class,
            () -> view.withMutation(new GraphMutation.AddEdge<>("b", "a"))
        );
        assertEquals(List.of("a", "b"), ex.getCycle());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=DesiredStateGraphViewTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — classes not found

- [ ] **Step 3: Create the graph subpackage interfaces**

Create `GraphReader.java`:
```java
package io.casehub.desiredstate.annotations.runtime.graph;

import java.util.Map;
import java.util.Set;

public interface GraphReader<G, N> {
    Map<String, N> nodes(G graph);
    N node(G graph, String id);
    String nodeId(N node);
    String nodeType(N node);
    Set<String> dependenciesOf(G graph, String nodeId);
    Set<String> dependentsOf(G graph, String nodeId);
}
```

Create `GraphWriter.java`:
```java
package io.casehub.desiredstate.annotations.runtime.graph;

public interface GraphWriter<G, N> {
    G applyMutation(G graph, GraphMutation<N> mutation) throws GraphCycleException;
}
```

Note: `GraphWriter` references `GraphMutation<N>`. At this point `GraphMutation` is not yet
parameterized (that's Task 2). For now, use `@SuppressWarnings("rawtypes")` with raw
`GraphMutation` and add a `// TODO: parameterize in Task 2` comment. Alternatively, define
the writer to accept `Object mutation` and cast in the adapter — but that's worse. The
cleanest approach: **skip `GraphWriter` in this task** and add it in Task 2 after
`GraphMutation<N>` is parameterized. The view's `withMutation()` can delegate directly to
the adapter's method without the writer interface temporarily.

**Revised approach:** Create `GraphReader`, `GraphView`, `MutableGraphView`,
`GraphCycleException` in this task. `GraphWriter` is created in Task 2 after
`GraphMutation<N>` is parameterized. The adapter implements only `GraphReader` in this task;
`GraphWriter` implementation is added in Task 2.

Create `GraphView.java`:
```java
package io.casehub.desiredstate.annotations.runtime.graph;

import java.util.Map;
import java.util.Set;

public interface GraphView<N> {
    Map<String, N> nodes();
    N node(String id);
    String nodeId(N node);
    String nodeType(N node);
    Set<String> dependenciesOf(String nodeId);
    Set<String> dependentsOf(String nodeId);
}
```

Create `MutableGraphView.java`:
```java
package io.casehub.desiredstate.annotations.runtime.graph;

public interface MutableGraphView<N> extends GraphView<N> {
    MutableGraphView<N> withMutation(Object mutation);
}
```

Note: `withMutation` takes `Object` temporarily. Signature changes to
`GraphMutation<N>` in Task 2. This lets us write and test the adapter now.

Create `GraphCycleException.java`:
```java
package io.casehub.desiredstate.annotations.runtime.graph;

import java.util.List;

public class GraphCycleException extends RuntimeException {
    private final List<String> cycle;

    public GraphCycleException(List<String> cycle) {
        super("Cyclic dependency detected: " +
              String.join(" → ", cycle));
        this.cycle = List.copyOf(cycle);
    }

    public List<String> getCycle() { return cycle; }
}
```

- [ ] **Step 4: Create DesiredStateGraphAdapter**

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.annotations.runtime.graph.GraphCycleException;
import io.casehub.desiredstate.annotations.runtime.graph.GraphReader;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

public class DesiredStateGraphAdapter implements GraphReader<DesiredStateGraph, DesiredNode> {

    @Override
    public Map<String, DesiredNode> nodes(DesiredStateGraph graph) {
        Map<String, DesiredNode> result = new LinkedHashMap<>();
        for (var entry : graph.nodes().entrySet()) {
            result.put(entry.getKey().value(), entry.getValue());
        }
        return result;
    }

    @Override
    public DesiredNode node(DesiredStateGraph graph, String id) {
        return graph.nodes().get(NodeId.of(id));
    }

    @Override
    public String nodeId(DesiredNode node) { return node.id().value(); }

    @Override
    public String nodeType(DesiredNode node) { return node.type().value(); }

    @Override
    public Set<String> dependenciesOf(DesiredStateGraph graph, String nodeId) {
        return graph.dependenciesOf(NodeId.of(nodeId)).stream()
                    .map(NodeId::value).collect(Collectors.toSet());
    }

    @Override
    public Set<String> dependentsOf(DesiredStateGraph graph, String nodeId) {
        return graph.dependentsOf(NodeId.of(nodeId)).stream()
                    .map(NodeId::value).collect(Collectors.toSet());
    }

    public DesiredStateGraph applyMutation(DesiredStateGraph graph, Object mutation) {
        try {
            return graph.withMutation((GraphMutation) mutation);
        } catch (CyclicDependencyException e) {
            throw new GraphCycleException(
                e.getCycle().stream().map(NodeId::value).toList());
        }
    }
}
```

- [ ] **Step 5: Create DesiredStateGraphView**

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.annotations.runtime.graph.MutableGraphView;

import java.util.Map;
import java.util.Set;

public class DesiredStateGraphView implements MutableGraphView<DesiredNode> {
    private final DesiredStateGraph graph;
    private final DesiredStateGraphAdapter adapter;

    public DesiredStateGraphView(DesiredStateGraph graph, DesiredStateGraphAdapter adapter) {
        this.graph = graph;
        this.adapter = adapter;
    }

    public DesiredStateGraph graph() { return graph; }

    @Override public Map<String, DesiredNode> nodes() { return adapter.nodes(graph); }
    @Override public DesiredNode node(String id) { return adapter.node(graph, id); }
    @Override public String nodeId(DesiredNode node) { return adapter.nodeId(node); }
    @Override public String nodeType(DesiredNode node) { return adapter.nodeType(node); }
    @Override public Set<String> dependenciesOf(String nodeId) {
        return adapter.dependenciesOf(graph, nodeId);
    }
    @Override public Set<String> dependentsOf(String nodeId) {
        return adapter.dependentsOf(graph, nodeId);
    }

    @Override
    public MutableGraphView<DesiredNode> withMutation(Object mutation) {
        return new DesiredStateGraphView(adapter.applyMutation(graph, mutation), adapter);
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=DesiredStateGraphViewTest`
Expected: PASS — all 9 tests green

- [ ] **Step 7: Commit**

```bash
git add annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/graph/
git add annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphAdapter.java
git add annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphView.java
git add annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphViewTest.java
git commit -m "feat(#129): add GraphView/Reader/Writer interfaces, adapter, and view

Introduce graph abstraction layer in annotations/runtime/graph subpackage:
- GraphView<N>, MutableGraphView<N> — read/write view interfaces
- GraphReader<G,N> — adapter interface for domain graph types
- GraphCycleException — generic cycle exception
- DesiredStateGraphAdapter — reader for DesiredStateGraph/DesiredNode
- DesiredStateGraphView — MutableGraphView<DesiredNode> implementation

GraphWriter<G,N> deferred to Task 2 (after GraphMutation<N> parameterization).

Refs #129"
```

---

## Batch 2: Type generification

### Task 2: Parameterize GraphMutation<N> and migrate all consumers

**Files:**
- Modify: `api/src/main/java/io/casehub/desiredstate/api/GraphMutation.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/ConflictingMutationException.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/DesiredStateGraph.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/GraphMutations.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/FaultPolicy.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/ThresholdFaultPolicy.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/TypedFaultPolicy.java`
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/ImmutableDesiredStateGraph.java`
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/FaultPolicyEngine.java`
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/GraphDiff.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlRuleConverter.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlFaultPolicyBuilder.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`
- Modify: `ts-dsl/runtime/src/main/java/io/casehub/desiredstate/ts/TsGraphRecorder.java`
- Create: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/graph/GraphWriter.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphAdapter.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphView.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/graph/MutableGraphView.java`
- Modify: All example fault policies, engine-adapter, work-adapter, test files referencing GraphMutation
- Test: Existing tests — migrate construction sites, verify all pass

**Interfaces:**
- Consumes: `GraphCycleException`, `DesiredStateGraphAdapter` from Task 1
- Produces: `GraphMutation<N>` (parameterized sealed interface), `GraphWriter<G,N>`, typed `MutableGraphView.withMutation(GraphMutation<N>)`

- [ ] **Step 1: Parameterize GraphMutation<N> in api/**

Replace the sealed interface with the parameterized version:

```java
package io.casehub.desiredstate.api;

public sealed interface GraphMutation<N> {
    default String targetNodeId() {
        return switch (this) {
            case AddNode<?> m -> null;  // needs nodeId extractor — see step 3
            case RemoveNode<?> m -> m.id();
            case UpdateNode<?> m -> m.id();
            case AddEdge<?> ignored -> null;
            case RemoveEdge<?> ignored -> null;
        };
    }

    record AddNode<N>(N node) implements GraphMutation<N> {}
    record RemoveNode<N>(String id) implements GraphMutation<N> {}
    record UpdateNode<N>(String id, N adaptedNode) implements GraphMutation<N> {}
    record AddEdge<N>(String from, String to) implements GraphMutation<N> {}
    record RemoveEdge<N>(String from, String to) implements GraphMutation<N> {}
}
```

`targetNodeId()` for `AddNode` needs a way to extract the ID from `N`. Two options:
(a) return `null` and let callers handle it, or (b) remove `targetNodeId()` from the
interface and make it a utility method that takes a `GraphView<N>`. Since `targetNodeId()`
is used by `GraphRuleEngine.detectNodeConflicts()` and `FaultPolicyEngine`, and both will
have a view available after generification, option (b) is cleaner. For now during migration,
use option (a) — return `null` for `AddNode`. The engine's `detectNodeConflicts` already
handles `null` (skips). Fix fully in Task 5 when the engine is generified.

- [ ] **Step 2: Update ConflictingMutationException**

```java
package io.casehub.desiredstate.api;

public class ConflictingMutationException extends RuntimeException {
    private final String nodeId;
    private final GraphMutation<?> mutationA;
    private final GraphMutation<?> mutationB;

    public ConflictingMutationException(String nodeId, GraphMutation<?> mutationA,
                                         GraphMutation<?> mutationB) {
        super("Conflicting mutations for node " + nodeId + ": " + mutationA + " vs " + mutationB);
        this.nodeId = nodeId;
        this.mutationA = mutationA;
        this.mutationB = mutationB;
    }

    public String getNodeId() { return nodeId; }
    public GraphMutation<?> getMutationA() { return mutationA; }
    public GraphMutation<?> getMutationB() { return mutationB; }
}
```

- [ ] **Step 3: Update DesiredStateGraph interface**

Change `withMutation` signature:
```java
DesiredStateGraph withMutation(GraphMutation<DesiredNode> mutation);
```

- [ ] **Step 4: Update ImmutableDesiredStateGraph.withMutation()**

Replace the switch expression to handle renamed variants and string IDs:
```java
@Override
public DesiredStateGraph withMutation(GraphMutation<DesiredNode> mutation) {
    return switch (mutation) {
        case GraphMutation.AddNode<DesiredNode> m -> withNode(m.node());
        case GraphMutation.RemoveNode<?> m -> withoutNode(NodeId.of(m.id()));
        case GraphMutation.UpdateNode<DesiredNode> m -> {
            if (!nodes.containsKey(NodeId.of(m.id()))) {
                throw new IllegalArgumentException("Cannot update non-existent node: " + m.id());
            }
            yield withNode(m.adaptedNode());
        }
        case GraphMutation.AddEdge<?> m ->
            withDependency(new Dependency(NodeId.of(m.from()), NodeId.of(m.to())));
        case GraphMutation.RemoveEdge<?> m ->
            withoutDependency(new Dependency(NodeId.of(m.from()), NodeId.of(m.to())));
    };
}
```

- [ ] **Step 5: Update GraphMutations utility**

```java
public static List<GraphMutation<DesiredNode>> addNodeDependingOn(DesiredNode node, NodeId dependsOn) {
    return List.of(
        new GraphMutation.AddNode<>(node),
        new GraphMutation.AddEdge<>(node.id().value(), dependsOn.value())
    );
}
```

- [ ] **Step 6: Update FaultPolicy and ThresholdFaultPolicy**

`FaultPolicy.onFault()` return type: `List<GraphMutation>` → `List<GraphMutation<DesiredNode>>`
`FaultPolicy.addReviewNode()` return type and construction sites.
`ThresholdFaultPolicy` — all `List<GraphMutation>` → `List<GraphMutation<DesiredNode>>`.
`TypedFaultPolicy` — same pattern.

- [ ] **Step 7: Update runtime consumers — FaultPolicyEngine, GraphDiff**

`FaultPolicyEngine`: `Map<NodeId, List<GraphMutation>>` → `Map<String, List<GraphMutation<DesiredNode>>>`.
Key extraction: `m.targetNodeId()` now returns `String` directly (no `.value()` needed).
`ConflictingMutationException` construction: pass `String` nodeId directly.

`GraphDiff`: construction sites per spec consumer impact section:
- `new GraphMutation.RemoveNode<>(id.value())`
- `new GraphMutation.AddEdge<>(dep.from().value(), dep.to().value())`
- `new GraphMutation.RemoveEdge<>(dep.from().value(), dep.to().value())`

- [ ] **Step 8: Update YAML surface — YamlRuleConverter, YamlFaultPolicyBuilder, YamlGraphRecorder**

`YamlRuleConverter.evaluateActions()` — simplifies (drops `NodeId.of()`/`Dependency` wrappers):
- `new GraphMutation.RemoveNode<>(id)` (was `new GraphMutation.RemoveNode(NodeId.of(id))`)
- `new GraphMutation.AddEdge<>(from, to)` (was `new GraphMutation.AddDependency(new Dependency(...))`)

`YamlFaultPolicyBuilder` — inherits `GraphMutations` changes.
`YamlGraphRecorder` — `List<GraphMutation>` → `List<GraphMutation<DesiredNode>>` in rule/invariant wiring.

- [ ] **Step 9: Update TS DSL surface — TsGraphRecorder**

Same pattern as YamlGraphRecorder — `List<GraphMutation>` → `List<GraphMutation<DesiredNode>>`.

- [ ] **Step 10: Update engine-adapter, work-adapter, examples, testing modules**

All mechanical: `GraphMutation` → `GraphMutation<DesiredNode>`, construction site updates.
Use `ide_search_text` with query `GraphMutation` scoped to each module to find all sites.

- [ ] **Step 11: Create GraphWriter<G,N> and update adapter/view**

Now that `GraphMutation<N>` exists, create the typed writer:

```java
package io.casehub.desiredstate.annotations.runtime.graph;

public interface GraphWriter<G, N> {
    G applyMutation(G graph, GraphMutation<N> mutation) throws GraphCycleException;
}
```

Update `DesiredStateGraphAdapter` to implement `GraphWriter<DesiredStateGraph, DesiredNode>`:
```java
public class DesiredStateGraphAdapter
        implements GraphReader<DesiredStateGraph, DesiredNode>,
                   GraphWriter<DesiredStateGraph, DesiredNode> {

    @Override
    public DesiredStateGraph applyMutation(DesiredStateGraph graph,
                                            GraphMutation<DesiredNode> mutation) {
        try {
            return graph.withMutation(mutation);
        } catch (CyclicDependencyException e) {
            throw new GraphCycleException(
                e.getCycle().stream().map(NodeId::value).toList());
        }
    }
    // ... reader methods unchanged
}
```

Update `MutableGraphView.withMutation` signature from `Object` to `GraphMutation<N>`.
Update `DesiredStateGraphView.withMutation` to match.

- [ ] **Step 12: Update DesiredStateGraphViewTest**

Tests from Task 1 should still compile and pass with the typed `withMutation` signature.
Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=DesiredStateGraphViewTest`
Expected: PASS

- [ ] **Step 13: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules compile, all tests pass

- [ ] **Step 14: Commit**

```bash
git add -A
git commit -m "feat(#129): parameterize GraphMutation<N> in place, migrate all consumers

GraphMutation becomes GraphMutation<N> with:
- AddNode<N>, RemoveNode<N>(String), UpdateNode<N>(String, N)
- AddEdge<N>(String, String), RemoveEdge<N>(String, String)
- All consumers migrated to GraphMutation<DesiredNode>
- ConflictingMutationException uses String nodeId
- GraphWriter<G,N> interface created
- DesiredStateGraphAdapter implements reader + writer

Refs #129"
```

### Task 3: Generify exception types

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphViolation.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphViolationException.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleCycleException.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleNonConvergenceException.java`
- Modify: All test files that construct these exceptions with `NodeId`
- Test: Existing tests — verify compilation and pass after type changes

**Interfaces:**
- Consumes: `GraphMutation<N>` from Task 2
- Produces: `GraphViolation(String, String, String, List<String>)`, `GraphViolationException(String, String...)`, `GraphRuleCycleException(List<String>, List<String>)`

- [ ] **Step 1: Update GraphViolation**

```java
package io.casehub.desiredstate.annotations.runtime;

import java.util.List;

public record GraphViolation(
        String invariantName,
        String sourceClassName,
        String message,
        List<String> affectedNodes) {}
```

- [ ] **Step 2: Update GraphViolationException**

```java
package io.casehub.desiredstate.annotations.runtime;

import java.util.List;

public class GraphViolationException extends RuntimeException {
    private final List<String> affectedNodes;

    public GraphViolationException(String message) {
        super(message);
        this.affectedNodes = List.of();
    }

    public GraphViolationException(String message, String... nodes) {
        super(message);
        this.affectedNodes = List.of(nodes);
    }

    public List<String> affectedNodes() { return affectedNodes; }
}
```

- [ ] **Step 3: Update GraphRuleCycleException**

```java
package io.casehub.desiredstate.annotations.runtime;

import java.util.List;

public class GraphRuleCycleException extends RuntimeException {
    private final List<String> ruleNames;
    private final List<String> cyclePath;

    public GraphRuleCycleException(List<String> ruleNames, List<String> cyclePath) {
        super("Graph rules introduced a cycle: "
              + String.join(" → ", cyclePath)
              + ". Rules: " + String.join(", ", ruleNames));
        this.ruleNames = ruleNames;
        this.cyclePath = cyclePath;
    }

    public List<String> getRuleNames() { return ruleNames; }
    public List<String> getCyclePath() { return cyclePath; }
}
```

- [ ] **Step 4: Update GraphRuleNonConvergenceException**

Change constructor to accept `List<? extends ResolvedRule<?>>`:
```java
public GraphRuleNonConvergenceException(List<? extends ResolvedRule<?>> activeRules,
                                         int maxIterations) {
```

- [ ] **Step 5: Fix compilation errors in engine and test files**

`GraphRuleEngine.applyMutations`: catches `CyclicDependencyException`, constructs
`GraphRuleCycleException`. Change `e.getCycle()` mapping:
```java
throw new GraphRuleCycleException(ruleNames,
    e.getCycle().stream().map(NodeId::value).toList());
```

`GraphInvariantEngine`: all `DesiredNode::id` → `node.id().value()` in violation
construction. All `anchor.stream().map(DesiredNode::id).toList()` →
`anchor.stream().map(n -> n.id().value()).toList()`.

Example invariant methods in tests/examples: `new GraphViolationException("msg", node.id())`
→ `new GraphViolationException("msg", node.id().value())`.

- [ ] **Step 6: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(#129): generify exception types to string IDs

- GraphViolation: List<NodeId> → List<String>
- GraphViolationException: NodeId... → String...
- GraphRuleCycleException: List<NodeId> → List<String>
- GraphRuleNonConvergenceException: List<ResolvedRule<?>>
- All construction sites migrated

Refs #129"
```

---

## Batch 3: Engine generification

### Task 4: Generify resolved types and pattern matching

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/ResolvedRule.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/ResolvedInvariant.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/PatternMatchingSupport.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/PatternEvaluator.java`
- Test: `annotations/runtime/src/test/java/.../PatternEvaluatorTest.java` — migrate to use views

**Interfaces:**
- Consumes: `GraphView<N>`, `MutableGraphView<N>`, `GraphMutation<N>` from Tasks 1-2
- Produces: `ResolvedRule<N>` (with `ImperativeRule<N>` carrying `Function`), `ResolvedInvariant<N>`, generic `PatternEvaluator.evaluate(GraphView<N>, ...)`, generic `PatternMatchingSupport` methods

- [ ] **Step 1: Generify ResolvedRule<N>**

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.annotations.runtime.graph.MutableGraphView;
import io.casehub.desiredstate.api.GraphMutation;

import java.lang.reflect.Method;
import java.util.List;
import java.util.Map;
import java.util.function.Function;

public sealed interface ResolvedRule<N> {
    String name();
    List<PatternParameterDescriptor> patterns();
    String[] bindingNames();

    record ImperativeRule<N>(String name,
        Function<MutableGraphView<N>, List<GraphMutation<N>>> evaluator) implements ResolvedRule<N> {
        @Override public List<PatternParameterDescriptor> patterns() { return List.of(); }
        @Override public String[] bindingNames() { return new String[0]; }
    }

    record ParameterizedRule<N>(String name, Method method, Object instance,
                                List<PatternParameterDescriptor> patterns) implements ResolvedRule<N> {
        @Override
        public String[] bindingNames() { return PatternMatchingSupport.getParameterNames(method); }
    }

    record DeclarativeRule<N>(String name, List<PatternParameterDescriptor> patterns,
                              String[] bindingNames,
                              Function<Map<String, N>, List<GraphMutation<N>>> actionEvaluator)
            implements ResolvedRule<N> {}
}
```

- [ ] **Step 2: Generify ResolvedInvariant<N>**

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.annotations.runtime.graph.GraphView;

import java.lang.reflect.Method;
import java.util.List;
import java.util.function.Consumer;

public sealed interface ResolvedInvariant<N> {
    String name();
    List<PatternParameterDescriptor> patterns();
    String[] bindingNames();

    record ImperativeInvariant<N>(String name,
        Consumer<GraphView<N>> validator) implements ResolvedInvariant<N> {
        @Override public List<PatternParameterDescriptor> patterns() { return List.of(); }
        @Override public String[] bindingNames() { return new String[0]; }
    }

    record ParameterizedReflectiveInvariant<N>(String name, Method method, Object instance,
                                               List<PatternParameterDescriptor> patterns)
            implements ResolvedInvariant<N> {
        @Override
        public String[] bindingNames() { return PatternMatchingSupport.getParameterNames(method); }
    }

    record DeclarativeInvariant<N>(String name, List<PatternParameterDescriptor> patterns,
                                   String[] bindingNames, String messageTemplate)
            implements ResolvedInvariant<N> {}
}
```

- [ ] **Step 3: Generify PatternMatchingSupport**

All methods gain `<N>` type parameter. `DesiredStateGraph` → `GraphView<N>`,
`DesiredNode` → `N`, `NodeId` → `String`, `NodeType.of(s).equals(n.type())` →
`view.nodeType(n).equals(s)`:

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.annotations.runtime.graph.GraphView;
import java.lang.reflect.Method;
import java.util.*;

public final class PatternMatchingSupport {
    private PatternMatchingSupport() {}

    public static <N> N resolveReference(PatternParameterDescriptor p, int paramIndex,
            String[] paramNames, Map<String, N> bindings) {
        if (!p.of().isEmpty()) { return bindings.get(p.of()); }
        for (int i = paramIndex - 1; i >= 0; i--) {
            N prev = bindings.get(paramNames[i]);
            if (prev != null) return prev;
        }
        return null;
    }

    public static <N> List<N> findDirectNeighbors(GraphView<N> view, N refNode,
                                                   PatternParameterDescriptor p) {
        boolean wildcard = "*".equals(p.nodeType());
        Set<String> neighbors = p.direction() == Direction.DEPENDENCIES
                                ? view.dependenciesOf(view.nodeId(refNode))
                                : view.dependentsOf(view.nodeId(refNode));
        return neighbors.stream()
                        .map(view::node)
                        .filter(n -> n != null && (wildcard || view.nodeType(n).equals(p.nodeType())))
                        .toList();
    }

    public static <N> List<N> findReachable(GraphView<N> view, N refNode,
                                             PatternParameterDescriptor p) {
        boolean wildcard = "*".equals(p.nodeType());
        List<N> found = new ArrayList<>();
        Set<String> visited = new HashSet<>();
        ArrayDeque<String> queue = new ArrayDeque<>();
        String startId = view.nodeId(refNode);
        queue.add(startId);
        visited.add(startId);

        while (!queue.isEmpty()) {
            String current = queue.poll();
            Set<String> neighbors = p.direction() == Direction.DEPENDENCIES
                                    ? view.dependenciesOf(current)
                                    : view.dependentsOf(current);
            for (String neighbor : neighbors) {
                if (visited.add(neighbor)) {
                    N node = view.node(neighbor);
                    if (node != null && (wildcard || view.nodeType(node).equals(p.nodeType()))) {
                        found.add(node);
                    }
                    queue.add(neighbor);
                }
            }
        }
        return found;
    }

    public static <N> boolean existsGlobal(GraphView<N> view, PatternParameterDescriptor p) {
        if ("*".equals(p.nodeType())) { return !view.nodes().isEmpty(); }
        return view.nodes().values().stream()
                   .anyMatch(n -> view.nodeType(n).equals(p.nodeType()));
    }

    public static <N> boolean existsRelational(GraphView<N> view, N refNode,
                                                PatternParameterDescriptor p) {
        boolean wildcard = "*".equals(p.nodeType());
        Set<String> neighbors = p.direction() == Direction.DEPENDENCIES
                                ? view.dependenciesOf(view.nodeId(refNode))
                                : view.dependentsOf(view.nodeId(refNode));
        return neighbors.stream()
                        .map(view::node)
                        .anyMatch(n -> n != null && (wildcard || view.nodeType(n).equals(p.nodeType())));
    }

    public static String[] getParameterNames(Method method) {
        var params = method.getParameters();
        String[] names = new String[params.length];
        for (int i = 0; i < params.length; i++) { names[i] = params[i].getName(); }
        return names;
    }

    public static <N> List<List<N>> crossProduct(List<List<N>> sets) {
        List<List<N>> result = new ArrayList<>();
        result.add(List.of());
        for (List<N> set : sets) {
            List<List<N>> newResult = new ArrayList<>();
            for (List<N> existing : result) {
                for (N item : set) {
                    List<N> combined = new ArrayList<>(existing);
                    combined.add(item);
                    newResult.add(combined);
                }
            }
            result = newResult;
        }
        return result;
    }
}
```

- [ ] **Step 4: Generify PatternEvaluator**

All methods gain `<N>`. `DesiredStateGraph` → `GraphView<N>`, `DesiredNode` → `N`:

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.annotations.runtime.graph.GraphView;
import java.util.*;

public final class PatternEvaluator {
    private PatternEvaluator() {}

    public static <N> List<Map<String, N>> evaluate(
            GraphView<N> view, List<PatternParameterDescriptor> patterns, String[] bindingNames) {

        List<List<N>> matchSets = new ArrayList<>();
        for (PatternParameterDescriptor p : patterns) {
            if (p.kind() == PatternKind.MATCH) {
                if ("*".equals(p.nodeType())) {
                    matchSets.add(new ArrayList<>(view.nodes().values()));
                } else {
                    matchSets.add(view.nodes().values().stream()
                            .filter(n -> view.nodeType(n).equals(p.nodeType()))
                            .toList());
                }
            }
        }

        List<Map<String, N>> results = new ArrayList<>();
        for (List<N> tuple : PatternMatchingSupport.crossProduct(matchSets)) {
            Map<String, N> bindings = new LinkedHashMap<>();
            int matchIdx = 0;
            for (int i = 0; i < patterns.size(); i++) {
                if (patterns.get(i).kind() == PatternKind.MATCH) {
                    bindings.put(bindingNames[i], tuple.get(matchIdx++));
                }
            }
            expandChain(view, patterns, bindingNames, bindings, 0, results);
        }
        return results;
    }

    private static <N> void expandChain(GraphView<N> view,
            List<PatternParameterDescriptor> patterns, String[] bindingNames,
            Map<String, N> bindings, int startIndex, List<Map<String, N>> results) {
        int idx = startIndex;
        while (idx < patterns.size() && patterns.get(idx).kind() == PatternKind.MATCH) { idx++; }
        if (idx >= patterns.size()) { results.add(new LinkedHashMap<>(bindings)); return; }

        PatternParameterDescriptor p = patterns.get(idx);
        switch (p.kind()) {
            case DIRECT_DEP -> {
                N refNode = PatternMatchingSupport.resolveReference(p, idx, bindingNames, bindings);
                for (N neighbor : PatternMatchingSupport.findDirectNeighbors(view, refNode, p)) {
                    var newBindings = new LinkedHashMap<>(bindings);
                    newBindings.put(bindingNames[idx], neighbor);
                    expandChain(view, patterns, bindingNames, newBindings, idx + 1, results);
                }
            }
            case REACHES -> {
                N refNode = PatternMatchingSupport.resolveReference(p, idx, bindingNames, bindings);
                for (N reached : PatternMatchingSupport.findReachable(view, refNode, p)) {
                    var newBindings = new LinkedHashMap<>(bindings);
                    newBindings.put(bindingNames[idx], reached);
                    expandChain(view, patterns, bindingNames, newBindings, idx + 1, results);
                }
            }
            case NOT_EXISTS -> {
                boolean exists = p.of().isEmpty()
                        ? PatternMatchingSupport.existsGlobal(view, p)
                        : PatternMatchingSupport.existsRelational(view, bindings.get(p.of()), p);
                if (exists) return;
                expandChain(view, patterns, bindingNames, bindings, idx + 1, results);
            }
            default -> throw new IllegalStateException("Unexpected pattern kind: " + p.kind());
        }
    }
}
```

- [ ] **Step 5: Fix compilation in engine files and tests**

`GraphRuleEngine` and `GraphInvariantEngine` still reference the old `ResolvedRule`/`ResolvedInvariant`
(non-generic). They will have compilation errors. These are fixed in Task 5. For now, add
`@SuppressWarnings({"rawtypes", "unchecked"})` to allow compilation while we migrate the
engines in the next task.

Update `PatternEvaluatorTest` — pass `DesiredStateGraphView` instead of `DesiredStateGraph`
to `PatternEvaluator.evaluate()`:

```java
// Before:
PatternEvaluator.evaluate(graph, patterns, paramNames);
// After:
var adapter = new DesiredStateGraphAdapter();
var view = new DesiredStateGraphView(graph, adapter);
PatternEvaluator.evaluate(view, patterns, paramNames);
```

Same for `GraphRuleTypesTest` if it calls PatternEvaluator directly.

- [ ] **Step 6: Run tests**

Run: `mvn --batch-mode test -pl annotations/runtime`
Expected: PASS (with suppressions on engines)

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(#129): generify ResolvedRule<N>, ResolvedInvariant<N>, PatternEvaluator, PatternMatchingSupport

- ResolvedRule<N>: ImperativeRule carries Function, ParameterizedRule/DeclarativeRule parameterized
- ResolvedInvariant<N>: ImperativeInvariant carries Consumer, others parameterized
- PatternMatchingSupport: all methods generic <N>, GraphView<N> instead of DesiredStateGraph
- PatternEvaluator: generic <N>, direct string type comparison

Refs #129"
```

### Task 5: Generify engines and update recorders with closures

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphRuleEngine.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/GraphInvariantEngine.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`
- Modify: `ts-dsl/runtime/src/main/java/io/casehub/desiredstate/ts/TsGraphRecorder.java`
- Test: `annotations/runtime/src/test/java/.../GraphRuleEngineTest.java` — migrate to views
- Test: `annotations/runtime/src/test/java/.../GraphInvariantEngineTest.java` — migrate to views
- Test: `yaml/runtime/src/test/java/.../YamlInvariantEvaluationTest.java` — migrate to views

**Interfaces:**
- Consumes: `ResolvedRule<N>`, `ResolvedInvariant<N>`, `GraphView<N>`, `MutableGraphView<N>`, `PatternEvaluator`, `PatternMatchingSupport` from Tasks 1-4
- Produces: `GraphRuleEngine.evaluate(MutableGraphView<N>, List<ResolvedRule<N>>)`, `GraphInvariantEngine.validate(GraphView<N>, List<ResolvedInvariant<N>>)`, recorder closures

- [ ] **Step 1: Generify GraphRuleEngine**

Replace entire class body. Key changes:
- `evaluate` signature: `<N> MutableGraphView<N> evaluate(MutableGraphView<N> view, List<ResolvedRule<N>> rules)`
- `evaluateRule` dispatch uses new sealed variants including `ImperativeRule.evaluator().apply(view)`
- `detectNodeConflicts`: `Map<NodeId, GraphMutation>` → `Map<String, GraphMutation<N>>`, uses `m.targetNodeId()` (returns `String`)
- `detectEdgeConflicts`: `Map<Dependency, GraphMutation>` → `Map<Map.Entry<String,String>, GraphMutation<N>>`, keys on `Map.entry(add.from(), add.to())`
- `filterNoOps`: edge checks use `view.dependenciesOf()` per spec
- `applyMutations`: catches `GraphCycleException` (from view.withMutation) instead of `CyclicDependencyException`
- `sortByType`: `AddDependency`/`RemoveDependency` → `AddEdge`/`RemoveEdge`

Full implementation follows the spec's "Engine generification" section. The structure is
identical to the current code — only types change.

- [ ] **Step 2: Generify GraphInvariantEngine**

Key changes:
- `validate` signature: `<N> void validate(GraphView<N> view, List<ResolvedInvariant<N>> invariants)`
- `validateImperative`: calls `invariant.validator().accept(view)` (closure from D1)
- `validateParameterized`: threads `GraphView<N> view` to all helper methods
- `resolveMatchTemplate`: `node.id().value()` → `view.nodeId(node)`, `node.type().value()` → `view.nodeType(node)`
- `buildExpectedAnchors`: `graph.nodes()` → `view.nodes()`, type matching via `view.nodeType(n).equals(p.nodeType())`
- `countMatchingNodes`: same pattern
- `checkExpansionCardinality`: `n.id().value()` → `view.nodeId(n)`, `DesiredNode::id` → `view::nodeId`

- [ ] **Step 3: Update DesiredStateGraphRecorder with closures**

In `resolveRules()`:
```java
if (grd.imperative()) {
    Method ruleMethod = findRuleMethod(ruleClass, grd);
    Object inst = ruleInstance;
    Function<MutableGraphView<DesiredNode>, List<GraphMutation<DesiredNode>>> evaluator = view -> {
        try {
            DesiredStateGraph graph = ((DesiredStateGraphView) view).graph();
            @SuppressWarnings("unchecked")
            var result = (List<GraphMutation<DesiredNode>>) ruleMethod.invoke(inst, graph);
            return result != null ? result : List.of();
        } catch (java.lang.reflect.InvocationTargetException e) {
            if (e.getCause() instanceof RuntimeException re) throw re;
            throw new RuntimeException("Rule " + grd.methodName() + " failed", e.getCause());
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Rule " + grd.methodName() + " inaccessible", e);
        }
    };
    rules.add(new ResolvedRule.ImperativeRule<>(grd.methodName(), evaluator));
} else {
    rules.add(new ResolvedRule.ParameterizedRule<>(grd.methodName(), ruleMethod,
                                                    ruleInstance, grd.patterns()));
}
```

In `resolveInvariants()`:
```java
if (gid.imperative()) {
    Method invMethod = findInvariantMethod(cls, gid);
    Object inst = instance;
    Consumer<GraphView<DesiredNode>> validator = view -> {
        try {
            DesiredStateGraph graph = ((DesiredStateGraphView) view).graph();
            if (inst != null) invMethod.invoke(inst, graph);
            else invMethod.invoke(null, graph);
        } catch (java.lang.reflect.InvocationTargetException e) {
            if (e.getCause() instanceof GraphViolationException gve) throw gve;
            if (e.getCause() instanceof RuntimeException re) throw re;
            throw new RuntimeException("Invariant " + gid.methodName() + " failed", e.getCause());
        } catch (Exception e) {
            throw new RuntimeException("Invariant " + gid.methodName() + " failed", e);
        }
    };
    invariants.add(new ResolvedInvariant.ImperativeInvariant<>(gid.methodName(), validator));
}
```

In `applyGraphRulesToResult()`:
```java
private CompilationResult applyGraphRulesToResult(CompilationResult result,
                                                   List<ResolvedRule<DesiredNode>> rules) {
    GraphRuleEngine engine = new GraphRuleEngine();
    DesiredStateGraphAdapter adapter = new DesiredStateGraphAdapter();
    return switch (result) {
        case CompilationResult.SingleGraph sg -> {
            var view = new DesiredStateGraphView(sg.graph(), adapter);
            var evaluated = engine.evaluate(view, rules);
            yield CompilationResult.single(((DesiredStateGraphView) evaluated).graph());
        }
        case CompilationResult.Lifecycle lc -> {
            List<Phase> rewritten = new ArrayList<>();
            for (Phase phase : lc.phases()) {
                var view = new DesiredStateGraphView(phase.graph(), adapter);
                var evaluated = engine.evaluate(view, rules);
                rewritten.add(new Phase(phase.id(),
                    ((DesiredStateGraphView) evaluated).graph(), phase.completionCondition()));
            }
            yield CompilationResult.lifecycle(rewritten);
        }
    };
}
```

In `validateInvariantsOnResult()`:
```java
private CompilationResult validateInvariantsOnResult(CompilationResult result,
                                                      List<ResolvedInvariant<DesiredNode>> invariants,
                                                      GraphInvariantEngine engine) {
    DesiredStateGraphAdapter adapter = new DesiredStateGraphAdapter();
    switch (result) {
        case CompilationResult.SingleGraph sg ->
            engine.validate(new DesiredStateGraphView(sg.graph(), adapter), invariants);
        case CompilationResult.Lifecycle lc -> {
            for (Phase phase : lc.phases()) {
                engine.validate(new DesiredStateGraphView(phase.graph(), adapter), invariants);
            }
        }
    }
    return result;
}
```

- [ ] **Step 4: Update YamlGraphRecorder and TsGraphRecorder**

Same pattern as DesiredStateGraphRecorder: create adapter + view, pass to engines, extract
graph from returned view. The YAML recorder's declarative rules use
`ResolvedRule.DeclarativeRule<DesiredNode>` and `ResolvedInvariant.DeclarativeInvariant<DesiredNode>`.

- [ ] **Step 5: Migrate GraphRuleEngineTest**

Tests create `ImmutableDesiredStateGraph` and call `engine.evaluate(graph, rules)`.
Change to wrap in view:

```java
private final DesiredStateGraphAdapter adapter = new DesiredStateGraphAdapter();

private DesiredStateGraph evaluate(DesiredStateGraph graph, List<ResolvedRule<DesiredNode>> rules) {
    var view = new DesiredStateGraphView(graph, adapter);
    var result = engine.evaluate(view, rules);
    return ((DesiredStateGraphView) result).graph();
}
```

Imperative test rules (e.g. `addMonitorRule`) need wrapping in `ImperativeRule` closures:
```java
ResolvedRule<DesiredNode> rule = new ResolvedRule.ImperativeRule<>("addMonitor", view -> {
    DesiredStateGraph g = ((DesiredStateGraphView) view).graph();
    return addMonitorRule(g);
});
```

Parameterized test rules use `ResolvedRule.ParameterizedRule<DesiredNode>`.

- [ ] **Step 6: Migrate GraphInvariantEngineTest and YamlInvariantEvaluationTest**

Same pattern — wrap graphs in views before passing to `engine.validate()`.

- [ ] **Step 7: Remove all @SuppressWarnings added in Task 4**

Clean up any temporary suppressions.

- [ ] **Step 8: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules compile, all tests pass

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#129): generify engines, update recorders with function closures

- GraphRuleEngine: <N> MutableGraphView<N> evaluate(MutableGraphView<N>, List<ResolvedRule<N>>)
- GraphInvariantEngine: <N> void validate(GraphView<N>, List<ResolvedInvariant<N>>)
- DesiredStateGraphRecorder: imperative closures extract graph from view
- YamlGraphRecorder, TsGraphRecorder: same closure pattern
- All engine tests migrated to use views

Refs #129"
```

---

## Batch 4: Verification and documentation

### Task 6: Import verification, full build, documentation update

**Files:**
- Verify: `annotations/runtime/src/main/java/.../GraphRuleEngine.java` — zero domain imports
- Verify: `annotations/runtime/src/main/java/.../GraphInvariantEngine.java` — zero domain imports
- Verify: `annotations/runtime/src/main/java/.../PatternEvaluator.java` — zero domain imports
- Verify: `annotations/runtime/src/main/java/.../PatternMatchingSupport.java` — zero domain imports
- Modify: `CLAUDE.md` — update module table for annotations/runtime
- Modify: `docs/guides/consumer-guide.md` — note GraphMutation<N> parameterization

**Interfaces:**
- Consumes: All tasks complete
- Produces: Verified import-clean engines, updated documentation

- [ ] **Step 1: Verify zero domain-specific imports in engines**

Use `ide_search_text` to scan each engine file for `io.casehub.desiredstate.api` imports:

```
ide_search_text(query="io.casehub.desiredstate.api", filePattern="GraphRuleEngine.java")
ide_search_text(query="io.casehub.desiredstate.api", filePattern="GraphInvariantEngine.java")
ide_search_text(query="io.casehub.desiredstate.api", filePattern="PatternEvaluator.java")
ide_search_text(query="io.casehub.desiredstate.api", filePattern="PatternMatchingSupport.java")
```

Expected results:
- `GraphRuleEngine.java`: only `GraphMutation` and `ConflictingMutationException` (generic types, move at extraction)
- `GraphInvariantEngine.java`: zero api imports
- `PatternEvaluator.java`: zero api imports
- `PatternMatchingSupport.java`: zero api imports

If any domain-specific import (`DesiredNode`, `DesiredStateGraph`, `NodeId`, `NodeType`,
`Dependency`) remains, fix it.

- [ ] **Step 2: Verify subpackage structure**

Confirm `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/graph/`
contains: `GraphView.java`, `MutableGraphView.java`, `GraphReader.java`, `GraphWriter.java`,
`GraphCycleException.java`. These are the types that move to graph-core at extraction.

- [ ] **Step 3: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 4: Update CLAUDE.md module table**

Add to the `annotations/runtime` entry:
- New types: `GraphView<N>`, `MutableGraphView<N>`, `GraphReader<G,N>`, `GraphWriter<G,N>`, `GraphCycleException` in `graph` subpackage
- New types: `DesiredStateGraphAdapter`, `DesiredStateGraphView`
- Updated: `GraphRuleEngine`, `GraphInvariantEngine` now generic via `GraphView<N>`

- [ ] **Step 5: Update consumer guide**

Note `GraphMutation<N>` parameterization in the consumer guide's relevant section.
Note variant renames: `AddDependency` → `AddEdge`, `RemoveDependency` → `RemoveEdge`.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#129): verify zero domain imports in engines, update docs

All four engine classes verified: zero imports of DesiredNode, DesiredStateGraph,
NodeId, NodeType, Dependency. GraphMutation and ConflictingMutationException remain
as generic types (move to graph-core at platform#267).

Refs #129"
```

## References

- `specs/issue-138-runtime-polish/2026-09-04-graphview-reader-pattern-design.md` — design spec
- `specs/issue-138-runtime-polish/decisions.md` — D1-D5 decisions
- `docs/specs/issue-128-migrate-yaml-core/2026-09-02-yaml-core-migration-context.md` §5 — migration path
- `annotations/runtime/src/main/java/.../GraphRuleEngine.java` — primary engine
- `annotations/runtime/src/main/java/.../GraphInvariantEngine.java` — validation engine
- `annotations/runtime/src/main/java/.../PatternMatchingSupport.java` — pattern primitives
- `annotations/runtime/src/main/java/.../PatternEvaluator.java` — pattern evaluation
- `annotations/runtime/src/main/java/.../DesiredStateGraphRecorder.java` — recorder (closure site)
- `api/src/main/java/.../GraphMutation.java` — sealed mutation interface
- GitHub #129 — focal issue
- GitHub #138 — runtime polish epic
- Platform#267 — graph-core extraction (step 2)
