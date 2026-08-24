---
layout: post
title: "The Type That Was Always Missing"
date: 2026-08-24
entry_type: note
subtype: diary
projects: [casehubio/casehub-desiredstate]
tags: [api-design, fault-policy, type-safety, annotations]
series: issue-112-reviewnodepolicy-tier
---

# The Type That Was Always Missing

Yesterday's `addReviewNode` cleanup (#103) removed a dead parameter. Today's follow-up removed the structural reason the parameter existed in the first place.

`ThresholdFaultPolicy.tier(int threshold, FaultPolicy action, NodeType nodeType)` took three arguments because `FaultPolicy` is opaque — a lambda with no way to ask "what NodeType will you produce?" The caller had to declare the answer separately, and the compiler couldn't verify they matched. Get it wrong and two things fail silently: the policy fires on its own review nodes (infinite escalation), and the tier-prerequisite check looks for the wrong type (skipped tiers).

The fix is a sub-interface: `TypedFaultPolicy extends FaultPolicy` with `outputNodeType()`. `addReviewNode` returns it. `tier()` accepts it. The separate `NodeType` parameter disappears.

The decision review caught something I'd missed — I'd named it `ReviewNodePolicy`. Claude pointed out that the contract has nothing to do with review nodes specifically. The tier's NodeType is used for `ignoreTypes` (preventing self-triggering) and predecessor gating (escalation chain). Neither is review-specific. `TypedFaultPolicy` names the structural contract: a fault policy that declares its output type. The review also surfaced the probe-vs-actual consistency gap — if `ReviewSpecFactory.nodeType()` returns a different type than `create().nodeType()`, the mismatch relocates rather than being eliminated. We added a runtime assertion inside `onFault()` that catches this with a clear diagnostic.

The annotations module got the biggest simplification. `DesiredStateGraphRecorder` had a `probeReviewNodeType()` method that called the review factory with a synthetic `FaultEvent` and null graph to discover the NodeType. If the review method touched the graph — NPE, with a silent fallback to method-name-derived NodeType. That fallback was the exact mismatch problem we were trying to fix. It's gone now. `addReviewNode(factory)` returns `TypedFaultPolicy` and the type is inside.

Filed #113 for the next step: a `@Tier(nodeType="ai-review")` annotation attribute that would give the deployment processor build-time validation, eliminating the runtime probe entirely for the annotation path.
