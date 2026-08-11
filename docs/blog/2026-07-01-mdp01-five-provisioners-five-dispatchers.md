---
layout: post
title: "Five Provisioners, Five Dispatchers"
date: 2026-07-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [multi-provisioner, scheduling, reconciliation, nodetype, spi]
series: issue-18-19-type-aware-runtime
---

Every `DesiredNode` carries a `NodeType` — the API javadoc even calls it "classifies a node by its provisioning domain." The runtime ignores it completely. `SimpleTransitionExecutor` takes a single `NodeProvisioner`. `DesiredStateDispatch` takes a single `NodeProvisioner`. One provisioner, one domain, no routing.

So every domain builds its own. GoblinProvisioner switches on NodeType with if/else. PipelineProvisioner does the same, then delegates to `ExecutionBackend.handles()` for a second dispatch layer. DeploymentNodeProvisioner uses a sealed spec switch into five per-type handlers. InfraNodeProvisioner routes through a `Map<String, InfraBackend>`. IoTNodeProvisioner routes through a `Map<String, DeviceProvider>`.

Five provisioners, five internal dispatch mechanisms, zero runtime support. And casehub-ops has all four of its provisioners as separate `@ApplicationScoped` beans — they can't co-exist on the same classpath without CDI ambiguity.

The scheduling problem is the same gap wearing different clothes. `ReconciliationLoop` runs one fixed 5-minute resync for everything. IoT devices that change every few seconds get the same polling frequency as infrastructure that changes once a month. The runtime can't differentiate because it doesn't know what it's reconciling.

Both problems are `NodeType` blindness. The fix is one coherent change: make the runtime type-aware.

`NodeProvisioner` gets two new methods. `handledTypes()` is abstract — every implementation must declare what types it handles. The compile error is the migration. `resyncInterval()` is a default returning five minutes — IoT overrides to 30 seconds, infra to 30 minutes. One method, zero ceremony for provisioners that don't care about scheduling.

`NodeProvisionerRouter` is an interface in the api module — four methods for provision, deprovision, interval lookup, and type enumeration. `DefaultNodeProvisionerRouter` in runtime builds a routing table at startup, validates no two provisioners claim the same type, enforces a 1-second floor on intervals. The engine-adapter depends on api at compile scope but runtime at test scope only — putting the interface in api means both `SimpleTransitionExecutor` and `DesiredStateDispatch` can inject it without adding a compile dependency from engine-adapter to runtime.

The scheduling change is the interesting one. The single resync timer becomes interval-grouped timers — types grouped by their effective interval, one `ScheduledFuture` per group. A new `reconcileTypes(Set<NodeType>)` method filters the desired graph to target nodes and runs the reconcile pipeline on the filtered view. Event-driven reconciliation stays full-graph. The scheduler pool grows from one thread to match the number of interval groups, because a slow infra reconciliation on a single thread would block the IoT timer from firing.

The CAS race in `detectDrift()` and `faultFeedback()` — garden entry GE-20260616-3d2605 — became relevant here. Both methods use `desiredRef.compareAndSet(desired, mutated)` to commit graph mutations. With multiple timers potentially running concurrent reconciliation cycles, the single-shot CAS drops mutations more often. The fix is a merge-and-retry loop: accumulate mutations locally, then CAS in a loop that re-reads and re-applies until it succeeds. Mutations are graph-structural and type-scoped, so concurrent cycles with disjoint type sets never produce conflicting mutations.

Operators get per-NodeType override via Preferences — `desiredstate.resync.physical-device: PT10S` — without touching the SPI. `DurationPreference` is the first production `MultiValuePreference` in casehub-platform-api. The infrastructure existed but had only been exercised in tests until now.

`ReactiveNodeProvisioner` got deleted along the way. Zero implementations across the entire ecosystem. Since `handledTypes()` already breaks every `NodeProvisioner` implementation, this was the right time to clean it up. One migration, complete cleanup.
