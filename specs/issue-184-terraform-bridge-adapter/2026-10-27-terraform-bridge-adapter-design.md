# Terraform/OpenTofu Bridge Adapter — Design Spec

**Issue:** #184
**Parent:** #182 (bridge adapter architecture, Tier 3)
**Depends on:** #183 (bridge adapter SPI — implemented)
**Phase:** 1 of 2

## Summary

A new module `bridge/terraform/` implements a `BridgeProvider` for the Terraform/OpenTofu
plugin protocol. It speaks the go-plugin gRPC protocol directly from Java — no Go shim,
no CLI wrapper. Supports both protocol v5 and v6.

The key differentiator is **zero-config provider adoption**: CaseHub auto-discovers resource
types and schemas from any Terraform provider via `GetProviderSchema`, generating `NodeType`
and `PluginSpecSchema` entries at runtime. No manifests, no code — point at a provider binary
(or declare `hashicorp/aws`) and every resource type is immediately available with full spec
validation.

### Integration Approach — Alternatives Considered

**Direct Java host (chosen):** Java replicates the go-plugin host-side behavior — stdout
handshake parse, gRPC channel connect, provider lifecycle management. The go-plugin host
protocol is a single stdout line (`core|app|network|address|protocol`), a magic cookie env
var, and standard gRPC. The provider side (signal handling, heartbeats, goroutine management)
runs unmodified in the provider binary. Java owns the host side only.

**Go shim proxy (rejected):** A ~100–200 line Go binary embedding the official `go-plugin`
host library, exposing a clean gRPC API to Java. Benefits: auto-mTLS handled by go-plugin
library, protocol negotiation absorbed by Go dependency updates. Rejected because: introduces
a second managed process per provider (Java → shim-gRPC → shim → go-plugin → provider-gRPC
→ provider), requires building/versioning/distributing a Go binary alongside the Java
application, and couples provider lifecycle to two process boundaries instead of one. The
shim absorbs protocol edge cases but adds operational complexity — health checking, restart
coordination, and signal forwarding across two process boundaries.

**Terraform CLI wrapper (rejected):** Shell out to `terraform plan/apply` for operations.
Rejected because: Terraform CLI operates on entire configuration directories with its own
state file management. It has no per-resource operation API — individual resource lifecycle
(provision one `aws_instance`, read its state, deprovision it) requires synthetic Terraform
configurations per operation. The process-per-operation overhead and loss of gRPC streaming
make this unsuitable for per-resource reconciliation in a continuous loop.

**Pulumi Terraform Bridge (rejected):** Pulumi's `pulumi-terraform-bridge` wraps any
Terraform provider as a Pulumi resource with Java SDK support. Rejected because: introduces
the full Pulumi SDK, its state management model (Pulumi state alongside CaseHub's
`ReconciliationStateStore`), and coupling to Pulumi's release cycle. The abstraction layer
adds no value — CaseHub already has reconciliation, planning, and state management. The
bridge exists to access provider binaries, not to adopt another orchestration framework.

### Positioning

CaseHub offers a spectrum of infrastructure authorship no other platform provides:

1. **Zero-code (Terraform bridge)** — Point at a provider, auto-discover types. Instant adoption.
2. **Low-code (YAML plugins)** — Wrap or extend resources with approval gates, validation steps,
   custom fault policies. Step pipelines compose with the bridge underneath.
3. **Full-code (Java plugins)** — Implement SPIs directly for complete control.

All three tiers land in the same reconciliation loop with dependency-aware ordering, drift
detection, fault escalation, CBR, pending approval, lifecycle management, and cross-domain
composition. Terraform alone gives plan/apply. CaseHub gives continuous reconciliation.

## Phase 1 Scope

**In scope:**
- Provider binary lifecycle (go-plugin handshake, process management)
- Schema auto-discovery → NodeType + PluginSpecSchema
- Plan/Apply → provision/deprovision
- State read → ActualState
- Provider registry auto-download + binary path override
- Three-tier test harness

**Phase 2 (follow-up issues):**
- Import support (adopt existing resources)
- Credential resolution via CaseHub's CredentialResolver
- Per-configuration provider process pooling (multi-tenant credential isolation)
- Auto-mTLS implementation (required before production deployment)
- Suspend/resume mapping
- GPG signature verification for registry downloads (required before production deployment)
- `sensitive` and `deprecated` attribute flags in `PluginFieldDef`
- Cloud provider integration tests

