# SuspendCapable Bridge Adapters Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #209 — SuspendCapable support for Terraform and Ansible bridge adapters
**Issue group:** #207, #208, #209 (bridge chapter)

**Goal:** Implement `SuspendCapable` on both `TerraformBridgeProvider` and
`AnsibleBridgeProvider` so bridge-managed resources can be suspended and
resumed through the existing runtime routing.

**Architecture:** Each adapter adds `SuspendCapable` to its `implements`
clause with per-type configuration controlling which resource types
support suspend/resume. Ansible uses module+state-param config in
`AnsibleModuleConfig`; Terraform uses an attribute-triplet config in a
new `TerraformSuspendConfig` record. Both read config from manifest YAML
extensions.

**Tech Stack:** Java 21, JUnit 5, AssertJ, Jackson YAML

## Global Constraints

- No changes to `bridge/api/SuspendCapable.java` or
  `bridge/runtime/BridgeProvisioner.java` — routing already handles
  `instanceof SuspendCapable`
- Unsupported types return `BridgeResult.Failed` with
  `BridgeErrorCategory.CONFIGURATION`
- Tests use mock process executors / protocols (no real subprocess calls)

---

## Batch 1: Ansible SuspendCapable

### Task 1: Extend AnsibleModuleConfig and manifest parser with suspend fields

**Files:**
- Modify: `bridge/ansible/src/main/java/io/casehub/desiredstate/bridge/ansible/AnsibleModuleConfig.java`
- Modify: `bridge/ansible/src/main/java/io/casehub/desiredstate/bridge/ansible/AnsibleManifestExtensionParser.java`
- Modify: `bridge/ansible/src/test/resources/META-INF/desiredstate/bridges/test-ansible.yaml`
- Modify: `bridge/ansible/src/test/java/io/casehub/desiredstate/bridge/ansible/AnsibleManifestExtensionParserTest.java`

**Interfaces:**
- Produces: `AnsibleModuleConfig` with six new fields — `suspendModule()`,
  `suspendStateParam()`, `suspendStateValue()`, `resumeModule()`,
  `resumeStateParam()`, `resumeStateValue()`. All nullable. Helper
  `fqSuspendModule()` returns `collection + "." + suspendModule` (null
  if `suspendModule` is null). `supportsSuspend()` returns
  `suspendModule != null`.

- [ ] **Step 1: Write the failing test for suspend field parsing**

Add to `AnsibleManifestExtensionParserTest`:

```java
@Test
void parsesSuspendResumeConfig() {
    var configs = AnsibleManifestExtensionParser.parse(
        getClass().getResourceAsStream("/META-INF/desiredstate/bridges/test-ansible.yaml"));

    var ec2 = configs.get("ansible:amazon.aws.ec2_instance");
    assertThat(ec2.suspendModule()).isEqualTo("ec2_instance");
    assertThat(ec2.suspendStateParam()).isEqualTo("state");
    assertThat(ec2.suspendStateValue()).isEqualTo("stopped");
    assertThat(ec2.resumeModule()).isNull();
    assertThat(ec2.resumeStateParam()).isNull();
    assertThat(ec2.resumeStateValue()).isEqualTo("running");
    assertThat(ec2.supportsSuspend()).isTrue();
    assertThat(ec2.fqSuspendModule()).isEqualTo("amazon.aws.ec2_instance");
}

@Test
void typeWithoutSuspendConfigReturnsFalseForSupportsSuspend() {
    var configs = AnsibleManifestExtensionParser.parse(
        getClass().getResourceAsStream("/META-INF/desiredstate/bridges/test-ansible.yaml"));

    var s3 = configs.get("ansible:amazon.aws.s3_bucket");
    assertThat(s3.supportsSuspend()).isFalse();
    assertThat(s3.suspendModule()).isNull();
    assertThat(s3.fqSuspendModule()).isNull();
}

@Test
void resumeModuleDefaultsToSuspendModule() {
    var configs = AnsibleManifestExtensionParser.parse(
        getClass().getResourceAsStream("/META-INF/desiredstate/bridges/test-ansible.yaml"));

    var ec2 = configs.get("ansible:amazon.aws.ec2_instance");
    assertThat(ec2.effectiveResumeModule()).isEqualTo("ec2_instance");
    assertThat(ec2.fqResumeModule()).isEqualTo("amazon.aws.ec2_instance");
    assertThat(ec2.effectiveResumeStateParam()).isEqualTo("state");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl bridge/ansible -Dtest=AnsibleManifestExtensionParserTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — new fields and methods do not exist on `AnsibleModuleConfig`

- [ ] **Step 3: Add suspend/resume fields to test YAML manifest**

Add suspend config to the ec2_instance entry in
`bridge/ansible/src/test/resources/META-INF/desiredstate/bridges/test-ansible.yaml`:

```yaml
    - id: ansible:amazon.aws.ec2_instance
      ansible:
        collection: amazon.aws
        provision_module: ec2_instance
        deprovision_strategy: state_absent
        info_module: ec2_instance_info
        info_filter_key: instance_ids
        result_key: instances
        output_keys:
          - instance_id
          - public_ip_address
          - instance_type
        drift_keys:
          - instance_type
          - security_groups
        suspend_module: ec2_instance
        suspend_state_param: state
        suspend_state_value: stopped
        resume_state_value: running
