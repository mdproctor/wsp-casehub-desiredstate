# Plugin Catalog and Versioning Design

**Issue:** #180 (part of #173 — complete YAML plugin Tier 2)
**Status:** Design
**Date:** 2026-10-26

## Problem Statement

YAML plugins are designed for operators who don't have a Java build chain — "no Java required."
But the current implementation defeats that promise: plugins are discovered from
`META-INF/desiredstate/plugins/*.yaml` on the classpath, requiring a Maven build and application
redeployment to add or update a plugin. There is no versioning (the `version` int field in
`PluginHeader` is unused), no dependency declaration, no conflict detection at runtime, and no
distribution model beyond "put YAML in a JAR."

## Goals

1. YAML plugins load dynamically from the filesystem without recompile or restart
2. Plugins are versioned with semver and declare their dependencies
3. Conflict detection and dependency validation fail fast at load time
4. Plugin packages can bundle custom step primitives alongside the node type definition
5. The extension model is consistent across the platform — no parallel mechanisms at tension
6. Existing classpath-based plugins continue to work with minimal migration

## Non-Goals

- Multi-version support (multiple versions of the same plugin active simultaneously)
- Per-tenant version pinning
- Remote repositories (git, HTTP) — SPI'd for future, not implemented now
- Cataloging GoalCompiler surfaces (annotations, YAML graphs, TS DSL) — these are compile-time
  declarations, not distributable plugins
- Multi-type plugins (one plugin providing multiple NodeTypes)

## Architecture

### Two-Layer Split

The design spans two repos following the existing platform convention (platform owns SPI
infrastructure, desiredstate builds domain-specific behavior on top):

**Platform** (new modules):
- `plugin-catalog-api/` — pure-Java SPI: `PluginManifest`, `PluginRepository`, `PluginCatalog`
  interfaces, `PluginRequirements`, `PluginEvent` (sealed), `ConflictException`
- `plugin-catalog-core/` — framework-neutral implementations: `FilesystemPluginRepository`,
  `DependencyResolver`, `ConflictDetector`, `DefaultPluginCatalog`

**Desiredstate** (domain-specific):
- `DesiredStatePluginManifest` extends `PluginManifest` with provisioner/actual-state/fault-policy/
  CBR/RAS fields
- `DesiredStatePluginActivator` — wires loaded plugins into NodeProvisioner, ActualStateAdapter, etc.
- `ExtensionRegistry` — combined query surface over all active extensions (GoalCompilers from
  CDI + plugins from catalog)
- Dynamic lifecycle management (watch/validate/activate)

### Plugin Identity

The plugin `name` IS the NodeType string (e.g., `monitoring-endpoint`). This preserves identity
coherence throughout the stack — `@NodeTypeId`, graph YAML `type:` references,
`DefaultNodeProvisionerRouter` routing, and `YamlPluginProcessor` conflict detection all use the
same string.

Version moves from int to semver (`major.minor.patch`). Only one version of a plugin may be active
per deployment. When a new version is loaded, it replaces the old (deactivate → activate).

No group prefix. Collision prevention is deployment-time fail-fast on duplicate names — the same
mechanism `YamlPluginProcessor` already uses at build time.

### Plugin Manifest (YAML Format)

The existing `plugin:` header evolves incrementally:

```yaml
plugin:
  type: monitoring-endpoint          # unchanged — IS the plugin identity
  version: 1.2.0                     # int → semver string
  description: "Monitoring endpoint lifecycle via REST API"
  resyncInterval: 60s                # unchanged
  auth:                              # unchanged
    monitoring-api:
      credentialRef: monitoring-api-credentials

  requires:                          # NEW — dependency declaration
    runtime: ">=1.0.0"
    primitives:
      - rest-call
      - compare-state
      - assert
    types:
      - human-review                 # NodeType from another plugin or Java SPI

# rest of the plugin YAML unchanged (spec, actual-state, provisioner, fault-policy, cbr, ras)
```

The `type` field stays as-is — it IS the plugin identity (D2). No `provides.types[]` — the
established one-file-per-type model makes it redundant (D3). The `requires` section is the core
new capability — it enables fail-fast validation at load time.