## Module Structure

**Module:** `bridge/terraform/`
**Artifact:** `casehub-desiredstate-bridge-terraform`
**Root package:** `io.casehub.desiredstate.bridge.terraform`

### Components

| Component | Class | Responsibility |
|-----------|-------|----------------|
| Go-plugin host | `GoPluginHost` | Start provider binary, parse handshake, establish gRPC channel |
| Process manager | `ProviderProcessManager` | Lazy start, restart on failure, shutdown, connection pooling |
| Protocol interface | `TerraformProtocol` | Version-agnostic operations (getSchema, configure, plan, apply, read) |
| Protocol v5 | `ProtocolV5Adapter` | v5 proto stub translation, JSON DynamicValue encoding |
| Protocol v6 | `ProtocolV6Adapter` | v6 proto stub translation, msgpack DynamicValue encoding |
| Protocol factory | `TerraformProtocolFactory` | Handshake version → adapter selection (prefers v6) |
| Cty decoder | `CtyDecoder` | Terraform cty type system → Java types (string, number, bool, list, set, map, object) |
| Schema translator | `SchemaTranslator` | GetProviderSchema → NodeType + PluginSpecSchema, excludes computed-only attributes |
| Schema cache | `SchemaCache` | Persist discovered schemas to disk, reload on startup, invalidate on version change |
| Bridge provider | `TerraformBridgeProvider` | `BridgeProvider` + `PlanCapable` implementation |
| State store | `BridgeOutputStateStore` | Per-resource Terraform output state persistence (resource IDs, ARNs, computed attributes) |
| Registry client | `ProviderRegistryClient` | Download providers from registry.terraform.io, SHA256 verification, local cache |

### Build Dependencies

- `protobuf-maven-plugin` — generates Java stubs from `src/main/proto/tfplugin5.proto` and `tfplugin6.proto`
- `grpc-java` — gRPC runtime + channel management
- `msgpack-java` — msgpack decoding for v6 DynamicValue
- `bridge/api` — BridgeProvider, PlanCapable, BridgeDescriptor
- `plugin/api` — PluginSpecSchema, PluginFieldDef (extracted from `plugin/runtime/` to
  `plugin/api/` — these are pure data records with no runtime dependencies)

## Go-Plugin Handshake and Provider Lifecycle

### GoPluginHost

Handles the go-plugin protocol's host side:

1. **Start** — Launches the provider binary as a subprocess. Sets the magic cookie
   environment variable (`PLUGIN_MAGIC_COOKIE=d602bf8f470bc67ca7faa0386276bbdd4330ea6918e54b83e925f0f7`).
   The provider refuses to start without it.

2. **Handshake parse** — Reads the first line from stdout:
   `<core_protocol_version>|<app_protocol_version>|<network_type>|<address>|<protocol>`.
   Extracts:
   - `app_protocol_version` → determines v5 or v6 (prefers v6 when both advertised)
   - `address` → `host:port` for gRPC connection
   - `protocol` → must be `grpc` (error if `netrpc`)

3. **Connect** — Opens a gRPC `ManagedChannel` to the extracted address. Phase 1 omits
   auto-mTLS, requiring plaintext localhost gRPC. Since go-plugin v5 (~2022), auto-mTLS
   is the default: the host generates a temporary CA, creates certificates, and passes
   them via `PLUGIN_CLIENT_CERT` env var. When launched by a non-Go host without this
   env var, providers fall back to plaintext. This is a deliberate security regression
   from Terraform CLI's default behavior. Phase 2 must implement auto-mTLS (CA generation,
   cert exchange via handshake) before production deployment.

4. **Health** — gRPC health checking on the channel.

5. **Shutdown** — Sends `StopProvider` RPC, then kills the process. JVM shutdown hook
   for cleanup.

### ProviderProcessManager

Wraps `GoPluginHost` with lifecycle management:

- **Lazy start** — Provider process started on first operation, not at application startup
  (providers are heavyweight Go binaries)
- **Restart** — On failure, restarts per `RestartPolicy` from `BridgeLifecycleConfig`
  (on-failure with exponential backoff)
