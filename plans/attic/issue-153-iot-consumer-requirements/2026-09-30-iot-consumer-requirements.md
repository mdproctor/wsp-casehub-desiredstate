# IoT Consumer Requirements — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #153 — IoT consumer requirements — gaps and enhancements for device orchestration
**Issue group:** #153

**Goal:** Create child issues for each foundation gap, reframe #130 for plan preview, close #136 as covered.

**Architecture:** This is a scoping epic — no code changes. The plan creates GitHub issues with enough design direction for each to be independently brainstormed and implemented. Priority order: idempotent provisioning → partial convergence → edge handling → plan preview → drift policy.

**Tech Stack:** GitHub Issues (gh CLI)

## Global Constraints

- Every child issue references #153 as parent
- Scale and complexity tags match the spec
- Cross-references between child issues capture the dependency map
- Issue bodies include scope boundaries (in-scope and not-in-scope) from the spec

---

## Batch 1: Create child issues and manage existing issues

### Task 1: Create child issue — Idempotent provisioning (AlreadyConverged)

**Files:**
- None (GitHub issue creation only)

- [ ] **Step 1: Create the issue**

```bash
gh issue create --repo casehubio/casehub-desiredstate \
  --title "feat: idempotent provisioning — AlreadyConverged result type" \
  --body "$(cat <<'EOF'
## Context

Surfaced by #153 (IoT consumer requirements). Both IoT and ops consumers need to distinguish "provisioned" from "already converged" — "38 already converged, 4 provisioned, 0 failed" is operationally valuable.

## What

Add `record AlreadyConverged() implements ProvisionResult {}` to the sealed interface. Executors treat as `Succeeded` for transition outcome but emit a distinct CloudEvent.

## Scope

**In scope:**
- `ProvisionResult.AlreadyConverged` record in `api/`
- `SimpleTransitionExecutor` and `ParallelTransitionExecutor` handling
- `StepOutcome` mapping — decide whether AlreadyConverged maps to existing Succeeded or gets a new variant
- CloudEvent emission for observability (`NODE_ALREADY_CONVERGED`)
- Example provisioner updates (pipeline, dungeon) demonstrating check-before-dispatch pattern

**Not in scope:**
- `DeprovisionResult` equivalent — ABSENT nodes are already skipped by the planner
- Consumer provisioner implementations (IoT, ops implement in their own repos)

## Key files

- `api/src/main/java/io/casehub/desiredstate/api/ProvisionResult.java`
- `api/src/main/java/io/casehub/desiredstate/api/StepOutcome.java`
- `runtime-core/src/main/java/io/casehub/desiredstate/runtime/NodeStepExecutor.java`
- `runtime-core/src/main/java/io/casehub/desiredstate/runtime/SimpleTransitionExecutor.java`
- `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ParallelTransitionExecutor.java`

## Scale

Scale: XS | Complexity: Low

Parent: #153
EOF
)"
```

- [ ] **Step 2: Record the issue number**

Note the created issue number for cross-referencing in later issues.

### Task 2: Create child issue — Partial convergence reporting

**Files:**
- None (GitHub issue creation only)

**Interfaces:**
- Consumes: Issue number from Task 1 (soft dependency reference)

- [ ] **Step 1: Create the issue**

```bash
gh issue create --repo casehubio/casehub-desiredstate \
  --title "feat: partial convergence reporting — per-node outcomes in ReconciliationCompletedData" \
  --body "$(cat <<'EOF'
## Context

Surfaced by #153 (IoT consumer requirements). IoT operators need "38 of 42 devices converged, 4 failed (light-3: TIMEOUT, lock-7: FAILED)" from a single event. Ops dashboards need the same for deployment topology health.

## What

Enrich `ReconciliationCompletedData` with per-node outcome summary. Currently carries aggregate counts only (`faultCount`, `additionsCount`). `TransitionResult` already has `Map<NodeId, StepOutcome>` — the data exists but isn't surfaced in the CloudEvent.

## Scope

**In scope:**
- Per-node outcome map in `ReconciliationCompletedData` (nodeId → outcome status string)
- Include `AlreadyConverged` outcomes if the idempotent provisioning issue has landed
- Payload size consideration — threshold for very large graphs where per-node detail is omitted

**Not in scope:**
- New CloudEvent types — per-node events already exist (`NodeFaultedData`, `NodeDriftedData`, `NodeRecoveredData`)
- Consumer-side dashboards or UI

## Soft dependency

Benefits from AlreadyConverged landing first (richer outcome map), but not blocked by it.

## Key files

- `api/src/main/java/io/casehub/desiredstate/api/ReconciliationCompletedData.java`
- `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java` (emitCycleEvents)
- `api/src/main/java/io/casehub/desiredstate/api/TransitionResult.java`

## Scale

Scale: S | Complexity: Low

Parent: #153
EOF
)"
```

