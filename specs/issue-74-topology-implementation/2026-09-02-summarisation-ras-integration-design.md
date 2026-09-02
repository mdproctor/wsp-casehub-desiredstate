# Summarisation→RAS Integration — Design Spec

**Date:** 2026-09-02
**Issue:** casehubio/casehub-desiredstate#74
**Decisions:** `specs/issue-74-topology-implementation/decisions.md`

---

## 1. Problem

The platform has two complementary capabilities that don't yet connect:

- **casehub-blocks summarisation** — layered event summarisation (L1 raw → L2 classified → L3 phases) with windowed batching and pluggable summariser logic
- **casehub-ras** — situational awareness via Ganglia (event detection), situation definitions (pattern correlation), and case triggers (automated response)

Today, RAS Ganglia operate directly on raw L1 CloudEvents (e.g., `NodeFaultGanglion` counts `NODE_FAULTED` events). Summarisation could give RAS *altitude* — detecting phase transitions like "east-region entered degraded phase" instead of counting individual faults. This is a qualitatively different signal.

The design answers three questions:
1. How do summarised events reach RAS? (Integration pattern)
2. Where does the reusable plumbing live? (Module placement)
3. How far can YAML go for declaring summarisation pipelines? (Declarative surface)

---

## 2. Integration Pattern — CloudEvent Re-entry

Summarised events re-enter the CDI CloudEvent bus as new CloudEvents with domain-specific type URIs. No new SPI needed.

```
ReconciliationLoop → CloudEvents (L1: NODE_FAULTED, NODE_DRIFTED, etc.)
    ↓ CDI @ObservesAsync CloudEvent
CloudEventIngestionAdapter (bridge module)
    → EventStreamBus<DomainPayload> → SummarisationRunner chain
    → L2/L3 summarised events
CloudEventEmitter (bridge module)
    → Event<CloudEvent>.fireAsync() with domain type URIs
    ↓ CDI @ObservesAsync CloudEvent
RAS Ganglia (expression-based or domain-specific)
    → Situation detection → Case trigger → Response
```

**Why CloudEvents:** The platform standardised on `Event<CloudEvent>.fireAsync()` as the universal async event envelope. RAS already consumes CloudEvents. The `tenancyid` extension attribute propagates naturally. No new transport or SPI surface needed.

**Trade-off:** Double CloudEvent serialisation (L1 emitted → ingested → L2/L3 emitted). Acceptable for detection latency requirements.

---

## 3. Module Structure

### 3.1 Summarisation API Extraction (D8)

Extract pure-Java summarisation types from the monolithic `casehub-blocks` jar into `casehub-blocks-summarisation-api`:

| Type | Current location | Extracted to |
|------|-----------------|--------------|
| `Summariser<IN,OUT>` | `blocks/` | `summarisation-api/` |
| `LevelEvent<E>` (gains `@Nullable String tenancyId`) | `blocks/` | `summarisation-api/` |
| `EventStreamBus<E>` | `blocks/` | `summarisation-api/` |
| `WindowPolicy` | `blocks/` | `summarisation-api/` |
| `EventAccumulator<E>` | `blocks/` | `summarisation-api/` |
| `Compactor<E>` | `blocks/` | `summarisation-api/` |
| `EventLevel` | `blocks/` | `summarisation-api/` |
| `SummarisationRunner<IN,OUT>` | `blocks/` | `summarisation-api/` |
| `KeyedAccumulator<K,E>` | `blocks/` | `summarisation-api/` |
| `KeyedSummarisationRunner<K,IN,OUT>` | `blocks/` | `summarisation-api/` |

These types are already pure Java — no CDI annotations, no Quarkus dependencies. Extraction is mechanical. This prevents bridge consumers from transitively depending on blocks' full dependency tree (qhorus-api, work-api, engine-api, eidos-api).

**Types NOT extracted (remain in `casehub-blocks`):**

| Type family | Reason |
|-------------|--------|
| `ContentSummariser<T>`, `TieredContentSummariser`, `VerbatimContentSummariser`, `ContentSummariserToSummariser` | Depends on `io.casehub.qhorus.api.spi.SummaryResult` — not pure Java |
| `LlmContentSummariser<T>`, `SummaryMode` | Depends on `qhorus-api` + `platform-agent-api` |
| `observation.*` package (`ObservationAccumulator`, `ObservationRenderer`, `TieredObservationRenderer`, `ObservationTier`, `ObservationContext`, `ObservationChunk`, `ObservationResult`) | Terminal consumer of the pipeline — separate concern from pipeline primitives. Uses `LevelEvent` (which moves to the API module) but blocks depends on its own API module, so no breakage |
| `observation.affordance.*` package (`AffordanceRenderer`, `ObservableEntity`, `Affordance`, `ObservationSection`, etc.) | Grounded rendering — agent-facing concern, not pipeline plumbing |

