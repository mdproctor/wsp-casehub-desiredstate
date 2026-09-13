# Design Journal — issue-140-cross-domain-orchestration

## 2026-09-12 — Design: brainstorming and decisions

Explored the cross-domain orchestration problem space. The runtime is already
multi-domain-ready for everything except GoalCompiler composition — routers,
MergedEventSource, FaultPolicyEngine all dispatch across domains. The gap is
an orchestration layer that calls multiple compilers and merges their graphs.

Key design decisions (10 captured, expanded to 15 after standard review):

- **Hierarchical from day one** (D1) — user override of review's YAGNI cut.
  Build for the future, not just current consumers. Transparent flattening (D6)
  makes single-process the default experience.
- **Push model** (D2) — domains compile themselves and register CompilationResult.
  Eliminates the GoalCompiler type erasure problem entirely. No DomainDescriptor SPI
  needed. The spatial example already validates the overlay pattern.
- **Type-level cross-domain deps** (D4) — provides/requires on DomainRegistration.
  Composition engine matches requires→provides automatically, adds edges in flattened
  mode. First-principles analysis confirmed type-level is the right abstraction
  boundary (more precise than domain-level, no naming coupling of node-level).

Standard decision review (2 rounds) revised D1 to flat-only on YAGNI grounds —
user overrode, restoring hierarchical. Review surfaced 5 valuable new decisions
(D11-D15) including tenancy model, fault propagation trust assumptions, per-domain
lifecycle tracking via GlobalReconciliationListener, single CAS, and node ID
uniqueness validation.

## 2026-09-13 — Design: spec writing and review

Wrote 787-line design spec covering both flattened and hierarchical modes.
Standard spec review (2 rounds) strengthened the spec with:
- Mode immutability (set at startup, cannot change at runtime)
- readinessCondition made non-null via Builder default
- Thread safety model with recomposeLock
- TenantCompositionState/DomainPhaseState immutable records
- SituationRecompiler ordering preserved via priority-sorted list
- DesiredStateReplanDispatch interception pattern for composed mode
- CompilationResult.single() always passed to LifecycleManager in composed mode

## 2026-09-13 — Implementation: Batch 1 (Foundation) and Batch 2 (Flattened)

Implemented 5 of 7 tasks across 2 batches:

**Batch 1 — Foundation (2 tasks):**
- DomainId value type in api/ (4 tests)
- DomainRegistration record with Builder, DomainPhaseState, TenantCompositionState (8 tests)
- CrossDomainCompositionEngine with registration + startup validation: duplicate provides,
  unsatisfied requires, circular dependency detection (Kahn's), node ID uniqueness,
  late registration guard (8 tests)

**Batch 2 — Flattened mode (3 tasks):**
- Overlay composition via recompose() + type-based cross-domain edge creation (5 tests)
- Per-domain lifecycle tracking via GlobalReconciliationListener — phase advancement,
  thread-safe recompose with synchronized(recomposeLock) (4 tests)
- SituationRecompiler integration — handleReplan() with domain graph scoping,
  DesiredStateReplanDispatch modified to delegate when composition is active (4 tests)

Total: 29 new tests, all passing. Only change to existing code: DesiredStateReplanDispatch
gains optional Instance<CrossDomainCompositionEngine> injection with null-safe check.

**Remaining: Batch 3 (2 tasks):**
- Task 6: Hierarchical mode — DomainNodeSpec, DomainNodeProvisioner, DomainActualStateAdapter,
  meta-loop, inner loops via ReconciliationLoop.Builder
- Task 7: End-to-end integration tests (flattened + hierarchical + backward compat)
