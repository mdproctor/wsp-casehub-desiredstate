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

## D5: BridgeProvider contract — operation-oriented with optional plan

**Choice:** Base SPI has `apply(BridgeRequest) → BridgeResult`, `destroy(BridgeRequest) → BridgeResult`, `readState(BridgeReadRequest) → BridgeStateResult`. Providers that support plan-apply cycles implement an additional `PlanCapable` mixin interface with `plan()` + `applyPlan()`.
**Alternatives:**
- Plan-then-execute for all — forces two-phase on simple providers that don't need it.
- Message-based (single handle method) — loses type safety and discoverability for maximum extensibility.
**Rationale:** Simple default for the common case. Terraform/Ansible implement PlanCapable for their native plan/check modes. Simple CLI-based providers just implement apply/destroy. Bridge runtime detects PlanCapable and uses it when available.
**Trade-offs:** Two interfaces to understand for providers with plan support. PlanCapable is optional but important — risk of providers skipping it when they should implement it.
**Depends on:** D1 (bridge-as-SPI-layer)
**Sources:** NodeProvisioner SPI contract, Terraform plan-apply workflow, Ansible check mode
**Exploration:** quick
**Status:** captured

## D6: Module structure — top-level bridge/ module

**Choice:** New top-level `bridge/` module parallel to `plugin/`. Sub-modules: `bridge/api/` (BridgeProvider SPI, request/response types), `bridge/runtime/` (BridgeProvisioner, BridgeActualStateAdapter, lifecycle management), `bridge/deployment/` (Quarkus build extension), `bridge/spring/` (Spring auto-config), `bridge/testing/` (test framework). Concrete providers in separate modules: `bridge/terraform/`, `bridge/ansible/`.
**Alternatives:**
- Under plugin/ — bridges have fundamentally different dependencies (gRPC, process mgmt) from YAML step pipelines. Muddles plugin module concerns.
- Separate repo — overkill while SPI is still evolving. Cross-repo SPI changes would slow iteration.
**Rationale:** Bridge is a distinct tier with distinct dependencies. Same organizational pattern as plugin/, annotations/, yaml/, ts-dsl/. Concrete providers have their own dependency trees (gRPC for Terraform).
**Trade-offs:** More modules in an already large repo. bridge/api/ must be kept minimal to avoid pulling bridge dependencies into the core.
**Depends on:** D1 (bridge-as-SPI-layer)
**Sources:** plugin/ module structure, annotations/ module structure, #131 tier model
**Exploration:** quick
**Status:** captured

## D7: Provider lifecycle — runtime-managed

**Choice:** Bridge runtime manages external provider processes via manifest descriptors. Manifest declares: binary path, health check config, startup args, restart policy, or external service endpoint. Runtime handles startup, health monitoring, restart on crash, graceful shutdown. BridgeProvider implementations receive a ready connection — they never manage processes. Two modes: "managed process" (runtime starts/stops binary) and "external service" (runtime manages connection to pre-existing endpoint).
**Alternatives:**
- Provider-managed — BridgeProvider SPI includes start/stop/isHealthy. Full control but every provider reimplements lifecycle.
- Pluggable lifecycle strategy — manifest selects strategy type. Extensible but more moving parts without clear benefit over runtime-managed with modes.
**Rationale:** Keeps provider implementations thin (just protocol translation). Consistent lifecycle management across all bridges. Centralized health monitoring.
**Trade-offs:** Manifest must be flexible enough for different lifecycle models. Unusual providers may need escape hatch to code-backed lifecycle.
**Depends on:** D2 (design-agnostic deployment)
**Sources:** #131 sidecar evaluation, Terraform go-plugin process model
**Exploration:** quick
**Status:** captured
