## D1: Overall extension model architecture

**Choice:** Unified Plugin Model with Catalog SPI — one `PluginManifest` metadata model across all extension types, with a two-channel model (static/classpath + dynamic/catalog) feeding into a single `PluginRegistry`.
**Alternatives:**
- YAML Plugin Catalog Only — simpler but creates parallel, inconsistent extension mechanisms
- Repository-Driven Everything — maximum consistency but massive refactoring, fights Quarkus build-time model
**Rationale:** Unifies the extension model without forcing existing classpath mechanisms to change how they work internally. ClasspathPluginRepository adapts existing discovery into the unified model. Dynamic YAML plugins get first-class lifecycle. Future extension types slot in by implementing PluginRepository.
**Trade-offs:** More upfront design work; classpath adapter adds a thin indirection layer over mechanisms that already work
**Sources:** Issue #131 (plugin architecture), #173 (Tier 2 epic), platform boundary-rules.md, platform capabilities.md, existing PluginDiscovery/PluginDescriptor/PluginHeader code
**Exploration:** quick
**Status:** captured

## D2: Plugin identity model

**Choice:** Maven-style coordinates — `group:name:version` (e.g., `io.casehub.plugins.monitoring:prometheus-endpoint:1.2.0`). Group provides reverse-domain namespacing for third parties. Categories are metadata on the manifest, not part of identity. Version moves from int to semver (`major.minor.patch`).
**Alternatives:**
- Scoped names (`@scope/name@version`, npm-style) — lighter but alien to the Java ecosystem this platform lives in
- Hierarchical path (`category/name:version`) — simpler but makes categories load-bearing; renaming a category changes identity
**Rationale:** Maven coordinates are the lingua franca of this ecosystem (everything builds with Maven). Group provides natural collision-free namespacing for third-party plugin authors. Categories as metadata can evolve without breaking identity. Semver enables proper compatibility ranges.
**Trade-offs:** More verbose than npm-style scoping; YAML plugin authors who aren't Java developers may find reverse-domain grouping unfamiliar
**Sources:** Existing `PluginHeader.version` (int, currently unused), `@NodeTypeId` type string, platform module-tier-structure convention
**Exploration:** quick
**Depends on:** D1 (unified model — identity must work across all extension types)
**Status:** captured

## D3: PluginManifest structure

**Choice:** Extend existing `plugin:` header in YAML plugins with identity (group, name, version), metadata (description, category), capabilities provided (types), and capabilities required (runtime version, step primitives, other node types). Existing fields (resyncInterval, auth) stay. For classpath extensions, manifest is inferred from Jandex/annotations or declared in a companion manifest file.
**Alternatives:**
- Separate manifest file alongside plugin YAML — keeps plugin YAML clean but splits metadata across files
- Minimal manifest (identity only, no provides/requires) — simpler but loses safety guardrails and self-documentation
**Rationale:** Inline manifest keeps each plugin self-contained (one file = one plugin). `provides.types` is derivable for single-type plugins but explicit declaration enables multi-type plugins in the future and makes dependency graphs inspectable. `requires` enables validation at load time — fail fast if a required primitive or type is missing.
**Trade-offs:** Existing `PluginHeader` needs migration — `type` becomes `name`, `version` becomes semver string. Breaking change to the plugin YAML format that must be handled gracefully.
**Sources:** Existing `PluginHeader(type, version, resyncInterval, auth)`, `PluginDescriptor`, monitoring-endpoint.yaml example
**Exploration:** quick
**Depends on:** D2 (identity model defines the coordinate fields in the manifest)
**Status:** captured

## D4: Repository model, filesystem layout, and plugin packaging

