---
layout: post
title: "Building the Rule Engine"
date: 2026-08-26
entry_type: note
subtype: diary
projects: [casehubio/casehub-desiredstate]
tags: [annotations, graph-rewriting, fixed-point, pattern-matching, convergence]
series: issue-106-graph-rule
---

# Building the Rule Engine

*Continues from [Graph Rewriting for the Annotation Model](2026-08-25-mdp01-graph-rewriting-for-the-annotation-model.md).*

The spec prescribed a separate cycle pre-validation pass: build a tentative edge graph from the composed mutation set, run DFS, report any cycle before applying mutations. I'd budgeted ~50 lines for it. But `ImmutableDesiredStateGraph.withDependency()` already does this — it throws `CyclicDependencyException` with the full cycle path on every edge addition. The mutation ordering (RemoveNode before AddDependency) means by the time any new edge is applied, removed nodes and their cascaded edges are already gone. No false positives. So I wrapped the exception into `GraphRuleCycleException` with rule attribution and deleted the custom detection entirely. One source of truth for cycle detection, not two.

The more interesting discovery was about convergence. The spec assumed rules would use `@NotExists` guards to prevent re-firing — standard production-rule-system thinking. But the first parameterized test proved that assumption wrong. A rule matching all "transformer" nodes and adding a "validator" per match fires correctly on iteration one — then fires identically on iteration two, because the transformers still match and the rule doesn't check whether its output already exists. Without guards, the mutations list is never empty and the engine exhausts 100 iterations.

The fix is `filterNoOps`: after deduplication, check each mutation against the current graph state. `AddNode` where the node already exists with identical content? Skip it. `RemoveNode` where the node is already absent? Skip it. Same for edges. If every mutation in an iteration is a no-op, the graph has converged — rules are producing work that's already been done. This gives correct fixed-point semantics without requiring guards on every rule. Guards still matter for efficiency — they prevent the engine from computing bindings and producing mutations that will just be filtered — but they're no longer required for correctness.

The build-time integration turned out cleaner than the spec suggested. The spec described modifying each of the recorder's three GoalCompiler creation paths. Instead, I restructured `createGoalCompiler` to capture its result, then wrap the GoalCompiler at the end:

```java
if (!descriptor.graphRules().isEmpty()) {
    List<ResolvedGraphRule> resolvedRules = resolveRules(descriptor.graphRules());
    GoalCompiler inner = runtimeValue.getValue();
    runtimeValue = new RuntimeValue<>((goals, factory) ->
        applyGraphRulesToResult(inner.compile(goals, factory), resolvedRules));
}
```

One wrapping point instead of three modified paths. The wrapper handles `CompilationResult.Lifecycle` too — each phase graph gets rules applied independently, because structural invariants must hold per phase, not just across the lifecycle.

Standalone rule classes (`@GraphRule(graph = "pipeline:*")` on a separate class, rules merged into matching graphs by namespace) didn't make it into this branch. Interface rules are the primary model — rules scoped to the declaring `@DesiredState` interface. Standalone rules extend that model for cross-graph sharing. Filed as #115 for a follow-up.
