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
        degraded-when: "${result.response.body.status.availableReplicas} < ${spec.replicas}"
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
      detect: "${result.response.body.status.unavailableReplicas} > 0"
      severity: warning
      correlation-key: "${spec.namespace}"
```

**Required sections:** `plugin`, `spec`, `actual-state`, `provisioner`.
**Optional sections:** `fault-policy`, `cbr`, `ras`.

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

### 5.3 NodeSpecRegistry Extension

`NodeSpecRegistry` gains a second resolution path:

| Source | Resolution |
|--------|-----------|
| Java `@NodeTypeId` | `Class<? extends NodeSpec>` → Jackson `convertValue` (existing) |
| YAML plugin | `NodeSpecFactory` → `YamlNodeSpec` wrapper with schema validation |

The `NodeSpecFactory` SPI already exists for this purpose. Each YAML plugin
registers a factory that validates the raw spec map against the schema and
produces a `YamlNodeSpec`.

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

Interpolation happens first (all `${}` references resolved to values),
then the expression is evaluated. Build-time validates expression syntax
and that all interpolation references resolve. Complex logic that exceeds
this vocabulary falls back to a Java `StepPrimitive`.

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
| `compare-state` | Map response to NodeStatus | `present-when`, `absent-when`, `degraded-when` |
| `assert` | Fail pipeline on condition | `condition`, `message` |

Retry is handled via `on-error: retry` on individual steps (§6.2), not as
a separate primitive. This avoids two retry mechanisms with different semantics.

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
}
```

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
        for (DesiredNode node : graph.nodes()) {
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

### 9.3 Fault Policy Registration

Reuses #116's `YamlGraphRecorder.createFaultPolicy()` path. Each plugin's
`fault-policy:` section produces `ThresholdFaultPolicy` beans via the
existing builder API. Same `${fault.*}` interpolation, same `FaultCountStore`
injection.

### 9.4 CBR Registration

`YamlPluginCbrRegistrar` creates per-type:
- Feature extractors: `source: spec.replicas` → extracts the spec field value
  and wraps as `FeatureValue.Numeric`
- Outcome evaluators: `success: <expression>` → evaluated after reconciliation
  to provide feedback to `CbrProposalTracker`

### 9.5 RAS Registration

`YamlPluginRasRegistrar` registers per-type:
- Situation definitions with the RAS Ganglia
- Correlation key extractors for aggregate detection
- Severity classification

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
7. Validate interpolation references:
   - ${spec.*} references exist in spec schema or Java record
   - ${auth.*} names resolve to declared auth stanzas
   - ${param.*} references exist in compound primitive parameters
   - ${result.*} references bind to a prior step's result name
   - Typo detection with "did you mean?" suggestions
8. Validate auth stanzas:
   - credentialRef format is valid
9. Validate fault-policy section (reuses #116 validation)
10. Validate CBR features:
    - source references resolve to spec fields or actual-state output
    - similarity types are known (numeric, categorical)
11. Validate RAS situations:
    - detect expressions are well-formed
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
  + `casehub-platform` (CredentialResolver)
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
| Java extension primitives (RestClient, GraphQlClient, AuthProvider) | Depends on StepPrimitive SPI design — separate spec | casehubio/casehub-ops#87 subsystem 2 |
| 10 concrete NodeSpec plugins | Depends on plugin schema + primitive registry | casehubio/casehub-ops#87 subsystem 3 |
| CaseHub capability integration depth (Trust, Ledger, Engine, Blocks) | CBR/RAS declarative metadata is v1; deep integration is v2 | casehubio/casehub-ops#87 subsystem 5 |
| Plugin testing model (CLI validator, WireMock integration) | Architecturally significant, companion spec | D13 |
| IDE plugin support (VSCode/IntelliJ autocomplete, refactoring) | Depends on PluginSchemaRegistry being in place | Future issue |
| `poll-until` and `paginate` primitives | Useful but not core — can be added as Java primitives later | Future issue |

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
