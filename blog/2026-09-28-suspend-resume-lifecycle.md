---
title: "Suspend/Resume — a four-verb lifecycle for stateful resources"
date: 2026-09-28
entry_type: note
subtype: diary
series: issue-152-suspend-resume-lifecycle
projects: [casehubio/casehub-desiredstate]
author: mdp
tags: [desired-state, lifecycle, spi, api-design]
---

# Suspend/Resume — a four-verb lifecycle for stateful resources

The desired-state runtime understood two things about resources: create them and destroy
them. That's fine for stateless infrastructure — DNS records, API routes, load balancer
rules. Tear it down, rebuild it, nothing is lost.

But tmux sessions carry conversation history. Containers carry filesystem layers.
Computation environments carry cached state. Destroying these on idle and recreating
on demand throws away the one thing that makes them worth keeping.

The forcing function was specific: Claudony's agent pool (casehubio/claudony#234).
Each agent session is a tmux process with conversation history on disk. When the pool
releases a session, the correct lifecycle is suspend (kill the tmux process, state
persists on disk) and resume (recreate the process, restore via the conversation ID).
Deprovision/provision destroys the conversation history and starts fresh — exactly
the wrong thing.

## The layering principle

We extended `NodeProvisioner` with two opt-in default methods — `suspend()` and
`resume()` — plus a `supportsStatefulLifecycle()` capability query. A provisioner
that doesn't override these gets the stateless path unchanged: deprovision on idle,
provision on demand. The runtime checks the capability flag and substitutes
automatically.

The key design choice was where the idle signal comes from. Three options:

1. **Desired graph annotation** — a `TargetStatus` enum (ACTIVE/SUSPENDED) on `DesiredNode`
2. External API call
3. Timer/policy-driven

We chose (1). The GoalCompiler already owns what-should-exist; extending it to
lifecycle intent keeps the desired graph as the single source of truth. No out-of-band
state, no separate APIs.

## The decision matrix

`TransitionPlanner` gained a 5×2 decision matrix — actual status crossed with target
status. Most cells are straightforward, but one is genuinely interesting: what happens
when a node is ABSENT but the target is SUSPENDED?

The answer: attempt `resume()`. The provisioner has domain knowledge the planner
doesn't. A crashed tmux session's process is gone (ABSENT) but the conversation
history persists on disk. `resume()` can detect whether the state is recoverable.
If it isn't, it returns Failed and the fault policy takes over.

This is the pattern: the planner decides *what* action to take; the provisioner
decides *how*. The planner doesn't need to know whether state is recoverable —
it just calls resume and trusts the provisioner to make the judgment.

## The HumanGating surprise

The design review recommended migrating `HumanGating` from an enum to an
`EnumSet<StepAction>`-based record — four actions with an enum means lossy merge
when combining arbitrary gating combinations. Sound reasoning. One problem:
`@Node(humanGating = HumanGating.PROVISION_ONLY)` requires an enum type. Java
annotation attributes are limited to primitives, String, Class, enums, annotations,
and arrays. Records are not permitted.

The fix was pragmatic: keep the enum, add `SUSPEND_ONLY` and `RESUME_ONLY`, accept
the combinatorial limitation for now. The full migration needs the annotation
processor updated in tandem — a separate issue.

## What this opens up

The stateful lifecycle is the foundation for resource pooling. A pool is just a
set of nodes with a controller that sets `TargetStatus.SUSPENDED` on release and
`TargetStatus.ACTIVE` on acquire. The reconciliation loop does the rest — suspend
the process, resume it, handle faults. No new runtime machinery needed, just a
GoalCompiler that manages the target statuses.

The deferred items — CaseTransitionExecutor support, YAML plugin `suspend:`/`resume:`
sections, annotation surface `targetStatus` — are follow-on work that layers
cleanly on top. The runtime is ready; the surfaces need wiring.
