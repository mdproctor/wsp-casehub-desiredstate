# TypedFaultPolicy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #112 — feat: eliminate NodeType redundancy in ThresholdFaultPolicy.tier()
**Issue group:** #112

**Goal:** Introduce `TypedFaultPolicy` sub-interface that bundles `FaultPolicy` with its output `NodeType`, eliminating the redundant `NodeType` parameter from `tier()` and the fragile `probeReviewNodeType()` from the annotations recorder.

**Architecture:** `TypedFaultPolicy extends FaultPolicy` with `outputNodeType()`. `ReviewSpecFactory` gains `default nodeType()` for probe-at-construction. `addReviewNode` returns `TypedFaultPolicy`. `ThresholdFaultPolicy.Tier` loses its separate `NodeType` field. Runtime consistency assertion guards probe-vs-actual mismatch.

**Tech Stack:** Java 21, Quarkus build extension, Maven multi-module

## Global Constraints

- Pre-release platform — breaking API changes acceptable
- All changes must compile and pass `mvn --batch-mode install`
- Use `ide_replace_text_in_file` for Java edits (IntelliJ hook enforced)
- `TypedFaultPolicy` and `FaultEvent.probe()` live in `io.casehub.desiredstate.api`

---

## Batch 1: API types + ThresholdFaultPolicy + addReviewNode + all callers

All changes in this batch must land atomically — the `tier()` signature change breaks all callers.

