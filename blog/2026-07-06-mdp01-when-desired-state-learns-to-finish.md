---
layout: post
title: "When desired state learns to finish"
date: 2026-07-06
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub DesiredState]
tags: [lifecycle, goal-compiler, completion, pending-approval, planner]
series: issue-46-pending-approval-lifecycle-planner
---

# When desired state learns to finish

The desired-state runtime has always had a philosophical gap: it doesn't know when it's done. You declare a graph of nodes, the reconciliation loop provisions them, detects drift, re-provisions. Forever. That's the value proposition — continuous reconciliation. But "forever" means the system can't express "build this base, then defend it." It can't express phases. It can't express succession.

Issue #58 asked whether the `GoalCompiler` SPI could carry that intent. The answer turns out to be yes, but only after the design review caught something I hadn't thought through.

## The SPI evolution

`GoalCompiler<G>` used to return `DesiredStateGraph` — a single graph, reconciled indefinitely. Now it returns `CompilationResult`, a sealed type: either `SingleGraph` (the old behaviour) or `Lifecycle` (a list of `Phase` records, each with a graph and a `CompletionCondition`). The single-graph case is a lifecycle with one terminal phase. Every existing GoalCompiler wraps its return in `CompilationResult.single()` — mechanical migration, one line per implementation.

The interesting type is `CompletionCondition`. It's a functional interface: `boolean isComplete(DesiredStateGraph desired, ActualState actual)`. The standard implementation is `allPresent()` — all nodes provisioned and stable. But the condition is evaluated against the *original phase graph*, not the current desired graph. That distinction matters.

## The listener placement problem

The design review found the show-stopper. The reconciliation loop has an early-return optimisation: when the plan is empty (nothing to do), the cycle exits before execution, fault feedback, and event emission. A `ReconciliationListener` placed "at the end of the cycle" — the natural mental model — sits after that early return. `CompletionCondition.allPresent()` is satisfied precisely when all nodes are stable and the plan is empty. The exact condition that triggers the early return. So the listener never fires on the cycle where completion is achieved.

The fix: fire the listener after `readActual()`, before the early return. Unconditionally. Every cycle. The `LifecycleManager` evaluates the completion condition from the listener callback, and if the phase is complete, CAS-swaps the graph to the next phase.

## CAS, not locks

Phase transitions use a double-CAS pattern: first CAS the lifecycle state in a `ConcurrentHashMap` (advance the phase index), then CAS the graph ref via `compareAndSetDesired()`. If the second CAS fails — a concurrent `SituationRecompiler` replan changed the graph — roll back the first. The transition is skipped and re-evaluated next cycle. No locks, no coordination, no gap where nothing is being reconciled.

The `LifecycleManager` sits above the `ReconciliationLoop`. The loop doesn't know about phases, completion, or lifecycle. It reconciles whatever graph it's given. All orchestration is in the manager. This keeps the loop a single-concern reconciler.

## PendingApproval in the pipeline

#46 was the smallest of the three issues. The pipeline example's three-tier fault escalation (retry → AI review → human review) happens after provisioning fails. PendingApproval gates happen before — the provisioner returns `PendingApproval` when it wants human sign-off before doing expensive work.

The SPI already existed. `SimpleTransitionExecutor` already calls `PendingApprovalHandler.check()` before every provision. The pipeline change was a `boolean approvalRequired` field on `TransformerSpec` and `SinkSpec`, a builder overload, and a guard in `PipelineProvisioner.dispatchToBackend()`. Gold-tier nodes can now require human approval before committing transformed data. The gate is off by default, configurable per-stage in the blueprint.

## The expansion example

The new `examples/expansion/` module demonstrates lifecycle transitions with a build-then-defend scenario. `ExpansionGoalCompiler` uses a hand-coded HTN decomposition: "establish base" decomposes into PROBE → NEXUS → PYLON → CANNON (linear dependency chain). The build phase completes when all nodes are PRESENT. The defend phase takes over with PATROL + MONITOR → RESPONSE.

Carry-forward is handled by the compiler, not the runtime. The defend phase includes a nexus node — identical spec to the build phase — so the nexus continues to be reconciled. If the nexus goes down during defense, the loop detects it and re-provisions. Nodes not in the defend graph (probe, pylon, cannon) are intentionally unreconciled. The compiler has the domain knowledge about what needs continued maintenance; the runtime shouldn't guess.

Fault-triggered replanning works through the existing RAS pipeline: persistent faults → CloudEvents → Ganglia pattern detection → `ActiveSituation` → `ExpansionSituationRecompiler` calls the GoalCompiler again with revised goals (switch to FORTIFY posture) → `LifecycleManager.updateDesired()` installs the result.

The two "notice-and-react" systems — reconciliation loop and planner — complement rather than conflict. The loop detects what happened. The planner decides what to do about it. They occupy different layers and never compete.
