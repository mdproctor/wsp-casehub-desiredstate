---
layout: post
title: "Desired State — Enforcing What Was Already True"
date: 2026-06-17
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub Desired State]
tags: [pipeline, constraints, api-design]
series: issue-35-pipeline-polish-and-layer-enforcement
---

*Part of a series on [#2 — data pipeline example](https://github.com/casehubio/casehub-desiredstate/issues/2). Previous: [The Pipeline That Broke the Runtime](2026-06-16-mdp01-the-pipeline-that-broke-the-runtime.md).*

# Desired State — Enforcing What Was Already True

**Date:** 2026-06-17
**Type:** phase-update

---

## What I was trying to achieve: make the pipeline example honest about its own rules

The data pipeline example has a medallion architecture — Bronze, Silver, Gold — where data flows through tiers in order. The dependency graph already enforced this implicitly: the goal compiler wires ingestion→cleanser→validator→transformer, and the topological sort respects those edges. But nothing prevented someone from constructing a graph where a Gold node depended directly on Bronze, skipping Silver entirely. The ordering was emergent, not enforced.

Two cleanup items came along for the ride: `layerOf()` returning null for fault-generated nodes (AI_REVIEW, HUMAN_REVIEW), and spec wording that drifted from the implementation.

## The `Optional` question

The interesting decision was `layerOf()`. It returned `PipelineLayer` — including null for node types that don't belong to any medallion layer. AI_REVIEW and HUMAN_REVIEW are operational nodes injected by fault policies. They exist outside the Bronze/Silver/Gold architecture entirely.

Returning null is a latent NPE. The fix was `Optional<PipelineLayer>`, which makes the type system say what the domain already knows: not every node in the graph participates in the medallion hierarchy. The visualizer was already handling this with `if (layer != null)` — `ifPresent()` is the same logic without the risk.

## Where enforcement belongs

The `TransitionPlanner` is domain-agnostic. It knows about graphs, edges, and topological ordering. It does not know about Bronze, Silver, or Gold. Adding layer validation to the planner would mean the generic runtime carries domain-specific knowledge — exactly the boundary this project exists to maintain.

So `MedallionLayerConstraint` lives in the pipeline example. It validates two properties: no backward dependencies (Bronze depending on Gold) and no layer skipping (Gold depending directly on Bronze without Silver in between). Nodes without a layer — the fault-generated ones — are exempt. The constraint runs inside `PipelineGoalCompiler.compile()`, catching violations at goal compilation time before the planner ever sees the graph.

If this pattern proves useful across domains, a generic `GraphConstraint` SPI in the runtime would be the natural evolution. But that is a future issue, not today's.

The medallion architecture was already being enforced by the dependency graph the goal compiler produced. Now it is also enforced as a named, testable rule.
