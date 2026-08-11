---
layout: post
title: "Two bugs and an orphaned epic"
date: 2026-07-19
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-desiredstate]
tags: [overlay, cdi, graph-equality, housekeeping]
---

Short session — two bug fixes and some housekeeping.

The more interesting fix was in `ImmutableDesiredStateGraph.overlay`. When two graphs share a node (same `NodeId`), overlay validates they're compatible before merging. But it was only comparing `spec` — ignoring `type` and `humanGating`. Meanwhile `GraphDiff.computeMutations` uses full record equality via `DesiredNode.equals()`. The inconsistency means overlay would silently accept two versions of a node that GraphDiff would flag as different. This was surfaced during the HumanGating design review but not fixed until now.

The fix is one line: `thisNode.spec().equals(otherNode.spec())` becomes `thisNode.equals(otherNode)`. `DesiredNode` is a record — `equals()` compares all four fields (id, type, spec, humanGating) automatically. Two new tests confirm humanGating and type mismatches now throw.

The CDI fix (#84) was mechanical — six classes needed `protected` no-args constructors for CDI proxy generation. casehub-ops had been working around this with duplicate `App*` clones and `quarkus.arc.exclude-types`, and the workaround had its own bug: `AppNodeProvisionerRouter` used the 1-arg `DefaultNodeProvisionerRouter(Collection)` constructor, which falls back to `NoOpPreferences` — silently losing preference-based resync interval overrides.

Also closed epic #61 (aggregate subgraph reasoning). All eight child issues and all three cross-repo dependencies had been closed in previous sessions — the epic just hadn't been closed itself. Found it during the handover resume cross-check.

That cross-check also caught stale items in HANDOFF.md — the Cross-Module "Blocked by" section was listing static architectural dependencies (`CallableDispatchRegistry`, `WorkItemCreator`) as if they were active blockers. They weren't. The SPIs exist, the code compiles, nothing was stuck. Updated the handover skill to require each Cross-Module item to reference a tracked issue and pass the test: "Is there an action someone in another repo needs to take before we can proceed?"
