# Design Journal — issue-74-canonical-deployment-topologies

## 2026-08-31 — Session 1: Foundation + Types + Modules + Exemplars

Implemented issues #75–#81 of the canonical deployment topologies epic (#74).

**Design decisions made during implementation:**

- **NodeSpecFactory backendId via spec map** — the factory extracts `backendId` from the spec map
  if present (injected by the caller from `YamlNode.backendId()`), else uses a config default.
  This avoids changing the `NodeSpecFactory` interface signature while still supporting per-node
  backend overrides.

- **YamlFaultPolicyBuilder bypasses registry** — the fault policy builder needs a custom coercion
  mapper (String coercion + NodeId deserializer) that differs from the registry's standard
  ObjectMapper. It loads classes directly from `typeRegistryMap` instead of using the registry's
  factory. This is intentional — the fault policy path has specialised deserialization needs.

- **Dynamic handledTypes via sealed permits** — `InfraNodeProvisioner`, `InfraActualStateAdapter`,
  and `InfraFaultPolicy` all derive their handled types from `InfraNodeSpec.getPermittedSubclasses()`
  + `@NodeTypeId` reflection. New InfraNodeSpec variants are automatically handled without code
  changes to consumers.

- **Service-mesh sidecar-injection rule uses flatId** — module-prefixed node IDs contain `.` which
  is a reserved separator. Rule actions that generate node IDs must use `${match.*.flatId}` to
  replace dots with dashes.

- **Module invariants should check existence, not direct dependency** — the `mesh-requires-control-plane`
  invariant was changed from requiring a `directDep` edge from each proxy to the control plane
  (too strict — the dependency is architectural, not graph-structural) to checking that a
  `mesh_control_plane` node exists.

**Pre-existing issues fixed:**
- `FaultPolicy.addReviewNode` API alignment in InfraFaultPolicy, KubernetesFaultPolicy,
  DeploymentFaultPolicy, IoTFaultPolicy (old 3-arg → new ReviewSpecFactory 1-arg)
