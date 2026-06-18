---
layout: post
title: "The Toolbox That Wasn't"
date: 2026-06-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [pipeline, execution-backend, worker-coordination, data-exchange, data-channel, architecture]
---

The issue said pluggable execution backends for pipeline stages. Camel for integration routes, Drools for business rules, Stream for lambda chains. The `NodeProvisioner` SPI already supports it — just make the per-type dispatch pluggable via a strategy pattern. Straightforward.

I started by asking whether this was pipeline-specific or something deeper. That question opened a door I hadn't expected.

## The bigger shape

What if each pipeline node's execution was a Serverless Workflow? quarkus-flow already chains Workers. Camel, Drools, and Stream would just be `call:` targets. The pipeline becomes a workflow of Workers.

But Workers share state via the blackboard — a shared key-value map on the case. Pipeline stages pass *data*. Records, batches, files. The blackboard is too coarse, wrong direction, wrong lifetime. What a pipeline needs is a data channel alongside the control channel. The control path says "run this stage now." The data path says "here's your input, put your output here."

We explored two patterns for this. **DataExchange** — a Camel-inspired Exchange object that flows stage to stage, body can be anything (stream, file handle, URI, POJO). **DataChannel** — a continuous streaming pipe with backpressure, producer writes records, consumer reads as a reactive stream. These aren't pipeline concepts. They're Worker concepts. Any connected Worker chain — case workflows, monitoring pipelines, financial reconciliation — could use either pattern alongside the existing Blackboard.

That's three data coordination patterns for Workers: Blackboard (shared mutable state), DataExchange (discrete payload handoff), DataChannel (continuous streaming). Each solves a different problem. Each is optional — Workers that don't need them never see them.

## Where it belongs

The brainstorming surfaced a design I liked. It also surfaced where that design doesn't belong. Exchange, DataChannel, and ExchangeProcessor are engine-tier Worker coordination primitives. casehub-desiredstate is a foundation-tier desired-state runtime. Building engine concepts here and planning to extract them later is pragmatic but wrong — if you know where something belongs, put it there.

I filed two engine issues: [#528](https://github.com/casehubio/engine/issues/528) for making `WorkerFunction` pluggable (it currently hard-codes `Workflow` from the Serverless Workflow API into a sealed interface — every consumer pays for that dependency whether they use workflows or not), and [#532](https://github.com/casehubio/engine/issues/532) for the DataExchange/DataChannel Worker coordination patterns.

## What we actually built

The right scope for #28 turned out to be exactly what the issue said all along: a strategy pattern on the provisioner. An `ExecutionBackend` SPI for processing stages (INGESTION through SINK), while the provisioner handles metadata registration (DATA_SOURCE, SCHEMA) and fault handling (AI_REVIEW, HUMAN_REVIEW) directly. `DefaultExecutionBackend` extracts the existing processing stage logic — same behaviour, pluggable dispatch. Fail-fast `AmbiguousBackendException` when multiple backends match.

The mechanical refactoring is the deliverable. The brainstorming is the value. We now have a clear picture of how the pipeline example evolves into something genuinely useful — through engine-tier Worker coordination primitives, not through building a parallel framework in the wrong repo.
