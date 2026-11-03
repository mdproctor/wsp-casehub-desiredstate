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
