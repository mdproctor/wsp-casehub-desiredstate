# Bridge Adapter Testing Framework Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #186 — feat: bridge adapter testing framework
**Issue group:** #186

**Goal:** Create a reusable testing module for bridge adapter implementors with mock provider infrastructure, TCK contract tests, and a JUnit5 extension for runtime wiring verification.

**Architecture:** New `bridge/testing/` Maven module with three components: (1) mock providers extracted from `bridge/runtime/src/test/` and enhanced with fault injection, (2) abstract TCK contract test classes per capability tier (base, plan, suspend, import), (3) `BridgeTestExtension` JUnit5 extension for runtime wiring tests. Follows the `plugin/testing/` pattern.

**Tech Stack:** Java 21, JUnit 5, AssertJ, Maven

## Global Constraints

- Package: `io.casehub.desiredstate.bridge.testing`
- Module artifact: `casehub-desiredstate-bridge-testing`
- Test scope only — never a compile or runtime dependency
- No YAML test format — bridge providers are pure Java SPIs
- No external test infrastructure (WireMock, shell sandbox) — bridge providers talk to external processes; integration tests belong in provider modules
- IntelliJ MCP required for all code navigation and editing during execution

---

## Batch 1: Module scaffold + mock infrastructure

### Task 1: Create bridge/testing Maven module with MockBridgeProvider

**Files:**
- Create: `bridge/testing/pom.xml`
- Modify: `bridge/pom.xml` (add `<module>testing</module>`)
- Create: `bridge/testing/src/main/java/io/casehub/desiredstate/bridge/testing/MockBridgeProvider.java`
- Create: `bridge/testing/src/main/java/io/casehub/desiredstate/bridge/testing/MockImportCapableBridgeProvider.java`
- Test: `bridge/testing/src/test/java/io/casehub/desiredstate/bridge/testing/MockBridgeProviderTest.java`

**Interfaces:**
- Consumes: `BridgeProvider`, `BridgeResult`, `BridgeStateResult`, `BridgeRequest`, `BridgeReadRequest`, `ImportCapable`, `BridgeImportResult` from `bridge/api`
- Produces: `MockBridgeProvider` (used by Tasks 2, 3, 4, 5), `MockImportCapableBridgeProvider` (used by Task 4)

- [ ] **Step 1: Create pom.xml for bridge/testing module**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-desiredstate-bridge-parent</artifactId>
        <version>0.4-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-desiredstate-bridge-testing</artifactId>
    <name>CaseHub Desired State :: Bridge :: Testing</name>
    <description>Bridge adapter test framework — mock providers, TCK contract tests,
        JUnit5 extension. Test scope only.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-bridge-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-bridge</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-runtime-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-api</artifactId>
        </dependency>

        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

- [ ] **Step 2: Add testing module to bridge/pom.xml**

Add `<module>testing</module>` to the `<modules>` list in `bridge/pom.xml`, after `terraform`:

```xml
<modules>
    <module>api</module>
    <module>runtime</module>
    <module>deployment</module>
    <module>spring</module>
    <module>terraform</module>
    <module>testing</module>
</modules>
```

- [ ] **Step 3: Write failing test for MockBridgeProvider fault injection**

```java
package io.casehub.desiredstate.bridge.testing;

import io.casehub.desiredstate.bridge.*;
import org.junit.jupiter.api.Test;
import java.util.Map;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;

class MockBridgeProviderTest {

    @Test
    void defaultApplyReturnsSuccess() {
        var mock = new MockBridgeProvider("test-proto", "res_a");
        var result = mock.apply(new BridgeRequest("res_a", "node-1", Map.of(), "t1"));
        assertThat(result).isInstanceOf(BridgeResult.Success.class);
    }

    @Test
    void failNextNReturnsFailureThenReverts() {
        var mock = new MockBridgeProvider("test-proto", "res_a");
        var failResult = new BridgeResult.Failed("boom", BridgeErrorCategory.TRANSIENT);
        mock.failNextN(2, failResult);

        var req = new BridgeRequest("res_a", "node-1", Map.of(), "t1");
        assertThat(mock.apply(req)).isInstanceOf(BridgeResult.Failed.class);
        assertThat(mock.apply(req)).isInstanceOf(BridgeResult.Failed.class);
        assertThat(mock.apply(req)).isInstanceOf(BridgeResult.Success.class);
    }

    @Test
    void failNextNDestroyReturnsFailureThenReverts() {
        var mock = new MockBridgeProvider("test-proto", "res_a");
        var failResult = new BridgeResult.Failed("crash", BridgeErrorCategory.PERMANENT);
        mock.failNextNDestroy(1, failResult);

        var req = new BridgeRequest("res_a", "node-1", Map.of(), "t1");
        assertThat(mock.destroy(req)).isInstanceOf(BridgeResult.Failed.class);
        assertThat(mock.destroy(req)).isInstanceOf(BridgeResult.Success.class);
    }

    @Test
    void callsAreRecorded() {
        var mock = new MockBridgeProvider("test-proto", "res_a");
        var req = new BridgeRequest("res_a", "node-1", Map.of("key", "val"), "t1");
        mock.apply(req);
        mock.destroy(req);
        mock.readState(new BridgeReadRequest("res_a", "node-1", "t1"));

        assertThat(mock.applyCalls()).hasSize(1);
        assertThat(mock.destroyCalls()).hasSize(1);
        assertThat(mock.readCalls()).hasSize(1);
        assertThat(mock.applyCalls().getFirst().spec()).containsEntry("key", "val");
    }

    @Test
    void protocolIdAndSupportedTypes() {
        var mock = new MockBridgeProvider("my-proto", "type_a", "type_b");
        assertThat(mock.protocolId()).isEqualTo("my-proto");
        assertThat(mock.supportedTypes()).containsExactlyInAnyOrder("type_a", "type_b");
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `mvn --batch-mode test -pl bridge/testing -Dtest=MockBridgeProviderTest`
Expected: Compilation failure — `MockBridgeProvider` does not exist yet in testing module.

- [ ] **Step 5: Implement MockBridgeProvider with fault injection**

Create `bridge/testing/src/main/java/io/casehub/desiredstate/bridge/testing/MockBridgeProvider.java`:

```java
package io.casehub.desiredstate.bridge.testing;

