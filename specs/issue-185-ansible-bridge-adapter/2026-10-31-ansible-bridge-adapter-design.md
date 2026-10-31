# Ansible Collection Bridge Adapter — Design Spec

**Issue:** casehubio/casehub-desiredstate#185
**Parent:** #182 (bridge adapter architecture)
**Depends on:** #183 (bridge SPI — closed), #184 (Terraform bridge — closed), #186 (testing framework — closed)

## Overview

A bridge adapter that invokes Ansible collection modules via the `ansible-playbook` CLI to provision
and deprovision resources. The adapter implements the `BridgeProvider` and `ImportCapable` SPIs,
mapping Ansible module execution onto the bridge contract. Types are declared in bridge manifest YAML;
actual state is read via info/facts companion modules.

## Architecture

### Module: `bridge/ansible/`

Artifact: `casehub-desiredstate-bridge-ansible`
Package: `io.casehub.desiredstate.bridge.ansible`

Dependencies:
- `casehub-desiredstate-bridge-api` — BridgeProvider, ImportCapable, BridgeRequest/Result types
- `casehub-desiredstate-bridge` (runtime) — BridgeDiscovery, BridgeManifestParser
- `com.fasterxml.jackson.core:jackson-databind` — JSON parsing of Ansible output
- JUnit 5, AssertJ, bridge-testing (test scope)

