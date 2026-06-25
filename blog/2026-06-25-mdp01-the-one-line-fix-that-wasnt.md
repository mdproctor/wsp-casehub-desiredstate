---
layout: post
title: "Desired State — The One-Line Fix That Wasn't"
date: 2026-06-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [transition-planner, drift, self-healing]
series: issue-38-drifted-reprovision
---

# Desired State — The One-Line Fix That Wasn't

**Date:** 2026-06-25
**Type:** phase-update

---

## What I was trying to achieve: make DRIFTED nodes self-heal

Issue #38 looked simple. `TransitionPlanner.plan()` checks two statuses — ABSENT and UNKNOWN — when deciding what to provision. DRIFTED isn't in the condition. A node that exists but has diverged from its spec is invisible to the planner. No automatic re-provisioning. No self-healing.

The issue even included the fix: add `|| status == NodeStatus.DRIFTED` to line 52. One line.

## What I believed going in: it's a one-line patch with a test or two

I expected to verify the claim, add DRIFTED to the condition, write a test, and close the branch. The kind of issue you pick up between larger pieces of work.

## Two bugs, not one

The first thing I did was read the actual planner code rather than trust the issue description. The addition path was exactly as described — DRIFTED falls through the if-chain silently. But then I looked at the removal path.

Line 34 removes orphan nodes — nodes PRESENT in actual but not in desired. Only PRESENT. A DRIFTED orphan (a broken node that shouldn't exist at all) is also invisible. Same root cause, different code path, not mentioned in the issue.

The root cause isn't "DRIFTED missing from one condition." It's that the planner dispatches on `NodeStatus` via ad-hoc if-chains that silently ignore any status they don't recognise. Two of four statuses are handled per path. The other two fall through. Today that's DRIFTED. Tomorrow it could be a fifth status someone adds to the enum without updating the planner.

## The 2×4 matrix

Once I stopped thinking about it as a one-line fix and started thinking about what the planner actually does, the structure became obvious. Every node the planner considers falls into one of two contexts (in the desired graph, or an orphan) and has one of four statuses. The correct action for each cell:

| Status | In desired | Orphan |
|---|---|---|
| PRESENT | — | DEPROVISION |
| ABSENT | PROVISION | — |
| DRIFTED | PROVISION | DEPROVISION |
| UNKNOWN | PROVISION | — |

Eight cells. The planner handled six. Two were missing — both DRIFTED.

The implementation uses Java switch expressions without a `default` clause. On an enum, this fails to compile if any constant is missing. Adding a fifth `NodeStatus` value now forces the planner author to handle it in both switches. The bug can't recur — not because of a test, but because the compiler won't let it.

## The double-fault surprise

The more interesting consequence came from tracing the full reconciliation cycle. `ReconciliationLoop.detectDrift()` runs before the planner — it fires `NODE_DEGRADED` fault events for drifted nodes and lets fault policies respond (adding review nodes, escalation). After this fix, the planner also re-provisions the drifted node.

If that re-provisioning fails, `faultFeedback()` fires a `PROVISION_FAILED` fault event. So a single drifted node can now generate two fault events in one cycle: `NODE_DEGRADED` from drift detection and `PROVISION_FAILED` from the failed re-provision attempt. Both are correct — the node is still degraded AND the fix attempt failed. They're handled by different policies with different fault type filters. But it's a new interaction pattern that didn't exist before, and fault policy implementors need to know about it.

## Where this leaves it

The planner's status handling is now exhaustive — every cell in the matrix is explicit, and the compiler enforces it. DRIFTED nodes in desired are re-provisioned (self-healing via idempotent provisioners). DRIFTED orphans are deprovisioned. The casehub-ops deployment domain (#7) that filed this issue can now detect drift and automatically converge back to the desired spec.

The fix turned out to be the right moment to make the planner's status dispatch honest rather than just patching the gap. A one-line fix would have closed #38 but left the same class of bug waiting for the next status value.
