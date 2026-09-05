# Durable ReconciliationStateStore Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #132 — Durable ReconciliationStateStore JPA
**Issue group:** #138, #129, #132

**Goal:** JPA-backed ReconciliationStateStore in persistence-jpa/ that
survives JVM restarts, following the established JpaFaultCountStore
pattern.

**Architecture:** Single JPA entity per tenant storing the full
DesiredStateGraph serialized as JSON (TEXT column). Jackson ObjectMapper
handles serialization with FQCN discriminator for polymorphic NodeSpec.
Graph reconstruction via DesiredStateGraphFactory. Classpath-activated —
displaces DefaultReconciliationStateStore.

**Tech Stack:** JPA (Hibernate), Jackson (jackson-databind), Flyway, H2
(test), Quarkus CDI

## Global Constraints

- Portable SQL: H2 `MODE=PostgreSQL` + PostgreSQL
- Flyway version continues from V1 (V2__create_reconciliation_state.sql)
- `api/` must remain free of Jackson annotations
- persistence-jpa/ depends only on api/ + jackson-databind (no yaml/ or
  annotations/ dependency)

---

## Batch 1: Entity, Migration, and Serializer

### Task 1: Flyway migration, JPA entity, and GraphSerializer

**Files:**
- Create: `persistence-jpa/src/main/resources/db/desiredstate/migration/V2__create_reconciliation_state.sql`
- Create: `persistence-jpa/src/main/java/io/casehub/desiredstate/persistence/jpa/ReconciliationStateEntity.java`
- Create: `persistence-jpa/src/main/java/io/casehub/desiredstate/persistence/jpa/GraphSerializer.java`
- Create: `persistence-jpa/src/test/java/io/casehub/desiredstate/persistence/jpa/GraphSerializerTest.java`
- Modify: `persistence-jpa/pom.xml` (add jackson-databind dependency)

**Interfaces:**
- Consumes: `DesiredStateGraph`, `DesiredNode`, `NodeId`, `NodeType`, `Dependency`, `HumanGating`, `NodeSpec`, `HookDescriptor`, `LifecycleStep` (all from api/)
- Consumes: `DesiredStateGraphFactory` (from api/, CDI-injected at store level)
- Produces: `GraphSerializer.serialize(DesiredStateGraph) → String` and `GraphSerializer.deserialize(String, DesiredStateGraphFactory) → DesiredStateGraph` (package-private, used by Task 2)
- Produces: `ReconciliationStateEntity` (JPA entity with `tenancyId`, `graphJson`, `updatedAt`)

- [ ] **Step 1: Add jackson-databind dependency to persistence-jpa/pom.xml**

Add to the `<dependencies>` section, after the `quarkus-arc` dependency:

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```

No version — managed by Quarkus BOM.

- [ ] **Step 2: Create the Flyway migration**

Create `persistence-jpa/src/main/resources/db/desiredstate/migration/V2__create_reconciliation_state.sql`:

```sql
CREATE TABLE ds_reconciliation_state (
    tenancy_id   VARCHAR(255) PRIMARY KEY,
    graph_json   TEXT NOT NULL,
    updated_at   TIMESTAMP NOT NULL
);
```

- [ ] **Step 3: Create the JPA entity**

Create `persistence-jpa/src/main/java/io/casehub/desiredstate/persistence/jpa/ReconciliationStateEntity.java`:

```java
package io.casehub.desiredstate.persistence.jpa;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import java.time.Instant;

@Entity
@Table(name = "ds_reconciliation_state")
public class ReconciliationStateEntity {

    @Id
    @Column(name = "tenancy_id")
    String tenancyId;

    @Column(name = "graph_json", nullable = false, columnDefinition = "TEXT")
    String graphJson;

    @Column(name = "updated_at", nullable = false)
    Instant updatedAt;
}
```

- [ ] **Step 4: Write the failing GraphSerializer test**

Create `persistence-jpa/src/test/java/io/casehub/desiredstate/persistence/jpa/GraphSerializerTest.java`.
This is a unit test (no `@QuarkusTest`) — GraphSerializer is a plain class.

```java
package io.casehub.desiredstate.persistence.jpa;

