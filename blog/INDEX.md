# Blog Index

| Entry | Date | Summary |
|-------|------|---------|
| [2026-06-16-mdp01-the-pipeline-that-broke-the-runtime.md](2026-06-16-mdp01-the-pipeline-that-broke-the-runtime.md) | 2026-06-16 | Data pipeline example exposed three runtime bugs — CAS race, DRIFTED detection gap, edge destruction on RemoveNode |
| [2026-06-17-mdp01-enforcing-what-was-already-true.md](2026-06-17-mdp01-enforcing-what-was-already-true.md) | 2026-06-17 | Medallion layer enforcement — Optional layerOf(), domain-level graph constraint, where enforcement belongs |
| [2026-06-17-mdp02-wiring-the-three-tiers.md](2026-06-17-mdp02-wiring-the-three-tiers.md) | 2026-06-17 | Three-tier fault escalation wired to real SPIs — OTel tracing, AgentProvider for AI review, humanTask bindings for WorkItems |
| [2026-06-18-mdp01-the-toolbox-that-wasnt.md](2026-06-18-mdp01-the-toolbox-that-wasnt.md) | 2026-06-18 | Per-node execution toolbox — brainstorming surfaced Worker data coordination patterns (DataExchange, DataChannel), built the right-sized strategy pattern instead |
| [2026-06-25-mdp01-the-one-line-fix-that-wasnt.md](2026-06-25-mdp01-the-one-line-fix-that-wasnt.md) | 2026-06-25 | DRIFTED re-provisioning — one-line fix revealed two bugs and a non-exhaustive status dispatch; 2×4 matrix, compile-time exhaustive switches |
| [2026-06-25-mdp02-clearing-the-backlog.md](2026-06-25-mdp02-clearing-the-backlog.md) | 2026-06-25 | Batch of 5 S/XS issues — PendingApproval sealed variants, tenancyId through SPIs, pipeline wired to CaseTransitionExecutor, Worker extraction epic (#40) closed |
| [2026-06-28-mdp01-the-spi-that-was-missing.md](2026-06-28-mdp01-the-spi-that-was-missing.md) | 2026-06-28 | HumanNodeHandler SPI — STE delegates requiresHuman to pluggable handler; work-adapter creates WorkItems via WorkItemCreator SPI |
| [2026-06-30-mdp01-the-dispatch-that-wasnt-there.md](2026-06-30-mdp01-the-dispatch-that-wasnt-there.md) | 2026-06-30 | CTE dispatch was non-functional — fixed via CallableDispatchRegistry (engine#590); PendingApproval lifecycle wired into workflow steps |
| [2026-07-01-mdp01-five-provisioners-five-dispatchers.md](2026-07-01-mdp01-five-provisioners-five-dispatchers.md) | 2026-07-01 | Type-aware runtime — multi-provisioner dispatch via NodeProvisionerRouter, per-NodeType reconciliation scheduling, CAS merge-and-retry |
| [2026-07-02-mdp01-the-plan-that-was-already-there.md](2026-07-02-mdp01-the-plan-that-was-already-there.md) | 2026-07-02 | Desired state ≈ classical planning — #24 reframed from data structure to planning paradigm; three POCs: lifecycle, GoalCompiler, spatial |
| [2026-07-06-mdp01-when-desired-state-learns-to-finish.md](2026-07-06-mdp01-when-desired-state-learns-to-finish.md) | 2026-07-06 | GoalCompiler returns CompilationResult (lifecycle phases), LifecycleManager orchestrates transitions via CAS, pipeline gains PendingApproval gate, expansion example with HTN planner |
