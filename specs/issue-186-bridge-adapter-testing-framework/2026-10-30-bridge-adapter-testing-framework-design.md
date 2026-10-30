# Bridge Adapter Testing Framework — Design Spec

**Issue:** casehubio/casehub-desiredstate#186
**Date:** 2026-10-30
**Scale:** S | **Complexity:** Med

## Context

Bridge adapters wrap external provider protocols (Terraform, Ansible, etc.) as CaseHub desired-state
participants via the `BridgeProvider` SPI and its capability mixins (`PlanCapable`, `SuspendCapable`,
`ImportCapable`). Currently, each bridge implementation rolls its own test mocks — `MockBridgeProvider`
and `MockImportCapableBridgeProvider` live in `bridge/runtime/src/test/` and are not reusable by
downstream provider modules.

The `plugin/testing/` module provides a mature test framework for YAML plugins, but its architecture
is tightly coupled to the YAML step pipeline (`StepRunner`, `StructuralStepEvaluator`, `PluginRegistry`).
Bridge providers are pure Java SPIs — they need a test framework that matches their simpler contract.

## Module

`bridge/testing/` — new Maven module `casehub-desiredstate-bridge-testing` under `bridge/pom.xml`.
Package: `io.casehub.desiredstate.bridge.testing`. Test scope only.

### Dependencies