import io.casehub.desiredstate.api.*;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class GraphSerializerTest {

    private final GraphSerializer serializer = new GraphSerializer();

    @NodeTypeId("test-alpha")
    public record AlphaSpec(String label, int priority) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("test-alpha"); }
    }

    @NodeTypeId("test-beta")
    public record BetaSpec(String region) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("test-beta"); }
    }

    @Test
    void roundTrip_multipleNodeTypes_withDependencies() {
        DesiredNode alpha = new DesiredNode(NodeId.of("a1"), new AlphaSpec("first", 1), HumanGating.NONE);
        DesiredNode beta = new DesiredNode(NodeId.of("b1"), new BetaSpec("eu-west"), HumanGating.PROVISION_ONLY);
        Dependency dep = new Dependency(NodeId.of("b1"), NodeId.of("a1"));

        DesiredStateGraphFactory factory = new TestGraphFactory();
        DesiredStateGraph original = factory.of(List.of(alpha, beta), List.of(dep));

        String json = serializer.serialize(original);
        DesiredStateGraph restored = serializer.deserialize(json, factory);

        assertThat(restored.nodes()).hasSize(2);
        assertThat(restored.dependencies()).containsExactly(dep);

        DesiredNode restoredAlpha = restored.nodes().get(NodeId.of("a1"));
        assertThat(restoredAlpha.spec()).isInstanceOf(AlphaSpec.class);
        AlphaSpec alphaSpec = (AlphaSpec) restoredAlpha.spec();
        assertThat(alphaSpec.label()).isEqualTo("first");
        assertThat(alphaSpec.priority()).isEqualTo(1);
        assertThat(restoredAlpha.humanGating()).isEqualTo(HumanGating.NONE);

        DesiredNode restoredBeta = restored.nodes().get(NodeId.of("b1"));
        assertThat(restoredBeta.spec()).isInstanceOf(BetaSpec.class);
        assertThat(((BetaSpec) restoredBeta.spec()).region()).isEqualTo("eu-west");
        assertThat(restoredBeta.humanGating()).isEqualTo(HumanGating.PROVISION_ONLY);
    }

    @Test
    void roundTrip_emptyGraph() {
        DesiredStateGraphFactory factory = new TestGraphFactory();
        DesiredStateGraph original = factory.of(List.of(), List.of());

        String json = serializer.serialize(original);
        DesiredStateGraph restored = serializer.deserialize(json, factory);

        assertThat(restored.nodes()).isEmpty();
        assertThat(restored.dependencies()).isEmpty();
    }

    @Test
    void roundTrip_withHooks() {
        HookDescriptor hooks = new HookDescriptor(
            List.of(new LifecycleStep.Verify("http://health", 30)),
            List.of(new LifecycleStep.Notify("ops", "deployed")),
            List.of(),
            List.of(new LifecycleStep.Wait(10))
        );
        DesiredNode node = new DesiredNode(NodeId.of("n1"), new AlphaSpec("hooked", 5), HumanGating.ALL, hooks);

        DesiredStateGraphFactory factory = new TestGraphFactory();
        DesiredStateGraph original = factory.of(List.of(node), List.of());

        String json = serializer.serialize(original);
        DesiredStateGraph restored = serializer.deserialize(json, factory);

        DesiredNode restoredNode = restored.nodes().get(NodeId.of("n1"));
        assertThat(restoredNode.hooks()).isNotNull();
        assertThat(restoredNode.hooks().provisionPre()).hasSize(1);
        assertThat(restoredNode.hooks().provisionPre().getFirst()).isInstanceOf(LifecycleStep.Verify.class);
        LifecycleStep.Verify verify = (LifecycleStep.Verify) restoredNode.hooks().provisionPre().getFirst();
        assertThat(verify.url()).isEqualTo("http://health");
        assertThat(verify.timeoutSeconds()).isEqualTo(30);
        assertThat(restoredNode.hooks().provisionPost()).hasSize(1);
        assertThat(restoredNode.hooks().deprovisionPost()).hasSize(1);
        assertThat(restoredNode.hooks().deprovisionPre()).isEmpty();
    }

    @Test
    void deserialize_returnsNull_onMalformedJson() {
        DesiredStateGraphFactory factory = new TestGraphFactory();
        DesiredStateGraph result = serializer.deserialize("not valid json {{{", factory);
        assertThat(result).isNull();
    }

    @Test
    void deserialize_returnsNull_onUnknownSpecClass() {
        String json = """
            {"nodes":[{"id":"x1","specClass":"com.nonexistent.Spec","spec":{},"humanGating":"NONE","hooks":null}],"dependencies":[]}
            """;
        DesiredStateGraphFactory factory = new TestGraphFactory();
        DesiredStateGraph result = serializer.deserialize(json, factory);
        assertThat(result).isNull();
    }

    /**
     * Minimal graph factory for unit tests — builds ImmutableDesiredStateGraph
     * via the withNode/withDependency chain.
     */
    private static class TestGraphFactory implements DesiredStateGraphFactory {
        @Override
        public DesiredStateGraph empty() {
            return io.casehub.desiredstate.runtime.ImmutableDesiredStateGraph.empty();
        }

        @Override
        public DesiredStateGraph of(java.util.Collection<DesiredNode> nodes, java.util.Collection<Dependency> deps) {
            DesiredStateGraph g = empty();
            for (DesiredNode n : nodes) g = g.withNode(n);
            for (Dependency d : deps) g = g.withDependency(d);
            return g;
        }
    }
}
```

- [ ] **Step 5: Run the test to verify it fails**

Run: `mvn --batch-mode test -pl persistence-jpa -Dtest=GraphSerializerTest`
Expected: compilation failure — `GraphSerializer` class does not exist.

- [ ] **Step 6: Implement GraphSerializer**

Create `persistence-jpa/src/main/java/io/casehub/desiredstate/persistence/jpa/GraphSerializer.java`:

```java
package io.casehub.desiredstate.persistence.jpa;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.desiredstate.api.*;

