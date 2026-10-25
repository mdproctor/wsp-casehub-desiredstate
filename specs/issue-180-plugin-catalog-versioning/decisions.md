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
