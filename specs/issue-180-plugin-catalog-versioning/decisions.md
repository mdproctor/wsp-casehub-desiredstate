## D1: Overall extension model architecture

**Choice:** Plugin Catalog scoped to YAML plugins — `PluginCatalog` manages discovery, versioning, and lifecycle for YAML-defined plugins (NodeProvisioner, ActualStateAdapter, FaultPolicy) from classpath (`META-INF/desiredstate/plugins/`) and filesystem sources. `ExtensionRegistry` provides a combined query surface over all active extensions (GoalCompilers from annotations/YAML/TS surfaces + plugins from catalog) but does NOT wrap GoalCompiler discovery in a catalog model.
**Alternatives:**
- Unified Plugin Model with ClasspathPluginRepository — wraps CDI-discovered GoalCompilers in PluginManifest, adding a thin indirection layer over mechanisms that already work. Rejected: GoalCompiler surfaces (annotations, YAML graphs, TS DSL) produce graph declarations ("what should exist"), not plugins ("how to manage it"). They don't need versioning, filesystem distribution, or dynamic loading. ClasspathPluginRepository is indirection with no architectural payoff.
- YAML Plugin Catalog Only (no ExtensionRegistry) — simpler but loses the combined operator query surface
- Repository-Driven Everything — maximum consistency but massive refactoring, fights Quarkus build-time model
**Rationale:** The three extension surfaces serve fundamentally different purposes: annotations/YAML/TS declare desired state (GoalCompiler beans); YAML plugins declare provisioning behavior (NodeProvisioner, ActualStateAdapter, FaultPolicy beans). The catalog model is valuable precisely for the capabilities YAML plugins need: filesystem distribution, versioning, dependency resolution, dynamic loading. GoalCompiler discovery via CDI/Jandex already works and doesn't benefit from a catalog wrapper. ExtensionRegistry provides operators a single query point without imposing catalog semantics on compile-time extension mechanisms.
**Trade-offs:** Two discovery mechanisms (CDI for GoalCompilers, PluginCatalog for YAML plugins) rather than one unified model. This mirrors reality — compile-time and runtime-loaded extensions are genuinely different concerns.
**Sources:** Issue #131 (plugin architecture), #173 (Tier 2 epic), platform boundary-rules.md, platform capabilities.md, existing PluginDiscovery/PluginDescriptor/PluginHeader code, #87 decisions (YAML plugin architecture)
**Exploration:** quick
**Status:** revised — scoped catalog to YAML plugins only; dropped ClasspathPluginRepository adapter; GoalCompiler surfaces remain CDI-discovered (R1-01)

## D2: Plugin identity model

