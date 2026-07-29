---
layout: post
title: "The Map That Forgot Who Was Counting"
date: 2026-07-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [spi, fault-policy, multi-tenancy]
series: issue-85-persisted-fault-counts
---

ThresholdFaultPolicy had a `ConcurrentHashMap<NodeId, Integer>`. Three bugs hiding in
plain sight: counts vanished on restart, leaked when nodes were removed, and — the one
that surprised me — ignored `tenancyId` entirely. Node "n1" in tenant A and tenant B
shared a single counter. The method signature had `tenancyId` as its first parameter.
Nobody was using it.

The fix wasn't to patch the map. The map was the wrong abstraction. Counting is a
separable concern, and the moment you want persistence or eviction, an inline
`ConcurrentHashMap` can't deliver either without dragging domain knowledge into the
policy. I extracted a `FaultCountStore` SPI — five methods, three key dimensions
(`namespace`, `tenancyId`, `nodeId`) — and an `InMemoryFaultCountStore` default that
preserves the original behaviour for callers who don't care about persistence.

The namespace dimension came out of the design review. I'd originally keyed by
`(tenancyId, nodeId)`, which works when every policy has its own store instance.
But in a CDI deployment where a persistent store is a singleton bean shared across
policies, two `ThresholdFaultPolicy` instances would collide on the same key space.
The reviewer traced this back to ARC42STORIES Quality Goal 4 — fault composability.
Adding `namespace` as the outermost key dimension costs nothing for the in-memory
default (each instance has its own map anyway) and makes shared persistent stores
safe by construction.

The lazy eviction design was more interesting than I expected. The original guard
order checked fault type *before* checking whether the node existed in the graph.
That meant if a node was removed and a non-matching fault type came in, the eviction
never fired — the fault type check returned early before reaching the null check.
Reordering so `node == null` runs before the fault type filter ensures eviction
happens for any fault on a removed node, regardless of type.

The single-listener constraint on `ReconciliationLoop` was a genuine surprise from
the review. I'd documented a `ReconciliationListener` integration pattern for both
bulk eviction and recovery reset, not realising that `LifecycleManager` already
occupies the listener slot for every tenant started through the standard path. The
fix was to recommend CloudEvent subscription for recovery reset instead —
`NodeRecoveredData` events already carry `tenancyId` and `nodeId`, so the information
is there without competing for the listener slot.

The SPI lands in `api/` alongside ThresholdFaultPolicy — both the interface and the
default implementation. That placement raised an eyebrow during review until I
spelled out why: ThresholdFaultPolicy's builder needs `InMemoryFaultCountStore` for
the default path, and moving the default to `runtime/` would create a dependency
cycle. Unlike `@DefaultBean` implementations that CDI discovers, this is manually
instantiated in the builder. The module boundary dictates the placement, not
convention.

Four casehub-ops policies wrap ThresholdFaultPolicy (Kubernetes, Deployment, IoT,
Infra). None needed changes — the builder defaults to an in-memory store with an
auto-derived namespace. Persistence is a future opt-in: inject a persistent store
via CDI, pass it to the builder with an explicit namespace. That's #86.
