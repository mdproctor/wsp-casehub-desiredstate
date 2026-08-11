---
layout: post
title: "When Deprovision Meets the Human Gate"
date: 2026-07-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [requiresHuman, HumanNodeHandler, deprovision, SPI]
series: issue-54-requireshuman-deprovision
---

The `requiresHuman` flag on a DesiredNode was always described as a routing signal — "consult the HumanNodeHandler for this node's lifecycle." But it only routed half the lifecycle. Provision checked it. Deprovision didn't. The comment in `executeDeprovision` said it plainly: `// no requiresHuman for deprovision`.

I hadn't thought much about this asymmetry until issue #54 made the failure concrete. A domain with physical devices — things a human must physically install — returns `DeprovisionResult.Failed` when the provisioner can't automate the removal. That failure triggers fault feedback. Every reconciliation cycle, the same node faults again. The workaround (returning `Success` from the provisioner for physical devices) is correct for IoT semantics but papers over a runtime gap.

The fix is the obvious one: make `requiresHuman` gate both directions. The handler decides what each action means. For IoT, `onDeprovision` creates a WorkItem — someone needs to physically remove the device. For a logical domain where deprovision is just "stop managing," the default `Skipped` is fine.

I went into the brainstorm thinking `onDeprovision` should be abstract — force every implementor to handle both cases at compile time. The design review pushed back. The argument was sharp: if `Skipped` is the right answer for `NoOpHumanNodeHandler` (which I proposed), then it's a reasonable default for the interface too. And making it abstract would kill the SAM type for no architectural benefit — two test lambdas would break, requiring a `MockHumanNodeHandler` that only existed to work around the breakage I caused. Self-inflicted complexity. I accepted the change.

The callerRef format needed attention too. The old format — `desiredstate:<tenancyId>:<nodeId>` — can't distinguish a provision WorkItem from a deprovision WorkItem for the same node. If a node is being deprovisioned while a provision WorkItem is still active, `findActiveByCallerRef` returns the wrong one. The fix appends the action as a suffix: `desiredstate:<tenancyId>:<nodeId>:<action>`. Clean break — no production data exists.

`CaseTransitionExecutor` had the same asymmetry. It already separated human additions from automated additions — human nodes get `humanTask` bindings instead of workflow steps. But all removals went through the prune workflow regardless of `requiresHuman`. The fix mirrors what it already does for additions: separate human removals into their own `humanTask` bindings. While there, we action-namespaced all the binding names — `human-provision-<nodeId>` and `human-deprovision-<nodeId>` — so the naming is consistent across callerRefs, WorkItem types, and CTE bindings.

The whole change is six production files and their tests. 462 tests pass across all twelve modules including all four examples. The interesting thing isn't the implementation — it's how a single boolean flag, described from day one as a "routing signal," quietly only routed one direction for months until a concrete failure made the gap visible.