import java.util.ArrayList;
import java.util.List;
import java.util.logging.Level;
import java.util.logging.Logger;

class GraphSerializer {

    private static final Logger LOG = Logger.getLogger(GraphSerializer.class.getName());
    private final ObjectMapper mapper = new ObjectMapper();

    String serialize(DesiredStateGraph graph) {
        ObjectNode root = mapper.createObjectNode();

        ArrayNode nodesArray = root.putArray("nodes");
        for (var entry : graph.nodes().entrySet()) {
            DesiredNode node = entry.getValue();
            ObjectNode nodeObj = nodesArray.addObject();
            nodeObj.put("id", node.id().value());
            nodeObj.put("specClass", node.spec().getClass().getName());
            nodeObj.set("spec", mapper.valueToTree(node.spec()));
            nodeObj.put("humanGating", node.humanGating().name());
            if (node.hooks() != null && !node.hooks().isEmpty()) {
                nodeObj.set("hooks", serializeHooks(node.hooks()));
            } else {
                nodeObj.putNull("hooks");
            }
        }

        ArrayNode depsArray = root.putArray("dependencies");
        for (Dependency dep : graph.dependencies()) {
            ObjectNode depObj = depsArray.addObject();
            depObj.put("from", dep.from().value());
            depObj.put("to", dep.to().value());
        }

        try {
            return mapper.writeValueAsString(root);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to serialize DesiredStateGraph", e);
        }
    }

    DesiredStateGraph deserialize(String json, DesiredStateGraphFactory factory) {
        try {
            JsonNode root = mapper.readTree(json);

            List<DesiredNode> nodes = new ArrayList<>();
            for (JsonNode nodeJson : root.get("nodes")) {
                String id = nodeJson.get("id").asText();
                String specClassName = nodeJson.get("specClass").asText();
                Class<?> specClass = Class.forName(specClassName);
                NodeSpec spec = (NodeSpec) mapper.treeToValue(nodeJson.get("spec"), specClass);
                HumanGating gating = HumanGating.valueOf(nodeJson.get("humanGating").asText());
                HookDescriptor hooks = deserializeHooks(nodeJson.get("hooks"));
                nodes.add(new DesiredNode(NodeId.of(id), spec, gating, hooks));
            }

            List<Dependency> deps = new ArrayList<>();
            for (JsonNode depJson : root.get("dependencies")) {
                deps.add(new Dependency(
                    NodeId.of(depJson.get("from").asText()),
                    NodeId.of(depJson.get("to").asText())
                ));
            }

            return factory.of(nodes, deps);
        } catch (Exception e) {
            LOG.log(Level.WARNING, "Failed to deserialize DesiredStateGraph: " + e.getMessage(), e);
            return null;
        }
    }

    private ObjectNode serializeHooks(HookDescriptor hooks) {
        ObjectNode obj = mapper.createObjectNode();
        obj.set("provisionPre", serializeSteps(hooks.provisionPre()));
        obj.set("provisionPost", serializeSteps(hooks.provisionPost()));
        obj.set("deprovisionPre", serializeSteps(hooks.deprovisionPre()));
        obj.set("deprovisionPost", serializeSteps(hooks.deprovisionPost()));
        return obj;
    }

