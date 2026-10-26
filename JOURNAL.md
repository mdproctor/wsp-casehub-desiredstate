# Design Journal — issue-180-plugin-catalog-versioning

## 2026-10-25 — Design complete, plan written

**Session scope:** Full brainstorming and design for plugin catalog and versioning (#180).

**Key realization:** The original issue was scoped as M/Med — "design-heavy, implementation
likely straightforward." Through first-principles analysis across the entire CaseHub org
(platform, desiredstate, ops, pages), the scope grew significantly:

1. YAML plugins should be dynamic (no classpath, no restart) — this was the user's core insight
   that reframed the entire issue from "add metadata" to "build a distribution system"
2. The extension model should be unified across the platform — not random mechanisms at tension
3. Plugin packages can bundle custom step primitives — tying platform's PluginRegistry
   (step primitives) and desiredstate's ExtensionRegistry (node types) together
4. Generic catalog infrastructure belongs in platform; domain-specific activation in desiredstate

**Decisions (11, all reviewed):**
- D1: Catalog scoped to YAML plugins (GoalCompiler surfaces stay CDI-discovered)
- D2: Name + semver (plugin name = NodeType string, preserving identity coherence)
- D3: Evolved PluginHeader with requires section, no provides.types[]
- D4: Bundled directory packages at plugins/<type>/<version>/
- D5: Watch + validate + activate lifecycle with deactivation guard
- D6: ExtensionRegistry naming, PluginDiscovery evolves (not removed)
- D7: Dual validation — strict for classpath, retry-with-timeout for filesystem
- D8: Platform owns generic catalog SPI, desiredstate owns domain activation
- D9: Single version per plugin (no multi-version)
- D10: Platform PluginRegistry needs unregister() + replacement semantics
- D11: PluginManifest as both file format and runtime model for v1

**Post-spec review findings (key):**
- Immutable routing tables (Map.copyOf) incompatible with dynamic plugins → ConcurrentHashMap routers
- CDI bean registration impossible at runtime in Quarkus → activator bypasses CDI entirely
- FaultPolicyEngine/DriftPolicyEngine need interface extraction for dynamic alternatives
- Deactivation guard uses ReconciliationStateStore.allTenancyIds() (new method)

**Platform issue filed:** casehubio/platform#662 — generic catalog SPI + filesystem repository.
Blocks batches 4-5 of the implementation plan. Batches 1-3 are unblocked.

**Actual scale:** L/High across two repos. Not the M/Med the issue originally estimated.
