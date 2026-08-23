# Design Decisions — #103 ops repo migration cleanup

## D1: Remove deprecated 2-arg addReviewNode and migrate all callers

**Choice:** Migrate all 7 desiredstate 2-arg `addReviewNode(NodeType, ReviewSpecFactory)` callers to the 1-arg form, then remove the deprecated method from `FaultPolicy.java`. File a GitHub issue on casehub-ops for their 4 remaining 2-arg callers.
**Alternatives:**
- Keep deprecated method until ops migrates first — delays cleanup, deprecated code lingers across releases
- Remove method without filing ops issue — breaks ops silently
- Also fix the redundant `NodeType` in `ThresholdFaultPolicy.tier()` (see Follow-up below) — correct direction but larger API redesign, out of scope for mechanical cleanup
**Rationale:** Pre-release platform — breaking changes are expected and acceptable. The 2-arg method was already marked `@Deprecated(forRemoval = true)` and **already delegates to the 1-arg form, ignoring the `NodeType` argument entirely.** This migration has zero behavior risk — it removes dead parameters from call sites. If any caller was passing a `NodeType` that differs from their `ReviewSpec.nodeType()`, they already have a latent bug (the type was never used). The migration exposes this mismatch — a benefit, not a risk.
**Trade-offs:** Ops build breaks until they remove the `NodeType` argument from 4 FaultPolicy classes (`KubernetesFaultPolicy`, `IoTFaultPolicy`, `DeploymentFaultPolicy`, `InfraFaultPolicy`). Mechanical fix — compiler catches every site.
**Ops migration status:** The core #103 scope (NodeSpec.nodeType() + DesiredNode 3-arg) is already complete in ops — all ~25 NodeSpec implementations already have `nodeType()`, all DesiredNode construction sites already use the 3-arg form. Only the `addReviewNode` 2-arg cleanup remains.
**Sources:** FaultPolicy.java (api), QuarantineFaultPolicy.java, SchemaDriftFaultPolicy.java, PipelineTest.java, ThresholdFaultPolicyTest.java, #102 design spec (§1.3)
**Exploration:** quick
**Status:** captured

### Follow-up: ThresholdFaultPolicy.tier() NodeType redundancy

The `tier(int threshold, FaultPolicy action, NodeType nodeType)` signature carries the same redundancy D1 removes from `addReviewNode` — the `NodeType` must match the review spec's `nodeType()` but this invariant cannot be enforced at compile time. A `ReviewNodePolicy` sub-interface that carries the output `NodeType` would eliminate both copies of redundancy. This is a separate API enhancement — file as a new issue.