**Runtime version constraint:** `requires.runtime` specifies a semver range (e.g. `>=1.0.0`) that
the plugin is compatible with. The runtime version is derived from the Maven artifact version of
`casehub-desiredstate` at build time — a generated `RuntimeVersion` constant (string field on a
`BuildInfo` class, populated via Maven resource filtering or Quarkus `@ConfigProperty`). Range
parsing uses `com.github.zafarkhaja:java-semver` (already the idiomatic Java semver library).
At plugin load time, `PluginValidator` checks `RuntimeVersion.current()` against the declared
range. Mismatch → plugin rejected with clear error message naming expected vs actual version.

**Backward compatibility:** The `version` field accepts both int (`version: 1`, interpreted as
`1.0.0`) and semver string (`version: 1.2.0`). `PluginParser` detects the type and normalizes to
semver internally. Existing plugin YAML files continue to parse without changes.

### Plugin Packaging

Each plugin is a directory at `plugins/<type>/<version>/`:

```
plugins/
  monitoring-endpoint/
    1.2.0/
      plugin.yaml              # node type definition (mandatory)
      primitives/              # optional YAML-defined step primitives
        prometheus-query.yaml
      scripts/                 # optional script-backed primitives
        health-check.py
        health-check.schema.yaml
    1.3.0/
      plugin.yaml
  custom-db/
    2.0.0/
      plugin.yaml
```

**Version selection:** When multiple version directories exist for a plugin (e.g.
`monitoring-endpoint/1.2.0/` and `monitoring-endpoint/1.3.0/`), the highest semver version
wins on discovery. Filesystem changes (new directory appearing, directory removed) trigger
re-evaluation. Downgrades are not rejected — if `1.3.0/` is removed while `1.2.0/` exists,
`1.2.0` activates (subject to the version update lifecycle in §Dynamic Plugin Lifecycle). This
matches Kubernetes's "desired state converges to what's on disk" model.

**Bundled primitives:** YAML-defined step primitives are loaded via a new
`FilesystemYamlDefinitionSource` (platform change — see §Platform Prerequisites) that reads from
filesystem `Path` objects instead of the classpath. Bundled scripts are loaded via the existing
`ScriptSource`, which already accepts `Path` objects. Activation order: bundled primitives
register into platform's `PluginRegistry` first, then the node type activates in
`ExtensionRegistry`. A plugin can satisfy its own `requires.primitives` from its bundled content.

### Plugin Manifest SPI

`PluginManifest` is the central interface in `plugin-catalog-api/` — all catalog components compose
with it:

```java
public interface PluginManifest {
    String name();                          // plugin identity (= NodeType string)
    String version();                       // semver string
    Optional<String> description();         // human-readable, optional
    PluginRequirements requirements();      // runtime version, required primitives, required types
}

public record PluginRequirements(
    Optional<String> runtimeVersionRange,   // semver range, e.g. ">=1.0.0"
    List<String> primitives,                // required step primitive names
    List<String> types                      // required NodeType names
) {}
```

`PluginManifest` is pure-Java, domain-agnostic — it carries only identity, version, and dependency
metadata. Domain-specific fields (provisioner steps, actual-state steps, fault policies, CBR, RAS)
live in `DesiredStatePluginManifest extends PluginManifest`, which adds the desiredstate-specific
contract.

### Repository SPI

```java
public interface PluginRepository {
    List<PluginManifest> discover();
    PluginPackage fetch(String name, String version);
    default void watch(Consumer<PluginEvent> listener) {}
}

public record PluginPackage(
    PluginManifest manifest,
    Path pluginDirectory,       // root of the version directory (e.g. plugins/monitoring-endpoint/1.2.0/)
    List<Path> primitiveFiles,  // YAML step definition files in primitives/
    List<Path> scriptFiles      // script-backed primitives in scripts/
) {}

public sealed interface PluginEvent {
    record Added(PluginManifest manifest) implements PluginEvent {}
    record Removed(String name, String version) implements PluginEvent {}
    record Updated(PluginManifest oldManifest, PluginManifest newManifest) implements PluginEvent {}
}
```

`PluginPackage` encapsulates a resolved plugin directory — the parsed manifest plus the paths to
bundled primitives and scripts. The `DesiredStatePluginActivator` uses these paths to register
bundled primitives via `FilesystemYamlDefinitionSource` and `ScriptSource`.

`PluginEvent.Updated` is the repository-level event emitted when a version change is detected
(e.g. `1.3.0/` directory appears while `1.2.0/` is active). It carries both manifests so the
lifecycle layer can compare schemas, validate compatibility, and implement rollback if activation
of the new version fails. The lifecycle layer decomposes `Updated` into deactivate-old →
activate-new, but the event itself preserves the association between versions.