The extraction creates a deliberate package split: `io.casehub.blocks.summarisation` spans two modules. This is architecturally intentional — the API module holds pipeline primitives; blocks holds domain-integrated types that compose those primitives with qhorus, agent, and observation concerns.

### 3.2 Bridge Module — `casehub-blocks-cloudevents`

New module in casehub-blocks. Dependencies: `casehub-blocks-summarisation-api`, `cloudevents-api`, `quarkus-arc`.

Two adapters (plain Java utilities, not CDI beans — instantiated by domain wiring code):

**CloudEventIngestionAdapter** — Catches CloudEvents by configurable type pattern, extracts typed payload via Jackson, wraps as `LevelEvent<T>` with tenancyId, feeds into an `EventStreamBus<T>`.

```java
public class CloudEventIngestionAdapter<T> {
    private final Set<String> acceptedTypes;
    private final Class<T> payloadType;
    private final EventStreamBus<T> outputBus;
    private final EventLevel level;

    public void onCloudEvent(CloudEvent event) {
        if (!acceptedTypes.contains(event.getType())) return;
        String tenancyId = (String) event.getExtension("tenancyid");
        T payload = deserialize(event, payloadType);
        outputBus.publish(new LevelEvent<>(payload, event.getTime().toEpochMilli(), level, tenancyId));
    }
}
```

**CloudEventEmitter** — Subscribes to an output `EventStreamBus<T>`, wraps summarised events as CloudEvents with configurable type URI and source, fires via CDI `Event<CloudEvent>.fireAsync()`. Reads `tenancyid` from each output `LevelEvent`.

```java
public class CloudEventEmitter<T> {
    private final String outputType;
    private final URI source;
    private final Event<CloudEvent> cloudEventBus;

    // Subscribes to EventStreamBus, wraps payloads as CloudEvents,
    // reads tenancyId from LevelEvent.tenancyId(), sets extension, fires async
}
```

**Tenancyid propagation — tenant-aware pipeline:** TenancyId flows through the entire pipeline via `LevelEvent`:

1. `LevelEvent<E>` gains a `@Nullable String tenancyId` component: `record LevelEvent<E>(E payload, long timestamp, EventLevel level, @Nullable String tenancyId)`. Null is valid for single-tenant / test scenarios.
2. `EventAccumulator<E>` partitions internally by tenancyId. Events with different tenancyIds are never batched together. `drainIfReady(now)` checks each tenant partition independently against the `WindowPolicy`. The external API is unchanged — `collect(LevelEvent)`, `drainIfReady(now)`.
3. `KeyedAccumulator<K,E>` uses composite key `(tenancyId, K)` internally. Domain key extraction remains `Function<IN,K>` — tenancyId partitioning is transparent.
4. `SummarisationRunner` receives tenant-homogeneous batches from the accumulator and propagates the batch's tenancyId to each output `LevelEvent`.
5. `CloudEventEmitter` reads `tenancyId` from each output `LevelEvent` and sets the `tenancyid` extension attribute on the outgoing CloudEvent.

This design is consistent with how RAS handles multi-tenancy: `SituationEvaluator` is `@ApplicationScoped` (one instance), handles ALL tenants per-event, and keys `SituationContext` by `(situationId, correlationKey, tenancyId)`. The summarisation pipeline must provide the same per-event tenant isolation — a single pipeline instance handles all tenants, with tenant partitioning inside the accumulators.

**Breaking change:** `LevelEvent` gains a 4th component. Existing callers using `new LevelEvent<>(payload, timestamp, level)` must add a tenancyId argument. This is mechanical and intentional — every event source must be explicit about tenant identity.

