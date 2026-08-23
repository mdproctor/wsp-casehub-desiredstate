---
layout: post
title: "Dead Parameters and Hidden Redundancy"
date: 2026-08-23
entry_type: note
subtype: diary
projects: [casehubio/casehub-desiredstate]
tags: [api-cleanup, fault-policy, annotations]
---

# Dead Parameters and Hidden Redundancy

Issue #103 was filed as an ops repo migration — migrate ~25 NodeSpec implementations to the new `nodeType()` method and update DesiredNode from 4-arg to 3-arg. When I looked at the ops codebase, every NodeSpec already had `nodeType()` and every DesiredNode was already 3-arg. The migration had been done as part of the #102 prerequisite refactoring and nobody had closed the issue.

What remained was the cleanup trail: `FaultPolicy.addReviewNode(NodeType, ReviewSpecFactory)`, deprecated and marked for removal. The method was already a no-op delegation — it accepted a `NodeType` argument and silently threw it away, forwarding to the 1-arg form. Seven callers in desiredstate were passing dead parameters. Four more in ops.

The removal itself was trivial — 11 lines deleted, 5 adjusted. But the decision review surfaced something I hadn't considered. The same redundancy pattern lives one level up in `ThresholdFaultPolicy.tier()`:

```java
.tier(4, FaultPolicy.addReviewNode(factory), PipelineNodeTypes.AI_REVIEW)
```

That trailing `NodeType` must match `AiReviewSpec.nodeType()`, but the compiler can't enforce it. Get it wrong and two things break silently: the policy fires on its own review nodes (infinite escalation), and the tier-prerequisite check looks for the wrong type (skipped tiers). The annotations module even has a `probeReviewNodeType()` that calls the factory with a null graph to discover a type that should be statically available — fragile in exactly the way you'd expect.

I filed #112 for the `tier()` fix: a `ReviewNodePolicy` sub-interface that carries the output `NodeType`, eliminating the redundancy at both levels. Filed casehub-ops#70 for their four remaining callers.
