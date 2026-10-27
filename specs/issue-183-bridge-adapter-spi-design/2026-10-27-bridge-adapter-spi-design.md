# Bridge Adapter SPI and Execution Model

**Issue:** #183 (part of #182 — bridge adapter architecture, Tier 3)
**Date:** 2026-10-27
**Status:** Draft

## Problem

CaseHub's desired-state runtime manages node graphs through `NodeProvisioner` and `ActualStateAdapter`
SPIs. Tier 1 (Java plugins) and Tier 2 (YAML plugins, #173) require provisioning logic to be written
in Java or declared in YAML step pipelines. External infrastructure tools — Terraform/OpenTofu
providers (~3,000 plugins covering virtually every cloud and SaaS API) and Ansible collections
(thousands of modules) — already contain extensive provisioning knowledge. Rewriting this in
Java or YAML is impractical and unnecessary.

Tier 3 bridge adapters wrap external provider protocols as CaseHub plugins, translating between
CaseHub's SPI contract and the external tool's native protocol. The reconciliation loop sees
standard `NodeProvisioner` and `ActualStateAdapter` implementations — it doesn't know or care
that an external tool is doing the real work.

## Design Principle

Target the open protocol, not a specific vendor. A Terraform bridge speaks the go-plugin gRPC
protocol — it works with any Terraform/OpenTofu provider. An Ansible bridge invokes
`ansible-playbook` — it works with any collection. CaseHub distributes the adapter code (our
code, our license), never the external tool.

## Licensing Constraints

- **OpenTofu** (MPL 2.0) — usable in tests and as a runtime dependency. No distribution concern.
- **Terraform providers** (mostly MPL 2.0) — same.
- **ansible-core** (GPL 3.0) — **test-scope only**. CaseHub never bundles or distributes
  ansible-core. The `AnsibleBridgeProvider` shells out to a user-supplied `ansible-playbook`
  binary at runtime. Integration tests use ansible-core as a test dependency (`pip install`
  in CI). This is subprocess integration, not linking — CaseHub distributes only the adapter.

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│  CaseHub Reconciliation Loop                             │
│  ┌───────────────────┐  ┌────────────────────────────┐   │
│  │NodeProvisionerRouter│ │ActualStateAdapterRouter    │   │
│  └────────┬──────────┘  └──────────┬─────────────────┘   │
│           │                        │                     │
│  ┌────────▼──────────┐  ┌──────────▼─────────────────┐   │
│  │BridgeProvisioner   │  │BridgeActualStateAdapter    │   │
│  │(one per protocol)  │  │(one per protocol)          │   │
│  └────────┬──────────┘  └──────────┬─────────────────┘   │
│           │  Bridge Runtime        │                     │
│  ┌────────▼────────────────────────▼─────────────────┐   │
│  │BridgeProvider SPI (per protocol implementation)    │   │
│  │  apply() / destroy() / readState()                │   │
│  │  [optional: PlanCapable — plan() / applyPlan()]   │   │
│  └────────┬──────────────────────────────────────────┘   │
│           │                                              │
│  ┌────────▼──────────────────────────────────────────┐   │
│  │BridgeConnector (transport abstraction)             │   │
│  │  Managed Process │ External Service │ Custom       │   │
│  └────────┬──────────────────────────────────────────┘   │
└───────────┼──────────────────────────────────────────────┘
            │
   ┌────────▼──────────┐
   │External Provider   │
   │(Terraform, Ansible)│
   └───────────────────┘
```

The reconciliation loop sees `NodeProvisioner` and `ActualStateAdapter` — the same interfaces
used by Java and YAML plugins. All bridge-specific concerns (transport, process lifecycle,
protocol translation) are encapsulated below the existing SPI boundary. Fault escalation,
drift detection, plan approval, and lifecycle management layer on top unchanged.

## SPI Interfaces

### BridgeProvider

The core interface that protocol-specific implementations provide. Each implementation handles
one external provider protocol (e.g., Terraform go-plugin gRPC, Ansible CLI).

```java
public interface BridgeProvider {
    String protocolId();
    Set<String> supportedTypes();
    BridgeResult apply(BridgeRequest request);
    BridgeResult destroy(BridgeRequest request);
    BridgeStateResult readState(BridgeReadRequest request);
}
```

### PlanCapable

Optional mixin for providers with native plan-apply cycles (Terraform plan/apply, Ansible
check/apply). Bridge runtime detects this at construction and uses it when available.

```java
public interface PlanCapable {
    BridgePlan plan(BridgeRequest request);
    BridgeResult applyPlan(BridgePlan plan);
}
```

When `PlanCapable` is present, `BridgeProvisioner` calls `plan()` first and can integrate
with `PlanApprovalGate` before calling `applyPlan()`. When absent, `apply()` is called
directly.

### SuspendCapable

Optional mixin for providers that support native suspend/resume operations (e.g., stopping
an EC2 instance without terminating it).

```java
public interface SuspendCapable {
    BridgeResult suspend(BridgeRequest request);
    BridgeResult resume(BridgeRequest request);
}
```

When `SuspendCapable` is present, `BridgeProvisioner` delegates `NodeProvisioner.suspend()`
and `resume()` to the bridge provider. When absent, `BridgeProvisioner` uses the stateless
path (deprovision/provision) — matching `NodeProvisioner.supportsStatefulLifecycle() = false`.

### Request and Response Types

All sealed — bridge providers return discriminated results, and the bridge runtime translates
them to CaseHub's `ProvisionResult` / `DeprovisionResult` / `NodeStatus`.

```java
public record BridgeRequest(
    String typeId,
    String resourceId,
    Map<String, Object> spec,
    String tenancyId,
    Map<String, Object> previousState    // optional, for updates
) {}

public record BridgeReadRequest(
    String typeId,
    String resourceId,
    String tenancyId
) {}

public sealed interface BridgeResult {
    record Success(Map<String, Object> outputState) implements BridgeResult {}
    record Failed(String reason, BridgeErrorCategory category) implements BridgeResult {}
    record PendingApproval(String planReference) implements BridgeResult {}
}

public sealed interface BridgeStateResult {
    record Found(Map<String, Object> state) implements BridgeStateResult {}
    record NotFound() implements BridgeStateResult {}
    record Error(String reason) implements BridgeStateResult {}
}

public record BridgePlan(
    String planId,
    String summary,
    List<ResourceChange> resourceChanges,
    Object rawPlan                        // provider-native plan data
) {}

public record ResourceChange(
    String typeId,
    String resourceId,
    ChangeAction action
) {}

public enum ChangeAction { CREATE, UPDATE, DELETE }

public enum BridgeErrorCategory {
    TRANSIENT,        // retryable — network timeout, rate limit
    PERMANENT,        // non-retryable — resource doesn't exist, invalid config
    CONFIGURATION,    // spec error — bad AMI ID, wrong region
    AUTHENTICATION,   // credentials — expired token, wrong key
    QUOTA             // limit — too many instances, storage full
}
```

### BridgeConnector

Transport abstraction managed by the bridge runtime. Protocol-specific implementations
use this to communicate with the external provider, regardless of deployment model
(in-process, sidecar, remote).

```java
public interface BridgeConnector extends AutoCloseable {
    boolean isHealthy();
    <T> T send(BridgeOperation operation, Class<T> responseType);
}

public sealed interface BridgeOperation {
    record Apply(String typeId, String resourceId, Map<String, Object> spec) implements BridgeOperation {}
    record Destroy(String typeId, String resourceId) implements BridgeOperation {}
    record ReadState(String typeId, String resourceId) implements BridgeOperation {}
    record Plan(String typeId, String resourceId, Map<String, Object> spec) implements BridgeOperation {}
    record ApplyPlan(String planId, Object rawPlan) implements BridgeOperation {}
}
```

Concrete implementations:
- `ProcessBridgeConnector` — manages a child process, communicates via gRPC or stdin/stdout
- `HttpBridgeConnector` — connects to an external HTTP endpoint
- `GrpcBridgeConnector` — connects to an external gRPC endpoint

### BridgeDescriptor

The parsed manifest — analogous to `PluginDescriptor` in the YAML plugin surface:

```java
public record BridgeDescriptor(
    String protocolId,
    Set<NodeType> handledTypes,
    Map<String, BridgeTypeDescriptor> typeDescriptors,
    BridgeLifecycleConfig lifecycleConfig
) {}

public record BridgeTypeDescriptor(
    NodeType nodeType,
    PluginSpecSchema specSchema,
    Map<String, NodeStatus> stateMapping,
    NodeStatus defaultState,
    NodeStatus notFoundState,
    Map<String, BridgeErrorCategory> errorMapping,
    BridgeErrorCategory defaultErrorCategory
) {}

public record BridgeLifecycleConfig(
    LifecycleMode mode,           // MANAGED_PROCESS or EXTERNAL_SERVICE
    String binaryPath,            // for managed process
    List<String> args,
    HealthCheckConfig healthCheck,
    RestartPolicy restartPolicy,
    String endpoint               // for external service
) {}

public enum LifecycleMode { MANAGED_PROCESS, EXTERNAL_SERVICE }

public record HealthCheckConfig(
    HealthCheckType type,         // GRPC, HTTP, TCP
    Duration interval,
    Duration timeout
) {}

public enum HealthCheckType { GRPC, HTTP, TCP }

public record RestartPolicy(
    RestartPolicyType policy,     // ON_FAILURE, ALWAYS, NEVER
    int maxRetries,
    BackoffStrategy backoff       // FIXED, EXPONENTIAL
) {}

public enum RestartPolicyType { ON_FAILURE, ALWAYS, NEVER }
public enum BackoffStrategy { FIXED, EXPONENTIAL }
```

## Bridge Manifest Format

Located at `META-INF/desiredstate/bridges/*.yaml`, following the YAML plugin convention.

```yaml
bridge:
  protocol: terraform
  version: "1.0"

  lifecycle:
    mode: managed-process
    binary: /usr/local/bin/terraform-provider-aws
    args: ["--grpc"]
    healthCheck:
      type: grpc
      interval: 30s
      timeout: 5s
    restart:
      policy: on-failure
      maxRetries: 3
      backoff: exponential

  types:
    - id: aws_instance
      spec:
        required: [ami, instance_type]
        properties:
          ami: { type: string }
          instance_type: { type: string }
          tags: { type: map, valueType: string }
      stateMapping:
        running: PRESENT
        stopped: SUSPENDED
        terminated: ABSENT
        pending: UNKNOWN
        _default: DEGRADED
      errorMapping:
        InvalidAMIID: { category: CONFIGURATION }
        RequestLimitExceeded: { category: TRANSIENT }
        UnauthorizedAccess: { category: AUTHENTICATION }
        _default: { category: PERMANENT }

    - id: aws_s3_bucket
      spec:
        required: [bucket]
        properties:
          bucket: { type: string }
          acl: { type: string, enum: [private, public-read] }
      stateMapping:
        _default: PRESENT
        _notFound: ABSENT
      errorMapping:
        _default: { category: PERMANENT }
```

The `spec` section uses the same schema language as YAML plugin `PluginSpecSchema` —
`PluginSpecValidator` validates bridge specs with no additional code.

**Manifest maintenance:** When a provider adds new resource types, the manifest must be
updated manually. Provider schema validation can be an optional build-time Maven plugin
that compares manifest types against the provider's schema and flags missing or changed
types — but this is not a runtime requirement.

**State mapping escape hatch:** For complex state mappings that are hard to express
declaratively, `BridgeProvider` implementations can override state mapping programmatically.
`BridgeActualStateAdapter` checks the provider first; if no programmatic mapping is
returned, it falls back to the manifest rules.

## Module Structure

```
bridge/
├── api/                    casehub-desiredstate-bridge-api
│   BridgeProvider, PlanCapable, SuspendCapable,
│   BridgeConnector, BridgeOperation,
│   BridgeRequest, BridgeResult, BridgeStateResult,
│   BridgePlan, ResourceChange, ChangeAction,
│   BridgeDescriptor, BridgeErrorCategory,
│   BridgeLifecycleConfig, HealthCheckConfig, RestartPolicy
│
├── runtime/                casehub-desiredstate-bridge
│   BridgeProvisioner (implements NodeProvisioner),
│   BridgeActualStateAdapter (implements ActualStateAdapter),
│   BridgeDiscovery (manifest parsing → BeanRegistration),
│   BridgeLifecycleManager (process start/stop/health),
│   ProcessBridgeConnector, HttpBridgeConnector,
│   BridgeRuntimeBeans (CDI @Produces)
│
├── deployment/             casehub-desiredstate-bridge-deployment
│   Quarkus build extension: META-INF/desiredstate/bridges/*.yaml
│   discovery, type registry scan, manifest validation
│
├── spring/                 casehub-desiredstate-bridge-spring
│   Spring @AutoConfiguration: classpath bridge YAML discovery,
│   BridgeProvisioner + BridgeActualStateAdapter bean registration
│
├── testing/                casehub-desiredstate-bridge-testing
│   MockBridgeProvider, BridgeTestExtension (JUnit5),
│   BridgeTestRunner, test manifest helpers.
│   Test scope only — no external binaries.
│
├── terraform/              casehub-desiredstate-bridge-terraform  (#184)
│   TerraformBridgeProvider (implements BridgeProvider + PlanCapable),
│   GrpcBridgeConnector for go-plugin protocol,
│   integration tests with OpenTofu + null/local providers (MPL 2.0)
│
└── ansible/                casehub-desiredstate-bridge-ansible    (#185)
    AnsibleBridgeProvider (implements BridgeProvider),
    CliBridgeConnector for ansible-playbook subprocess,
    integration tests with ansible-core (GPL 3.0, test-scope only)
```

### Dependency Rules

- `bridge/api/` depends only on `casehub-desiredstate-api` — pure Java, no framework
- `bridge/runtime/` depends on `bridge/api/` + `runtime-core/` — same tier as `plugin/runtime/`
- `bridge/terraform/` depends on `bridge/api/` + gRPC libraries — isolated dependency tree
- `bridge/ansible/` depends on `bridge/api/` only — subprocess-based, no special Java deps
- `bridge/testing/` depends on `bridge/api/` — test scope only, no external binaries
- Foundation tier preserved — no upward dependencies to engine/work

### Licensing Boundary

CaseHub distributes `bridge/api/`, `bridge/runtime/`, `bridge/deployment/`, `bridge/spring/`,
`bridge/testing/`, `bridge/terraform/`, `bridge/ansible/` — all our code, our license.

External tools are never distributed:
- OpenTofu/Terraform providers: user-supplied at runtime (managed process mode) or
  pre-deployed (external service mode)
- ansible-core: user-installed, invoked via subprocess. Test dependency only (`pip install`
  in CI, `<scope>test</scope>` in Maven)

## Runtime Flow

### Provision Cycle

1. Reconciliation loop detects node is ABSENT in actual state but PRESENT in desired graph
2. TransitionPlanner creates an addition step
3. NodeProvisionerRouter dispatches `provision()` to `BridgeProvisioner`
4. BridgeProvisioner looks up `BridgeDescriptor` for the node type, extracts spec fields,
   builds `BridgeRequest`
5. BridgeProvisioner checks if its `BridgeProvider` implements `PlanCapable`:
   - Yes: calls `plan()`, optionally integrates with `PlanApprovalGate`, then `applyPlan()`
   - No: calls `apply()` directly
6. BridgeProvider sends the operation via `BridgeConnector` to the external provider
7. External provider does the real work, returns result
8. BridgeProvider returns `BridgeResult.Success(outputState)` or
   `BridgeResult.Failed(reason, category)`
9. BridgeProvisioner translates `BridgeResult` → `ProvisionResult`
10. On failure, `BridgeErrorCategory` maps to `FaultType` via manifest `errorMapping` →
    `FaultPolicyEngine` handles escalation

### Actual State Read

1. ReconciliationLoop calls `readActual()` on `BridgeActualStateAdapter`
2. Adapter iterates bridge-handled nodes, calls `BridgeProvider.readState()` for each
3. BridgeProvider queries external provider via `BridgeConnector`
4. `BridgeStateResult.Found(state)` → manifest `stateMapping` resolves to `NodeStatus`
5. `BridgeStateResult.NotFound` → manifest `_notFound` mapping (typically ABSENT)
6. `BridgeStateResult.Error` → `NodeStatus.UNKNOWN`

### Provider Lifecycle

`BridgeLifecycleManager` runs independently of the reconciliation loop:

- **Startup:** For managed-process bridges, starts the external provider binary, waits
  for health check to pass, then makes the `BridgeConnector` available
- **Health monitoring:** Periodic health checks per manifest config. On failure, triggers
  restart per restart policy
- **Restart:** Exponential backoff, max retries. After max retries exhausted, marks all
  types handled by this provider as UNKNOWN
- **Shutdown:** Graceful stop on application shutdown — drains in-flight operations, then
  terminates the process

For external-service bridges, lifecycle management is connection-only — no process to
manage, just connection health and reconnection.

## Testing Strategy

### bridge/testing/ — SPI Contract Tests

Mock-based tests verifying the bridge runtime wiring:
- `MockBridgeProvider` returns configurable results
- `BridgeTestExtension` (JUnit5 `@RegisterExtension`) sets up bridge runtime with mock
  provider and test manifest
- Tests verify: provision/deprovision flow, state mapping, error mapping, PlanCapable
  integration, PlanApprovalGate interaction, fault escalation

No external binaries. Pure Java. Runs in any environment.

### bridge/terraform/ — Protocol Integration Tests

- OpenTofu + `terraform-provider-local` (MPL 2.0) for filesystem-based tests
- Proves: gRPC go-plugin protocol works end-to-end, plan/apply cycle, state reads
- CI-safe: no cloud credentials, no network access, local filesystem only
- Optional `terraform-provider-null` for minimal protocol verification

### bridge/ansible/ — CLI Integration Tests

- `ansible-core` installed via pip in CI (GPL 3.0, test-scope only — never distributed)
- `ansible.builtin.file` + `ansible.builtin.copy` for local filesystem operations
- Proves: subprocess invocation, output parsing, state mapping
- CI-safe: no cloud credentials, no network access

## References

- NodeProvisioner SPI — `api/src/main/java/io/casehub/desiredstate/api/NodeProvisioner.java`
- ActualStateAdapter SPI — `api/src/main/java/io/casehub/desiredstate/api/ActualStateAdapter.java`
- NodeSpecFactory SPI — `api/src/main/java/io/casehub/desiredstate/api/NodeSpecFactory.java`
- YamlPluginProvisioner — `plugin/runtime/.../YamlPluginProvisioner.java`
- PluginDiscovery — `plugin/runtime/.../PluginDiscovery.java`
- DefaultNodeProvisionerRouter — `runtime-core/.../DefaultNodeProvisionerRouter.java`
- Issue #131 — plugin architecture design (three-tier model, execution model evaluation)
- Issue #182 — bridge adapter architecture epic
- Issue #184 — Terraform/OpenTofu bridge (planned)
- Issue #185 — Ansible collection bridge (planned)
- Issue #186 — bridge adapter testing framework (planned)
