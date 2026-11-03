# Bridge Cloud Example — Design Spec

**Issue:** #208 — feat: bridge adapter teaching example — mock cloud-resource domain
**Branch:** issue-207-arc42-bridge-chapter
**Date:** 2026-11-03

## Overview

A self-contained teaching example at `examples/bridge-cloud/` demonstrating bridge adapter integration with the desired-state runtime. A mock cloud-infrastructure domain (VMs, networks, storage) uses `BridgeProvider` + `PlanCapable` + `ImportCapable`. Java-provisioned application deployment nodes coexist in the same graph, proving that bridge-managed and Java-managed nodes work together in a single reconciliation loop.

## Module

- **Path:** `examples/bridge-cloud/`
- **Artifact:** `casehub-desiredstate-example-bridge-cloud`
- **Package:** `io.casehub.desiredstate.example.bridge.cloud`

## Dependencies

| Dependency | Scope |
|-----------|-------|
| `casehub-desiredstate-api` | compile |
| `casehub-desiredstate-bridge-api` | compile |
| `casehub-desiredstate-runtime-core` | compile |
| `casehub-desiredstate-bridge` (runtime) | compile |
| `casehub-desiredstate-bridge-testing` | test |
| `casehub-desiredstate-testing` | test |
| JUnit5 + AssertJ | test |

No Quarkus or Spring dependencies — this is a pure Java example using runtime-core constructors directly.

## Resource Types

Three bridge-managed types simulating a cloud provider:

| Type ID | Description | State mapping |
|---------|------------|---------------|
| `cloud:vm` | Virtual machine | running→PRESENT, stopped→SUSPENDED, terminated→ABSENT |
| `cloud:network` | Virtual network/VPC | active→PRESENT, deleted→ABSENT |
| `cloud:storage` | Storage bucket | available→PRESENT, deleted→ABSENT |

One Java-provisioned type:

| Type | Description | NodeSpec |
|------|------------|---------|
| `app:deployment` | Application deployed onto a VM | `AppDeploymentSpec(appName, version, vmNodeId)` |

## Classes

### Bridge Provider

**`MockCloudProvider`** implements `BridgeProvider`, `PlanCapable`, `ImportCapable`

- In-memory `ConcurrentHashMap<String, Map<String, Object>>` keyed by `typeId::resourceId`
- `protocolId()` → `"mock-cloud"`
- `supportedTypes()` → `{"cloud:vm", "cloud:network", "cloud:storage"}`
- `apply(request)` → stores resource in map, returns `Success(outputState)` with generated `id`, `status`, `created_at`
- `destroy(request)` → removes from map, returns `Success`
- `readState(request)` → reads from map, returns `Found(state)` or `NotFound`
- `plan(request)` → compares request spec against stored state, returns `BridgePlan` with `CREATE`/`UPDATE`/no-change
- `importResource(typeId, importId, tenancyId)` → looks up by importId, returns `Success(state)` or `Failed`

Supports injectable fault simulation:
- `failNextApply(BridgeErrorCategory)` — next apply call returns `Failed` with given category
- `injectDrift(typeId, resourceId, field, value)` — modifies stored state to simulate drift

### Bridge Manifest

`src/main/resources/META-INF/desiredstate/bridges/mock-cloud.yaml`:

```yaml
bridge:
  protocol: mock-cloud
  types:
    - nodeType: "cloud:vm"
      stateMapping:
        running: PRESENT
        stopped: SUSPENDED
        terminated: ABSENT
        _notFound: ABSENT
        _default: UNKNOWN
      defaultErrorCategory: TRANSIENT
    - nodeType: "cloud:network"
      stateMapping:
        active: PRESENT
        deleted: ABSENT
        _notFound: ABSENT
        _default: UNKNOWN
      defaultErrorCategory: TRANSIENT
    - nodeType: "cloud:storage"
      stateMapping:
        available: PRESENT
        deleted: ABSENT
        _notFound: ABSENT
        _default: UNKNOWN
      defaultErrorCategory: TRANSIENT
```

### Goal Compiler

**`CloudGoalCompiler`** implements `GoalCompiler<CloudBlueprint>`

