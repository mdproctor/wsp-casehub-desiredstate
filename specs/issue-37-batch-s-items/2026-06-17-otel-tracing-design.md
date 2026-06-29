# OTel Tracing for Desired State Runtime

**Issue:** #22  
**Scale:** S — instrumentation of 3 runtime classes  

## Decision

Add `opentelemetry-api` to `runtime/pom.xml` (compile scope). Not in `api/` — tracing is an implementation concern, not an SPI contract. Uni context propagates span context through the TransitionExecutor SPI without changes.

No-op when no OTel SDK is on classpath (examples, lightweight deployments). Active when `quarkus-opentelemetry` is present in the consuming app.

## Instrumentation Scope

**Instrumented:**
- `ReconciliationLoop.reconcile()` — root span with phase-level children
- `SimpleTransitionExecutor.execute()` — per-node child spans under the execute phase
- Drift detection and fault feedback — conditional child spans

**Not instrumented (pure computation, phase span sufficient):**
- `TransitionPlanner.topologicalSort()`
- `FaultPolicyEngine` per-policy evaluation

## Span Tree

```
reconcile [desiredstate.tenant.id={tenancyId}]
├── readActual [desiredstate.node.count={n}]
├── detectDrift [desiredstate.drift.count={n}]
├── plan [desiredstate.additions={n}, desiredstate.removals={n}]
├── execute
│   ├── deprovision [desiredstate.node.id, desiredstate.node.type]
│   └── provision [desiredstate.node.id, desiredstate.node.type, desiredstate.requires.human]
└── faultFeedback [desiredstate.fault.count={n}, desiredstate.mutation.count={n}]
```

## Implementation Details

- **Tracer:** `GlobalOpenTelemetry.getTracer("io.casehub.desiredstate")` — library instrumentation pattern, no CDI injection.
- **Attribute prefix:** `desiredstate.*` for domain-specific attributes.
- **Error semantics:** Failed cycles set span status ERROR with exception. Per-node failures set node span ERROR. Cycle span stays OK when node failures occur (handled by fault policies).
- **detectDrift and faultFeedback spans:** only created when work is done (drift found / failures exist). No empty spans.

## Testing

- `opentelemetry-sdk-testing` (test scope) with in-memory `SpanExporter`.
- Verify span names, parent-child relationships, and key attributes.
- Existing tests unaffected — no SDK means no-op tracer.

## Files Changed

| File | Change |
|------|--------|
| `runtime/pom.xml` | Add `opentelemetry-api` dependency |
| `ReconciliationLoop.java` | Instrument `reconcile()` with phase spans |
| `SimpleTransitionExecutor.java` | Instrument per-node provision/deprovision spans |
| `runtime/pom.xml` (test) | Add `opentelemetry-sdk-testing` test scope |
| New: `ReconciliationTracingTest.java` | Span structure and attribute assertions |