**FilesystemPluginRepository** is the first implementation. It watches `plugins/` using
`io.methvin:directory-watcher` (new platform dependency — see §Platform Prerequisites) and emits
`PluginEvent` objects on changes. Future implementations (git, HTTP) implement the same SPI.

### Dynamic Routing Infrastructure

The existing routing infrastructure (`DefaultNodeProvisionerRouter`, `DefaultActualStateAdapterRouter`)
uses `Map.copyOf()` — immutable snapshots constructed once from the set of discovered provisioners.
This is correct for classpath plugins (known at startup) but cannot support runtime addition/removal.

Dynamic plugins require a new `DynamicNodeProvisionerRouter` that implements `NodeProvisionerRouter`
with a `ConcurrentHashMap<NodeType, NodeProvisioner>` routing table:

```java
public class DynamicNodeProvisionerRouter implements NodeProvisionerRouter {
    private final ConcurrentHashMap<NodeType, NodeProvisioner> routing = new ConcurrentHashMap<>();

    void register(NodeProvisioner provisioner) { ... }   // adds all handledTypes()
    void unregister(NodeType type) { ... }               // removes type → provisioner mapping
}
```

Same pattern for `DynamicActualStateAdapterRouter`. Both are `@Singleton` CDI beans (Quarkus) or
Spring beans, produced at build time and populated with classpath plugins at startup. Dynamic plugins
are added/removed via the `register()`/`unregister()` methods at runtime.

`StatefulNodeProvisionerRouter` wrapping (lifecycle state machines) is applied by the
`DesiredStatePluginActivator` before registration — it wraps each dynamic plugin's provisioner
with the appropriate `NodeLifecycleDefinition` if one exists.

### Dynamic Policy Infrastructure

`FaultPolicyEngine` and `DriftPolicyEngine` have the same immutability constraint as the routing
tables — both use `List.copyOf(policies)` in their constructors and are held as `final` fields in
`ReconciliationLoop`. Dynamic plugins that declare fault policies or drift policies need a mutable
policy registration path.

**Interface extraction:** `ReconciliationLoop` currently declares its fields using the concrete
classes `FaultPolicyEngine` and `DriftPolicyEngine` (not interfaces). To enable dynamic
alternatives, two interfaces are extracted — following the same pattern that `NodeProvisionerRouter`
(interface) established for `DynamicNodeProvisionerRouter` in R1-02:

```java
public interface FaultPolicyEvaluator {
    List<GraphMutation<DesiredNode>> evaluate(String tenancyId, FaultEvent event,
                                              DesiredStateGraph current, ActualState actual);
}

public interface DriftPolicyEvaluator {
    DriftDecision evaluate(NodeId nodeId, NodeStatus status,
                           DesiredNode node, DriftContext context);
}
```

`ReconciliationLoop` changes its field types from `FaultPolicyEngine` → `FaultPolicyEvaluator` and
`DriftPolicyEngine` → `DriftPolicyEvaluator`. The existing concrete classes implement the new
interfaces with no behavioral change. The `ReconciliationLoop.Builder` accepts the interface types.

**`DynamicFaultPolicyEngine`** implements `FaultPolicyEvaluator` with a mutable policy list:

```java
public class DynamicFaultPolicyEngine implements FaultPolicyEvaluator {
    private final CopyOnWriteArrayList<FaultPolicy> policies;

    public DynamicFaultPolicyEngine(List<FaultPolicy> staticPolicies) {
        this.policies = new CopyOnWriteArrayList<>(staticPolicies);
    }

    void addPolicy(FaultPolicy policy) { ... }
    void removePolicy(FaultPolicy policy) { ... }

    // evaluate() identical to FaultPolicyEngine — iterates policies, merges mutations, detects conflicts
}
```

Same pattern for **`DynamicDriftPolicyEngine`** implementing `DriftPolicyEvaluator` with
`CopyOnWriteArrayList<DriftPolicy>`.

`CopyOnWriteArrayList` is the correct data structure: reads are lock-free (every reconciliation
cycle evaluates policies), writes are rare (only when plugins are added/removed), and the copy-on-
write semantics guarantee that in-flight evaluations always see a consistent snapshot.

For static-only deployments, the existing `FaultPolicyEngine`/`DriftPolicyEngine` with
`List.copyOf()` satisfy the interfaces unchanged.

