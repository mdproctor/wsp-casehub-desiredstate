# AgentProvider Integration for AI_REVIEW Nodes

**Issue:** #29  
**Scale:** S — single provisioner change + dependency  

## Decision

Add `casehub-platform-agent-api` as compile dependency to the pipeline example. Wire `AgentProvider` into `PipelineProvisioner`. When an AI_REVIEW node is provisioned and no pre-set outcome exists, invoke the LLM to diagnose the fault.

## Behavior

1. Pre-set outcome exists in PipelineWorld → use it (test path, unchanged)
2. No pre-set + AgentProvider available → invoke `AgentProvider.invoke()` with diagnostic prompt → parse response → store RESOLVED/UNRESOLVED in PipelineWorld
3. No pre-set + NoOpAgentProvider (empty response) → register as PENDING (existing behavior)

## Diagnostic Prompt

Built from `AiReviewSpec(targetNodeId, errorDetail)`:
- System: "You are a data pipeline fault diagnostic agent. Analyze the error and determine if you can resolve it. Respond with RESOLVED if you can fix the issue, or UNRESOLVED if human intervention is needed."
- User: "Node {targetNodeId} failed with: {errorDetail}"

## Response Parsing

Collect `Multi<AgentEvent>` into a single string. If empty → PENDING. If contains "RESOLVED" (case-insensitive, checked before "UNRESOLVED") → RESOLVED. Otherwise → UNRESOLVED.

## Files Changed

| File | Change |
|------|--------|
| `examples/pipeline/pom.xml` | Add `casehub-platform-agent-api` compile dep |
| `PipelineProvisioner.java` | Accept `AgentProvider`, invoke on AI_REVIEW provision |
| `PipelineTest.java` | New test: agent integration with mock AgentProvider |