    private ArrayNode serializeSteps(List<LifecycleStep> steps) {
        ArrayNode arr = mapper.createArrayNode();
        for (LifecycleStep step : steps) {
            ObjectNode stepObj = arr.addObject();
            stepObj.put("stepClass", step.getClass().getName());
            stepObj.set("data", mapper.valueToTree(step));
        }
        return arr;
    }

    private HookDescriptor deserializeHooks(JsonNode hooksJson) throws Exception {
        if (hooksJson == null || hooksJson.isNull()) return null;
        return new HookDescriptor(
            deserializeSteps(hooksJson.get("provisionPre")),
            deserializeSteps(hooksJson.get("provisionPost")),
            deserializeSteps(hooksJson.get("deprovisionPre")),
            deserializeSteps(hooksJson.get("deprovisionPost"))
        );
    }

    private List<LifecycleStep> deserializeSteps(JsonNode stepsJson) throws Exception {
        if (stepsJson == null || stepsJson.isNull()) return List.of();
        List<LifecycleStep> steps = new ArrayList<>();
        for (JsonNode stepJson : stepsJson) {
            String stepClassName = stepJson.get("stepClass").asText();
            Class<?> stepClass = Class.forName(stepClassName);
            steps.add((LifecycleStep) mapper.treeToValue(stepJson.get("data"), stepClass));
        }
        return steps;
    }
}
```

- [ ] **Step 7: Run the tests to verify they pass**

Run: `mvn --batch-mode test -pl persistence-jpa -Dtest=GraphSerializerTest`
Expected: all 5 tests PASS.

Note: the test uses `ImmutableDesiredStateGraph.empty()` from `runtime/`.
If this causes a classpath issue in the test, add runtime/ as a test-scope
dependency in `persistence-jpa/pom.xml`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-desiredstate</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 8: Commit**

```bash
git add persistence-jpa/pom.xml \
      persistence-jpa/src/main/resources/db/desiredstate/migration/V2__create_reconciliation_state.sql \
      persistence-jpa/src/main/java/io/casehub/desiredstate/persistence/jpa/ReconciliationStateEntity.java \
      persistence-jpa/src/main/java/io/casehub/desiredstate/persistence/jpa/GraphSerializer.java \
      persistence-jpa/src/test/java/io/casehub/desiredstate/persistence/jpa/GraphSerializerTest.java
git commit -m "feat(#132): add GraphSerializer, entity, and Flyway V2 for ReconciliationStateStore

Refs #132"
```

---

## Batch 2: JPA Store and Integration Test

### Task 2: JpaReconciliationStateStore and integration test

**Files:**
- Create: `persistence-jpa/src/main/java/io/casehub/desiredstate/persistence/jpa/JpaReconciliationStateStore.java`
- Create: `persistence-jpa/src/test/java/io/casehub/desiredstate/persistence/jpa/JpaReconciliationStateStoreTest.java`

**Interfaces:**
- Consumes: `ReconciliationStateStore` (api/ — SPI interface)
- Consumes: `GraphSerializer` (from Task 1)
- Consumes: `ReconciliationStateEntity` (from Task 1)
- Consumes: `DesiredStateGraphFactory` (api/, CDI-injected)
- Produces: `JpaReconciliationStateStore` — `@ApplicationScoped`, displaces `DefaultReconciliationStateStore`

- [ ] **Step 1: Write the failing integration test**

Create `persistence-jpa/src/test/java/io/casehub/desiredstate/persistence/jpa/JpaReconciliationStateStoreTest.java`:

```java
package io.casehub.desiredstate.persistence.jpa;

