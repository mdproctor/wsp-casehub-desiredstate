# addReviewNode Deprecated Method Cleanup — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #103 — chore: ops repo migration for NodeSpec.nodeType() + DesiredNode refactoring
**Issue group:** #103

**Goal:** Remove the deprecated 2-arg `FaultPolicy.addReviewNode(NodeType, ReviewSpecFactory)` method and migrate all 7 desiredstate callers to the 1-arg form.

**Architecture:** Zero-behavior-change migration. The deprecated 2-arg method already delegates to the 1-arg form, ignoring the `NodeType` argument. All callers are updated atomically — callers first, then remove the deprecated method.

**Tech Stack:** Java 21, Quarkus build extension, Maven multi-module

## Global Constraints

- Pre-release platform — breaking changes acceptable
- No ops repo changes — file an issue instead
- All changes must compile and pass `mvn --batch-mode install`

---

## Batch 1: Migrate callers and remove deprecated method

### Task 1: Migrate all 7 desiredstate callers to 1-arg addReviewNode

**Files:**
- Modify: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/QuarantineFaultPolicy.java:23-25`
- Modify: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/SchemaDriftFaultPolicy.java:24-26`
- Modify: `examples/pipeline/src/test/java/io/casehub/desiredstate/example/pipeline/PipelineTest.java:319-322,375-378`
- Modify: `runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java:237`

**Interfaces:**
- Consumes: `FaultPolicy.addReviewNode(ReviewSpecFactory)` — 1-arg form (already exists)
- Produces: No new interfaces — callers switch from 2-arg to 1-arg

- [ ] **Step 1: Run existing tests to establish green baseline**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules compile and all tests pass.

- [ ] **Step 2: Migrate QuarantineFaultPolicy.java**

In `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/QuarantineFaultPolicy.java`, change line 23-25 from:

```java
    private final FaultPolicy   reviewPolicy = FaultPolicy.addReviewNode(
            PipelineNodeTypes.HUMAN_REVIEW,
            (event, graph) -> new HumanReviewSpec(event.node(), event.detail(), "Quarantined data requires manual review"));
```

to:

```java
    private final FaultPolicy   reviewPolicy = FaultPolicy.addReviewNode(
            (event, graph) -> new HumanReviewSpec(event.node(), event.detail(), "Quarantined data requires manual review"));
```

- [ ] **Step 3: Migrate SchemaDriftFaultPolicy.java**

In `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/SchemaDriftFaultPolicy.java`, change lines 24-26 from:

```java
    private final FaultPolicy reviewPolicy = FaultPolicy.addReviewNode(
            PipelineNodeTypes.HUMAN_REVIEW,
            (event, graph) -> new HumanReviewSpec(event.node(), event.detail(), "Schema drift requires approval"));
```

to:

```java
    private final FaultPolicy reviewPolicy = FaultPolicy.addReviewNode(
            (event, graph) -> new HumanReviewSpec(event.node(), event.detail(), "Schema drift requires approval"));
```

- [ ] **Step 4: Migrate PipelineTest.java (4 call sites)**

In `examples/pipeline/src/test/java/io/casehub/desiredstate/example/pipeline/PipelineTest.java`:

Change lines 319-322 from:
```java
                .tier(4, FaultPolicy.addReviewNode(PipelineNodeTypes.AI_REVIEW,
                        (event, current) -> new AiReviewSpec(event.node(), event.detail())), PipelineNodeTypes.AI_REVIEW)
                .tier(7, FaultPolicy.addReviewNode(PipelineNodeTypes.HUMAN_REVIEW,
                        (event, current) -> new HumanReviewSpec(event.node(), event.detail(), "Escalated")), PipelineNodeTypes.HUMAN_REVIEW)
```

to:
```java
                .tier(4, FaultPolicy.addReviewNode(
                        (event, current) -> new AiReviewSpec(event.node(), event.detail())), PipelineNodeTypes.AI_REVIEW)
                .tier(7, FaultPolicy.addReviewNode(
                        (event, current) -> new HumanReviewSpec(event.node(), event.detail(), "Escalated")), PipelineNodeTypes.HUMAN_REVIEW)
```

