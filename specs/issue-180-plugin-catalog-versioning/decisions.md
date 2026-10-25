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

## D4: Repository model and filesystem layout

**Choice:** Maven-style directory layout for filesystem repositories — `plugins/<group-as-path>/<name>/<version>/plugin.yaml`. PluginRepository SPI abstracts over transport: `discover() → List<PluginManifest>`, `fetch(coordinates) → PluginModel`, `watch() → change events`. Filesystem implementation is first; git and HTTP implementations follow later via the same SPI.
**Alternatives:**
- Category-based layout (`plugins/<category>/<name>/<version>/`) — more browsable but loses namespace collision prevention; two third parties could collide on `monitoring/endpoint`
- Flat with manifest index (`plugins/catalog.yaml` + flat files) — simplest but doesn't scale, no structural collision prevention
**Rationale:** Group-as-path mirrors Maven local repository conventions, prevents collisions structurally, and the directory tree is itself a browsable catalog. The SPI returns PluginManifest objects regardless of internal layout, so git/HTTP repositories can use whatever structure suits their transport.
**Trade-offs:** Deeper directory nesting than category-based; operators must know the group to navigate the tree (but discovery/search tools can flatten this)
**Sources:** Maven local repository layout, existing `META-INF/desiredstate/plugins/` classpath convention, platform module-tier-structure
**Exploration:** quick
**Depends on:** D2 (coordinates define the directory structure)
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