import io.casehub.desiredstate.api.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class JpaReconciliationStateStoreTest {

    @Inject
    JpaReconciliationStateStore store;

    @Inject
    DesiredStateGraphFactory graphFactory;

    @NodeTypeId("jpa-test")
    public record JpaTestSpec(String name, int value) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("jpa-test"); }
    }

    @BeforeEach
    void clean() {
        store.remove("t1");
        store.remove("t2");
    }

    @Test
    void load_returnsEmpty_whenNothingStored() {
        Optional<DesiredStateGraph> result = store.load("t1");
        assertThat(result).isEmpty();
    }

    @Test
    void store_thenLoad_roundTrips() {
        DesiredNode n1 = new DesiredNode(NodeId.of("a"), new JpaTestSpec("alpha", 1), HumanGating.NONE);
        DesiredNode n2 = new DesiredNode(NodeId.of("b"), new JpaTestSpec("beta", 2), HumanGating.PROVISION_ONLY);
        Dependency dep = new Dependency(NodeId.of("b"), NodeId.of("a"));
        DesiredStateGraph graph = graphFactory.of(List.of(n1, n2), List.of(dep));

        store.store("t1", graph);

        Optional<DesiredStateGraph> loaded = store.load("t1");
        assertThat(loaded).isPresent();
        DesiredStateGraph restored = loaded.get();
        assertThat(restored.nodes()).hasSize(2);
        assertThat(restored.dependencies()).containsExactly(dep);

        DesiredNode restoredA = restored.nodes().get(NodeId.of("a"));
        assertThat(restoredA.spec()).isInstanceOf(JpaTestSpec.class);
        assertThat(((JpaTestSpec) restoredA.spec()).name()).isEqualTo("alpha");
        assertThat(restoredA.humanGating()).isEqualTo(HumanGating.NONE);

        DesiredNode restoredB = restored.nodes().get(NodeId.of("b"));
        assertThat(((JpaTestSpec) restoredB.spec()).value()).isEqualTo(2);
        assertThat(restoredB.humanGating()).isEqualTo(HumanGating.PROVISION_ONLY);
    }

    @Test
    void store_overwritesPreviousValue() {
        DesiredNode n1 = new DesiredNode(NodeId.of("a"), new JpaTestSpec("first", 1), HumanGating.NONE);
        DesiredNode n2 = new DesiredNode(NodeId.of("b"), new JpaTestSpec("second", 2), HumanGating.NONE);
        DesiredStateGraph graph1 = graphFactory.of(List.of(n1), List.of());
        DesiredStateGraph graph2 = graphFactory.of(List.of(n2), List.of());

        store.store("t1", graph1);
        store.store("t1", graph2);

        Optional<DesiredStateGraph> loaded = store.load("t1");
        assertThat(loaded).isPresent();
        assertThat(loaded.get().nodes()).hasSize(1);
        assertThat(loaded.get().nodes().containsKey(NodeId.of("b"))).isTrue();
    }

    @Test
    void remove_clearsStoredGraph() {
        DesiredNode n = new DesiredNode(NodeId.of("a"), new JpaTestSpec("x", 0), HumanGating.NONE);
        store.store("t1", graphFactory.of(List.of(n), List.of()));

        store.remove("t1");

        assertThat(store.load("t1")).isEmpty();
    }

    @Test
    void tenantIsolation() {
        DesiredNode n1 = new DesiredNode(NodeId.of("a"), new JpaTestSpec("t1-node", 1), HumanGating.NONE);
        DesiredNode n2 = new DesiredNode(NodeId.of("b"), new JpaTestSpec("t2-node", 2), HumanGating.NONE);
        store.store("t1", graphFactory.of(List.of(n1), List.of()));
        store.store("t2", graphFactory.of(List.of(n2), List.of()));

        assertThat(store.load("t1").get().nodes()).hasSize(1);
        assertThat(store.load("t1").get().nodes().containsKey(NodeId.of("a"))).isTrue();
        assertThat(store.load("t2").get().nodes().containsKey(NodeId.of("b"))).isTrue();

        store.remove("t1");
        assertThat(store.load("t1")).isEmpty();
        assertThat(store.load("t2")).isPresent();
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `mvn --batch-mode test -pl persistence-jpa -Dtest=JpaReconciliationStateStoreTest`
Expected: compilation failure — `JpaReconciliationStateStore` class does not exist.

- [ ] **Step 3: Implement JpaReconciliationStateStore**

Create `persistence-jpa/src/main/java/io/casehub/desiredstate/persistence/jpa/JpaReconciliationStateStore.java`:

```java
package io.casehub.desiredstate.persistence.jpa;

import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.DesiredStateGraphFactory;
import io.casehub.desiredstate.api.ReconciliationStateStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import java.time.Instant;
import java.util.Optional;

@ApplicationScoped
public class JpaReconciliationStateStore implements ReconciliationStateStore {

    @Inject
    EntityManager em;

    @Inject
    DesiredStateGraphFactory graphFactory;

    private final GraphSerializer serializer = new GraphSerializer();

    @Override
    @Transactional
    public void store(String tenancyId, DesiredStateGraph lastReconciledDesired) {
        String json = serializer.serialize(lastReconciledDesired);
        ReconciliationStateEntity entity = em.find(ReconciliationStateEntity.class, tenancyId);
        if (entity == null) {
            entity = new ReconciliationStateEntity();
            entity.tenancyId = tenancyId;
            entity.graphJson = json;
            entity.updatedAt = Instant.now();
            em.persist(entity);
        } else {
            entity.graphJson = json;
            entity.updatedAt = Instant.now();
        }
        em.flush();
    }

    @Override
    public Optional<DesiredStateGraph> load(String tenancyId) {
        ReconciliationStateEntity entity = em.find(ReconciliationStateEntity.class, tenancyId);
        if (entity == null) {
            return Optional.empty();
        }
        DesiredStateGraph graph = serializer.deserialize(entity.graphJson, graphFactory);
        return Optional.ofNullable(graph);
    }

    @Override
    @Transactional
    public void remove(String tenancyId) {
        ReconciliationStateEntity entity = em.find(ReconciliationStateEntity.class, tenancyId);
        if (entity != null) {
            em.remove(entity);
        }
    }
}
```

- [ ] **Step 4: Run the integration tests to verify they pass**

Run: `mvn --batch-mode test -pl persistence-jpa -Dtest=JpaReconciliationStateStoreTest`
Expected: all 5 tests PASS.

If `DesiredStateGraphFactory` CDI injection fails (no bean in test context),
add runtime/ as a test-scope dependency (if not already added in Task 1 Step 7):

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-desiredstate</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 5: Run the full persistence-jpa test suite**

Run: `mvn --batch-mode test -pl persistence-jpa`
Expected: all tests pass (both JpaFaultCountStoreTest and JpaReconciliationStateStoreTest).

- [ ] **Step 6: Commit**

```bash
git add persistence-jpa/src/main/java/io/casehub/desiredstate/persistence/jpa/JpaReconciliationStateStore.java \
      persistence-jpa/src/test/java/io/casehub/desiredstate/persistence/jpa/JpaReconciliationStateStoreTest.java
git commit -m "feat(#132): add JpaReconciliationStateStore with integration tests

Classpath-activated — displaces DefaultReconciliationStateStore when
persistence-jpa is present.

Refs #132"
```

---

## Batch 3: Full build verification and docs

### Task 3: Full build verification and documentation update

**Files:**
- Modify: `docs/guides/contributor-guide.md` (document JpaReconciliationStateStore in persistence-jpa/ section)

**Interfaces:**
- Consumes: all production code from Tasks 1-2

- [ ] **Step 1: Run the full multi-module build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules compile and all tests pass. This
verifies that the new `@ApplicationScoped JpaReconciliationStateStore`
doesn't conflict with `@DefaultBean DefaultReconciliationStateStore` in
other modules' test suites.

- [ ] **Step 2: Update contributor-guide.md**

Add `JpaReconciliationStateStore` to the `persistence-jpa/` module
documentation in `docs/guides/contributor-guide.md`. Find the section
that documents `JpaFaultCountStore` and add a parallel entry for the
new store.

The entry should mention:
- `JpaReconciliationStateStore` — JPA-backed ReconciliationStateStore
- Classpath-activated, displaces DefaultReconciliationStateStore
- Stores serialized DesiredStateGraph as JSON per tenant
- Flyway migration V2 at `db/desiredstate/migration/`

- [ ] **Step 3: Verify the docs change compiles (if doc tooling exists)**

Run: `mvn --batch-mode install -DskipTests`
Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
git add docs/guides/contributor-guide.md
git commit -m "docs(#132): document JpaReconciliationStateStore in contributor guide

Refs #132"
```

---

## References

- [2026-09-05-durable-reconciliation-state-store-design.md] — design spec this plan implements
- [ReconciliationStateStore.java] (api/) — SPI contract
- [JpaFaultCountStore.java] (persistence-jpa/) — established JPA store pattern
- [FaultCountEntity.java] (persistence-jpa/) — entity pattern
- [V1__create_fault_count.sql] — Flyway migration pattern
- [JpaFaultCountStoreTest.java] (persistence-jpa/) — test pattern
- [TransitionPlanner.java:36-78] — orphan spec resolution usage
- [ReconciliationLoop.java:747-761] — store usage in plan()
- [GraphSerializer unit tests] — FQCN serialization verification
- [GE-20260703-b2073a] — orphan deprovision with UnknownSpec
- [GE-20260609-ef7dbe] — Flyway NOT NULL + DEFAULT H2 gotcha
- [GitHub #132] — focal issue
- [GitHub #138] — parent runtime-polish issue