**CBR metadata** registration uses `CbrPluginMetadata` beans — these are already managed by the
`DesiredStatePluginActivator` internally (not injected into `ReconciliationLoop`). The activator
maintains a `ConcurrentHashMap<NodeType, CbrPluginMetadata>` for dynamic plugins.

**RAS situation definitions** use `SituationDefinitionProvider` — `YamlPluginRasRegistrar` implements
this interface and produces `SituationRegistration` objects. The activator registers dynamic RAS
situations with `SituationDefinitionRegistry` (`io.casehub.ras.runtime`) via its existing
`register(SituationRegistration)` method. Deregistration on plugin deactivation uses
`deregister(String situationId)`. Both methods are `synchronized` and rebuild the internal
`RegistrySnapshot` atomically.

### Dynamic Plugin Lifecycle

YAML plugins are dynamic — they load, update, and unload at runtime without restart.

**Activation mechanism — catalog-aware activator (not per-plugin CDI beans):**

Quarkus's build-time CDI model (ArC) does not support runtime bean registration.
The `DesiredStatePluginActivator` is itself a CDI/Spring bean registered at build time. It manages
dynamic plugins internally without per-plugin CDI bean registration:

1. `DesiredStatePluginActivator` holds a reference to `DynamicNodeProvisionerRouter`,
   `DynamicActualStateAdapterRouter`, `DynamicFaultPolicyEngine`, `DynamicDriftPolicyEngine`,
   platform `PluginRegistry`, and `ExtensionRegistry`
2. For each activated plugin, it creates per-plugin `YamlPluginProvisioner` and
   `YamlPluginActualStateAdapter` instances (each handling a single type)
3. It registers these directly with the dynamic routers via `register()`
4. Bundled primitives are registered with the platform `PluginRegistry`
5. Fault policies, CBR, and RAS definitions are registered with their respective runtime services

No CDI/Spring bean registration is needed for individual dynamic plugins. The activator is the
single entry point for all plugin lifecycle operations.

For Spring: `DesiredStatePluginAutoConfiguration` follows the same activator pattern. Spring's
`GenericApplicationContext.registerBean()` is available but not needed — the activator manages
plugins internally for consistency with the Quarkus path.

**Activation cycle** (on filesystem change or explicit reload):
1. Parse plugin YAML, validate manifest (including `requires.runtime` check)
2. Register bundled primitives via `FilesystemYamlDefinitionSource` and `ScriptSource`
3. Check required primitives against platform `PluginRegistry` (bundled primitives now registered)
4. Check required types against `ExtensionRegistry`
5. Check for type collisions with existing active plugins
6. Create per-plugin `YamlPluginProvisioner` and `YamlPluginActualStateAdapter`
7. Register provisioner/adapter with dynamic routers
8. Register fault policies via `DynamicFaultPolicyEngine.addPolicy()`, drift policies via
   `DynamicDriftPolicyEngine.addPolicy()`, CBR metadata into activator's internal map, and RAS
   situations via `SituationDefinitionRegistry.register(SituationRegistration)`
9. Update `ExtensionRegistry` with the new type
10. Fire `PluginActivatedEvent` via CDI/Spring event bus

**Activation rollback on partial failure:** If any step 2-9 fails after earlier steps have
succeeded, the activator executes a compensating cleanup that undoes all completed registrations in
reverse order. This prevents inconsistent states such as a routable provisioner (step 7) with no
fault policies (step 8 failed), or orphaned primitives in `PluginRegistry` (step 2 succeeded but
step 7 failed). The cleanup sequence:

- Unregister from `ExtensionRegistry` (undo step 9)
- Deregister RAS situations via `SituationDefinitionRegistry.deregister()` (undo step 8)
- Remove policies from `DynamicFaultPolicyEngine`/`DynamicDriftPolicyEngine` (undo step 8)
- Unregister provisioner/adapter from dynamic routers (undo step 7)
- Unregister bundled primitives from platform `PluginRegistry` via `unregister()` (undo step 2)

Steps 1, 3-6 are validation/creation steps with no external side effects — they don't need
compensating actions. The plugin enters `FAILED` state with the originating exception logged.
Retry on next filesystem change or explicit reload.

