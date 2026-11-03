# SuspendCapable Support for Terraform and Ansible Bridge Adapters

**Issue:** casehubio/casehub-desiredstate#209
**Scale:** S | **Complexity:** Med
**Date:** 2026-11-03

## Context

The `SuspendCapable` mixin (bridge/api) and its runtime routing in `BridgeProvisioner`
are fully implemented. The mock provider and TCK contract test exist in bridge/testing.
Neither `TerraformBridgeProvider` nor `AnsibleBridgeProvider` implements `SuspendCapable`.

Both original adapter specs (#184, #185) explicitly deferred suspend/resume as future work.

## Design

Both adapters add `SuspendCapable` to their `implements` clause. Each receives per-type
suspend configuration via its existing configuration mechanism. Types without suspend
config return `BridgeResult.Failed`. No changes needed in `BridgeProvisioner` — its
`instanceof SuspendCapable` check activates automatically.

### Terraform

**Config model:** `TerraformSuspendConfig(String attribute, String suspendValue, String resumeValue)`

Injected as `Map<String, TerraformSuspendConfig>` in the `TerraformBridgeProvider`
constructor (keyed by type ID). The config specifies which schema attribute controls the
resource lifecycle state and the values for suspended/running states.

**suspend(BridgeRequest):**
1. Look up `TerraformSuspendConfig` for `request.typeId()`
2. If absent → `BridgeResult.Failed("Suspend not supported for type: " + typeId, CONFIGURATION)`
3. Read current state via `protocol.read()`
4. Copy state, set `config.attribute()` to `config.suspendValue()`
5. Plan + apply via the standard `plan()` → `apply()` path (reusing `doApply` pattern
   with the modified spec)
6. Return `BridgeResult.Success` or `BridgeResult.Failed` based on apply result

**resume(BridgeRequest):**
Same flow with `config.resumeValue()` instead of `config.suspendValue()`.

**Manifest YAML extension:**
```yaml
bridge:
  protocol: terraform
  types:
    - id: tf:hashicorp/aws:aws_instance
      terraform:
        suspend_attribute: instance_state
        suspend_value: stopped
        resume_value: running
```

New `TerraformManifestExtensionParser` extracts suspend config per type from the
`terraform:` block in the bridge manifest, following the same pattern as
`AnsibleManifestExtensionParser`. Consumes only the suspend fields — future Terraform
manifest extensions can be added to the same parser.

### Ansible

**Config model:** Extend `AnsibleModuleConfig` with four new fields:
- `suspendModule` — module to call for suspend (nullable)
- `suspendStateParam` — parameter name for the suspend state (nullable)
- `suspendStateValue` — value to set (nullable)
- `resumeStateParam` — parameter name for the resume state (nullable, defaults to
  `suspendStateParam` if absent)
- `resumeStateValue` — value to set (nullable)

When `suspendModule` is null, the type does not support suspend. When present, the resume
module defaults to the same module (most common pattern: same module, different state value).

**suspend(BridgeRequest):**
1. Look up `AnsibleModuleConfig` for `request.typeId()`
2. If `config.suspendModule()` is null →
   `BridgeResult.Failed("Suspend not supported for type: " + typeId, CONFIGURATION)`
3. Generate playbook via `PlaybookGenerator.suspend()` — same structure as `provision()`
   but with the suspend module and `{suspendStateParam: suspendStateValue}` merged into
   the module params
4. Execute playbook via `executePlaybook()`
5. Map result

**resume(BridgeRequest):**
Same flow using `resumeStateParam`/`resumeStateValue`. Resume module defaults to
`suspendModule` when not explicitly set (the common case).

**Manifest YAML extension:**
```yaml
bridge:
  protocol: ansible
  types:
    - id: ansible:amazon.aws.ec2_instance
      ansible:
        collection: amazon.aws
        provision_module: ec2_instance
        # ... existing fields ...
        suspend_module: ec2_instance
        suspend_state_param: state
        suspend_state_value: stopped
        resume_state_value: running
```

`AnsibleManifestExtensionParser.toModuleConfig()` reads the new fields. When
`resume_state_param` is absent, it defaults to `suspend_state_param`.

### PlaybookGenerator additions

New static methods:
- `PlaybookGenerator.suspend(String fqModule, Map<String, Object> params,
  String stateParam, String stateValue)` — generates a playbook that calls the module
  with `{stateParam: stateValue}` merged into params
- `PlaybookGenerator.resume(...)` — identical structure, different state value

These follow the existing `provision()` and `deprovision()` method patterns.

### Testing

**Terraform:**
- `TerraformSuspendCapableContractTest` extends `SuspendCapableContractTest` — provides
  a `TerraformBridgeProvider` with mock suspend config
- Unit test for `TerraformManifestExtensionParser` — round-trip YAML parsing

**Ansible:**
- `AnsibleSuspendCapableContractTest` extends `SuspendCapableContractTest` — provides
  an `AnsibleBridgeProvider` with suspend-configured module config
- Unit test for updated `AnsibleManifestExtensionParser` — verifies suspend fields are
  parsed from YAML
- `PlaybookGeneratorTest` — new test cases for `suspend()` and `resume()` playbook
  generation

**Integration:**
- Verify `BridgeProvisioner.supportsStatefulLifecycle()` returns `true` for both
  providers once they implement `SuspendCapable`
- Verify unsupported types (no suspend config) return `SuspendResult.Failed` through
  the full `BridgeProvisioner` → `BridgeProvider` chain

### Files Changed

| File | Change |
|------|--------|
| `bridge/terraform/TerraformBridgeProvider.java` | Add `SuspendCapable`, constructor param, `suspend()`/`resume()` |
| `bridge/terraform/TerraformSuspendConfig.java` | New record |
| `bridge/terraform/TerraformManifestExtensionParser.java` | New parser for `terraform:` YAML blocks |
| `bridge/ansible/AnsibleBridgeProvider.java` | Add `SuspendCapable`, `suspend()`/`resume()` |
| `bridge/ansible/AnsibleModuleConfig.java` | Add suspend/resume fields |
| `bridge/ansible/AnsibleManifestExtensionParser.java` | Parse suspend/resume fields |
| `bridge/ansible/PlaybookGenerator.java` | Add `suspend()`/`resume()` methods |
| Test files | Contract tests, parser tests, playbook tests |

## Out of Scope

- Credential resolution for suspend/resume (follows same path as provision)
- Bridge-cloud example updates (can be added in a follow-up)
- Build-time validation of suspend config in deployment modules

## References

- bridge/api/SuspendCapable.java — mixin interface
- bridge/runtime/BridgeProvisioner.java:103-135 — existing suspend/resume routing
- bridge/testing/MockSuspendCapableBridgeProvider.java — mock pattern
- bridge/testing/SuspendCapableContractTest.java — TCK base
- specs/issue-184-terraform-bridge-adapter (line 79) — deferred suspend/resume
- specs/issue-185-ansible-bridge-adapter (lines 413-418) — deferred SuspendCapable
