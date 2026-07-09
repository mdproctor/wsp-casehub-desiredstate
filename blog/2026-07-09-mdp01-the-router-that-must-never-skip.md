---
layout: post
title: "The Router That Must Never Skip"
date: 2026-07-09
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub DesiredState]
tags: [multi-domain, routing, CDI, event-composition]
---

The desired-state runtime started with an implicit assumption: one domain per deployment. A single `ActualStateAdapter`, a single `EventSource`, a single `NodeProvisioner`. That was fine for teaching examples and single-purpose apps. It stops being fine the moment two domains share a classpath — CDI finds multiple candidates for each SPI and refuses to start.

`NodeProvisionerRouter` solved the provisioner case weeks ago — route by `NodeType`, dispatch to the right provisioner, fail fast on overlaps. The question was whether the remaining SPIs needed the same treatment, or something else entirely.

We analysed all four candidates. Two needed work; two didn't.

`FaultPolicy` was already composable — `FaultPolicyEngine` takes all policy beans and evaluates them all per fault event, merging mutations with conflict detection. Multiple domains just register more policies. `GoalCompiler<G>` was never injected by the runtime at all — the generic type parameter means callers already know which compiler to use because they have typed goals. Both turned out to be non-problems. That scope reduction saved half the estimated work.

`ActualStateAdapter` got a router — same pattern as `NodeProvisionerRouter`. Add `handledTypes()` to the interface, build a routing table at construction, dispatch `readActual()` to the right adapter with a type-filtered subgraph, merge the results. `DesiredStateGraph.filterByTypes()` became a default method shared between the router and `ReconciliationLoop`'s interval-grouped scheduling — the same filtering logic that was duplicated as a private method.

`EventSource` got composition instead of routing. Events are signals, not dispatched operations. A `MergedEventSource` interface — separate from `EventSource` — merges all domain streams via `Multi.merge()` with per-stream error isolation. If one domain's event source fails, the others keep running. Periodic resync covers the gap.

The design review caught the interesting one. We'd originally proposed `CompositeEventSource implements EventSource` — a class that discovers itself via `Instance<EventSource>` and filters itself out with an `instanceof` check. The reviewer pointed out that this is structurally wrong: `MergedEventSource` should be a separate type, not an `EventSource` subtype. The consumer (`ReconciliationLoop`) depends on the composed abstraction, not on the raw SPI. Same separation as `ActualStateAdapterRouter` vs `ActualStateAdapter`. Clean CDI, no self-filtering hack.

The gotcha that earned a garden entry: the adapter router initially skipped adapters when the type-filtered subgraph was empty. Natural optimisation — why call an adapter with nothing to report? Because adapters also detect orphans. When the desired graph goes empty (full deprovision), no adapters get called, actual state returns empty, the planner sees no diff, and orphaned nodes silently persist forever. The fix: always call every adapter, even with an empty graph. Provisioners are write operations — nothing to write means skip. Adapters are read operations that discover state beyond the input set.

Thirty files changed, roughly two thousand lines, all tests green across every module. The runtime now handles multi-domain deployments for provisioners, actual state adapters, and event sources — the three CDI injection points that needed it.
