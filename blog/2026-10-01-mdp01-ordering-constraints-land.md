---
layout: post
title: "The Constraint That Isn't an Edge"
date: 2026-10-01
entry_type: note
subtype: diary
projects: [casehubio/casehub-desiredstate]
tags: [ordering, graph, planner, surfaces, plan-approval]
series: issue-130-plan-preview-approval-gate
---

# The Constraint That Isn't an Edge

A dependency edge says "this node needs that node." An ordering constraint says something subtler: "all nodes of this type should provision before all nodes of that type." No instance-level coupling, no compile-time resolution — just a type-level sequencing hint that the planner honours at BFS time.

The distinction matters. A medallion data pipeline has Bronze, Silver, and Gold tiers. Each Bronze node doesn't depend on a specific Silver node — they're independent units. But you want all Bronze ingestion running before Silver transformation starts. An explicit dependency edge per node pair creates a combinatorial mess. An ordering constraint says `before: bronze, after: silver` and the planner does the rest.

The implementation lives in `TransitionPlanner.topologicalSort()`. Ordering constraints become virtual in-degree entries — the planner inflates in-degree counts for "after" type nodes by the number of "before" type nodes, builds a reverse-edge map for BFS propagation, and lets the existing cycle detection catch contradictions. No separate ordering pass, no post-sort rewrite. The BFS that already handles real dependency edges now handles type-level constraints through the same mechanism.

I also added a fast-path: when a graph has zero edges and zero constraints, `topologicalSort()` is pure overhead. The planner now detects flat graphs and returns a single-layer plan directly. Most simple deployments — a handful of independent services with no ordering requirements — skip the BFS entirely.

The second feature on this branch is the plan approval gate, which injects between `plan()` and `execute()` in the reconciliation loop. The design uses skip-and-recheck semantics: when a plan requires approval, the loop stores it and returns immediately. Next cycle, it checks whether the stored plan is still valid (desired state might have changed), whether it's been approved, or whether it was rejected. Rejected plans emit a `PLAN_REJECTED` fault. Approved plans proceed to execution with the original `TransitionPlan` — no re-planning needed because the version check already verified the desired state hasn't drifted.

The surface integration was the part I found most satisfying. Ordering constraints needed to work identically across YAML, Java annotations, and the TypeScript DSL — three declaration models with fundamentally different compilation paths. YAML resolves type strings at compile time via `NodeType.of()`. Annotations resolve at Jandex scan time via `@NodeTypeId`, catching missing annotations at build time rather than runtime. TypeScript carries strings through the envelope JSON and resolves on the Java side.

Each surface has its own model type (`YamlOrderingConstraint`, `@OrderBefore` + `OrderingConstraintDescriptor`, `TsOrderingConstraint`) but they all collapse to the same `OrderingConstraint(NodeType before, NodeType after)` record in the API. The e2e tests prove the full path for each: declare in the surface → compile to a `DesiredStateGraph` → run the `TransitionPlanner` → verify the resulting plan has the correct layer ordering. These caught a pre-existing bug in `TsDslDiscovery` where `humanGating` was passed as null instead of the envelope value — a bug that had been invisible because no test previously exercised the discovery-to-planner path end to end.

One gap remains: `GraphSerializer` doesn't persist ordering constraints through JPA round-trips. The planner reads constraints from the current desired graph, not from the stored previous graph, so the omission has no behavioural impact today. But it's the kind of thing that bites when someone later adds a feature that does read the stored graph's constraints.
