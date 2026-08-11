---
layout: post
title: "Desired State as Goal"
date: 2026-06-14
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [desired-state, goals, architecture, casehub-engine]
---

The point of the session was to build the generic desired-state runtime — the graph model, planner, reconciliation loop, fault policies. Foundation tier. Domain-agnostic. SPIs for everything. What I didn't expect was the design question that mattered most.

I started with the obvious architecture: `DesiredStateGraph` holds nodes and dependency edges, `TransitionPlanner` topologically sorts them into prune-then-grow phases, `ReconciliationLoop` runs continuously comparing desired against actual. Domains plug in via `GoalCompiler`, `ActualStateAdapter`, `NodeProvisioner`, `FaultPolicy`. Standard stuff — the research doc had most of this mapped out already.

The interesting decisions came at the boundaries. The graph needed to be immutable and versioned — each fault policy mutation produces a new graph, the reconciliation loop works against a snapshot. We explored Clojure's persistent data structures, Erwig's inductive graphs, Bifurcan's persistent DAG, algebraic graphs. At 10-200 nodes, none of them matter — `Map.copyOf()` with dual adjacency maps (forward deps, reverse deps) copies in microseconds. But the *pattern* from Clojure's loom — adjacency maps as the primitive, algorithms on top — gave us the right internal representation. The graph itself is an SPI, backed by the CDI priority ladder. The default is 200 lines of Java records. If someone needs structural sharing at scale, they bring their own.

The execution model was where it got genuinely interesting. The research doc assumed delegation to `casehub-engine-flow`, but `FlowWorkerExecutor` requires `CaseInstance` — the desired-state runtime doesn't have cases. The first instinct was to use `WorkflowApplication` directly, but that misses the point. The prune phase and grow phase are generated Serverless Workflows. And a reconciliation cycle — the thing that produces and executes a transition plan — is structurally isomorphic to a case. The plan is the case plan. The ordered steps are plan items. The provisioners are workers. Human nodes are WorkItems.

So we SPI'd the executor. `SimpleTransitionExecutor` (`@DefaultBean`) walks the plan sequentially — good for testing. `CaseTransitionExecutor` (engine-adapter module, classpath-activated) generates a case with two `Worker(Workflow)` phases, starts it via `CaseHubRuntime`. You get durability, tracing, visualization for free.

But the real discovery was what that isomorphism means at the platform level. Desired-state and casehub-engine aren't competing models — they're two types of the same thing. A case is an *achievement goal*: "reach this outcome." Desired-state is an *invariant goal*: "maintain this state." Both decompose goals into sub-goals. Both produce plans. Both orchestrate workers. They differ in lifecycle — one completes, the other holds.

And some goals are both. "Deploy this topology" is an achievement goal (get it set up) that transitions into an invariant goal (keep it running). The root case becomes a long-lived reactive container — holding continuous desired-state loops alongside on-demand sub-cases for upgrades, failures, audits. That's not traditional CMMN. It's an event-driven process manager.

"Goal" might be the missing platform concept — the thing that sits above both cases and desired-state and unifies them. `GoalCompiler<G>` already has the right shape. It takes a goal and produces a plan. The generic type `G` is the goal. The output is either a graph (invariant) or a case plan (achievement). The routing is the lifecycle.

The Nefarious Dungeons example is the validation. `GoblinProvisioner` builds rooms. Creatures arrive when their room dependencies are satisfied. Hero raids trigger fault policies that rebuild destroyed rooms. A dragon requires human approval — `requiresHuman = true` skips the automated provisioner and waits for a WorkItem. The 2D tile visualizer makes it visceral: you watch goblins dig, creatures arrive, heroes attack, and the reconciliation loop rebuild. It's the kind of example that makes the abstraction click.