- **Connection sharing** — Phase 1: one provider process per binary, single configuration
  (Phase 1 uses credential-free providers). Phase 2: one process per (binary, configuration
  hash) for multi-tenant credential isolation — Terraform providers hold credentials in
  memory after `ConfigureProvider`, so distinct tenant configurations require distinct
  provider processes
- **Concurrent operations** — Provider process lifecycle (start, stop, restart, health check)
  is synchronized through the process manager. Individual gRPC calls (plan, apply, read)
  are dispatched concurrently on the shared `ManagedChannel` — gRPC channels are multiplexed
  and Terraform providers handle concurrent requests via goroutines

## Protocol Adapters (v5/v6)

### TerraformProtocol Interface

```java
interface TerraformProtocol {
    ProviderSchema getSchema();
    void configure(Map<String, Object> config);
    PlanResponse plan(String resourceType, Map<String, Object> priorState,
                      Map<String, Object> proposedState);
    ApplyResponse apply(String resourceType, Map<String, Object> priorState,
                        Object plannedChange);
    ReadResponse read(String resourceType, Map<String, Object> currentState);
}
```

### Version Differences Absorbed

| Concern | v5 | v6 |
|---------|----|----|
| Schema | `Schema.Block` + `SchemaAttribute` | `Schema.Object` + `Schema.Attribute` with `type` as msgpack cty |
| State encoding | JSON-encoded `DynamicValue` | msgpack-encoded `DynamicValue` |
| Diagnostics | Same structure | Same structure |

`TerraformProtocolFactory` inspects the handshake `app_protocol_version` and instantiates
the right adapter. If a provider advertises both (`5|6`), picks v6.

### CtyDecoder

Handles Terraform's `cty` type system — decodes msgpack/JSON-encoded values to Java types:

| cty type | Java type |
|----------|-----------|
| `string` | `String` |
| `number` | `BigDecimal` |
| `bool` | `Boolean` |
| `list(T)` | `List<Object>` |
| `set(T)` | `Set<Object>` |
| `map(T)` | `Map<String, Object>` |
| `object({...})` | `Map<String, Object>` |
| `tuple([...])` | `List<Object>` |
| `dynamic` | decoded at runtime |
| `null` | `null` |

The type system is finite and well-specified — no plugin extensibility.

## Schema Discovery and Zero-Config Type Registration

On first access to a provider:

1. **`GetProviderSchema`** — Returns every resource type with full attribute schemas.

2. **`SchemaTranslator`** converts each resource type:
   - `aws_instance` → `NodeType.of("tf:hashicorp/aws:aws_instance")` — namespaced as
     `tf:{namespace}/{type}:{resource}` to prevent collision with native CaseHub types,
     other providers defining the same resource type, or future bridges (Pulumi, Crossplane)
   - Terraform attribute → `PluginSpecSchema` field:
     - `required` → `required: true`
     - `string` → `type: STRING`, `number` → `type: NUMBER`, `bool` → `type: BOOLEAN`
     - `list`/`set` → `type: LIST` with constraints
     - Nested blocks → `type: MAP` with `valueType`
     - `computed`-only attributes (outputs) excluded from spec schema
   - Generates `BridgeTypeDescriptor` with state mapping:
     - `ReadResource` returns state and plan shows no changes → `PRESENT`
     - `ReadResource` returns state and plan shows changes → `DRIFTED`
     - `ReadResource` returns null → `ABSENT`
     - Read error → `UNKNOWN`

   **Schema translation limitations** (known lossy mappings):
   - Terraform block nesting modes (NestingSingle, NestingList, NestingSet, NestingMap) are
     collapsed to `MAP` — ordering and cardinality semantics are lost
   - `sensitive` attribute flag is not propagated — CaseHub won't mask these fields in logs
     or diagnostics (Phase 2: add `sensitive` to `PluginFieldDef`)
   - `deprecated` attribute flag is not propagated (Phase 2: add `deprecated` to
     `PluginFieldDef`)
   - Recursive nesting beyond one level is flattened — `PluginFieldDef.valueType` is a
     string, not a recursive structure. Deep nesting (`instance.network_interface[0].ipv6_addresses`)
     is accepted without structural validation

3. **`SchemaCache`** — Persists schemas to `~/.casehub/cache/schemas/{namespace}/{type}/{version}/schema.json`
   (configurable via `casehub.bridge.terraform.cache-dir`). On subsequent startups, loads from
   cache unless provider version changes.