```

S3 bucket entry stays unchanged (no suspend support).

- [ ] **Step 4: Extend AnsibleModuleConfig record**

Add six new fields to the record. Add `supportsSuspend()`,
`fqSuspendModule()`, `fqResumeModule()`, `effectiveResumeModule()`,
`effectiveResumeStateParam()` helper methods:

```java
public record AnsibleModuleConfig(
    String collection,
    String provisionModule,
    String deprovisionStrategy,
    String deprovisionModule,
    String infoModule,
    String infoFilterKey,
    String resultKey,
    List<String> outputKeys,
    List<String> driftKeys,
    String suspendModule,
    String suspendStateParam,
    String suspendStateValue,
    String resumeModule,
    String resumeStateParam,
    String resumeStateValue
) {
    public AnsibleModuleConfig {
        if (deprovisionStrategy == null) deprovisionStrategy = "state_absent";
        if (outputKeys == null) outputKeys = List.of();
        if (driftKeys == null) driftKeys = List.of();
    }

    public String fqProvisionModule() {
        return collection + "." + provisionModule;
    }

    public String fqInfoModule() {
        return infoModule != null ? collection + "." + infoModule : null;
    }

    public String fqDeprovisionModule() {
        return deprovisionModule != null ? collection + "." + deprovisionModule : null;
    }

    public boolean supportsSuspend() {
        return suspendModule != null;
    }

    public String fqSuspendModule() {
        return suspendModule != null ? collection + "." + suspendModule : null;
    }

    public String effectiveResumeModule() {
        return resumeModule != null ? resumeModule : suspendModule;
    }

    public String fqResumeModule() {
        String effective = effectiveResumeModule();
        return effective != null ? collection + "." + effective : null;
    }

    public String effectiveResumeStateParam() {
        return resumeStateParam != null ? resumeStateParam : suspendStateParam;
    }
}
```

- [ ] **Step 5: Update AnsibleManifestExtensionParser to read suspend fields**

In `toModuleConfig()`, add parsing for the six new fields:

```java
private static AnsibleModuleConfig toModuleConfig(JsonNode node) {
    return new AnsibleModuleConfig(
        node.path("collection").asText(null),
        node.path("provision_module").asText(null),
        node.path("deprovision_strategy").asText("state_absent"),
        node.path("deprovision_module").asText(null),
        node.path("info_module").asText(null),
        node.path("info_filter_key").asText(null),
        node.path("result_key").asText(null),
        toStringList(node.path("output_keys")),
        toStringList(node.path("drift_keys")),
        node.path("suspend_module").asText(null),
        node.path("suspend_state_param").asText(null),
        node.path("suspend_state_value").asText(null),
        node.path("resume_module").asText(null),
        node.path("resume_state_param").asText(null),
        node.path("resume_state_value").asText(null)
    );
}
```

- [ ] **Step 6: Fix all existing AnsibleModuleConfig constructor calls**

Every existing call to `new AnsibleModuleConfig(...)` in tests now needs
six additional `null` arguments. Update:
- `AnsibleBridgeProviderContractTest.java:22`
- `AnsiblePlanCapableContractTest.java:21`
- `AnsibleImportCapableContractTest.java` (find the constructor call)
- Any other test creating `AnsibleModuleConfig` directly

Append `, null, null, null, null, null, null` to each constructor call.

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl bridge/ansible`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add bridge/ansible/
git commit -m "feat(#209): extend AnsibleModuleConfig and parser with suspend/resume fields"
```

### Task 2: Add PlaybookGenerator suspend/resume methods and implement SuspendCapable on AnsibleBridgeProvider

**Files:**
- Modify: `bridge/ansible/src/main/java/io/casehub/desiredstate/bridge/ansible/PlaybookGenerator.java`
- Modify: `bridge/ansible/src/main/java/io/casehub/desiredstate/bridge/ansible/AnsibleBridgeProvider.java`
- Modify: `bridge/ansible/src/test/java/io/casehub/desiredstate/bridge/ansible/PlaybookGeneratorTest.java`
- Create: `bridge/ansible/src/test/java/io/casehub/desiredstate/bridge/ansible/AnsibleSuspendCapableContractTest.java`

**Interfaces:**
- Consumes: `AnsibleModuleConfig.supportsSuspend()`, `fqSuspendModule()`,
  `fqResumeModule()`, `suspendStateParam()`, `suspendStateValue()`,
  `effectiveResumeStateParam()`, `resumeStateValue()` from Task 1
- Produces: `AnsibleBridgeProvider implements SuspendCapable` with
  `suspend(BridgeRequest)` and `resume(BridgeRequest)`

- [ ] **Step 1: Write failing test for PlaybookGenerator.suspend()**

Add to `PlaybookGeneratorTest`:

```java
@Test
void suspendPlaybookContainsModuleAndStateParam() {
    String yaml = PlaybookGenerator.suspend(
        "amazon.aws.ec2_instance",
        Map.of("instance_type", "t3.micro"),
        "state", "stopped");

    assertThat(yaml).contains("amazon.aws.ec2_instance:");
    assertThat(yaml).contains("state:");
    assertThat(yaml).contains("stopped");
    assertThat(yaml).contains("instance_type:");
    assertThat(yaml).contains("gather_facts: false");
    assertThat(yaml).contains("register:");
}

