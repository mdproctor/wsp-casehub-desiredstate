# Decisions — Issue #74: Summarisation→RAS Integration Scope

## D1: Integration pattern — CloudEvent re-entry via CDI bus

**Choice:** Loose coupling via CloudEvents. Summarised L2/L3 events re-enter the CDI CloudEvent bus as new CloudEvents with domain-specific type URIs. RAS Ganglia consume them identically to raw L1 events.
**Alternatives:**
- Direct RAS feed — summarised events fed directly to RAS engine via new API, bypassing CloudEvents. Tighter coupling, lower latency, but new SPI surface and non-standard integration path.
- EventSource bridge — summarised events wrap as desiredstate StateEvents via EventSource SPI, feeding back into ReconciliationLoop. Conflates summarised signals with actual-state events.
**Rationale:** CloudEvents are the platform's universal async event envelope (`Event<CloudEvent>.fireAsync()`). RAS already consumes CloudEvents. No new SPI needed — summarised events are just CloudEvents with different type URIs. The platform's existing CDI event infrastructure handles dispatch, and the `tenancyid` extension attribute propagates naturally.
**Trade-offs:** Double CloudEvent serialisation (L1 emitted → ingested → L2/L3 emitted). Acceptable for detection latency requirements. Every domain does the same CDI wiring for ingestion/emission, mitigated by bridge adapters.
**Sources:** `casehub-platform-api` CloudEvent convention (capability-ownership.md), `DesiredStateEventTypes.java`, `NodeFaultGanglion.java`, GE-20260730-d761e5 (tenancyid extension requirement)
**Exploration:** quick
**Status:** captured

## D2: Bridge module location — new module in blocks

**Choice:** CloudEvent↔Summarisation bridge adapters live in a new module in casehub-blocks.
**Alternatives:**
- New module in desiredstate — keeps it close to the reconciliation consumer, but creates a Foundation→Foundation-adjacent dependency edge (desiredstate→blocks). Violates the natural dependency flow.
- New standalone repo — maximum decoupling but repo management overhead for a small utility.
**Rationale:** blocks already owns the summarisation framework. blocks is downstream (depends on engine-api, work-api, qhorus-api) — adding platform-api (for CloudEvent types) follows the same pattern. No new cross-tier dependency edges created.
**Trade-offs:** Consumers of the bridge need blocks as a dependency. For desiredstate examples that want summarisation, this means a blocks dependency on the example module (acceptable for examples, not for core modules).
**Sources:** Platform overview (dependency/build order), boundary-rules.md ("do not add domain logic to foundation repos")
**Exploration:** quick
**Status:** captured

## D3: Bridge scope — adapters only, no pipeline builder

**Choice:** The bridge module provides two adapters: CloudEventIngestionAdapter (CloudEvent → LevelEvent) and CloudEventEmitter (summarised output → CloudEvent). Domains wire their own SummarisationRunner pipelines explicitly.
**Alternatives:**
- Adapters + pipeline builder — a declarative SummariserPipeline that auto-discovers @ApplicationScoped Summariser beans and wires L1→L2→...→CloudEvent output. More turnkey but more magic.
- Full framework — adapters + pipeline builder + generic phase-tracking state machine. Maximum reuse but risks over-engineering before a second consumer validates the abstraction.
**Rationale:** Explicit wiring is ~10 lines of CDI setup per domain. Auto-discovery adds non-obvious resolution rules and ordering problems. The YAML surface (D7) provides the declarative pipeline wiring, making a programmatic pipeline builder redundant.
**Trade-offs:** Each domain writes its own wiring code. Mitigated by the YAML surface making this declarative.
**Sources:** GE-20260629-e8b16d (EventStreamBus lifecycle gotcha — explicit wiring avoids hidden subscription management)
**Exploration:** quick
**Depends on:** D7 (YAML surface makes programmatic pipeline builder unnecessary)
**Status:** captured

## D4: Ganglia — domain-specific, not generic

**Choice:** Each domain writes its own RAS Ganglia for its own summarised CloudEvent types. The bridge module does not provide generic Ganglia.
**Alternatives:**
- Generic PhaseTransitionGanglion in the bridge module — parameterised by event type and phase field extraction. Reusable but assumes all domains model phase transitions the same way.
**Rationale:** Ganglia detection logic is inherently domain-specific. A "congestion" phase means something different in logistics vs. infrastructure vs. IoT. The CloudEvent type URI is the only abstraction boundary — Ganglia subscribe to specific types and implement domain-appropriate detection.
**Trade-offs:** Each domain writes ~30 lines of Ganglion code. This is the correct amount of domain-specific logic.
**Sources:** `NodeFaultGanglion.java` (30-line reference implementation), GE-20260817-ce1de5 (CloudEventExpressionContext structure for ExpressionRules)
**Exploration:** quick
**Status:** captured

