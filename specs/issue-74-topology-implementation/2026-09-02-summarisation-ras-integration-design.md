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
| `LevelEvent<E>` | `blocks/` | `summarisation-api/` |
| `EventStreamBus<E>` | `blocks/` | `summarisation-api/` |
| `WindowPolicy` | `blocks/` | `summarisation-api/` |
| `EventAccumulator<E>` | `blocks/` | `summarisation-api/` |
| `Compactor<E>` | `blocks/` | `summarisation-api/` |
| `EventLevel` | `blocks/` | `summarisation-api/` |
| `SummarisationRunner<IN,OUT>` | `blocks/` | `summarisation-api/` |

These types are already pure Java — no CDI annotations, no Quarkus dependencies. Extraction is mechanical. This prevents bridge consumers from transitively depending on blocks' full dependency tree (qhorus-api, work-api, engine-api, eidos-api).

### 3.2 Bridge Module — `casehub-blocks-cloudevents`

New module in casehub-blocks. Dependencies: `casehub-blocks-summarisation-api`, `cloudevents-api`, `quarkus-arc`.

Two adapters:

**CloudEventIngestionAdapter** — CDI observer that catches CloudEvents by configurable type pattern, extracts typed payload via Jackson, wraps as `LevelEvent<T>`, feeds into an `EventStreamBus<T>`.

```java
@ApplicationScoped
public class CloudEventIngestionAdapter<T> {
    private final Set<String> acceptedTypes;
    private final Class<T> payloadType;
    private final EventStreamBus<T> outputBus;
    private final EventLevel level;

    // Observes all CloudEvents, filters by type, extracts payload, publishes to bus
}
```

**CloudEventEmitter** — Subscribes to an output `EventStreamBus<T>`, wraps summarised events as CloudEvents with configurable type URI and source, fires via CDI `Event<CloudEvent>.fireAsync()`. Propagates `tenancyid` from input context.

```java
@ApplicationScoped
public class CloudEventEmitter<T> {
    private final String outputType;  // CloudEvent type URI
    private final URI source;
    private final Event<CloudEvent> cloudEventBus;

    // Subscribes to EventStreamBus, wraps payloads as CloudEvents, fires async
}
```

**Explicit wiring:** Domains construct their own `SummarisationRunner` chains, connecting ingestion adapter → runner(s) → emitter. ~30 lines of CDI setup per domain. No auto-discovery.

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

#### 3.3.3 Built-in Summariser Types

| Type | Purpose | Expression language |
|------|---------|-------------------|
| `threshold-classify` | Classify events by field-matching rules into named categories | MVEL3 (boolean predicates) |
| `phase-detect` | State machine over classified events — detect phase transitions | MVEL3 (transition predicates) |
| `count` | Count events per category within window | None (structural) |
| `field-extract` | Extract/reshape fields from CloudEvent data payload | JQ (document transformation) |
| `pass-through` | Identity — rebatch without transformation | None |

Expression language selection follows the RAS YAML situation system precedent: `threshold-classify` and `phase-detect` use MVEL3 (natural for boolean predicates); `field-extract` uses JQ (natural for document transformation). Both are available via `casehub-platform-expression`'s `CompiledExpression<CTX, RESULT>` interface.

#### 3.3.4 Standard CloudEvent Type URIs (D9)

Built-in summariser output uses standardised type URIs:
```
io.casehub.blocks.summarisation.<level>.<builtin-type>
```

Example: `io.casehub.blocks.summarisation.L2.threshold-classify`

Custom domain summarisers use domain-specific URIs (e.g., `io.casehub.logistics.phase.congestion`). The YAML `output.cloudEvent` field overrides the default URI — if specified, it takes precedence over the standard pattern.

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
3. Build-time validation: unknown summariser types, invalid window policies, expression syntax
4. CDI bean registration: `PipelineCompiler` that produces wired `SummarisationRunner` chains from YAML at `RUNTIME_INIT`

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
- **Java escape hatches** — only for: (a) event simulation/generation, (b) test assertions, (c) any classification logic that exceeds expression capabilities

The example validates whether Tier 1 (YAML standalone) is genuinely sufficient for a non-trivial pipeline. If it requires Java escape hatches beyond simulation and testing, that's a signal the built-in summariser types need extension.

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
- `capability-ownership.md` — "Temporal event summarisation" owned by casehub-blocks
- `boundary-rules.md` — "Do not add domain logic to foundation repos"
- `overview.md` — tier architecture and dependency order