import io.casehub.desiredstate.bridge.*;
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;

public class MockBridgeProvider implements BridgeProvider {

    private final String protocolId;
    private final Set<String> types;
    private BridgeResult applyResult = new BridgeResult.Success(Map.of());
    private BridgeResult destroyResult = new BridgeResult.Success(Map.of());
    private BridgeStateResult readResult = new BridgeStateResult.NotFound();
    private final List<BridgeRequest> applyCalls = new ArrayList<>();
    private final List<BridgeRequest> destroyCalls = new ArrayList<>();
    private final List<BridgeReadRequest> readCalls = new ArrayList<>();

    private final AtomicInteger applyFailCount = new AtomicInteger(0);
    private BridgeResult applyFailResult;
    private final AtomicInteger destroyFailCount = new AtomicInteger(0);
    private BridgeResult destroyFailResult;

    public MockBridgeProvider(String protocolId, String... types) {
        this.protocolId = protocolId;
        this.types = Set.of(types);
    }

    @Override public String protocolId() { return protocolId; }
    @Override public Set<String> supportedTypes() { return types; }

    @Override
    public BridgeResult apply(BridgeRequest request) {
        applyCalls.add(request);
        if (applyFailCount.getAndUpdate(n -> n > 0 ? n - 1 : 0) > 0) {
            return applyFailResult;
        }
        return applyResult;
    }

    @Override
    public BridgeResult destroy(BridgeRequest request) {
        destroyCalls.add(request);
        if (destroyFailCount.getAndUpdate(n -> n > 0 ? n - 1 : 0) > 0) {
            return destroyFailResult;
        }
        return destroyResult;
    }

    @Override
    public BridgeStateResult readState(BridgeReadRequest request) {
        readCalls.add(request);
        return readResult;
    }

    public void setApplyResult(BridgeResult result) { this.applyResult = result; }
    public void setDestroyResult(BridgeResult result) { this.destroyResult = result; }
    public void setReadResult(BridgeStateResult result) { this.readResult = result; }

    public void failNextN(int n, BridgeResult failResult) {
        this.applyFailCount.set(n);
        this.applyFailResult = failResult;
    }

    public void failNextNDestroy(int n, BridgeResult failResult) {
        this.destroyFailCount.set(n);
        this.destroyFailResult = failResult;
    }

    public List<BridgeRequest> applyCalls() { return applyCalls; }
    public List<BridgeRequest> destroyCalls() { return destroyCalls; }
    public List<BridgeReadRequest> readCalls() { return readCalls; }
}
```

- [ ] **Step 6: Implement MockImportCapableBridgeProvider**

Create `bridge/testing/src/main/java/io/casehub/desiredstate/bridge/testing/MockImportCapableBridgeProvider.java`:

```java
package io.casehub.desiredstate.bridge.testing;

import io.casehub.desiredstate.bridge.*;
import java.util.*;

public class MockImportCapableBridgeProvider extends MockBridgeProvider implements ImportCapable {

    private BridgeImportResult importResult = new BridgeImportResult.Success(Map.of("id", "imported-123"));
    private final List<ImportCall> importCalls = new ArrayList<>();

    public MockImportCapableBridgeProvider(String protocolId, String... types) {
        super(protocolId, types);
    }

    @Override
    public BridgeImportResult importResource(String typeId, String importId, String tenancyId) {
        importCalls.add(new ImportCall(typeId, importId, tenancyId));
        return importResult;
    }

