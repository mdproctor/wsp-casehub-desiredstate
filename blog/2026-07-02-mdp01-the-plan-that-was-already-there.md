---
layout: post
title: "The Plan That Was Already There"
date: 2026-07-02
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [planning, quarkmind, architecture]
---

I sat down to explore #24 — the state-vector abstraction for QuarkMind. The issue described a new `DesiredStateVector` data type for continuous/spatial state that didn't fit the graph model. Reasonable enough on the surface. But the conversation went somewhere else entirely.

The first question was straightforward: what does desired state actually look like in QuarkMind? StarCraft has plans running in parallel, plans that are part of bigger plans, plans that adapt. Where does desired state fit alongside planning?

Two levels became clear. At the tactical level, a plan to hold a map region is self-adaptive — enemy pushes in, the plan adjusts. That's the plan being self-healing within its scope. At the strategic level, something needs to decide *which plans should exist* and what resources they should have. That's the reconciliation loop's natural role. The nodes in the graph aren't "have 20 stalkers" — they're "hold north expansion." The provisioner for that node IS the plan.

Then we turned a corner. Claude ran a coupling analysis on the runtime — ReconciliationLoop, TransitionPlanner, FaultPolicyEngine — mapping how deeply they're tied to the graph model. The answer: deeply. ReconciliationLoop and TransitionPlanner are hard-wired to DesiredStateGraph, NodeId, Dependency, COW mutations, CAS loops. FaultPolicyEngine is loosely coupled. EventSource and SituationSource are fully generic. The graph isn't a peripheral concern — it's the organising principle.

But the more interesting finding came from looking at the graph model alongside classical planning. A desired-state graph IS a classical plan. Nodes are states/actions, dependencies are causal links, topological provisioning order is plan execution. You could write any desired-state graph in PDDL and any classical plan as a desired-state graph. Structurally they're isomorphic.

The difference is temporal, not structural. Classical planning executes once — reach the goal, done. Desired state never finishes. The reconciliation loop is the thing that classical planning lacks: the continuous "keep watching, keep correcting" semantics.

Which led to the uncomfortable observation: "desired state" is a form of classical planning invented by engineers who like to come up with simple names and have simplified versions of classical AI techniques. Kubernetes didn't cite STRIPS. They reinvented it with simpler vocabulary — no search (just topological sort), no precondition reasoning (explicit dependency edges), no goal regression (goal IS the graph), fixed operator binding (one provisioner per type). Then added the reconciliation loop.

The conclusion wasn't to choose between planning and desired state. It was "both." Classical planning produces the goal decomposition via GoalCompiler. Desired state provides the execution backbone — continuous reconciliation, drift detection, fault-triggered replanning. The planner doesn't need its own execution engine. The runtime IS the execution engine.

But the concrete scenario pulled us back from the abstraction. Expanding in StarCraft: you build a base (a graph of steps), then the base needs maintaining (reconciliation), then it's achieved and you need a *different* desired state to defend it (lifecycle transition), and then defending an area involves vectors over space — force distributions, not chains of steps. Graph for building, lifecycle for transitioning, spatial presence for defending. They interleave.

So #24 isn't one issue. It's three POCs:

- **#58** — lifecycle transitions: can the runtime handle "desired state achieved, now replace it"? Smallest piece. `updateDesired()` exists as a primitive; the work is completion detection and successor declaration.
- **#56** — planner-backed GoalCompiler: can a classical planner sit behind the GoalCompiler SPI and use the runtime for execution? Highest architectural value.
- **#57** — spatial/vector stress test: does the graph model break for "defend this region" scenarios, or can creative node design handle it?

That ordering matters. #58 is a prerequisite in practice — both replanning and spatial handoff are lifecycle transitions. #56 tests the most valuable architectural claim. #57 is the most uncertain — the graph model might handle spatial scenarios fine with the right abstractions.

The original #24 asked for a new data structure. The answer turned out to be a paradigm question: desired state isn't competing with planning — it's planning's missing execution backbone, and the GoalCompiler SPI is already the right seam for it.
