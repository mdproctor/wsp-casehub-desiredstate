# Handoff — casehub-desiredstate

## Last Session

Pure R&D session exploring #24 (QuarkMind desired-state integration). No code written. Reframed #24 from "add a DesiredStateVector data type" to "desired state as a planning paradigm" — desired state is simplified classical planning (STRIPS/HTN) reinvented by engineers. Promoted #24 to epic with three POC sub-issues: #58 (lifecycle transitions), #56 (planner-backed GoalCompiler), #57 (spatial/vector stress test). Recommended ordering: #58 → #56 → #57. ARC42STORIES.MD stale scan fixed 3 references where #47 was closed but still marked pending.

## Immediate Next Step

Start #57 (spatial/vector stress test) — model a "defend this region" scenario as a DesiredStateGraph with region-as-node, strength-as-spec. Try the graph model first. Full context in the issue body. Run `/work` to begin.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #58 | POC: desired state lifecycle transitions | S | Med | Prerequisite for #56 and #57 |
| #56 | POC: planner-backed GoalCompiler | M | High | Highest architectural value |
| #57 | POC: spatial/vector stress test | M | High | Most uncertain — graph model might suffice |
| #51 | ActualStateAdapter routing for multi-domain | M | Med | Natural follow-on to #18 |
| #52 | Routing for remaining CDI-ambiguous SPIs | M | Med | GoalCompiler, EventSource, FaultPolicy |
| #46 | Pipeline example — PendingApproval gate | S | Low | Unblocked |
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | Paused on stack |
| #23 | CBR integration for desired-state evolution | M | High | Needs casehub-neural-text |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Epic: `casehubio/casehub-desiredstate#24` — full conversation analysis and coupling matrix
- Blog: `blog/2026-07-02-mdp01-the-plan-that-was-already-there.md`
- Garden: GE-20260701-82909e (ScheduledFuture map replacement race)