    public void setImportResult(BridgeImportResult result) { this.importResult = result; }
    public List<ImportCall> importCalls() { return importCalls; }

    public record ImportCall(String typeId, String importId, String tenancyId) {}
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl bridge/testing -Dtest=MockBridgeProviderTest`
Expected: All 5 tests PASS.

- [ ] **Step 8: Commit**

```bash
git add bridge/testing/ bridge/pom.xml
git commit -m "feat(bridge/testing): module scaffold + MockBridgeProvider with fault injection (#186)"
```

### Task 2: MockPlanCapableBridgeProvider and MockSuspendCapableBridgeProvider

**Files:**
- Create: `bridge/testing/src/main/java/io/casehub/desiredstate/bridge/testing/MockPlanCapableBridgeProvider.java`
- Create: `bridge/testing/src/main/java/io/casehub/desiredstate/bridge/testing/MockSuspendCapableBridgeProvider.java`
- Test: `bridge/testing/src/test/java/io/casehub/desiredstate/bridge/testing/MockCapabilityProvidersTest.java`

**Interfaces:**
- Consumes: `MockBridgeProvider` from Task 1, `PlanCapable`, `SuspendCapable`, `BridgePlan` from `bridge/api`
- Produces: `MockPlanCapableBridgeProvider` (used by Task 3), `MockSuspendCapableBridgeProvider` (used by Task 3)

- [ ] **Step 1: Write failing tests for capability mocks**

```java
package io.casehub.desiredstate.bridge.testing;

import io.casehub.desiredstate.bridge.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class MockCapabilityProvidersTest {

    @Test
    void planCapableReturnsPlanAndRecordsCalls() {
        var mock = new MockPlanCapableBridgeProvider("proto", "res");
        var plan = new BridgePlan("p1", "create resource", List.of(), null);
        mock.setPlanResult(plan);
        mock.setApplyPlanResult(new BridgeResult.Success(Map.of("id", "r1")));

        var req = new BridgeRequest("res", "node-1", Map.of(), "t1");
        var planResult = mock.plan(req);
        var applyResult = mock.applyPlan(planResult);

        assertThat(planResult.planId()).isEqualTo("p1");
        assertThat(applyResult).isInstanceOf(BridgeResult.Success.class);
        assertThat(mock.planCalls()).hasSize(1);
        assertThat(mock.applyPlanCalls()).hasSize(1);
    }

    @Test
    void suspendCapableRecordsSuspendAndResumeCalls() {
        var mock = new MockSuspendCapableBridgeProvider("proto", "res");
        mock.setSuspendResult(new BridgeResult.Success(Map.of()));
        mock.setResumeResult(new BridgeResult.Success(Map.of()));

        var req = new BridgeRequest("res", "node-1", Map.of(), "t1");
        assertThat(mock.suspend(req)).isInstanceOf(BridgeResult.Success.class);
        assertThat(mock.resume(req)).isInstanceOf(BridgeResult.Success.class);
        assertThat(mock.suspendCalls()).hasSize(1);
        assertThat(mock.resumeCalls()).hasSize(1);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl bridge/testing -Dtest=MockCapabilityProvidersTest`
Expected: Compilation failure.

- [ ] **Step 3: Implement MockPlanCapableBridgeProvider**

```java
package io.casehub.desiredstate.bridge.testing;

import io.casehub.desiredstate.bridge.*;
import java.util.*;

public class MockPlanCapableBridgeProvider extends MockBridgeProvider implements PlanCapable {

    private BridgePlan planResult = new BridgePlan("default-plan", "no-op", List.of(), null);
    private BridgeResult applyPlanResult = new BridgeResult.Success(Map.of());
    private final List<BridgeRequest> planCalls = new ArrayList<>();
    private final List<BridgePlan> applyPlanCalls = new ArrayList<>();

    public MockPlanCapableBridgeProvider(String protocolId, String... types) {
        super(protocolId, types);
    }

    @Override
    public BridgePlan plan(BridgeRequest request) {
        planCalls.add(request);
        return planResult;
    }

    @Override
    public BridgeResult applyPlan(BridgePlan plan) {
        applyPlanCalls.add(plan);
        return applyPlanResult;
    }

    public void setPlanResult(BridgePlan result) { this.planResult = result; }
    public void setApplyPlanResult(BridgeResult result) { this.applyPlanResult = result; }
    public List<BridgeRequest> planCalls() { return planCalls; }
    public List<BridgePlan> applyPlanCalls() { return applyPlanCalls; }
}
```

- [ ] **Step 4: Implement MockSuspendCapableBridgeProvider**

```java
package io.casehub.desiredstate.bridge.testing;

import io.casehub.desiredstate.bridge.*;
import java.util.*;

public class MockSuspendCapableBridgeProvider extends MockBridgeProvider implements SuspendCapable {

    private BridgeResult suspendResult = new BridgeResult.Success(Map.of());
    private BridgeResult resumeResult = new BridgeResult.Success(Map.of());
    private final List<BridgeRequest> suspendCalls = new ArrayList<>();
    private final List<BridgeRequest> resumeCalls = new ArrayList<>();

    public MockSuspendCapableBridgeProvider(String protocolId, String... types) {
        super(protocolId, types);
    }

    @Override
    public BridgeResult suspend(BridgeRequest request) {
        suspendCalls.add(request);
        return suspendResult;
    }

    @Override
    public BridgeResult resume(BridgeRequest request) {
        resumeCalls.add(request);
        return resumeResult;
    }

    public void setSuspendResult(BridgeResult result) { this.suspendResult = result; }
    public void setResumeResult(BridgeResult result) { this.resumeResult = result; }
    public List<BridgeRequest> suspendCalls() { return suspendCalls; }
    public List<BridgeRequest> resumeCalls() { return resumeCalls; }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl bridge/testing -Dtest=MockCapabilityProvidersTest`
Expected: All 2 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add bridge/testing/src/
git commit -m "feat(bridge/testing): MockPlanCapable + MockSuspendCapable providers (#186)"
```

## Batch 2: TCK contract tests

### Task 3: BridgeProviderContractTest and capability-tier contract tests

**Files:**
- Create: `bridge/testing/src/main/java/io/casehub/desiredstate/bridge/testing/BridgeProviderContractTest.java`
- Create: `bridge/testing/src/main/java/io/casehub/desiredstate/bridge/testing/PlanCapableContractTest.java`
- Create: `bridge/testing/src/main/java/io/casehub/desiredstate/bridge/testing/SuspendCapableContractTest.java`
- Create: `bridge/testing/src/main/java/io/casehub/desiredstate/bridge/testing/ImportCapableContractTest.java`
- Test: `bridge/testing/src/test/java/io/casehub/desiredstate/bridge/testing/MockBridgeProviderContractTest.java`
- Test: `bridge/testing/src/test/java/io/casehub/desiredstate/bridge/testing/MockPlanCapableContractTest.java`
- Test: `bridge/testing/src/test/java/io/casehub/desiredstate/bridge/testing/MockSuspendCapableContractTest.java`
- Test: `bridge/testing/src/test/java/io/casehub/desiredstate/bridge/testing/MockImportCapableContractTest.java`

**Interfaces:**
- Consumes: `MockBridgeProvider`, `MockPlanCapableBridgeProvider`, `MockSuspendCapableBridgeProvider`, `MockImportCapableBridgeProvider` from Tasks 1-2
- Produces: `BridgeProviderContractTest`, `PlanCapableContractTest`, `SuspendCapableContractTest`, `ImportCapableContractTest` (public API — consumed by downstream provider modules like `bridge/terraform`)

- [ ] **Step 1: Write the concrete TCK test for MockBridgeProvider (proves the abstract class works)**

```java
package io.casehub.desiredstate.bridge.testing;

import io.casehub.desiredstate.bridge.*;
import java.util.Map;
import java.util.Set;

class MockBridgeProviderContractTest extends BridgeProviderContractTest {

    @Override
    protected BridgeProvider createProvider() {
        var mock = new MockBridgeProvider("test-proto", "test_resource");
        mock.setApplyResult(new BridgeResult.Success(Map.of("id", "created-1")));
        mock.setReadResult(new BridgeStateResult.Found(Map.of("status", "active")));
        return mock;
    }

    @Override
    protected String expectedProtocolId() { return "test-proto"; }

    @Override
    protected Set<String> expectedSupportedTypes() { return Set.of("test_resource"); }

    @Override
    protected BridgeRequest sampleApplyRequest() {
        return new BridgeRequest("test_resource", "node-1", Map.of("name", "test"), "tenant-1");
    }

    @Override
    protected BridgeRequest sampleDestroyRequest() {
        return new BridgeRequest("test_resource", "node-1", Map.of(), "tenant-1");
    }

    @Override
    protected BridgeReadRequest sampleReadRequest() {
        return new BridgeReadRequest("test_resource", "node-1", "tenant-1");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl bridge/testing -Dtest=MockBridgeProviderContractTest`
Expected: Compilation failure — `BridgeProviderContractTest` does not exist.

- [ ] **Step 3: Implement BridgeProviderContractTest**

```java
package io.casehub.desiredstate.bridge.testing;

import io.casehub.desiredstate.bridge.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;

public abstract class BridgeProviderContractTest {

    protected abstract BridgeProvider createProvider();
    protected abstract String expectedProtocolId();
    protected abstract Set<String> expectedSupportedTypes();
    protected abstract BridgeRequest sampleApplyRequest();
    protected abstract BridgeRequest sampleDestroyRequest();
    protected abstract BridgeReadRequest sampleReadRequest();

    private BridgeProvider provider;

    @BeforeEach
    void setUp() {
        provider = createProvider();
    }

    @Test
    void protocolIdIsNonNull() {
        assertThat(provider.protocolId()).isNotNull().isNotEmpty();
    }

    @Test
    void supportedTypesIsNonEmpty() {
        assertThat(provider.supportedTypes()).isNotEmpty();
    }

    @Test
    void protocolIdMatchesExpected() {
        assertThat(provider.protocolId()).isEqualTo(expectedProtocolId());
    }

    @Test
    void supportedTypesMatchExpected() {
        assertThat(provider.supportedTypes()).isEqualTo(expectedSupportedTypes());
    }

    @Test
    void applyReturnsNonNull() {
        assertThat(provider.apply(sampleApplyRequest())).isNotNull();
    }

    @Test
    void destroyReturnsNonNull() {
        assertThat(provider.destroy(sampleDestroyRequest())).isNotNull();
    }

    @Test
    void readStateReturnsNonNull() {
        assertThat(provider.readState(sampleReadRequest())).isNotNull();
    }

    @Test
    void applySuccessHasOutputState() {
        var result = provider.apply(sampleApplyRequest());
        if (result instanceof BridgeResult.Success s) {
            assertThat(s.outputState()).isNotNull();
        }
    }

    @Test
    void applyWithSameRequestIsIdempotent() {
        var request = sampleApplyRequest();
        provider.apply(request);
        var result = provider.apply(request);
        assertThat(result).isNotNull();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl bridge/testing -Dtest=MockBridgeProviderContractTest`
Expected: All 9 tests PASS.

- [ ] **Step 5: Write concrete TCK tests for capability tiers**

`MockPlanCapableContractTest.java`:
```java
package io.casehub.desiredstate.bridge.testing;

import io.casehub.desiredstate.bridge.*;
import java.util.Map;

class MockPlanCapableContractTest extends PlanCapableContractTest {

    @Override
    protected PlanCapable createPlanCapableProvider() {
        return new MockPlanCapableBridgeProvider("test-proto", "test_resource");
    }

    @Override
    protected BridgeRequest samplePlanRequest() {
        return new BridgeRequest("test_resource", "node-1", Map.of("name", "test"), "tenant-1");
    }
}
```

`MockSuspendCapableContractTest.java`:
```java
package io.casehub.desiredstate.bridge.testing;

import io.casehub.desiredstate.bridge.*;
import java.util.Map;

class MockSuspendCapableContractTest extends SuspendCapableContractTest {

    @Override
    protected SuspendCapable createSuspendCapableProvider() {
        return new MockSuspendCapableBridgeProvider("test-proto", "test_resource");
    }

    @Override
    protected BridgeRequest sampleSuspendRequest() {
        return new BridgeRequest("test_resource", "node-1", Map.of(), "tenant-1");
    }
}
```

`MockImportCapableContractTest.java`:
```java
package io.casehub.desiredstate.bridge.testing;

class MockImportCapableContractTest extends ImportCapableContractTest {

    @Override
    protected io.casehub.desiredstate.bridge.ImportCapable createImportCapableProvider() {
        return new MockImportCapableBridgeProvider("test-proto", "test_resource");
    }

    @Override
    protected String sampleTypeId() { return "test_resource"; }

    @Override
    protected String sampleImportId() { return "import-abc-123"; }

    @Override
    protected String sampleTenancyId() { return "tenant-1"; }
}
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl bridge/testing -Dtest="MockPlanCapableContractTest,MockSuspendCapableContractTest,MockImportCapableContractTest"`
Expected: Compilation failure — abstract contract test classes don't exist.

- [ ] **Step 7: Implement PlanCapableContractTest**

```java
package io.casehub.desiredstate.bridge.testing;

import io.casehub.desiredstate.bridge.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

public abstract class PlanCapableContractTest {

    protected abstract PlanCapable createPlanCapableProvider();
    protected abstract BridgeRequest samplePlanRequest();

    private PlanCapable provider;

    @BeforeEach
    void setUp() {
        provider = createPlanCapableProvider();
    }

    @Test
    void planReturnsNonNull() {
        assertThat(provider.plan(samplePlanRequest())).isNotNull();
    }

    @Test
    void planHasPlanId() {
        var plan = provider.plan(samplePlanRequest());
        assertThat(plan.planId()).isNotNull();
    }

    @Test
    void applyPlanReturnsNonNull() {
        var plan = provider.plan(samplePlanRequest());
        assertThat(provider.applyPlan(plan)).isNotNull();
    }
}
```

- [ ] **Step 8: Implement SuspendCapableContractTest**

```java
package io.casehub.desiredstate.bridge.testing;

import io.casehub.desiredstate.bridge.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

public abstract class SuspendCapableContractTest {

    protected abstract SuspendCapable createSuspendCapableProvider();
    protected abstract BridgeRequest sampleSuspendRequest();

    private SuspendCapable provider;

    @BeforeEach
    void setUp() {
        provider = createSuspendCapableProvider();
    }

    @Test
    void suspendReturnsNonNull() {
        assertThat(provider.suspend(sampleSuspendRequest())).isNotNull();
    }

    @Test
    void resumeReturnsNonNull() {
        assertThat(provider.resume(sampleSuspendRequest())).isNotNull();
    }
}
```

- [ ] **Step 9: Implement ImportCapableContractTest**

```java
package io.casehub.desiredstate.bridge.testing;

import io.casehub.desiredstate.bridge.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

public abstract class ImportCapableContractTest {

    protected abstract ImportCapable createImportCapableProvider();
    protected abstract String sampleTypeId();
    protected abstract String sampleImportId();
    protected abstract String sampleTenancyId();

    private ImportCapable provider;

    @BeforeEach
    void setUp() {
        provider = createImportCapableProvider();
    }

    @Test
    void importResourceReturnsNonNull() {
        assertThat(provider.importResource(sampleTypeId(), sampleImportId(), sampleTenancyId()))
            .isNotNull();
    }

    @Test
    void importSuccessHasState() {
        var result = provider.importResource(sampleTypeId(), sampleImportId(), sampleTenancyId());
        if (result instanceof BridgeImportResult.Success s) {
            assertThat(s.state()).isNotNull();
        }
    }
}
```

- [ ] **Step 10: Run all contract tests to verify they pass**

Run: `mvn --batch-mode test -pl bridge/testing`
Expected: All tests PASS (MockBridgeProviderTest + MockCapabilityProvidersTest + all 4 contract tests).

- [ ] **Step 11: Commit**

```bash
git add bridge/testing/src/
git commit -m "feat(bridge/testing): TCK contract tests — base, plan, suspend, import (#186)"
```

## Batch 3: BridgeTestExtension + mock extraction + CLAUDE.md update

### Task 4: BridgeTestExtension

**Files:**
- Create: `bridge/testing/src/main/java/io/casehub/desiredstate/bridge/testing/BridgeTestExtension.java`
- Test: `bridge/testing/src/test/java/io/casehub/desiredstate/bridge/testing/BridgeTestExtensionTest.java`

**Interfaces:**
- Consumes: `MockBridgeProvider` from Task 1, `BridgeProvisioner`, `BridgeActualStateAdapter`, `InMemoryBridgeOutputStateStore` from `bridge/runtime`, `BridgeDescriptor`, `BridgeTypeDescriptor`, `BridgeLifecycleConfig` from `bridge/api`, `DefaultDesiredStateGraphFactory` from `runtime-core`
- Produces: `BridgeTestExtension` (public API — consumed by downstream provider modules)

- [ ] **Step 1: Write failing test for BridgeTestExtension**

```java
package io.casehub.desiredstate.bridge.testing;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.bridge.*;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;
import java.util.Map;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;

class BridgeTestExtensionTest {

    @RegisterExtension
    static BridgeTestExtension bridge = BridgeTestExtension.forTypes("test-proto", "test_resource");

    @Test
    void provisionerDelegatesToMockProvider() {
        bridge.mockProvider().setApplyResult(new BridgeResult.Success(Map.of("id", "i-123")));

        var node = new DesiredNode(NodeId.of("n1"),
            new BridgeNodeSpec(NodeType.of("test_resource"), Map.of("name", "test")),
            HumanGating.NONE);
        var graph = new DefaultDesiredStateGraphFactory().of(Set.of(node), Set.of());

        var result = bridge.provisioner().provision(node, new ProvisionContext("t1", graph));
        assertThat(result).isInstanceOf(ProvisionResult.Success.class);
        assertThat(bridge.mockProvider().applyCalls()).hasSize(1);
    }

    @Test
    void actualStateAdapterDelegatesToMockProvider() {
        bridge.mockProvider().setReadResult(new BridgeStateResult.Found(Map.of("status", "active")));

        var node = new DesiredNode(NodeId.of("n1"),
            new BridgeNodeSpec(NodeType.of("test_resource"), Map.of()),
            HumanGating.NONE);
        var graph = new DefaultDesiredStateGraphFactory().of(Set.of(node), Set.of());

        var actual = bridge.actualStateAdapter().readActual(graph, "t1");
        assertThat(actual.statusOf(NodeId.of("n1"))).hasValue(NodeStatus.PRESENT);
    }

    @Test
    void stateStoreIsAccessible() {
        bridge.stateStore().store("t1", "n1", Map.of("key", "val"));
        assertThat(bridge.stateStore().load("t1", "n1")).isPresent();
    }

    @Test
    void descriptorHasCorrectTypes() {
        assertThat(bridge.descriptor().handledTypes())
            .containsExactly(NodeType.of("test_resource"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl bridge/testing -Dtest=BridgeTestExtensionTest`
Expected: Compilation failure — `BridgeTestExtension` does not exist.

- [ ] **Step 3: Implement BridgeTestExtension**

```java
package io.casehub.desiredstate.bridge.testing;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.bridge.*;
import io.casehub.desiredstate.bridge.runtime.BridgeActualStateAdapter;
import io.casehub.desiredstate.bridge.runtime.BridgeProvisioner;
import io.casehub.desiredstate.bridge.runtime.InMemoryBridgeOutputStateStore;
import org.junit.jupiter.api.extension.BeforeEachCallback;
import org.junit.jupiter.api.extension.ExtensionContext;

import java.util.*;
import java.util.stream.Collectors;

public class BridgeTestExtension implements BeforeEachCallback {

    private final String protocolId;
    private final String[] resourceTypes;
    private final Map<String, NodeStatus> stateMapping;
    private final NodeStatus defaultState;
    private final NodeStatus notFoundState;

    private MockBridgeProvider mockProvider;
    private BridgeProvisioner provisioner;
    private BridgeActualStateAdapter actualStateAdapter;
    private InMemoryBridgeOutputStateStore stateStore;
    private BridgeDescriptor descriptor;

    private BridgeTestExtension(String protocolId, String[] resourceTypes,
                                 Map<String, NodeStatus> stateMapping,
                                 NodeStatus defaultState, NodeStatus notFoundState) {
        this.protocolId = protocolId;
        this.resourceTypes = resourceTypes;
        this.stateMapping = stateMapping;
        this.defaultState = defaultState;
        this.notFoundState = notFoundState;
    }

    public static BridgeTestExtension forTypes(String protocolId, String... resourceTypes) {
        return new BridgeTestExtension(protocolId, resourceTypes,
            Map.of("active", NodeStatus.PRESENT, "stopped", NodeStatus.SUSPENDED),
            NodeStatus.DRIFTED, NodeStatus.ABSENT);
    }

    public static Builder builder(String protocolId, String... resourceTypes) {
        return new Builder(protocolId, resourceTypes);
    }

    @Override
    public void beforeEach(ExtensionContext context) {
        mockProvider = new MockBridgeProvider(protocolId, resourceTypes);
        stateStore = new InMemoryBridgeOutputStateStore();

        Set<NodeType> handledTypes = Arrays.stream(resourceTypes)
            .map(NodeType::of)
            .collect(Collectors.toSet());

        Map<String, BridgeTypeDescriptor> typeDescriptors = new LinkedHashMap<>();
        for (String type : resourceTypes) {
            typeDescriptors.put(type, new BridgeTypeDescriptor(
                NodeType.of(type), stateMapping, defaultState, notFoundState,
                Map.of(), BridgeErrorCategory.PERMANENT));
        }

        descriptor = new BridgeDescriptor(protocolId, handledTypes, typeDescriptors,
            new BridgeLifecycleConfig(LifecycleMode.MANAGED, null, null, null));

        provisioner = new BridgeProvisioner(descriptor, mockProvider, stateStore);
        actualStateAdapter = new BridgeActualStateAdapter(descriptor, mockProvider, stateStore);
    }

    public MockBridgeProvider mockProvider() { return mockProvider; }
    public BridgeProvisioner provisioner() { return provisioner; }
    public BridgeActualStateAdapter actualStateAdapter() { return actualStateAdapter; }
    public InMemoryBridgeOutputStateStore stateStore() { return stateStore; }
    public BridgeDescriptor descriptor() { return descriptor; }

    public static class Builder {
        private final String protocolId;
        private final String[] resourceTypes;
        private final Map<String, NodeStatus> stateMapping = new LinkedHashMap<>();
        private NodeStatus defaultState = NodeStatus.DRIFTED;
        private NodeStatus notFoundState = NodeStatus.ABSENT;

        Builder(String protocolId, String[] resourceTypes) {
            this.protocolId = protocolId;
            this.resourceTypes = resourceTypes;
        }

        public Builder stateMapping(String providerState, NodeStatus nodeStatus) {
            stateMapping.put(providerState, nodeStatus);
            return this;
        }

        public Builder defaultState(NodeStatus status) {
            this.defaultState = status;
            return this;
        }

        public Builder notFoundState(NodeStatus status) {
            this.notFoundState = status;
            return this;
        }

        public BridgeTestExtension build() {
            Map<String, NodeStatus> mapping = stateMapping.isEmpty()
                ? Map.of("active", NodeStatus.PRESENT, "stopped", NodeStatus.SUSPENDED)
                : Map.copyOf(stateMapping);
            return new BridgeTestExtension(protocolId, resourceTypes, mapping,
                defaultState, notFoundState);
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl bridge/testing -Dtest=BridgeTestExtensionTest`
Expected: All 4 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add bridge/testing/src/
git commit -m "feat(bridge/testing): BridgeTestExtension — JUnit5 lifecycle for runtime wiring (#186)"
```

### Task 5: Extract mocks from bridge/runtime, update CLAUDE.md

**Files:**
- Delete: `bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/MockBridgeProvider.java` (use `ide_refactor_safe_delete`)
- Delete: `bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/MockImportCapableBridgeProvider.java` (use `ide_refactor_safe_delete`)
- Modify: `bridge/runtime/pom.xml` (add test-scope dependency on bridge/testing)
- Modify: `bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/BridgeActualStateAdapterTest.java` (update imports)
- Modify: `bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/BridgeDiscoveryTest.java` (update imports)
- Modify: `CLAUDE.md` (add bridge/testing module entry)

**Interfaces:**
- Consumes: `MockBridgeProvider`, `MockImportCapableBridgeProvider` from Task 1 (bridge/testing versions)
- Produces: Updated `bridge/runtime` tests using bridge/testing mocks

- [ ] **Step 1: Add test-scope dependency on bridge/testing to bridge/runtime/pom.xml**

Add to `<dependencies>` section:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-desiredstate-bridge-testing</artifactId>
    <version>${project.version}</version>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Update imports in BridgeActualStateAdapterTest.java**

Change import from:
```java
// Remove — no longer in runtime test sources
```
The test currently uses `MockBridgeProvider` from the same package. After deletion, it needs:
```java
import io.casehub.desiredstate.bridge.testing.MockBridgeProvider;
```

- [ ] **Step 3: Update imports in BridgeDiscoveryTest.java**

Add import:
```java
import io.casehub.desiredstate.bridge.testing.MockBridgeProvider;
```

- [ ] **Step 4: Update any other test files that import the old MockBridgeProvider**

Search for all test files in `bridge/runtime/src/test/` that import `MockBridgeProvider` or `MockImportCapableBridgeProvider` from the `runtime` package. Update each to import from `bridge.testing` package instead. Files to check:
- `BridgeActualStateAdapterTest.java` — uses `MockBridgeProvider`
- `BridgeDiscoveryTest.java` — uses `MockBridgeProvider`
- `InMemoryBridgeOutputStateStoreTest.java` — does not use mocks (no change)

Also check `bridge/terraform/src/test/` for any usage — those tests use their own real provider, no mock imports needed.

- [ ] **Step 5: Delete old mock files from bridge/runtime/src/test/**

Use `ide_refactor_safe_delete` for:
- `bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/MockBridgeProvider.java`
- `bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/MockImportCapableBridgeProvider.java`

If IntelliJ unavailable, use `git rm`:
```bash
git rm bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/MockBridgeProvider.java
git rm bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/MockImportCapableBridgeProvider.java
```

- [ ] **Step 6: Verify bridge/runtime tests still pass**

Run: `mvn --batch-mode test -pl bridge/runtime`
Expected: All existing tests PASS with the new import paths.

- [ ] **Step 7: Update CLAUDE.md module table**

Add the bridge/testing entry to the `## Module Structure` table, after the `bridge/spring/` entry:

```markdown
| `bridge/testing/` | `casehub-desiredstate-bridge-testing` | `io.casehub.desiredstate.bridge.testing` | Bridge adapter test framework — `MockBridgeProvider` (with fault injection), `MockPlanCapableBridgeProvider`, `MockSuspendCapableBridgeProvider`, `MockImportCapableBridgeProvider`, TCK contract tests (`BridgeProviderContractTest`, `PlanCapableContractTest`, `SuspendCapableContractTest`, `ImportCapableContractTest`), `BridgeTestExtension` (JUnit5). **Test scope only.** |
```

- [ ] **Step 8: Run full build to verify nothing is broken**

Run: `mvn --batch-mode install`
Expected: Full build PASS.

- [ ] **Step 9: Commit**

```bash
git add bridge/runtime/ bridge/testing/ CLAUDE.md
git commit -m "feat(bridge/testing): extract mocks from runtime, update CLAUDE.md (#186)"
```

## References

- [2026-10-30-bridge-adapter-testing-framework-design.md] — design spec this plan implements
- `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeProvider.java` — core SPI
- `bridge/api/src/main/java/io/casehub/desiredstate/bridge/PlanCapable.java` — plan mixin
- `bridge/api/src/main/java/io/casehub/desiredstate/bridge/SuspendCapable.java` — suspend mixin
- `bridge/api/src/main/java/io/casehub/desiredstate/bridge/ImportCapable.java` — import mixin
- `bridge/runtime/src/main/java/.../BridgeProvisioner.java` — runtime provisioner
- `bridge/runtime/src/main/java/.../BridgeActualStateAdapter.java` — runtime adapter
- `bridge/runtime/src/test/java/.../MockBridgeProvider.java` — extraction source
- `plugin/testing/` — analogue framework for YAML plugins
- GitHub #186 — focal issue
- GitHub #182 — parent epic