4. **Dynamic `BridgeDescriptor`** — Built programmatically from discovered schema.
   `BridgeDiscovery.discoverAll()` returns `List<BeanRegistration>` containing
   `BridgeProvisioner` (as `NodeProvisioner`) and `BridgeActualStateAdapter` (as
   `ActualStateAdapter`) instances. These are registered at runtime via
   `DynamicNodeProvisionerRouter.register()` and `DynamicActualStateAdapterRouter.register()`.
   This is runtime registration — types are discovered after application start when the
   provider binary is first accessed, not at Quarkus build time. The dynamic routers use
   `ConcurrentHashMap` and support `register()`/`unregister()` for hot-loading.

### Zero-Config User Experience

```yaml
desiredState:
  nodes:
    web-server:
      type: tf:hashicorp/aws:aws_instance
      spec:
        ami: ami-0abcdef1234567890
        instance_type: t3.micro
        tags:
          Name: web-server
```

No manifest, no type declaration. CaseHub validates the spec against the auto-discovered
schema, plans, and reconciles.

## Bridge SPI Integration

`TerraformBridgeProvider` implements `BridgeProvider` + `PlanCapable`.

### Provision (plan + apply)

1. `plan(BridgeRequest)` → `TerraformProtocol.plan()` with empty prior state + spec as
   proposed. Returns `BridgePlan` with planned changes and raw plan.
2. `applyPlan(BridgePlan)` → `TerraformProtocol.apply()` with raw plan. Returns
   `BridgeResult.Success` with output state (computed attributes: IDs, ARNs, IPs).
3. Direct `apply(BridgeRequest)` → plan then apply as single operation.

### Deprovision

`destroy(BridgeRequest)` → `TerraformProtocol.plan()` with null proposed state (destroy
signal), then `apply()` with destroy plan.

### Read State

`readState(BridgeReadRequest)` → `TerraformProtocol.read()` with last-known state from
`BridgeOutputStateStore`. `BridgeReadRequest` is extended with `previousState` and
`desiredSpec` fields. The Terraform bridge:

1. Loads previous state from `BridgeOutputStateStore`
2. Calls `ReadResource` with previous state (Terraform needs the resource ID to read)
3. If resource exists, calls `PlanResourceChange` comparing actual state against desired
   spec to detect attribute-level drift
4. Returns `BridgeStateResult.Found` with a synthetic `status` field: `"present"` if plan
   shows no changes, `"drifted"` if plan shows changes
5. Returns `BridgeStateResult.NotFound` if resource is gone

### State Persistence

Terraform providers are stateless — they expect the caller to track resource state between
operations. A new `BridgeOutputStateStore` SPI persists per-resource output state:

```java
interface BridgeOutputStateStore {
    void store(String tenancyId, String nodeId, Map<String, Object> outputState);
    Optional<Map<String, Object>> load(String tenancyId, String nodeId);
    void remove(String tenancyId, String nodeId);
}
```

On `BridgeResult.Success(outputState)`, `BridgeProvisioner.provision()` stores the output
state via `BridgeOutputStateStore.store()`. On subsequent operations,
`BridgeProvisioner.toBridgeRequest()` loads prior state via `BridgeOutputStateStore.load()`
and populates `BridgeRequest.previousState()`. On deprovision success,
`BridgeProvisioner.deprovision()` removes the stored state.

This is distinct from `ReconciliationStateStore` (which stores desired-state graphs, not
per-resource Terraform state). The `BridgeOutputStateStore` stores Terraform-specific
output: resource IDs, ARNs, computed attributes — everything needed for subsequent
plan/apply/read/destroy operations.

Default implementation: file-based at `~/.casehub/state/{tenancyId}/{nodeId}/output.json`
(configurable). JPA implementation for persistence-jpa deployments.

### Error Mapping

Terraform diagnostics with severity `ERROR` → `BridgeResult.Failed`:

| Error pattern | BridgeErrorCategory |
|---------------|---------------------|
| Authentication/credential errors | `AUTHENTICATION` |
| Rate limiting / throttling | `TRANSIENT` |
| Timeout errors | `TRANSIENT` |
| Resource not found during apply | `PERMANENT` |
| Validation / invalid configuration | `PERMANENT` |
| Everything else | `TRANSIENT` |

