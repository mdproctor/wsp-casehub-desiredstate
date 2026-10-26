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
  category: monitoring
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

Bundled primitives are loaded via platform's existing `YamlDefinitionSource` and `ScriptSource`
patterns. Activation order: bundled primitives register into platform's `PluginRegistry` first,
then the node type activates in `ExtensionRegistry`. A plugin can satisfy its own
`requires.primitives` from its bundled content.

### Repository SPI

```java
public interface PluginRepository {
    List<PluginManifest> discover();
    PluginPackage fetch(String name, String version);
    default void watch(Consumer<PluginEvent> listener) {}
}

public sealed interface PluginEvent {
    record Added(PluginManifest manifest) implements PluginEvent {}
    record Removed(String name, String version) implements PluginEvent {}
    record Updated(PluginManifest oldManifest, PluginManifest newManifest) implements PluginEvent {}
}
```

**FilesystemPluginRepository** is the first implementation. It watches `plugins/` using platform's
`io.methvin:directory-watcher` library and emits `PluginEvent` objects on changes. Future
implementations (git, HTTP) implement the same SPI.

### Dynamic Plugin Lifecycle

YAML plugins are dynamic — they load, update, and unload at runtime without restart.

**Activation cycle** (on filesystem change or explicit reload):
1. Parse plugin YAML, validate manifest
2. Check required primitives against platform `PluginRegistry` (after bundled primitives registered)
3. Check required types against `ExtensionRegistry`
4. Check for type collisions with existing active plugins
5. Register NodeProvisioner, ActualStateAdapter, fault policies, CBR, RAS
6. Fire `PluginActivatedEvent` (CDI/Spring)

**Deactivation guard:** A plugin cannot be deactivated if any active tenancy has nodes of its type
in a non-terminal state. `LifecycleManager` maintains a `Map<NodeType, Set<TenancyId>>` index,
eagerly populated when each TenantLoop starts from the `DesiredStateGraph`. The guard queries this
index — O(1) lookup. Operators can force-deactivate if needed.

**Version updates:** Deactivate old version (subject to guard), activate new version. Atomic
per-plugin — all tenants switch simultaneously.

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

**Loading order:** Plugins sorted topologically by type dependencies. Plugins with no dependencies
load first. Circular dependencies detected and rejected regardless of source.

**Conflict detection:**
- Type collision (two plugins providing same NodeType) → fatal, second rejected
- Primitive collision (bundled primitive clashes with platform primitive) → fatal
- Version conflict (same name, different versions) → only one active (D9)

### Cleanup and Migration

Existing mechanisms evolve into the new model — no remnant code left drifting:

| Current | Migration |
|---------|-----------|
| `PluginHeader(type, version int, resyncInterval, auth)` | Gains `requires:` section; `version` int → semver string |
| `PluginDescriptor` | Gains semver version field |
| `PluginDiscovery` | Evolves: classpath scanning preserved, consumed by `PluginCatalog` as one source alongside filesystem repositories |
| `PluginRuntimeBeans` (Quarkus CDI) | Refactored to consume from `PluginCatalog` |
| `DesiredStatePluginAutoConfiguration` (Spring) | Refactored to consume from `PluginCatalog` |
| `YamlPluginProcessor` (Quarkus build-time) | Validates classpath plugins with evolved `PluginHeader`; duplicate detection shared with catalog |
| Platform `PluginRegistry` (yaml-plugin-api) | Unchanged name (different layer); gains `unregister()` and replacement semantics (D10) |

**Naming:** Desiredstate's combined query registry is `ExtensionRegistry` (not `PluginRegistry`) to
avoid collision with platform's `PluginRegistry` for step primitives.

## Platform Prerequisites

Two changes required in platform before desiredstate can implement dynamic lifecycle:

### 1. Plugin Catalog Modules (D8)

New `plugin-catalog-api/` and `plugin-catalog-core/` modules providing generic, domain-agnostic
catalog infrastructure. See "Two-Layer Split" above for scope.

### 2. PluginRegistry SPI Evolution (D10)

Platform's `PluginRegistry` interface needs:
- `unregister(String actionName)` — removes a previously registered Definition
- `register(Definition)` behavior: `putIfAbsent` → `put` (replace, log warning on override)

Without these, deactivating a plugin leaves its bundled primitives orphaned, and version updates
cannot replace old primitives with new ones. The change is backward-compatible — existing consumers
that never call `unregister()` are unaffected.

## ExtensionRegistry

`ExtensionRegistry` provides operators a single query surface over all active extensions:

```java
public class ExtensionRegistry {
    Set<NodeType> availableTypes();
    boolean isTypeAvailable(NodeType type);
    Optional<PluginManifest> pluginManifest(NodeType type);  // null for Java SPI types
    Set<NodeType> dynamicTypes();     // types from catalog (dynamic)
    Set<NodeType> staticTypes();      // types from classpath (static)
}
```

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
| `plugin/runtime/` | `PluginCatalog` integration, `DesiredStatePluginActivator`, dynamic lifecycle, cross-tenant type index |
| `plugin/deployment/` | `YamlPluginProcessor` evolves to validate with evolved `PluginHeader` |
| `plugin/spring/` | `DesiredStatePluginAutoConfiguration` refactored to consume `PluginCatalog` |
| `plugin/testing/` | Test fixtures for dynamic plugin loading |

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
- Platform boundary-rules.md — cross-repo ownership rules
- Platform capabilities.md — org-wide capability index
- `examples/plugin-yaml/src/main/resources/META-INF/desiredstate/plugins/monitoring-endpoint.yaml` — example plugin