No gRPC, no protobuf, no Ansible Python dependencies. The adapter shells out to the `ansible-playbook`
binary which must be on PATH (or configured in the manifest's `lifecycle.binary`).

### Class Inventory

| Class | Responsibility |
|-------|---------------|
| `AnsibleBridgeProvider` | Implements `BridgeProvider` + `ImportCapable`. Orchestrates playbook generation → execution → output parsing for each SPI method. |
| `PlaybookGenerator` | Generates minimal YAML playbooks for provision, deprovision, readState, and import operations. Each playbook targets a single host with a single task invoking the declared module. |
| `AnsibleOutputParser` | Parses JSON callback output from `ansible-playbook --callback json`. Extracts task results, classifies errors by status (unreachable/failed/changed). |
| `InventoryGenerator` | Generates single-host dynamic inventory JSON from reserved spec keys (`_host`, `_connection`, `_become`). |
| `AnsibleProcess` | Wraps `ProcessBuilder` execution — sets up environment, captures stdout/stderr, enforces timeout. |
| `AnsibleErrorClassifier` | Maps Ansible task status to `BridgeErrorCategory`: unreachable → TRANSIENT, failed+auth → AUTHENTICATION, failed → PERMANENT. |
| `AnsibleModuleConfig` | Record holding per-type module configuration from the manifest extension: provision module, deprovision module (or `state: absent`), info module, module parameters mapping. |

### Manifest Extension

The standard bridge manifest YAML is extended with Ansible-specific fields per type:

```yaml
bridge:
  protocol: ansible
  version: "1.0"

  lifecycle:
    mode: managed-process
    binary: ansible-playbook          # or full path
    healthCheck:
      type: process                   # verify binary exists
      interval: 60s
      timeout: 5s

  types:
    - id: ansible:amazon.aws.ec2_instance
      ansible:
        collection: amazon.aws
        provision_module: ec2_instance    # module for apply()
        deprovision_strategy: state_absent  # re-invoke with state=absent
        info_module: ec2_instance_info    # module for readState()
        info_filter_key: instance_ids     # key to pass importId for filtering
        result_key: instances             # key in module return to extract state
      stateMapping:
        _default: PRESENT
        _notFound: ABSENT
      errorMapping:
        _default: { category: PERMANENT }
        Throttled: { category: TRANSIENT }

    - id: ansible:amazon.aws.s3_bucket
      ansible:
        collection: amazon.aws
        provision_module: s3_bucket
        deprovision_strategy: state_absent
        info_module: s3_bucket_info
        info_filter_key: name
        result_key: buckets
      stateMapping:
        _default: PRESENT
        _notFound: ABSENT
      errorMapping:
        _default: { category: PERMANENT }
```

The `ansible:` section is an extension block parsed by `AnsibleBridgeProvider` from the type
descriptor's raw configuration (not by `BridgeManifestParser` which handles the standard fields).

### Deprovision Strategies

Two deprovision approaches for Ansible modules:

1. **`state_absent`** (default) — re-invoke the provision module with `state: absent` appended to
   parameters. Most Ansible modules support this convention.
2. **`deprovision_module`** — invoke a separate module for deprovision (rare, for modules that don't
   follow the state convention). Declared as `deprovision_module: <module_name>` in the manifest.

### Spec-to-Module Parameter Mapping

`BridgeRequest.spec()` is a `Map<String, Object>`. Reserved keys (prefixed with `_`) are consumed by
the bridge infrastructure:

| Key | Default | Purpose |
|-----|---------|---------|
| `_host` | `localhost` | Target host for inventory |
| `_connection` | `local` | Ansible connection type |
| `_become` | `false` | Privilege escalation |
| `_become_method` | `sudo` | Privilege escalation method |
| `_extra_vars` | `{}` | Additional variables passed via `--extra-vars` |
| `_environment` | `{}` | Environment variables for the playbook |
| `_timeout` | `300` | Execution timeout in seconds |

All non-reserved keys are passed directly as module parameters. For example, a spec of
`{ "_host": "localhost", "instance_type": "t3.micro", "image_id": "ami-123" }` generates a playbook
task with `instance_type: t3.micro` and `image_id: ami-123`.

### Playbook Generation

`PlaybookGenerator` creates minimal YAML playbooks. Example for provision:

```yaml
---
- hosts: target
  gather_facts: false
  tasks:
    - name: provision
      amazon.aws.ec2_instance:
        instance_type: t3.micro
        image_id: ami-123
      register: result
```

For deprovision (state_absent strategy):

```yaml
---
- hosts: target
  gather_facts: false
  tasks:
    - name: deprovision
      amazon.aws.ec2_instance:
        instance_type: t3.micro
        image_id: ami-123
        state: absent
      register: result
```

For readState (info module):

```yaml
---
- hosts: target
  gather_facts: false
  tasks:
    - name: read_state
      amazon.aws.ec2_instance_info:
        instance_ids:
          - i-1234567890
      register: result
```

Playbooks are written to a temporary directory (per-invocation), executed, and cleaned up.

### Inventory Generation

`InventoryGenerator` produces Ansible dynamic inventory JSON:

```json
{
  "all": {
    "hosts": {
      "target": {
        "ansible_host": "localhost",
        "ansible_connection": "local",
        "ansible_become": false
      }
    }
  }
}
```

Written to a temp file alongside the playbook.

### Execution Flow

**apply(BridgeRequest):**
1. Extract reserved keys from spec, remaining keys become module parameters
2. Generate provision playbook via `PlaybookGenerator`
3. Generate inventory via `InventoryGenerator`
4. Execute: `ansible-playbook -i inventory.json playbook.yml --callback json [--extra-vars ...]`
5. Parse JSON output via `AnsibleOutputParser`
6. If task changed/ok → `BridgeResult.Success(outputState)` (output state from registered result)
7. If task failed → `BridgeResult.Failed(reason, AnsibleErrorClassifier.classify(...))`
8. Clean up temp files

**destroy(BridgeRequest):**
Same as apply but with deprovision playbook (state_absent or deprovision_module).

**readState(BridgeReadRequest):**
1. Generate info-module playbook
2. Execute with JSON callback
3. If info module returns results → extract via `result_key`, build state map → `BridgeStateResult.Found(state)`
4. If empty results → `BridgeStateResult.NotFound()`
5. If execution error → `BridgeStateResult.Error(reason)`

**importResource(typeId, importId, tenancyId):**
1. Generate info-module playbook with `info_filter_key: [importId]`
2. Execute, parse output
3. If found → `BridgeImportResult.Success(state)`
4. If not found → `BridgeImportResult.Failed("Resource not found: " + importId)`

### Error Classification

`AnsibleErrorClassifier` maps Ansible task statuses to `BridgeErrorCategory`:

| Ansible Status | BridgeErrorCategory | Rationale |
|---------------|--------------------:|-----------|
| `unreachable` | TRANSIENT | Network/connectivity — retry may succeed |
| `failed` + auth keywords in msg | AUTHENTICATION | Credential issue |
| `failed` (other) | PERMANENT | Module-level failure |
| Timeout | TRANSIENT | Execution timeout — retry may succeed |

Auth keyword detection: case-insensitive scan of task `msg` for `auth`, `credential`, `unauthorized`,
`accessdenied`, `forbidden`, `permission denied`. Mirrors the Terraform bridge's `isAuthError` pattern.

### Testing Strategy

1. **Unit tests** — `PlaybookGeneratorTest`, `AnsibleOutputParserTest`, `InventoryGeneratorTest`,
   `AnsibleErrorClassifierTest`. Pure logic, no subprocess.

2. **Contract tests** — `AnsibleBridgeProviderContractTest extends BridgeProviderContractTest`,
   `AnsibleImportCapableContractTest extends ImportCapableContractTest`. Use a mock
   `AnsibleProcess` that returns canned JSON responses. Validates SPI contract compliance.

3. **Integration tests** (tagged `integration`) — require Ansible binary on PATH. Invoke real
   `ansible-playbook` against `ansible.builtin` modules (e.g., `file`, `copy`) on localhost.
   Validates end-to-end flow without cloud credentials.

4. **Cloud integration tests** (tagged `cloud-integration`) — future scope, similar to Terraform's
   AWS/GCP integration tests. Require cloud credentials and real Ansible collections.

### Manifest YAML Discovery

The bridge manifest lives at `META-INF/desiredstate/bridges/ansible.yaml` on the classpath.
Discovered by the existing `bridge/deployment/` Quarkus build extension and `bridge/spring/`
auto-configuration — no new discovery code needed.

The `ansible:` extension block is not part of the standard `BridgeManifest` schema. The manifest
parser's `FAIL_ON_UNKNOWN_PROPERTIES = false` setting means the standard fields parse normally.
`AnsibleBridgeProvider` reads the extension block by re-parsing the manifest or receiving the raw
config as part of construction. Design choice: `AnsibleModuleConfig` is parsed at provider
construction time from a supplementary YAML read, keyed by type ID.

### Ansible-Specific Considerations

- **Ansible binary availability** — the lifecycle healthCheck (`type: process`) verifies the binary
  exists at startup. If missing, the bridge fails fast with a clear error message.
- **Collection installation** — out of scope. Collections must be pre-installed via
  `ansible-galaxy collection install`. The bridge does not manage collection installation.
- **Ansible configuration** — respects `ANSIBLE_CONFIG` environment variable and standard
  `ansible.cfg` lookup order. No CaseHub-specific config injection.
- **Concurrency** — Ansible playbook execution is process-based, naturally isolated. No shared state
  between concurrent invocations. Each call gets its own temp directory.
- **Output handling** — the `json` callback plugin is used via `ANSIBLE_STDOUT_CALLBACK=json`
  environment variable, ensuring structured output without requiring global Ansible configuration.

## References

- bridge/api/BridgeProvider.java — SPI contract (5 methods)
- bridge/api/ImportCapable.java — import mixin
- bridge/api/BridgeRequest.java, BridgeResult.java, BridgeStateResult.java — request/response types
- bridge/api/BridgeDescriptor.java, BridgeTypeDescriptor.java — descriptor records
- bridge/api/BridgeLifecycleConfig.java — lifecycle configuration
- bridge/runtime/BridgeDiscovery.java — auto-wiring into NodeProvisioner + ActualStateAdapter
- bridge/runtime/BridgeManifestParser.java — YAML manifest parsing
- bridge/runtime/src/test/resources/META-INF/desiredstate/bridges/test-bridge.yaml — manifest format
- bridge/terraform/TerraformBridgeProvider.java — reference implementation pattern
- bridge/testing/BridgeProviderContractTest.java — TCK contract test
- bridge/testing/ImportCapableContractTest.java — import TCK
- casehubio/casehub-desiredstate#183 — bridge SPI design (closed)
- casehubio/casehub-desiredstate#184 — Terraform bridge (closed)
- casehubio/casehub-desiredstate#186 — bridge testing framework (closed)