### Task 1: TypedFaultPolicy, FaultEvent.probe(), ReviewSpecFactory.nodeType()

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/TypedFaultPolicy.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/FaultEvent.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/ReviewSpecFactory.java`
- Test: `runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java` (new test methods)

**Interfaces:**
- Produces: `TypedFaultPolicy.outputNodeType()`, `TypedFaultPolicy.of(NodeType, FaultPolicy)`, `FaultEvent.probe()`, `ReviewSpecFactory.nodeType()`

- [ ] **Step 1: Write failing test for TypedFaultPolicy.of()**

In `runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java`, add:

```java
@Test
void typedFaultPolicy_of_wrapsDelegate() {
    NodeType type = NodeType.of("test-type");
    FaultPolicy delegate = (t, e, g, a) -> List.of();
    TypedFaultPolicy typed = TypedFaultPolicy.of(type, delegate);

    assertThat(typed.outputNodeType()).isEqualTo(type);
    assertThat(typed.onFault("t1", new FaultEvent(NodeId.of("n1"), FaultType.PROVISION_FAILED, "x"),
            graphFactory.of(List.of(), List.of()), ActualState.EMPTY)).isEmpty();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl runtime -Dtest=ThresholdFaultPolicyTest#typedFaultPolicy_of_wrapsDelegate`
Expected: FAIL — `TypedFaultPolicy` does not exist

- [ ] **Step 3: Create TypedFaultPolicy.java**

Create `api/src/main/java/io/casehub/desiredstate/api/TypedFaultPolicy.java`:

```java
package io.casehub.desiredstate.api;

import java.util.List;

public interface TypedFaultPolicy extends FaultPolicy {

    NodeType outputNodeType();

    static TypedFaultPolicy of(NodeType nodeType, FaultPolicy delegate) {
        return new TypedFaultPolicy() {
            @Override public NodeType outputNodeType() { return nodeType; }
            @Override public List<GraphMutation> onFault(String tenancyId, FaultEvent event,
                    DesiredStateGraph current, ActualState actual) {
                return delegate.onFault(tenancyId, event, current, actual);
            }
        };
    }
}
```

- [ ] **Step 4: Add FaultEvent.probe()**

In `api/src/main/java/io/casehub/desiredstate/api/FaultEvent.java`, add static factory:

```java
public static FaultEvent probe() {
    return new FaultEvent(NodeId.of("__probe__"), FaultType.PROVISION_FAILED, "probe");
}
```

- [ ] **Step 5: Add ReviewSpecFactory.nodeType() default**

In `api/src/main/java/io/casehub/desiredstate/api/ReviewSpecFactory.java`, add default method:

```java
default NodeType nodeType() {
    return create(FaultEvent.probe(), null).nodeType();
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn --batch-mode test -pl runtime -Dtest=ThresholdFaultPolicyTest#typedFaultPolicy_of_wrapsDelegate`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/TypedFaultPolicy.java \
  api/src/main/java/io/casehub/desiredstate/api/FaultEvent.java \
  api/src/main/java/io/casehub/desiredstate/api/ReviewSpecFactory.java \
  runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java
git commit -m "feat(#112): TypedFaultPolicy, FaultEvent.probe(), ReviewSpecFactory.nodeType()

Refs #112"
```

### Task 2: addReviewNode return type + consistency assertion + ThresholdFaultPolicy changes + all callers

**Files:**
- Modify: `api/src/main/java/io/casehub/desiredstate/api/FaultPolicy.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/ThresholdFaultPolicy.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java`
- Modify: `runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java` (17 tier() calls)
- Modify: `examples/pipeline/src/test/java/io/casehub/desiredstate/example/pipeline/PipelineTest.java` (4 tier() calls)
- Test: new test method for consistency assertion

**Interfaces:**
- Consumes: `TypedFaultPolicy`, `TypedFaultPolicy.of()`, `FaultEvent.probe()`, `ReviewSpecFactory.nodeType()`
- Produces: `FaultPolicy.addReviewNode() → TypedFaultPolicy`, `Builder.tier(int, TypedFaultPolicy)`, `Tier(int, TypedFaultPolicy)`

- [ ] **Step 1: Write failing test for addReviewNode returning TypedFaultPolicy**

In `ThresholdFaultPolicyTest.java`, add:

```java
@Test
void addReviewNode_returnsTypedFaultPolicy_withCorrectNodeType() {
    TypedFaultPolicy policy = FaultPolicy.addReviewNode(
            (event, current) -> new TestReviewSpec(event.node(), event.detail()));
    assertThat(policy.outputNodeType()).isEqualTo(REVIEW);
}
```

- [ ] **Step 2: Write failing test for consistency assertion**

```java
@Test
void addReviewNode_inconsistentNodeType_throwsOnFault() {
    ReviewSpecFactory inconsistentFactory = new ReviewSpecFactory() {
        private boolean probing = true;
        @Override public NodeSpec create(FaultEvent event, DesiredStateGraph current) {
            if (probing) { probing = false; return new TestReviewSpec(event.node(), "probe"); }
            return new TestNodeSpec(NodeType.of("wrong-type"));
        }
        @Override public NodeType nodeType() { probing = true; return create(FaultEvent.probe(), null).nodeType(); }
    };
    TypedFaultPolicy policy = FaultPolicy.addReviewNode(inconsistentFactory);
    var graph = graphWith("n1", TARGET);
    var event = new FaultEvent(NodeId.of("n1"), FaultType.PROVISION_FAILED, "fail");

    assertThatThrownBy(() -> policy.onFault("t1", event, graph, ActualState.EMPTY))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("consistent NodeType");
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl runtime -Dtest=ThresholdFaultPolicyTest#addReviewNode_returnsTypedFaultPolicy_withCorrectNodeType+addReviewNode_inconsistentNodeType_throwsOnFault`
Expected: FAIL — addReviewNode still returns FaultPolicy, not TypedFaultPolicy

- [ ] **Step 4: Change addReviewNode return type + add consistency assertion**

In `api/src/main/java/io/casehub/desiredstate/api/FaultPolicy.java`, replace the `addReviewNode` method body. Use `ide_replace_text_in_file`:

Search: the entire `addReviewNode` method
Replace with:

```java
static TypedFaultPolicy addReviewNode(ReviewSpecFactory specFactory) {
    NodeType reviewType = specFactory.nodeType();
    return new TypedFaultPolicy() {
        @Override public NodeType outputNodeType() { return reviewType; }
        @Override public List<GraphMutation> onFault(String tenancyId, FaultEvent event,
                DesiredStateGraph current, ActualState actual) {
            NodeSpec reviewSpec = specFactory.create(event, current);
            if (!reviewSpec.nodeType().equals(reviewType)) {
                throw new IllegalStateException(
                    "ReviewSpecFactory.nodeType() probe returned " + reviewType
                    + " but create() produced spec with nodeType " + reviewSpec.nodeType()
                    + " — factory must return a consistent NodeType");
            }
            NodeId   reviewId   = NodeId.of(reviewType.value() + "-" + event.node().value());
            if (current.nodes().containsKey(reviewId)) {
                return List.of();
            }
            DesiredNode node = new DesiredNode(reviewId, reviewSpec, HumanGating.ALL);
            return GraphMutations.addNodeDependingOn(node, event.node());
        }
    };
}
```

- [ ] **Step 5: Change ThresholdFaultPolicy.Tier record**

In `api/src/main/java/io/casehub/desiredstate/api/ThresholdFaultPolicy.java`, use `ide_replace_text_in_file`:

Search: `public record Tier(int threshold, FaultPolicy action, NodeType nodeType)`
Replace: `public record Tier(int threshold, TypedFaultPolicy action)`

Search: `Objects.requireNonNull(action, "action is required");`
  + the line `Objects.requireNonNull(nodeType, "nodeType is required");`
Replace: `Objects.requireNonNull(action, "action is required");`

- [ ] **Step 6: Change Builder.tier() signature**

Search: `public Builder tier(int threshold, FaultPolicy action, NodeType nodeType)`
Replace: `public Builder tier(int threshold, TypedFaultPolicy action)`

Search: `this.tiers.add(new Tier(threshold, action, nodeType));`
Replace: `this.tiers.add(new Tier(threshold, action));`

- [ ] **Step 7: Update ignoreTypes assembly**

Search: `merged.add(tier.nodeType());`
Replace: `merged.add(tier.action().outputNodeType());`

- [ ] **Step 8: Update graph-presence guard**

Search: `NodeType previousNodeType = tiers.get(i - 1).nodeType();`
Replace: `NodeType previousNodeType = tiers.get(i - 1).action().outputNodeType();`

- [ ] **Step 9: Migrate ThresholdFaultPolicyTest — 10 addReviewNode callers**

Use `ide_replace_text_in_file` on `runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java`:

For each `addReviewNode` caller, drop the trailing NodeType from `tier()`:

Search: `FaultPolicy.addReviewNode(\n                        (event, current) -> new TestReviewSpec(event.node(), event.detail())), REVIEW)`
Replace: `FaultPolicy.addReviewNode(\n                        (event, current) -> new TestReviewSpec(event.node(), event.detail())))`

Do for all patterns — there are multiple variations (REVIEW, AI_REVIEW, HUMAN_REVIEW).

- [ ] **Step 10: Migrate ThresholdFaultPolicyTest — 7 raw lambda callers**

For each raw lambda caller, wrap in `TypedFaultPolicy.of()`:

Search: `(t, e, g, a) -> List.of(), NodeType.of("x")`
Replace: `TypedFaultPolicy.of(NodeType.of("x"), (t, e, g, a) -> List.of())`

Search: `(t, e, g, a) -> List.of(), NodeType.of("escalation")`
Replace: `TypedFaultPolicy.of(NodeType.of("escalation"), (t, e, g, a) -> List.of())`

Search: `(t, e, g, a) -> List.of(), AI_REVIEW`
Replace: `TypedFaultPolicy.of(AI_REVIEW, (t, e, g, a) -> List.of())`

Search: `(t, e, g, a) -> List.of(), HUMAN_REVIEW`
Replace: `TypedFaultPolicy.of(HUMAN_REVIEW, (t, e, g, a) -> List.of())`

- [ ] **Step 11: Migrate PipelineTest — 4 addReviewNode callers**

In `examples/pipeline/src/test/java/io/casehub/desiredstate/example/pipeline/PipelineTest.java`:

Drop trailing NodeType from each `tier()` call — 4 sites (lines 319, 321, 375, 377).

Search: `FaultPolicy.addReviewNode(\n                        (event, current) -> new AiReviewSpec(event.node(), event.detail())), PipelineNodeTypes.AI_REVIEW)`
Replace: `FaultPolicy.addReviewNode(\n                        (event, current) -> new AiReviewSpec(event.node(), event.detail())))`

Search: `FaultPolicy.addReviewNode(\n                        (event, current) -> new HumanReviewSpec(event.node(), event.detail(), "Escalated")), PipelineNodeTypes.HUMAN_REVIEW)`
Replace: `FaultPolicy.addReviewNode(\n                        (event, current) -> new HumanReviewSpec(event.node(), event.detail(), "Escalated")))`

- [ ] **Step 12: Simplify DesiredStateGraphRecorder — remove probeReviewNodeType()**

In `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java`:

Remove the `probeReviewNodeType()` call and trailing `tierNodeType` argument:

Search: `NodeType tierNodeType = probeReviewNodeType(instance, reviewMethod);`
Remove this line entirely.

Search: `tierNodeType);` (the trailing argument to `builder.tier()`)
Replace: `);`  (close the `builder.tier()` call without the NodeType)

Delete the entire `probeReviewNodeType()` method (lines 230-241).

- [ ] **Step 13: Build and verify all tests pass**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all 14 modules compile and pass

- [ ] **Step 14: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/FaultPolicy.java \
  api/src/main/java/io/casehub/desiredstate/api/ThresholdFaultPolicy.java \
  annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java \
  runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java \
  examples/pipeline/src/test/java/io/casehub/desiredstate/example/pipeline/PipelineTest.java
git commit -m "feat(#112): TypedFaultPolicy — tier() uses outputNodeType(), probeReviewNodeType() eliminated

addReviewNode returns TypedFaultPolicy with eagerly-captured NodeType.
ThresholdFaultPolicy.Tier loses separate NodeType field — derived from action.
Runtime consistency assertion guards probe-vs-actual NodeType mismatch.
DesiredStateGraphRecorder.probeReviewNodeType() deleted.

Refs #112"
```

## Batch 2: Documentation

### Task 3: Update CLAUDE.md and consumer-guide

**Files:**
- Modify: `CLAUDE.md`
- Modify: `docs/guides/consumer-guide.md`

- [ ] **Step 1: Update CLAUDE.md**

Add `TypedFaultPolicy` to Core Runtime Types table. Update `ThresholdFaultPolicy` entry — `Tier` record loses NodeType field. Update `FaultPolicy` in Core SPIs — `addReviewNode` returns `TypedFaultPolicy`. Add `FaultEvent.probe()`.

- [ ] **Step 2: Update consumer-guide.md**

Update `tier()` examples to drop trailing `NodeType`. Add `TypedFaultPolicy` to API reference. Update `FaultPolicy.addReviewNode` signature description.

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md docs/guides/consumer-guide.md
git commit -m "docs(#112): update CLAUDE.md and consumer-guide for TypedFaultPolicy

Refs #112"
```

## References

- [2026-08-24-typedfaultpolicy-design.md](/Users/mdproctor/claude/public/casehub-desiredstate/specs/issue-112-reviewnodepolicy-tier/2026-08-24-typedfaultpolicy-design.md) — design spec this plan implements
- [ThresholdFaultPolicy.java](/Users/mdproctor/claude/casehub/desiredstate/api/src/main/java/io/casehub/desiredstate/api/ThresholdFaultPolicy.java) — Tier, Builder, ignoreTypes, graph-presence guard
- [FaultPolicy.java](/Users/mdproctor/claude/casehub/desiredstate/api/src/main/java/io/casehub/desiredstate/api/FaultPolicy.java) — addReviewNode
- [ReviewSpecFactory.java](/Users/mdproctor/claude/casehub/desiredstate/api/src/main/java/io/casehub/desiredstate/api/ReviewSpecFactory.java) — gaining nodeType()
- [FaultEvent.java](/Users/mdproctor/claude/casehub/desiredstate/api/src/main/java/io/casehub/desiredstate/api/FaultEvent.java) — gaining probe()
- [DesiredStateGraphRecorder.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java) — probeReviewNodeType() elimination
- [ThresholdFaultPolicyTest.java](/Users/mdproctor/claude/casehub/desiredstate/runtime/src/test/java/io/casehub/desiredstate/api/ThresholdFaultPolicyTest.java) — 10 addReviewNode + 7 raw lambda callers
- [PipelineTest.java](/Users/mdproctor/claude/casehub/desiredstate/examples/pipeline/src/test/java/io/casehub/desiredstate/example/pipeline/PipelineTest.java) — 4 addReviewNode callers
- [decisions.md](/Users/mdproctor/claude/public/casehub-desiredstate/specs/issue-112-reviewnodepolicy-tier/decisions.md) — D1-D4
- [GitHub #112](https://github.com/casehubio/casehub-desiredstate/issues/112) — focal issue
- [GitHub #113](https://github.com/casehubio/casehub-desiredstate/issues/113) — @Tier(nodeType) follow-up
