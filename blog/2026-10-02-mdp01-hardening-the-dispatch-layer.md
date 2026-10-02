---
layout: post
title: "Hardening the Dispatch Layer"
date: 2026-10-02
entry_type: note
subtype: diary
projects: [casehubio/casehub-desiredstate]
tags: [testing, hardening, situation-recompiler, production-readiness]
---

# Hardening the Dispatch Layer

Started the session intending to build the dispatch layer for `SituationRecompilerDispatch` — the CDI observer that bridges RAS situation events to the desired-state reconciliation engine. Opened issue #145, created a branch, kicked off brainstorming. Then discovered the code already existed.

Issues #146 and #147 had delivered the full implementation months ago: `situationResolved()` default method on the SPI, `SituationRecompilerDispatch` with `@ObservesAsync`, cold-start recovery via `SituationSource`, and `applyResult()` wiring through `LifecycleManager` and `ReconciliationLoop`. The merge commit explicitly closed #146 and #147 but never #145. The issue was just orphaned.

That raised a question worth pursuing: if the dispatch layer shipped without anyone noticing whether it was tested, what else is in the same state?

## The audit

The `SituationRecompilerDispatchTest` had three tests — all covering a single static helper method (`toActiveSituation()`). The actual observer methods — `onSituationChange()` for both TRIGGERED and RESOLVED events, `onColdStart()` for cold-start recovery, `applyResult()` for the graph-swap-plus-reconciliation wiring — had zero test coverage. The code worked because it was structurally straightforward, but there was no proof it worked.

Worse: `SituationRecompilerDispatch` did a hard `@Inject` on `SituationSource` with no `@DefaultBean` fallback. Every other optional SPI in the runtime has a NoOp — `NoOpHumanNodeHandler`, `NoOpPendingApprovalHandler`, `NoOpConfigurationRetriever`, and so on. `SituationSource` was the only one missing. Any deployment without `casehub-ras` on the classpath would fail at startup with `UnsatisfiedResolutionException`.

I broadened the audit across the whole runtime. The results painted a consistent picture: the runtime is architecturally sound but under-proven in its wiring and glue code.

`NodeStepExecutor` — the per-node execution unit handling human gating, approval lifecycle, and all four step actions — had 2 of roughly 15 behavioural paths tested. Deprovision, suspend, and resume were entirely untested. `ReconciliationEventEmitter` was missing emitter methods for three event types that had constants defined (`NODE_SUSPENDED`, `NODE_RESUMED`, `PLAN_APPROVED`), and 7 of its 12 existing emitter methods had no tests. `ExemptionEvictionListener` was structurally identical to the well-tested `FaultCountEvictionListener` but had no test file at all.

The Spring surface was further behind. Eight fallback beans that CDI provides via `@DefaultBean` were missing entirely from the Spring auto-configuration — `MergedEventSource`, `ActualStateAdapterRouter`, `HumanNodeHandler`, `PendingApprovalHandler`, `LifecycleStepExecutor`, `NotificationSink`, `ConfigurationRetriever`, `ConfigurationAdapter`. Spring apps would fail to start unless consumers provided all eight. `LifecycleManager` and `SituationRecompilerDispatch` had no Spring equivalents at all.

## The fix and the plan

I filed five issues (#163–#167) to cover the gaps: dispatch hardening (#163), `NodeStepExecutor` coverage (#164), `ReconciliationEventEmitter` completeness (#165), listener and lifecycle glue (#166), and Spring parity (#167).

Then I closed #163 in the same session. Added `NoOpSituationSource` as a `@DefaultBean` — one class, one method, returns `List.of()`. Wrote 18 new tests: 5 for `SituationRecompilerEngine.situationResolved()` (the chain-of-responsibility path that had zero coverage), and 13 for `SituationRecompilerDispatch` covering every observer method. The dispatch tests use real `ReconciliationLoop` and `LifecycleManager` instances — not mocks — so they exercise the full path from event through engine through graph swap through reconciliation trigger.

## The pattern

"Structurally complete but unproven" is a category worth watching for. The code reads correctly. The architecture is right. The wiring follows established patterns. But there's no test that fires an event and checks whether the graph actually changed. As the runtime moves toward real deployments, every SPI integration point needs a test that proves the wiring works end-to-end — not just that the individual pieces compile and the helper methods map fields correctly.