**Deactivation guard:** A plugin cannot be deactivated if any active tenancy has nodes of its type
in a non-terminal state. The guard uses the existing `ReconciliationStateStore` interface
(`api/src/main/java/io/casehub/desiredstate/api/ReconciliationStateStore.java`), which currently
provides `store()`, `load()`, and `remove()` keyed by tenancyId. Two methods must be added:

- `Set<String> allTenancyIds()` — returns all tenancyIds with stored graphs (for the in-memory
  store, this is `snapshots.keySet()`; for JPA stores, a simple `SELECT DISTINCT tenancy_id` query)
- No query-by-NodeType method is needed — the guard iterates `allTenancyIds()`, calls `load()` for
  each, and checks the graph's `nodes()` for the target type. This is correct because deactivation
  is a rare operation (not on the reconciliation hot path), and the iteration is bounded by the
  number of tenancies.

**"Last reconciled" vs "current" graph gap:** `ReconciliationStateStore` stores the last *reconciled*
desired graph, which may lag behind the *current* desired graph (held in `TenantLoop.desiredRef`'s
`AtomicReference`). This gap is bounded by one reconciliation cycle and is safe given the multi-step
deactivation protocol:
1. Mark the plugin as `DEACTIVATING` — the dynamic router stops accepting new provisions for this type
   (returns `ProvisionResult.Failed("Plugin deactivating")`)
2. Wait for in-flight reconciliation cycles to complete (bounded by the reconciliation cycle timeout)
3. After in-flight cycles complete, the store reflects the latest reconciled state — the gap is closed
4. Re-check the guard against the now-current store state
5. If clear, unregister from dynamic routers and `ExtensionRegistry`

The `DEACTIVATING` state in step 1 prevents new nodes of the target type from being provisioned,
which closes the window for new graph mutations to add nodes of that type during steps 2-4.

Operators can force-deactivate if needed, bypassing the guard.

**Version updates:** When a new version directory appears:
1. The `PluginEvent.Updated` event carries both old and new manifests
2. If the deactivation guard blocks the old version, the update is **queued**: the new version is
   validated and prepared but not activated. The queue is re-evaluated on each reconciliation cycle.
3. If the guard doesn't clear within a configurable timeout (default 5 minutes), a warning is logged
   and the old version remains active. The new version stays on disk — it will be retried on the
   next explicit reload or filesystem change.
4. If activation of the new version fails after deactivation of the old, the old version is
   **rolled back** — reactivated from its still-present directory. If the old version's directory
   has been removed (operator upgraded by replacing rather than adding alongside), rollback is
   impossible — the plugin type becomes **unavailable** until the operator intervenes. This is
   logged as a clear error: `"Plugin <type> version <new> activation failed; rollback to <old>
   impossible — directory removed. Restore <old> to plugins/<type>/<old>/ to recover."` The
   deactivation guard (step 2) minimizes this window, but it cannot prevent a determined operator
   from deleting the old directory between deactivation and activation.

**Reload endpoint:** `POST /plugins/reload` triggers re-scan of the filesystem plugin directory.
Handles environments where filesystem notifications are unreliable (NFS, container overlay mounts).

### Dependency Validation

Dual validation strategy matching the deployment model:

**Classpath plugins** (deterministic build-time ordering): strict fail-fast. Missing primitive or
type → immediate error. This is the existing behavior from `PluginValidator`.

**Filesystem plugins** (non-deterministic runtime ordering): retry-with-timeout. Missing dependency
→ "waiting" state with warning log. Retried on each filesystem change event. Timeout after
configurable period (default 60s), then fail with clear error. This matches Kubernetes controller
behavior — controllers don't fail-fast on missing CRDs, they retry until dependencies appear.

**Loading order:** Plugins sorted topologically by **type** dependencies. Plugins with no type
dependencies load first. Circular type dependencies are detected and rejected regardless of source.

**Primitive-mediated dependencies** are handled separately from the topological sort. If Plugin A
requires primitive `custom-query` which is bundled in Plugin B, this dependency is resolved by the
retry-with-timeout mechanism (not topological ordering). The rationale: primitive dependencies are
structurally looser than type dependencies — a primitive can be provided by the platform itself, by
any plugin, or by the deployment environment. Topological ordering would require the resolver to
know *which* plugin provides each primitive, creating tight coupling that the retry mechanism avoids.
In practice, bundled primitives are registered during activation step 2 (before dependency checks at
step 3), so intra-plugin primitive dependencies are always satisfied. Cross-plugin primitive
dependencies resolve naturally as plugins load in parallel.