- `casehub-desiredstate-bridge-api` (bridge SPI types)
- `casehub-desiredstate-bridge-runtime` (BridgeProvisioner, BridgeActualStateAdapter, InMemoryBridgeOutputStateStore)
- `casehub-desiredstate-api` (DesiredNode, NodeStatus, ProvisionResult, etc.)
- `casehub-desiredstate-runtime-core` (DefaultDesiredStateGraphFactory)
- `org.junit.jupiter:junit-jupiter-api`
- `org.assertj:assertj-core` (test scope — for the module's own tests)

## Component 1: Mock Infrastructure

### MockBridgeProvider

Extracted from `bridge/runtime/src/test/` and enhanced with fault injection.

**Capabilities:**
- Programmable return values for `apply()`, `destroy()`, `readState()` via setters
- Call recording: `applyCalls()`, `destroyCalls()`, `readCalls()` return captured requests
- **Fault injection:**
  - `failNextN(int n, BridgeResult failResult)` — next N calls to `apply()` return the failure, then revert to normal
  - `failNextNDestroy(int n, BridgeResult failResult)` — same for `destroy()`
  - `setLatencyMs(long ms)` — simulated delay (Thread.sleep) before returning

**Not in scope:** Timeout simulation via actual thread interruption — too platform-dependent. Use `setLatencyMs` + test timeouts instead.

### MockPlanCapableBridgeProvider

Extends `MockBridgeProvider`, implements `PlanCapable`.

- Programmable `plan()` result via `setPlanResult(BridgePlan)`
- Programmable `applyPlan()` result via `setApplyPlanResult(BridgeResult)`
- Call recording: `planCalls()`, `applyPlanCalls()`

### MockSuspendCapableBridgeProvider

Extends `MockBridgeProvider`, implements `SuspendCapable`.

- Programmable `suspend()` / `resume()` results
- Call recording: `suspendCalls()`, `resumeCalls()`

### MockImportCapableBridgeProvider

Extracted from `bridge/runtime/src/test/`. Already complete — no changes needed.

## Component 2: TCK Contract Tests

Abstract test classes per capability tier. Each defines a single abstract factory method; provider
implementors extend the class and supply their provider.

### BridgeProviderContractTest

```java
public abstract class BridgeProviderContractTest {
    protected abstract BridgeProvider createProvider();
    protected abstract String expectedProtocolId();
    protected abstract Set<String> expectedSupportedTypes();
    protected abstract BridgeRequest sampleApplyRequest();
    protected abstract BridgeRequest sampleDestroyRequest();
    protected abstract BridgeReadRequest sampleReadRequest();
}
```

**Contract tests:**
- `protocolIdIsNonNull` — `protocolId()` returns non-null, non-empty
- `supportedTypesIsNonEmpty` — `supportedTypes()` returns at least one type
- `protocolIdMatchesExpected` — matches `expectedProtocolId()`
- `supportedTypesMatchExpected` — matches `expectedSupportedTypes()`
- `applyReturnsNonNull` — `apply()` never returns null
- `destroyReturnsNonNull` — `destroy()` never returns null
- `readStateReturnsNonNull` — `readState()` never returns null
- `applySuccessHasOutputState` — when `apply()` returns `Success`, `outputState()` is non-null
- `applyWithSameRequestIsIdempotent` — calling `apply()` twice with the same request does not throw

### PlanCapableContractTest

```java
public abstract class PlanCapableContractTest {
    protected abstract PlanCapable createPlanCapableProvider();
    protected abstract BridgeRequest samplePlanRequest();
}
```

**Contract tests:**
- `planReturnsNonNull` — `plan()` returns non-null `BridgePlan`
- `planHasPlanId` — `BridgePlan.planId()` is non-null
- `applyPlanReturnsNonNull` — `applyPlan()` returns non-null result

### SuspendCapableContractTest

```java
public abstract class SuspendCapableContractTest {
    protected abstract SuspendCapable createSuspendCapableProvider();
    protected abstract BridgeRequest sampleSuspendRequest();
}
```

**Contract tests:**
- `suspendReturnsNonNull` — `suspend()` returns non-null result
- `resumeReturnsNonNull` — `resume()` returns non-null result

### ImportCapableContractTest

```java
public abstract class ImportCapableContractTest {
    protected abstract ImportCapable createImportCapableProvider();
    protected abstract String sampleTypeId();
    protected abstract String sampleImportId();
    protected abstract String sampleTenancyId();
}
```

**Contract tests:**
- `importResourceReturnsNonNull` — `importResource()` returns non-null result
- `importSuccessHasState` — when returns `Success`, `state()` is non-null

## Component 3: BridgeTestExtension

JUnit5 `BeforeEachCallback` / `AfterEachCallback` that wires up the full bridge runtime stack with
a `MockBridgeProvider`. Provider authors use this to test their provider through the
BridgeProvisioner/BridgeActualStateAdapter path.

```java
public class BridgeTestExtension implements BeforeEachCallback, AfterEachCallback {
    private final String protocolId;
    private final String[] resourceTypes;
    // ...
    public static BridgeTestExtension forTypes(String protocolId, String... resourceTypes);
    public MockBridgeProvider mockProvider();
    public BridgeProvisioner provisioner();
    public BridgeActualStateAdapter actualStateAdapter();
    public InMemoryBridgeOutputStateStore stateStore();
    public BridgeDescriptor descriptor();
}
```

**Per-test lifecycle:**
1. `beforeEach`: Creates fresh `MockBridgeProvider`, `BridgeDescriptor` (with default state mappings),
   `InMemoryBridgeOutputStateStore`, `BridgeProvisioner`, `BridgeActualStateAdapter`.
2. `afterEach`: Clears state store.

**Default state mappings** (overridable via builder):
- `"active"` → `PRESENT`
- `"stopped"` → `SUSPENDED`
- Default: `DRIFTED`
- Not found: `ABSENT`

### BridgeTestExtension.Builder

For custom `BridgeTypeDescriptor` configuration:

```java
BridgeTestExtension.builder("my-protocol", "my_resource")
    .stateMapping("running", NodeStatus.PRESENT)
    .stateMapping("terminated", NodeStatus.ABSENT)
    .defaultState(NodeStatus.UNKNOWN)
    .notFoundState(NodeStatus.ABSENT)
    .build();
```

## Extraction from bridge/runtime

`MockBridgeProvider` and `MockImportCapableBridgeProvider` move from `bridge/runtime/src/test/` to
`bridge/testing/src/main/`. The originals in `bridge/runtime/src/test/` are deleted; `bridge/runtime`
gains a test-scope dependency on `bridge/testing`.

## What This Does NOT Include

- **YAML-driven test suites** — bridge providers are Java SPIs, not YAML pipelines. No test YAML format.
- **Integration test infrastructure** (WireMock, shell sandbox) — bridge providers talk to external
  processes; integration tests belong in the provider module (e.g., `bridge/terraform/`), not the
  generic testing framework.
- **Decorator-based fault injection** — wrapping real providers is out of scope; the mock covers
  fault injection for unit tests.

## References

- `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeProvider.java` — core SPI
- `bridge/api/src/main/java/io/casehub/desiredstate/bridge/PlanCapable.java` — plan mixin
- `bridge/api/src/main/java/io/casehub/desiredstate/bridge/SuspendCapable.java` — suspend mixin
- `bridge/api/src/main/java/io/casehub/desiredstate/bridge/ImportCapable.java` — import mixin
- `bridge/runtime/src/main/java/.../BridgeProvisioner.java` — runtime provisioner
- `bridge/runtime/src/main/java/.../BridgeActualStateAdapter.java` — runtime actual-state adapter
- `bridge/runtime/src/test/java/.../MockBridgeProvider.java` — extraction source
- `bridge/runtime/src/test/java/.../MockImportCapableBridgeProvider.java` — extraction source
- `plugin/testing/` — analogue framework for YAML plugins
- `testing/` — core mock SPIs pattern
- casehubio/casehub-desiredstate#182 — parent epic (bridge adapter architecture)
- casehubio/casehub-desiredstate#183 — dependency (bridge adapter SPI design, closed)
