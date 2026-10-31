# Ansible Collection Bridge Adapter — Design Spec

**Issue:** casehubio/casehub-desiredstate#185
**Parent:** #182 (bridge adapter architecture)
**Depends on:** #183 (bridge SPI — closed), #184 (Terraform bridge — closed), #186 (testing framework — closed)

## Overview

A bridge adapter that invokes Ansible collection modules via the `ansible-playbook` CLI to provision
and deprovision resources. The adapter implements the `BridgeProvider`, `ImportCapable`, and
`PlanCapable` SPIs, mapping Ansible module execution onto the bridge contract. Types are declared in
bridge manifest YAML; actual state is read via info/facts companion modules; drift is detected by
comparing actual attributes against desired spec.

**Minimum Ansible version:** ansible-core >= 2.14 (JSON callback stable, collection support mature).

## Architecture

### Module: `bridge/ansible/`

Artifact: `casehub-desiredstate-bridge-ansible`
Package: `io.casehub.desiredstate.bridge.ansible`

Dependencies:
- `casehub-desiredstate-bridge-api` — BridgeProvider, ImportCapable, PlanCapable, BridgeRequest/Result types
- `casehub-desiredstate-bridge` (runtime) — BridgeDiscovery, BridgeManifestParser
- `com.fasterxml.jackson.core:jackson-databind` — JSON parsing of Ansible output
- `com.fasterxml.jackson.dataformat:jackson-dataformat-yaml` — YAML parsing for extension config
- JUnit 5, AssertJ, bridge-testing (test scope)