**Tick scheduling:** Every `SummarisationRunner` and `KeyedSummarisationRunner` requires periodic `tick(now)` calls to drain age-based windows and stale groups. The bridge module provides a `PipelineTickScheduler` that calls `tick()` on all runners in the pipeline at a configurable interval. Default interval: the smallest `window.maxAge` or `keyed.staleTimeout` across all pipeline levels, divided by 2 (Nyquist — ensures no window can expire unnoticed for more than half its max age). For YAML pipelines, the `PipelineCompiler` auto-generates a `@Scheduled` CDI bean that drives the tick scheduler. For explicit Java wiring, the domain's CDI setup must create and start the scheduler.

**Explicit wiring:** Domains construct their own `SummarisationRunner` chains in a `@Produces` method, connecting ingestion adapter → runner(s) → emitter. The wiring method receives `Event<CloudEvent>` via CDI injection and passes it to the `CloudEventEmitter` constructor. ~30 lines of CDI setup per domain. No auto-discovery.

### 3.3 YAML Surface — `casehub-blocks-summarisation-yaml`

New module in casehub-blocks. Dependencies: `casehub-blocks-summarisation-api`, `casehub-blocks-cloudevents`, `casehub-platform-expression` (MVEL3 + JQ), Jackson YAML.

#### 3.3.1 Composable Runtime Model

**Tier 1 — YAML standalone:** Built-in summariser types ship in this module. A pipeline can be declared entirely in YAML with no Java required.

**Tier 2 — YAML + Java:** Custom `@SummariserTypeId` classes on the classpath extend the available types. YAML references them by type string. The `SummariserRegistry` discovers annotated classes at build time via Jandex (same pattern as `NodeSpecRegistry` in desiredstate).

#### 3.3.2 YAML Model

```yaml
pipeline:
  namespace: logistics
  name: hub-monitoring

  sources:
    - type: io.casehub.desiredstate.node.faulted
    - type: io.casehub.desiredstate.node.drifted
    - type: io.casehub.desiredstate.node.recovered

  levels:
    anomaly:
      window:
        maxAge: 30000
        maxCount: 10
      summariser:
        type: threshold-classify
        rules:
          - when: "${data.detail contains 'timeout'}"
            classify: DELAY
          - when: "${data.detail contains 'misroute'}"
            classify: MISROUTE
          - default: CAPACITY_BREACH
      output:
        cloudEvent: io.casehub.logistics.anomaly

    phase:
      window:
        maxAge: 60000
        maxCount: 5
      summariser:
        type: phase-detect
        states: [NORMAL, CONGESTION, RECOVERY]
        transitions:
          - from: NORMAL
            to: CONGESTION
            when: "count(DELAY) + count(CAPACITY_BREACH) >= 3"
          - from: CONGESTION
            to: RECOVERY
            when: "count(DELAY) == 0 && count(CAPACITY_BREACH) == 0"
      output:
        cloudEvent: io.casehub.logistics.phase
```

**Keyed grouping mode:** Levels may use `keyed` instead of `window` to group events by a key expression and drain each group independently:

```yaml
levels:
  per-region:
    keyed:
      keyExpression: "${data.region}"          # MVEL3 — extracts group key from payload
      completionTest: "size() >= 10"           # MVEL3 — predicate over group's event list
      staleTimeout: 120000                     # ms — drain group if no new events arrive
    summariser:
      type: threshold-classify
      rules:
        - when: "${data.severity == 'CRITICAL'}"
          classify: CRITICAL_FAULT
        - default: MINOR_FAULT
    output:
      cloudEvent: io.casehub.ops.region-anomaly
```

`keyed` compiles to `KeyedSummarisationRunner` (key extraction → per-group accumulation → completion/stale drain). `window` compiles to `SummarisationRunner` (flat windowed batching). A level specifies exactly one of `window` or `keyed`.

**Topology:** The YAML model defines a linear chain: `sources → level1 → level2 → ... → levelN`. Non-linear topologies (fanout, diamond, merge) require Tier 2 Java wiring via explicit `EventStreamBus` subscription. This is a deliberate Tier 1 simplification — linear chains (L1→L2→L3) are the most common pattern. The Java wiring API (`LogisticsPipelineTest` demonstrates arbitrary bus topologies) remains available for complex cases.

#### 3.3.3 Built-in Summariser Types