- [ ] **Step 2: Record the issue number**

Note the created issue number for cross-referencing.

### Task 3: Create child issue — Edge handling (flat graph + ordering constraints)

**Files:**
- None (GitHub issue creation only)

- [ ] **Step 1: Create the issue**

```bash
gh issue create --repo casehubio/casehub-desiredstate \
  --title "feat: edge handling — flat graph optimisation and ordering constraint declarations" \
  --body "$(cat <<'EOF'
## Context

Surfaced by #153 (IoT consumer requirements). IoT graphs are mostly flat (edgeless — all nodes independent) but need occasional ordering constraints ("power breaker before equipment"). The planner should fast-path edgeless graphs and support clean constraint declaration.

## What

Two deliverables:

**A — Flat graph fast-path:** When the desired state graph has no edges, `TransitionPlanner` skips topological sort and fans out all nodes as a single layer. Currently `topologicalSort()` runs regardless.

**B — Ordering constraint declarations:** Structural ordering constraints declared separately from desired state values. "Equipment before breaker" declared once, applied to any desired state touching those nodes.

## Why combined

The fast-path definition of "edgeless" must account for structural constraints. Designing gap A without gap B risks optimising for "no edges in the graph" when the real condition is "no edges AND no structural constraints apply."

## Scope

**In scope:**
- Fast-path detection in `TransitionPlanner` (empty dependency set → single-layer plan)
- Structural constraint declaration model (API types)
- Surface integration for constraint declarations (YAML, annotations, TS DSL)
- Interaction with `ParallelTransitionExecutor` (fan-out is already layer-based — fast-path feeds a single layer)

**Not in scope:**
- Constraint evaluation/validation engine — `@GraphInvariant` and graph rules already exist for that
- Graph cycle detection changes — `topologicalSort()` already detects cycles

## Key files

- `runtime-core/src/main/java/io/casehub/desiredstate/runtime/TransitionPlanner.java`
- `api/src/main/java/io/casehub/desiredstate/api/DesiredStateGraph.java`
- `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ParallelTransitionExecutor.java`

## Scale

Scale: M | Complexity: Medium

Parent: #153
EOF
)"
```

- [ ] **Step 2: Record the issue number**

Note the created issue number for cross-referencing.

### Task 4: Reframe #130 and close #136

**Files:**
- None (GitHub issue management only)

- [ ] **Step 1: Add IoT/ops consumer context to #130**

```bash
gh issue comment 130 --repo casehubio/casehub-desiredstate \
  --body "$(cat <<'EOF'
## Consumer validation from #153

Two consumers validate this feature:

**IoT:** Operators want "here's what will change" before 50 devices converge. Pre-execution review of the transition plan.

**Ops:** Audit trail — "desired state transition applied these 12 commands." Compliance requirement for all topology changes.

**Implementation note:** `TransitionPlanner.plan()` already returns structured `TransitionPlan` with layered steps and before/after graphs. The planner is separable from execution today — the gap is API and lifecycle (two-phase: plan, then optionally execute), not new computation.

**Scoping from #153:** Plan preview only. Simulated provisioning (provisioner `dryRun()`) is a clean future extension that layers on top without rework.

Parent: #153
EOF
)"
```

- [ ] **Step 2: Close #136 as covered**

```bash
gh issue close 136 --repo casehubio/casehub-desiredstate \
  --comment "$(cat <<'EOF'
Closing — scoped by #153 as covered by #130 (plan preview).

"Dry-run" resolves to two things:
1. **Plan preview** — expose transition plan before execution → #130
2. **Simulated provisioning** — provisioner `dryRun()` method → future SPI extension if needed

No foundation gap remains that isn't addressed by #130.
EOF
)"
```

### Task 5: Create child issue — Drift policy layer

**Files:**
- None (GitHub issue creation only)

- [ ] **Step 1: Create the issue**

