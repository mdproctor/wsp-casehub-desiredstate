---
layout: post
title: "Worker breaks free"
date: 2026-06-19
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub]
tags: [architecture, worker, governance, foundation]
---

I came into this session planning to design managed pipeline execution. Within an hour we'd uncovered something more fundamental: Worker doesn't belong in the engine.

The question started simply enough — how should pipeline processing stages use Quarkus Flow for durable execution? But the more we explored the design, the more the engine's ownership of Worker felt wrong. A Worker is a named unit of automated work with a function and an execution policy. That's as generic as a WorkItem. WorkItem lives in casehub-work at foundation tier. Worker was stuck in casehub-engine-api at orchestration tier, making it invisible to every foundation repo without a bridge module.

The extraction turned out to be cleaner than I expected. Worker carries several engine entanglements — `PlanElement` (case plan model), `AgentDescriptor` (eidos capability probe), `PlannedAction` on `WorkerResult` (action risk classification), and `WorkerFunction` is sealed with variants that pull in Serverless Workflow and LangChain4j. None of those belong on the foundation primitive. The foundation Worker is a record with name, capabilities, function, and execution policy. `WorkerFunction` becomes an extensible interface — `Sync` is the built-in variant; engine-flow adds `Flow`, engine-api adds `AgentExec`. The engine uses the foundation Worker via composition, not inheritance.

The second insight was about execution governance. Retry, timeout, and backoff are reimplemented everywhere — on Workers, on WorkItem SLA timers, on agent invocations, on connector delivery. Same mechanics, different values. We put the types (`ExecutionPolicy`, `RetryPolicy`, `BackoffStrategy`) in platform-api and a `PolicyEnforcer` runtime in a new `governance/` submodule. Workers are the first consumer; the pattern is there for everything else to converge onto.

The concrete deliverables: `casehub-worker` exists as a new foundation repo with api, runtime, and testing modules. `casehub-platform-governance` ships `PolicyEnforcer` with retry, timeout, and three backoff strategies. The engine migration is a 60+ file change tracked in engine#543 with a detailed guide — structural adaptations for PlanElement, AgentDescriptor, PlannedAction, plus the mechanical import sweep.

The pipeline brainstorming surfaced one distinction that matters: provisioning a stage (one-time setup) is not the same as running it (ongoing data processing). `ExecutionBackend` sits at the provisioning layer. A "DroolsWorkerFunction" doesn't make sense as a provisioning operation — Drools is runtime processing technology. The relationship between these concepts is unresolved, and trying to force-fit it during this session was producing muddled thinking. The right move was to stop, get the extraction landed across repos, then come back to pipeline with clean context.
