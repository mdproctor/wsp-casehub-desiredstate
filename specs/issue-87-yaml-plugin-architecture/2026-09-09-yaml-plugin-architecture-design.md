# YAML Plugin Architecture — Design Spec

**Issue:** casehubio/casehub-ops#87 — YAML-first plugin architecture
**Date:** 2026-09-09
**Status:** Draft
**Scope:** Plugin schema + interpreter (subsystem 1 of 5 from the issue)

## 1. Summary

A YAML-declared plugin system for casehub-desiredstate where each plugin is a
complete self-healing unit: spec schema, actual-state detection, provisioning,
fault escalation, CBR learning surface, and RAS situation definitions — all in
one YAML file. No Java required for new resource types.

Plugins are discovered at Quarkus build time, validated exhaustively against
the spec schema and primitive registry, and interpreted at runtime by generic
Java beans that implement the existing `NodeProvisioner` and `ActualStateAdapter`
SPIs. The plugin system composes with the existing YAML surface (#116) — graph
YAML declares *what* should exist, plugin YAML declares *how* to manage it.

## 2. Background

The desired-state runtime has proven SPIs across 4 domain examples (Dungeon,
Pipeline, Expansion, Spatial) and 4 ops modules (infra, deployment, iot,
compliance). Each new node type currently requires Java: a `NodeSpec` record,
a `NodeProvisioner`, an `ActualStateAdapter`, and fault policies
(GE-20260806-272a90: "6 components + 4 ripple updates"). The YAML language
extensions (#116) brought the graph *declaration* surface to "no Java required"
but left the *behavior* surface (provisioning, detection) in Java.

This spec closes that gap: an operator writes one YAML file per resource type,
and the runtime provisions, monitors, and self-heals it using the same
reconciliation loop, the same fault policies, and the same CBR/RAS integration
as Java-implemented types.

## 3. Design Principles

1. **Build-time validation, runtime interpretation.** All structural errors
   caught before deployment. Runtime interprets validated descriptors — the
   overhead is negligible compared to external API latency.

2. **Dual declaration.** Java NodeSpec records and YAML spec schemas coexist.
   A plugin author picks whichever fits. Mixed mode: Java spec + YAML
   provisioner, or YAML spec + Java provisioner.

3. **Composable primitives.** YAML over YAML over Java. Compound primitives
   compose other primitives hierarchically. Build-time expansion to flat
   step sequences.

4. **Credential-free plugins.** Auth references resolved at runtime via
   `CredentialResolver`. Plugin YAML never contains credentials.

5. **Reuse, not reinvent.** Fault policy reuses #116 syntax. Interpolation
   extends #116 namespace model. SPI mapping reuses existing router
   infrastructure.

## 4. Plugin File Structure

One file per node type at `META-INF/desiredstate/plugins/<type>.yaml`.
Discovered at build time via classpath scan. Plugins can ship in library JARs.

```yaml
plugin:
  type: k8s-deployment
  version: 1
  resyncInterval: 30s
  auth:
    k8s:
      credentialRef: k8s-cluster-credentials

spec:
  fields:
    namespace: { type: string, required: true }
    name: { type: string, required: true }
    image: { type: string, required: true }
    replicas: { type: integer, default: 1, min: 0, max: 100 }

actual-state:
  steps:
    - rest-call:
        method: GET
        url: "https://${auth.k8s.endpoint}/apis/apps/v1/namespaces/${spec.namespace}/deployments/${spec.name}"
        auth: k8s
        result: response
    - compare-state:
        present-when: "${result.response.status} == 200"
        drifted-when: "${result.response.body.status.availableReplicas} < ${spec.replicas}"
        absent-when: "${result.response.status} == 404"

provisioner:
  provision:
    steps:
      - rest-call:
          method: PUT
          url: "https://${auth.k8s.endpoint}/apis/apps/v1/namespaces/${spec.namespace}/deployments/${spec.name}"
          auth: k8s
          headers:
            Content-Type: application/json
          body:
            apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: "${spec.name}"
              namespace: "${spec.namespace}"
            spec:
              replicas: ${spec.replicas}
              template:
                spec:
                  containers:
                    - name: app
                      image: "${spec.image}"
          result: response
      - assert:
          condition: "${result.response.status} in [200, 201]"
          message: "Provision failed: HTTP ${result.response.status}"
  deprovision:
    steps:
      - rest-call:
          method: DELETE
          url: "https://${auth.k8s.endpoint}/apis/apps/v1/namespaces/${spec.namespace}/deployments/${spec.name}"
          auth: k8s
          result: response
      - assert:
          condition: "${result.response.status} in [200, 404]"
          message: "Deprovision failed: HTTP ${result.response.status}"

fault-policy:
  - faultTypes: [PROVISION_FAILED]
    nodeTypes: [k8s-deployment]
    namespace: k8s-deployment-escalation
    tiers:
      - threshold: 3
        reviewNode:
          type: human-review
          humanGating: ALL
          spec:
            target: "${fault.nodeId}"
            detail: "${fault.detail}"

cbr:
  features:
    - { name: replicas, source: spec.replicas, similarity: numeric }
    - { name: image-family, source: spec.image, similarity: categorical,
        transform: "regex:^([^:]+)" }
  outcome-signals:
    success: "${result.response.body.status.availableReplicas} >= ${spec.replicas}"

**Feature transforms** (optional): pre-process the source value before
similarity comparison. Built-in transforms:

| Transform | Semantics | Example |
|-----------|-----------|---------|
| `regex:<pattern>` | Extract first capture group | `regex:^([^:]+)` extracts image name from `nginx:1.25` → `nginx` |
| `lowercase` | Case-insensitive comparison | `lowercase` normalizes `PostgreSQL` → `postgresql` |
| `hash` | SHA-256 hash (for high-cardinality values) | `hash` reduces unique URIs to fixed-length tokens |

Custom transforms fall back to Java (`FeatureTransform` SPI — future).

ras:
  situations:
    - name: crash-loop-backoff
      events: [NODE_FAULTED, NODE_RECOVERED]
      correlation-window: 10m
      chain-mode:
        streak: 3
      trigger: create-case
      trigger-mode: fire-once
      correlation-key: "${spec.namespace}"
```

**Required sections:** `plugin`, `spec`, `actual-state`, `provisioner`, `cbr`, `ras`.
**Optional sections:** `fault-policy`.

**Hooks:** Lifecycle hooks (Verify, Notify, Wait) are a graph-level concern,
declared per node in graph YAML via the existing `provision:`/`deprovision:`
hook syntax from #116. Plugin YAML does not declare default hooks — the plugin
defines *how* to provision, while the graph node defines *what happens around*
provisioning. The two concerns are independent.

## 5. YAML Spec Schema

The `spec:` section defines the node type's data shape, replacing Java
`NodeSpec` records for types that don't need Java.

### 5.1 Field Types

| Type | Java equivalent | Validation constraints |
|------|----------------|----------------------|
| `string` | `String` | `required`, `default`, `pattern`, `minLength`, `maxLength` |
| `integer` | `int` / `long` | `required`, `default`, `min`, `max` |
| `number` | `double` | `required`, `default`, `min`, `max` |
| `boolean` | `boolean` | `required`, `default` |
| `enum` | `enum` | `required`, `default`, `values: [A, B, C]` |
| `list` | `List<T>` | `required`, `default`, `item-type`, `minItems`, `maxItems` |
| `map` | `Map<String, T>` | `required`, `default`, `value-type` |

### 5.2 Runtime Representation — YamlNodeSpec

YAML-declared types produce a `YamlNodeSpec` adapter implementing `NodeSpec`:

```java
public final class YamlNodeSpec implements NodeSpec {
    private final NodeType nodeType;
    private final HumanGating humanGating;
    private final Map<String, Object> fields;

    @Override public NodeType nodeType() { return nodeType; }
    @Override public HumanGating humanGating() { return humanGating; }
    public Object get(String fieldName) { return fields.get(fieldName); }
    public Map<String, Object> fields() { return fields; }
}
```

`${spec.*}` interpolation resolves via `YamlNodeSpec.get(fieldName)` for
YAML-declared types. For Java NodeSpec records, `${spec.*}` resolves via
record component accessors (reflective, cached at build time).

**Supersedes #117 D10:** The YAML surface foundation (#117) required Java
NodeSpec classes on classpath for every type (D10: "operators cannot define
new node types purely in YAML"). This spec explicitly supersedes that
constraint — `YamlNodeSpec` enables new node types without Java. The
constraint was appropriate for #117's scope (graph declaration only); #87
extends the surface to include behavior (provisioning, detection), making
Java-free types both possible and desirable.

### 5.3 NodeSpecRegistry Extension

`NodeSpecRegistry` gains a factory resolution path alongside the existing
class-based path:

```java
public class NodeSpecRegistry {
    // Existing: class-based resolution for Java @NodeTypeId types
    public Class<? extends NodeSpec> resolve(String typeName);

    // New: factory-based resolution for YAML plugin types
    public Optional<NodeSpecFactory> resolveFactory(String typeName);

    // New: check which path a type uses
    public boolean isFactoryType(String typeName);
}
```

| Source | Resolution |
|--------|-----------|
| Java `@NodeTypeId` | `resolve(typeName)` → `Class<? extends NodeSpec>` → Jackson `convertValue` (existing) |
| YAML plugin | `resolveFactory(typeName)` → `NodeSpecFactory.create(specMap)` → `YamlNodeSpec` with schema validation |

Callers (e.g., `YamlGraphRecorder`) check `resolveFactory()` first. If present,
the factory validates the raw spec map against the plugin's field definitions and
produces a `YamlNodeSpec`. If absent, falls back to `resolve()` for class-based
Jackson deserialization. The `NodeSpecFactory` SPI (`create(Map<String, Object>) →
NodeSpec`) already exists — each YAML plugin registers a factory via
`NodeSpecFactoryProvider` at build time.

### 5.4 Dual Declaration

A type can use either surface. Mixed mode examples:

- **Full YAML:** YAML spec schema + YAML provisioner (no Java)
- **Full Java:** Java NodeSpec + Java provisioner (current pattern, unchanged)
- **Java spec + YAML provisioner:** Java defines complex data shape with
  validation logic; YAML defines REST-based provisioning
- **YAML spec + Java provisioner:** YAML defines simple data shape; Java
  handles complex provisioning (e.g., multi-step transactions)

Build-time conflict detection: if both a Java `@NodeTypeId` and a YAML
plugin declare the same type string, the build fails with a clear error.

## 6. Step Pipeline

### 6.1 Step Contract

Each step invokes a named primitive with parameters and produces a result:

```yaml
steps:
  - <primitive-name>:
      <param1>: <value>
      <param2>: <value>
      result: <binding-name>
```

A step receives:
- Primitive name (resolved from the primitive registry)
- Parameters (YAML map, primitive-specific)
- `StepContext`: accumulated bindings (spec fields, auth, prior results)

A step produces:
- A `StepResult` value bound to the `result:` name
- Available to subsequent steps via `${result.<name>.*}`

### 6.2 Step Control Flow

Minimal control flow at the step level:

| Directive | Semantics |
|-----------|-----------|
| `when:` | Conditional execution (reuses #116 truthy/falsy vocabulary) |
| `on-error: retry` | Retry the step. Optional: `max-retries` (default 3), `backoff` (default `fixed:1s`, also `exponential:1s`) |
| `on-error: fail` | Fail the pipeline (default) |
| `on-error: skip` | Skip the step, continue pipeline |

No loops — use `forEach:` at the graph level for iteration. No parallel
steps — per-node provisioning is sequential by design (D4).

### 6.3 Condition Expression Vocabulary

Conditions in `compare-state`, `assert`, and `when:` use a simple operator
vocabulary — not a full expression language. Consistent with the #116
principle: "YAML is data, not code."

| Operator | Example | Semantics |
|----------|---------|-----------|
| `==` | `${result.r.status} == 200` | Equality (string or numeric) |
| `!=` | `${result.r.status} != 404` | Inequality |
| `<`, `>`, `<=`, `>=` | `${spec.replicas} > 0` | Numeric comparison |
| `in` | `${result.r.status} in [200, 201]` | Set membership |
| `contains` | `${result.r.body.message} contains "Ready"` | Substring match |
| `and`, `or` | `... == 200 and ... > 0` | Boolean combinators |
| `not` | `not ${result.r.status} == 404` | Negation |

Interpolation resolves `${}` references to typed values (string, number,
boolean), then the expression evaluator operates on those typed operands.
Interpolated values are atomic tokens — they cannot inject operators or
sub-expressions. `${result.response.body.message}` resolving to
`"Ready or true"` is treated as a single string value, not three tokens.
The expression `${result.response.body.message} contains "Ready"` evaluates
as `<string:"Ready or true"> contains <string:"Ready">` → `true`.

Build-time validates expression syntax and that all interpolation references
resolve. Complex logic that exceeds this vocabulary falls back to a Java
`StepPrimitive`.

### 6.4 StepResult Structure

The `rest-call` primitive produces:

```
result:
  status: 200              # HTTP status code
  headers: { ... }         # Response headers
  body: { ... }            # Parsed JSON body (or raw string)
```

Accessible via `${result.response.status}`, `${result.response.body.field}`.
Deep path traversal follows Jackson property path conventions.

## 7. Primitive Registry

### 7.1 Java Primitives (Leaf Nodes)

CDI beans implementing `StepPrimitive`:

```java
public interface StepPrimitive {
    String name();
    StepResult execute(StepParameters params, StepContext context);
}
```

**Built-in Java primitives shipped in desiredstate:**

| Primitive | Purpose | Key parameters |
|-----------|---------|---------------|
| `rest-call` | HTTP REST call | `method`, `url`, `auth`, `headers`, `body`, `result` |
| `graphql-call` | GraphQL query/mutation | `url`, `auth`, `query`, `variables`, `result` |
| `json-extract` | Extract values from JSON | `input`, `path` (JSONPath), `result` |
| `compare-state` | Map response to NodeStatus | `present-when`, `absent-when`, `drifted-when` |
| `assert` | Fail pipeline on condition | `condition`, `message` |
| `approval-gate` | Conditional PendingApproval | `when`, `plan-reference` |

Retry is handled via `on-error: retry` on individual steps (§6.2), not as
a separate primitive. This avoids two retry mechanisms with different semantics.

**`compare-state` evaluation precedence:** Conditions are evaluated in fixed
order: `absent-when` → `drifted-when` → `present-when`. The first condition
that evaluates to `true` determines the `NodeStatus`. If no condition matches,
the result is `NodeStatus.UNKNOWN`. This ordering ensures drift is detected
even when the resource technically exists (HTTP 200 with insufficient replicas
→ `DRIFTED`, not `PRESENT`).

**`approval-gate`** returns `ProvisionResult.PendingApproval` when the `when:`
condition evaluates to `true` and no prior approval exists in the
`ProvisionContext`. On re-entry with `context.hasApproval()`, the gate is
skipped and the pipeline continues. The `plan-reference:` expression provides
the opaque string round-tripped through the approval lifecycle. Example:

```yaml
provisioner:
  provision:
    steps:
      - approval-gate:
          when: "${spec.replicas} > 10"
          plan-reference: "scale-${spec.name}-to-${spec.replicas}"
      - rest-call:
          # ... proceeds only after approval or if gate condition is false
```

### 7.2 YAML Compound Primitives

Files at `META-INF/desiredstate/primitives/<name>.yaml`:

```yaml
primitive:
  name: k8s-api-call
  parameters:
    method: { type: string, required: true }
    path: { type: string, required: true }
    body: { type: object }
  steps:
    - rest-call:
        method: "${param.method}"
        url: "https://${auth.k8s.endpoint}${param.path}"
        auth: k8s
        headers:
          Content-Type: application/json
        body: "${param.body}"
        result: response
  result: response
```

Usage in a plugin:
```yaml
actual-state:
  steps:
    - k8s-api-call:
        method: GET
        path: "/apis/apps/v1/namespaces/${spec.namespace}/deployments/${spec.name}"
        result: deployment
```

### 7.3 Composition Rules

- Java primitives are leaf nodes — execute directly
- YAML primitives expand at build time to step sequence templates
- Expansion is macro-style: the compound primitive's steps are inlined at
  each invocation site with parameter bindings
- `${param.*}` scoping: innermost-wins. Primitive A invoking compound B
  invoking leaf C — C's `${param.*}` resolves against B's parameters.
  A's parameters are not visible to C unless B explicitly passes them.
- Cycle detection at build time: A → B → A is a build error
- Max nesting depth: 5 (configurable via
  `casehub.desiredstate.plugin.primitive.max-depth`)
- Build-time validates all referenced primitives exist in the registry

### 7.4 Primitive Discovery

At Quarkus build time:

1. Jandex scan for `StepPrimitive` implementations → Java primitive registry
2. Classpath scan `META-INF/desiredstate/primitives/*.yaml` → YAML primitives
3. Validate: no name collisions between Java and YAML primitives
4. Expand YAML primitives (resolve composition, detect cycles)
5. Resulting flat registry: `Map<String, PrimitiveDescriptor>` where each
   descriptor is either a `JavaPrimitive` (CDI bean ref) or
   `ExpandedCompoundPrimitive` (step sequence template)

## 8. Interpolation Model

### 8.1 Namespace Table

Extending #116's prefix-based dispatch architecture:

| Prefix | Scope | Available in | Resolved by |
|--------|-------|-------------|-------------|
| `${spec.*}` | Node spec fields | All plugin sections | `YamlNodeSpec.get()` or record accessor |
| `${auth.<name>.*}` | Credential properties | Steps with `auth:` | `CredentialResolver.resolve(credentialRef)` |
| `${result.<name>.*}` | Prior step output | Subsequent steps | Step context accumulator |
| `${param.*}` | Compound primitive parameters | Inside YAML primitives | Parameter binding map |
| `${var.*}` | Variables (from #116) | Everywhere | `VariableResolver` |
| `${fault.*}` | Fault context (from #116) | fault-policy section | `FaultEvent` properties |

### 8.2 Auth Resolution

Plugin YAML declares auth stanzas at the plugin level:

```yaml
plugin:
  type: k8s-deployment
  version: 1
  auth:
    k8s:
      credentialRef: k8s-cluster-credentials
```

Steps reference by name: `auth: k8s`. At runtime:
1. Resolve `credentialRef` via `CredentialResolver.resolve("k8s-cluster-credentials")`
2. Returns `Map<String, String>` credential properties
3. Available as `${auth.k8s.<key>}` (e.g., `${auth.k8s.endpoint}`, `${auth.k8s.token}`)

### 8.3 Spec Field Resolution

For YAML-declared types: `${spec.namespace}` → `YamlNodeSpec.get("namespace")`.
For Java NodeSpec records: `${spec.replicas}` → reflective record component
accessor (cached at build time for performance).

Build-time validates all `${spec.*}` references against the spec schema (YAML)
or record component names (Java).

## 9. SPI Mapping

### 9.1 Generic Provisioner

One `YamlPluginProvisioner` bean implements `NodeProvisioner`:

```java
@ApplicationScoped
public class YamlPluginProvisioner implements NodeProvisioner {
    private final Map<NodeType, PluginDescriptor> plugins;
    private final StepPipelineExecutor executor;

    @Override
    public Set<NodeType> handledTypes() { return plugins.keySet(); }

    @Override
    public ProvisionResult provision(DesiredNode node, ProvisionContext ctx) {
        PluginDescriptor plugin = plugins.get(node.type());
        StepContext context = buildContext(node, ctx, plugin);
        return executor.execute(plugin.provisionSteps(), context);
    }

    @Override
    public DeprovisionResult deprovision(DesiredNode node, DeprovisionContext ctx) {
        PluginDescriptor plugin = plugins.get(node.type());
        StepContext context = buildContext(node, ctx, plugin);
        return executor.execute(plugin.deprovisionSteps(), context);
    }

    @Override
    public Duration resyncIntervalFor(NodeType type) {
        PluginDescriptor plugin = plugins.get(type);
        return plugin != null ? plugin.resyncInterval() : Duration.ofMinutes(5);
    }
}
```

**Approval support:** When the step pipeline encounters an `approval-gate`
step whose condition is met, `StepPipelineExecutor.execute()` returns
`ProvisionResult.PendingApproval(nodeId, planReference)`. The `buildContext()`
method passes `ProvisionContext.approval()` into the `StepContext`, making it
available to `approval-gate` steps via `context.hasApproval()`.

**Per-type resync interval:** `YamlPluginProvisioner` overrides a new default
method on `NodeProvisioner`:

```java
// New default method on NodeProvisioner
default Duration resyncIntervalFor(NodeType type) {
    return resyncInterval();
}
```

`DefaultNodeProvisionerRouter.resyncIntervalFor()` calls
`p.resyncIntervalFor(type)` instead of `p.resyncInterval()`, enabling
per-type intervals. `YamlPluginProvisioner.resyncIntervalFor()` returns
the interval declared in each plugin's `resyncInterval:` field, falling
back to 5 minutes if unspecified.

**Result mapping:** `StepPipelineExecutor.execute()` maps pipeline outcomes
to `ProvisionResult` / `DeprovisionResult`:

| Pipeline outcome | Result |
|-----------------|--------|
| All steps succeed | `Success` |
| `approval-gate` triggers (no approval) | `PendingApproval(nodeId, planReference)` |
| Any step fails (`assert`, `on-error: fail`) | `Failed(reason)` — reason from the failing step |
| Step throws exception | `Failed(exception.getMessage())` |

### 9.2 Generic Actual State Adapter

One `YamlPluginActualStateAdapter` bean implements `ActualStateAdapter`:

```java
@ApplicationScoped
public class YamlPluginActualStateAdapter implements ActualStateAdapter {
    private final Map<NodeType, PluginDescriptor> plugins;
    private final StepPipelineExecutor executor;

    @Override
    public Set<NodeType> handledTypes() { return plugins.keySet(); }

    @Override
    public ActualState readActual(DesiredStateGraph graph, String tenancyId) {
        Map<NodeId, NodeStatus> states = new HashMap<>();
        for (DesiredNode node : graph.nodes().values()) {
            if (plugins.containsKey(node.type())) {
                PluginDescriptor plugin = plugins.get(node.type());
                StepContext context = buildContext(node, tenancyId, plugin);
                NodeStatus status = executor.executeActualState(
                    plugin.actualStateSteps(), context);
                states.put(node.id(), status);
            }
        }
        return ActualState.of(states);
    }
}
```

**Result mapping:** `StepPipelineExecutor.executeActualState()` extracts
`NodeStatus` from the `compare-state` step's result. If the pipeline fails
before reaching `compare-state`, or if no `compare-state` step exists, the
result is `NodeStatus.UNKNOWN`. Build-time validation enforces that every
`actual-state` pipeline contains exactly one `compare-state` step (§10 step 6a).

### 9.3 Fault Policy Registration

Reuses #116's `YamlGraphRecorder.createFaultPolicy()` path. Each plugin's
`fault-policy:` section produces `ThresholdFaultPolicy` beans via the
existing builder API. Same `${fault.*}` interpolation, same `FaultCountStore`
injection.

### 9.4 CBR Registration

The `cbr:` section is declarative metadata — it tells the CBR infrastructure
what features and outcome signals are relevant for this node type. The
declarations are registered as `CbrPluginMetadata` records at build time;
they do not create runtime behavior in this spec's scope.

**Feature declarations** describe what spec and actual-state properties are
relevant for case similarity matching. A future feature-aware
`ConfigurationRetriever` implementation will consume these declarations to
build per-type similarity indices.

**Outcome signal declarations** describe what constitutes success/failure
after reconciliation. `${result.*}` references in outcome signals resolve
against the **actual-state** pipeline's step context — the pipeline that
observes current state after a CBR-proposed change has been applied. This
is distinct from the provisioner pipeline's context. Outcome signals are
not evaluated by the plugin system itself — a future CBR Revise step
(currently outside desiredstate scope — see CBR integration design
§Deferred) will consume these declarations along with actual-state
pipeline results to provide per-type outcome feedback to
`CbrProposalTracker`.

The existing CBR infrastructure (`ConfigurationRetriever`,
`ConfigurationAdapter`, `CbrFaultPolicy`, `CbrSituationRecompiler`) operates
at the graph level. Per-type feature declarations extend this to type-aware
similarity matching — designed in a companion CBR evolution spec (tracked
under casehubio/casehub-ops#87 subsystem 5).

### 9.5 RAS Registration

`YamlPluginRasRegistrar` builds `SituationDefinition` records from the
plugin's `ras:` declarations and registers them via a synthetic
`SituationDefinitionProvider` bean.

The `ras:` YAML vocabulary maps to `SituationDefinition` record fields:

| YAML field | SituationDefinition field | Mapping |
|-----------|--------------------------|---------|
| `name` | `situationId` | Prefixed: `plugin.<type>.<name>` |
| `events` | `eventTypes` | Mapped to `DesiredStateEventTypes` constants |
| `correlation-window` | `correlationWindow` | Parsed as `Duration` |
| `chain-mode` | `chainMode` | See chain mode mapping below |
| `trigger` | `triggerAction` | `create-case` → `TriggerAction.CreateCase(...)`, `emit-event` → `TriggerAction.EmitEvent(...)` |
| `trigger-mode` | `triggerMode` | `fire-once` → `FireOnce()`, `repeating: <duration>` → `Repeating(duration)` |
| `correlation-key` | `correlationKeyExpression` | Expression evaluator from interpolation |

**Chain mode mapping:** The YAML chain mode declaration selects a `ChainMode`
variant. The referenced ganglion is inferred from event types —
`NODE_FAULTED`/`NODE_RECOVERED` events map to `NodeFaultGanglion.ID`,
`NODE_DRIFTED` events map to `PersistentDriftGanglion.ID`:

| YAML | ChainMode | Semantics |
|------|-----------|-----------|
| `streak: N` | `Streak(ganglionId, N)` | N consecutive positive evaluations |
| `count: N` | `Count(ganglionId, N)` | N total positive evaluations in window |
| `rate: { threshold: F, window: N }` | `Rate(ganglia, F, N)` | F fraction positive in window of N |

**Build-time ganglion compatibility validation:** `streak` and `count`
chain modes require a single ganglion ID. If the declared `events` contain
primary events (excluding `NODE_RECOVERED`) that map to different ganglia,
the build fails. Example: `events: [NODE_FAULTED, NODE_DRIFTED]` with
`chain-mode: streak: 3` is a build error — `NODE_FAULTED` maps to
`NodeFaultGanglion.ID` and `NODE_DRIFTED` maps to
`PersistentDriftGanglion.ID`. Use `rate` for multi-ganglion situations
(accepts a `Set<String>` of ganglia), or split into separate situation
definitions.

Custom chain modes (And, Or, Threshold, Sequence) or custom ganglia fall
back to Java `SituationDefinitionProvider` implementations.

## 10. Build-Time Validation Pipeline

```
BUILD TIME (YamlPluginProcessor)
────────────────────────────────
1. Discover plugin files at META-INF/desiredstate/plugins/
2. Parse YAML → PluginModel (structured model types)
3. Validate plugin version (reject unknown versions)
4. Validate spec schema:
   - Field types are supported
   - Constraints are consistent (min ≤ max, pattern compiles)
   - Default values match field types
5. Validate type conflicts:
   - No Java @NodeTypeId claims same type
   - No duplicate YAML plugin types
6. Discover and validate primitives:
   - All step primitive names exist (Java or YAML registry)
   - Composition cycle detection
   - Nesting depth check (≤ 5)
6a. Validate actual-state pipelines contain exactly one `compare-state` step
6b. Validate actual-state pipelines do not contain `approval-gate` steps
7. Validate interpolation references:
   - ${spec.*} references exist in spec schema or Java record
   - ${auth.*} names resolve to declared auth stanzas
   - ${param.*} references exist in compound primitive parameters
   - ${result.*} references bind to a prior step's result name
   - Typo detection with "did you mean?" suggestions
8. Validate auth stanzas:
   - All `auth:` step references resolve to declared plugin auth stanzas
   - credentialRef format is valid
9. Validate fault-policy section (reuses #116 validation)
10. Validate CBR features:
    - source references resolve to spec fields or actual-state output
    - similarity types are known (numeric, categorical)
11. Validate RAS situations:
    - events map to known DesiredStateEventTypes constants
    - chain-mode is a supported variant (streak, count, rate)
    - streak/count chain modes: all primary events (excluding NODE_RECOVERED)
      must infer the same ganglion; mixed-ganglion events are a build error
    - correlation-window parses as valid Duration
    - trigger and trigger-mode are known values
    - correlation-key references resolve
12. Cross-file type coverage:
    - Every node type referenced in graph files has a provisioner
      (Java @NodeTypeId or YAML plugin)
    - Union of annotation, YAML graph, and YAML plugin type sets
13. Register synthetic beans:
    - YamlPluginProvisioner
    - YamlPluginActualStateAdapter
    - ThresholdFaultPolicy per plugin fault-policy
    - NodeSpecFactory per YAML-declared type
    - SituationDefinitionProvider for YAML-declared RAS situations
    - CbrPluginMetadata per YAML-declared CBR section
```

Errors reference source YAML file and line number. Typo detection uses
Levenshtein distance for "did you mean?" suggestions on spec field names,
primitive names, and auth ref names.

## 11. Module Structure

New modules in casehub-desiredstate:

| Module | Artifact | Purpose |
|--------|----------|---------|
| `plugin/api/` | `casehub-desiredstate-plugin-api` | `StepPrimitive` SPI, `StepResult`, `StepContext`, `StepParameters`, `YamlNodeSpec`, `PluginSchemaRegistry` |
| `plugin/runtime/` | `casehub-desiredstate-plugin` | `StepPipelineExecutor`, `YamlPluginProvisioner`, `YamlPluginActualStateAdapter`, built-in Java primitives, `YamlPluginCbrRegistrar`, `YamlPluginRasRegistrar` |
| `plugin/deployment/` | `casehub-desiredstate-plugin-deployment` | `YamlPluginProcessor` (build-time validation), `PluginBuildItem`, primitive registry, cross-file type coverage |

**Dependency direction:**
- `plugin/api/` depends on `casehub-desiredstate-api` (NodeSpec, NodeType, etc.)
- `plugin/runtime/` depends on `plugin/api/` + `casehub-desiredstate` (runtime)
  + `casehub-platform` (CredentialResolver) + `casehub-ras-api`
  (SituationDefinition, ChainMode, TriggerAction for YamlPluginRasRegistrar)
- `plugin/deployment/` depends on `plugin/runtime/` + yaml deployment
  infrastructure

**Why separate modules:** The `StepPrimitive` SPI must be in an api module so
library JARs can implement custom Java primitives without pulling in the full
runtime. Same pattern as `casehub-desiredstate-api` for NodeProvisioner.

## 12. Runtime Error Reporting

When a step fails at runtime, the error includes:
- Plugin file path and type name
- Step name/index in the pipeline
- Primitive being executed
- Interpolated parameter values (credentials masked)
- Full step context (prior results, spec values)

Example:
```
StepExecutionException: Plugin 'k8s-deployment' step 2 (assert) failed
  at META-INF/desiredstate/plugins/k8s-deployment.yaml
  Primitive: assert
  Condition: "${result.response.status} in [200, 201]" → "404 in [200, 201]"
  Context: spec.namespace=default, spec.name=my-app
  Cause: Provision failed: HTTP 404
```

## 13. Plugin Schema Versioning

Plugin YAML files include `version: 1`. Schema evolution defaults to
backward-compatible (additive changes). Non-backward-compatible changes
increment the version. Build-time validation rejects unknown versions.
No migration transformers in v1.

## 14. Deferred Items

| Item | Rationale | Tracking |
|------|-----------|----------|
| Java extension primitives (RestClient, GraphQlClient, AuthProvider, StreamingStateSource, RateLimiter, NodeSpecSchemaGenerator) | Depends on StepPrimitive SPI design — separate spec | casehubio/casehub-ops#87 subsystem 2 |
| 10 concrete NodeSpec plugins | Depends on plugin schema + primitive registry | casehubio/casehub-ops#87 subsystem 3 |
| CaseHub capability integration depth | CBR/RAS declarative metadata is v1; deep integration is v2. Per-capability: trust-weighted execution, audit trail (Ledger), case creation (Engine), event summarisation (Blocks) | casehubio/casehub-ops#87 subsystem 5 |
| CBR runtime integration (feature-aware ConfigurationRetriever, outcome feedback) | Plugin `cbr:` sections declare metadata; retrieval/feedback infrastructure is a companion spec | casehubio/casehub-ops#87 subsystem 5 |
| Custom RAS ganglia and advanced chain modes (And, Or, Threshold, Sequence) | v1 maps to existing ganglia (NodeFaultGanglion, PersistentDriftGanglion); custom ganglia need Java | Future issue |
| Plugin testing model (CLI validator, WireMock integration) | Architecturally significant, companion spec | D13 |
| IDE plugin support (VSCode/IntelliJ autocomplete, refactoring) | Depends on PluginSchemaRegistry being in place | Future issue |
| `poll-until` and `paginate` primitives | Useful but not core — can be added as Java primitives later | Future issue |

**Absorbed primitives from issue #87:** `auth-ref` is absorbed by the `auth:`
stanza and `${auth.*}` interpolation (D6) — not a separate primitive.
`retry` is absorbed by `on-error: retry` step control flow (§6.2) — not a
separate primitive.

## 15. Decisions

See `decisions.md` in this spec directory for the full decision log (D1–D14).

## 16. References

- `api/src/.../NodeSpec.java` — NodeSpec interface with nodeType() and humanGating()
- `api/src/.../NodeProvisioner.java` — Provisioner SPI with handledTypes()
- `api/src/.../ActualStateAdapter.java` — Actual state SPI with handledTypes()
- `api/src/.../NodeSpecFactory.java` — Factory SPI for NodeSpec creation from raw maps
- `api/src/.../ThresholdFaultPolicy.java` — Reusable fault policy builder
- `runtime/src/.../DefaultNodeProvisionerRouter.java` — Router building type→provisioner table
- `yaml/runtime/src/.../YamlGraphRecorder.java` — Existing YAML runtime interpretation
- `yaml/runtime/src/.../NodeSpecRegistry.java` — Type string → NodeSpec class mapping
- `yaml/deployment/src/.../YamlDesiredStateProcessor.java` — Existing YAML build-time validation
- `annotations/deployment/src/.../DesiredStateGraphBuildItem.java` — Cross-surface validation
- `casehub-platform: CredentialResolver` — Named credential resolution SPI
- #116 YAML language extensions design spec — interpolation model, fault policy syntax
- #116 decisions D1 — explicit interpolation namespaces prevent ambiguity
- GE-20260806-272a90 — "Adding a deployment node type requires 6 components + 4 ripple updates"
- GE-20260709-774697 — Adapter router must call all adapters for orphan detection
- casehubio/casehub-ops#87 — Source issue