```bash
gh issue create --repo casehubio/casehub-desiredstate \
  --title "feat: drift policy layer — permitted drift exemptions with revert modes" \
  --body "$(cat <<'EOF'
## Context

Surfaced by #153 (IoT consumer requirements), validated against ops (casehubio/casehub-ops#26). Any domain with reactive overrides needs permitted drift. IoT: "thermostat overridden by user, don't fight it for 2 hours." Ops/SOC: "emergency manual scaling, exempt from reconciliation until threat level drops."

See casehubio/platform#486 § Drift policy model.

## What

New `DriftPolicy` SPI consulted before the planner — filters which drifted nodes enter the transition plan. Separate from `FaultPolicy` (which handles post-execution failure response).

Drift exemption is a pre-planning concern ("should I act on this drift?"), not a fault response ("action failed, what now?"). Different lifecycle, different question.

## Scope

**In scope:**
- `DriftPolicy` SPI in `api/` — `evaluate(NodeId, NodeStatus, DesiredNode, DriftContext) → DriftDecision`
- `DriftDecision`: `RECONCILE` (default) | `EXEMPT(ExemptionSpec)`
- `ExemptionSpec`: revert modes (duration, schedule, event-based, never), metadata
- `ReconciliationLoop.detectDrift()` consults DriftPolicy before creating fault events
- Exemption lifecycle: grant, track, expire, reconcile on next loop
- Exemption storage SPI (pluggable, analogous to FaultCountStore)
- `@DefaultBean` / no-op fallback (all drift reconciled — backward compatible)

**Not in scope:**
- Drift policy declaration surfaces (YAML, annotations) — those layer on top of the SPI
- Domain-specific drift policies (IoT, ops implement in their own repos)

## Key files

- `api/` — new `DriftPolicy` SPI, `DriftDecision`, `ExemptionSpec`
- `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java` (detectDrift)
- `runtime/` — CDI wiring, `@DefaultBean` fallback

## Cross-references

- casehubio/platform#486 — drift policy model
- casehubio/iot#120 — IoT desired state integration
- casehubio/casehub-ops#26 — SOC adaptive ops

## Scale

Scale: L | Complexity: High

Parent: #153
EOF
)"
```

- [ ] **Step 2: Record the issue number**

Note the created issue number.

### Task 6: Update #153 with child issue cross-references

**Files:**
- None (GitHub issue management only)

**Interfaces:**
- Consumes: All issue numbers from Tasks 1-5

- [ ] **Step 1: Add tracking comment to #153**

Post a comment on #153 listing all child issues with their scale and priority order:

```bash
gh issue comment 153 --repo casehubio/casehub-desiredstate \
  --body "$(cat <<'EOF'
## Child issues created

Priority order (smallest-first):

1. #<N1> — Idempotent provisioning (AlreadyConverged) — XS/Low
2. #<N2> — Partial convergence reporting — S/Low
3. #<N3> — Edge handling (flat graph + ordering constraints) — M/Med
4. #130 — Plan preview (reframed) — S/Med
5. #<N4> — Drift policy layer — L/High

**Dropped:** Gap 5 (saved state presets) — application concern, not foundation. Presets are just DesiredStateGraphs; variants handled by SituationRecompiler + GoalCompiler.

**Closed:** #136 (dry-run) — covered by #130 + future provisioner dry-run SPI.

Spec: `specs/issue-153-iot-consumer-requirements/2026-09-30-iot-consumer-requirements-design.md`
EOF
)"
```

- [ ] **Step 2: Commit workspace changes**

```bash
git -C /Users/mdproctor/claude/public/casehub-desiredstate add .
git -C /Users/mdproctor/claude/public/casehub-desiredstate commit -m "wip(plan): finalize #153 child issue creation Refs #153"
```

## References

- [2026-09-30-iot-consumer-requirements-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/desiredstate/api/ProvisionResult.java` — AlreadyConverged target
- `api/src/main/java/io/casehub/desiredstate/api/ReconciliationCompletedData.java` — convergence reporting target
- `runtime-core/src/main/java/io/casehub/desiredstate/runtime/TransitionPlanner.java` — flat graph optimisation target
- `runtime-core/src/main/java/io/casehub/desiredstate/runtime/ReconciliationLoop.java` — drift policy integration point
- GitHub #130 — plan preview (reframe target)
- GitHub #136 — dry-run (close target)
- GitHub #153 — parent epic
