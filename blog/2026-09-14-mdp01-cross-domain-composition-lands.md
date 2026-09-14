---
title: "Cross-Domain Composition Lands"
date: 2026-09-14
author: mdp
entry_type: note
subtype: diary
projects:
  - casehub-desiredstate
tags: [composition, cross-domain, orchestration, reconciliation]
issue: 140
status: draft
---

# Cross-Domain Composition Lands

The desired-state runtime has always been multi-domain-ready in one direction: provisioners, adapters, event sources, and fault policies all dispatch by NodeType. Four domains on one classpath? Fine — the routers handle it. But GoalCompiler is parameterised by goal type, and CDI can't discover `Instance<GoalCompiler<?>>` across different type parameters. The composition gap lived at the top of the stack.

Issue #140 closes that gap. Two sessions — brainstorming and design in the first, implementation across both.

## The Push Model

The design review surfaced a clean solution that I almost didn't go with. Instead of the composition engine calling each GoalCompiler (which hits the type erasure wall), domains compile themselves and push the result. Each domain's startup observer knows its own `GoalCompiler<InfraGoals>` or `GoalCompiler<DeploymentGoals>` — type safety stays intact. The composition engine receives `CompilationResult`, which has no type parameter.

This also means the engine never needs to know how to load goals. That's domain business. The engine's job is merging graphs and creating cross-domain edges.

## Provides/Requires

Domain registration carries `provides` (which NodeTypes this domain contributes) and `requires` (which NodeTypes from other domains must exist first). The engine validates at startup — duplicate provides, unsatisfied requires, circular dependencies via Kahn's algorithm, node ID collisions across domains.

Cross-domain edges connect the requiring domain's root nodes to the providing domain's typed nodes. TransitionPlanner already handles dependency ordering, so the composed graph works without any planner changes. The integration tests verify this directly: register infra providing `k8s-namespace`, register deployment requiring `k8s-namespace`, compose, plan — the planner orders namespace provisioning before agent provisioning. Three-domain chains (A → B → C) work the same way.

## Per-Domain Lifecycle

Each domain can return `CompilationResult.Lifecycle` with multiple phases. The engine tracks phase state per tenant per domain and evaluates CompletionCondition after each reconciliation cycle. When a domain's current phase completes, the engine advances that domain's phase, recomposes the merged graph with the new phase's nodes, and pushes the update to LifecycleManager.

This is where the concurrency model matters. Phase advancement (from the reconciliation scheduler thread) and replan (from the workflow executor thread) both read all domain states and recompose. A `recomposeLock` serializes both paths. Claude's code review caught that the initial handleReplan implementation read domain state outside the lock — a narrow race but a real one. The full iteration now runs inside `synchronized(recomposeLock)`.

## Hierarchical Mode

The second deployment mode builds a meta-graph with one node per domain, dependency edges from provides/requires matching, and a DomainNodeProvisioner that checks readiness via CompletionCondition. The types are in place — DomainNodeSpec, DomainNodeProvisioner, DomainActualStateAdapter, buildMetaGraph(). Inner loop creation via ReconciliationLoop.Builder is skeleton-only; the flattened mode is the first consumer path (casehub-ops#23), so hierarchical gets wired when someone needs it.

Making `ReconciliationLoop.shutdown()` public was a quiet but necessary change — inner loops created via Builder need explicit lifecycle management, and the method was package-private.

## What This Opens Up

casehub-ops can replace its hand-coded `ApplicationGoalCompiler` with per-domain registrars. Each domain module registers its graph with provides/requires metadata; the composition engine handles ordering and lifecycle. The four-domain ordering (infra → deployment → compliance → IoT) becomes declarative rather than procedural.

The hierarchical mode is the longer play. Isolated fault domains, independent reconciliation frequencies, domain-level readiness gates — the foundation is there for when multi-process deployment needs it.
