# Design Journal — issue-130-plan-preview-approval-gate

## 2026-10-01 — Session 1: Design + runtime implementation

### What happened

Brainstormed, designed, and implemented the runtime layer for both #130 (plan preview/approval gate)
and #159 (ordering constraints/flat-graph fast-path). The two features share a branch because both
enhance the TransitionPlan pipeline — one gates execution, the other orders it.

### Key decisions

- **PlanApprovalGate** as injectable component (not loop-internal, not planner-aware) — follows
  the FaultPolicyEngine/DriftPolicyEngine injection pattern. Skip-and-recheck semantics: full
  operational skip during pending prevents drift-triggered invalidation loops.
- **OrderingConstraint on DesiredStateGraph** (revised from CompilationResult) — constraints must
  participate in graph operations (overlay, connect, filterByTypes). Putting them in
  CompilationResult would require plumbing through 5+ methods.
- **Virtual edges at plan time** — TransitionPlanner BFS algorithm unchanged; constraint-matching
  node pairs add virtual in-degree entries during the scan phase.
- Decision review (3 rounds, $31.68) surfaced D9: plan-level and per-node approval coexist
  as independent concerns at different layers.

### What's done

- Batch 1: OrderingConstraint + DesiredStateGraph + ImmutableDesiredStateGraph + TransitionPlanner
  fast-path + virtual edges (419 tests passing)
- Batch 2: PlanApprovalPolicy/PlanApprovalHandler SPIs, PlanApprovalGate, ReconciliationLoop
  integration, CDI + Spring wiring, CloudEvent types, FaultType.PLAN_REJECTED

### What's left

- Batch 3: YAML, annotation, and TS DSL surface integration for ordering constraints
  (YamlGraph, @OrderBefore, TsEnvelope extensions + respective compiler factories)
