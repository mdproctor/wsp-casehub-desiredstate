# TypedFaultPolicy — Eliminate NodeType Redundancy in tier() — Design Spec

**Date:** 2026-08-24
**Issue:** casehubio/casehub-desiredstate#112
**Status:** Draft

## Motivation

`ThresholdFaultPolicy.Builder.tier(int threshold, FaultPolicy action, NodeType nodeType)` requires
callers to pass a `NodeType` that must match the review spec's `nodeType()`. This invariant cannot
be enforced at compile time. A mismatch silently breaks two mechanisms:

1. **Infinite escalation:** Tier NodeTypes are auto-merged into `ignoreTypes` (line 33-35). A wrong
   NodeType means the policy doesn't ignore its own review nodes — infinite self-triggering.
2. **Skipped tiers:** Escalation to tier N checks that tier N-1's NodeType exists among the faulted
   node's dependents (lines 86-91). A wrong NodeType breaks this check.

The annotations module works around this with `probeReviewNodeType()` — calling the review method
with a synthetic `FaultEvent` and **null** graph. Any review method that touches the graph NPEs,
silently falling back to method-name-derived NodeType.

## Scope

**In scope:**
- `TypedFaultPolicy` sub-interface of `FaultPolicy` in the api module
- `FaultEvent.probe()` static factory for named probe constant
- `ReviewSpecFactory.nodeType()` default method
- `FaultPolicy.addReviewNode` return type narrowing to `TypedFaultPolicy`
- Runtime consistency assertion in `addReviewNode`'s `onFault()` (probe vs actual NodeType)
- `ThresholdFaultPolicy.Tier` and `Builder.tier()` signature changes
- `DesiredStateGraphRecorder.probeReviewNodeType()` elimination
- All desiredstate caller migration (17 `tier()` call sites)
- `@Customize` path acknowledgment (consumers calling `builder.tier()` must update)

