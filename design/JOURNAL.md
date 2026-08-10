# Design Journal — issue-27-managed-pipeline-mode

## 2026-06-18/19 — Worker Foundation Extraction + #27 brainstorming

### §WorkerExtraction

Issue #27 (managed pipeline mode) surfaced a deeper architectural question: Worker primitives live in casehub-engine-api (orchestration tier), making them unavailable to foundation repos without bridge modules. Worker is conceptually foundational — "a named unit of automated work" — peer to WorkItem.

**Decision:** Extract Worker to a new foundation-tier `casehub-worker` repo. Add execution governance types (ExecutionPolicy, RetryPolicy, BackoffStrategy) and PolicyEnforcer to casehub-platform. Both delivered this session.

**Key design choices:**
- WorkerFunction is an extensible interface (not sealed) — higher tiers add variants (Flow, AgentExec)
- Engine uses foundation Worker via composition — no PlanElement on Worker, no AgentDescriptor field
- PolicyEnforcer is generic (not Worker-specific) — any callable can use it
- Governance module is a separate submodule in platform repo to encourage co-location of future governance concerns

### §PipelineBrainstorming

Brainstorming for #27 identified two concerns that must not be conflated:
1. **Provisioning** — making a stage real (one-time: preconditions, state, wiring)
2. **Runtime processing** — what the stage does with data (ongoing: Camel, Drools, streaming)

ExecutionBackend sits at the provisioning layer. "DroolsWorkerFunction" doesn't make sense as a provisioning concept — Drools is a runtime processing technology. The relationship between ExecutionBackend, Worker, and WorkerFunction remains unresolved. Session paused to complete the Worker extraction first; #27 brainstorming resumes after all migration issues land.
