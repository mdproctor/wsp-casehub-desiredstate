---
title: "The Infrastructure Before the Infrastructure"
date: 2026-10-26
author: mdp
entry_type: note
subtype: diary
series: issue-180-plugin-catalog-versioning
projects: [casehubio/casehub-desiredstate]
tags: [plugins, versioning, dynamic-loading, architecture]
---

# The Infrastructure Before the Infrastructure

The plugin catalog design calls for YAML plugins that load from the filesystem at runtime — no restart, no recompile. An operator drops a directory into `plugins/monitoring-endpoint/1.2.0/`, and the reconciliation loop picks it up, validates it, wires it into the routing tables, and starts reconciling nodes of that type. That's the promise.

But the current runtime wasn't built for that. Every routing table is `Map.copyOf()` — immutable snapshots frozen at startup. The policy engines are the same: `List.copyOf(policies)` in the constructor, final field, no way to add or remove a policy without reconstructing the engine. The `ReconciliationLoop` holds these as concrete class references, not interfaces. There's no way to swap them out.

So before you can build the thing that loads plugins dynamically, you have to build the infrastructure that makes dynamic loading possible. That's what this branch does — and nothing more. The catalog itself is blocked on platform work (a generic `PluginManifest` SPI and `FilesystemPluginRepository` that belong in the platform repo, not here). Rather than wait, I decided to build everything on the desiredstate side that doesn't need platform, so when those modules land, the integration is straightforward wiring.

## What changed

**Semver versioning and dependency declaration.** `PluginHeader.version` was an `int` — unused, carried forward from the original prototype. It's now a semver string. The parser normalises both formats: `version: 2` becomes `"2.0.0"`, `version: "1.3.1"` passes through, and a missing version defaults to `"1.0.0"`. A new `PluginRequires` record lets plugins declare what they need: a runtime version range (`>=1.0.0`), a list of step primitives (`rest-call`, `compare-state`), and a list of node types they depend on. The `requires` section is optional — existing plugins parse without changes.

**Policy interface extraction.** `FaultPolicyEngine` and `DriftPolicyEngine` were concrete classes used directly by `ReconciliationLoop`. I extracted `FaultPolicyEvaluator` and `DriftPolicyEvaluator` interfaces into the `api/` module, made the existing classes implement them, and migrated every reference in `ReconciliationLoop`, its `Builder`, `RuntimeBeans` (CDI), and `DesiredStateRuntimeAutoConfiguration` (Spring) to the interface types. The concrete classes still work identically — they're just no longer the only option.

**Dynamic routing and policy infrastructure.** Four new classes in `runtime-core/`: `DynamicNodeProvisionerRouter`, `DynamicActualStateAdapterRouter`, `DynamicFaultPolicyEngine`, and `DynamicDriftPolicyEngine`. The routers use `ConcurrentHashMap` with `register()`/`unregister()` methods. The policy engines use `CopyOnWriteArrayList` — correct for the access pattern (every reconciliation cycle reads the policy list; writes happen only when plugins are added or removed).

One thing the code review caught: the original `register()` used `putIfAbsent` in a loop over the provisioner's handled types. If a provisioner handles types `[A, B]` and `B` collides with an existing registration, `A` is already in the map when the exception fires — leaving an inconsistent routing table. Claude flagged this during the branch audit. We switched to check-all-then-register: validate every type first, then insert them all. The fix is six lines but the bug would have been painful to diagnose in production.

**`ReconciliationStateStore.allTenancyIds()`.** The deactivation guard in the dynamic lifecycle needs to know which tenants have active nodes of a given type. The existing SPI had `store()`, `load()`, and `remove()` — all keyed by a single tenancy ID. The new `allTenancyIds()` default method lets the guard iterate all tenants. The in-memory implementation returns `Set.copyOf(snapshots.keySet())`; the JPA implementations use `SELECT DISTINCT tenancy_id`.

**`ExtensionRegistry`.** A query surface that answers "what types are available?" across both static extensions (GoalCompilers from CDI/Jandex) and dynamic plugins (from the catalog, when it exists). Thread-safe via `ConcurrentHashMap.newKeySet()` for static types and a `ConcurrentHashMap` for dynamic manifests. Deliberately not atomic across both collections — a query can see a slightly stale view, which is fine for an advisory surface.

## What's left

The integration layer — `DesiredStatePluginManifest`, `DesiredStatePluginActivator`, filesystem watching, the reload endpoint — is all blocked on the platform catalog SPI. The design is done, the plan exists, and the foundation is in place. When platform ships `plugin-catalog-api` and `plugin-catalog-core`, the desiredstate side is wiring and activation logic on top of infrastructure that's already tested and reviewed.

The interesting question is whether the check-then-register pattern in `DynamicNodeProvisionerRouter` needs to become `synchronized` when the `DesiredStatePluginActivator` starts calling it from filesystem watch callbacks. Right now, concurrent registration from two threads could both pass the collision check. The design spec describes activation as a sequential cycle — but that assumption needs to hold all the way through, or the routing table needs a lock.
