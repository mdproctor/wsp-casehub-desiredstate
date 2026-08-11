---
layout: post
title: "The approval handler that almost cleaned up after itself"
date: 2026-07-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [approval, work-adapter, workitem, design-review]
---

The last piece of the work-adapter module landed today: `WorkItemPendingApprovalHandler`, a stateless bridge between the desired-state reconciliation loop and casehub-work's WorkItem system. A provisioner returns `PendingApproval` with a plan reference, the handler creates a WorkItem, and each subsequent reconciliation cycle polls `findByCallerRef` until a human approves or rejects.

The interesting part wasn't the implementation — it's a straightforward delegator with no internal state. The interesting part was what the design review caught about cleanup timing.

I'd originally designed `check()` to be self-cleaning: when it detects a COMPLETED WorkItem, return `Approved` and immediately obsolete the WorkItem so it can't be reused. Clean, right? One-shot consumption.

The review pointed out the flaw. If the provisioner fails after receiving the approval — a transient network error, a timeout, anything — the next reconciliation cycle calls `check()` again. With self-cleaning, the WorkItem is already gone. `check()` returns `None`. The provisioner runs without approval context. Either it works (bypassing the approval gate entirely) or it returns `PendingApproval` again, forcing the human to re-approve something they already approved.

The fix: don't clean up on read. Let the COMPLETED WorkItem persist. `check()` returns `Approved` idempotently across cycles. If the provisioner succeeds, the node leaves the transition plan — no further `check()` calls, natural cleanup. If it fails, the same approval carries forward on retry. The only place that obsoletes a WorkItem is `recordPending` (clearing stale state before creating a fresh one) and `acknowledgeRejection` (clearing a rejected WorkItem so re-approval is possible).

That second cleanup — `acknowledgeRejection` calling `obsoleteByCallerRef` — addresses a known gotcha: a REJECTED WorkItem blocks its callerRef permanently. Without cleanup, once a human rejects, no new approval can ever be created for that node+action+tenancy combination. The rejection is acknowledged, the fault fires, and the state is cleared for a potential retry.

The callerRef format settled on `desiredstate-approval:<tenancyId>:<nodeId>:<action>` — hyphen-namespaced to distinguish from the human handler's `desiredstate:` prefix, tenancyId first to enable prefix-based tenancy queries. The rejection reason includes `WorkItemRef.outcome()` when non-null, since CANCELLED, EXPIRED, and ESCALATED carry meaningfully different semantics that fault policies can differentiate.

The branch also closed #82 — the casehub-ops DesiredNode constructor migration from `boolean` to `HumanGating`. Turned out it was already done. All five ops GoalCompilers already use the 4-arg constructor, including IoT's `HumanGating.ALL` for physical devices. The ops build passes clean. Filed as "already done, no code changes" and closed.

The stale-approval edge case is worth noting. Because COMPLETED WorkItems persist, a node that re-enters the transition plan after successful provisioning — goal change, drift recovery, different context — will receive an old `PlanApproval`. The handler can't detect staleness because `check()` carries no planReference. Provisioners that care about approval currency need to validate the `planReference` against their current context and return a fresh `PendingApproval` if it's stale. That's a provisioner responsibility, not a handler concern — but it's the kind of thing that needs documenting because the failure mode is silent.
