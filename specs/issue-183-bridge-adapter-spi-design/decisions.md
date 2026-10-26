# Bridge Adapter SPI — Design Decisions

## D1: SPI architecture — bridge-as-SPI-layer

**Choice:** New `BridgeProvider` SPI sits between external providers and existing CaseHub SPIs. Bridge runtime implements `NodeProvisioner` + `ActualStateAdapter` and dispatches to `BridgeProvider` implementations. Mirrors the YamlPluginProvisioner pattern (single adapter, descriptor-based routing).
**Alternatives:**
- Direct SPI implementation — each provider implements NodeProvisioner directly. Loses manifest-driven config, forces lifecycle reimplementation per provider.
- Extend YAML plugin surface — add protocol-specific step primitives. Provider protocols (Terraform plan-apply, Ansible playbook execution) are too complex for the step model.
**Rationale:** Clean separation of concerns. Bridge runtime handles lifecycle, health, retry. Provider implementations are thin protocol translators. Existing router, fault policy, drift policy work unchanged. Proven pattern from YAML plugins.
**Trade-offs:** One more abstraction layer. Provider authors learn BridgeProvider SPI + manifest format.
**Sources:** NodeProvisioner SPI (api/), YamlPluginProvisioner (plugin/runtime/), PluginDiscovery pattern, #131 plugin architecture, #182 bridge epic
**Exploration:** quick
**Status:** captured

## D2: Deployment model — design-agnostic SPI

**Choice:** SPI abstracts over deployment model. Transport is pluggable — colocated (in-process), sidecar (local socket), or remote (network) are all valid implementations of the same `BridgeConnector` interface.
**Alternatives:**
- Colocated only — simplest but locks out sidecar/remote deployments.
- Sidecar only — good isolation but unnecessary complexity for simple providers.
- Remote only — full isolation but adds network failure modes everywhere.
**Rationale:** Different providers and deployment environments have different needs. Terraform may run colocated; a heavy Ansible controller may run as a sidecar. The SPI should not prescribe this.
**Trade-offs:** More complex SPI surface. Implementations must handle transport-specific concerns (connection pooling, process lifecycle) behind the abstraction.
**Sources:** #131 (evaluated all three models), #182 design principle
**Exploration:** quick
**Status:** captured

## D3: Type discovery — static manifest

**Choice:** Bridge declares types in a static YAML manifest at build/deploy time. NodeProvisionerRouter sees them at construction. No runtime schema negotiation.
**Alternatives:**
- Dynamic at startup — bridge queries provider for schema. Self-describing but adds startup latency and failure modes.
- Hybrid (manifest + validation) — static manifest with optional provider schema validation. More robust but more complex.
**Rationale:** Consistent with YAML plugins at `META-INF/desiredstate/plugins/*.yaml`. Fast startup, no provider dependency at construction time. Provider schema validation can be an optional build-time tool, not a runtime requirement.
**Trade-offs:** Manifest must be maintained when provider adds new types. No automatic discovery of new resource types.
**Sources:** PluginDiscovery.discoverAll() pattern, YAML plugin deployment extension, @NodeTypeId registry
**Exploration:** quick
**Status:** captured

## D4: State mapping — manifest-declared

**Choice:** Bridge manifest declares state mapping rules: which provider states map to PRESENT, ABSENT, DEGRADED, UNKNOWN, SUSPENDED. Generic bridge runtime applies the rules.
**Alternatives:**
- Code-implemented — each bridge implements StateMapper in Java. Maximum flexibility but requires Java for every provider.
- Convention with overrides — default convention (exists=PRESENT, not-found=ABSENT) with manifest overrides. Less config but implicit.
**Rationale:** Keeps bridge adapters declarative. Operators can customize state mapping per type without Java. Consistent with the manifest-driven philosophy.
**Trade-offs:** Complex state mappings may be hard to express declaratively. May need escape hatch for code-backed mapping.
**Sources:** ActualStateAdapter SPI, NodeStatus enum, YamlPluginActualStateAdapter
**Exploration:** quick
**Status:** captured
