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

Two categories of listener:

| Category | Scope | Registration | Example |
|----------|-------|-------------|---------|
| Global | Application-scoped | CDI-discovered at construction | Bulk eviction, metrics |
| Per-tenant | Tenant lifecycle | `start(tenancyId, desired, listener)` / `setListener()` | LifecycleManager |

ReconciliationLoop accepts `List<ReconciliationListener>` at construction time — global
listeners discovered via CDI `Instance<ReconciliationListener>`. The per-tenant `listener`
field remains. `fireListener()` iterates global listeners, then calls the per-tenant listener.

This follows the established composition pattern: `MergedEventSource` (EventSource beans),
`FaultPolicyEngine` (FaultPolicy beans), `NodeProvisionerRouter` (NodeProvisioner beans).

### Changes

**`ReconciliationLoop`** (runtime/):
- Add `private final List<ReconciliationListener> globalListeners` field
- Thread `List<ReconciliationListener>` through all constructors (CDI constructor
  takes `Instance<ReconciliationListener>`, test constructors default to `List.of()`)
- `fireListener()` iterates `globalListeners` then calls per-tenant `listener`,
  each wrapped in try/catch (existing error isolation pattern)

**`ReconciliationListener`** (api/): No change. Stays `@FunctionalInterface`.

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

**`JpaFaultCountStore` operations:**

| SPI method | JPA operation |
|-----------|---------------|
| `incrementAndGet` | Find by key → if absent create(count=1), else set count+1, merge. `@Transactional` |
| `getCount` | Find by key → return count or 0 |
| `reset` | Find by key → set count=0, merge |
| `remove` | Find by key → remove |
| `evict` | Named query: DELETE WHERE namespace=? AND tenancy_id=? AND node_id NOT IN (?) |

All mutating methods are `@Transactional`. Contention is low (faults are
relatively rare), so find-then-merge without pessimistic locking is sufficient.

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

**Three-tier escalation logic unchanged.** Only the counting substrate changes.
The tenancyId parameter (already available in `onFault`) is now used for isolation.

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
- Unit test: multiple global listeners + per-tenant listener all fire on cycle
- Unit test: one global listener throws → others still fire
- Unit test: global listeners fire on empty-plan cycles (GE-20260707 compliance)
- Existing `ReconciliationLoopLifecycleTest` tests remain green (per-tenant path unchanged)

### #94 — JpaFaultCountStore
- Unit tests mirroring `InMemoryFaultCountStoreTest` contract (all 5 SPI methods)
- `@QuarkusTest` with H2 for JPA integration
- Test `incrementAndGet` atomicity under concurrent calls
- Test `evict` with retained node set
- Test CDI priority: JPA beats `@DefaultBean` when both on classpath

### #95 — ProvisionEscalationFaultPolicy migration
- Existing pipeline example tests remain green
- Test tenant isolation: two tenants, independent fault counts
- Test namespace scoping: escalation counts isolated from other policies

---

## CLAUDE.md Updates

After implementation:
- Add `persistence-jpa/` to module table
- Add `DefaultFaultCountStore` to core runtime types
- Update `ReconciliationLoop` constructor documentation (global listeners param)
- Add `JpaFaultCountStore` and `FaultCountEntity` to core runtime types
- Note Flyway migration path: `db/desiredstate/migration/`
