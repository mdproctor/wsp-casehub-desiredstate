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
**Rationale:** Explicit wiring is ~30 lines of CDI setup per domain (WindowPolicy, Summariser, EventStreamBus, SummarisationRunner construction, CloudEvent observer, output subscriber, tick scheduling). This is comparable to a GoalCompiler or NodeProvisioner implementation — the cost of entry for a domain that wants summarisation. Auto-discovery adds non-obvious resolution rules and ordering problems. The EventStreamBus lifecycle gotcha (GE-20260629-e8b16d) argues for explicit wiring — hidden subscription management is a known source of bugs. D3 stands independently of D7: even without the YAML surface, explicit wiring is the right default because the pipeline builder's auto-discovery magic introduces ordering and lifecycle issues that explicit construction avoids.
**Trade-offs:** Each domain writes ~30 lines of wiring code. Acceptable — this is one-time per domain, not per endpoint. The YAML surface (D7) provides a declarative alternative for domains that prefer configuration over code.
**Sources:** GE-20260629-e8b16d (EventStreamBus lifecycle gotcha — explicit wiring avoids hidden subscription management), SummarisationRunner constructor (6 parameters: WindowPolicy, Compactor, Summariser, EventStreamBus, EventLevel, onFailure)
**Exploration:** quick → revised via adversarial review (R1-03)
**Reinforced by:** D7 (YAML surface provides a declarative alternative, but D3's rationale does not depend on D7)
**Status:** revised — decoupled from D7 dependency, corrected wiring estimate from ~10 to ~30 lines, strengthened independent rationale

## D4: Ganglia — tiered: expression-based default, domain-specific escape hatch

**Choice:** Built-in summariser types (D7 Tier 1) produce standardised CloudEvent output schemas. Ganglia for built-in summariser output use ExpressionRulesGanglion — parameterised via YAML with expression-based rules over CloudEventExpressionContext. Domain-specific Java Ganglia (extending JavaSwitchGanglion) remain available for custom summarisers or complex multi-signal correlation.
**Alternatives:**
- Domain-specific only — each domain writes Java Ganglia for all its CloudEvent types. Simple and type-safe but misses the declarative opportunity that ExpressionRulesGanglion and D7's standardised output provide.
- Generic only — all Ganglia are expression-based. Overly constraining for domains with complex correlation logic (e.g., multi-signal systemic failure detection across summarised streams).
**Rationale:** ExpressionRulesGanglion already exists in casehub-ras-runtime and provides YAML-configurable detection with CloudEventExpressionContext. NodeFaultGanglion's pattern (type-switch → detected/anti/noise) is a subset of what ExpressionRulesGanglion supports — a `when: type == 'io.casehub.desiredstate.node.faulted'` rule with signal DETECTED is equivalent. If D7's built-in summarisers produce standardised output schemas with known field paths, expression rules can match those paths without domain-specific Java. The two-tier model mirrors D7's Tier 1 (built-in) / Tier 2 (custom Java) structure: YAML-configured expression Ganglia for standardised summariser output, Java Ganglia for custom summariser types.
**Trade-offs:** Expression-based Ganglia are less type-safe than Java switch Ganglia. Complex multi-signal correlation (e.g., congestion + capacity + route failure = systemic breakdown) may exceed expression language capabilities. The escape hatch to Java Ganglia ensures no capability loss.
**Additional capability (R2-01):** RAS YAML situation system (`YamlSituationDefinitionProvider`) provides three ganglion types: `expression-rules` (boolean rule matching), `naive-bayes` (probabilistic detection — relevant for noisy summarised streams), and `situation-watcher` (situation-on-situation composition). D4's tiered approach uses `expression-rules` as the default; `naive-bayes` is a natural fit for domains where summarised signals are probabilistic rather than deterministic (e.g., IoT sensor streams with noise). Chain modes (`and`, `or`, `threshold`, `sequence`, `count`, `streak`, `rate`) compose Ganglia output into complex situation definitions declaratively.
**Sources:** ExpressionRulesGanglion (io.casehub.ras.runtime), CloudEventExpressionContext (GE-20260817-ce1de5), NodeFaultGanglion (30-line reference implementation), JavaSwitchGanglion (api base class), YamlSituationDefinitionProvider (RAS YAML surface — three ganglion types, situation templates, chain modes)
**Exploration:** quick → revised via adversarial review (R1-04), enriched with RAS YAML evidence (R2-01)
**Status:** revised — changed from domain-specific-only to tiered (expression-based default + domain-specific escape hatch)

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
**Trade-offs:** Desiredstate examples remain self-contained. The logistics example is in a different repo from the pipeline/dungeon/expansion examples, but it tests a different capability (summarisation, not graph management). Cross-reference documentation (a note in desiredstate's ARC42STORIES.MD §9.3 and README pointing to the blocks logistics example) addresses the discoverability gap for operators looking for desiredstate + summarisation integration.
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
**Trade-offs:** Designing a good YAML surface and built-in summariser types is significant work. The built-in types must cover enough cases to make Tier 1 genuinely useful standalone, or the YAML surface is just ceremony over Java. The logistics example is the validation — if it can be expressed primarily in YAML, the surface works. Built-in summariser types MUST produce standardised output schemas — this is a design requirement, not optional. Without standardised output, built-in types are not genuinely useful standalone and Tier 1 loses its value.
**Pending sub-decision:** Expression language for built-in summariser rules. `field-extract` needs document transformation (JQ territory), `threshold-classify` needs boolean evaluation (MVEL3 territory). Both are available via `casehub-platform-expression`'s `CompiledExpression<CTX, RESULT>` interface. Options: (a) JQ for all — natural for CloudEvent JSON data, awkward for boolean predicates; (b) MVEL3 for all — natural for predicates, awkward for document transformation; (c) both, selected per built-in type — each type uses the natural language, operator sees only the expression string in YAML. **Production precedent for option (c) (R2-01):** `YamlSituationDefinitionProvider.parseExpressionEntry` in casehub-ras already supports per-expression language selection via a `language` field (`"jq"` → `JQExpressionEvaluator`, `"mvel"` → `MvelExpressionEvaluator`). This is production-proven — option (c) is established practice, not speculative.
**Design pattern (R2-01):** RAS situation templates (`SituationTemplate` with `${paramName}` substitution and deep merge of overrides) provide a reusable parameterisation pattern. D7's YAML surface could adopt this for common summarisation pipeline configurations — e.g., a `threshold-classify` template parameterised by field path and threshold value, instantiated per domain. Resolution deferred to D7 implementation.
**Sources:** desiredstate YAML model types (`YamlGraph`, `YamlNode`, `YamlRule`), `NodeSpecRegistry`, `@NodeTypeId`, `medallion-pipeline.yaml` (reference YAML example), `casehub-platform-expression` (MVEL3 + JQ), `CompiledExpression<CTX, RESULT>` (unified evaluation interface)
**Exploration:** deep-analysis
**Status:** captured — expression language sub-decision noted (R1-06)

## D8: Summarisation types extracted to blocks API module

**Choice:** Extract summarisation types (Summariser, LevelEvent, EventStreamBus, WindowPolicy, EventAccumulator, Compactor, EventLevel, SummarisationRunner) to a new `casehub-blocks-summarisation-api` module before building the bridge and YAML surface on top.
**Alternatives:**
- Keep monolithic blocks jar — simpler module structure but consumers of the bridge take a transitive dependency on ALL of blocks (qhorus-api, work-api, engine-api, eidos-api, worker-api, and all provided-scope deps).
- Full blocks api/ extraction — more comprehensive (extract ALL blocks API types) but larger scope than needed for this issue. Can be done later.
**Rationale:** The module-tier-structure protocol (PP-20260512-module-tiers) requires foundation-tier modules to separate API contracts from runtime implementations. The summarisation types are already pure Java — no CDI annotations, no Quarkus dependencies. Extraction is mechanical: move the `io.casehub.blocks.summarisation` package to a new module. Bridge consumers (D2) and YAML surface consumers (D7) depend on the lightweight API jar, not the full blocks jar. This is a prerequisite for the bridge module (D2) to avoid pulling blocks' full dependency tree.
**Trade-offs:** New module adds build management overhead. Acceptable — the alternative (every bridge consumer transitively depends on qhorus-api, work-api, engine-api, eidos-api) is a clear module-tier-structure protocol violation.
**Sources:** module-tier-structure protocol (PP-20260512-module-tiers), blocks pom.xml (compile deps: qhorus-api, work-api, engine-api, eidos-api, worker-api)
**Exploration:** surfaced by adversarial review (R1-08)
**Status:** captured

## D9: Standard CloudEvent type URIs for built-in summariser output

**Choice:** Built-in summariser types (D7 Tier 1) emit CloudEvents with standardised type URIs following the pattern `io.casehub.blocks.summarisation.<level>.<builtin-type>` (e.g., `io.casehub.blocks.summarisation.L2.threshold-classify`). Custom domain summarisers use domain-specific type URIs (e.g., `io.casehub.logistics.phase.congestion`).
**Alternatives:**
- All domain-specific URIs — each domain defines its own type URIs for all summarised events, including output from built-in summariser types. Prevents cross-domain generic Ganglia (D4) from consuming standardised output without per-domain configuration.
- All standardised URIs — imposes a uniform URI scheme on custom domain summarisers. Too constraining — custom summarisers produce domain-shaped output that doesn't fit a standard schema.
**Rationale:** Standardised URIs for built-in types enable generic ExpressionRulesGanglion configurations (D4) to detect patterns across domains without per-domain Ganglion registration. Combined with D4's tiered Ganglia approach and D7's standardised output schemas, this means an operator can wire a full summarisation→detection pipeline in YAML for standard cases. Domain-specific URIs remain available for custom summariser types — the standard applies only to D7 Tier 1 built-ins.
**Trade-offs:** Standardised URIs couple the CloudEvent contract to the built-in summariser type taxonomy. If a built-in type's output schema changes, all consumers of that URI are affected. Mitigated by treating built-in output schemas as stable API contracts.
**Sources:** DesiredStateEventTypes.java (reference: `io.casehub.desiredstate.reconciliation.completed` pattern), D4 (ExpressionRulesGanglion for standardised output), D7 (built-in summariser types)
**Exploration:** surfaced by adversarial review (R1-11)
**Status:** captured
