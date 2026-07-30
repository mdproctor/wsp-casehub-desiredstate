# Fault Infrastructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #94 — persistent FaultCountStore implementation
**Issue group:** #94, #95, #96

**Goal:** Add composite ReconciliationListener support, JPA-backed persistent
FaultCountStore, and migrate ProvisionEscalationFaultPolicy to FaultCountStore SPI.

**Architecture:** Three changes on one branch. #96 adds `GlobalReconciliationListener`
interface and wires CDI-discovered global listeners into `ReconciliationLoop`. #94
creates `persistence-jpa/` module with `JpaFaultCountStore` and `DefaultFaultCountStore`
CDI fallback. #95 migrates `ProvisionEscalationFaultPolicy` from private
`ConcurrentHashMap` to `FaultCountStore` SPI.

**Tech Stack:** Java 21, Quarkus (CDI/ARC, Hibernate ORM), JUnit 5, AssertJ, H2

## Global Constraints

- Pre-release platform — breaking API changes are free
- IntelliJ MCP mandatory for all .java edits — use `ide_*` tools exclusively
- CDI priority ladder protocol (`persistence-backend-cdi-priority`) governs backend activation
- Flyway migrations scoped to `db/desiredstate/migration/` (protocol: `flyway-repo-scoped-migration-path`)
- Portable SQL — H2 `MODE=PostgreSQL` + PostgreSQL
- `project_path` for all IntelliJ calls: `/Users/mdproctor/claude/casehub/desiredstate`

---

### Task 1: GlobalReconciliationListener interface and composite wiring (#96)

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/GlobalReconciliationListener.java`
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java`
- Create: `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopGlobalListenerTest.java`

**Interfaces:**
- Produces: `GlobalReconciliationListener` interface (consumed by Task 2 DefaultFaultCountStore and any future listener)
- Produces: `ReconciliationLoop` constructors accepting `List<GlobalReconciliationListener>` parameter

- [ ] **Step 1: Create GlobalReconciliationListener interface**

Use `ide_create_file`:

```java
package io.casehub.desiredstate.api;

public interface GlobalReconciliationListener {
    void onReconciliationCycleCompleted(String tenancyId, DesiredStateGraph desired, ActualState actual);
}
```

File: `api/src/main/java/io/casehub/desiredstate/api/GlobalReconciliationListener.java`

- [ ] **Step 2: Write failing tests for global listener integration**

Use `ide_create_file` to create `runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopGlobalListenerTest.java`:

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.ActualState;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.GlobalReconciliationListener;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.ReconciliationListener;
import io.casehub.desiredstate.testing.MockActualStateAdapter;
import io.casehub.desiredstate.testing.MockTransitionExecutor;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.List;
import java.util.Set;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static io.casehub.desiredstate.testing.TestTimeouts.AWAIT;
import static org.assertj.core.api.Assertions.assertThat;

class ReconciliationLoopGlobalListenerTest {

    private TransitionPlanner planner;
    private MockActualStateAdapter adapter;
    private DefaultActualStateAdapterRouter adapterRouter;
    private FaultPolicyEngine faultPolicyEngine;

    private record TestSpec() implements NodeSpec {}

    @BeforeEach
    void setUp() {
        planner = new TransitionPlanner();
        adapter = new MockActualStateAdapter();
        adapter.setHandledTypes(Set.of(NodeType.of("t")));
        adapterRouter = new DefaultActualStateAdapterRouter(List.of(adapter));
        faultPolicyEngine = new FaultPolicyEngine(List.of());
    }

    @Test
    void globalListeners_fireOnReconciliationCycle() throws Exception {
        DesiredNode node = new DesiredNode(NodeId.of("a"), NodeType.of("t"), new TestSpec(), HumanGating.NONE);
        DesiredStateGraph graph = ImmutableDesiredStateGraph.empty().withNode(node);
        adapter.setStatus(NodeId.of("a"), NodeStatus.ABSENT);

        List<String> fired = new CopyOnWriteArrayList<>();
        CountDownLatch latch = new CountDownLatch(2);

        GlobalReconciliationListener gl1 = (tid, d, a) -> { fired.add("gl1"); latch.countDown(); };
        GlobalReconciliationListener gl2 = (tid, d, a) -> { fired.add("gl2"); latch.countDown(); };

        ReconciliationLoop loop = new ReconciliationLoop(
            planner, new MockTransitionExecutor(), adapterRouter,
            faultPolicyEngine, () -> Multi.createFrom().nothing(),
            Duration.ofMillis(50), Duration.ofSeconds(60),
            null, null, List.of(gl1, gl2));
        loop.start("t1", graph);

        assertThat(latch.await(AWAIT.toSeconds(), TimeUnit.SECONDS)).isTrue();
        assertThat(fired).containsExactly("gl1", "gl2");
        loop.stop("t1");
    }

