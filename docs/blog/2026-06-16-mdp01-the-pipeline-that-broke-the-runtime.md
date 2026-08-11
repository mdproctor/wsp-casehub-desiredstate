---
layout: post
title: "The Pipeline That Broke the Runtime"
date: 2026-06-16
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [data-pipeline, medallion-architecture, fault-escalation, aifusion, runtime-bugs]
---

The issue said infrastructure provisioning. Kubernetes namespaces, deployments,
services, ingress. Six node types, a dependency chain, a crash-loop fault policy.
Standard validation exercise — prove the SPIs are domain-agnostic by building
something different from the dungeon.

I asked myself whether I could simulate Kubernetes in a meaningful way. The
answer was no. A simulated K8s is either a `ConcurrentHashMap` pretending to
have API server semantics — which validates nothing — or a reimplementation of
K8s behaviour that becomes its own project. The dungeon works as a simulation
because the dungeon domain IS the simulation. There is no real dungeon to fail
to represent. A fake Kubernetes has a real Kubernetes to be compared against,
and every shortcut undermines the claim.

So the question became: what domain has the right graph shape — root, fan-out,
fan-in, linear chain — where the in-memory model IS the actual system?

## Data pipelines and the AiFusion angle

A data processing pipeline. Not because it's simpler than K8s, but because
it's strategically relevant. CaseHub is being positioned for AiFusion, and
data processing, cleansing, enrichment, and validation are core capabilities
in that domain. An example that uses real AI/data vocabulary makes the
platform's value proposition concrete.

The pipeline shape maps directly to the original K8s dependency chain.
`datasource → ingestion → cleanser → enricher → validator → transformer → sink`.
Two independent roots (datasource and schema — schema is a validation contract,
not a data source), three fan-in points (cleanser depends on both ingestion
and schema, validator depends on both enricher and schema). The dependency
graph is richer than the dungeon's flat room-creature hierarchy.

I added medallion architecture on top — Bronze, Silver, Gold. Industry-standard
data lakehouse pattern. Ingestion is Bronze (preserve fidelity). Cleansing,
enrichment, validation are Silver (enterprise view). Transformation and sink
are Gold (business-ready). The layers are an emergent property of the dependency
graph's topological ordering, not a planner-level enforcement. The planner
knows nothing about medallion layers. It just follows dependencies, and the
layers fall out.

## What the spec review found

The spec went through seven review iterations. The first three were
straightforward — fix the dependency graph arrows, add idempotency guards,
specify deterministic node IDs. The interesting findings started at iteration
four.

**The runtime had no mechanism to detect DRIFTED nodes.** `ReconciliationLoop.reconcile()`
only created fault events from `StepOutcome.Failed` — always `PROVISION_FAILED`.
DRIFTED nodes were invisible to the entire fault policy system. `FaultType.NODE_DEGRADED`
existed in the API enum but nothing in the runtime produced it. The dungeon never
triggered this because its only fault policy handles `NODE_DESTROYED` from external
events, not from the reconciliation loop. The pipeline's schema drift and quarantine
policies both depend on detecting DRIFTED nodes. Without the fix, two of the four
fault policies could never activate.

**The CAS race was worse.** When multiple nodes failed provisioning in the same
reconciliation cycle, only the first fault's mutations survived. The loop captured
`desired = desiredRef.get()` once, then called `compareAndSet(desired, mutated)`
inside the per-failure iteration. The first CAS succeeded. The second CAS used
the original `desired` as the expected value, but `desiredRef` had already changed.
Silent failure. No log, no warning, no error. Mutations permanently lost. The dungeon
never triggered this either — its fault policy handles a different event type entirely.
The fault feedback path in `reconcile()` was effectively dead code.

**`ImmutableDesiredStateGraph.withoutNode()` destroys all dependency edges.** The
original spec had `SchemaDriftFaultPolicy` returning `RemoveNode` mutations for
downstream stages, intending to re-add them after human approval. But `withoutNode()`
strips both forward and reverse edges. `AddNode` for the same ID later does not
restore them — the node re-enters as an isolated root. The graph topology is
permanently lost. The only way to restore it is to re-invoke the `GoalCompiler`
with the original blueprint, which fault policies don't have access to.

The fix was to not remove downstream nodes at all. Leave them in the desired graph
as DRIFTED. The planner ignores DRIFTED nodes (they're not ABSENT). Once the
human approves the schema change, the stages return to RUNNING, the adapter reports
them as PRESENT, and reconciliation resolves naturally.

## Three-tier escalation

The three-tier escalation design — auto-retry → AI review → human WorkItem — went
through two redesigns before it worked.

The first version had four separate fault policies sharing a retry counter. Claude
traced the lifecycle and found the counter couldn't be shared without breaking CDI
composition. Then it traced what `RetryFaultPolicy`'s `AddNode` mutation actually
does: `withNode()` on a node already in the graph replaces it with an identical copy.
The edges are untouched. The version bumps. Structurally, nothing changes. The
mutation is a no-op. The reconciliation loop already retries ABSENT nodes every cycle
automatically — that IS the retry mechanism.

The second version collapsed retry and AI review into a single `ProvisionEscalationFaultPolicy`
that owns the full `PROVISION_FAILED` lifecycle. Events 1-3: return empty (the loop
retries naturally). Event 4: create an `AI_REVIEW` node with a deterministic ID.
Event 5+: check the review registry — PENDING means wait, RESOLVED means the fix
takes effect next cycle, UNRESOLVED means escalate to a `HUMAN_REVIEW` node with
`requiresHuman = true`.

Three fault policies total. `ProvisionEscalationFaultPolicy` for provision failures.
`QuarantineFaultPolicy` for QUARANTINED validators. `SchemaDriftFaultPolicy` for
schema drift. Clean separation by `FaultType` and `NodeType`. No shared state.

## The forcing function

The pipeline example is a harder domain than the dungeon. It has fan-in dependencies,
multiple interacting fault policies, fault-generated transient nodes, schema
compatibility as a provisioning constraint, and a reconciliation loop that actually
exercises the fault feedback path. The dungeon's fault policy responds to
`NODE_DESTROYED` from `EventSource` — it never touches the reconciliation loop's
internal fault processing. The pipeline forces every path through that loop.

Three runtime bugs fixed: DRIFTED detection (#32), CAS race (#33), deprovision
fault typing (#34). All three existed since the runtime was written. The dungeon
was too gentle to expose them.

The runtime grew up because the second example was harder than the first. That's
the point of a second example.
