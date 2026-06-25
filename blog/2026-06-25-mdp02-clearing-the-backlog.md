---
layout: post
title: "Desired State — Clearing the Backlog"
date: 2026-06-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [api, multi-tenancy, worker-extraction]
---

# Desired State — Clearing the Backlog

**Date:** 2026-06-25

Five small issues had been accumulating in the desiredstate backlog — two XS, three S. I batched them onto one branch and worked through them in dependency order.

The first one was already fixed. #34 (deprovision failures tagged as PROVISION_FAILED) had been committed months ago but the issue was never closed. The code already had the correct logic — building a set of removal node IDs and checking membership to select DEPROVISION_FAILED. Closed it and moved on.

#44 was a cross-repo doc update: PLATFORM.md in casehub-parent didn't mention the ExecutionBackend SPI or TransitionExecutor in the desiredstate row. One edit, committed to parent.

The two API changes were more interesting. #26 added `PendingApproval` to the sealed result types — both `ProvisionResult` and `DeprovisionResult`. The issue only mentioned DeprovisionResult, but neither type had the variant. Symmetry was the obvious call. A provisioner returning PendingApproval signals "I have a plan, but it needs human sign-off before I execute it." SimpleTransitionExecutor maps it to `Skipped`; the engine-adapter returns `PENDING_APPROVAL` status so the orchestration tier can create a WorkItem for the approval gate.

#36 was the biggest change by file count — adding `tenancyId` to `ActualStateAdapter.readActual()` and `TransitionExecutor.execute()`. Fifteen files touched, but the change was mechanical. The ReconciliationLoop already held a per-tenant `tenancyId`; it just had no way to pass it through to the SPIs. SimpleTransitionExecutor was hardcoding `"default"` into every ProvisionContext and DeprovisionContext. Now the real tenant identity flows end to end.

#39 wired the pipeline example to CaseTransitionExecutor by adding engine-adapter as a compile dependency. CDI does the rest — CaseTransitionExecutor is `@ApplicationScoped`, SimpleTransitionExecutor is `@DefaultBean`, so the engine-backed executor wins when both are on the classpath.

After the batch, I checked on #40 — the Worker extraction epic. Seven cross-repo steps, and the last two (openclaw#37 and ops#8) turned out to already be done. Openclaw had already migrated its Worker imports to `io.casehub.worker.api`. Ops never used Worker types at all — the "Capability" references there are IoT device capabilities, not worker capabilities. Closed the epic.

The desiredstate API surface is tighter now. Multi-tenancy flows through the SPIs properly, destructive operations can require approval, and the Worker extraction is complete across the entire platform.
