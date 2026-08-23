# addReviewNode Deprecated Method Cleanup — Design Spec

**Date:** 2026-08-23
**Issue:** casehubio/casehub-desiredstate#103
**Status:** Draft

## Motivation

Issue #103's core scope — `NodeSpec.nodeType()` addition and `DesiredNode` 4-arg to 3-arg
simplification — has already landed in both the desiredstate and ops repos. The remaining
cleanup is removing the deprecated `FaultPolicy.addReviewNode(NodeType, ReviewSpecFactory)`
method and migrating all callers to the 1-arg form. This desiredstate-side cleanup
originates from #102's prerequisite refactoring (commit `ac1e6ea`), extended here under
#103 to complete the migration trail.

The 2-arg method is already marked `@Deprecated(forRemoval = true)` and **ignores its
`NodeType` argument entirely** — it delegates to the 1-arg form. This migration has zero
behavior risk. It removes dead parameters from call sites. If any caller was passing a
`NodeType` that differs from their `ReviewSpec.nodeType()`, they have a latent bug that
this migration will surface.

## Scope

**In scope:**
- Remove the deprecated 2-arg `addReviewNode` from `FaultPolicy.java`
- Migrate 7 desiredstate call sites to 1-arg form
- File GitHub issue on casehub-ops for their 4 remaining 2-arg callers
- File follow-up issue for `ThresholdFaultPolicy.tier()` NodeType redundancy

**Out of scope:**
- `ThresholdFaultPolicy.Builder.tier()` API redesign (follow-up issue)
- Ops repo code changes (separate issue)

## Changes

### 1. FaultPolicy.java (api module)

Remove the deprecated method (lines 21–24):

```java
// REMOVE:
@Deprecated(forRemoval = true)
static FaultPolicy addReviewNode(NodeType reviewType, ReviewSpecFactory specFactory) {
    return addReviewNode(specFactory);
}
```

The 1-arg method at lines 8–19 remains unchanged.

### 2. Desiredstate callers — 7 sites

Each migration: remove the leading `NodeType` argument from `addReviewNode(...)`.

| File | Module | Count | Pattern |
|------|--------|-------|---------|
| `QuarantineFaultPolicy.java` | examples/pipeline | 1 | `addReviewNode(PipelineNodeTypes.HUMAN_REVIEW, factory)` → `addReviewNode(factory)` |
| `SchemaDriftFaultPolicy.java` | examples/pipeline | 1 | Same pattern |
| `PipelineTest.java` | examples/pipeline | 4 | `addReviewNode(PipelineNodeTypes.AI_REVIEW, factory)` and `addReviewNode(PipelineNodeTypes.HUMAN_REVIEW, factory)` |
| `ThresholdFaultPolicyTest.java` | runtime | 1 | `addReviewNode(REVIEW, factory)` (line 237 only) |

### 3. Ops issue

File on casehubio/casehub-ops with:
- Title: `chore: migrate addReviewNode 2-arg to 1-arg`
- Body: 4 FaultPolicy classes, mechanical removal of NodeType argument
- Files: `KubernetesFaultPolicy.java`, `IoTFaultPolicy.java`, `DeploymentFaultPolicy.java`, `InfraFaultPolicy.java`

### 4. Follow-up issue

File on casehubio/casehub-desiredstate:
- Title: `feat: eliminate NodeType redundancy in ThresholdFaultPolicy.tier()`
- Body: `ReviewNodePolicy` sub-interface proposal. Key motivation:
  - **Eliminates `probeReviewNodeType()`** in `DesiredStateGraphRecorder.createFaultPolicy()` — a fragile probe that calls the factory with a synthetic `FaultEvent` and **null** graph; any review method that touches the graph NPEs during probing, silently falling back to method-name-derived NodeType
  - **Prevents infinite escalation:** Tier NodeTypes are auto-added to `ignoreTypes` (ThresholdFaultPolicy line 35); a mismatch means the policy fires on its own review nodes
  - **Prevents skipped tiers:** Escalation checks that tier N-1's review node is present among dependents (lines 92–97); a wrong NodeType breaks this check

## Issue Lifecycle

#103 closes when this branch merges — its stated NodeSpec+DesiredNode ops scope is
complete, and the desiredstate-side addReviewNode cleanup lands here. The ops addReviewNode
migration is tracked by the new casehub-ops issue. The `tier()` API redesign is tracked
by the new desiredstate follow-up issue.

## Testing Strategy

No new tests needed. This is a dead-parameter removal with zero behavior change.

- Run `mvn --batch-mode install` — verifies all desiredstate modules compile and pass
- Existing `ThresholdFaultPolicyTest` and `PipelineTest` cover all migrated call sites

## References

- [FaultPolicy.java](/Users/mdproctor/claude/casehub/desiredstate/api/src/main/java/io/casehub/desiredstate/api/FaultPolicy.java) — deprecated method at lines 21–24
- [#102 design spec](/Users/mdproctor/claude/casehub/desiredstate/docs/specs/issue-102-desiredstate-annotations/2026-08-20-desiredstate-annotations-design.md) — §1.3 addReviewNode simplification
- [decisions.md](/Users/mdproctor/claude/public/casehub-desiredstate/specs/issue-103-ops-nodespec-migration/decisions.md) — D1 with review findings
- [Decision review R1-02](/Users/mdproctor/reviews/casehub-desiredstate/issue-103-decision-20260823-154354/responses/reviewer-1.md) — tier() redundancy analysis
