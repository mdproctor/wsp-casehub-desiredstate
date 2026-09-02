---
layout: post
title: "The Question That Killed the Example"
date: 2026-09-03
entry_type: note
subtype: diary
projects: [casehubio/casehub-desiredstate, casehubio/blocks]
tags: [desiredstate, blocks, summarisation, ras, yaml, design]
series: issue-74-topology-implementation
---

# The Question That Killed the Example

I started this session expecting to design a logistics example that would wire desiredstate's reconciliation loop through blocks' summarisation framework into RAS for phase-aware detection. The pipeline was clear in my head: raw CloudEvents from ReconciliationLoop, classified by summarisers into anomaly types, aggregated into hub operational phases, detected by Ganglia, triggering case-driven replanning. A full vertical slice through three repos.

Then Claude asked: "have we found a practical use of desiredstate here?"

The answer was no. The logistics scenario — events flowing in, getting classified, phase transitions detected, responses triggered — is an event processing and situation detection problem. Desiredstate's value proposition is managing the gap between a declared graph and reality: provisioning nodes, detecting drift, reconciling. There's no graph to reconcile in a logistics hub. You could model routes and capacity allocations as nodes, but the interesting part of the example is the summarisation pipeline, not node provisioning. Shoehorning desiredstate in would make the example about the wrong thing.

The genuine desiredstate use case is the one that already exists: ops deployment topologies. ReconciliationLoop already emits CloudEvents. The existing Ganglia detect raw faults. Adding summarisation between them gives RAS altitude — "east-region entered degraded phase" instead of "node X faulted three times." That's a qualitatively different detection signal, and it's a real improvement to something that already works.

So the design split into two tracks. The logistics example lives in blocks — pure summarisation-to-RAS, no desiredstate. The desiredstate payoff is an ops enhancement that consumes the same bridge.

## The composable runtime

The more interesting design question was how far YAML could go. I wanted to push it — see where the declarative surface works and where Java becomes the necessary escape hatch.

The answer is a two-tier model. Tier 1: a YAML pipeline declaration with built-in summariser types (`threshold-classify`, `phase-detect`, `count`, `field-extract`, `pass-through`). An operator can wire a full L1→L2→L3 pipeline without writing Java. Tier 2: custom `@SummariserTypeId` Java classes on the classpath that YAML references by type string. The registry discovers them at build time via Jandex — the same pattern desiredstate uses for `@NodeTypeId` and `NodeSpecRegistry`.

The expression language split is pragmatic: MVEL3 for boolean predicates in classification rules and phase transition conditions, JQ for document transformation in field extraction. Both are already on the platform via `casehub-platform-expression`. RAS's YAML situation system already does per-expression language selection — we're following established practice, not inventing.

The spec review caught things I'd missed. Tenant partitioning in the accumulator — without it, multi-tenant events get mixed in batches and the pipeline produces incorrect results. The `phase-detect` semantics needed pinning down: emit on transition only, first-match-wins for competing transitions, per-tenant state, in-memory only (RAS owns durable situation tracking). Build-time validation for cross-level consistency — if a `phase-detect` references `count(DELAY)` but the upstream `threshold-classify` never produces a "DELAY" classification, that's a build error, not a runtime surprise.

## What shipped

I implemented the foundation layer in blocks: extracted 10 pure-Java summarisation types into `casehub-blocks-summarisation-api`, added `tenancyId` to `LevelEvent` with pipeline propagation, built the CloudEvent bridge module (`CloudEventIngestionAdapter`, `CloudEventEmitter`, `PipelineTickScheduler`), and rewrote `EventAccumulator` with tenant partitioning. All existing tests green.

The YAML surface and logistics example are next. They're the validation — if the logistics example can be expressed primarily in YAML with built-in types, the composable runtime works. If it can't, the built-in types need extension. Either outcome is useful information.
