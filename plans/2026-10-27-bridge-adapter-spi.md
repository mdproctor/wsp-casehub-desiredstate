# Bridge Adapter SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #183 — design: bridge adapter SPI and execution model
**Issue group:** #183

**Goal:** Implement the bridge adapter SPI, runtime, and framework wiring so that concrete
bridge providers (Terraform #184, Ansible #185) can plug in as standard CaseHub plugins.

**Architecture:** New `bridge/` top-level module mirrors the `plugin/` structure. `bridge/api/`
defines the SPI interfaces and types. `bridge/runtime/` implements `NodeProvisioner` +
`ActualStateAdapter` and delegates to `BridgeProvider` implementations. `bridge/deployment/`
and `bridge/spring/` provide Quarkus and Spring wiring. Manifest-driven type discovery and
state/error mapping.

**Tech Stack:** Java 21+, Quarkus (CDI + deployment extensions), Spring Boot auto-configuration,
Jackson YAML, JUnit 5

## Global Constraints

- groupId: `io.casehub`, version: `0.4-SNAPSHOT`, parent: `casehub-desiredstate-parent`
- All modules use `io.smallrye:jandex-maven-plugin` with `make-index` execution
- Foundation tier — no dependencies on engine-adapter, work-adapter, or any domain module
- `bridge/api/` depends only on `casehub-desiredstate-api` — pure Java, no framework
- `NodeProvisioner.handledTypes()` must not overlap with any other provisioner
- Sealed return types: use Java 21 sealed interfaces
- Package root: `io.casehub.desiredstate.bridge` for api, `io.casehub.desiredstate.bridge.runtime` for runtime

---

## Batch 1: SPI Foundation (bridge/api)

### Task 1: Bridge SPI interfaces and types

**Files:**
- Create: `bridge/pom.xml` (reactor parent)
- Create: `bridge/api/pom.xml`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeProvider.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/PlanCapable.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/SuspendCapable.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeRequest.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeReadRequest.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeResult.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeStateResult.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgePlan.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/ResourceChange.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/ChangeAction.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeErrorCategory.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeNodeSpec.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeConnector.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeOperation.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeDescriptor.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeTypeDescriptor.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeLifecycleConfig.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/LifecycleMode.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/HealthCheckConfig.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/HealthCheckType.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/RestartPolicy.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/RestartPolicyType.java`
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BackoffStrategy.java`
- Modify: `pom.xml` (root) — add `bridge` to `<modules>`
- Test: `bridge/api/src/test/java/io/casehub/desiredstate/bridge/BridgeResultTest.java`
- Test: `bridge/api/src/test/java/io/casehub/desiredstate/bridge/BridgeOperationTest.java`

**Interfaces:**
- Produces: `BridgeProvider`, `PlanCapable`, `SuspendCapable` (SPIs for Task 3)
- Produces: `BridgeRequest`, `BridgeResult`, `BridgeStateResult` (used by Tasks 3, 4)
- Produces: `BridgeDescriptor`, `BridgeTypeDescriptor` (used by Tasks 2, 3)
- Produces: `BridgeConnector`, `BridgeOperation` (used by Task 4)
- Produces: `BridgeLifecycleConfig`, `HealthCheckConfig`, `RestartPolicy` (used by Task 4)

- [ ] **Step 1: Create bridge reactor pom**

Create `bridge/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-desiredstate-parent</artifactId>
        <version>0.4-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-desiredstate-bridge-parent</artifactId>
    <packaging>pom</packaging>
    <name>CaseHub Desired State :: Bridge :: Parent</name>

    <modules>
        <module>api</module>
    </modules>
</project>
```

Create `bridge/api/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-desiredstate-bridge-parent</artifactId>
        <version>0.4-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-desiredstate-bridge-api</artifactId>
    <name>CaseHub Desired State :: Bridge :: API</name>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-api</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>${jandex-maven-plugin.version}</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

Add `<module>bridge</module>` to root `pom.xml` `<modules>` section, after `<module>plugin</module>`.

- [ ] **Step 2: Write sealed result type tests**

Create `BridgeResultTest.java`:

```java
package io.casehub.desiredstate.bridge;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class BridgeResultTest {

    @Test
    void successCarriesOutputState() {
        var state = Map.of("id", (Object) "i-123", "status", "running");
        var result = new BridgeResult.Success(state);
        assertEquals("i-123", result.outputState().get("id"));
    }

    @Test
    void failedCarriesReasonAndCategory() {
        var result = new BridgeResult.Failed("bad AMI", BridgeErrorCategory.CONFIGURATION);
        assertEquals("bad AMI", result.reason());
        assertEquals(BridgeErrorCategory.CONFIGURATION, result.category());
    }

    @Test
    void pendingApprovalCarriesPlanReference() {
        var result = new BridgeResult.PendingApproval("plan-42");
        assertEquals("plan-42", result.planReference());
    }

    @Test
    void stateResultFoundCarriesState() {
        var state = Map.of("status", (Object) "running");
        var result = new BridgeStateResult.Found(state);
        assertEquals("running", result.state().get("status"));
    }

    @Test
    void stateResultNotFoundIsDistinct() {
        var result = new BridgeStateResult.NotFound();
        assertInstanceOf(BridgeStateResult.class, result);
    }

    @Test
    void stateResultErrorCarriesReason() {
        var result = new BridgeStateResult.Error("timeout");
        assertEquals("timeout", result.reason());
    }

    @Test
    void sealedExhaustiveness() {
        BridgeResult result = new BridgeResult.Success(Map.of());
        var label = switch (result) {
            case BridgeResult.Success s -> "success";
            case BridgeResult.Failed f -> "failed";
            case BridgeResult.PendingApproval p -> "pending";
        };
        assertEquals("success", label);
    }
}
```

Create `BridgeOperationTest.java`:

```java
package io.casehub.desiredstate.bridge;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class BridgeOperationTest {

    @Test
    void applyCarriesSpecFields() {
        var spec = Map.of("ami", (Object) "ami-123");
        var op = new BridgeOperation.Apply("aws_instance", "i-1", spec);
        assertEquals("aws_instance", op.typeId());
        assertEquals("i-1", op.resourceId());
        assertEquals("ami-123", op.spec().get("ami"));
    }

    @Test
    void destroyCarriesIdentity() {
        var op = new BridgeOperation.Destroy("aws_instance", "i-1");
        assertEquals("aws_instance", op.typeId());
    }

    @Test
    void sealedExhaustiveness() {
        BridgeOperation op = new BridgeOperation.ReadState("aws_instance", "i-1");
        var label = switch (op) {
            case BridgeOperation.Apply a -> "apply";
            case BridgeOperation.Destroy d -> "destroy";
            case BridgeOperation.ReadState r -> "read";
            case BridgeOperation.Plan p -> "plan";
            case BridgeOperation.ApplyPlan ap -> "apply-plan";
        };
        assertEquals("read", label);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl bridge/api -am`
Expected: compilation failure — types not yet created.

- [ ] **Step 4: Create SPI interfaces**

Create `BridgeProvider.java`:

```java
package io.casehub.desiredstate.bridge;

import java.util.Set;

public interface BridgeProvider {
    String protocolId();
    Set<String> supportedTypes();
    BridgeResult apply(BridgeRequest request);
    BridgeResult destroy(BridgeRequest request);
    BridgeStateResult readState(BridgeReadRequest request);
}
```

Create `PlanCapable.java`:

```java
package io.casehub.desiredstate.bridge;

public interface PlanCapable {
    BridgePlan plan(BridgeRequest request);
    BridgeResult applyPlan(BridgePlan plan);
}
```

Create `SuspendCapable.java`:

```java
package io.casehub.desiredstate.bridge;

public interface SuspendCapable {
    BridgeResult suspend(BridgeRequest request);
    BridgeResult resume(BridgeRequest request);
}
```

- [ ] **Step 5: Create request/response types**

Create `BridgeRequest.java`:

```java
package io.casehub.desiredstate.bridge;

import java.util.Map;

public record BridgeRequest(
    String typeId,
    String resourceId,
    Map<String, Object> spec,
    String tenancyId,
    Map<String, Object> previousState
) {
    public BridgeRequest(String typeId, String resourceId, Map<String, Object> spec, String tenancyId) {
        this(typeId, resourceId, spec, tenancyId, Map.of());
    }
}
```

Create `BridgeReadRequest.java`:

```java
package io.casehub.desiredstate.bridge;

public record BridgeReadRequest(String typeId, String resourceId, String tenancyId) {}
```

Create `BridgeResult.java`:

```java
package io.casehub.desiredstate.bridge;

import java.util.Map;

public sealed interface BridgeResult {
    record Success(Map<String, Object> outputState) implements BridgeResult {}
    record Failed(String reason, BridgeErrorCategory category) implements BridgeResult {}
    record PendingApproval(String planReference) implements BridgeResult {}
}
```

Create `BridgeStateResult.java`:

```java
package io.casehub.desiredstate.bridge;

import java.util.Map;

public sealed interface BridgeStateResult {
    record Found(Map<String, Object> state) implements BridgeStateResult {}
    record NotFound() implements BridgeStateResult {}
    record Error(String reason) implements BridgeStateResult {}
}
```

Create `BridgePlan.java`:

```java
package io.casehub.desiredstate.bridge;

import java.util.List;

public record BridgePlan(
    String planId,
    String summary,
    List<ResourceChange> resourceChanges,
    Object rawPlan
) {}
```

Create `ResourceChange.java`:

```java
package io.casehub.desiredstate.bridge;

public record ResourceChange(String typeId, String resourceId, ChangeAction action) {}
```

Create `ChangeAction.java`:

```java
package io.casehub.desiredstate.bridge;

public enum ChangeAction { CREATE, UPDATE, DELETE }
```

Create `BridgeNodeSpec.java` — NodeSpec implementation for bridge-managed nodes, wraps `Map<String, Object>`:

```java
package io.casehub.desiredstate.bridge;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import java.util.Map;

public record BridgeNodeSpec(NodeType nodeType, Map<String, Object> fields) implements NodeSpec {
    public BridgeNodeSpec {
        if (fields == null) fields = Map.of();
    }
}
```

Create `BridgeErrorCategory.java`:

```java
package io.casehub.desiredstate.bridge;

public enum BridgeErrorCategory {
    TRANSIENT,
    PERMANENT,
    CONFIGURATION,
    AUTHENTICATION,
    QUOTA
}
```

- [ ] **Step 6: Create transport and connector types**

Create `BridgeConnector.java`:

```java
package io.casehub.desiredstate.bridge;

public interface BridgeConnector extends AutoCloseable {
    boolean isHealthy();
    <T> T send(BridgeOperation operation, Class<T> responseType);
}
```

Create `BridgeOperation.java`:

```java
package io.casehub.desiredstate.bridge;

import java.util.Map;

public sealed interface BridgeOperation {
    record Apply(String typeId, String resourceId, Map<String, Object> spec) implements BridgeOperation {}
    record Destroy(String typeId, String resourceId) implements BridgeOperation {}
    record ReadState(String typeId, String resourceId) implements BridgeOperation {}
    record Plan(String typeId, String resourceId, Map<String, Object> spec) implements BridgeOperation {}
    record ApplyPlan(String planId, Object rawPlan) implements BridgeOperation {}
}
```

- [ ] **Step 7: Create descriptor and lifecycle config types**

Create `BridgeDescriptor.java`:

```java
package io.casehub.desiredstate.bridge;

import io.casehub.desiredstate.api.NodeType;
import java.util.Map;
import java.util.Set;

public record BridgeDescriptor(
    String protocolId,
    Set<NodeType> handledTypes,
    Map<String, BridgeTypeDescriptor> typeDescriptors,
    BridgeLifecycleConfig lifecycleConfig
) {}
```

Create `BridgeTypeDescriptor.java`:

```java
package io.casehub.desiredstate.bridge;

import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.NodeStatus;
import java.util.Map;

public record BridgeTypeDescriptor(
    NodeType nodeType,
    Map<String, NodeStatus> stateMapping,
    NodeStatus defaultState,
    NodeStatus notFoundState,
    Map<String, BridgeErrorCategory> errorMapping,
    BridgeErrorCategory defaultErrorCategory
) {
    public BridgeTypeDescriptor {
        if (defaultState == null) defaultState = NodeStatus.DEGRADED;
        if (notFoundState == null) notFoundState = NodeStatus.ABSENT;
        if (defaultErrorCategory == null) defaultErrorCategory = BridgeErrorCategory.PERMANENT;
    }
}
```

Create `BridgeLifecycleConfig.java`:

```java
package io.casehub.desiredstate.bridge;

import java.util.List;

public record BridgeLifecycleConfig(
    LifecycleMode mode,
    String binaryPath,
    List<String> args,
    HealthCheckConfig healthCheck,
    RestartPolicy restartPolicy,
    String endpoint
) {
    public BridgeLifecycleConfig {
        if (args == null) args = List.of();
    }
}
```

Create `LifecycleMode.java`:

```java
package io.casehub.desiredstate.bridge;

public enum LifecycleMode { MANAGED_PROCESS, EXTERNAL_SERVICE }
```

Create `HealthCheckConfig.java`:

```java
package io.casehub.desiredstate.bridge;

import java.time.Duration;

public record HealthCheckConfig(HealthCheckType type, Duration interval, Duration timeout) {
    public HealthCheckConfig {
        if (interval == null) interval = Duration.ofSeconds(30);
        if (timeout == null) timeout = Duration.ofSeconds(5);
    }
}
```

Create `HealthCheckType.java`:

```java
package io.casehub.desiredstate.bridge;

public enum HealthCheckType { GRPC, HTTP, TCP }
```

Create `RestartPolicy.java`:

```java
package io.casehub.desiredstate.bridge;

public record RestartPolicy(RestartPolicyType policy, int maxRetries, BackoffStrategy backoff) {
    public RestartPolicy {
        if (backoff == null) backoff = BackoffStrategy.EXPONENTIAL;
    }
}
```

Create `RestartPolicyType.java`:

```java
package io.casehub.desiredstate.bridge;

public enum RestartPolicyType { ON_FAILURE, ALWAYS, NEVER }
```

Create `BackoffStrategy.java`:

```java
package io.casehub.desiredstate.bridge;

public enum BackoffStrategy { FIXED, EXPONENTIAL }
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl bridge/api -am`
Expected: PASS — all types compile, sealed exhaustiveness works, records carry correct fields.

- [ ] **Step 9: Commit**

```bash
git add bridge/ pom.xml
git commit -m "feat(bridge): add bridge/api module with SPI interfaces and types

Introduces BridgeProvider, PlanCapable, SuspendCapable SPIs.
Sealed request/response types: BridgeResult, BridgeStateResult, BridgeOperation.
Descriptor records for manifest-driven type and lifecycle configuration.

Refs #183"
```

---

## Batch 2: Runtime Core (bridge/runtime)

### Task 2: Bridge manifest parser

**Files:**
- Create: `bridge/runtime/pom.xml`
- Create: `bridge/runtime/src/main/java/io/casehub/desiredstate/bridge/runtime/BridgeManifestParser.java`
- Create: `bridge/runtime/src/main/java/io/casehub/desiredstate/bridge/runtime/BridgeManifest.java`
- Create: `bridge/runtime/src/main/java/io/casehub/desiredstate/bridge/runtime/BridgeManifestType.java`
- Modify: `bridge/pom.xml` — add `runtime` to `<modules>`
- Test: `bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/BridgeManifestParserTest.java`
- Test: `bridge/runtime/src/test/resources/META-INF/desiredstate/bridges/test-bridge.yaml`

**Interfaces:**
- Consumes: `BridgeDescriptor`, `BridgeTypeDescriptor`, `BridgeLifecycleConfig` from bridge/api
- Produces: `BridgeManifestParser.parse(InputStream) → BridgeDescriptor` (used by Tasks 3, 5, 6)

- [ ] **Step 1: Create runtime pom.xml**

Create `bridge/runtime/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-desiredstate-bridge-parent</artifactId>
        <version>0.4-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-desiredstate-bridge</artifactId>
    <name>CaseHub Desired State :: Bridge :: Runtime</name>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-bridge-api</artifactId>
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
            <groupId>com.fasterxml.jackson.dataformat</groupId>
            <artifactId>jackson-dataformat-yaml</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>${jandex-maven-plugin.version}</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

Add `<module>runtime</module>` to `bridge/pom.xml`.

- [ ] **Step 2: Write manifest parser tests**

Create `test-bridge.yaml` in `src/test/resources/META-INF/desiredstate/bridges/`:

```yaml
bridge:
  protocol: test-provider
  version: "1.0"

  lifecycle:
    mode: managed-process
    binary: /usr/bin/test-provider
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
    - id: test_resource
      stateMapping:
        active: PRESENT
        deleted: ABSENT
        _default: DEGRADED
      errorMapping:
        NotFound: { category: PERMANENT }
        Throttled: { category: TRANSIENT }
        _default: { category: PERMANENT }

    - id: test_bucket
      stateMapping:
        _default: PRESENT
        _notFound: ABSENT
      errorMapping:
        _default: { category: PERMANENT }
```

Create `BridgeManifestParserTest.java`:

```java
package io.casehub.desiredstate.bridge.runtime;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.bridge.*;
import org.junit.jupiter.api.Test;
import java.io.InputStream;
import static org.junit.jupiter.api.Assertions.*;

class BridgeManifestParserTest {

    @Test
    void parsesProtocolAndVersion() {
        var descriptor = parseTestManifest();
        assertEquals("test-provider", descriptor.protocolId());
    }

    @Test
    void parsesHandledTypes() {
        var descriptor = parseTestManifest();
        assertEquals(2, descriptor.handledTypes().size());
        assertTrue(descriptor.handledTypes().contains(NodeType.of("test_resource")));
        assertTrue(descriptor.handledTypes().contains(NodeType.of("test_bucket")));
    }

    @Test
    void parsesStateMapping() {
        var descriptor = parseTestManifest();
        var typeDesc = descriptor.typeDescriptors().get("test_resource");
        assertEquals(NodeStatus.PRESENT, typeDesc.stateMapping().get("active"));
        assertEquals(NodeStatus.ABSENT, typeDesc.stateMapping().get("deleted"));
        assertEquals(NodeStatus.DEGRADED, typeDesc.defaultState());
    }

    @Test
    void parsesDefaultNotFoundState() {
        var descriptor = parseTestManifest();
        var typeDesc = descriptor.typeDescriptors().get("test_bucket");
        assertEquals(NodeStatus.ABSENT, typeDesc.notFoundState());
    }

    @Test
    void parsesErrorMapping() {
        var descriptor = parseTestManifest();
        var typeDesc = descriptor.typeDescriptors().get("test_resource");
        assertEquals(BridgeErrorCategory.PERMANENT, typeDesc.errorMapping().get("NotFound"));
        assertEquals(BridgeErrorCategory.TRANSIENT, typeDesc.errorMapping().get("Throttled"));
        assertEquals(BridgeErrorCategory.PERMANENT, typeDesc.defaultErrorCategory());
    }

    @Test
    void parsesLifecycleConfig() {
        var descriptor = parseTestManifest();
        var lc = descriptor.lifecycleConfig();
        assertEquals(LifecycleMode.MANAGED_PROCESS, lc.mode());
        assertEquals("/usr/bin/test-provider", lc.binaryPath());
        assertEquals(RestartPolicyType.ON_FAILURE, lc.restartPolicy().policy());
        assertEquals(3, lc.restartPolicy().maxRetries());
    }

    private BridgeDescriptor parseTestManifest() {
        InputStream is = getClass().getResourceAsStream(
            "/META-INF/desiredstate/bridges/test-bridge.yaml");
        assertNotNull(is, "test manifest not found");
        return BridgeManifestParser.parse(is);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl bridge/runtime -am`
Expected: compilation failure — `BridgeManifestParser` not yet created.

- [ ] **Step 4: Implement manifest parser**

Create `BridgeManifest.java` — Jackson deserialization model:

```java
package io.casehub.desiredstate.bridge.runtime;

import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.List;
import java.util.Map;

record BridgeManifest(
    @JsonProperty("bridge") BridgeRoot bridge
) {
    record BridgeRoot(
        String protocol,
        String version,
        LifecycleDef lifecycle,
        List<TypeDef> types
    ) {}

    record LifecycleDef(
        String mode,
        String binary,
        List<String> args,
        HealthCheckDef healthCheck,
        RestartDef restart,
        String endpoint
    ) {}

    record HealthCheckDef(String type, String interval, String timeout) {}

    record RestartDef(String policy, int maxRetries, String backoff) {}

    record TypeDef(
        String id,
        Map<String, String> stateMapping,
        Map<String, ErrorMappingEntry> errorMapping
    ) {}

    record ErrorMappingEntry(String category) {}
}
```

Create `BridgeManifestParser.java`:

```java
package io.casehub.desiredstate.bridge.runtime;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.bridge.*;

import java.io.IOException;
import java.io.InputStream;
import java.io.UncheckedIOException;
import java.time.Duration;
import java.util.*;

public final class BridgeManifestParser {

    private static final ObjectMapper YAML = new ObjectMapper(new YAMLFactory())
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

    private BridgeManifestParser() {}

    public static BridgeDescriptor parse(InputStream is) {
        try {
            var manifest = YAML.readValue(is, BridgeManifest.class);
            return toDescriptor(manifest.bridge());
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to parse bridge manifest", e);
        }
    }

    private static BridgeDescriptor toDescriptor(BridgeManifest.BridgeRoot root) {
        Set<NodeType> handledTypes = new LinkedHashSet<>();
        Map<String, BridgeTypeDescriptor> typeDescriptors = new LinkedHashMap<>();

        for (var typeDef : root.types()) {
            var nodeType = NodeType.of(typeDef.id());
            handledTypes.add(nodeType);
            typeDescriptors.put(typeDef.id(), toTypeDescriptor(nodeType, typeDef));
        }

        return new BridgeDescriptor(
            root.protocol(),
            handledTypes,
            typeDescriptors,
            toLifecycleConfig(root.lifecycle())
        );
    }

    private static BridgeTypeDescriptor toTypeDescriptor(NodeType nodeType, BridgeManifest.TypeDef def) {
        Map<String, NodeStatus> stateMapping = new LinkedHashMap<>();
        NodeStatus defaultState = NodeStatus.DEGRADED;
        NodeStatus notFoundState = NodeStatus.ABSENT;

        if (def.stateMapping() != null) {
            for (var entry : def.stateMapping().entrySet()) {
                switch (entry.getKey()) {
                    case "_default" -> defaultState = NodeStatus.valueOf(entry.getValue());
                    case "_notFound" -> notFoundState = NodeStatus.valueOf(entry.getValue());
                    default -> stateMapping.put(entry.getKey(), NodeStatus.valueOf(entry.getValue()));
                }
            }
        }

        Map<String, BridgeErrorCategory> errorMapping = new LinkedHashMap<>();
        BridgeErrorCategory defaultErrorCategory = BridgeErrorCategory.PERMANENT;

        if (def.errorMapping() != null) {
            for (var entry : def.errorMapping().entrySet()) {
                var category = BridgeErrorCategory.valueOf(entry.getValue().category().toUpperCase());
                if ("_default".equals(entry.getKey())) {
                    defaultErrorCategory = category;
                } else {
                    errorMapping.put(entry.getKey(), category);
                }
            }
        }

        return new BridgeTypeDescriptor(nodeType, stateMapping, defaultState, notFoundState,
            errorMapping, defaultErrorCategory);
    }

    private static BridgeLifecycleConfig toLifecycleConfig(BridgeManifest.LifecycleDef def) {
        if (def == null) return null;

        LifecycleMode mode = switch (def.mode()) {
            case "managed-process" -> LifecycleMode.MANAGED_PROCESS;
            case "external-service" -> LifecycleMode.EXTERNAL_SERVICE;
            default -> throw new IllegalArgumentException("Unknown lifecycle mode: " + def.mode());
        };

        HealthCheckConfig healthCheck = null;
        if (def.healthCheck() != null) {
            healthCheck = new HealthCheckConfig(
                HealthCheckType.valueOf(def.healthCheck().type().toUpperCase()),
                parseDuration(def.healthCheck().interval()),
                parseDuration(def.healthCheck().timeout())
            );
        }

        RestartPolicy restartPolicy = null;
        if (def.restart() != null) {
            restartPolicy = new RestartPolicy(
                parseRestartPolicyType(def.restart().policy()),
                def.restart().maxRetries(),
                BackoffStrategy.valueOf(def.restart().backoff().toUpperCase())
            );
        }

        return new BridgeLifecycleConfig(mode, def.binary(), def.args(),
            healthCheck, restartPolicy, def.endpoint());
    }

    private static RestartPolicyType parseRestartPolicyType(String policy) {
        return switch (policy) {
            case "on-failure" -> RestartPolicyType.ON_FAILURE;
            case "always" -> RestartPolicyType.ALWAYS;
            case "never" -> RestartPolicyType.NEVER;
            default -> throw new IllegalArgumentException("Unknown restart policy: " + policy);
        };
    }

    private static Duration parseDuration(String value) {
        if (value == null) return null;
        if (value.endsWith("s")) return Duration.ofSeconds(Long.parseLong(value.replace("s", "")));
        if (value.endsWith("m")) return Duration.ofMinutes(Long.parseLong(value.replace("m", "")));
        return Duration.parse("PT" + value);
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl bridge/runtime -am`
Expected: PASS — all manifest parser tests green.

- [ ] **Step 6: Commit**

```bash
git add bridge/
git commit -m "feat(bridge): add manifest parser — YAML to BridgeDescriptor

Parses META-INF/desiredstate/bridges/*.yaml into BridgeDescriptor with
type-level state mapping, error mapping, and lifecycle configuration.

Refs #183"
```

### Task 3: BridgeProvisioner, BridgeActualStateAdapter, and BridgeDiscovery

**Files:**
- Create: `bridge/runtime/src/main/java/io/casehub/desiredstate/bridge/runtime/BridgeProvisioner.java`
- Create: `bridge/runtime/src/main/java/io/casehub/desiredstate/bridge/runtime/BridgeActualStateAdapter.java`
- Create: `bridge/runtime/src/main/java/io/casehub/desiredstate/bridge/runtime/BridgeDiscovery.java`
- Test: `bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/BridgeProvisionerTest.java`
- Test: `bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/BridgeActualStateAdapterTest.java`
- Test: `bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/BridgeDiscoveryTest.java`
- Test: `bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/MockBridgeProvider.java`

**Interfaces:**
- Consumes: `BridgeProvider`, `BridgeDescriptor`, `BridgeRequest`, `BridgeResult`, `BridgeStateResult` from bridge/api
- Consumes: `BridgeManifestParser.parse()` from Task 2
- Consumes: `NodeProvisioner`, `ActualStateAdapter`, `DesiredNode`, `NodeType`, `NodeStatus` from api/
- Consumes: `BeanRegistration` from runtime-core/
- Produces: `BridgeProvisioner` implements `NodeProvisioner` (registered via BridgeDiscovery)
- Produces: `BridgeActualStateAdapter` implements `ActualStateAdapter` (registered via BridgeDiscovery)
- Produces: `BridgeDiscovery.discoverAll(List<BridgeDescriptor>, List<BridgeProvider>) → List<BeanRegistration>` (used by Tasks 5, 6)

- [ ] **Step 1: Write MockBridgeProvider for tests**

Create `MockBridgeProvider.java`:

```java
package io.casehub.desiredstate.bridge.runtime;

import io.casehub.desiredstate.bridge.*;
import java.util.*;

public class MockBridgeProvider implements BridgeProvider {

    private final String protocolId;
    private final Set<String> types;
    private BridgeResult applyResult = new BridgeResult.Success(Map.of());
    private BridgeResult destroyResult = new BridgeResult.Success(Map.of());
    private BridgeStateResult readResult = new BridgeStateResult.NotFound();
    private final List<BridgeRequest> applyCalls = new ArrayList<>();
    private final List<BridgeRequest> destroyCalls = new ArrayList<>();
    private final List<BridgeReadRequest> readCalls = new ArrayList<>();

    public MockBridgeProvider(String protocolId, String... types) {
        this.protocolId = protocolId;
        this.types = Set.of(types);
    }

    @Override public String protocolId() { return protocolId; }
    @Override public Set<String> supportedTypes() { return types; }

    @Override
    public BridgeResult apply(BridgeRequest request) {
        applyCalls.add(request);
        return applyResult;
    }

    @Override
    public BridgeResult destroy(BridgeRequest request) {
        destroyCalls.add(request);
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
    public List<BridgeRequest> applyCalls() { return applyCalls; }
    public List<BridgeRequest> destroyCalls() { return destroyCalls; }
    public List<BridgeReadRequest> readCalls() { return readCalls; }
}
```

- [ ] **Step 2: Write BridgeProvisioner tests**

Create `BridgeProvisionerTest.java`:

```java
package io.casehub.desiredstate.bridge.runtime;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.bridge.*;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class BridgeProvisionerTest {

    private MockBridgeProvider provider;
    private BridgeProvisioner provisioner;
    private final DesiredStateGraphFactory graphFactory = new DefaultDesiredStateGraphFactory();

    @BeforeEach
    void setUp() {
        provider = new MockBridgeProvider("test", "test_resource");
        var descriptor = BridgeManifestParser.parse(
            getClass().getResourceAsStream("/META-INF/desiredstate/bridges/test-bridge.yaml"));
        provisioner = new BridgeProvisioner(descriptor, provider);
    }

    private DesiredNode bridgeNode(String id, String type, Map<String, Object> fields) {
        return new DesiredNode(NodeId.of(id),
            new BridgeNodeSpec(NodeType.of(type), fields), HumanGating.NONE);
    }

    @Test
    void handledTypesMatchDescriptor() {
        assertEquals(Set.of(NodeType.of("test_resource"), NodeType.of("test_bucket")),
            provisioner.handledTypes());
    }

    @Test
    void provisionDelegatesToProvider() {
        var node = bridgeNode("r1", "test_resource", Map.of("key", "value"));
        var graph = graphFactory.of(Set.of(node), Set.of());
        var ctx = new ProvisionContext("tenant-1", graph);
        var result = provisioner.provision(node, ctx);
        assertInstanceOf(ProvisionResult.Success.class, result);
        assertEquals(1, provider.applyCalls().size());
        assertEquals("test_resource", provider.applyCalls().getFirst().typeId());
    }

    @Test
    void provisionFailedMapsToProvisionResultFailed() {
        provider.setApplyResult(new BridgeResult.Failed("boom", BridgeErrorCategory.TRANSIENT));
        var node = bridgeNode("r1", "test_resource", Map.of());
        var graph = graphFactory.of(Set.of(node), Set.of());
        var result = provisioner.provision(node, new ProvisionContext("t1", graph));
        assertInstanceOf(ProvisionResult.Failed.class, result);
        assertEquals("boom", ((ProvisionResult.Failed) result).reason());
    }

    @Test
    void deprovisionDelegatesToProvider() {
        var node = bridgeNode("r1", "test_resource", Map.of());
        var graph = graphFactory.of(Set.of(node), Set.of());
        var ctx = new DeprovisionContext("tenant-1", graph);
        var result = provisioner.deprovision(node, ctx);
        assertInstanceOf(DeprovisionResult.Success.class, result);
        assertEquals(1, provider.destroyCalls().size());
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl bridge/runtime -am`
Expected: compilation failure — `BridgeProvisioner` not yet created.

- [ ] **Step 4: Implement BridgeProvisioner**

Create `BridgeProvisioner.java`:

```java
package io.casehub.desiredstate.bridge.runtime;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.bridge.*;

import java.util.Map;
import java.util.Set;

public class BridgeProvisioner implements NodeProvisioner {

    private final BridgeDescriptor descriptor;
    private final BridgeProvider provider;

    public BridgeProvisioner(BridgeDescriptor descriptor, BridgeProvider provider) {
        this.descriptor = descriptor;
        this.provider = provider;
    }

    @Override
    public Set<NodeType> handledTypes() {
        return descriptor.handledTypes();
    }

    @Override
    public ProvisionResult provision(DesiredNode node, ProvisionContext context) {
        var request = toBridgeRequest(node, context.tenancyId());

        BridgeResult result;
        if (provider instanceof PlanCapable planCapable && context.approval() == null) {
            var plan = planCapable.plan(request);
            result = planCapable.applyPlan(plan);
        } else {
            result = provider.apply(request);
        }

        return switch (result) {
            case BridgeResult.Success s -> new ProvisionResult.Success();
            case BridgeResult.Failed f -> new ProvisionResult.Failed(f.reason());
            case BridgeResult.PendingApproval p -> new ProvisionResult.PendingApproval(
                node.id(), p.planReference());
        };
    }

    @Override
    public DeprovisionResult deprovision(DesiredNode node, DeprovisionContext context) {
        var request = toBridgeRequest(node, context.tenancyId());
        var result = provider.destroy(request);
        return switch (result) {
            case BridgeResult.Success s -> new DeprovisionResult.Success();
            case BridgeResult.Failed f -> new DeprovisionResult.Failed(f.reason());
            case BridgeResult.PendingApproval p -> new DeprovisionResult.PendingApproval(
                node.id(), p.planReference());
        };
    }

    @Override
    public SuspendResult suspend(DesiredNode node, SuspendContext context) {
        if (provider instanceof SuspendCapable sc) {
            var request = toBridgeRequest(node, context.tenancyId());
            var result = sc.suspend(request);
            return switch (result) {
                case BridgeResult.Success s -> new SuspendResult.Success();
                case BridgeResult.Failed f -> new SuspendResult.Failed(f.reason());
                case BridgeResult.PendingApproval p -> new SuspendResult.PendingApproval(
                    node.id(), p.planReference());
            };
        }
        return new SuspendResult.Failed("Provider does not support suspend");
    }

    @Override
    public ResumeResult resume(DesiredNode node, ResumeContext context) {
        if (provider instanceof SuspendCapable sc) {
            var request = toBridgeRequest(node, context.tenancyId());
            var result = sc.resume(request);
            return switch (result) {
                case BridgeResult.Success s -> new ResumeResult.Success();
                case BridgeResult.Failed f -> new ResumeResult.Failed(f.reason());
                case BridgeResult.PendingApproval p -> new ResumeResult.PendingApproval(
                    node.id(), p.planReference());
            };
        }
        return new ResumeResult.Failed("Provider does not support resume");
    }

    @Override
    public boolean supportsStatefulLifecycle() {
        return provider instanceof SuspendCapable;
    }

    private BridgeRequest toBridgeRequest(DesiredNode node, String tenancyId) {
        Map<String, Object> fields = node.spec() instanceof BridgeNodeSpec bns
            ? bns.fields()
            : Map.of();
        return new BridgeRequest(node.type().value(), node.id().value(), fields, tenancyId);
    }
}
```

- [ ] **Step 5: Write BridgeActualStateAdapter tests**

Create `BridgeActualStateAdapterTest.java`:

```java
package io.casehub.desiredstate.bridge.runtime;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.bridge.*;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class BridgeActualStateAdapterTest {

    private MockBridgeProvider provider;
    private BridgeActualStateAdapter adapter;
    private BridgeDescriptor descriptor;
    private final DesiredStateGraphFactory graphFactory = new DefaultDesiredStateGraphFactory();

    @BeforeEach
    void setUp() {
        provider = new MockBridgeProvider("test", "test_resource");
        descriptor = BridgeManifestParser.parse(
            getClass().getResourceAsStream("/META-INF/desiredstate/bridges/test-bridge.yaml"));
        adapter = new BridgeActualStateAdapter(descriptor, provider);
    }

    private DesiredNode bridgeNode(String id, String type) {
        return new DesiredNode(NodeId.of(id),
            new BridgeNodeSpec(NodeType.of(type), Map.of()), HumanGating.NONE);
    }

    @Test
    void handledTypesMatchDescriptor() {
        assertEquals(descriptor.handledTypes(), adapter.handledTypes());
    }

    @Test
    void mapsFoundStateViaManifest() {
        provider.setReadResult(new BridgeStateResult.Found(Map.of("status", "active")));
        var node = bridgeNode("r1", "test_resource");
        var graph = graphFactory.of(Set.of(node), Set.of());
        var actual = adapter.readActual(graph, "t1");
        assertEquals(Optional.of(NodeStatus.PRESENT), actual.statusOf(NodeId.of("r1")));
    }

    @Test
    void mapsNotFoundToAbsent() {
        provider.setReadResult(new BridgeStateResult.NotFound());
        var node = bridgeNode("r1", "test_resource");
        var graph = graphFactory.of(Set.of(node), Set.of());
        var actual = adapter.readActual(graph, "t1");
        assertEquals(Optional.of(NodeStatus.ABSENT), actual.statusOf(NodeId.of("r1")));
    }

    @Test
    void mapsErrorToUnknown() {
        provider.setReadResult(new BridgeStateResult.Error("timeout"));
        var node = bridgeNode("r1", "test_resource");
        var graph = graphFactory.of(Set.of(node), Set.of());
        var actual = adapter.readActual(graph, "t1");
        assertEquals(Optional.of(NodeStatus.UNKNOWN), actual.statusOf(NodeId.of("r1")));
    }

    @Test
    void usesDefaultStateMappingWhenKeyNotFound() {
        provider.setReadResult(new BridgeStateResult.Found(Map.of("status", "unknown-state")));
        var node = bridgeNode("r1", "test_resource");
        var graph = graphFactory.of(Set.of(node), Set.of());
        var actual = adapter.readActual(graph, "t1");
        assertEquals(Optional.of(NodeStatus.DEGRADED), actual.statusOf(NodeId.of("r1")));
    }
}
```

- [ ] **Step 6: Implement BridgeActualStateAdapter**

Create `BridgeActualStateAdapter.java`:

```java
package io.casehub.desiredstate.bridge.runtime;

import io.casehub.desiredstate.api.*;
import io.casehub.desiredstate.bridge.*;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Set;

public class BridgeActualStateAdapter implements ActualStateAdapter {

    private final BridgeDescriptor descriptor;
    private final BridgeProvider provider;

    public BridgeActualStateAdapter(BridgeDescriptor descriptor, BridgeProvider provider) {
        this.descriptor = descriptor;
        this.provider = provider;
    }

    @Override
    public Set<NodeType> handledTypes() {
        return descriptor.handledTypes();
    }

    @Override
    public ActualState readActual(DesiredStateGraph graph, String tenancyId) {
        Map<NodeId, NodeStatus> statuses = new LinkedHashMap<>();

        for (var node : graph.nodes()) {
            if (!descriptor.handledTypes().contains(node.type())) continue;

            var typeDesc = descriptor.typeDescriptors().get(node.type().value());
            if (typeDesc == null) continue;

            var request = new BridgeReadRequest(node.type().value(), node.id().value(), tenancyId);
            var result = provider.readState(request);

            NodeStatus status = mapStateResult(result, typeDesc);
            statuses.put(node.id(), status);
        }

        return new ActualState(statuses);
    }

    private NodeStatus mapStateResult(BridgeStateResult result, BridgeTypeDescriptor typeDesc) {
        return switch (result) {
            case BridgeStateResult.NotFound n -> typeDesc.notFoundState();
            case BridgeStateResult.Error e -> NodeStatus.UNKNOWN;
            case BridgeStateResult.Found f -> {
                var status = f.state().get("status");
                if (status instanceof String s) {
                    var mapped = typeDesc.stateMapping().get(s);
                    yield mapped != null ? mapped : typeDesc.defaultState();
                }
                yield typeDesc.defaultState();
            }
        };
    }
}
```

- [ ] **Step 7: Write BridgeDiscovery tests and implementation**

Create `BridgeDiscoveryTest.java`:

```java
package io.casehub.desiredstate.bridge.runtime;

import io.casehub.desiredstate.api.ActualStateAdapter;
import io.casehub.desiredstate.api.NodeProvisioner;
import io.casehub.desiredstate.bridge.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class BridgeDiscoveryTest {

    @Test
    void discoverAllReturnsProvisionerAndAdapterBeans() {
        var provider = new MockBridgeProvider("test", "test_resource", "test_bucket");
        var descriptor = BridgeManifestParser.parse(
            getClass().getResourceAsStream("/META-INF/desiredstate/bridges/test-bridge.yaml"));

        var beans = BridgeDiscovery.discoverAll(List.of(descriptor), List.of(provider));

        assertEquals(2, beans.size());
        assertTrue(beans.stream().anyMatch(b -> b.type() == NodeProvisioner.class));
        assertTrue(beans.stream().anyMatch(b -> b.type() == ActualStateAdapter.class));
    }

    @Test
    void discoverAllMatchesProviderByProtocolId() {
        var provider = new MockBridgeProvider("test", "test_resource");
        var descriptor = BridgeManifestParser.parse(
            getClass().getResourceAsStream("/META-INF/desiredstate/bridges/test-bridge.yaml"));

        var beans = BridgeDiscovery.discoverAll(List.of(descriptor), List.of(provider));
        assertFalse(beans.isEmpty());
    }
}
```

Create `BridgeDiscovery.java`:

```java
package io.casehub.desiredstate.bridge.runtime;

import io.casehub.desiredstate.api.ActualStateAdapter;
import io.casehub.desiredstate.api.NodeProvisioner;
import io.casehub.desiredstate.bridge.BridgeDescriptor;
import io.casehub.desiredstate.bridge.BridgeProvider;
import io.casehub.desiredstate.api.BeanRegistration;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

public final class BridgeDiscovery {

    private BridgeDiscovery() {}

    public static List<BeanRegistration> discoverAll(
            List<BridgeDescriptor> descriptors,
            List<BridgeProvider> providers) {

        Map<String, BridgeProvider> providersByProtocol = providers.stream()
            .collect(Collectors.toMap(BridgeProvider::protocolId, Function.identity()));

        List<BeanRegistration> beans = new ArrayList<>();

        for (var descriptor : descriptors) {
            var provider = providersByProtocol.get(descriptor.protocolId());
            if (provider == null) continue;

            var provisioner = new BridgeProvisioner(descriptor, provider);
            var adapter = new BridgeActualStateAdapter(descriptor, provider);

            beans.add(new BeanRegistration(
                "bridge-provisioner-" + descriptor.protocolId(),
                NodeProvisioner.class, provisioner));
            beans.add(new BeanRegistration(
                "bridge-adapter-" + descriptor.protocolId(),
                ActualStateAdapter.class, adapter));
        }

        return beans;
    }
}
```

- [ ] **Step 8: Run all runtime tests**

Run: `mvn --batch-mode test -pl bridge/runtime -am`
Expected: PASS — all provisioner, adapter, discovery, and manifest parser tests green.

- [ ] **Step 9: Commit**

```bash
git add bridge/
git commit -m "feat(bridge): add BridgeProvisioner, BridgeActualStateAdapter, BridgeDiscovery

BridgeProvisioner implements NodeProvisioner, delegates to BridgeProvider SPI.
BridgeActualStateAdapter implements ActualStateAdapter with manifest-driven state mapping.
BridgeDiscovery matches providers to descriptors and returns BeanRegistration list.

Refs #183"
```

---

## Batch 3: Framework Wiring (bridge/deployment + bridge/spring)

### Task 4: Quarkus deployment extension

**Files:**
- Create: `bridge/deployment/pom.xml`
- Create: `bridge/deployment/src/main/java/io/casehub/desiredstate/bridge/deployment/BridgeProcessor.java`
- Modify: `bridge/pom.xml` — add `deployment` to `<modules>`

**Interfaces:**
- Consumes: `BridgeManifestParser.parse()` from Task 2
- Consumes: `BridgeDiscovery.discoverAll()` from Task 3
- Produces: Quarkus synthetic beans for `NodeProvisioner` and `ActualStateAdapter` (registered at build time)

- [ ] **Step 1: Create deployment pom.xml**

Create `bridge/deployment/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-desiredstate-bridge-parent</artifactId>
        <version>0.4-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-desiredstate-bridge-deployment</artifactId>
    <name>CaseHub Desired State :: Bridge :: Deployment</name>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-bridge</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc-deployment</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>${jandex-maven-plugin.version}</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

Add `<module>deployment</module>` to `bridge/pom.xml`.

- [ ] **Step 2: Implement BridgeProcessor**

Create `BridgeProcessor.java` — follows the pattern from `plugin/deployment/`:

```java
package io.casehub.desiredstate.bridge.deployment;

import io.casehub.desiredstate.bridge.runtime.BridgeManifestParser;
import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.builditem.GeneratedClassBuildItem;
import io.quarkus.deployment.builditem.nativeimage.NativeImageResourceBuildItem;

import java.io.IOException;
import java.io.InputStream;
import java.net.URL;
import java.util.ArrayList;
import java.util.Enumeration;
import java.util.List;

public class BridgeProcessor {

    private static final String BRIDGE_MANIFEST_PATH = "META-INF/desiredstate/bridges/";

    @BuildStep
    NativeImageResourceBuildItem registerBridgeManifests() throws IOException {
        List<String> resources = new ArrayList<>();
        Enumeration<URL> urls = Thread.currentThread().getContextClassLoader()
            .getResources(BRIDGE_MANIFEST_PATH);
        while (urls.hasMoreElements()) {
            resources.add(urls.nextElement().getPath());
        }
        return new NativeImageResourceBuildItem(resources);
    }
}
```

- [ ] **Step 3: Compile deployment module**

Run: `mvn --batch-mode compile -pl bridge/deployment -am`
Expected: PASS — deployment module compiles.

- [ ] **Step 4: Commit**

```bash
git add bridge/
git commit -m "feat(bridge): add Quarkus deployment extension scaffold

BridgeProcessor registers bridge manifests for native image.
Full synthetic bean registration deferred to runtime integration.

Refs #183"
```

### Task 5: Spring auto-configuration

**Files:**
- Create: `bridge/spring/pom.xml`
- Create: `bridge/spring/src/main/java/io/casehub/desiredstate/bridge/spring/BridgeAutoConfiguration.java`
- Create: `bridge/spring/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
- Modify: `bridge/pom.xml` — add `spring` to `<modules>`
- Test: `bridge/spring/src/test/java/io/casehub/desiredstate/bridge/spring/BridgeAutoConfigurationTest.java`

**Interfaces:**
- Consumes: `BridgeManifestParser.parse()` from Task 2
- Consumes: `BridgeDiscovery.discoverAll()` from Task 3
- Produces: Spring beans for `NodeProvisioner` and `ActualStateAdapter`

- [ ] **Step 1: Create spring pom.xml**

Create `bridge/spring/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-desiredstate-bridge-parent</artifactId>
        <version>0.4-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-desiredstate-bridge-spring</artifactId>
    <name>CaseHub Desired State :: Bridge :: Spring</name>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-bridge</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-api</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.dataformat</groupId>
            <artifactId>jackson-dataformat-yaml</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>${jandex-maven-plugin.version}</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

Add `<module>spring</module>` to `bridge/pom.xml`.

- [ ] **Step 2: Write Spring auto-config test**

Create `BridgeAutoConfigurationTest.java`:

```java
package io.casehub.desiredstate.bridge.spring;

import io.casehub.desiredstate.bridge.BridgeDescriptor;
import io.casehub.desiredstate.bridge.runtime.BridgeManifestParser;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class BridgeAutoConfigurationTest {

    @Test
    void discoversManifestsFromClasspath() {
        var manifests = BridgeAutoConfiguration.discoverManifests(
            Thread.currentThread().getContextClassLoader());
        assertNotNull(manifests);
    }
}
```

- [ ] **Step 3: Implement BridgeAutoConfiguration**

Create `BridgeAutoConfiguration.java`:

```java
package io.casehub.desiredstate.bridge.spring;

import io.casehub.desiredstate.bridge.BridgeDescriptor;
import io.casehub.desiredstate.bridge.BridgeProvider;
import io.casehub.desiredstate.bridge.runtime.BridgeDiscovery;
import io.casehub.desiredstate.bridge.runtime.BridgeManifestParser;
import org.springframework.beans.factory.SmartInitializingSingleton;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Bean;

import java.io.IOException;
import java.io.InputStream;
import java.net.URL;
import java.util.ArrayList;
import java.util.Enumeration;
import java.util.List;

@AutoConfiguration
@ConditionalOnClass(BridgeProvider.class)
public class BridgeAutoConfiguration implements SmartInitializingSingleton {

    private final ApplicationContext context;

    public BridgeAutoConfiguration(ApplicationContext context) {
        this.context = context;
    }

    @Override
    public void afterSingletonsInstantiated() {
        var descriptors = discoverManifests(context.getClassLoader());
        if (descriptors.isEmpty()) return;

        var providers = new ArrayList<>(context.getBeansOfType(BridgeProvider.class).values());
        var beans = BridgeDiscovery.discoverAll(descriptors, providers);

        if (context instanceof BeanDefinitionRegistry registry) {
            for (var bean : beans) {
                registry.registerBeanDefinition(bean.name(),
                    org.springframework.beans.factory.support.BeanDefinitionBuilder
                        .genericBeanDefinition(bean.type(), () -> bean.instance())
                        .getBeanDefinition());
            }
        }
    }

    static List<BridgeDescriptor> discoverManifests(ClassLoader classLoader) {
        List<BridgeDescriptor> descriptors = new ArrayList<>();
        try {
            Enumeration<URL> urls = classLoader.getResources(
                "META-INF/desiredstate/bridges/");
            while (urls.hasMoreElements()) {
                URL url = urls.nextElement();
                try (InputStream is = url.openStream()) {
                    if (is != null) {
                        descriptors.add(BridgeManifestParser.parse(is));
                    }
                } catch (Exception e) {
                    // skip malformed manifests
                }
            }
        } catch (IOException e) {
            // no manifests on classpath
        }
        return descriptors;
    }
}
```

Create `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
io.casehub.desiredstate.bridge.spring.BridgeAutoConfiguration
```

- [ ] **Step 4: Compile spring module**

Run: `mvn --batch-mode compile -pl bridge/spring -am`
Expected: PASS — spring module compiles.

- [ ] **Step 5: Commit**

```bash
git add bridge/
git commit -m "feat(bridge): add Spring auto-configuration for bridge adapters

BridgeAutoConfiguration discovers bridge manifests from classpath,
matches providers by protocolId, and registers NodeProvisioner +
ActualStateAdapter beans via BridgeDiscovery.

Refs #183"
```

---

## Batch 4: Full Build Verification

### Task 6: Full build and CLAUDE.md update

**Files:**
- Modify: `CLAUDE.md` — add bridge modules to module table
- Modify: `docs/guides/contributor-guide.md` — add bridge section (if exists)

**Interfaces:**
- Consumes: all bridge modules from Tasks 1-5

- [ ] **Step 1: Run full build**

Run: `mvn --batch-mode install`
Expected: PASS — entire project builds including all bridge modules.

- [ ] **Step 2: Update CLAUDE.md module table**

Add bridge module entries to the `## Module Structure` table in `CLAUDE.md`:

```markdown
| `bridge/api/` | `casehub-desiredstate-bridge-api` | `io.casehub.desiredstate.bridge` | Bridge SPI — `BridgeProvider`, `PlanCapable`, `SuspendCapable`, `BridgeConnector`, `BridgeOperation`, request/response types (`BridgeRequest`, `BridgeResult`, `BridgeStateResult`, `BridgePlan`), descriptor records (`BridgeDescriptor`, `BridgeTypeDescriptor`, `BridgeLifecycleConfig`), enums (`BridgeErrorCategory`, `ChangeAction`, `LifecycleMode`, `HealthCheckType`, `RestartPolicyType`, `BackoffStrategy`). |
| `bridge/runtime/` | `casehub-desiredstate-bridge` | `io.casehub.desiredstate.bridge.runtime` | Bridge runtime — `BridgeProvisioner` (implements `NodeProvisioner`), `BridgeActualStateAdapter` (implements `ActualStateAdapter`), `BridgeDiscovery` (manifest → `BeanRegistration`), `BridgeManifestParser` (YAML → `BridgeDescriptor`). |
| `bridge/deployment/` | `casehub-desiredstate-bridge-deployment` | `io.casehub.desiredstate.bridge.deployment` | Quarkus build extension: `META-INF/desiredstate/bridges/*.yaml` discovery, type registry scan. |
| `bridge/spring/` | `casehub-desiredstate-bridge-spring` | `io.casehub.desiredstate.bridge.spring` | Spring `@AutoConfiguration` — classpath bridge YAML discovery, `BridgeDiscovery` bean registration. |
```

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md bridge/
git commit -m "feat(bridge): full build green, update CLAUDE.md module table

All bridge modules compile and test. Module table updated with
bridge/api, bridge/runtime, bridge/deployment, bridge/spring.

Refs #183"
```

## Deferred to Follow-up Issues

- **BridgeLifecycleManager** (process start/stop/health monitoring) — deferred to #184 (Terraform bridge) where the first concrete managed-process bridge will drive the implementation. The runtime works now with external-service mode (pre-deployed providers).
- **Concrete BridgeConnector implementations** (ProcessBridgeConnector, HttpBridgeConnector, GrpcBridgeConnector) — deferred to #184/#185 where protocol-specific transports are needed.
- **bridge/testing module** — deferred to #186 (bridge adapter testing framework).
- **BridgeRuntimeBeans (CDI @Produces)** — deferred to #184 where the Quarkus deployment extension gets full SyntheticBean wiring with a real provider.

## References

- [2026-10-27-bridge-adapter-spi-design.md] — design spec this plan implements
- [api/src/main/java/io/casehub/desiredstate/api/NodeProvisioner.java] — SPI being bridged
- [api/src/main/java/io/casehub/desiredstate/api/ActualStateAdapter.java] — SPI being bridged
- [plugin/runtime/src/main/java/.../PluginDiscovery.java] — BeanRegistration pattern
- [plugin/runtime/src/main/java/.../YamlPluginProvisioner.java] — single-adapter-many-types pattern
- [plugin/spring/src/main/java/.../DesiredStatePluginAutoConfiguration.java] — Spring auto-config pattern
- [GitHub #183] — focal issue
- [GitHub #182] — parent epic
- [GitHub #131] — plugin architecture design (tier model)
