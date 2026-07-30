---
layout: post
title: "The Registry That Didn't Need to Exist"
date: 2026-07-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [fault-infrastructure, cdi, reconciliation]
---

# CaseHub Desired State — The Registry That Didn't Need to Exist

**Date:** 2026-07-30
**Type:** phase-update

---

## What I was trying to achieve: clean up the constructor telescope and ship the eviction listener

Three issues from the last session's design review follow-ups: #98 (ReconciliationLoop's seven public constructors collapsing into a builder), #97 (the FaultCountEvictionListener that gives `evict()` its first caller), and #99 (updating the CDI priority protocol to acknowledge functional fallbacks). Individually small. Together, they reshaped how the reconciliation loop is constructed, how listeners fire, and when fault counts get cleaned up.

## What I believed going in: the eviction listener needed a namespace registry

The issue description said it plainly: "each policy owns its namespace, so the listener needs a registry of (namespace, FaultCountStore) pairs." I started the brainstorm assuming I'd build that registry — some kind of coordinator that knows which ThresholdFaultPolicy owns which namespace string and which FaultCountStore instance backs it.

First-principles killed it in about thirty seconds. If a node leaves the desired graph, every fault count for that node is stale — regardless of which policy recorded it, regardless of namespace. The node can't fault anymore. Cross-namespace eviction is correct, and the registry is unnecessary complexity.

## The CbrProposalTracker bug hiding in the telescope

The constructor telescope existed because Java has no optional parameters. Seven public constructors, each adding one more argument, all delegating to a private master. Replacing them with a builder was straightforward — five mandatory params in the builder constructor, six optional params as fluent setters.

But tracing the CDI constructor revealed something worse than aesthetics. The CDI constructor passes `null` for `CbrProposalTracker`. The master constructor creates a fresh instance. CbrFaultPolicy, meanwhile, gets the CDI-managed `@ApplicationScoped` tracker injected. Two different instances — proposals recorded by the fault policy are invisible to the loop's outcome matching. CBR outcome CloudEvents silently never fire in production.

The fix was one parameter added to the CDI constructor. The builder made finding the bug possible — you don't scrutinise a constructor you're about to delete the same way you would one you're keeping.

## reconcileTypes() was lying to its listeners

The design review caught something I'd gotten wrong in the original spec. I claimed `reconcileTypes()` passed a filtered graph to listeners, making eviction unsafe. Claude's reviewer checked the code — it actually passes the full desired graph. The filtered graph goes to planning and execution only.

But the reviewer went further: even with the full graph, listener firing from `reconcileTypes()` is wrong for three independent reasons. It's a sub-operation, not a full cycle — "cycle completed" is semantically false. The actual state is partial (only the filtered types were read). And with N interval groups, listeners fire N times per resync with the same graph. We removed the `fireListener` call entirely and split it into `fireGlobalListeners()` and `firePerTenantListener()` for the full `reconcile()` path.

The design review also surfaced the tenant stop cleanup gap — when a tenant stops, its fault counts persist indefinitely. We added `onTenantStopped(String tenancyId)` as a default method on GlobalReconciliationListener, with a specific stop sequence: unsubscribe events first (prevents races), then fire onTenantStopped, then cancel futures, then clear CBR state.

## Where this leaves the eviction story

`FaultCountEvictionListener` is thirteen lines of real code. It implements both lifecycle hooks: `onReconciliationCycleCompleted` calls `evictAcrossNamespaces` with the current node set, `onTenantStopped` calls it with an empty set. CDI discovers it automatically. The runtime doesn't know or care that it exists.

The CDI priority protocol (#99) now distinguishes functional fallbacks from no-op mocks — `DefaultFaultCountStore` wraps `InMemoryFaultCountStore` as Tier 1a, not because it's a placeholder but because in-memory counting is a valid default that works without infrastructure. Tier 1b is the genuine no-op (like `NoOpPendingApprovalHandler`). The distinction matters: a functional fallback means the SPI works out of the box, silently degraded rather than silently broken.

The open question is whether the same cross-namespace insight applies elsewhere. ThresholdFaultPolicy's lazy eviction (removing a single node's count when a fault arrives for a missing node) and the listener's bulk eviction (cleaning up after each cycle) are complementary, but there may be other per-cycle cleanup operations that follow the same "if the node is gone, all associated state is stale" pattern. RAS correlation state, for one.
