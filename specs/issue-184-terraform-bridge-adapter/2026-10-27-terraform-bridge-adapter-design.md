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
- Suspend/resume mapping
- GPG signature verification for registry downloads
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
| Registry client | `ProviderRegistryClient` | Download providers from registry.terraform.io, SHA256 verification, local cache |

### Build Dependencies

- `protobuf-maven-plugin` — generates Java stubs from `src/main/proto/tfplugin5.proto` and `tfplugin6.proto`
- `grpc-java` — gRPC runtime + channel management
- `msgpack-java` — msgpack decoding for v6 DynamicValue
- `bridge/api` — BridgeProvider, PlanCapable, BridgeDescriptor
- `plugin/api` — PluginSpecSchema (for schema translation)

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

3. **Connect** — Opens a gRPC `ManagedChannel` to the extracted address. No mTLS in
   Phase 1 (rare — only when provider opts in via client cert in handshake).

4. **Health** — gRPC health checking on the channel.

5. **Shutdown** — Sends `StopProvider` RPC, then kills the process. JVM shutdown hook
   for cleanup.

### ProviderProcessManager

Wraps `GoPluginHost` with lifecycle management:

- **Lazy start** — Provider process started on first operation, not at application startup
  (providers are heavyweight Go binaries)
- **Restart** — On failure, restarts per `RestartPolicy` from `BridgeLifecycleConfig`
  (on-failure with exponential backoff)
- **Connection sharing** — One provider process per binary, shared across tenants.
  Concurrent provision/deprovision calls share the same gRPC channel
- **Thread-safe** — All operations synchronized through the process manager

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
   - `aws_instance` → `NodeType.of("aws_instance")`
   - Terraform attribute → `PluginSpecSchema` field:
     - `required` → `required: true`
     - `string` → `type: STRING`, `number` → `type: NUMBER`, `bool` → `type: BOOLEAN`
     - `list`/`set` → `type: LIST` with constraints
     - Nested blocks → `type: MAP` with `valueType`
     - `computed`-only attributes (outputs) excluded from spec schema
   - Generates `BridgeTypeDescriptor` with uniform state mapping:
     - `ReadResource` returns state → `PRESENT`
     - `ReadResource` returns null → `ABSENT`
     - Read error → `UNKNOWN`

3. **`SchemaCache`** — Persists schemas to `~/.casehub/cache/schemas/{namespace}/{type}/{version}/schema.json`
   (configurable via `casehub.bridge.terraform.cache-dir`). On subsequent startups, loads from
   cache unless provider version changes.

4. **Dynamic `BridgeDescriptor`** — Built programmatically from discovered schema.
   `BridgeDiscovery.discoverAll()` wires into `NodeProvisioner` + `ActualStateAdapter`.

### Zero-Config User Experience

```yaml
desiredState:
  nodes:
    web-server:
      type: aws_instance
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

`readState(BridgeReadRequest)` → `TerraformProtocol.read()` with last-known state. Returns
`BridgeStateResult.Found` with current attributes, or `BridgeStateResult.NotFound` if gone.

### State Persistence

Terraform providers are stateless — they expect the caller to track resource state. Output
state from each apply is persisted via `ReconciliationStateStore`. `BridgeNodeSpec.fields()`
carries desired spec; actual state from Terraform is stored as provisioned output, available
via `BridgeRequest.previousState()` on subsequent operations.

### Error Mapping

Terraform diagnostics with severity `ERROR` → `BridgeResult.Failed`:

| Error pattern | BridgeErrorCategory |
|---------------|---------------------|
| Authentication/credential errors | `AUTHENTICATION` |
| Rate limiting / throttling | `TRANSIENT` |
| Timeout errors | `TRANSIENT` |
| Resource not found during apply | `PERMANENT` |
| Everything else | `PERMANENT` |

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

Versioned directory structure — no invalidation needed. `latest` re-checked every 24h
(configurable).

### Security

SHA256 verification of downloaded archives. GPG signature verification deferred to Phase 2.

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
- `plugin/api/` — PluginSpecSchema, PluginFieldDef (spec validation types)
- Issue #182 — bridge adapter architecture epic, design principle: target the protocol
- Issue #183 — bridge adapter SPI design (implemented)
- Issue #184 — this issue
- Terraform plugin protocol v5/v6 proto definitions
- HashiCorp go-plugin framework handshake specification
- Terraform registry API (registry.terraform.io/v1/)
