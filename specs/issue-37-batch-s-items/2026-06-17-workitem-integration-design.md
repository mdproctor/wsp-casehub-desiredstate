# WorkItem Integration for HUMAN_REVIEW Nodes

**Issue:** #30  
**Scale:** S — CaseTransitionExecutor change  

## Decision

`CaseTransitionExecutor.buildCaseDefinition()` partitions additions into human nodes (`requiresHuman=true`) and automated nodes. Automated nodes go into the grow Worker(Workflow) as before. Each human node gets a separate `humanTask` binding using `HumanTaskTarget.inline()` — the engine's HITL infrastructure creates a WorkItem when the case starts.

## Changes to buildCaseDefinition

1. Filter `plan.additions()` into `humanSteps` and `automatedSteps`
2. `automatedSteps` → grow Worker(Workflow) (existing behavior)
3. Each `humanSteps` entry → `Binding.builder().humanTask(HumanTaskTarget.inline().title(...).build()).on(ContextChangeTrigger).build()`
4. Title derived from node ID: `"Review: {nodeId}"`
5. Input mapping passes node context as JQ expression

## Changes to buildOptimisticResult

Human nodes → `StepOutcome.Skipped("routed to WorkItem")` instead of `Succeeded`.

## Files Changed

| File | Change |
|------|--------|
| `CaseTransitionExecutor.java` | Partition human nodes, add humanTask bindings, update result |
| `CaseTransitionExecutorTest.java` | New: verify humanTask bindings for human nodes |
