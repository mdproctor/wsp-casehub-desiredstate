# Decisions — Brownfield Discovery (#221)

## D1: DiscoveryCapable SPI shape

**Choice:** Flat per-type — `discover(typeId, tenancyId) → List<DiscoveredResource>`
**Alternatives:**
- Batched multi-type — single call for multiple types; pushes orchestration into provider, not needed at this scale
- Streaming (Mutiny Multi) — best for massive environments but adds Mutiny dependency to bridge-api which is framework-neutral
**Rationale:** Matches the existing per-type granularity of `ImportCapable`. The caller (DiscoveryAdapter or orchestrator) can parallelize calls across types. For practical brownfield scenarios (tens to hundreds of resources), per-type enumeration is sufficient.
**Trade-offs:** Can't batch cross-type API calls within a single provider call. Acceptable — optimization can be added later as a separate mixin.
**Sources:** `bridge/api/ImportCapable.java`, `bridge/api/BridgeProvider.java`
**Exploration:** quick
**Status:** captured

## D2: DiscoveryAdapter SPI and DiscoveredResource model

**Choice:** Symmetric with ActualStateAdapter — separate `DiscoveryAdapter` with `handledTypes()` + `discover(Set<NodeType>, String tenancyId)`. Returns fully hydrated `DiscoveredResource(resourceId, type, state, spec)`.
**Alternatives:**
- Merged into ActualStateAdapter — add default `discover()` method; muddies the contract since reading known state vs. enumerating unknown resources are different operations
- Minimal ID-only — returns just resource IDs, requires a second `ImportCapable` pass to hydrate; thinnest SPI but forces two-pass discovery
**Rationale:** Clean separation of concerns. `state` carries raw cloud state (computed fields, IDs); `spec` carries the configurable fields that belong in YAML. The bridge implementation translates between them using schema knowledge already in `SchemaTranslator`.
**Trade-offs:** New SPI + router adds surface area. Acceptable — the alternative (muddying ActualStateAdapter) is worse.
**Sources:** `api/ActualStateAdapter.java`, `bridge/terraform/SchemaTranslator.java`, `bridge/terraform/DiscoveredType.java`
**Exploration:** quick
**Depends on:** D1 (DiscoveryCapable provides the data that DiscoveryAdapter wraps)
**Status:** captured

## D3: YAML serializer location

**Choice:** `yaml/runtime` — standalone `YamlGraphSerializer` utility class
**Alternatives:**
- New `discovery/` module — more isolation but adds another module to an already large tree
- `runtime-core/` — keeps pipeline together but couples core to YAML model types
**Rationale:** YAML serialization is a YAML surface concern. The serializer converts `DiscoveredResource` → `YamlGraph`/`YamlNode` model and writes YAML text via Jackson. Framework-neutral utility, reusable by any discovery source. Returns string content; caller decides where to write it.
**Trade-offs:** `yaml/runtime` depends on `api/` but not on `runtime-core/`. The orchestrator in `runtime-core/` cannot directly call the serializer — the caller (CLI/CDI layer) composes both. This is correct layering.
**Sources:** `yaml/runtime/model/YamlGraph.java`, `yaml/runtime/model/YamlNode.java`
**Exploration:** quick
**Depends on:** D2 (serializer consumes DiscoveredResource)
**Status:** captured

## D4: Discovery orchestrator location

**Choice:** `runtime-core/` — `DiscoveryOrchestrator` alongside `ReconciliationLoop` and `TransitionPlanner`
**Alternatives:**
- New `discovery/` module — cleanest isolation but adds a new module
- `bridge/runtime` — co-locates with BridgeDiscovery but limits non-bridge discovery sources
**Rationale:** The orchestrator composes runtime SPIs: `DiscoveryAdapterRouter`, `DesiredStateGraphFactory`, `ReconciliationStateStore`. It's a runtime composition concern, not a surface concern. Returns `DiscoveryResult` (graph + resources) without YAML dependency — YAML serialization happens upstream in the calling layer.
**Trade-offs:** `runtime-core/` grows slightly. Acceptable — it's the natural home for graph-building orchestration.
**Sources:** `runtime-core/ReconciliationLoop.java`, `runtime-core/TransitionPlanner.java`
**Exploration:** quick
**Depends on:** D2 (orchestrator consumes DiscoveryAdapter), D3 (serializer is a separate downstream step)
**Status:** captured
