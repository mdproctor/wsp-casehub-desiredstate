## D1: Domain theme

**Choice:** Cloud infrastructure — VMs, networks, storage buckets
**Alternatives:**
- Game/fantasy theme — playful but doesn't map to the actual bridge use case
- IoT / device fleet — valid but less immediately recognisable
**Rationale:** Bridge adapters exist specifically for cloud infrastructure. A cloud domain makes the teaching example immediately legible and directly maps to the Terraform/Ansible patterns it demonstrates.
**Trade-offs:** Less playful than dungeon/expansion examples
**Sources:** Issue #208, existing examples (dungeon, pipeline, expansion)
**Exploration:** quick
**Status:** captured

## D2: Mixed graph composition

**Choice:** Bridge provisions infra (VMs, networks, storage), Java NodeProvisioner deploys applications onto the provisioned VMs
**Alternatives:**
- Monitoring/observability — valid but less canonical
- DNS/routing — valid but narrower scope
**Rationale:** Infra-then-app is the canonical cloud deployment pattern. Demonstrates the primary reason bridges exist: Java code manages application logic while bridge adapters manage infrastructure.
**Trade-offs:** None significant — this is the most natural pattern
**Sources:** Issue #208 scope, bridge/runtime BridgeProvisioner adapter pattern
**Exploration:** quick
**Status:** captured

## D3: Provider module location

**Choice:** Inside the example module — self-contained, package-private MockCloudBridgeProvider
**Alternatives:**
- Separate bridge/mock/ module — follows bridge/terraform structure but adds a module for a mock with no real consumers
**Rationale:** Teaching examples should be self-contained. The mock provider exists to demonstrate the SPI, not to be reused. A separate module adds complexity without value.
**Trade-offs:** Can't be consumed from other modules, but that's the point
**Sources:** Existing example structure (dungeon, pipeline are self-contained)
**Exploration:** quick
**Status:** captured

## D4: Terraform suspend/resume manifest extension structure

**Choice:** Attribute triplet — `suspend_attribute`, `suspend_value`, `resume_value`
**Alternatives:**
- Full state patches (`suspend_state: {key: val}`, `resume_state: {key: val}`) — more flexible for multi-attribute changes, but suspend/resume is inherently single-attribute in cloud providers
**Rationale:** Suspend/resume maps to a single state-controlling attribute in every real cloud provider (EC2 `instance_state`, RDS `engine_lifecycle_support`). The triplet is explicit, minimal, and mirrors how Terraform's `ApplyResourceChange` works — read current state, override one attribute, apply.
**Trade-offs:** Cannot set multiple attributes in a single suspend operation. If such a case arises, it would need a different desired state rather than suspend.
**Sources:** bridge/terraform/TerraformBridgeProvider.java, bridge/api/SuspendCapable.java, specs/issue-184-terraform-bridge-adapter (deferred suspend/resume as future item)
**Exploration:** quick
**Status:** captured

## D5: Ansible suspend/resume manifest extension structure

**Choice:** Module + state parameter — `suspend_module`, `suspend_state_param`, `suspend_state_value`, `resume_module`, `resume_state_param`, `resume_state_value`
**Alternatives:**
- Module only (`suspend_module`, `resume_module`) — simpler but requires dedicated modules for every suspend/resume operation, won't work with common `ec2_instance state=stopped` pattern
**Rationale:** Most Ansible modules use the same module with a state parameter for lifecycle operations (`ec2_instance state=stopped/running`). The module+param pattern handles both same-module-different-state and different-module cases without requiring wrapper roles.
**Trade-offs:** Six fields per type is verbose, but only types that support suspend need them. Types without these fields return `SuspendResult.Failed`.
**Sources:** bridge/ansible/AnsibleBridgeProvider.java, bridge/ansible/AnsibleManifestExtensionParser.java, specs/issue-185-ansible-bridge-adapter (noted SuspendCapable as "Not implemented" with per-type future path)
**Exploration:** quick
**Status:** captured
