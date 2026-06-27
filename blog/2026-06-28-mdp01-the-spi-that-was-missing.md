---
layout: post
title: "Desired State — The SPI That Was Missing"
date: 2026-06-28
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [spi, human-nodes, work-integration, architecture]
series: issue-43-workitem-requires-human
---

# Desired State — The SPI That Was Missing

**Date:** 2026-06-28

## What I was trying to achieve: WorkItem creation for requiresHuman nodes

`SimpleTransitionExecutor` has a hard-coded branch for `requiresHuman=true` nodes — it returns `Skipped("requires human")` and moves on. No WorkItem, no notification, no tracking. Every reconciliation cycle, the planner re-plans the node, the executor skips it again, forever. The issue asked for the obvious fix: create a WorkItem so a human actually gets told to do something.

## What I believed going in: a direct dependency on casehub-work

The issue description said "add casehub-work-api as a compile dependency on the runtime module." That sounded right — casehub-work-api is the SPI module, WorkItemService lives in the runtime. Straightforward.

## The tier violation I almost shipped

When I checked the actual API surface, the design fell apart. `casehub-work-api` has policies and events — `ClaimSlaPolicy`, `ExclusionPolicy`, `WorkLifecycleEvent`. It does not have `WorkItemCreateRequest` or `WorkItemService`. Those live in `casehub-work` runtime, which drags in JPA, Hibernate, Flyway, and the full persistence stack.

I initially proposed a `work-adapter/` module depending on casehub-work runtime, claiming it "followed the engine-adapter pattern." It didn't. I verified: engine-adapter depends on `casehub-engine-api` where `CaseHubRuntime` is an *interface*, not a concrete class. Every one of its six dependencies is an API or SPI module. My proposed work-adapter would have depended on a concrete `@ApplicationScoped` service backed by JPA entities. Same pattern name, completely different dependency hygiene.

The fix was upstream. casehub-work needed a creation SPI extracted into its API module — a `WorkItemCreator` interface analogous to `CaseHubRuntime`, with `WorkItemCreateRequest`, `WorkItemPriority`, and `WorkItemRef` moved from runtime to API. All pure value types with no persistence dependencies.

## The idempotency bug that survived two design passes

The second thing I missed: `findByCallerRef`. The handler uses `callerRef` to prevent duplicate WorkItems — same node, same tenant, same callerRef, don't create again. But `findByCallerRef` checks existence, not status.

Walk through the re-provision scenario: node X appears, handler creates WorkItem 123, human completes it, node X is removed and deprovisioned, node X is re-added to the desired graph. The handler calls `findByCallerRef`, finds *completed* WorkItem 123, returns `Skipped("pending human action")`. The node is permanently stuck. A completed WorkItem is not pending human action — it's finished.

The fix: `findActiveByCallerRef` — filters by `WorkItemStatus.isActive()`, which returns true only for PENDING, ASSIGNED, IN_PROGRESS, SUSPENDED, and DELEGATED. Completed, cancelled, rejected, and expired WorkItems are invisible to the query. Re-provision creates a fresh WorkItem.

## What it is now

Three things landed. `HumanNodeHandler` is a new SPI in `casehub-desiredstate-api` — one method, `onProvision(DesiredNode, ProvisionContext) → StepOutcome`. `NoOpHumanNodeHandler` is the `@DefaultBean` fallback in the runtime, returning `Skipped` exactly as before. `SimpleTransitionExecutor` delegates to the handler instead of hard-coding the skip. When nothing else is on the classpath, behaviour is identical to before.

The `work-adapter/` module provides `WorkItemHumanNodeHandler` — `@ApplicationScoped`, displaces the no-op by CDI precedence. Depends on `casehub-work-api` only (the SPI module, post work#275 extraction). Creates a WorkItem via `WorkItemCreator.create()`, checks for active duplicates via `findActiveByCallerRef`, returns `Skipped("pending human action: WorkItem <uuid>")`.

The architecture mirrors engine-adapter exactly: SPI in api, default in runtime, real implementation in an adapter module with API-only dependencies. Four deployment combinations work correctly — STE with or without work-adapter, CTE with or without work-adapter — because CDI displacement is transitive.