**Choice:** Name + semver version. The plugin `name` IS the NodeType string (e.g., `monitoring-endpoint`), which is already globally unique within a deployment via existing conflict detection in `YamlPluginProcessor`. Version moves from int to semver (`major.minor.patch`). No group prefix — collision prevention at deployment time (fail-fast on duplicate names from different sources) is both simpler and already proven.
**Alternatives:**
- Maven-style coordinates (`group:name:version`) — provides structural collision prevention via reverse-domain namespacing, but imposes Java ecosystem conventions on a surface designed for "no Java required" (#87). Creates an identity coherence gap: graph YAML files reference `type: monitoring-endpoint` (NodeType string), but the catalog would identify plugins by `io.casehub.plugins.monitoring:prometheus-endpoint:1.2.0` — two incompatible identity systems. Designs for a third-party ecosystem with zero validated demand.
- Scoped names (`@scope/name@version`, npm-style) — lighter but alien to the Java ecosystem
- Hierarchical path (`category/name:version`) — makes categories load-bearing; renaming a category changes identity
**Rationale:** NodeType string is already the canonical identity for plugins — `@NodeTypeId`, graph YAML `type:` references, `DefaultNodeProvisionerRouter` routing, and `YamlPluginProcessor` conflict detection all use it. Adding Maven coordinates creates a second, incompatible identity system that must be reconciled everywhere NodeType is referenced. Name + version preserves identity coherence while adding the version dimension. If a third-party ecosystem materialises in the future, a group prefix can be added as an optional namespace without retrofitting every existing plugin.
**Trade-offs:** No structural collision prevention (reverse-domain). Mitigated by deployment-time fail-fast on duplicate names — the same mechanism `YamlPluginProcessor` already uses at build time.
**Sources:** Existing `PluginHeader.version` (int, currently unused), `@NodeTypeId` type string, `YamlPluginProcessor` duplicate detection, #87 D16 ("no Java required")
**Exploration:** quick
**Depends on:** D1 (catalog scoped to YAML plugins — identity must work for this scope)
**Status:** revised — replaced Maven coordinates with name + semver; preserves NodeType identity coherence (R1-02)

## D3: PluginManifest structure

**Choice:** Evolve existing `plugin:` header in YAML plugins incrementally: `type` stays as-is (it IS the NodeType string), `version` evolves from int to semver string, new `requires:` section declares dependencies (runtime version, step primitives, other node types). Existing fields (resyncInterval, auth) stay. `provides.types[]` dropped — the existing one-file-per-type model (#87 D8) makes it redundant, and multi-type plugins have zero validated demand.
**Alternatives:**
- Full manifest redesign with provides.types[] — adds a multi-type capability with no evidence of need; the `types[]` list always has one element
- Separate manifest file alongside plugin YAML — keeps plugin YAML clean but splits metadata across files
- Minimal manifest (identity only, no requires) — simpler but loses the fail-fast validation that is the core value proposition
**Rationale:** The `requires` section is the genuinely valuable addition — fail-fast validation of missing primitives and types at load time catches errors that would otherwise surface as confusing runtime failures. The rest of the header evolves minimally: `type` stays (it IS the plugin identity per revised D2), `version` gains semver semantics. No `provides.types[]` because one-file-per-type is the established model and there's no evidence multi-type plugins are needed.
**Trade-offs:** Breaking change to `version` field (int → semver string). PluginHeader, PluginDescriptor, PluginDiscovery, PluginRuntimeBeans, YamlPluginProcessor, PluginParser, and Spring auto-config all need migration — this is a cross-module breaking change that should be explicitly acknowledged.
**Sources:** Existing `PluginHeader(type, version, resyncInterval, auth)`, `PluginDescriptor`, #87 D8 (one file per type), monitoring-endpoint.yaml example
**Exploration:** quick
**Depends on:** D2 (identity model defines the version semantics)
**Status:** revised — dropped provides.types[]; type field unchanged; version int→semver; explicit cross-module migration cost acknowledgement (R1-03)

## D4: Repository model, filesystem layout, and plugin packaging

**Choice:** Flat directory layout for filesystem repositories. Each plugin is a **directory** at `plugins/<type>/<version>/`. The directory contains:
- `plugin.yaml` — node type definition (mandatory)
- `primitives/` — optional YAML-defined step primitives (loaded via platform's `YamlDefinitionSource` pattern)
- `scripts/` — optional script-backed primitives (loaded via platform's `ScriptSource` pattern)

PluginRepository SPI abstracts over transport: `discover() → List<PluginManifest>`, `fetch(name, version) → PluginPackage`, `watch() → change events`. Filesystem implementation is first; git and HTTP implementations follow later via the same SPI.

Activation order: bundled primitives are registered into platform's `PluginRegistry` (step primitives) first, then the node type is activated in `ExtensionRegistry`. This means a plugin can satisfy its own `requires.primitives` from its bundled content.

**Prerequisite:** Platform's `PluginRegistry` SPI must evolve to support `unregister()` and `replace()` before dynamic lifecycle (D5) can function. See D10.
**Alternatives:**
- Maven group-as-path (`plugins/<group-as-path>/<name>/<version>/`) — produces 7-level nesting for `io.casehub.plugins.monitoring:prometheus-endpoint:1.2.0`; operator-hostile. Rejected with D2 revision.
- Single-file model (plugin.yaml only, no bundled primitives) — simpler but forces third parties to contribute primitives separately to the platform, fragmenting the distribution
- Flat watched directory (single YAML files with naming convention) — `<type>.yaml` + `<type>-primitives/` is messier than a directory and doesn't version
**Rationale:** Bundled-directory model ties the two extension layers together cleanly — platform `PluginRegistry` (vocabulary) and desiredstate `ExtensionRegistry` (compositions) are both extendable from one package. Flat directory layout (`plugins/<type>/<version>/`) is only 3 levels deep — operator-friendly. The platform already has `YamlDefinitionSource` and `ScriptSource` patterns for non-classpath primitives — bundled primitives reuse these mechanisms.
**Trade-offs:** Plugin packages are directories, not single files, which is slightly less portable (but archives/tarballs can wrap them for transport). Requires platform PluginRegistry SPI evolution (D10) for dynamic lifecycle support.
**Sources:** Platform `YamlDefinitionSource`, platform `ScriptSource`, existing `META-INF/desiredstate/plugins/` classpath convention
**Exploration:** quick
**Depends on:** D2 (name + version defines the directory structure), D1 (catalog scoped to YAML plugins), D10 (PluginRegistry SPI evolution)
**Status:** revised — simplified directory layout from Maven group-as-path to flat type/version; explicit PluginRegistry SPI prerequisite (R1-04, R1-10)

## D5: Dynamic plugin lifecycle

**Choice:** Watch + validate + activate cycle. FilesystemPluginRepository watches directories for changes using `io.methvin:directory-watcher` (platform filesystem-watching protocol). On change: (1) parse & validate manifest, check required primitives and type conflicts; (2) register in PluginRegistry, wire NodeProvisioner, ActualStateAdapter, fault policies, CBR, RAS; (3) fire CDI/Spring events (PluginActivatedEvent, PluginDeactivatedEvent). Updates deactivate old then activate new. Deactivation guard: a plugin cannot be deactivated if any active tenancy has nodes of its type in a non-terminal state (wait or require explicit force).

**Cross-tenant type index:** LifecycleManager maintains a `Map<NodeType, Set<TenancyId>>` index tracking which tenants have active nodes of each type. Updated on plugin activation (tenant starts reconciling a type) and node state changes. Deactivation guard queries this index — O(1) lookup, no iteration over tenant loops required.

**Reload fallback:** In addition to file watching, an explicit reload endpoint (`POST /plugins/reload`) triggers re-scan of the filesystem plugin directory. This handles environments where filesystem notifications are unreliable (NFS, some container overlay mounts) without requiring a full application restart.
**Alternatives:**
- Restart-on-change — watch for changes but restart the application; simpler but defeats the "no restart for YAML plugins" promise
- Poll-based only — more controlled but less responsive; unnecessary given platform's `directory-watcher` library handles container edge cases
- Raw `WatchService` — platform protocol explicitly mandates `io.methvin:directory-watcher` over raw WatchService for cross-platform reliability
**Rationale:** Delivers on the dynamic/no-restart requirement. Watch + validate + activate is the standard pattern (Kubernetes controllers, OSGi bundles). Platform's `io.methvin:directory-watcher` protocol handles the cross-platform edge cases (NFS, container mounts) that raw `WatchService` misses. The explicit reload endpoint provides a fallback for environments where even `directory-watcher` is unreliable. Cross-tenant type index makes deactivation guard efficient without cross-tenant iteration.
**Trade-offs:** More complex than restart-on-change; deactivation guard adds lifecycle state tracking via the type→tenancy index; reload endpoint adds an operational surface
**Sources:** Existing ReconciliationLoop event model, StatefulNodeProvisioner lifecycle, DesiredStateEventTypes CloudEvent patterns, platform filesystem-watching-library.md protocol
**Exploration:** quick
**Depends on:** D1 (catalog scoped to YAML plugins), D4 (repository SPI provides watch capability), D10 (PluginRegistry SPI evolution for deactivation)
**Status:** revised — added cross-tenant type index mechanism, explicit reload endpoint, specified platform's directory-watcher library (R1-05)

## D6: Cleanup, alignment, and naming

**Choice:** Desiredstate's combined query registry is named `ExtensionRegistry` (not `PluginRegistry`) to avoid collision with platform's `PluginRegistry` (step primitives). Existing mechanisms evolve into the new model:
- `PluginHeader` → gains `requires:` section, `version` int→semver (D3)
- `PluginDescriptor` → gains version (semver) field, populated from evolved PluginHeader
- `PluginDiscovery` → evolves to support both classpath (`META-INF/desiredstate/plugins/`) and filesystem sources; existing `discover()` and `discoverAll()` methods remain for classpath plugins; new `PluginCatalog` aggregates classpath discovery + filesystem `PluginRepository` sources
- `PluginRuntimeBeans` (Quarkus) → refactored to consume from PluginCatalog
- `DesiredStatePluginAutoConfiguration` (Spring) → refactored to consume from PluginCatalog
- `YamlPluginProcessor` (Quarkus build-time) → validates classpath plugins, produces evolved PluginHeader metadata; duplicate detection shared with catalog
- Platform `PluginRegistry` (yaml-plugin-api) → unchanged, different layer
**Alternatives:**
- Remove PluginDiscovery entirely, replace with PluginCatalog — more disruptive than necessary; PluginDiscovery's classpath scanning logic is reusable within the catalog
- Rename platform's to `StepPrimitiveRegistry` — more accurate but larger blast radius (platform repo, many consumers)
- Keep both as `PluginRegistry` in different packages — creates confusion in imports and documentation
**Rationale:** `ExtensionRegistry` accurately describes the broader scope (GoalCompilers + YAML plugins). PluginDiscovery evolves rather than being removed wholesale — its classpath scanning is sound and reusable. The catalog is an aggregation layer that includes PluginDiscovery as one source alongside filesystem repositories. Migration is incremental, not wholesale replacement.
**Trade-offs:** PluginDiscovery continues to exist but is consumed by PluginCatalog rather than being called directly by PluginRuntimeBeans/Spring auto-config. Breaking change to PluginHeader version field.
**Sources:** Platform PluginRegistry (yaml-plugin-api), existing PluginDiscovery, PluginRuntimeBeans, DesiredStatePluginAutoConfiguration
**Exploration:** quick
**Depends on:** D1 (catalog scoped to YAML plugins), D3 (manifest structure defines migration target)
**Status:** revised — PluginDiscovery evolves rather than being removed; PluginCatalog aggregates classpath + filesystem sources (R1-06)

## D7: Conflict detection and dependency resolution

**Choice:** Dual validation strategy — strict fail-fast for classpath plugins, retry-with-timeout for filesystem plugins. The `PluginCatalog` validates the full dependency graph:

Conflicts detected:
- Type collision — two plugins providing the same NodeType → fatal, second plugin rejected
- Version conflict — same name with different versions loaded → only one version active per plugin (see D9)
- Primitive collision — bundled primitive name clashes with an existing platform primitive → fatal

Dependencies resolved:
- Required primitives — checked against platform `PluginRegistry` after bundled primitives are registered. Missing → behavior depends on source:
  - **Classpath plugins:** fail-fast at build time (ordering is deterministic)
  - **Filesystem plugins:** log warning, enter "waiting" state, retry on each filesystem change event. Timeout after configurable period (default 60s), then fail with clear error message
- Required types — same dual strategy: classpath fail-fast, filesystem retry-with-timeout
- Runtime version — compared against running desiredstate runtime version. Incompatible → fail regardless of source
- Circular dependencies detected and rejected regardless of source

Loading order: plugins sorted topologically by type dependencies. Plugins with no type dependencies load first.
**Alternatives:**
- Strict fail-fast for everything — catches errors immediately but creates a deployment ordering problem for filesystem plugins: Plugin B (requires X) arrives before Plugin A (provides X) → application fails to start. Operators must carefully order deployments or deploy atomically — defeating the purpose of separate plugin packaging.
- Lenient mode (warn and load anyway) — defers errors to runtime when provisioner fails mid-reconciliation. Strictly worse than retry-with-timeout — errors should surface, just not instantly when ordering is non-deterministic.
**Rationale:** The validation mode should match the deployment model. Classpath plugins have deterministic ordering (build-time) — strict fail-fast is correct. Filesystem plugins have non-deterministic ordering (operator deploys at any time) — retry-with-timeout gives the deployment time to complete while still failing clearly if dependencies are genuinely missing. This matches Kubernetes controller behavior: controllers don't fail-fast on missing CRDs, they retry with backoff until the CRD appears.
**Trade-offs:** Two validation paths (strict vs retry) add complexity. Mitigated by sharing the validation logic — only the error handling differs (immediate fail vs queue for retry).
**Sources:** Existing `PluginValidator.validatePlugin()` (build-time validation), `PluginValidationException`, `YamlPluginProcessor` duplicate detection, Kubernetes controller CRD retry behavior
**Exploration:** quick
**Depends on:** D3 (manifest requires section defines what to validate), D4 (bundled primitives affect validation order), D6 (ExtensionRegistry is the unified validation target)
**Status:** revised — dual validation strategy (strict for classpath, retry-with-timeout for filesystem) replaces strict-only (R1-07)

## D8: Module placement — two-layer split across platform and desiredstate

**Choice:** Split into generic catalog infrastructure (platform) and domain-specific activation (desiredstate). Platform implements the generic layer first; desiredstate consumes it.

**Platform** (new modules) — generic, domain-agnostic catalog infrastructure:
- `plugin-catalog-api/` — pure-Java SPI: `PluginManifest` (interface), `PluginRequirements`, `PluginRepository` SPI, `PluginCatalog` (interface), `PluginEvent` (sealed), `ConflictException`
- `plugin-catalog-core/` — framework-neutral implementations: `FilesystemPluginRepository`, `DependencyResolver` (topological sort on provides/requires), `ConflictDetector` (provides-collision check), `DefaultPluginCatalog` (aggregates repositories, validates, tracks active plugins)

**Desiredstate** — domain-specific layer consuming platform's catalog:
- `DesiredStatePluginManifest` extends `PluginManifest` with provisioner/actual-state/fault-policy/CBR/RAS fields
- `DesiredStatePluginActivator` — wires loaded plugins into NodeProvisioner, ActualStateAdapter, FaultPolicy, etc.
- `ExtensionRegistry` — desiredstate's view of all active extensions (static classpath + dynamic catalog)
- Dynamic lifecycle management (watch/validate/activate cycle from D5)
- Evolution of existing `PluginDiscovery`, `PluginRuntimeBeans`, Spring auto-config (from D6)

**Ops** — future consumer: YAML plugins for container/infra types, loaded via desiredstate's catalog

**Implementation sequence:** Design spec for platform's generic layer is produced as a platform issue from this brainstorming session. Platform implements. Then desiredstate builds on top.

**Proposed future work (requiring separate design discussions in each affected repo):**
- Platform step primitives (`Definition`) gaining optional version field — requires platform design discussion, not a desiredstate decision
- Pages TS `CompositeCatalog` alignment with catalog concepts — requires Pages design discussion
- Ops `InfraTypeDiscovery` and `ContainerNodeTypes` consolidation — requires Ops design discussion
**Alternatives:**
- Build in desiredstate, extract to platform later — avoids cross-repo coordination now but means building the same thing twice or refactoring on extraction
- Everything in desiredstate — simpler but creates fragmentation when pages or agency need catalogs; the generic concepts (manifests, dependency resolution, conflict detection) are genuinely domain-agnostic
**Rationale:** The org-wide audit revealed the same pattern repeated across platform (4-source CompositePluginRegistry), pages (TS port of same), and desiredstate (YAML plugins). No repo has versioning or dependency resolution. The generic catalog abstractions (manifests, repositories, dependency resolution, conflict detection) are domain-agnostic and belong in platform alongside the existing step primitive infrastructure. Domain-specific activation (what a loaded plugin DOES) belongs in each consuming repo. This follows the existing convention: platform owns `PluginRegistry` (step primitive SPI), desiredstate builds `DesiredStatePrimitiveRegistrar` on top of it.
**Trade-offs:** Requires cross-repo coordination (platform issue first, then desiredstate work). But the platform work is small and well-defined.
**Sources:** Org-wide audit: platform 15+ registries with no versioning, pages TS port of step infrastructure, ops two competing type discovery strategies, platform boundary-rules.md SPI convention, platform spi-adapter-module-placement.md
**Exploration:** deep-analysis (org-wide audit across platform, ops, pages)
**Depends on:** D1 (catalog model), D2 (identity), D3 (manifest), D4 (repository/packaging)
**Status:** revised — alignment roadmap reframed as "proposed future work requiring separate design discussions" rather than decisions made here (R1-08)

## D9: Single version per plugin

**Choice:** Only one version of a plugin (same name) may be active per deployment. When a new version is loaded, it replaces the old version (deactivate old → activate new, per D5). Per-tenant version pinning is not supported.
**Alternatives:**
- Multi-version support (OSGi-style) — each version loaded in isolation, per-tenant version pinning (tenant A uses v1.2.0, tenant B uses v2.0.0). Dramatically more complex: requires per-version CDI bean sets, per-version PluginRegistry entries, per-version NodeProvisionerRouter routing, per-tenant plugin resolution. The NodeProvisionerRouter routes by NodeType, not NodeType+Version — multi-version would require a fundamental routing redesign.
- Per-tenant version pinning with single-active-version — only one version loaded at runtime but tenants can request specific versions. Requires version-aware deployment orchestration outside the catalog.
**Rationale:** Multi-version support is an order of magnitude more complex than the current design, and the operational benefit (gradual migration) can be achieved by other means: blue-green deployments, canary releases, or version-locked deployment slots. The platform's existing routing infrastructure (NodeProvisionerRouter, ActualStateAdapterRouter) routes by NodeType with no version dimension. Adding version-aware routing would be a fundamental architecture change across the entire stack, not just the catalog.
**Trade-offs:** Upgrading a plugin is an atomic operation across all tenants. For multi-tenant deployments, this means all tenants must be ready for the new version simultaneously. Mitigated by deployment-level strategies (blue-green, canary) and the deactivation guard (D5) which prevents deactivating a version with active nodes.
**Sources:** NodeProvisionerRouter (routes by NodeType), ActualStateAdapterRouter (routes by NodeType), ReconciliationLoop per-tenant instances (ARC42STORIES.MD §8)
**Exploration:** quick (surfaced by R1-09)
**Depends on:** D5 (lifecycle manages version transitions), D7 (version conflict detection)
**Status:** captured

## D10: Platform PluginRegistry SPI evolution

**Choice:** Platform's `PluginRegistry` interface must evolve to support dynamic lifecycle. Required additions:
- `unregister(String actionName)` — removes a previously registered Definition
- `register(Definition)` behavior changes from `putIfAbsent` (ignore duplicate) to `put` (replace, log warning on override)

This is a platform SPI change, requiring a platform design discussion and issue. The current `CompositePluginRegistry` implementation uses `ConcurrentHashMap.putIfAbsent()` — duplicate registrations are warned and ignored, preventing version updates. Without `unregister()`, deactivating a plugin (D5) leaves its primitives orphaned in the registry, and updating to a new version cannot replace old primitives with new ones.
**Alternatives:**
- Work around with a wrapper registry — desiredstate maintains its own registry that delegates to platform's, with additional unregister/replace semantics. Bad design: the gap is in the platform SPI, not in desiredstate's consumption of it.
- Scoped registries per plugin — each dynamically loaded plugin gets its own PluginRegistry instance. Complicates resolution (must search multiple registries) and conflicts with the existing single-registry model.
**Rationale:** The platform SPI should be fixed, not worked around. `unregister()` and replacement semantics are straightforward additions. The `CompositePluginRegistry` implementation change is mechanical: `putIfAbsent` → `put` for `register()`, add `remove()` for `unregister()`. This is a prerequisite for D4/D5's dynamic lifecycle model.
**Trade-offs:** Requires cross-repo coordination (platform issue). The platform SPI change is backward-compatible — existing consumers that never call `unregister()` are unaffected. The `register()` behavior change (putIfAbsent → put) could theoretically affect consumers that rely on the "first registration wins" guarantee, but this guarantee was never documented and the warning log suggests it was treated as a misconfiguration, not a feature.
**Sources:** Platform `PluginRegistry` interface (yaml-plugin-api), `CompositePluginRegistry` implementation (yaml-step-runtime), platform boundary-rules.md
**Exploration:** quick (surfaced by R1-10)
**Depends on:** D8 (platform work), D5 (dynamic lifecycle requires unregistration)
**Status:** captured

## D11: PluginManifest as both file format and runtime model

**Choice:** For v1, `PluginManifest` serves as both the YAML file deserialization model and the runtime metadata interface. Plugin authors write a YAML file that deserializes directly to the runtime model.
**Alternatives:**
- Separate `PluginManifestFile` (deserialization) and `PluginManifest` (runtime interface) — allows independent evolution of file format and runtime representation. More flexible but adds a mapping layer with no current need.
- Schema-based validation with runtime projection — YAML validated against JSON Schema, then projected into a runtime model. Maximum decoupling but heavyweight.
**Rationale:** For v1, the file format and runtime model are identical. Introducing a mapping layer before there's evidence of divergence pressure is unnecessary complexity. When evolution pressure arrives (e.g., the file format needs shorthand that the runtime model doesn't support, or the runtime model gains computed fields), the split can be introduced without breaking existing plugin YAML files — the deserialization model simply adds defaulting/expansion logic.
**Trade-offs:** Any change to the runtime model requires a compatible change to the file format (and vice versa) until the split is introduced. Acceptable for v1 where the model is still stabilizing.
**Sources:** Maven POM model (serves as both file format and runtime model), existing PluginHeader/PluginModel pattern
**Exploration:** quick (surfaced by R1-11)
**Status:** captured
