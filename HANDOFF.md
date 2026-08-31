# Handoff — casehub-desiredstate

## Last Session

Implemented issues #75–#81 of epic #74 (Canonical deployment topologies — 5×4 matrix proving YAML frontend). Both desiredstate and ops repos modified in this slot.

**desiredstate repo (8 commits):** NodeSpecFactory SPI + NodeSpecFactoryProvider in api/, NodeSpecRegistry generalised from Class→Factory, backendId on YamlNode, scanNodeTypes guard removal (all @NodeTypeId classes discovered), jar-protocol module discovery, lifecycle+module interaction fix, factory provider wiring into recorder.

**ops repo (6 commits):** 5 new InfraNodeSpec sealed variants (LoadBalancerSpec, SidecarProxySpec, ServiceMeshControlPlaneSpec, DnsFailoverSpec, DataReplicationSpec), 3 enums, @NodeTypeId on all 14 non-generic variants, null-coalescing, InfraWrappingFactory + InfraNodeSpecFactoryProvider, dynamic handledTypes, 4 YAML modules, topology-tests module with 14 YAML exemplars + 27 compilation tests. Also fixed pre-existing FaultPolicy.addReviewNode API mismatch in 4 policy classes.

**Next:** #82 — reconciliation + live K8s deployment tests (profile-gated). Needs FailableResourceProvisioner for deterministic failure injection, stubbed backends, and optionally real K8s for live tests.

## Branch

`issue-74-canonical-deployment-topologies` — both project + workspace repos, also ops repo

## References

| Artifact | Path |
|----------|------|
| Design spec | `wsp-casehub-ops/specs/canonical-deployment-topologies/2026-08-29-canonical-deployment-topologies-design.md` |
| Decisions | `wsp-casehub-ops/specs/canonical-deployment-topologies/decisions.md` |
| Journal | `JOURNAL.md` |
| Plan | `.plan` (position 8/9, #82 active) |