No gRPC, no protobuf, no Ansible Python dependencies. The adapter shells out to the `ansible-playbook`
binary which must be on PATH (or configured in the manifest's `lifecycle.binary`).

### Class Inventory

| Class | Responsibility |
|-------|---------------|
| `AnsibleBridgeProvider` | Implements `BridgeProvider` + `ImportCapable` + `PlanCapable`. Orchestrates playbook generation → execution → output parsing for each SPI method. `protocolId()` returns `"ansible"`. `supportedTypes()` returns the key set from `moduleConfigs`. |
| `PlaybookGenerator` | Generates minimal YAML playbooks for provision, deprovision, readState (info module), check (--check mode), and import operations. Each playbook targets a single host with a single task invoking the declared module. |
| `AnsibleOutputParser` | Parses JSON callback output from `ansible-playbook`. Extracts task results, normalises output state via `output_keys` config, classifies errors by status (unreachable/failed/changed). |
| `InventoryGenerator` | Generates single-host dynamic inventory JSON from reserved spec keys (`_host`, `_connection`, `_become`). |
| `AnsibleProcess` | Wraps `ProcessBuilder` execution — sets up environment (including `ANSIBLE_STDOUT_CALLBACK=json`), captures stdout/stderr, enforces timeout. Creates temp dir with restrictive permissions (0700), cleans up in finally block. |
| `AnsibleErrorClassifier` | Maps Ansible task status to `BridgeErrorCategory`. Default: TRANSIENT for unrecognised errors (aligned with Terraform bridge philosophy). |
| `AnsibleModuleConfig` | Record: `collection`, `provisionModule`, `deprovisionStrategy`, `deprovisionModule`, `infoModule`, `infoFilterKey`, `resultKey`, `outputKeys`, `driftKeys`. |
| `AnsibleManifestExtensionParser` | Parses `ansible:` extension blocks from manifest YAML. Produces `Map<String, AnsibleModuleConfig>` keyed by type ID. |

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
      type: tcp                       # existing HealthCheckType — runtime interprets as
                                      # binary-exists check for managed-process mode
      interval: 60s
      timeout: 5s

  types:
    - id: ansible:amazon.aws.ec2_instance
      ansible:
        collection: amazon.aws
        provision_module: ec2_instance
        deprovision_strategy: state_absent
        info_module: ec2_instance_info
        info_filter_key: instance_ids     # key to pass resource ID for filtering
        result_key: instances             # key in module return to extract state
        output_keys:                      # keys to extract from provision result → outputState
          - instance_id
          - public_ip_address
          - private_ip_address
          - instance_type
        drift_keys:                       # keys to compare for drift detection
          - instance_type
          - security_groups
          - tags
      stateMapping:
        _default: PRESENT
        _notFound: ABSENT
      errorMapping:
        _default: { category: TRANSIENT }
        Throttled: { category: TRANSIENT }

    - id: ansible:amazon.aws.s3_bucket
      ansible:
        collection: amazon.aws
        provision_module: s3_bucket
        deprovision_strategy: state_absent
        info_module: s3_bucket_info
        info_filter_key: name
        result_key: buckets
        output_keys:
          - name
          - creation_date
        drift_keys:
          - versioning
          - tags
      stateMapping:
        _default: PRESENT
        _notFound: ABSENT
      errorMapping:
        _default: { category: TRANSIENT }
```

The `ansible:` section is an extension block parsed by `AnsibleManifestExtensionParser` — not by
`BridgeManifestParser` which handles the standard fields.

**HealthCheckType note:** The `HealthCheckType` enum has `GRPC`, `HTTP`, `TCP` — no `PROCESS`
variant. The manifest uses `type: tcp` as the existing closest match. The Ansible module's health
check implementation interprets this for `managed-process` mode by verifying the binary exists and
is executable (`ansible-playbook --version`), rather than attempting a TCP connection. This is a
pragmatic reuse of the existing enum. If more CLI bridges are added, a `PROCESS` variant should be
introduced to the SPI.

### Extension YAML Parsing

`AnsibleManifestExtensionParser` handles the dual-parse problem:

1. At construction, scans `META-INF/desiredstate/bridges/` on the classpath
2. For each YAML file, reads only `bridge.protocol` — filters to `protocol: ansible`
3. Parses the `ansible:` extension block from each `types[]` entry into `AnsibleModuleConfig` records
4. Returns `Map<String, AnsibleModuleConfig>` keyed by type ID

**CDI wiring:** `AnsibleRuntimeBeans` (Quarkus `@Produces`) calls `AnsibleManifestExtensionParser`,
constructs `AnsibleBridgeProvider` with the config map. `@DefaultBean` on the producer — no new
deployment processor needed.

**Spring wiring:** `AnsibleBridgeAutoConfiguration` (`@AutoConfiguration`) does the same via
`@Bean @ConditionalOnMissingBean`. Scans classpath resources, constructs the provider.

Both configurations reuse the standard `BridgeDiscovery.discoverAll()` path — the
`AnsibleBridgeProvider` CDI/Spring bean matches the `BridgeDescriptor.protocolId()` during discovery.

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
| `_extra_vars` | `{}` | Additional variables (written to file, passed via `--extra-vars @file`) |
| `_environment` | `{}` | Environment variables for the playbook (non-sensitive only) |
| `_timeout` | `300` | Execution timeout in seconds |

All non-reserved keys are passed directly as module parameters. For example, a spec of
`{ "_host": "localhost", "instance_type": "t3.micro", "image_id": "ami-123" }` generates a playbook
task with `instance_type: t3.micro` and `image_id: ami-123`.

**Sensitive data:** `_extra_vars` are written to a temp file and passed via `--extra-vars @file.json`,
not on the command line. This avoids exposing values in `ps` output.

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

For check mode (PlanCapable):

```yaml
---
- hosts: target
  gather_facts: false
  tasks:
    - name: check
      amazon.aws.ec2_instance:
        instance_type: t3.micro
        image_id: ami-123
      register: result
      check_mode: true
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

### Output State Contract

When `apply()` succeeds, the Ansible module returns a task result (the `register: result` variable).
`AnsibleOutputParser` normalises this into a deterministic `outputState` map:

1. Extract the raw module return value from the JSON callback output
2. If `result_key` is configured, navigate into that key (e.g., `instances[0]` for ec2_instance)
3. Filter to only `output_keys` — the manifest declares which keys to retain in output state
4. The resulting map is returned as `BridgeResult.Success(outputState)`

**Example:** For `ec2_instance` with `output_keys: [instance_id, public_ip_address, instance_type]`,
a successful provision produces:
```json
{
  "instance_id": "i-1234567890",
  "public_ip_address": "54.123.45.67",
  "instance_type": "t3.micro"
}
```

This output state is stored by `BridgeProvisioner` in `BridgeOutputStateStore` and becomes
`previousState` in subsequent `BridgeRequest`/`BridgeReadRequest` calls.

**Resource identification chain:**
1. `apply()` → Ansible returns `instance_id` → included in `outputState` via `output_keys`
2. `BridgeProvisioner` stores `outputState` → `BridgeOutputStateStore`
3. Next reconciliation → `BridgeActualStateAdapter` loads `previousState` from store
4. `readState()` → extracts resource ID from `previousState` using `info_filter_key` mapping:
   `previousState.get("instance_id")` → passed as `instance_ids: [i-1234567890]` to info module
5. Info module returns current state → compared against desired spec for drift

The `info_filter_key` in the manifest must correspond to a key present in `output_keys`. Validation
at parse time ensures this.

### Drift Detection

`readState()` includes drift detection by comparing actual attributes against desired spec:

1. Invoke info module, extract current state via `result_key`
2. For each key in `drift_keys` (manifest-declared), compare `actual[key]` vs `desiredSpec[key]`
3. If any drift key differs → set `status = "drifted"` in the returned state map
4. If all drift keys match → set `status = "present"`

This mirrors the Terraform bridge's approach: `BridgeActualStateAdapter.mapStateResult()` reads
`state.get("status")` to map to `NodeStatus`. Without explicit drift detection, all resources
would report `PRESENT` regardless of actual configuration.

`drift_keys` is optional — if omitted, readState returns `status = "present"` (no drift detection
for that type). This allows types where drift detection is not meaningful (e.g., one-shot actions).

### PlanCapable — Check Mode

The adapter implements `PlanCapable` using Ansible's `--check` mode:

**`plan(BridgeRequest)` →** Generates a check-mode playbook (provision module with `check_mode: true`),
executes it, and returns a `BridgePlan` with:
- `planId` = resourceId
- `description` = summary of what would change (from check mode output)
- `changes` = single `ResourceChange(typeId, resourceId, CREATE or UPDATE)`
- `rawPlan` = serialised check-mode output (for `applyPlan` to reference)

**`applyPlan(BridgePlan)` →** Executes the real provision playbook (without check mode). The plan
output is informational — Ansible doesn't have a plan-then-apply two-step like Terraform, so
`applyPlan` is functionally identical to `apply`. The value is in the approval gate: when
`PlanApprovalPolicy` requires approval, the check-mode output gives operators visibility into what
will change before execution.

**Limitation:** Not all Ansible modules support `--check` mode. Modules that don't will skip the
task in check mode and report no changes. The plan output will show "no changes" even when real
execution would make changes. This is a known Ansible limitation, not a bridge design flaw.

### Execution Flow

**apply(BridgeRequest):**
1. Extract reserved keys from spec, remaining keys become module parameters
2. Create temp directory with 0700 permissions
3. Generate provision playbook via `PlaybookGenerator`
4. Generate inventory via `InventoryGenerator`
5. Write `_extra_vars` to temp file (not command line)
6. Execute: `ansible-playbook -i inventory.json playbook.yml --extra-vars @extra_vars.json`
   with env `ANSIBLE_STDOUT_CALLBACK=json`
7. Parse JSON output via `AnsibleOutputParser`
8. If task changed/ok → extract output state via `output_keys` → `BridgeResult.Success(outputState)`
9. If task failed → `BridgeResult.Failed(reason, AnsibleErrorClassifier.classify(...))`
10. Clean up temp directory in `finally` block

**destroy(BridgeRequest):**
Same as apply but with deprovision playbook (state_absent or deprovision_module).

**readState(BridgeReadRequest):**
1. Extract resource ID from `previousState` using `info_filter_key` mapping
2. Generate info-module playbook with the resource ID as filter
3. Execute with JSON callback
4. If info module returns results → extract via `result_key`, compare `drift_keys` against
   `desiredSpec` → set `status` → `BridgeStateResult.Found(state)`
5. If empty results → `BridgeStateResult.NotFound()`
6. If execution error → `BridgeStateResult.Error(reason)`

**plan(BridgeRequest):**
1. Generate check-mode playbook
2. Execute with `--check` flag
3. Parse output → build `BridgePlan` with change summary

**applyPlan(BridgePlan):**
1. Generate provision playbook (no check mode)
2. Execute normally → return `BridgeResult`

**importResource(typeId, importId, tenancyId):**
1. Generate info-module playbook with `info_filter_key: [importId]`
2. Execute, parse output via `output_keys`
3. If found → `BridgeImportResult.Success(state)`
4. If not found → `BridgeImportResult.Failed("Resource not found: " + importId)`

### Error Classification

`AnsibleErrorClassifier` maps Ansible task statuses to `BridgeErrorCategory`:

| Ansible Status | BridgeErrorCategory | Rationale |
|---------------|--------------------:|-----------|
| `unreachable` | TRANSIENT | Network/connectivity — retry may succeed |
| `failed` + auth keywords in msg | AUTHENTICATION | Credential issue |
| `failed` + config keywords in msg | CONFIGURATION | Invalid parameters, missing module args, collection not installed |
| `failed` + quota keywords in msg | QUOTA | Cloud resource limits |
| `failed` (unrecognised) | TRANSIENT | Default: retry with budget, escalate on exhaustion — aligned with Terraform bridge |
| Timeout | TRANSIENT | Execution timeout — retry may succeed |

**Keyword patterns:**
- **Auth:** `auth`, `credential`, `unauthorized`, `accessdenied`, `forbidden`, `permission denied`
- **Config:** `required`, `missing parameter`, `unknown module`, `not found in collection`, `invalid`
- **Quota:** `limit`, `quota`, `exceeded`, `capacity`, `too many`

### Temp File Security

`AnsibleProcess` manages temp directory lifecycle:

1. **Creation:** `Files.createTempDirectory("ansible-bridge-", PosixFilePermissions 0700)`
2. **Contents:** playbook YAML, inventory JSON, extra-vars JSON — all potentially sensitive
3. **Cleanup:** `finally` block deletes the entire temp directory recursively after every execution,
   regardless of success or failure
4. **JVM crash:** Register a shutdown hook that cleans up any surviving temp directories from the
   current process (tracked via a static `Set<Path>`)
5. **Extra vars:** Always file-based (`--extra-vars @file.json`), never on the command line

### Credential Handling

**Deferred to Phase 2.** The initial implementation passes credentials via `_environment` in the spec
map (e.g., `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` as environment variables for the subprocess).

This is acceptable for the initial release because:
1. Environment variables are per-process (not visible in `ps`, not logged by Ansible)
2. The `_environment` map is part of `BridgeRequest.spec()` which is already handled as potentially
   sensitive by the runtime — `BridgeOutputStateStore` stores only `outputState`, not the input spec
3. The Terraform bridge's `CredentialResolver` integration is specific to its `TenantProviderRegistry`
   architecture — a generic credential mechanism for CLI bridges would be a cross-cutting concern
   better designed when a second CLI bridge exists

**Phase 2 scope:** Integration with `CredentialResolver` SPI via `credentialRef` in the manifest.
The bridge would resolve credentials at execution time and inject them as environment variables,
removing credentials from the spec map entirely.

### SuspendCapable

**Not implemented.** Some Ansible modules support start/stop semantics (e.g., `ec2_instance` with
`state: stopped/running`), but suspend/resume is not a universal Ansible module pattern. If needed,
it can be added per-type in a future iteration without breaking changes — `BridgeProvisioner` already
handles the `instanceof SuspendCapable` check gracefully (falls through to the default path).

### Testing Strategy

1. **Unit tests** — `PlaybookGeneratorTest`, `AnsibleOutputParserTest`, `InventoryGeneratorTest`,
   `AnsibleErrorClassifierTest`, `AnsibleManifestExtensionParserTest`. Pure logic, no subprocess.

2. **Contract tests** — `AnsibleBridgeProviderContractTest extends BridgeProviderContractTest`,
   `AnsibleImportCapableContractTest extends ImportCapableContractTest`,
   `AnsiblePlanCapableContractTest extends PlanCapableContractTest`. Use a mock `AnsibleProcess`
   that returns canned JSON responses. Validates SPI contract compliance.

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
`AnsibleManifestExtensionParser` performs a separate parse of the same YAML file to extract the
extension blocks. This dual-parse approach keeps `BridgeManifestParser` unchanged.

### Ansible-Specific Considerations

- **Ansible binary availability** — verified at startup by running `ansible-playbook --version`.
  If missing or below minimum version, the bridge fails fast with a clear error message.
- **Collection installation** — out of scope. Collections must be pre-installed via
  `ansible-galaxy collection install`. The bridge does not manage collection installation.
- **Ansible configuration** — respects `ANSIBLE_CONFIG` environment variable and standard
  `ansible.cfg` lookup order. No CaseHub-specific config injection.
- **Concurrency** — Ansible playbook execution is process-based, naturally isolated. No shared state
  between concurrent invocations. Each call gets its own temp directory.
- **Output handling** — the `json` callback plugin is used via `ANSIBLE_STDOUT_CALLBACK=json`
  environment variable, ensuring structured output without requiring global Ansible configuration.
- **Version compatibility** — ansible-core >= 2.14 required. JSON callback format is stable since
  2.4. Collection support is mature since 2.10. The bridge does not support Ansible 2.x classic.

## References

- bridge/api/BridgeProvider.java — SPI contract (5 methods)
- bridge/api/ImportCapable.java — import mixin
- bridge/api/PlanCapable.java — plan mixin
- bridge/api/BridgeRequest.java, BridgeResult.java, BridgeStateResult.java — request/response types
- bridge/api/BridgeDescriptor.java, BridgeTypeDescriptor.java — descriptor records
- bridge/api/BridgeLifecycleConfig.java — lifecycle configuration
- bridge/api/BridgeErrorCategory.java — TRANSIENT, PERMANENT, CONFIGURATION, AUTHENTICATION, QUOTA
- bridge/api/HealthCheckType.java — GRPC, HTTP, TCP (no PROCESS variant)
- bridge/runtime/BridgeDiscovery.java — auto-wiring into NodeProvisioner + ActualStateAdapter
- bridge/runtime/BridgeProvisioner.java — PlanCapable/ImportCapable dispatch, BridgeOutputStateStore
- bridge/runtime/BridgeActualStateAdapter.java — readState → NodeStatus via state.get("status")
- bridge/runtime/BridgeManifestParser.java — YAML manifest parsing (FAIL_ON_UNKNOWN_PROPERTIES=false)
- bridge/runtime/src/test/resources/META-INF/desiredstate/bridges/test-bridge.yaml — manifest format
- bridge/terraform/TerraformBridgeProvider.java — reference implementation pattern
- bridge/terraform/TerraformErrorClassifier.java — TRANSIENT default philosophy
- bridge/testing/BridgeProviderContractTest.java — TCK contract test
- bridge/testing/ImportCapableContractTest.java — import TCK
- bridge/testing/PlanCapableContractTest.java — plan TCK
- casehubio/casehub-desiredstate#183 — bridge SPI design (closed)
- casehubio/casehub-desiredstate#184 — Terraform bridge (closed)
- casehubio/casehub-desiredstate#186 — bridge testing framework (closed)
