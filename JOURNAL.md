# Design Journal — issue-106-graph-rule

## 2026-08-24 — Design phase complete, Batch 1 implemented

Brainstormed and spec'd @GraphRule — declarative graph rewriting for the annotation-driven compilation pipeline. 10 design decisions captured, 6 revised during standard decision review. Key decisions: sequential parameter chaining with optional named `of` bindings (D1), collect-then-apply fixed-point semantics (D4), explicit @GraphRule per method on standalone classes (D8, revised from convention-over-configuration after review caught accidental discovery hazard), and clean boundary between @GraphRule and FaultPolicy (D10, surfaced by reviewer).

Spec review added mutation ordering (AddNode before AddDependency, RemoveNode before AddDependency to prevent false-positive cycles), cycle pre-validation before applying mutations, and CompilationResult.Lifecycle handling (rules evaluate per-phase independently).

Batch 1 (Foundation) complete: 5 annotations, Direction enum, 4 IR types, GraphDescriptor extended with graphRules field, GraphMutation.targetNodeId() extracted from GraphDiff, 2 exception classes. All 16 modules build green.