**Out of scope:**
- `@Tier` annotation `nodeType` attribute for build-time validation (tracked as follow-up issue)
- Ops repo caller migration (tracked by casehub-ops#70)

---

## Changes

### 1. TypedFaultPolicy.java (api — new file)

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

### 2. FaultEvent.probe() (api — static factory)

```java
public static FaultEvent probe() {
    return new FaultEvent(NodeId.of("__probe__"), FaultType.PROVISION_FAILED, "probe");
}
```

Named probe constant for `ReviewSpecFactory.nodeType()` default. Self-documenting intent —
domain factories that override `nodeType()` never see this event.

### 3. ReviewSpecFactory.java (api — add default method)

```java
@FunctionalInterface
public interface ReviewSpecFactory {
    NodeSpec create(FaultEvent event, DesiredStateGraph current);

    default NodeType nodeType() {
        return create(FaultEvent.probe(), null).nodeType();
    }
}
```

The `@FunctionalInterface` contract is preserved — `nodeType()` is a default method. Lambda callers
are unaffected. Domain factories that are concrete classes should override `nodeType()` to return
a constant, eliminating the probe.

**Intentional behavioral change from current `probeReviewNodeType()`:** Unlike the current
`probeReviewNodeType()` which catches all exceptions and falls back to method-name-derived NodeType,
the proposed default method propagates the exception. This is intentional — a silent fallback to a
wrong NodeType is worse than a clear failure at construction time. The fallback to method-name
NodeType is the exact mismatch problem this issue aims to fix; fail-fast exposes misconfiguration
immediately.

### 4. FaultPolicy.addReviewNode (api — return type narrowing)

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

Key changes:
- `reviewType` captured once at construction time via `specFactory.nodeType()`
- **Runtime consistency assertion:** if the factory's `create()` returns a spec with a different
  `nodeType()` than the probe, throw `IllegalStateException` with clear diagnostic. This closes the
  probe-vs-actual mismatch hole that would otherwise relocate the problem from caller-provided
  NodeType to probe-vs-actual NodeType.
- **Invariant:** `ReviewSpecFactory.create()` must return a NodeSpec with the same `nodeType()`
  regardless of the `FaultEvent` and `DesiredStateGraph` arguments. This invariant is not new —
  `ThresholdFaultPolicy`'s `ignoreTypes` and predecessor checks already depend on it. The assertion
  makes it explicit and guarded.

### 5. ThresholdFaultPolicy (api — Tier record + Builder)

**Tier record:**
```java
public record Tier(int threshold, TypedFaultPolicy action) {
    public Tier {
        if (threshold < 1) throw new IllegalArgumentException("threshold must be >= 1");
        Objects.requireNonNull(action, "action is required");
    }
}
```

No separate `NodeType` — derived from `action.outputNodeType()`.

**Builder.tier():**
```java
public Builder tier(int threshold, TypedFaultPolicy action) {
    this.tiers.add(new Tier(threshold, action));
    return this;
}
```

**ignoreTypes assembly** (constructor):
```java
for (Tier tier : builder.tiers) {
    merged.add(tier.action().outputNodeType());
}
```

**Graph-presence guard** (onFault):
```java
NodeType previousNodeType = tiers.get(i - 1).action().outputNodeType();
```

### 6. DesiredStateGraphRecorder (annotations/runtime — simplification)

**Before** (actual code, lines 134-148):
```java
NodeType tierNodeType = probeReviewNodeType(instance, reviewMethod);
builder.tier(td.threshold(),
        io.casehub.desiredstate.api.FaultPolicy.addReviewNode(
                (event, graph) -> {
                    try {
                        return (NodeSpec) reviewMethod.invoke(instance, event, graph);
                    } catch (Exception e) {
                        throw new RuntimeException("Review method invocation failed: "
                                + reviewMethod.getName(), e);
                    }
                }),
        tierNodeType);
```

**After:**
```java
builder.tier(td.threshold(),
        io.casehub.desiredstate.api.FaultPolicy.addReviewNode(
                (event, graph) -> {
                    try {
                        return (NodeSpec) reviewMethod.invoke(instance, event, graph);
                    } catch (Exception e) {
                        throw new RuntimeException("Review method invocation failed: "
                                + reviewMethod.getName(), e);
                    }
                }));
```

`probeReviewNodeType()` method is deleted. `addReviewNode` returns `TypedFaultPolicy` which carries
the NodeType via `ReviewSpecFactory.nodeType()` default probe. The existing try/catch on the
reflective invocation is preserved — only the separate probe and trailing `NodeType` argument are removed.

### 7. Caller migration — 17 sites

**10 callers using `addReviewNode` (production + test):** Drop the trailing `NodeType` argument from `tier()`.

```java
// Before:
.tier(4, FaultPolicy.addReviewNode(factory), PipelineNodeTypes.AI_REVIEW)

// After:
.tier(4, FaultPolicy.addReviewNode(factory))
```

**7 callers using raw lambdas (test-only):** Wrap via `TypedFaultPolicy.of()`.

```java
// Before:
.tier(1, (t, e, g, a) -> List.of(), NodeType.of("x"))

// After:
.tier(1, TypedFaultPolicy.of(NodeType.of("x"), (t, e, g, a) -> List.of()))
```

**`@Customize` path:** Any `@Customize` method that calls `builder.tier(int, FaultPolicy, NodeType)`
directly (the 3-arg signature being removed) must update to `tier(int, TypedFaultPolicy)`. No current
production code in this repo uses `@Customize` for fault policy builders. The `@Customize` annotation
is public API — consumers must be aware of the signature change.

---

## Testing Strategy

No new test classes needed. Existing tests verify the same behavior through the new API surface.

**Adapted tests (17 callers):**
- **ThresholdFaultPolicyTest:** 10 callers adapted — verifies threshold mechanics, ignoreTypes,
  graph-presence guard, multi-tier escalation all still work
- **PipelineTest:** 4 callers adapted — verifies end-to-end fault escalation in example domain
- **DesiredStateAnnotationsProcessorTest / FaultPolicyWiringTest:** verify annotation-driven
  fault policy creation still produces correct ThresholdFaultPolicy beans

**Required new tests:**
- **Probe-at-construction test:** `addReviewNode(factory).outputNodeType()` must match the factory's
  spec's `nodeType()` — verifies the probe path works correctly
- **Consistency assertion test:** A factory that returns different `nodeType()` from probe vs
  `create()` must throw `IllegalStateException` — verifies the runtime guard

**Verification:** `mvn --batch-mode install` — full build across all 14 modules.

---

## CLAUDE.md Updates

Update the `ThresholdFaultPolicy` entry in Core Runtime Types:
- `Tier` record: remove "NodeType" from the field list
- Add `TypedFaultPolicy` to the types table
- Update `FaultPolicy.addReviewNode` signature in Core SPIs
- Add `FaultEvent.probe()` to the types table

Update consumer-guide.md:
- `tier()` examples drop the trailing `NodeType`
- Add `TypedFaultPolicy` to the API reference

## References

- [ThresholdFaultPolicy.java](/Users/mdproctor/claude/casehub/desiredstate/api/src/main/java/io/casehub/desiredstate/api/ThresholdFaultPolicy.java) — Tier record, Builder.tier(), ignoreTypes, graph-presence guard
- [FaultPolicy.java](/Users/mdproctor/claude/casehub/desiredstate/api/src/main/java/io/casehub/desiredstate/api/FaultPolicy.java) — addReviewNode static factory
- [ReviewSpecFactory.java](/Users/mdproctor/claude/casehub/desiredstate/api/src/main/java/io/casehub/desiredstate/api/ReviewSpecFactory.java) — functional interface gaining nodeType()
- [DesiredStateGraphRecorder.java:230-241](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java) — probeReviewNodeType() to be eliminated
- [#103 cleanup spec](/Users/mdproctor/claude/casehub/desiredstate/docs/specs/issue-103-ops-nodespec-migration/2026-08-23-addreviewnode-cleanup-design.md) — parent cleanup that filed this issue
- [Decision review R1-01](/Users/mdproctor/reviews/casehub-desiredstate/issue-112-decision-20260824-044627/responses/reviewer-1.md) — naming and annotation path analysis
- [Spec review R1-02,R1-03](/Users/mdproctor/reviews/casehub-desiredstate/issue-112-spec-20260824-134830/responses/reviewer-1.md) — probe regression and consistency assertion
- [decisions.md](/Users/mdproctor/claude/public/casehub-desiredstate/specs/issue-112-reviewnodepolicy-tier/decisions.md) — D1-D4 with review findings