**Conflict detection:**
- Type collision (two plugins providing same NodeType) → fatal, second rejected
- Primitive collision (bundled primitive clashes with platform primitive) → fatal
- Version conflict (same name, different versions) → only one active (D9)

### Cleanup and Migration

Existing mechanisms evolve into the new model — no remnant code left drifting:

| Current | Migration |
|---------|-----------|
| `PluginHeader(type, version int, resyncInterval, auth)` | Gains `description` (optional String), `requires:` section; `version` int → semver string |
| `PluginDescriptor` | Gains semver version field |
| `PluginDiscovery` | Evolves: classpath scanning preserved, consumed by `PluginCatalog` as one source alongside filesystem repositories |
| `PluginRuntimeBeans` (Quarkus CDI) | Refactored to consume from `PluginCatalog` |
| `DesiredStatePluginAutoConfiguration` (Spring) | Refactored to consume from `PluginCatalog` |
| `YamlPluginProcessor` (Quarkus build-time) | Validates classpath plugins with evolved `PluginHeader`; duplicate detection shared with catalog |
| Platform `PluginRegistry` (yaml-plugin-api) | Unchanged name (different layer); gains `unregister()` and replacement semantics (D10) |

**Naming:** Desiredstate's combined query registry is `ExtensionRegistry` (not `PluginRegistry`) to
avoid collision with platform's `PluginRegistry` for step primitives.

## Platform Prerequisites

Four changes required in platform before desiredstate can implement dynamic lifecycle:

### 1. Plugin Catalog Modules (D8)

New `plugin-catalog-api/` and `plugin-catalog-core/` modules providing generic, domain-agnostic
catalog infrastructure. See "Two-Layer Split" above for scope.

### 2. PluginRegistry SPI Evolution (D10)

Platform's `PluginRegistry` interface needs:
- `unregister(String actionName)` — removes a previously registered Definition
- `register(Definition)` behavior: `putIfAbsent` → `put` (replace, log warning on override)

Without these, deactivating a plugin leaves its bundled primitives orphaned, and version updates
cannot replace old primitives with new ones. The change is backward-compatible — existing consumers
that never call `unregister()` are unaffected. `CompositePluginRegistry` already uses
`ConcurrentHashMap` internally, so the implementation change is minimal.

### 3. `io.methvin:directory-watcher` Dependency

New platform dependency for filesystem watching. This is NOT an existing platform library — it must
be introduced. Rationale for `io.methvin:directory-watcher` over raw `java.nio.file.WatchService`:
- macOS native events (FSEvents via JNA) — `WatchService` uses polling on macOS, which is slow and
  misses rapid changes
- Recursive directory watching built-in
- Event deduplication (multiple rapid filesystem events collapsed to one)
- On Linux (production), delegates to JDK's `WatchService` (inotify) with no overhead

The explicit `POST /plugins/reload` endpoint handles environments where filesystem notifications
are unreliable (NFS, container overlay mounts, Kubernetes ConfigMap symlink-swap updates).

### 4. Filesystem YAML Definition Loading

`YamlDefinitionSource` currently loads exclusively from the classpath (`getResourceAsStream()`).
For filesystem-loaded plugin directories, this must be extended. Two options:

- **(Recommended)** New `FilesystemYamlDefinitionSource` class that accepts `List<Path>` instead of
  `List<String>` and reads from the filesystem. Same parsing logic, different I/O source. Clean
  separation — classpath loading is unchanged, filesystem loading is a new code path.
- Alternative: add a `loadFromPath(Path)` method to existing `YamlDefinitionSource`. Less clean —
  mixes two I/O sources in one class.

`ScriptSource` already accepts `Path` objects and loads from the filesystem — no change needed for
script-backed primitives.

## ExtensionRegistry

`ExtensionRegistry` provides operators a single query surface over all active extensions:

```java
public class ExtensionRegistry {
    Set<NodeType> availableTypes();
    boolean isTypeAvailable(NodeType type);
    Optional<DesiredStatePluginManifest> pluginManifest(NodeType type);  // empty for Java SPI types
    Set<NodeType> dynamicTypes();     // types from catalog (dynamic)
    Set<NodeType> staticTypes();      // types from classpath (static)
}
```