Compiles a `CloudBlueprint` into a mixed graph:

```
network-1 (cloud:network, bridge)
  ├── vm-web (cloud:vm, bridge, depends on network-1)
  │     └── app-web (app:deployment, Java, depends on vm-web)
  ├── vm-api (cloud:vm, bridge, depends on network-1)
  │     └── app-api (app:deployment, Java, depends on vm-api)
  └── storage-data (cloud:storage, bridge, independent)
```

Network nodes are roots. VMs depend on network. App deployments depend on VMs. Storage is independent (no dependencies).

### Blueprint

**`CloudBlueprint`** — builder-pattern goal declaration:

```java
CloudBlueprint.builder()
    .network("network-1", Map.of("cidr", "10.0.0.0/16"))
    .vm("vm-web", "network-1", Map.of("instance_type", "t3.micro", "ami", "ami-abc123"))
    .vm("vm-api", "network-1", Map.of("instance_type", "t3.small", "ami", "ami-abc123"))
    .app("app-web", "vm-web", "web-service", "1.0.0")
    .app("app-api", "vm-api", "api-service", "2.1.0")
    .storage("storage-data", Map.of("region", "us-east-1"))
    .build();
```

### Java Provisioner

**`AppDeploymentProvisioner`** implements `NodeProvisioner`

- `handledTypes()` → `{NodeType.of("app:deployment")}`
- `provision()` → records deployment in `CloudWorld`, returns `Success`
- `deprovision()` → removes deployment from `CloudWorld`, returns `Success`

### Simulation World

**`CloudWorld`** — in-memory state for Java-provisioned nodes (app deployments). The bridge provider has its own internal state for cloud resources.

### Fault Policy

**`VmFailureFaultPolicy`** — responds to `PROVISION_FAILED` on `cloud:vm` nodes by incrementing a retry count. After 3 retries, adds a replacement VM node with a dependency edge to the same network.

### Error Classifier

**`MockCloudErrorClassifier`** — maps error reasons to `BridgeErrorCategory`:
- "timeout", "connection refused" → TRANSIENT
- "invalid parameter", "resource conflict" → PERMANENT
- "invalid credentials" → AUTHENTICATION
- default → TRANSIENT

## Tests

### TCK Contract Tests

1. **`MockCloudProviderContractTest extends BridgeProviderContractTest`** — validates core SPI contract (protocolId, supportedTypes, apply/destroy/readState return non-null, apply idempotent)
2. **`MockCloudPlanContractTest extends PlanCapableContractTest`** — validates plan returns non-null with planId, applyPlan returns non-null
3. **`MockCloudImportContractTest extends ImportCapableContractTest`** — validates importResource returns non-null, success has state

### Integration Tests

4. **`CloudBridgeIntegrationTest`** — end-to-end reconciliation:
   - Compile blueprint → verify mixed graph structure (bridge + Java nodes)
   - Run reconciliation loop → verify all nodes provisioned (bridge via BridgeProvisioner, apps via AppDeploymentProvisioner)
   - Inject drift on a VM → verify drift detected and reconciled
   - Import existing cloud resource → verify BridgeOutputStateStore seeded
   - Inject VM failure → verify fault policy adds replacement node

## What This Example Does NOT Do

- No Quarkus or Spring wiring — pure Java with runtime-core constructors
- No real infrastructure — everything is in-memory
- No SuspendCapable — the issue says PlanCapable + ImportCapable only (#209 adds SuspendCapable separately)
- No visualizer — the existing examples already have visualizers; this example focuses on the bridge SPI pattern

## References

- Issue #208 — feature scope
- `bridge/api/` — BridgeProvider, PlanCapable, ImportCapable SPIs
- `bridge/runtime/` — BridgeProvisioner adapter pattern
- `bridge/testing/` — BridgeProviderContractTest, PlanCapableContractTest, ImportCapableContractTest
- `examples/dungeon/` — GoalCompiler + blueprint pattern reference
- `examples/pipeline/` — fault policy pattern reference
- `docs/guides/contributor-guide.md` §Bridge Adapter Architecture — "Implementing a New BridgeProvider" prose this example validates
