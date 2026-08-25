---
layout: post
title: "Graph Rewriting for the Annotation Model"
date: 2026-08-25
entry_type: note
subtype: diary
projects: [casehubio/casehub-desiredstate]
tags: [annotations, graph-rewriting, fixed-point, pattern-matching]
series: issue-106-graph-rule
---

# Graph Rewriting for the Annotation Model

The annotation module from #102 handles static graph declarations — you declare nodes and dependencies, and the build extension assembles them into a `DesiredStateGraph`. But static declarations can't express implied structure. "Every transformer needs a downstream validator" is a rule about the graph's shape, not a declaration of a specific node. You need something that can look at the graph and grow it.

That's what `@GraphRule` is for. Two signatures: parameterized rules where the engine does the matching, and imperative rules where you get the whole graph and write Java. Both return `List<GraphMutation>` — the same sealed type that FaultPolicy already uses at runtime. The engine runs them in a fixed-point loop until nothing changes.

The parameterized path is the interesting part. Four parameter annotations — `@Match`, `@DirectDep`, `@Reaches`, `@NotExists` — let you declare a pattern over the graph's topology. `@Match` binds nodes by type. `@DirectDep` and `@Reaches` navigate edges (direct or transitive). `@NotExists` is the guard that prevents infinite loops: it checks that a pattern is absent, so the rule fires once to add the missing structure and then the guard prevents it from firing again. The annotations chain sequentially by default — each refers to the previous binding — with an optional `of` attribute for non-linear patterns.

The design review caught a genuinely important gap. I'd originally left mutation ordering unspecified — just apply them in whatever order they arrive. But consider: if one rule removes node B (which sits on path A→B→C) and another rule adds edge C→A, the order matters. Apply AddDependency first and you get a false-positive cycle A→B→C→A. Apply RemoveNode first and the path is broken, C→A is valid. The spec now prescribes a fixed ordering: AddNode → UpdateNode → RemoveDependency → RemoveNode → AddDependency. Plus cycle pre-validation on the composed mutation set before any individual application.

The other significant revision was standalone rule classes. I'd initially gone with convention-over-configuration — any public method returning `List<GraphMutation>` is automatically a rule. The review pointed out the accidental discovery hazard: a utility method like `rebalanceIfNeeded(DesiredStateGraph graph)` would silently become a rule just because its signature matches. Explicit `@GraphRule` per method is slightly more verbose but prevents that class of bug entirely. Consistent with how `@Node` marks methods on interfaces, too.

Implementation started with the foundation types — five annotations, the `Direction` enum, four IR descriptor records. Then `GraphMutation.targetNodeId()` got extracted from the runtime's `GraphDiff` (where it was a package-private static method) to the sealed interface in api/ as a default method. Both `GraphRuleEngine` and `FaultPolicyEngine` need it, and the sealed interface is the natural home. Two batches of engine and integration work remain.