    @Test
    void globalListeners_fireAlongsidePerTenantListener() throws Exception {
        DesiredNode node = new DesiredNode(NodeId.of("a"), NodeType.of("t"), new TestSpec(), HumanGating.NONE);
        DesiredStateGraph graph = ImmutableDesiredStateGraph.empty().withNode(node);
        adapter.setStatus(NodeId.of("a"), NodeStatus.PRESENT);

        List<String> fired = new CopyOnWriteArrayList<>();
        CountDownLatch latch = new CountDownLatch(2);

        GlobalReconciliationListener global = (tid, d, a) -> { fired.add("global"); latch.countDown(); };
        ReconciliationListener perTenant = (tid, d, a) -> { fired.add("tenant"); latch.countDown(); };

        ReconciliationLoop loop = new ReconciliationLoop(
            planner, new MockTransitionExecutor(), adapterRouter,
            faultPolicyEngine, () -> Multi.createFrom().nothing(),
            Duration.ofMillis(50), Duration.ofSeconds(60),
            null, null, List.of(global));
        loop.start("t1", graph, perTenant);

        assertThat(latch.await(AWAIT.toSeconds(), TimeUnit.SECONDS)).isTrue();
        assertThat(fired).containsExactly("global", "tenant");
        loop.stop("t1");
    }

    @Test
    void globalListener_exceptionDoesNotBlockOthers() throws Exception {
        DesiredNode node = new DesiredNode(NodeId.of("a"), NodeType.of("t"), new TestSpec(), HumanGating.NONE);
        DesiredStateGraph graph = ImmutableDesiredStateGraph.empty().withNode(node);
        adapter.setStatus(NodeId.of("a"), NodeStatus.PRESENT);

        List<String> fired = new CopyOnWriteArrayList<>();
        CountDownLatch latch = new CountDownLatch(2);

        GlobalReconciliationListener failing = (tid, d, a) -> { fired.add("failing"); latch.countDown(); throw new RuntimeException("boom"); };
        GlobalReconciliationListener surviving = (tid, d, a) -> { fired.add("surviving"); latch.countDown(); };

        ReconciliationLoop loop = new ReconciliationLoop(
            planner, new MockTransitionExecutor(), adapterRouter,
            faultPolicyEngine, () -> Multi.createFrom().nothing(),
            Duration.ofMillis(50), Duration.ofSeconds(60),
            null, null, List.of(failing, surviving));
        loop.start("t1", graph);

        assertThat(latch.await(AWAIT.toSeconds(), TimeUnit.SECONDS)).isTrue();
        assertThat(fired).containsExactly("failing", "surviving");
        loop.stop("t1");
    }