Change lines 375-378 (identical pattern — second test method):
```java
                .tier(4, FaultPolicy.addReviewNode(PipelineNodeTypes.AI_REVIEW,
                        (event, current) -> new AiReviewSpec(event.node(), event.detail())), PipelineNodeTypes.AI_REVIEW)
                .tier(7, FaultPolicy.addReviewNode(PipelineNodeTypes.HUMAN_REVIEW,
                        (event, current) -> new HumanReviewSpec(event.node(), event.detail(), "Escalated")), PipelineNodeTypes.HUMAN_REVIEW)
```

to:
```java
                .tier(4, FaultPolicy.addReviewNode(
                        (event, current) -> new AiReviewSpec(event.node(), event.detail())), PipelineNodeTypes.AI_REVIEW)
                .tier(7, FaultPolicy.addReviewNode(
                        (event, current) -> new HumanReviewSpec(event.node(), event.detail(), "Escalated")), PipelineNodeTypes.HUMAN_REVIEW)
```

- [ ] **Step 5: Migrate ThresholdFaultPolicyTest.java (1 call site)**

In `runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java`, change line 237-238 from:

```java
                                         .tier(2, FaultPolicy.addReviewNode(REVIEW,
                                                 (event, current) -> new TestReviewSpec(event.node(), event.detail())), REVIEW)
```

to:

```java
                                         .tier(2, FaultPolicy.addReviewNode(
                                                 (event, current) -> new TestReviewSpec(event.node(), event.detail())), REVIEW)
```

- [ ] **Step 6: Remove deprecated 2-arg method from FaultPolicy.java**

In `api/src/main/java/io/casehub/desiredstate/api/FaultPolicy.java`, remove lines 21-24:

```java
    @Deprecated(forRemoval = true)
    static FaultPolicy addReviewNode(NodeType reviewType, ReviewSpecFactory specFactory) {
        return addReviewNode(specFactory);
    }
```

Also remove the now-unused `NodeType` import if no other reference to it remains in the file. (Check with `ide_find_references` — `NodeType` is still used in the 1-arg method body at line 11, so the import stays.)

- [ ] **Step 7: Build and verify all tests pass**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — zero behavior change confirmed by all existing tests passing.

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/FaultPolicy.java \
  examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/QuarantineFaultPolicy.java \
  examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/SchemaDriftFaultPolicy.java \
  examples/pipeline/src/test/java/io/casehub/desiredstate/example/pipeline/PipelineTest.java \
  runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java
git commit -m "chore(#103): remove deprecated 2-arg addReviewNode, migrate callers to 1-arg

Remove FaultPolicy.addReviewNode(NodeType, ReviewSpecFactory) — the NodeType
argument was already ignored (delegated to 1-arg form). Migrate 7 callers:
QuarantineFaultPolicy, SchemaDriftFaultPolicy, PipelineTest (4), ThresholdFaultPolicyTest (1).

Zero behavior change — all existing tests pass unchanged.

Refs #103"
```

## Batch 2: File follow-up issues

### Task 2: File ops issue for remaining 2-arg callers

**Files:** None (GitHub API only)

- [ ] **Step 1: Create ops issue**

```bash
gh issue create --repo casehubio/casehub-ops \
  --title "chore: migrate addReviewNode 2-arg to 1-arg" \
  --body "$(cat <<'EOF'
## Context

casehub-desiredstate removed the deprecated `FaultPolicy.addReviewNode(NodeType, ReviewSpecFactory)` method (#103). The 1-arg form `addReviewNode(ReviewSpecFactory)` derives `NodeType` from `ReviewSpec.nodeType()`.

## Scope

4 FaultPolicy classes still use the 2-arg form — mechanical removal of the first `NodeType` argument:

| File | Module |
|------|--------|
| `KubernetesFaultPolicy.java` | app |
| `IoTFaultPolicy.java` | iot |
| `DeploymentFaultPolicy.java` | deployment |
| `InfraFaultPolicy.java` | infra |

## Migration pattern

```java
// Before:
.tier(3, FaultPolicy.addReviewNode(K8S_REVIEW, factory), K8S_REVIEW)

// After:
.tier(3, FaultPolicy.addReviewNode(factory), K8S_REVIEW)
```

Compiler catches every site. Zero behavior change — the 2-arg form already ignored the NodeType argument.

Refs casehubio/casehub-desiredstate#103
EOF
)"
```

### Task 3: File follow-up issue for tier() NodeType redundancy

**Files:** None (GitHub API only)

- [ ] **Step 1: Create desiredstate follow-up issue**

```bash
gh issue create --repo casehubio/casehub-desiredstate \
  --title "feat: eliminate NodeType redundancy in ThresholdFaultPolicy.tier()" \
  --body "$(cat <<'EOF'
