# Fault Infrastructure: Persistent Store, Composite Listener, Escalation Migration

> Covers: #94, #95, #96 on branch `issue-94-persistent-faultcount-store`

## Context

#85 introduced the `FaultCountStore` SPI and `InMemoryFaultCountStore` in `api/`.
`ThresholdFaultPolicy` was refactored to delegate counting through this SPI.
Three follow-on issues surfaced during design review:

- **#94** — Persistent `FaultCountStore` (counts survive restarts)
- **#95** — Migrate `ProvisionEscalationFaultPolicy` to `FaultCountStore`
- **#96** — Composite `ReconciliationListener` (unblocks bulk eviction pattern)

## Implementation Order

#96 → #94 → #95

Rationale: #96 enables the bulk eviction integration pattern that a persistent
store (#94) benefits from. #95 is a clean example migration independent of both.

---

## #96 — Composite ReconciliationListener

### Problem

`ReconciliationLoop.TenantLoop` has a single `private volatile ReconciliationListener listener`
field. `LifecycleManager` occupies this slot for phase transitions. No other listener
(bulk eviction, monitoring, metrics) can register.

### Design

Two categories of listener, distinguished at the type level:

| Category | Interface | Scope | Registration | Example |
|----------|-----------|-------|-------------|---------|
| Global | `GlobalReconciliationListener` | Application-scoped | CDI-discovered at construction | Bulk eviction, metrics |
| Per-tenant | `ReconciliationListener` | Tenant lifecycle | `start(tenancyId, desired, listener)` / `setListener()` | LifecycleManager |

**`GlobalReconciliationListener`** (api/) — new interface with the same method signature
as `ReconciliationListener`. The type distinction prevents a CDI-discovered
`@ApplicationScoped` bean from being accidentally passed to `setListener()` as a
per-tenant listener, and follows the established pattern where separate roles use
separate types (`EventSource` / `MergedEventSource`, `FaultPolicy` / `FaultPolicyEngine`,
`NodeProvisioner` / `NodeProvisionerRouter`).

ReconciliationLoop accepts `List<GlobalReconciliationListener>` at construction time —
discovered via CDI `Instance<GlobalReconciliationListener>`. The per-tenant `listener`
field remains as `ReconciliationListener`. `fireListener()` iterates global listeners,
then calls the per-tenant listener.

### Changes

**`GlobalReconciliationListener`** (api/): New interface — same method signature as
`ReconciliationListener`. Not `@FunctionalInterface` (CDI beans, not lambdas).

```java
public interface GlobalReconciliationListener {
    void onReconciliationCycleCompleted(String tenancyId, DesiredStateGraph desired, ActualState actual);
}
```

**`ReconciliationListener`** (api/): No change. Stays `@FunctionalInterface` for
per-tenant use.

**`ReconciliationLoop`** (runtime/):
- Add `private final List<GlobalReconciliationListener> globalListeners` field
- Thread `List<GlobalReconciliationListener>` through all constructors (CDI constructor
  takes `Instance<GlobalReconciliationListener>`, test constructors default to `List.of()`)
- `fireListener()` iterates `globalListeners` then calls per-tenant `listener`,
  each wrapped in try/catch (existing error isolation pattern)
- Constructor telescope: this adds one parameter to each of the 7 existing constructors.
  The telescope is pre-existing tech debt — refactoring deferred (see Deferred Issues).

**Garden GE-20260707 compliance:** `fireListener()` is already called before the
early return on empty plan (line 578 in reconcile()). Global listeners inherit
this placement — they fire unconditionally on every cycle.

### Error Isolation

Each global listener is wrapped in its own try/catch — one listener failure
does not prevent others from firing. Same pattern as the existing per-tenant
listener (lines 550-560).

---

## #94 — Persistent FaultCountStore

### Problem

`InMemoryFaultCountStore` resets on restart. Deployments needing durable fault
thresholds (e.g., long-running reconciliation with rare faults) lose counts.

### Design

Apply the platform CDI priority ladder (protocol `persistence-backend-cdi-priority`):

| Tier | Class | Module | Annotation | Behaviour |
|------|-------|--------|------------|-----------|
| 1 | `DefaultFaultCountStore` | runtime/ | `@DefaultBean @ApplicationScoped` | Delegates to `InMemoryFaultCountStore`. Yields to any real backend. |
| 2 | `JpaFaultCountStore` | persistence-jpa/ | `@ApplicationScoped` | JPA entity, Flyway migration. Beats Tier 1 on classpath. |

`InMemoryFaultCountStore` stays in `api/` as a plain class — used by
`ThresholdFaultPolicy` builder default (non-CDI path unchanged).

MongoDB backend deferred to follow-up issue.

### Module: `persistence-jpa/`

**Artifact:** `casehub-desiredstate-persistence-jpa`
**Package:** `io.casehub.desiredstate.persistence.jpa`

**Dependencies:**
- `casehub-desiredstate-api` (FaultCountStore SPI)
- `quarkus-hibernate-orm` (JPA)
- `quarkus-jdbc-h2` (test)

**Entity: `FaultCountEntity`**

```java
@Entity
@Table(name = "ds_fault_count")
@IdClass(FaultCountEntity.Key.class)
public class FaultCountEntity {
    @Id String namespace;
    @Id @Column(name = "tenancy_id") String tenancyId;
    @Id @Column(name = "node_id") String nodeId;
    int count;
}
```

Table prefix `ds_` scopes to desiredstate — avoids collision with other modules
sharing the default datasource.

**Flyway migration:** `db/desiredstate/migration/V1__create_fault_count.sql`

```sql
CREATE TABLE ds_fault_count (
    namespace    VARCHAR(255) NOT NULL,
    tenancy_id   VARCHAR(255) NOT NULL,
    node_id      VARCHAR(255) NOT NULL,
    count        INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (namespace, tenancy_id, node_id)
);
```

Portable SQL — works on H2 (MODE=PostgreSQL) and Postgres.

**Consumer Flyway configuration:** Applications using `persistence-jpa/` must add
the desiredstate migration path to their Flyway locations:

```properties
quarkus.flyway.locations=classpath:db/desiredstate/migration
```

Per `flyway-repo-scoped-migration-path` protocol — Quarkus has no runtime
auto-registration mechanism.

**NodeId conversion:** The entity stores `node_id` as `String`. `JpaFaultCountStore`
converts between `NodeId` and `String` at the repository boundary — `nodeId.value()`
on write, `NodeId.of(column)` on read. No `AttributeConverter`; the entity is a
persistence concern, not a domain type.

**`JpaFaultCountStore` operations:**

| SPI method | Implementation |
|-----------|---------------|
| `incrementAndGet` | Native SQL upsert: `INSERT INTO ds_fault_count (...) VALUES (?, ?, ?, 1) ON CONFLICT (namespace, tenancy_id, node_id) DO UPDATE SET count = ds_fault_count.count + 1 RETURNING count`. Atomic — no read-modify-write race. `@Transactional` |
| `getCount` | Find by key → return count or 0 |
| `reset` | Native SQL upsert: `INSERT INTO ds_fault_count (...) VALUES (?, ?, ?, 0) ON CONFLICT (namespace, tenancy_id, node_id) DO UPDATE SET count = 0`. Creates zero-count row if absent — matches `InMemoryFaultCountStore.reset()` semantics. `@Transactional` |
| `remove` | Find by key → remove |
| `evict` | Branches on `retainedNodes`: when empty, `DELETE WHERE namespace=? AND tenancy_id=?`; when non-empty, `DELETE WHERE namespace=? AND tenancy_id=? AND node_id NOT IN (?)`. JPQL `NOT IN` with empty collection is undefined in JPA — the branch avoids this. `@Transactional` |

All mutating methods are `@Transactional`. `incrementAndGet` and `reset` use native
SQL upserts for atomicity — the `INSERT ON CONFLICT` pattern is portable across
PostgreSQL and H2 (`MODE=PostgreSQL`). No `@Version` field is needed on the entity
since the database handles atomicity at the SQL level.

### Module: runtime/ addition

**`DefaultFaultCountStore`** — single class:

```java
@DefaultBean
@ApplicationScoped
public class DefaultFaultCountStore extends InMemoryFaultCountStore {}
```

Yields to `JpaFaultCountStore` when `persistence-jpa` is on classpath.
Provides a working CDI bean for deployments that don't need durability.

### Parent POM changes

- Add `persistence-jpa` to `<modules>` list
- Add `casehub-desiredstate-persistence-jpa` to `<dependencyManagement>`

---

## #95 — Migrate ProvisionEscalationFaultPolicy to FaultCountStore

### Problem

`ProvisionEscalationFaultPolicy` (examples/pipeline/) uses a private
`ConcurrentHashMap<NodeId, Integer>` — no namespace, no tenant isolation,
no eviction, no persistence path.

### Design

Replace the private map with `FaultCountStore`:

- Accept `FaultCountStore` in constructor, default to `new InMemoryFaultCountStore()`
- Define namespace constant: `"pipeline-escalation"`
- Replace `faultCounts.merge(event.node(), 1, Integer::sum)` with
  `store.incrementAndGet(NAMESPACE, tenancyId, event.node())`
- Replace `faultCounts` map reads with `store.getCount(NAMESPACE, tenancyId, nodeId)`
- Remove the `faultCounts` field

**Three-tier escalation logic unchanged.** The counting substrate changes from a
private `ConcurrentHashMap` to `FaultCountStore`.

**Behavioral change — per-tenant counting:** The current code keys counts by `NodeId`
only. If a shared policy instance handles multiple tenants, counts are global across
tenants. The migration keys by `(namespace, tenancyId, nodeId)`, isolating counts per
tenant. This is the correct semantics: a node failing in tenant-A should not contribute
to tenant-B's escalation threshold. Per-tenant isolation makes the behavior consistent
regardless of instantiation strategy.

**Lazy eviction for removed nodes:** When the faulted node is no longer in the graph
(`current.nodes().get(event.node()) == null`), call
`store.remove(NAMESPACE, tenancyId, event.node())` before returning. This matches the
pattern established in `ThresholdFaultPolicy.onFault()` and prevents orphaned counts
from accumulating in the store.

### Wiring

Since this is a teaching example, explicit constructor wiring:

```java
public ProvisionEscalationFaultPolicy(PipelineWorld world) {
    this(world, new InMemoryFaultCountStore());
}

public ProvisionEscalationFaultPolicy(PipelineWorld world, FaultCountStore store) {
    this.world = world;
    this.store = store;
}
```

---

## Testing Strategy

### #96 — Composite ReconciliationListener
- Unit test: multiple `GlobalReconciliationListener` beans + per-tenant `ReconciliationListener` all fire on cycle
- Unit test: one global listener throws → others and per-tenant listener still fire
- Unit test: global listeners fire on empty-plan cycles (GE-20260707 compliance)
- Existing `ReconciliationLoopLifecycleTest` tests remain green (per-tenant path unchanged)

### #94 — JpaFaultCountStore
- Unit tests mirroring `InMemoryFaultCountStoreTest` contract (all 5 SPI methods)
- `@QuarkusTest` with H2 (`MODE=PostgreSQL`) for JPA integration
- Test `incrementAndGet` atomicity under concurrent calls (upsert correctness)
- Test `evict` with retained node set and with empty retained set (branch coverage)
- Test `reset` on non-existent key creates zero-count row
- Test CDI priority: JPA beats `@DefaultBean` when both on classpath

### #95 — ProvisionEscalationFaultPolicy migration
- Existing pipeline example tests remain green
- Test tenant isolation: two tenants, independent fault counts
- Test namespace scoping: escalation counts isolated from other policies
- Test lazy eviction: removed node triggers `store.remove()`, count resets on re-add

---

## Deferred Issues

| Concern | Follow-up | Rationale |
|---------|-----------|-----------|
| `evict()` has zero callers | File issue: FaultCountEvictionListener design | SPI method is correctly designed infrastructure; the eviction consumer requires namespace-aware design (each policy owns its namespace) — deferred to separate issue |
| ReconciliationLoop constructor telescope (8+ params) | File issue: constructor refactoring | Pre-existing tech debt. This spec adds one parameter; the telescope predates it. Introduce config/options object for test constructors |
| `persistence-backend-cdi-priority` protocol narrows @DefaultBean to "no-op or mock" | File protocol update issue | `DefaultFaultCountStore` is functional (extends `InMemoryFaultCountStore`), matching `SimpleTransitionExecutor @DefaultBean`. Protocol should clarify @DefaultBean can be a functional fallback |

---

## CLAUDE.md Updates

After implementation:
- Add `persistence-jpa/` to module table
- Add `DefaultFaultCountStore` to core runtime types
- Add `GlobalReconciliationListener` to api/ types
- Update `ReconciliationLoop` constructor documentation (global listeners param)
- Add `JpaFaultCountStore` and `FaultCountEntity` to core runtime types
- Note Flyway migration path: `db/desiredstate/migration/`