    @Test
    void globalListeners_fireOnEmptyPlanCycles() throws Exception {
        DesiredNode node = new DesiredNode(NodeId.of("a"), NodeType.of("t"), new TestSpec(), HumanGating.NONE);
        DesiredStateGraph graph = ImmutableDesiredStateGraph.empty().withNode(node);
        adapter.setStatus(NodeId.of("a"), NodeStatus.PRESENT);

        CountDownLatch latch = new CountDownLatch(1);
        GlobalReconciliationListener global = (tid, d, a) -> latch.countDown();

        ReconciliationLoop loop = new ReconciliationLoop(
            planner, new MockTransitionExecutor(), adapterRouter,
            faultPolicyEngine, () -> Multi.createFrom().nothing(),
            Duration.ofMillis(50), Duration.ofSeconds(60),
            null, null, List.of(global));
        loop.start("t1", graph);

        assertThat(latch.await(AWAIT.toSeconds(), TimeUnit.SECONDS)).isTrue();
        loop.stop("t1");
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=ReconciliationLoopGlobalListenerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: COMPILATION FAILURE — constructors don't accept `List<GlobalReconciliationListener>` yet

- [ ] **Step 4: Add globalListeners field and update constructors**

Use `ide_insert_member` to add field after `cbrTracker` (line 107):
```java
private final List<GlobalReconciliationListener> globalListeners;
```

Add import: `import io.casehub.desiredstate.api.GlobalReconciliationListener;`
Add import: `import jakarta.enterprise.inject.Instance;`

Use `ide_edit_member` to update the private master constructor (line 209) — add
`List<GlobalReconciliationListener> globalListeners` as the last parameter. Add
`this.globalListeners = globalListeners != null ? List.copyOf(globalListeners) : List.of();`
to the body.

Update the CDI constructor (line 116) — add `Instance<GlobalReconciliationListener> globalListeners`
as the last parameter. Pass `globalListeners.stream().toList()` to the master constructor.

Update each test constructor to pass `List.of()` as the `globalListeners` argument
to the master constructor. There are 6 public constructors that delegate to the
private master — each needs `List.of()` appended to its delegation call.

- [ ] **Step 5: Update fireListener to chain global listeners**

Use `ide_replace_member` on the `fireListener` method (line 550 in TenantLoop):

```java
private void fireListener(DesiredStateGraph desired, ActualState actual) {
    for (GlobalReconciliationListener gl : globalListeners) {
        try {
            gl.onReconciliationCycleCompleted(tenancyId, desired, actual);
        } catch (Exception e) {
            LOG.log(Level.WARNING,
                "Global reconciliation listener failed for tenant " + tenancyId, e);
        }
    }
    ReconciliationListener l = listener;
    if (l != null) {
        try {
            l.onReconciliationCycleCompleted(tenancyId, desired, actual);
        } catch (Exception e) {
            LOG.log(Level.WARNING,
                "Reconciliation listener failed for tenant " + tenancyId, e);
        }
    }
}
```

- [ ] **Step 6: Verify compilation**

Run: `ide_diagnostics` on `ReconciliationLoop.java` and `GlobalReconciliationListener.java`
Run: `ide_build_project`

- [ ] **Step 7: Run tests**

Run: `mvn --batch-mode test -pl runtime -Dtest=ReconciliationLoopGlobalListenerTest`
Expected: ALL PASS

Also run existing lifecycle tests to confirm no regression:
Run: `mvn --batch-mode test -pl runtime -Dtest=ReconciliationLoopLifecycleTest`
Expected: ALL PASS

- [ ] **Step 8: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules pass

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/desiredstate add api/src/main/java/io/casehub/desiredstate/api/GlobalReconciliationListener.java runtime/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java runtime/src/test/java/io/casehub/desiredstate/runtime/ReconciliationLoopGlobalListenerTest.java
```

Message: `feat(#96): composite ReconciliationListener — GlobalReconciliationListener SPI with CDI discovery`

---

### Task 2: Persistent FaultCountStore — JPA module and CDI default (#94)

**Files:**
- Create: `persistence-jpa/pom.xml`
- Create: `persistence-jpa/src/main/java/io/casehub/desiredstate/persistence/jpa/FaultCountEntity.java`
- Create: `persistence-jpa/src/main/java/io/casehub/desiredstate/persistence/jpa/JpaFaultCountStore.java`
- Create: `persistence-jpa/src/main/resources/db/desiredstate/migration/V1__create_fault_count.sql`
- Create: `persistence-jpa/src/test/java/io/casehub/desiredstate/persistence/jpa/JpaFaultCountStoreTest.java`
- Create: `persistence-jpa/src/test/resources/application.properties`
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/DefaultFaultCountStore.java`
- Modify: `pom.xml` (parent — add module + dependency management)

**Interfaces:**
- Consumes: `FaultCountStore` SPI from `api/`
- Produces: `JpaFaultCountStore` (`@ApplicationScoped`, beats `@DefaultBean`)
- Produces: `DefaultFaultCountStore` (`@DefaultBean`, CDI fallback)

- [ ] **Step 1: Create DefaultFaultCountStore in runtime/**

Use `ide_create_file` for `runtime/src/main/java/io/casehub/desiredstate/runtime/DefaultFaultCountStore.java`:

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.InMemoryFaultCountStore;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class DefaultFaultCountStore extends InMemoryFaultCountStore {
}
```

- [ ] **Step 2: Create persistence-jpa module scaffold**

Create directory structure:
```bash
mkdir -p /Users/mdproctor/claude/casehub/desiredstate/persistence-jpa/src/main/java/io/casehub/desiredstate/persistence/jpa
mkdir -p /Users/mdproctor/claude/casehub/desiredstate/persistence-jpa/src/main/resources/db/desiredstate/migration
mkdir -p /Users/mdproctor/claude/casehub/desiredstate/persistence-jpa/src/test/java/io/casehub/desiredstate/persistence/jpa
mkdir -p /Users/mdproctor/claude/casehub/desiredstate/persistence-jpa/src/test/resources
```

Write `persistence-jpa/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-desiredstate-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-desiredstate-persistence-jpa</artifactId>

    <name>CaseHub Desired State :: Persistence JPA</name>
    <description>JPA-backed FaultCountStore — durable fault counts across restarts.
        Tier 2 in the CDI priority ladder. Portable SQL (H2 + PostgreSQL).</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-hibernate-orm</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>

        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-jdbc-h2</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>${jandex-maven-plugin.version}</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 3: Create Flyway migration**

Write `persistence-jpa/src/main/resources/db/desiredstate/migration/V1__create_fault_count.sql`:

```sql
CREATE TABLE ds_fault_count (
    namespace    VARCHAR(255) NOT NULL,
    tenancy_id   VARCHAR(255) NOT NULL,
    node_id      VARCHAR(255) NOT NULL,
    count        INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (namespace, tenancy_id, node_id)
);
```

- [ ] **Step 4: Create FaultCountEntity**

Use `ide_create_file` for `persistence-jpa/src/main/java/io/casehub/desiredstate/persistence/jpa/FaultCountEntity.java`:

```java
package io.casehub.desiredstate.persistence.jpa;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.IdClass;
import jakarta.persistence.Table;
import java.io.Serializable;

@Entity
@Table(name = "ds_fault_count")
@IdClass(FaultCountEntity.Key.class)
public class FaultCountEntity {

    @Id
    @Column(name = "namespace")
    String namespace;

    @Id
    @Column(name = "tenancy_id")
    String tenancyId;

    @Id
    @Column(name = "node_id")
    String nodeId;

    @Column(name = "count", nullable = false)
    int count;

    public record Key(String namespace, String tenancyId, String nodeId) implements Serializable {}
}
```

- [ ] **Step 5: Create JpaFaultCountStore**

Use `ide_create_file` for `persistence-jpa/src/main/java/io/casehub/desiredstate/persistence/jpa/JpaFaultCountStore.java`:

```java
package io.casehub.desiredstate.persistence.jpa;

import io.casehub.desiredstate.api.FaultCountStore;
import io.casehub.desiredstate.api.NodeId;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import java.util.Set;
import java.util.stream.Collectors;

@ApplicationScoped
public class JpaFaultCountStore implements FaultCountStore {

    @Inject
    EntityManager em;

    @Override
    @Transactional
    public int incrementAndGet(String namespace, String tenancyId, NodeId nodeId) {
        em.createNativeQuery(
                "INSERT INTO ds_fault_count (namespace, tenancy_id, node_id, count) " +
                "VALUES (:ns, :tid, :nid, 1) " +
                "ON CONFLICT (namespace, tenancy_id, node_id) " +
                "DO UPDATE SET count = ds_fault_count.count + 1")
            .setParameter("ns", namespace)
            .setParameter("tid", tenancyId)
            .setParameter("nid", nodeId.value())
            .executeUpdate();

        return ((Number) em.createNativeQuery(
                "SELECT count FROM ds_fault_count WHERE namespace = :ns AND tenancy_id = :tid AND node_id = :nid")
            .setParameter("ns", namespace)
            .setParameter("tid", tenancyId)
            .setParameter("nid", nodeId.value())
            .getSingleResult()).intValue();
    }

    @Override
    public int getCount(String namespace, String tenancyId, NodeId nodeId) {
        FaultCountEntity entity = em.find(FaultCountEntity.class,
            new FaultCountEntity.Key(namespace, tenancyId, nodeId.value()));
        return entity != null ? entity.count : 0;
    }

    @Override
    @Transactional
    public void reset(String namespace, String tenancyId, NodeId nodeId) {
        em.createNativeQuery(
                "INSERT INTO ds_fault_count (namespace, tenancy_id, node_id, count) " +
                "VALUES (:ns, :tid, :nid, 0) " +
                "ON CONFLICT (namespace, tenancy_id, node_id) " +
                "DO UPDATE SET count = 0")
            .setParameter("ns", namespace)
            .setParameter("tid", tenancyId)
            .setParameter("nid", nodeId.value())
            .executeUpdate();
    }

    @Override
    @Transactional
    public void remove(String namespace, String tenancyId, NodeId nodeId) {
        FaultCountEntity entity = em.find(FaultCountEntity.class,
            new FaultCountEntity.Key(namespace, tenancyId, nodeId.value()));
        if (entity != null) {
            em.remove(entity);
        }
    }

    @Override
    @Transactional
    public void evict(String namespace, String tenancyId, Set<NodeId> retainedNodes) {
        if (retainedNodes.isEmpty()) {
            em.createQuery("DELETE FROM FaultCountEntity e WHERE e.namespace = :ns AND e.tenancyId = :tid")
                .setParameter("ns", namespace)
                .setParameter("tid", tenancyId)
                .executeUpdate();
        } else {
            Set<String> retained = retainedNodes.stream()
                .map(NodeId::value)
                .collect(Collectors.toSet());
            em.createQuery("DELETE FROM FaultCountEntity e WHERE e.namespace = :ns AND e.tenancyId = :tid AND e.nodeId NOT IN :retained")
                .setParameter("ns", namespace)
                .setParameter("tid", tenancyId)
                .setParameter("retained", retained)
                .executeUpdate();
        }
    }
}
```

- [ ] **Step 6: Create test application.properties**

Write `persistence-jpa/src/test/resources/application.properties`:

```properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:test;MODE=PostgreSQL
quarkus.hibernate-orm.database.generation=none
quarkus.flyway.locations=classpath:db/desiredstate/migration
quarkus.flyway.migrate-at-start=true
```

- [ ] **Step 7: Write JpaFaultCountStore tests**

Use `ide_create_file` for `persistence-jpa/src/test/java/io/casehub/desiredstate/persistence/jpa/JpaFaultCountStoreTest.java`:

```java
package io.casehub.desiredstate.persistence.jpa;

import io.casehub.desiredstate.api.NodeId;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class JpaFaultCountStoreTest {

    @Inject
    JpaFaultCountStore store;

    @BeforeEach
    @Transactional
    void cleanTable() {
        store.evict("ns", "t1", Set.of());
        store.evict("ns", "t2", Set.of());
        store.evict("ns-a", "t1", Set.of());
        store.evict("ns-b", "t1", Set.of());
        store.evict("policy-a", "t1", Set.of());
        store.evict("policy-b", "t1", Set.of());
        store.evict("pipeline-escalation", "t1", Set.of());
    }

    @Test
    void incrementAndGet_returnsSequentialCounts() {
        assertThat(store.incrementAndGet("ns", "t1", NodeId.of("n1"))).isEqualTo(1);
        assertThat(store.incrementAndGet("ns", "t1", NodeId.of("n1"))).isEqualTo(2);
        assertThat(store.incrementAndGet("ns", "t1", NodeId.of("n1"))).isEqualTo(3);
    }

    @Test
    void getCount_returnsZeroForAbsent() {
        assertThat(store.getCount("ns", "t1", NodeId.of("n1"))).isEqualTo(0);
    }

    @Test
    void reset_setsCountToZero() {
        store.incrementAndGet("ns", "t1", NodeId.of("n1"));
        store.incrementAndGet("ns", "t1", NodeId.of("n1"));
        store.reset("ns", "t1", NodeId.of("n1"));
        assertThat(store.getCount("ns", "t1", NodeId.of("n1"))).isEqualTo(0);
        assertThat(store.incrementAndGet("ns", "t1", NodeId.of("n1"))).isEqualTo(1);
    }

    @Test
    void reset_onNonExistentKeyCreatesZeroCountRow() {
        store.reset("ns", "t1", NodeId.of("never-seen"));
        assertThat(store.getCount("ns", "t1", NodeId.of("never-seen"))).isEqualTo(0);
        assertThat(store.incrementAndGet("ns", "t1", NodeId.of("never-seen"))).isEqualTo(1);
    }

    @Test
    void remove_deletesEntry() {
        store.incrementAndGet("ns", "t1", NodeId.of("n1"));
        store.incrementAndGet("ns", "t1", NodeId.of("n1"));
        store.remove("ns", "t1", NodeId.of("n1"));
        assertThat(store.getCount("ns", "t1", NodeId.of("n1"))).isEqualTo(0);
        assertThat(store.incrementAndGet("ns", "t1", NodeId.of("n1"))).isEqualTo(1);
    }

    @Test
    void evict_removesEntriesNotInRetainedSet() {
        store.incrementAndGet("ns", "t1", NodeId.of("a"));
        store.incrementAndGet("ns", "t1", NodeId.of("b"));
        store.incrementAndGet("ns", "t1", NodeId.of("c"));

        store.evict("ns", "t1", Set.of(NodeId.of("a"), NodeId.of("c")));

        assertThat(store.getCount("ns", "t1", NodeId.of("a"))).isEqualTo(1);
        assertThat(store.getCount("ns", "t1", NodeId.of("b"))).isEqualTo(0);
        assertThat(store.getCount("ns", "t1", NodeId.of("c"))).isEqualTo(1);
    }

    @Test
    void evict_withEmptyRetainedSet_deletesAll() {
        store.incrementAndGet("ns", "t1", NodeId.of("a"));
        store.incrementAndGet("ns", "t1", NodeId.of("b"));

        store.evict("ns", "t1", Set.of());

        assertThat(store.getCount("ns", "t1", NodeId.of("a"))).isEqualTo(0);
        assertThat(store.getCount("ns", "t1", NodeId.of("b"))).isEqualTo(0);
    }

    @Test
    void tenantIsolation() {
        store.incrementAndGet("ns", "t1", NodeId.of("n1"));
        store.incrementAndGet("ns", "t1", NodeId.of("n1"));
        store.incrementAndGet("ns", "t2", NodeId.of("n1"));

        assertThat(store.getCount("ns", "t1", NodeId.of("n1"))).isEqualTo(2);
        assertThat(store.getCount("ns", "t2", NodeId.of("n1"))).isEqualTo(1);
    }

    @Test
    void namespaceIsolation() {
        store.incrementAndGet("policy-a", "t1", NodeId.of("n1"));
        store.incrementAndGet("policy-a", "t1", NodeId.of("n1"));
        store.incrementAndGet("policy-b", "t1", NodeId.of("n1"));

        assertThat(store.getCount("policy-a", "t1", NodeId.of("n1"))).isEqualTo(2);
        assertThat(store.getCount("policy-b", "t1", NodeId.of("n1"))).isEqualTo(1);
    }

    @Test
    void evict_scopedToNamespaceAndTenancy() {
        store.incrementAndGet("ns-a", "t1", NodeId.of("n1"));
        store.incrementAndGet("ns-b", "t1", NodeId.of("n1"));
        store.incrementAndGet("ns-a", "t2", NodeId.of("n1"));

        store.evict("ns-a", "t1", Set.of());

        assertThat(store.getCount("ns-a", "t1", NodeId.of("n1"))).isEqualTo(0);
        assertThat(store.getCount("ns-b", "t1", NodeId.of("n1"))).isEqualTo(1);
        assertThat(store.getCount("ns-a", "t2", NodeId.of("n1"))).isEqualTo(1);
    }
}
```

- [ ] **Step 8: Update parent POM**

Add to `<modules>` list (after `ras-adapter`):
```xml
<module>persistence-jpa</module>
```

Add to `<dependencyManagement>` (after `casehub-desiredstate-ras`):
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-desiredstate-persistence-jpa</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 9: Sync IntelliJ and verify**

Run: `ide_sync_files` to pick up new module
Run: `ide_reload_project` to reload Maven
Run: `ide_build_project`

- [ ] **Step 10: Run persistence-jpa tests**

Run: `mvn --batch-mode test -pl persistence-jpa`
Expected: ALL PASS

- [ ] **Step 11: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 12: Commit**

Stage: `persistence-jpa/`, `runtime/src/main/java/.../DefaultFaultCountStore.java`, `pom.xml`
Message: `feat(#94): JPA-backed FaultCountStore with CDI priority ladder and Flyway migration`

---

### Task 3: Migrate ProvisionEscalationFaultPolicy to FaultCountStore (#95)

**Files:**
- Modify: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/ProvisionEscalationFaultPolicy.java`
- Create: `examples/pipeline/src/test/java/io/casehub/desiredstate/example/pipeline/ProvisionEscalationFaultPolicyTest.java`

**Interfaces:**
- Consumes: `FaultCountStore` SPI from `api/`, `InMemoryFaultCountStore` from `api/`

- [ ] **Step 1: Write failing tests for FaultCountStore migration**

Use `ide_create_file` for `examples/pipeline/src/test/java/io/casehub/desiredstate/example/pipeline/ProvisionEscalationFaultPolicyTest.java`:

```java
package io.casehub.desiredstate.example.pipeline;

import io.casehub.desiredstate.api.ActualState;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.FaultCountStore;
import io.casehub.desiredstate.api.FaultEvent;
import io.casehub.desiredstate.api.FaultType;
import io.casehub.desiredstate.api.GraphMutation;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.InMemoryFaultCountStore;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.casehub.desiredstate.runtime.ImmutableDesiredStateGraph;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class ProvisionEscalationFaultPolicyTest {

    private PipelineWorld world;
    private FaultCountStore store;
    private ProvisionEscalationFaultPolicy policy;

    @BeforeEach
    void setUp() {
        world = new PipelineWorld();
        store = new InMemoryFaultCountStore();
        policy = new ProvisionEscalationFaultPolicy(world, store);
    }

    @Test
    void tenantIsolation_independentFaultCounts() {
        DesiredNode node = new DesiredNode(NodeId.of("ingest"), PipelineNodeTypes.INGESTION,
            new IngestionSpec("data", 100, "json"), HumanGating.NONE);
        DesiredStateGraph graph = new DefaultDesiredStateGraphFactory().of(List.of(node), List.of());
        FaultEvent fault = new FaultEvent(NodeId.of("ingest"), FaultType.PROVISION_FAILED, "down");
        ActualState actual = new ActualState(Map.of());

        for (int i = 0; i < 3; i++) {
            policy.onFault("tenant-a", fault, graph, actual);
        }
        policy.onFault("tenant-b", fault, graph, actual);

        assertThat(store.getCount("pipeline-escalation", "tenant-a", NodeId.of("ingest"))).isEqualTo(3);
        assertThat(store.getCount("pipeline-escalation", "tenant-b", NodeId.of("ingest"))).isEqualTo(1);
    }

    @Test
    void namespaceIsolation_fromOtherPolicies() {
        FaultCountStore shared = new InMemoryFaultCountStore();
        shared.incrementAndGet("other-policy", "t1", NodeId.of("ingest"));
        shared.incrementAndGet("other-policy", "t1", NodeId.of("ingest"));

        ProvisionEscalationFaultPolicy p = new ProvisionEscalationFaultPolicy(world, shared);
        DesiredNode node = new DesiredNode(NodeId.of("ingest"), PipelineNodeTypes.INGESTION,
            new IngestionSpec("data", 100, "json"), HumanGating.NONE);
        DesiredStateGraph graph = new DefaultDesiredStateGraphFactory().of(List.of(node), List.of());
        FaultEvent fault = new FaultEvent(NodeId.of("ingest"), FaultType.PROVISION_FAILED, "err");

        p.onFault("t1", fault, graph, new ActualState(Map.of()));

        assertThat(shared.getCount("pipeline-escalation", "t1", NodeId.of("ingest"))).isEqualTo(1);
        assertThat(shared.getCount("other-policy", "t1", NodeId.of("ingest"))).isEqualTo(2);
    }

    @Test
    void lazyEviction_removedNodeCleansStore() {
        DesiredNode node = new DesiredNode(NodeId.of("ingest"), PipelineNodeTypes.INGESTION,
            new IngestionSpec("data", 100, "json"), HumanGating.NONE);
        DesiredStateGraph graph = new DefaultDesiredStateGraphFactory().of(List.of(node), List.of());
        FaultEvent fault = new FaultEvent(NodeId.of("ingest"), FaultType.PROVISION_FAILED, "err");

        policy.onFault("t1", fault, graph, new ActualState(Map.of()));
        policy.onFault("t1", fault, graph, new ActualState(Map.of()));
        assertThat(store.getCount("pipeline-escalation", "t1", NodeId.of("ingest"))).isEqualTo(2);

        DesiredStateGraph emptyGraph = ImmutableDesiredStateGraph.empty();
        policy.onFault("t1", fault, emptyGraph, new ActualState(Map.of()));
        assertThat(store.getCount("pipeline-escalation", "t1", NodeId.of("ingest"))).isEqualTo(0);
    }

    @Test
    void defaultConstructor_usesInMemoryStore() {
        ProvisionEscalationFaultPolicy defaultPolicy = new ProvisionEscalationFaultPolicy(world);
        DesiredNode node = new DesiredNode(NodeId.of("n"), PipelineNodeTypes.INGESTION,
            new IngestionSpec("d", 10, "json"), HumanGating.NONE);
        DesiredStateGraph graph = new DefaultDesiredStateGraphFactory().of(List.of(node), List.of());
        FaultEvent fault = new FaultEvent(NodeId.of("n"), FaultType.PROVISION_FAILED, "err");

        for (int i = 0; i < 3; i++) {
            assertThat(defaultPolicy.onFault("t1", fault, graph, new ActualState(Map.of()))).isEmpty();
        }
        assertThat(defaultPolicy.onFault("t1", fault, graph, new ActualState(Map.of()))).hasSize(1);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl examples/pipeline -Dtest=ProvisionEscalationFaultPolicyTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: COMPILATION FAILURE — `ProvisionEscalationFaultPolicy(PipelineWorld, FaultCountStore)` constructor doesn't exist

- [ ] **Step 3: Migrate ProvisionEscalationFaultPolicy**

Use `ide_edit_member` to replace the class body of `ProvisionEscalationFaultPolicy`:

Replace field `faultCounts` with `store` field and `NAMESPACE` constant. Add second constructor
accepting `FaultCountStore`. Update `onFault` to:
- Use `store.incrementAndGet(NAMESPACE, tenancyId, event.node())` instead of `faultCounts.merge`
- Add lazy eviction: if faulted node not in graph, call `store.remove(NAMESPACE, tenancyId, event.node())` and return
- Pass `tenancyId` through all counting operations

The full updated class:

```java
public class ProvisionEscalationFaultPolicy implements FaultPolicy {

    private static final String NAMESPACE = "pipeline-escalation";

    private final PipelineWorld world;
    private final FaultCountStore store;

    public ProvisionEscalationFaultPolicy(PipelineWorld world) {
        this(world, new InMemoryFaultCountStore());
    }

    public ProvisionEscalationFaultPolicy(PipelineWorld world, FaultCountStore store) {
        this.world = world;
        this.store = store;
    }

    public List<GraphMutation> onFault(String tenancyId, FaultEvent event, DesiredStateGraph current, ActualState actual) {
        if (event.type() != FaultType.PROVISION_FAILED) {
            return List.of();
        }

        DesiredNode faultedNode = current.nodes().get(event.node());

        if (faultedNode == null) {
            store.remove(NAMESPACE, tenancyId, event.node());
            return List.of();
        }

        if (PipelineNodeTypes.AI_REVIEW.equals(faultedNode.type())
                || PipelineNodeTypes.HUMAN_REVIEW.equals(faultedNode.type())) {
            return List.of();
        }

        int count = store.incrementAndGet(NAMESPACE, tenancyId, event.node());

        if (count <= 3) {
            return List.of();
        }

        NodeId aiReviewId    = NodeId.of("ai-review-" + event.node().value());
        NodeId humanReviewId = NodeId.of("human-review-" + event.node().value());

        if (current.nodes().containsKey(humanReviewId)) {
            return List.of();
        }
        PipelineWorld.ReviewEntry humanReview = world.review(humanReviewId);
        if (humanReview != null) {
            return List.of();
        }

        if (!current.nodes().containsKey(aiReviewId)) {
            DesiredNode reviewNode = new DesiredNode(aiReviewId, PipelineNodeTypes.AI_REVIEW,
                                                     new AiReviewSpec(event.node(), event.detail()), HumanGating.NONE);
            return List.of(new GraphMutation.AddNode(reviewNode));
        }

        PipelineWorld.ReviewEntry review = world.review(aiReviewId);
        if (review == null || review.state() == PipelineWorld.ReviewState.PENDING) {
            return List.of();
        }
        if (review.state() == PipelineWorld.ReviewState.RESOLVED) {
            return List.of();
        }

        DesiredNode humanNode = new DesiredNode(humanReviewId, PipelineNodeTypes.HUMAN_REVIEW,
                                                new HumanReviewSpec(event.node(), event.detail(), "AI review could not resolve"), HumanGating.ALL);
        return List.of(new GraphMutation.AddNode(humanNode));
    }
}
```

Add imports: `import io.casehub.desiredstate.api.FaultCountStore;` and `import io.casehub.desiredstate.api.InMemoryFaultCountStore;`
Remove: `import java.util.Map;`, `import java.util.concurrent.ConcurrentHashMap;`

- [ ] **Step 4: Run new tests**

Run: `mvn --batch-mode test -pl examples/pipeline -Dtest=ProvisionEscalationFaultPolicyTest`
Expected: ALL PASS

- [ ] **Step 5: Run existing pipeline tests**

Run: `mvn --batch-mode test -pl examples/pipeline`
Expected: ALL PASS (existing tests use single-arg constructor, unchanged behavior)

- [ ] **Step 6: Full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

Stage: `examples/pipeline/src/main/java/.../ProvisionEscalationFaultPolicy.java`, `examples/pipeline/src/test/java/.../ProvisionEscalationFaultPolicyTest.java`
Message: `feat(#95): migrate ProvisionEscalationFaultPolicy to FaultCountStore SPI`

---

### Task 4: CLAUDE.md updates and deferred issues

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: All work from Tasks 1-3

- [ ] **Step 1: Update CLAUDE.md module table**

Add `persistence-jpa/` row to the Module Structure table.

- [ ] **Step 2: Update CLAUDE.md core types**

Add `GlobalReconciliationListener`, `DefaultFaultCountStore`, `JpaFaultCountStore`,
`FaultCountEntity` to the appropriate tables.

- [ ] **Step 3: File deferred issues**

File three GitHub issues per the spec's Deferred Issues table:
1. `FaultCountEvictionListener` design — evict() consumer via GlobalReconciliationListener
2. `ReconciliationLoop` constructor telescope refactoring
3. Protocol update — `persistence-backend-cdi-priority` clarify `@DefaultBean` for functional fallbacks

- [ ] **Step 4: Full build verification**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

Stage: `CLAUDE.md`
Message: `docs(#94): update CLAUDE.md — persistence-jpa module, GlobalReconciliationListener, deferred issues`
