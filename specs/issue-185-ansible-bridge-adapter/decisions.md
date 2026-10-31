## D1: Execution model

**Choice:** CLI via `ansible-playbook` subprocess
**Alternatives:**
- Python API (direct) — faster per-call but tightly couples to Ansible Python internals and version
- Hybrid (CLI + Python) — optimises readState path but doubles complexity
**Rationale:** Matches the Terraform bridge's external-process pattern. Portable, version-decoupled — Ansible binary is a runtime dependency, not a compile-time one. JSON callback plugin provides structured output.
**Trade-offs:** Subprocess overhead per operation (~100-500ms). Acceptable for infrastructure provisioning where operations take seconds to minutes.
**Sources:** bridge/terraform/TerraformBridgeProvider.java (external process pattern), bridge/api/BridgeProvider.java (SPI contract)
**Exploration:** quick
**Status:** captured

## D2: Type discovery

**Choice:** Manifest-declared types in bridge YAML
**Alternatives:**
- Runtime collection scanning — auto-discovers via `ansible-galaxy collection list` but slower startup and harder to test
- Both (manifest + auto-discovery) — complexity of both approaches
**Rationale:** Consistent with existing bridge manifest pattern. User explicitly declares which Ansible modules to use with state/error mappings. No runtime scanning, no startup latency.
**Trade-offs:** User must manually list each module type. Acceptable — manifest is the deployment-time contract.
**Sources:** bridge/runtime/src/test/resources/META-INF/desiredstate/bridges/test-bridge.yaml (manifest format), bridge/runtime/BridgeManifestParser.java
**Exploration:** quick
**Status:** captured

## D3: Actual state reading strategy

**Choice:** Info module convention — manifest maps each resource type to a corresponding info/facts module
**Alternatives:**
- Check mode only — universal but can't extract current state values, only drift detection
- Configurable per type — maximum flexibility but more manifest complexity
**Rationale:** Ansible collections follow a convention where resource modules have companion info modules (e.g., `ec2_instance` → `ec2_instance_info`). These return structured facts suitable for state mapping. More precise than check mode.
**Trade-offs:** Not all modules have info companions. Manifest must declare the info module explicitly; if missing, readState returns NotFound.
**Sources:** Ansible collection conventions, bridge/api/BridgeStateResult.java
**Exploration:** quick
**Status:** captured

## D4: Inventory and connection handling

**Choice:** Spec-embedded with reserved key conventions
**Alternatives:**
- External inventory reference — more powerful for multi-host but requires files on disk
- Tenant-level inventory provider — separates inventory from spec but complex wiring
**Rationale:** Reserved spec keys (`_host`, `_connection`, `_become`) generate a single-host dynamic inventory JSON. Most Ansible modules for resource provisioning run locally or against one target. Simple, sufficient, no external file dependencies.
**Trade-offs:** Multi-host orchestration not supported. Acceptable — bridge operates on individual nodes, not host groups.
**Sources:** bridge/api/BridgeRequest.java (spec as Map<String,Object>)
**Exploration:** quick
**Status:** captured

## D5: Import capability

**Choice:** Yes — implement `ImportCapable` via info module
**Alternatives:**
- Defer to future issue — keeps scope smaller but misses parity with Terraform bridge
**Rationale:** Natural extension of D3's info-module pattern. `importResource()` invokes the info module with importId as a filter parameter, returns discovered state. Terraform bridge implements this for brownfield adoption.
**Trade-offs:** Slightly larger scope. Minimal — import reuses the same info-module invocation path as readState.
**Sources:** bridge/api/ImportCapable.java, bridge/terraform/TerraformBridgeProvider.java (import pattern)
**Exploration:** quick
**Depends on:** D3 (info module convention)
**Status:** captured

## D6: Error classification

**Choice:** Status-based classification from Ansible task output
**Alternatives:**
- Module-specific error parsing with regex patterns — more precise but harder to maintain
**Rationale:** Ansible JSON output includes per-task `failed`/`changed`/`unreachable` status. Map: `unreachable` → TRANSIENT, `failed` with auth keywords → AUTHENTICATION, `failed` otherwise → PERMANENT. Manifest can override per-type via existing errorMapping mechanism.
**Trade-offs:** Coarser classification than module-specific parsing. Acceptable — the bridge error categories are themselves coarse (TRANSIENT/PERMANENT/AUTHENTICATION).
**Sources:** bridge/api/BridgeErrorCategory.java, bridge/api/BridgeTypeDescriptor.java (errorMapping)
**Exploration:** quick
**Status:** captured

## D7: Internal architecture

**Choice:** Thin adapter with focused helpers (~6-8 classes)
**Alternatives:**
- Process abstraction layer — adds AnsibleExecutor indirection, more testable but unnecessary for simple CLI
- Shared CLI framework — extract CliProcessBridge base, YAGNI until second CLI bridge exists
**Rationale:** `AnsibleBridgeProvider` + `PlaybookGenerator` + `AnsibleOutputParser` + `InventoryGenerator`. Mirrors Terraform bridge pattern (one provider class with protocol helpers). CLI invocation is simple enough that an abstraction layer adds indirection without value.
**Trade-offs:** Less test isolation than Option B. Mitigated by bridge TCK contract tests and testable helper classes.
**Sources:** bridge/terraform/TerraformBridgeProvider.java (single provider pattern), bridge/testing/BridgeProviderContractTest.java (TCK)
**Exploration:** quick
**Status:** captured
