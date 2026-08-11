---
layout: post
title: "Wiring the Three Tiers"
date: 2026-06-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [otel, tracing, agent-api, workitem, fault-escalation]
series: issue-37-batch-s-items
---

*Part of a series on [#37 — batch S items](https://github.com/casehubio/casehub-desiredstate/issues/37). Previous: [Enforcing What Was Already True](2026-06-17-mdp01-enforcing-what-was-already-true.md).*

The three-tier fault escalation — auto-retry, LLM diagnosis, human WorkItem — has been the pipeline example's selling point since the spec. But until today, two of the three tiers were simulated. The retry logic was real. The AI review was a pre-set outcome in PipelineWorld. The human review was a `StepOutcome.Skipped("requires human")` that went nowhere.

I wanted all three connected to real platform SPIs. Not production-grade — this is a teaching example — but real enough that you can see the actual APIs at work.

## OTel first

The tracing came first because it's foundational — the other two tiers are invisible without it. ReconciliationLoop now emits phase-level spans: `reconcile → readActual, detectDrift, plan, execute, faultFeedback`. SimpleTransitionExecutor adds per-node `provision` and `deprovision` spans underneath `execute`.

The interesting decision was where `opentelemetry-api` lives. It went into `runtime/`, not `api/`. Tracing is an implementation concern — the `TransitionExecutor` SPI already propagates span context via Mutiny's `Uni`, so no SPI change was needed. When no OTel SDK is on the classpath (the examples, any lightweight deployment), the API's no-op tracer means zero overhead.

I got bitten by a static field trap: caching the `Tracer` in a `static final` captures the no-op instance at class-load time, before any SDK registers. Every span call silently does nothing, no error, no warning. The fix is to call `GlobalOpenTelemetry.getTracer()` at method invocation time — the API caches internally, so the cost is negligible.

## AgentProvider for AI review

The pipeline provisioner now takes an `AgentProvider` and invokes it when an AI_REVIEW node has no pre-set outcome. The prompt is built from the `AiReviewSpec` — target node ID and error detail go in, a diagnostic response comes back. The response is parsed for a RESOLVED/UNRESOLVED signal: if the text contains "RESOLVED" but not "UNRESOLVED", it's treated as resolved.

With `NoOpAgentProvider` (no LLM on classpath), the `invoke()` returns an empty `Multi` — which falls through to the existing PENDING behaviour. The twenty existing tests pass unchanged. Add `casehub-platform-agent-claude` and the LLM actually runs.

## humanTask bindings for WorkItem

`CaseTransitionExecutor` now partitions additions into human and automated nodes. Automated nodes go into the grow `Worker(Workflow)` as before. Each human node gets a `humanTask` binding via `HumanTaskTarget.inline()` — the engine's HITL infrastructure creates a WorkItem when the case fires.

This is the part the CLAUDE.md already described as the intended behaviour. The code just hadn't caught up. Human nodes are now `StepOutcome.Skipped("routed to WorkItem")` instead of the generic `Skipped("requires human")` from SimpleTransitionExecutor — a small distinction that matters for tracing: you can tell whether the node was skipped because no executor handles it, or because it was deliberately routed to a human.

The three tiers are wired. A pipeline fault now retries, consults an LLM, and if that fails, creates a WorkItem. Each step is visible in the trace.