`pluginManifest()` returns `Optional<DesiredStatePluginManifest>` (the desiredstate-specific
subtype, not the platform `PluginManifest`), preserving access to fault policies, CBR metadata,
and RAS definitions. `ExtensionRegistry` lives in desiredstate's `plugin/api/` module, so using the
domain-specific type is appropriate.

It combines: GoalCompiler-provided types (from CDI-discovered annotation/YAML/TS surfaces) +
plugin-provided types (from `PluginCatalog`). GoalCompiler discovery remains CDI/Jandex-based —
no catalog wrapper needed for compile-time extension mechanisms.

## Module Structure

### Platform (new)

| Module | Artifact | Purpose |
|--------|----------|---------|
| `plugin-catalog-api/` | `casehub-platform-plugin-catalog-api` | Pure-Java SPI: PluginManifest, PluginRepository, PluginCatalog interfaces |
| `plugin-catalog-core/` | `casehub-platform-plugin-catalog-core` | FilesystemPluginRepository, DependencyResolver, ConflictDetector, DefaultPluginCatalog |

### Desiredstate (evolution of existing plugin/ modules)

| Module | Change |
|--------|--------|
| `plugin/api/` | Gains `DesiredStatePluginManifest`, `ExtensionRegistry` |
| `plugin/runtime/` | `PluginCatalog` integration, `DesiredStatePluginActivator`, dynamic lifecycle management, deactivation guard via `ReconciliationStateStore` query |
| `plugin/deployment/` | `YamlPluginProcessor` evolves to validate with evolved `PluginHeader` |
| `plugin/spring/` | `DesiredStatePluginAutoConfiguration` refactored to consume `PluginCatalog` |
| `plugin/testing/` | Test fixtures for dynamic plugin loading — extends the existing plugin testing framework (#135: `PluginTestExtension`, declarative `*.test.yaml`, `HttpMockInfrastructure`, `ShellSandboxInfrastructure`, `FaultInjectingStepRunner`) with dynamic lifecycle test support: loading/unloading plugins at runtime during tests, validating deactivation guards, and testing version update rollback |

## Future Work (Out of Scope)

These require separate design discussions in their respective repos:

- **Remote repositories** — git and HTTP `PluginRepository` implementations (desiredstate)
- **Platform step primitive versioning** — `Definition` gaining optional version field (platform)
- **Pages TS catalog alignment** — `CompositeCatalog` aligned with catalog concepts (pages)
- **Ops type discovery consolidation** — `InfraTypeDiscovery` and `ContainerNodeTypes` unified (ops)
- **Multi-type plugins** — one plugin providing multiple NodeTypes (desiredstate, if demand emerges)
- **Per-tenant version pinning** — requires fundamental routing redesign (desiredstate)

## References

- Issue #180 — plugin catalog and versioning
- Issue #173 — epic: complete YAML plugin Tier 2
- Issue #131 — design: plugin architecture
- `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginHeader.java` — current header model
- `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginParser.java` — current parser
- `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginDiscovery.java` — current discovery
- `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginDescriptor.java` — current descriptor
- `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginValidator.java` — current validation
- `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginRuntimeBeans.java` — Quarkus CDI wiring
- `plugin/spring/src/main/java/io/casehub/desiredstate/plugin/spring/DesiredStatePluginAutoConfiguration.java` — Spring wiring
- `plugin/deployment/src/main/java/io/casehub/desiredstate/plugin/deployment/YamlPluginProcessor.java` — build-time validation
- Platform `PluginRegistry` — `yaml-plugin-api/src/main/java/io/casehub/yaml/plugin/api/PluginRegistry.java`
- Platform `CompositePluginRegistry` — `yaml-step-runtime/src/main/java/io/casehub/yaml/step/catalog/CompositePluginRegistry.java`
- Platform `YamlDefinitionSource` — `yaml-step-runtime/src/main/java/io/casehub/yaml/step/catalog/YamlDefinitionSource.java`
- Platform `ScriptSource` — `yaml-step-runtime/src/main/java/io/casehub/yaml/step/catalog/ScriptSource.java`
- Issue #135 — plugin testing framework design (`docs/specs/issue-135-plugin-testing-framework/2026-09-29-plugin-testing-framework-design.md`)
- Platform boundary-rules.md — cross-repo ownership rules
- Platform capabilities.md — org-wide capability index
- `examples/plugin-yaml/src/main/resources/META-INF/desiredstate/plugins/monitoring-endpoint.yaml` — example plugin