**Choice:** Maven-style directory layout for filesystem repositories. Each plugin is a **directory** (not a single file) at its coordinates: `plugins/<group-as-path>/<name>/<version>/`. The directory contains:
- `plugin.yaml` — node type definition (mandatory)
- `primitives/` — optional YAML-defined step primitives (loaded via platform's `YamlDefinitionSource` pattern)
- `scripts/` — optional script-backed primitives (loaded via platform's `ScriptSource` pattern)

PluginRepository SPI abstracts over transport: `discover() → List<PluginManifest>`, `fetch(coordinates) → PluginPackage`, `watch() → change events`. Filesystem implementation is first; git and HTTP implementations follow later via the same SPI.

Activation order: bundled primitives are registered into platform's `PluginRegistry` (step primitives) first, then the node type is activated in `ExtensionRegistry`. This means a plugin can satisfy its own `requires.primitives` from its bundled content.
**Alternatives:**
- Category-based layout (`plugins/<category>/<name>/<version>/`) — more browsable but loses namespace collision prevention; two third parties could collide on `monitoring/endpoint`
- Single-file model (plugin.yaml only, no bundled primitives) — simpler but forces third parties to contribute primitives separately to the platform, fragmenting the distribution
**Rationale:** Bundled-directory model ties the two extension layers together cleanly — platform `PluginRegistry` (vocabulary) and desiredstate `ExtensionRegistry` (compositions) are both extendable from one package. Group-as-path mirrors Maven conventions and prevents collisions structurally. The platform already has `YamlDefinitionSource` and `ScriptSource` patterns for non-classpath primitives — bundled primitives reuse these mechanisms.
**Trade-offs:** Deeper directory nesting; more complex loading sequence (primitives first, then plugin). Plugin packages are directories, not single files, which is slightly less portable (but archives/tarballs can wrap them for transport).
**Sources:** Maven local repository layout, platform `YamlDefinitionSource`, platform `ScriptSource`, existing `META-INF/desiredstate/plugins/` classpath convention
**Exploration:** quick
**Depends on:** D2 (coordinates define the directory structure), D1 (unified model spans both plugin layers)
**Status:** captured

## D5: Dynamic plugin lifecycle

**Choice:** Watch + validate + activate cycle. FilesystemPluginRepository watches directories for changes. On change: (1) parse & validate manifest, check required primitives and type conflicts; (2) register in PluginRegistry, wire NodeProvisioner, ActualStateAdapter, fault policies, CBR, RAS; (3) fire CDI/Spring events (PluginActivatedEvent, PluginDeactivatedEvent). Updates deactivate old then activate new. Deactivation guard: a plugin cannot be deactivated if any active tenancy has nodes of its type in a non-terminal state (wait or require explicit force).
**Alternatives:**
- Restart-on-change — watch for changes but restart the application; simpler but defeats the "no restart for YAML plugins" promise
- Poll-based with explicit reload command — more controlled but less dynamic; could complement file watching as an additional trigger
**Rationale:** Delivers on the dynamic/no-restart requirement. Watch + validate + activate is the standard pattern (Kubernetes controllers, OSGi bundles). Deactivation guard prevents yanking provisioners from under running reconciliation. CDI/Spring events let the reconciliation loop and other consumers react to plugin changes.
**Trade-offs:** More complex than restart-on-change; deactivation guard adds lifecycle state tracking; file watching has platform-specific edge cases (NFS, container mounts)
**Sources:** Existing ReconciliationLoop event model, StatefulNodeProvisioner lifecycle, DesiredStateEventTypes CloudEvent patterns
**Exploration:** quick
**Depends on:** D1 (unified model — lifecycle must handle both static and dynamic channels), D4 (repository SPI provides watch capability)
**Status:** captured

## D6: Cleanup, alignment, and naming

**Choice:** Desiredstate's unified registry is named `ExtensionRegistry` (not `PluginRegistry`) to avoid collision with platform's `PluginRegistry` (step primitives). Existing mechanisms are migrated into the unified model, not wrapped with parallel adapters:
- `PluginHeader` → migrated to `PluginManifest` (type→provides.types[0], version int→semver)
- `PluginDescriptor` → gains coordinate fields, populated from PluginManifest
- `PluginDiscovery` → removed, replaced by `PluginCatalog` aggregating PluginRepository sources
- `PluginRuntimeBeans` (Quarkus) → refactored to consume from PluginCatalog
- `DesiredStatePluginAutoConfiguration` (Spring) → refactored to consume from PluginCatalog
- `YamlPluginProcessor` (Quarkus build-time) → validates classpath plugins, produces PluginManifest-shaped metadata; duplicate detection moves to catalog
- Platform `PluginRegistry` (yaml-plugin-api) → unchanged, different layer
**Alternatives:**
- Rename platform's to `StepPrimitiveRegistry` — more accurate but larger blast radius (platform repo, many consumers)
- Keep both as `PluginRegistry` in different packages — technically works but creates confusion in imports and documentation
**Rationale:** `ExtensionRegistry` accurately describes the broader scope (Java SPIs, annotations, YAML graphs, YAML plugins, TS DSL — not just "plugins"). Rename is contained to desiredstate. Platform's `PluginRegistry` keeps its name and all its consumers stay untouched. Migration (not wrapping) ensures no remnant code drifts.
**Trade-offs:** Breaking changes to PluginHeader YAML format require graceful migration. PluginDiscovery removal touches PluginRuntimeBeans and Spring auto-config.
**Sources:** Platform PluginRegistry (yaml-plugin-api), existing PluginDiscovery, PluginRuntimeBeans, DesiredStatePluginAutoConfiguration
**Exploration:** quick
**Depends on:** D1 (unified model mandates cleanup), D3 (manifest structure defines migration target)
**Status:** captured

## D7: Conflict detection and dependency resolution

**Choice:** Strict fail-fast validation. The `PluginCatalog` validates the full dependency graph at load time:

Conflicts detected:
- Type collision — two plugins providing the same NodeType → fatal, second plugin rejected
- Coordinate collision — same group:name:version from different repositories → first repository wins (priority ordering in catalog config)
- Version conflict — same group:name with different versions loaded → only one version active per plugin
- Primitive collision — bundled primitive name clashes with an existing platform primitive → fatal

Dependencies resolved (topological order, fail-fast):
- Required primitives — checked against platform `PluginRegistry` after bundled primitives are registered. Missing → load fails with clear error
- Required types — checked against `ExtensionRegistry`. Missing → load fails. Circular dependencies detected and rejected
- Runtime version — compared against running desiredstate runtime version. Incompatible → load fails

Loading order: plugins sorted topologically by type dependencies. Plugins with no type dependencies load first; plugins that depend on types from other plugins load after their dependencies.
**Alternatives:**
- Lenient mode (warn and load anyway) — defers errors to runtime when provisioner fails mid-reconciliation. Strictly worse — fail-fast at load time gives a clear error message vs. an obscure runtime failure
**Rationale:** The whole point of the manifest's `requires` section is to fail fast. A plugin that declares a dependency and loads without it will fail at reconciliation time with a confusing error. Better to reject at load time with "plugin X requires primitive Y which is not available."
**Trade-offs:** Strict validation means an operator can't partially deploy (e.g., load a plugin before its dependency is available). They must deploy dependencies first or deploy the whole bundle atomically.
**Sources:** Existing `PluginValidator.validatePlugin()` (already does primitive validation at build time), `PluginValidationException`, `YamlPluginProcessor` duplicate detection
**Exploration:** quick
**Depends on:** D3 (manifest requires section defines what to validate), D4 (bundled primitives affect validation order), D6 (ExtensionRegistry is the unified validation target)
**Status:** captured

## D8: Module placement — two-layer split across platform and desiredstate

**Choice:** Split into generic catalog infrastructure (platform) and domain-specific activation (desiredstate). Platform implements the generic layer first; desiredstate consumes it.

**Platform** (new modules) — generic, domain-agnostic catalog infrastructure:
- `plugin-catalog-api/` — pure-Java SPI: `PluginCoordinates`, `PluginManifest` (interface), `PluginRequirements`, `PluginRepository` SPI, `PluginCatalog` (interface), `PluginEvent` (sealed), `ConflictException`
- `plugin-catalog-core/` — framework-neutral implementations: `FilesystemPluginRepository`, `DependencyResolver` (topological sort on provides/requires), `ConflictDetector` (provides-collision check), `DefaultPluginCatalog` (aggregates repositories, validates, tracks active plugins)

**Desiredstate** — domain-specific layer consuming platform's catalog:
- `DesiredStatePluginManifest` extends `PluginManifest` with provisioner/actual-state/fault-policy/CBR/RAS fields
- `DesiredStatePluginActivator` — wires loaded plugins into NodeProvisioner, ActualStateAdapter, FaultPolicy, etc.
- `ExtensionRegistry` — desiredstate's view of all active extensions (static classpath + dynamic catalog)
- Dynamic lifecycle management (watch/validate/activate cycle from D5)
- Migration of existing `PluginDiscovery`, `PluginRuntimeBeans`, Spring auto-config (from D6)

**Ops** — future consumer: YAML plugins for container/infra types, loaded via desiredstate's catalog

**Implementation sequence:** Design spec for platform's generic layer is produced as a platform issue from this brainstorming session. Platform implements. Then desiredstate builds on top.

**Alignment roadmap (future issues, not this scope):**
- Platform step primitives (`Definition`) gain optional version field, aligned with `PluginManifest` model
- Pages TS `CompositeCatalog` aligned with the same catalog concepts (coordinates, manifests) — TS-native transport (npm) but same metadata model
- Ops `InfraTypeDiscovery` and `ContainerNodeTypes` consolidated to use catalog-consistent patterns
**Alternatives:**
- Build in desiredstate, extract to platform later — avoids cross-repo coordination now but means building the same thing twice or refactoring on extraction
- Everything in desiredstate — simpler but creates fragmentation when pages or agency need catalogs; the generic concepts (coordinates, manifests, dependency resolution) are genuinely domain-agnostic
**Rationale:** The org-wide audit revealed the same pattern repeated across platform (4-source CompositePluginRegistry), pages (TS port of same), and desiredstate (YAML plugins). No repo has versioning or dependency resolution. The generic catalog abstractions (coordinates, manifests, repositories, dependency resolution, conflict detection) are domain-agnostic and belong in platform alongside the existing step primitive infrastructure. Domain-specific activation (what a loaded plugin DOES) belongs in each consuming repo. This follows the existing convention: platform owns `PluginRegistry` (step primitive SPI), desiredstate builds `DesiredStatePrimitiveRegistrar` on top of it.
**Trade-offs:** Requires cross-repo coordination (platform issue first, then desiredstate work). But the platform work is small and well-defined.
**Sources:** Org-wide audit: platform 15+ registries with no versioning, pages TS port of step infrastructure, ops two competing type discovery strategies, platform boundary-rules.md SPI convention
**Exploration:** deep-analysis (org-wide audit across platform, ops, pages)
**Depends on:** D1 (unified model), D2 (coordinates), D3 (manifest), D4 (repository/packaging)
**Status:** captured
