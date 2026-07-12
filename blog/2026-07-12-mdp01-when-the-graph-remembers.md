---
layout: post
title: "When the Graph Remembers"
date: 2026-07-12
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [cbr, fault-policy, situation-recompiler, spi-evolution]
series: issue-23-cbr-integration
---

I've been thinking about what happens when desired-state reconciliation hits a wall. A node keeps failing to provision. The fault policy fires, proposes a retry, the retry fails, the policy fires again. Eventually the escalation chain runs out — AI review, human review, the full three-tier ladder — and the system has no more ideas. It just keeps cycling.

The missing piece was obvious once I looked at it through the CBR lens: the system never looks at what worked before. It has access to the full reconciliation history — casehub-ledger records every outcome, CaseMemoryStore holds the rich case representations — but the desired-state runtime never asks "has this happened before, and what fixed it?"

That's what #23 adds. Two integration points, both plugging into existing reconciliation machinery.

## The Tactical Tier

`CbrFaultPolicy` sits alongside existing fault policies in the `FaultPolicyEngine`. When a fault fires, it retrieves past configurations where similar faults were resolved, adapts them to the current graph topology, and proposes mutations — just like any other policy. The `FaultPolicyEngine` doesn't even know CBR is involved. It calls `onFault()`, gets back a `List<GraphMutation>`, and merges them with mutations from other policies.

The retrieval itself is an SPI — `ConfigurationRetriever`. Domain modules implement it by wiring `CaseRetriever` from casehub-neocortex under the hood. The desired-state runtime never touches embeddings or similarity search directly. It asks "give me configurations that look like this situation" and gets back ranked candidates with confidence scores.

Adaptation is a second SPI — `ConfigurationAdapter`. The retrieved configuration is from a *similar* situation, not an identical one. The adapter adjusts node specs, rewires dependencies, adds or removes nodes to fit the current graph. Both SPIs have `@DefaultBean` no-op implementations — when no domain provides retrieval, CBR is inert.

## The Strategic Tier

`CbrSituationRecompiler` operates at the whole-graph level, triggered by RAS when aggregate patterns emerge — three consecutive failures in a zone, persistent drift that won't resolve. Here the CBR chain returns a complete replacement graph wrapped in `CompilationResult.single()`, not per-node mutations.

This is where the design review caught the biggest problem. I'd originally planned `CbrSituationRecompiler` as a `@DefaultBean` that activates when no domain-specific `SituationRecompiler` exists. Claude's adversarial reviewer pointed out that this is mutually exclusive — any domain that provides its own recompiler silently kills CBR at the strategic tier, even if the domain recompiler only handles one specific situation type.

The fix was `SituationRecompilerEngine` — a chain-of-responsibility aggregator. Multiple recompilers register with `priority()`. Domain recompilers run first (default priority 0), and `CbrSituationRecompiler` runs last (`Integer.MAX_VALUE`) as a fallback for situations no domain recompiler covers. This mirrors `FaultPolicyEngine` architecturally but with a different aggregation pattern: fault policies merge mutations, recompiler engine takes first-non-empty-wins.

## The SPI Evolution

Adding CBR forced a breaking change to `SituationRecompiler`. The CBR chain needs both desired and actual state to find similar past configurations — a fault context without the actual state is a query without half its features. I added `ActualState` as a direct parameter rather than having `CbrSituationRecompiler` inject `ActualStateAdapterRouter` privately. Any intelligent recompiler that reasons about runtime state benefits from the same access. Pre-release platform, no external consumers — the breakage is mechanical and every implementation updated in the same commit.

Parameter reordering too: `(current, actual, situation, factory)` groups state inputs together, trigger next, utility last. Since it was already a breaking change, might as well make the signature deliberate.

## Graph Diffing

`CbrFaultPolicy` receives an adapted graph fragment and needs to produce `List<GraphMutation>`. The `GraphDiff` utility handles this — it diffs the adapted graph against the current graph and emits `AddNode`, `UpdateNode`, `RemoveNode`, `AddDependency`, `RemoveDependency` as needed.

The scoping rule is the interesting part. The adapted graph only covers a subset of the current graph's nodes. Nodes of the same *type* as adapted nodes are in scope for removal — if the adapted graph replaces all nodes of type "validator" with a different set, validators not in the adapted graph get `RemoveNode`. Nodes of other types are untouched. Cross-boundary dependencies (one endpoint in scope, one out) are never removed by the diff — the adapted fragment has no authority over them.

## What's Deferred

The Revise step — the feedback loop where reconciliation outcomes update the case store so future retrievals improve. Without it, CBR recommendations are static. The quality depends on whatever's in the case corpus when a domain wires up the retriever. That's tracked as #76. It's the step that makes this genuinely learn instead of just look up.

Also deferred: promoting `DoublePreference` and `IntPreference` to platform-api (#77). They're defined locally in the desiredstate api module for now, duplicating what casehub-engine-api already has. Foundation tier can't depend on engine — the promotion is a coordinated cross-repo change.
