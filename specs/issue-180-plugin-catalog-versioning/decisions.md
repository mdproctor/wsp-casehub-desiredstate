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