## Motivation

After removing the deprecated 2-arg `addReviewNode` (#103), the same NodeType redundancy remains in `ThresholdFaultPolicy.Builder.tier(int threshold, FaultPolicy action, NodeType nodeType)` — the `NodeType` must match the review spec's `nodeType()` but this invariant cannot be enforced at compile time.

### Failure modes from mismatch

1. **Infinite escalation:** Tier NodeTypes are auto-added to `ignoreTypes` (`ThresholdFaultPolicy` line 35). A wrong NodeType means the policy fires on its own review nodes — creating an infinite escalation loop.
2. **Skipped tiers:** Before escalating to tier N, the policy checks that tier N-1's review node is present among the faulted node's dependents (lines 92-97). A wrong NodeType breaks this check — tiers are skipped or escalation happens prematurely.

### Proposed solution

A `ReviewNodePolicy` sub-interface of `FaultPolicy` that carries the output `NodeType`:

```java
interface ReviewNodePolicy extends FaultPolicy {
    NodeType reviewNodeType();
}

static ReviewNodePolicy addReviewNode(ReviewSpecFactory specFactory) {
    // derives both FaultPolicy behavior and NodeType from specFactory
}
```

`ThresholdFaultPolicy.Builder.tier()` accepts `ReviewNodePolicy` and derives the `NodeType` automatically:

```java
builder.tier(4, FaultPolicy.addReviewNode(
    (e, g) -> new AiReviewSpec(...)))  // no NodeType needed anywhere
```

### Additional cleanup

This would eliminate `DesiredStateGraphRecorder.probeReviewNodeType()` — a fragile method that calls the factory with a synthetic `FaultEvent` and **null** graph to discover a type that should be statically available. Any review method that touches the graph NPEs during probing, silently falling back to method-name-derived NodeType.

Refs #103
EOF
)"
```

- [ ] **Step 2: Commit any remaining workspace changes**

```bash
git -C "$WORKSPACE" add plans/ specs/
git -C "$WORKSPACE" commit -m "wip(plan): implementation plan for #103 Refs #103"
```

## References

- [2026-08-23-addreviewnode-cleanup-design.md](/Users/mdproctor/claude/public/casehub-desiredstate/specs/issue-103-ops-nodespec-migration/2026-08-23-addreviewnode-cleanup-design.md) — design spec this plan implements
- [FaultPolicy.java:8-24](/Users/mdproctor/claude/casehub/desiredstate/api/src/main/java/io/casehub/desiredstate/api/FaultPolicy.java) — deprecated method and 1-arg method
- [QuarantineFaultPolicy.java:23](/Users/mdproctor/claude/casehub/desiredstate/examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/QuarantineFaultPolicy.java) — 2-arg caller
- [SchemaDriftFaultPolicy.java:24](/Users/mdproctor/claude/casehub/desiredstate/examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/SchemaDriftFaultPolicy.java) — 2-arg caller
- [PipelineTest.java:319-322,375-378](/Users/mdproctor/claude/casehub/desiredstate/examples/pipeline/src/test/java/io/casehub/desiredstate/example/pipeline/PipelineTest.java) — 4 test callers
- [ThresholdFaultPolicyTest.java:237](/Users/mdproctor/claude/casehub/desiredstate/runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java) — 1 test caller
- [decisions.md](/Users/mdproctor/claude/public/casehub-desiredstate/specs/issue-103-ops-nodespec-migration/decisions.md) — D1 decision with review findings
- [GitHub #103](https://github.com/casehubio/casehub-desiredstate/issues/103) — focal issue
- [GitHub #102](https://github.com/casehubio/casehub-desiredstate/issues/102) — parent annotation issue