## D5: Desiredstate is not needed in the logistics example

**Choice:** The logistics example is a pure blocks + RAS demonstration. Desiredstate's value with summarisation belongs in casehub-ops (deployment topology enhancement), not in a teaching example.
**Alternatives:**
- Logistics example uses desiredstate — models the logistics network as a desired-state graph (routes, hubs, capacity). Desiredstate provisions and reconciles the topology; summarisation feeds RAS for replanning. Possible but forced — the interesting part is the summarisation→RAS pipeline, not node provisioning.
**Rationale:** The logistics scenario is fundamentally an event processing + situation detection problem. Desiredstate's value proposition (gap between desired and actual state, provisioning, drift, reconciliation) doesn't naturally apply. The genuine desiredstate + summarisation use case is ops deployment topologies, where ReconciliationLoop already emits CloudEvents and adding summarisation gives RAS altitude for detection.
**Trade-offs:** Desiredstate's practical integration with summarisation is validated via a separate ops issue, not in this design's scope. The logistics example validates the bridge and YAML surface without proving the desiredstate use case directly.
**Sources:** GE-20260616-02d0a7 (CaseHub entities have zero hard creation-time dependencies — flat graph), issue #74 original analysis
**Exploration:** deep-analysis
**Status:** captured

## D6: Example lives in blocks

**Choice:** The logistics example lives in casehub-blocks as a new example module (e.g. `examples/logistics/`).
**Alternatives:**
- Example in desiredstate — keeps desiredstate examples together but creates an upstream→downstream dependency edge (desiredstate→blocks).
- Separate integration repo — avoids new edges between existing repos but adds repo management overhead.
**Rationale:** blocks is downstream in the dependency graph (already depends on engine-api, work-api, qhorus-api). Adding desiredstate-api for the ops enhancement example follows the same direction. The logistics example doesn't use desiredstate at all (D5), so it's purely a blocks + RAS example — natural home is blocks.
**Trade-offs:** Desiredstate examples remain self-contained. The logistics example is in a different repo from the pipeline/dungeon/expansion examples, but it tests a different capability (summarisation, not graph management).
**Sources:** Platform overview (build/dependency order)
**Depends on:** D5 (logistics example doesn't need desiredstate)
**Exploration:** quick
**Status:** captured

## D7: Composable YAML runtime for blocks summarisation

**Choice:** Design a YAML surface for blocks summarisation with composable runtimes. Tier 1: YAML standalone with built-in summariser types (threshold-classify, phase-detect, count, field-extract, pass-through). Tier 2: YAML + Java custom @SummariserTypeId classes on classpath extend available types. Built-in summariser types ship in the YAML module itself — one dependency for standalone use.
**Alternatives:**
- Java-only API — no YAML surface. Domains wire everything programmatically. Simpler to implement but misses the operator-accessible declarative goal.
- Separate builtins module — YAML module is pure parsing; built-in summariser types in a separate jar. More granular but two dependencies for the standalone case.
**Rationale:** Follows the proven desiredstate pattern: YAML declares topology, @NodeTypeId maps types to Java classes, NodeSpecRegistry discovers at build time. Same model: YAML declares pipeline, @SummariserTypeId maps types to Java Summariser implementations, SummariserRegistry discovers at build time. Expression language from casehub-platform-expression (MVEL3 or JQ) powers the built-in rule-based summarisers.
**Trade-offs:** Designing a good YAML surface and built-in summariser types is significant work. The built-in types must cover enough cases to make Tier 1 genuinely useful standalone, or the YAML surface is just ceremony over Java. The logistics example is the validation — if it can be expressed primarily in YAML, the surface works.
**Sources:** desiredstate YAML model types (`YamlGraph`, `YamlNode`, `YamlRule`), `NodeSpecRegistry`, `@NodeTypeId`, `medallion-pipeline.yaml` (reference YAML example), `casehub-platform-expression` (MVEL3 + JQ)
**Exploration:** deep-analysis
**Status:** captured