@Test
void resumePlaybookContainsModuleAndStateParam() {
    String yaml = PlaybookGenerator.resume(
        "amazon.aws.ec2_instance",
        Map.of("instance_type", "t3.micro"),
        "state", "running");

    assertThat(yaml).contains("amazon.aws.ec2_instance:");
    assertThat(yaml).contains("state:");
    assertThat(yaml).contains("running");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl bridge/ansible -Dtest=PlaybookGeneratorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `suspend()` and `resume()` methods do not exist

- [ ] **Step 3: Implement PlaybookGenerator.suspend() and resume()**

Add to `PlaybookGenerator`:

```java
public static String suspend(String fqModule, Map<String, Object> params,
                              String stateParam, String stateValue) {
    Map<String, Object> taskParams = new LinkedHashMap<>(params);
    taskParams.put(stateParam, stateValue);
    return buildPlaybook("suspend", fqModule, taskParams, false);
}

public static String resume(String fqModule, Map<String, Object> params,
                             String stateParam, String stateValue) {
    Map<String, Object> taskParams = new LinkedHashMap<>(params);
    taskParams.put(stateParam, stateValue);
    return buildPlaybook("resume", fqModule, taskParams, false);
}
```

- [ ] **Step 4: Run PlaybookGenerator tests to verify they pass**

Run: `mvn --batch-mode test -pl bridge/ansible -Dtest=PlaybookGeneratorTest`
Expected: ALL PASS

- [ ] **Step 5: Write the TCK contract test for AnsibleBridgeProvider**

Create `AnsibleSuspendCapableContractTest.java`:

```java
package io.casehub.desiredstate.bridge.ansible;

import io.casehub.desiredstate.bridge.BridgeRequest;
import io.casehub.desiredstate.bridge.SuspendCapable;
import io.casehub.desiredstate.bridge.testing.SuspendCapableContractTest;

import java.util.List;
import java.util.Map;

class AnsibleSuspendCapableContractTest extends SuspendCapableContractTest {

    private static final String SUCCESS_JSON = """
        {"plays":[{"play":{"name":"target"},"tasks":[{"task":{"name":"suspend"},
        "hosts":{"target":{"changed":true,"msg":""}}}]}],
        "stats":{"target":{"ok":1,"changed":1,"unreachable":0,"failed":0}}}""";

    @Override
    protected SuspendCapable createSuspendCapableProvider() {
        return new AnsibleBridgeProvider(
            Map.of("ansible:test.collection.resource",
                new AnsibleModuleConfig("test.collection", "resource", "state_absent",
                    null, null, null, null, List.of(), List.of(),
                    "resource", "state", "stopped", null, null, "running")),
            "ansible-playbook",
            (workDir, cmd, env, timeout) -> new AnsibleProcessResult(0, SUCCESS_JSON, ""));
    }

    @Override
    protected BridgeRequest sampleSuspendRequest() {
        return new BridgeRequest("ansible:test.collection.resource", "r-1",
            Map.of("param1", "val1"), "t-1");
    }
}
```

- [ ] **Step 6: Run contract test to verify it fails**

Run: `mvn --batch-mode test -pl bridge/ansible -Dtest=AnsibleSuspendCapableContractTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `AnsibleBridgeProvider` does not implement `SuspendCapable`

- [ ] **Step 7: Implement SuspendCapable on AnsibleBridgeProvider**

Change the class declaration:

```java
public class AnsibleBridgeProvider implements BridgeProvider, ImportCapable, PlanCapable, SuspendCapable {
```

Add the `suspend()` and `resume()` methods:

```java
@Override
public BridgeResult suspend(BridgeRequest request) {
    var config = requireConfig(request.typeId());
    if (!config.supportsSuspend()) {
        return new BridgeResult.Failed(
            "Suspend not supported for type: " + request.typeId(),
            BridgeErrorCategory.CONFIGURATION);
    }
    var specParts = splitSpec(request.spec());
    String playbook = PlaybookGenerator.suspend(
        config.fqSuspendModule(), specParts.moduleParams,
        config.suspendStateParam(), config.suspendStateValue());

    var pbResult = executePlaybook(playbook, specParts, request.tenancyId());
    if (pbResult.succeeded()) {
        return new BridgeResult.Success(Map.of());
    }
    return new BridgeResult.Failed(pbResult.taskResult().msg(),
        AnsibleErrorClassifier.classify(pbResult.taskResult()));
}

@Override
public BridgeResult resume(BridgeRequest request) {
    var config = requireConfig(request.typeId());
    if (!config.supportsSuspend()) {
        return new BridgeResult.Failed(
            "Resume not supported for type: " + request.typeId(),
            BridgeErrorCategory.CONFIGURATION);
    }
    var specParts = splitSpec(request.spec());
    String playbook = PlaybookGenerator.resume(
        config.fqResumeModule(), specParts.moduleParams,
        config.effectiveResumeStateParam(), config.resumeStateValue());

    var pbResult = executePlaybook(playbook, specParts, request.tenancyId());
    if (pbResult.succeeded()) {
        return new BridgeResult.Success(Map.of());
    }
    return new BridgeResult.Failed(pbResult.taskResult().msg(),
        AnsibleErrorClassifier.classify(pbResult.taskResult()));
}
```

Add the import for `SuspendCapable`.

- [ ] **Step 8: Write test for unsupported type returning Failed**

Add to `AnsibleSuspendCapableContractTest` or a new test in
`AnsibleBridgeProviderTest` (if one exists — otherwise add inline):

```java
@Test
void suspendReturnsFailedForUnsupportedType() {
    var provider = new AnsibleBridgeProvider(
        Map.of("ansible:test.collection.resource",
            new AnsibleModuleConfig("test.collection", "resource", "state_absent",
                null, null, null, null, List.of(), List.of(),
                null, null, null, null, null, null)),
        "ansible-playbook",
        (workDir, cmd, env, timeout) -> new AnsibleProcessResult(0, "{}", ""));

    var request = new BridgeRequest("ansible:test.collection.resource", "r-1",
        Map.of(), "t-1");
    var result = ((SuspendCapable) provider).suspend(request);

    assertThat(result).isInstanceOf(BridgeResult.Failed.class);
    assertThat(((BridgeResult.Failed) result).reason()).contains("Suspend not supported");
}
```

- [ ] **Step 9: Run all Ansible bridge tests**

Run: `mvn --batch-mode test -pl bridge/ansible`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```bash
git add bridge/ansible/
git commit -m "feat(#209): implement SuspendCapable on AnsibleBridgeProvider"
```

## Batch 2: Terraform SuspendCapable

### Task 3: Create TerraformSuspendConfig and manifest parser

**Files:**
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/TerraformSuspendConfig.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/TerraformManifestExtensionParser.java`
- Create: `bridge/terraform/src/test/resources/META-INF/desiredstate/bridges/test-terraform.yaml`
- Create: `bridge/terraform/src/test/java/io/casehub/desiredstate/bridge/terraform/TerraformManifestExtensionParserTest.java`

**Interfaces:**
- Produces: `TerraformSuspendConfig(String attribute, String suspendValue, String resumeValue)`.
  `TerraformManifestExtensionParser.parse(InputStream) → Map<String, TerraformSuspendConfig>`
  keyed by type ID.

- [ ] **Step 1: Write the failing test for manifest parser**

Create `TerraformManifestExtensionParserTest.java`:

```java
package io.casehub.desiredstate.bridge.terraform;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class TerraformManifestExtensionParserTest {

    @Test
    void parsesSuspendConfigForEc2Instance() {
        var configs = TerraformManifestExtensionParser.parse(
            getClass().getResourceAsStream(
                "/META-INF/desiredstate/bridges/test-terraform.yaml"));

        assertThat(configs).containsKey("tf:hashicorp/aws:aws_instance");
        var config = configs.get("tf:hashicorp/aws:aws_instance");
        assertThat(config.attribute()).isEqualTo("instance_state");
        assertThat(config.suspendValue()).isEqualTo("stopped");
        assertThat(config.resumeValue()).isEqualTo("running");
    }

    @Test
    void typeWithoutTerraformBlockIsNotIncluded() {
        var configs = TerraformManifestExtensionParser.parse(
            getClass().getResourceAsStream(
                "/META-INF/desiredstate/bridges/test-terraform.yaml"));

        assertThat(configs).doesNotContainKey("tf:hashicorp/aws:aws_s3_bucket");
    }

    @Test
    void parsesMultipleSuspendableTypes() {
        var configs = TerraformManifestExtensionParser.parse(
            getClass().getResourceAsStream(
                "/META-INF/desiredstate/bridges/test-terraform.yaml"));

        assertThat(configs).containsKey("tf:hashicorp/aws:aws_rds_cluster");
        var rds = configs.get("tf:hashicorp/aws:aws_rds_cluster");
        assertThat(rds.attribute()).isEqualTo("engine_lifecycle_support");
        assertThat(rds.suspendValue()).isEqualTo("stop");
        assertThat(rds.resumeValue()).isEqualTo("start");
    }
}
```

- [ ] **Step 2: Create test YAML manifest**

Create `bridge/terraform/src/test/resources/META-INF/desiredstate/bridges/test-terraform.yaml`:

```yaml
bridge:
  protocol: terraform
  version: "1.0"

  types:
    - id: tf:hashicorp/aws:aws_instance
      terraform:
        suspend_attribute: instance_state
        suspend_value: stopped
        resume_value: running
      stateMapping:
        _default: PRESENT
        _notFound: ABSENT

    - id: tf:hashicorp/aws:aws_s3_bucket
      stateMapping:
        _default: PRESENT
        _notFound: ABSENT

    - id: tf:hashicorp/aws:aws_rds_cluster
      terraform:
        suspend_attribute: engine_lifecycle_support
        suspend_value: stop
        resume_value: start
      stateMapping:
        _default: PRESENT
        _notFound: ABSENT
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn --batch-mode test -pl bridge/terraform -Dtest=TerraformManifestExtensionParserTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — classes do not exist

- [ ] **Step 4: Create TerraformSuspendConfig record**

Create `TerraformSuspendConfig.java`:

```java
package io.casehub.desiredstate.bridge.terraform;

public record TerraformSuspendConfig(
    String attribute,
    String suspendValue,
    String resumeValue
) {
    public TerraformSuspendConfig {
        if (attribute == null || attribute.isBlank()) {
            throw new IllegalArgumentException("suspend_attribute is required");
        }
        if (suspendValue == null || suspendValue.isBlank()) {
            throw new IllegalArgumentException("suspend_value is required");
        }
        if (resumeValue == null || resumeValue.isBlank()) {
            throw new IllegalArgumentException("resume_value is required");
        }
    }
}
```

- [ ] **Step 5: Create TerraformManifestExtensionParser**

Create `TerraformManifestExtensionParser.java`:

```java
package io.casehub.desiredstate.bridge.terraform;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;

import java.io.IOException;
import java.io.InputStream;
import java.io.UncheckedIOException;
import java.util.LinkedHashMap;
import java.util.Map;

public final class TerraformManifestExtensionParser {

    private static final ObjectMapper YAML = new ObjectMapper(new YAMLFactory())
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

    private TerraformManifestExtensionParser() {}

    public static Map<String, TerraformSuspendConfig> parse(InputStream is) {
        try {
            JsonNode root = YAML.readTree(is);
            JsonNode types = root.path("bridge").path("types");

            Map<String, TerraformSuspendConfig> configs = new LinkedHashMap<>();
            if (types.isArray()) {
                for (JsonNode typeDef : types) {
                    String id = typeDef.path("id").asText();
                    JsonNode terraform = typeDef.path("terraform");
                    if (!terraform.isMissingNode()
                            && !terraform.path("suspend_attribute").isMissingNode()) {
                        configs.put(id, toSuspendConfig(terraform));
                    }
                }
            }
            return configs;
        } catch (IOException e) {
            throw new UncheckedIOException(
                "Failed to parse Terraform manifest extension", e);
        }
    }

    private static TerraformSuspendConfig toSuspendConfig(JsonNode node) {
        return new TerraformSuspendConfig(
            node.path("suspend_attribute").asText(),
            node.path("suspend_value").asText(),
            node.path("resume_value").asText());
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl bridge/terraform -Dtest=TerraformManifestExtensionParserTest`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add bridge/terraform/
git commit -m "feat(#209): TerraformSuspendConfig and manifest extension parser"
```

### Task 4: Implement SuspendCapable on TerraformBridgeProvider

**Files:**
- Modify: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/TerraformBridgeProvider.java`
- Modify: `bridge/terraform/src/test/java/io/casehub/desiredstate/bridge/terraform/TerraformBridgeProviderTest.java`
- Create: `bridge/terraform/src/test/java/io/casehub/desiredstate/bridge/terraform/TerraformSuspendCapableContractTest.java`

**Interfaces:**
- Consumes: `TerraformSuspendConfig(attribute, suspendValue, resumeValue)` from Task 3
- Produces: `TerraformBridgeProvider implements SuspendCapable` with
  `suspend(BridgeRequest)` and `resume(BridgeRequest)`

- [ ] **Step 1: Write the TCK contract test**

Create `TerraformSuspendCapableContractTest.java`:

```java
package io.casehub.desiredstate.bridge.terraform;

import io.casehub.desiredstate.bridge.BridgeRequest;
import io.casehub.desiredstate.bridge.SuspendCapable;
import io.casehub.desiredstate.bridge.testing.SuspendCapableContractTest;

import java.util.List;
import java.util.Map;

class TerraformSuspendCapableContractTest extends SuspendCapableContractTest {

    @Override
    protected SuspendCapable createSuspendCapableProvider() {
        var mockProtocol = new TerraformBridgeProviderTest.MockTerraformProtocol();
        mockProtocol.readResponse = new ReadResponse(
            Map.of("id", "i-123", "instance_state", "running"), new byte[0], List.of());
        mockProtocol.planResponse = new PlanResponse(
            Map.of("id", "i-123", "instance_state", "stopped"), new byte[]{1}, false, List.of());
        mockProtocol.applyResponse = new ApplyResponse(
            Map.of("id", "i-123", "instance_state", "stopped"), new byte[0], List.of());

        var registry = new TerraformBridgeProviderTest.MockTenantProviderRegistry(mockProtocol);
        return new TerraformBridgeProvider(
            registry,
            Map.of("tf:mock/test:mock_resource",
                new TerraformSuspendConfig("instance_state", "stopped", "running")));
    }

    @Override
    protected BridgeRequest sampleSuspendRequest() {
        return new BridgeRequest("tf:mock/test:mock_resource", "web-1",
            Map.of("ami", "ami-123"), "tenant-1",
            Map.of("id", "i-123", "instance_state", "running"));
    }
}
```

- [ ] **Step 2: Run contract test to verify it fails**

Run: `mvn --batch-mode test -pl bridge/terraform -Dtest=TerraformSuspendCapableContractTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — constructor and SuspendCapable interface missing

- [ ] **Step 3: Add suspend config to TerraformBridgeProvider constructor**

Add a new constructor that accepts the suspend config map. Keep the
existing single-arg constructor for backward compatibility:

```java
public class TerraformBridgeProvider implements BridgeProvider, PlanCapable, ImportCapable, SuspendCapable {

    private final TenantProviderRegistry registry;
    private final Map<String, TerraformSuspendConfig> suspendConfigs;
    private Set<String> supportedTypes;

    public TerraformBridgeProvider(TenantProviderRegistry registry) {
        this(registry, Map.of());
    }

    public TerraformBridgeProvider(TenantProviderRegistry registry,
                                    Map<String, TerraformSuspendConfig> suspendConfigs) {
        this.registry = registry;
        this.suspendConfigs = suspendConfigs;
    }
```

Add the import for `SuspendCapable`.

- [ ] **Step 4: Implement suspend() and resume()**

```java
@Override
public BridgeResult suspend(BridgeRequest request) {
    var config = suspendConfigs.get(request.typeId());
    if (config == null) {
        return new BridgeResult.Failed(
            "Suspend not supported for type: " + request.typeId(),
            BridgeErrorCategory.CONFIGURATION);
    }
    return withAuthRetry(request.tenancyId(),
        () -> doSuspendResume(request, config.attribute(), config.suspendValue()));
}

@Override
public BridgeResult resume(BridgeRequest request) {
    var config = suspendConfigs.get(request.typeId());
    if (config == null) {
        return new BridgeResult.Failed(
            "Resume not supported for type: " + request.typeId(),
            BridgeErrorCategory.CONFIGURATION);
    }
    return withAuthRetry(request.tenancyId(),
        () -> doSuspendResume(request, config.attribute(), config.resumeValue()));
}

private BridgeResult doSuspendResume(BridgeRequest request,
                                      String attribute, String targetValue) {
    var protocol = registry.getProtocol(request.tenancyId());
    Map<String, Object> currentState = request.previousState().isEmpty()
        ? protocol.read(stripNamespace(request.typeId()), Map.of()).currentState()
        : request.previousState();

    if (currentState == null || currentState.isEmpty()) {
        return new BridgeResult.Failed(
            "Cannot read current state for " + request.resourceId(),
            BridgeErrorCategory.PERMANENT);
    }

    Map<String, Object> proposedState = new HashMap<>(currentState);
    proposedState.put(attribute, targetValue);

    var planResp = protocol.plan(
        stripNamespace(request.typeId()), currentState, proposedState);
    if (hasErrors(planResp.diagnostics())) {
        return new BridgeResult.Failed(
            formatErrors(planResp.diagnostics()),
            TerraformErrorClassifier.classify(planResp.diagnostics()));
    }

    var applyResp = protocol.apply(
        stripNamespace(request.typeId()), currentState,
        planResp.plannedPrivate());
    if (hasErrors(applyResp.diagnostics())) {
        return new BridgeResult.Failed(
            formatErrors(applyResp.diagnostics()),
            TerraformErrorClassifier.classify(applyResp.diagnostics()));
    }

    return new BridgeResult.Success(applyResp.newState());
}
```

- [ ] **Step 5: Write unit tests for suspend/resume**

Add to `TerraformBridgeProviderTest`:

```java
@Test
void suspendModifiesAttributeAndApplies() {
    var suspendProvider = new TerraformBridgeProvider(
        mockRegistry,
        Map.of("tf:hashicorp/aws:aws_instance",
            new TerraformSuspendConfig("instance_state", "stopped", "running")));

    mockProtocol.readResponse = new ReadResponse(
        Map.of("id", "i-123", "instance_state", "running"), new byte[0], List.of());
    mockProtocol.planResponse = new PlanResponse(
        Map.of("id", "i-123", "instance_state", "stopped"),
        new byte[]{1}, false, List.of());
    mockProtocol.applyResponse = new ApplyResponse(
        Map.of("id", "i-123", "instance_state", "stopped"), new byte[0], List.of());

    var request = new BridgeRequest("tf:hashicorp/aws:aws_instance", "web-1",
        Map.of("ami", "ami-123"), "tenant-1",
        Map.of("id", "i-123", "instance_state", "running"));
    var result = suspendProvider.suspend(request);

    assertThat(result).isInstanceOf(BridgeResult.Success.class);
    assertThat(mockProtocol.lastPlanProposedState)
        .containsEntry("instance_state", "stopped");
}

@Test
void resumeModifiesAttributeAndApplies() {
    var suspendProvider = new TerraformBridgeProvider(
        mockRegistry,
        Map.of("tf:hashicorp/aws:aws_instance",
            new TerraformSuspendConfig("instance_state", "stopped", "running")));

    mockProtocol.planResponse = new PlanResponse(
        Map.of("id", "i-123", "instance_state", "running"),
        new byte[]{1}, false, List.of());
    mockProtocol.applyResponse = new ApplyResponse(
        Map.of("id", "i-123", "instance_state", "running"), new byte[0], List.of());

    var request = new BridgeRequest("tf:hashicorp/aws:aws_instance", "web-1",
        Map.of("ami", "ami-123"), "tenant-1",
        Map.of("id", "i-123", "instance_state", "stopped"));
    var result = suspendProvider.resume(request);

    assertThat(result).isInstanceOf(BridgeResult.Success.class);
    assertThat(mockProtocol.lastPlanProposedState)
        .containsEntry("instance_state", "running");
}

@Test
void suspendReturnsFailedForUnsupportedType() {
    var result = provider.suspend(
        new BridgeRequest("tf:hashicorp/aws:aws_instance", "web-1",
            Map.of(), "tenant-1"));

    assertThat(result).isInstanceOf(BridgeResult.Failed.class);
    assertThat(((BridgeResult.Failed) result).reason())
        .contains("Suspend not supported");
    assertThat(((BridgeResult.Failed) result).category())
        .isEqualTo(BridgeErrorCategory.CONFIGURATION);
}

@Test
void resumeReturnsFailedForUnsupportedType() {
    var result = provider.resume(
        new BridgeRequest("tf:hashicorp/aws:aws_instance", "web-1",
            Map.of(), "tenant-1"));

    assertThat(result).isInstanceOf(BridgeResult.Failed.class);
    assertThat(((BridgeResult.Failed) result).reason())
        .contains("Resume not supported");
}
```

- [ ] **Step 6: Run all Terraform bridge tests**

Run: `mvn --batch-mode test -pl bridge/terraform`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add bridge/terraform/
git commit -m "feat(#209): implement SuspendCapable on TerraformBridgeProvider"
```

- [ ] **Step 8: Run full build to verify nothing is broken**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 9: Final commit if any compilation fixes were needed**

```bash
git add -u
git commit -m "fix(#209): compilation fixes from full build"
```

## References

- [2026-11-03-suspendcapable-bridge-adapters-design.md] — design spec this plan implements
- bridge/api/SuspendCapable.java — mixin interface (2 methods)
- bridge/runtime/BridgeProvisioner.java:103-135 — existing suspend/resume routing
- bridge/testing/SuspendCapableContractTest.java — TCK base (2 abstract methods)
- bridge/testing/MockSuspendCapableBridgeProvider.java — mock pattern reference
- bridge/ansible/AnsibleBridgeProvider.java — current implements clause
- bridge/ansible/AnsibleModuleConfig.java — record to extend
- bridge/ansible/AnsibleManifestExtensionParser.java — parser to extend
- bridge/ansible/PlaybookGenerator.java — playbook generation
- bridge/terraform/TerraformBridgeProvider.java — current implements clause
- bridge/terraform/TerraformProtocol.java — plan/apply/read signatures
- bridge/terraform/TerraformBridgeProviderTest.java — existing test + mock classes
- GitHub #209 — focal issue
