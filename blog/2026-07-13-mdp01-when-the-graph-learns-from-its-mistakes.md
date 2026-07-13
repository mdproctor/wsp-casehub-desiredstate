---
layout: post
title: "When the Graph Learns From Its Mistakes"
date: 2026-07-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [cbr, revise, outcome-feedback, cloudevents]
series: issue-76-cbr-revise-outcome-feedback
---

*Part of a series on [#76 — CBR Revise step](https://github.com/casehubio/casehub-desiredstate/issues/76). Previous: [When the Graph Remembers](2026-07-12-mdp01-when-the-graph-remembers.md).*

Yesterday's CBR integration gave desiredstate a memory — it can retrieve past configurations that worked in similar situations and adapt them to the current problem. But it was a one-way street. The system could remember, but it couldn't learn. Every recommendation carried the same weight forever, regardless of whether it actually worked when applied.

The standard CBR cycle has four phases: Retrieve, Reuse, Revise, Retain. We'd built the first two. Revise is the feedback loop — observe whether the applied configuration succeeded or failed, and feed that outcome back to the case store so future retrievals improve.

The fundamental challenge was correlation. CbrFaultPolicy produces graph mutations in cycle N, but those mutations aren't executed until cycle N+1. By then, the reconciliation loop has no memory of where they came from. It just sees desired state vs actual state and plans accordingly.

The solution is a mediator — `CbrProposalTracker`. When CbrFaultPolicy or CbrSituationRecompiler selects a configuration, they record a proposal: sourceId, which nodes were affected, and whether it was a fault or situation path. On the next reconciliation cycle, after execution, the tracker matches per-node outcomes against pending proposals and emits a `io.casehub.cbr.outcome` CloudEvent.

The design had a subtlety I didn't anticipate until the adversarial review caught it. Drift-triggered proposals (from `detectDrift()`) are recorded *before* plan and execute in the same cycle, so they're observable immediately. But fault-feedback proposals (from `faultFeedback()`) are recorded *after* execution, so they're deferred to the next cycle. The spec originally claimed a clean one-cycle delay for everything — the review forced me to distinguish the two paths explicitly.

The CloudEvent type is `io.casehub.cbr.outcome`, not `io.casehub.desiredstate.cbr.outcome`. This was a deliberate scope decision. I looked at the app repos — devtown, clinical, aml, life, soc — and every one of them has its own CBR feedback loop through casehub-neocortex, each building its own outcome writer from scratch. The event type is platform-level because neocortex will eventually provide a single consumer for all CBR outcomes regardless of which module produced them.

The SPI change was the most disruptive part: `FaultPolicy.onFault()` and `SituationRecompiler.recompile()` both gained a `tenancyId` parameter. Thirteen FaultPolicy implementations and two SituationRecompiler implementations across casehub-ops need updating — a one-liner each, but still a cross-repo breaking change. Pre-release means the cost is just coordination, not migration.

The neocortex side — `CbrCaseMemoryStore.recordOutcome()` with EMA confidence adjustment — is a separate issue. Desiredstate emits the event; neocortex will consume it. When it does, the loop closes: retrieve a past configuration, apply it, observe the outcome, adjust the confidence. Cases that consistently work get stronger. Cases that fail get weaker. The graph learns from its mistakes.