| Type | Purpose | Expression language | Output cardinality | Output schema (`data` payload) |
|------|---------|-------------------|--------------------|-------------------------------|
| `threshold-classify` | Classify events by field-matching rules into named categories | MVEL3 (boolean predicates) | 1 output per input event | `{ "classification": String, "matchedRule": int }` |
| `phase-detect` | State machine over classified events — detect phase transitions | MVEL3 (transition predicates) | 0 or 1 per batch (transition-only) | `{ "phase": String, "previousPhase": String, "transitionTime": long, "triggerCounts": Map<String,int> }` |
| `count` | Count events per category within window | None (structural) | 1 per batch | `{ "counts": Map<String,int>, "total": int, "windowStart": long, "windowEnd": long }` |
| `field-extract` | Extract/reshape fields from CloudEvent data payload | JQ (document transformation) | 1 output per input event | Schema defined by the JQ expression — user-controlled |
| `pass-through` | Identity — rebatch without transformation | None | 1 output per input event | Original payload, re-wrapped |

**Output cardinality matters for downstream `count()` in `phase-detect`:** `threshold-classify` emits one classified event per input event. A batch of 10 input events produces 10 output `LevelEvent`s. When `phase-detect` receives these 10 events and evaluates `count(DELAY)`, it counts how many of the 10 have `classification == "DELAY"`.

These schemas are the contract between summarisation output and downstream consumers (RAS Ganglia, other pipeline levels). Expression rules in RAS situation definitions reference these field paths (e.g., `${data.phase == 'CONGESTION'}`).

**`phase-detect` semantics:**
- **Initial state:** The first element in the `states` list is the initial state (e.g., `states: [NORMAL, CONGESTION, RECOVERY]` → starts in `NORMAL`).
- **Emit semantics:** `phase-detect` emits only on phase TRANSITIONS — when the evaluated batch causes the current phase to change. If the system stays in CONGESTION across multiple batches, no output events are emitted. The `previousPhase` field records the state before the transition; for the first transition, `previousPhase` is the initial state.
- **State persistence:** `phase-detect` is stateful — it tracks the current phase across batches in memory. On pipeline restart (redeployment, crash), the state resets to the initial state. This means a transition that occurred before the restart will not re-fire after restart. This is a known limitation — RAS provides durable detection state via `SituationContext` + `SituationStore` with `storeVersion`-based optimistic locking. The summarisation pipeline produces event signals; RAS owns durable situation tracking. If durable phase state is needed, it should be a follow-on enhancement (file as a child issue).

Expression language selection follows the RAS YAML situation system precedent: `threshold-classify` and `phase-detect` use MVEL3 (natural for boolean predicates); `field-extract` uses JQ (natural for document transformation). Both are available via `casehub-platform-expression`'s `CompiledExpression<CTX, RESULT>` interface (`CompiledExpression` is defined in `casehub-platform-api`; engine implementations are in `casehub-platform-expression`).

**Expression contexts by position:**

| Position | YAML example | Context object | Available fields/functions | Return type |
|----------|-------------|----------------|---------------------------|-------------|
| `threshold-classify` rule `when` | `"${data.detail contains 'timeout'}"` | `Map<String, Object>` mirroring `CloudEventExpressionContext`: `data` = payload, `timestamp`, `level`, `tenancyId` | All payload fields under `data.*`; event metadata at top level | `Boolean` |
| `keyed.keyExpression` | `"${data.region}"` | Same as `threshold-classify` `when` — evaluated per event | Same as above | `Object` (the group key) |
| `keyed.completionTest` | `"size() >= 10"` | `List<LevelEvent<IN>>` — the group's accumulated events | `size()` (list size); list-level operations | `Boolean` |
| `phase-detect` transition `when` | `"count(DELAY) >= 3"` | Summariser-specific aggregate context over the classified batch | `count(CATEGORY)` — counts events where `classification == CATEGORY` | `Boolean` |

The `threshold-classify` and `keyed.keyExpression` contexts use the same map structure as `CloudEventExpressionContext.build()` in RAS, with the `LevelEvent` payload placed under the `data` key. This ensures expressions are portable between summarisation YAML and RAS situation definitions.

#### 3.3.4 Standard CloudEvent Type URIs (D9)

Built-in summariser output uses standardised type URIs. The default pattern uses the level's ordinal (not its domain-specific name) for cross-domain consistency:
```
io.casehub.blocks.summarisation.L<ordinal>.<builtin-type>
```

Example: A level at ordinal 2 using `threshold-classify` → `io.casehub.blocks.summarisation.L2.threshold-classify`

