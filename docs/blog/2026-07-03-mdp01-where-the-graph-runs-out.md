---
layout: post
title: "Where the graph runs out"
date: 2026-07-03
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub DesiredState]
tags: [spatial, graph-model, evaluation, constraint-model]
series: issue-57-spatial-vector-poc
---

# Where the graph runs out

The question behind #57 was whether the desired-state graph model — nodes, edges, dependency-ordered provisioning — can handle spatial problems. Not infrastructure-as-code spatial, but RTS spatial: a 10x10 terrain grid with fog of war, units with strength values, zones distributing force across a frontier. The kind of problem where the state is a distribution over an area, not a chain of dependent steps.

I expected the graph to struggle with ratio constraints — "60% north, 30% south, 10% east" doesn't look like a dependency DAG. The design review pushed back on that: the transition planner processes zone nodes before their children (dependency ordering guarantees it), and GoalCompiler recompilation replaces the graph atomically. The ratio gap might be a phantom.

So we built three scenarios to find out.

## What works

**Defense posture** — base at (0,0), scouts revealing terrain, zones distributing force across a perimeter. Recompile when fog lifts, reconcile against the new graph. The planner diffs old graph vs actual state, deprovisions orphans, provisions new nodes. It's verbose — every discovery means a full recompilation — but it works. The graph model handles topology without strain.

**Attack with waypoints** — a dependency chain of positions heading northeast, each waypoint depending on the previous. Terrain collapse invalidates the route: old waypoints become orphans, new path gets provisioned. Both recompilation and incremental `UpdateNode` mutations work for force rebalancing. The graph handles sequential advance naturally — that's what dependency ordering was designed for.

## Where it breaks

**Force distribution across a shifting frontier** — and specifically, what happens when things go wrong. The graph handles the static case (compile ratios, provision units, done). It handles structural changes (frontier expands, zone splits into two — teardown and rebuild). But when a unit is destroyed and the system needs to respond intelligently, the model hits three walls.

**The fault policy can't see what happened.** `FaultPolicy.onFault()` receives `(FaultEvent, DesiredStateGraph)` but not `ActualState`. When a zone reports as DRIFTED, the policy knows something is wrong but not what — it can't determine which child unit was destroyed. The information it needs to redistribute force is in the actual state, which it doesn't have access to.

**The fault policy and the planner fight.** Even if the policy could diagnose the problem, the `TransitionPlanner` independently detects the missing unit as ABSENT and schedules restoration with the original spec. The policy's redistribution mutations and the planner's restoration compete. There's no mechanism for the policy to say "don't re-provision that node."

**Nobody notices the pattern.** Three consecutive cycles of losing units at the same position: each handled independently. The fault policy has no memory, no aggregate view, no way to detect "this approach keeps failing." Five failures in the same zone are five unrelated events. The decision to abandon a failing approach and try from a different direction — the strategic pivot — has no home in any SPI. The GoalCompiler can produce the alternative graph. The reconciliation loop can transition between them. But the decision itself requires something that doesn't exist yet: evaluation of aggregate outcomes across a subgraph.

## The real finding

I went in expecting the graph to break on ratio constraints. It didn't — zone group nodes handle ratios through the provisioner pattern, and dependency ordering keeps them consistent. The graph handles spatial *topology* well.

What it can't handle is spatial *strategy*. The layer above topology — where the system evaluates whether an approach is working, detects correlated failures, and chooses between alternatives — has no mechanism at all. That's the evidence for a constraint/evaluation model, and it's stronger evidence than the ratio argument would have been. The gap isn't "the graph can't represent this state." It's "the graph can't reason about whether this state is achieving its goal."