The default is `TRANSIENT` (retry with budget) rather than `PERMANENT` (never retry).
For infrastructure operations, most unrecognized errors are retriable (network timeouts,
cloud API outages, eventual consistency delays). If genuinely permanent, repeated retries
hit the retry budget and the `FaultPolicyEngine` escalates. Pattern matching is inherently
fragile across providers (AWS says "AuthFailure", GCP says "PERMISSION_DENIED"), so the
safe default is to retry and let the fault policy engine handle escalation.

## Provider Registry Client

### Discovery Flow

1. User declares provider: `hashicorp/aws` at version `5.60.0` (or `latest`)
2. `GET /v1/providers/{namespace}/{type}/versions` → resolve `latest`
3. `GET /v1/providers/{namespace}/{type}/{version}/download/{os}/{arch}` → download URL + SHA256
4. Download zip, verify SHA256, extract binary
5. Cache at `~/.casehub/providers/{namespace}/{type}/{version}/{os}_{arch}/`

### Binary Path Override

When `binaryPath` set in configuration → skip registry, use local binary directly.

### Platform Detection

Auto-detect OS (`linux`/`darwin`/`windows`) and architecture (`amd64`/`arm64`). Fail fast
if provider doesn't publish for current platform.

### Cache

Versioned directory structure — no invalidation needed. Explicit version pinning is the
default (`hashicorp/aws` at `5.60.0`). `latest` resolution is opt-in for development
(re-checked every 24h, configurable). Schema compatibility is validated on version change:
`SchemaCache` compares the new schema against the cached schema and logs warnings for
removed/renamed attributes that could affect existing resources.

### Security

SHA256 verification of downloaded archives. GPG signature verification deferred to Phase 2
but required before production deployment with public registry downloads — SHA256 confirms
integrity but not authenticity.

## Testing Strategy

### Tier 1 — Protocol Mock (unit tests)

Java gRPC server implementing the Terraform plugin proto, started in-process.

**Tests:**
- Go-plugin handshake parsing — valid lines, malformed lines, version negotiation
- v5 and v6 protocol adapter correctness — schema translation, DynamicValue encoding/decoding
- `CtyDecoder` — all type combinations (string, number, bool, list, set, map, object, nested)
- Error classification — diagnostic severity → `BridgeErrorCategory`
- `SchemaTranslator` — Terraform schema → `PluginSpecSchema`, computed-only exclusion
- Provider process failure — crash during handshake, crash mid-operation, timeout

Fast, no external binaries, runs on every `mvn install`.

### Tier 2 — Real Providers (integration tests)

Download real provider binaries, exercise the full stack.

**`hashicorp/null`:**
- Proves go-plugin handshake and gRPC protocol with a real Go binary
- Schema discovery, plan, apply, read, destroy

**`hashicorp/random`:**
- Proves the zero-config lifecycle claim
- Start with no manifest, auto-discover schemas
- Provision `random_pet` → read state (PRESENT) → deprovision → read (ABSENT)
- Drift detection: modify spec → re-plan → apply → verify convergence
- `PluginSpecSchema` validation: invalid spec → rejection before reaching provider

Gated behind Maven profile `-Pintegration`. Provider binaries downloaded once and cached in CI.

### Tier 3 — Cloud Providers (Phase 2)

Real AWS or GCP provider against actual infrastructure. Gated behind `-Pcloud-integration`
with credential environment variables. Deferred until credential resolution lands.

## References

- `bridge/api/` — BridgeProvider, PlanCapable, BridgeDescriptor, BridgeOperation SPI
- `bridge/runtime/` — BridgeProvisioner, BridgeActualStateAdapter, BridgeDiscovery, BridgeManifestParser
- `plugin/api/` — PluginSpecSchema, PluginFieldDef (spec validation types — to be extracted
  from `plugin/runtime/` as a prerequisite for this work)
- Issue #182 — bridge adapter architecture epic, design principle: target the protocol
- Issue #183 — bridge adapter SPI design (implemented)
- Issue #184 — this issue
- Terraform plugin protocol v5/v6 proto definitions
- HashiCorp go-plugin framework handshake specification
- Terraform registry API (registry.terraform.io/v1/)
