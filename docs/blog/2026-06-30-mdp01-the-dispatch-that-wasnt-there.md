---
layout: post
title: "The Dispatch That Wasn't There"
date: 2026-06-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [engine-adapter, pendingapproval, dispatch, workflow]
series: issue-47-cte-pending-approval
---

CTE has been broken since day one. Not in an obvious way — tests pass, the case starts, the result comes back. But inside the case, nothing happens. The workflow steps call `desiredstate:dispatch` and there's no handler for that call name. The entire provisioning pipeline is dead code wrapped in an optimistic result.

I'd known PendingApproval was silently swallowed by CTE — that was the filed issue, #47. What I hadn't traced was the root: `CasehubCallableTaskBuilder` in engine-flow accepts ALL `CallFunction.class` instances and throws `UnsupportedOperationException` on any name it doesn't recognise. There's exactly one name it recognises: `casehub:dispatch`. Every other name is a hard failure. No fallback, no second builder, no extension point.

So PendingApproval wasn't being swallowed. Nothing was being swallowed. The case was starting, the workflow was failing silently, and `buildOptimisticResult()` was painting over the rubble.

The first instinct was to discard workflows entirely. Replace them with a plain `WorkerFunction` — a Java loop that iterates over nodes and calls the provisioner directly. No `CallableTaskBuilder`, no dispatch mechanism, no cross-repo dependency. Simpler, correct, and doesn't fight the engine.

That instinct was wrong.

The workflow abstraction is right for this problem. A transition plan IS a structured workflow — ordered steps with dependency constraints, each independently observable, retryable, and potentially parallel. Per-step visibility in the case plan, Serverless Workflow retry semantics, crash recovery, composability — these aren't theoretical benefits. They're the reason CTE exists instead of just using SimpleTransitionExecutor. Replacing the workflow with a loop would be simpler in the same way that removing the engine would be simpler: it avoids work by discarding capability.

The real fix was making the `call:` namespace extensible. `CallableDispatchRegistry` in engine-flow (engine#590) — a CDI registry that maps call names to dispatchers. `CasehubCallableTaskBuilder` delegates to it instead of hardcoding `casehub:dispatch`. Any module can register its own call namespace. The builder stays CDI-free for ServiceLoader compatibility; the registry is `@ApplicationScoped`; dispatchers self-register at `@PostConstruct`.

With the registry in place, the desiredstate implementation follows naturally. `DesiredStateDispatch` registers for `desiredstate:dispatch` and mirrors SimpleTransitionExecutor's approval lifecycle exactly — check before provisioning, record after PendingApproval, acknowledge on rejection. CTE pre-filters approval-gated nodes before building the case, avoiding empty cases for nodes that are just going to skip. Context passes through via an `executionId` embedded in the workflow step args — generated before `startCase()`, looked up by the dispatch handler, no race condition.

The distinction between a broken implementation and a wrong abstraction matters. I reached for "discard the workflow" because the dispatch mechanism was broken. But the dispatch mechanism was broken because it was V1, not because workflows are wrong for node provisioning. Fixing the dispatch mechanism — making it extensible — is a platform investment that pays forward. The next module that generates workflows with custom call steps won't hit the same wall.