The YAML `output.cloudEvent` field overrides this default — **in practice, domain pipelines should always specify an override** with a domain-specific URI (e.g., `io.casehub.logistics.anomaly`). The default pattern exists as a fallback for pipelines that don't specify output URIs, ensuring every emitted CloudEvent has a deterministic type.

Custom domain summarisers (Tier 2) always use domain-specific URIs.

#### 3.3.5 `@SummariserTypeId` Annotation

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface SummariserTypeId {
    String value();  // maps to YAML "type" field
}
```

Custom Java summarisers:
```java
@SummariserTypeId("ml-anomaly-detector")
public class MlAnomalyDetector implements Summariser<CloudEventPayload, Anomaly> {
    // Complex ML-based classification — can't be expressed in YAML
}
```

Referenced in YAML identically to built-in types:
```yaml
summariser:
  type: ml-anomaly-detector
  config:
    modelPath: /models/anomaly-v2.onnx
```

### 3.4 Build-time Processing

Quarkus build extension (`casehub-blocks-summarisation-yaml-deployment`):

1. Classpath scan for `META-INF/summarisation/*.yaml` pipeline definitions
2. `@SummariserTypeId` registry scan via Jandex
3. Build-time validation:
   - **Syntactic:** unknown summariser types, invalid window/keyed policies, expression syntax, level ordering
   - **Semantic — state reachability (`phase-detect`):** all declared states are reachable from the initial state; no dead states with no incoming transitions; unreachable states are build errors
   - **Semantic — cross-level consistency:** classification names used in downstream `phase-detect` transition predicates (e.g., `count(DELAY)`) must match classifications produced by the upstream `threshold-classify` level; mismatches are build errors
   - **Semantic — loop prevention:** no ingestion adapter's `acceptedTypes` (from `sources`) may overlap with any emitter's `outputType` (from `output.cloudEvent`) across all pipelines in the deployment; overlaps are build errors. This prevents CloudEvent re-entry loops where summarised output re-triggers the same pipeline via the shared CDI bus
4. CDI bean registration: `PipelineCompiler` produces wired `SummarisationRunner`/`KeyedSummarisationRunner` chains from YAML at `RUNTIME_INIT`, plus a `@Scheduled` tick driver bean per pipeline (interval derived from the smallest `window.maxAge` or `keyed.staleTimeout` / 2)

---

## 4. Ganglia — Tiered Detection (D4)

### 4.1 Expression-based Default

Built-in summariser types produce standardised CloudEvent output schemas with known field paths. ExpressionRulesGanglion (already in `casehub-ras-runtime`) evaluates boolean expressions over `CloudEventExpressionContext` — a `Map<String, Object>` with keys: `type`, `source`, `subject`, `id`, `time`, `tenancyid`, `data`.

For standardised output, expression rules can match field paths without domain-specific Java:

```yaml
# RAS situation definition (existing YAML format)
situations:
  hub-congestion:
    events: [io.casehub.logistics.phase]
    ganglion:
      type: expression-rules
      rules:
        - when: "${data.phase == 'CONGESTION'}"
          signal: DETECTED
        - when: "${data.phase == 'NORMAL'}"
          signal: ANTI
```

### 4.2 Domain-specific Escape Hatch

Complex multi-signal correlation (e.g., congestion + capacity + route failure = systemic breakdown) may exceed expression language capabilities. Domain-specific Java Ganglia extending `JavaSwitchGanglion` remain available:

```java
@ApplicationScoped
public class SystemicBreakdownGanglion extends JavaSwitchGanglion {
    // Multi-signal correlation logic
}
```

### 4.3 Additional RAS Ganglion Types

The RAS YAML situation system provides three ganglion types:
- `expression-rules` — boolean rule matching (default for this design)
- `naive-bayes` — probabilistic detection (natural for noisy summarised streams)
- `situation-watcher` — situation-on-situation composition (meta-detection)

Chain modes (`and`, `or`, `threshold`, `sequence`, `count`, `streak`, `rate`) compose Ganglion output into complex situation definitions declaratively.

---

## 5. Logistics Example — blocks `examples/logistics/`

Pure blocks + RAS demonstration. No desiredstate dependency.

### 5.1 Domain Model

| Type | Level | Purpose |
|------|-------|---------|
| Raw CloudEvent | L1 | Simulated package scan events (delay, misroute, capacity breach) |
| `Anomaly` | L2 | Classified anomaly (DELAY, MISROUTE, CAPACITY_BREACH) |
| `HubPhase` | L3 | Hub operational phase (NORMAL, CONGESTION, RECOVERY) |

### 5.2 YAML-first

The example aims to be maximally YAML-driven:
- **Pipeline definition** — YAML with built-in `threshold-classify` and `phase-detect` summarisers
- **RAS situation definitions** — YAML with `expression-rules` Ganglia
- **Java escape hatches** — only for: (a) event simulation/generation, (b) test assertions

The example validates Tier 1 (YAML standalone) for per-event classification and state-machine phase detection — the two most common summarisation patterns. The built-in `threshold-classify` evaluates per-event boolean predicates; it does not perform cross-event correlation (e.g., grouping events by destination and counting distinct warehouses). Cross-event correlation is inherently Tier 2 — the expressiveness of Java is the right tool for batch-level aggregation logic.

The existing Java logistics test (`AnomalyDetectorSummariser`) demonstrates exactly this Tier 2 pattern: its misroute detection groups events by destination and counts distinct warehouses, which is batch-level aggregation that cannot and should not be expressed as YAML predicates. The YAML example uses per-event classification rules (field-matching) to classify anomalies, then feeds those classifications into `phase-detect` for state-machine transitions. This is a genuinely non-trivial multi-level pipeline, validating that Tier 1 covers the common case without Java.

If a domain requires cross-event correlation at the classification level, it provides a custom `@SummariserTypeId` Java class (Tier 2) and references it from YAML identically to built-in types.

### 5.3 Dependencies

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-blocks-summarisation-yaml</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ras-api</artifactId>
</dependency>
```

---

## 6. Ops Enhancement — Separate Issue (casehub-ops)

The genuine desiredstate + summarisation use case: adding summarisation between ReconciliationLoop CloudEvents and existing RAS Ganglia in ops deployment topologies. This gives RAS altitude — detecting "east-region entered degraded phase" instead of counting individual `NODE_FAULTED` events.

**Not in scope for this design.** File as a child issue on casehub-ops referencing this spec. The ops enhancement consumes the same bridge module (§3.2) and can use the YAML surface (§3.3) for its summarisation pipeline.

---

## 7. Deliverables and Issue Map

| Deliverable | Repo | Issue |
|-------------|------|-------|
| Summarisation API extraction | casehub-blocks | Child issue to file |
| CloudEvent bridge module | casehub-blocks | Child issue to file |
| Summarisation YAML surface + built-in types | casehub-blocks | Child issue to file |
| Logistics example (YAML-first) | casehub-blocks | Child issue to file |
| Ops deployment + summarisation enhancement | casehub-ops | Child issue to file |
| Close #74 with scope decision | casehub-desiredstate | This issue |

---

## 8. What Does NOT Change

- **casehub-desiredstate** — no code changes. Existing CloudEvent emission from ReconciliationLoop and ras-adapter Ganglia are sufficient.
- **casehub-ras** — no code changes. ExpressionRulesGanglion and YAML situation system already support the detection patterns needed.
- **casehub-platform** — no changes. CloudEvent and expression engine infrastructure already exists.

---

## References

- `DesiredStateEventTypes.java` — L1 CloudEvent type URI definitions
- `NodeFaultGanglion.java`, `PersistentDriftGanglion.java` — existing L1 Ganglia pattern
- `DesiredStateSituationDefinitionProvider.java` — situation definition registration pattern
- `blocks/summarisation/` — `Summariser`, `EventStreamBus`, `SummarisationRunner`, `LevelEvent`, `WindowPolicy`
- `yaml/model/YamlGraph.java`, `medallion-pipeline.yaml` — desiredstate YAML surface pattern
- `yaml/registry/NodeSpecRegistry.java` — type registry pattern for `@SummariserTypeId`
- GE-20260730-d761e5 — RasEngine requires `tenancyid` extension on CloudEvents
- GE-20260629-e8b16d — EventStreamBus.clear() lifecycle gotcha (argues for explicit wiring)
- GE-20260817-ce1de5 — CloudEventExpressionContext map structure for expression rules
- GE-20260616-02d0a7 — CaseHub entities are independently creatable (flat graph)
- `parent/docs/platform/capability-ownership.md` — "Temporal event summarisation" owned by casehub-blocks
- `parent/docs/platform/boundary-rules.md` — "Do not add domain logic to foundation repos"
- `parent/docs/platform/overview.md` — tier architecture and dependency order
