# YAML Plugin Architecture Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/casehub-ops#87 — YAML-first plugin architecture
**Issue group:** casehubio/casehub-ops#87

**Goal:** Build the YAML plugin system that lets operators define complete
self-healing resource types (spec schema, actual-state detection, provisioning,
fault policies, CBR features, RAS situations) in a single YAML file with no
Java required.

**Architecture:** Three new Maven modules (`plugin/api`, `plugin/runtime`,
`plugin/deployment`) following the existing yaml/ module pattern. Plugin YAML
files at `META-INF/desiredstate/plugins/<type>.yaml` are discovered at build
time, validated exhaustively, and interpreted at runtime by generic
`YamlPluginProvisioner` and `YamlPluginActualStateAdapter` beans that implement
existing SPIs. Step pipelines compose Java and YAML primitives via a
`StepPipelineExecutor`.

**Tech Stack:** Java 21, Quarkus 3.x (CDI, Jandex, Gizmo for synthetic beans),
Jackson YAML, casehub-platform-api (CredentialResolver), casehub-ras-api
(SituationDefinition).

## Global Constraints

- All new production code in `io.casehub.desiredstate.plugin` package hierarchy
- Plugin api/ module depends only on `casehub-desiredstate-api` — no CDI runtime deps
- Plugin runtime/ depends on `casehub-desiredstate-api` + `casehub-platform-api` (CredentialResolver) + `casehub-ras-api` (SituationDefinition)
- Plugin deployment/ depends on `plugin/runtime` + `yaml/deployment` infrastructure
- Build-time validation catches all structural errors — runtime never fails on schema/reference issues
- All `${}` interpolation uses explicit prefixes: `${spec.*}`, `${auth.*}`, `${result.*}`, `${param.*}`
- YAML 1.2 Core Schema boolean resolution (only `true`/`false` are boolean literals)
- Every test uses `@QuarkusTest` or plain JUnit as appropriate — no mocks for types under test

---

## Batch 1: SPI Foundation + Expression Engine

After this batch: the foundational types exist, expressions can be evaluated,
and interpolation resolves plugin-specific namespaces. No runtime behavior yet
but the contracts are locked in for all subsequent batches.

### Task 1: Module scaffolding + SPI types

Create the three plugin modules and define the step pipeline SPI contract.

**Files:**
- Create: `plugin/pom.xml` (parent pom with 3 submodules)
- Create: `plugin/api/pom.xml`
- Create: `plugin/runtime/pom.xml`
- Create: `plugin/deployment/pom.xml`
- Modify: `pom.xml:root` — add `<module>plugin</module>`
- Create: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/StepPrimitive.java`
- Create: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/StepResult.java`
- Create: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/StepContext.java`
- Create: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/StepParameters.java`
- Create: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/YamlNodeSpec.java`
- Create: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/PluginSchemaRegistry.java`
- Test: `plugin/api/src/test/java/io/casehub/desiredstate/plugin/api/YamlNodeSpecTest.java`
- Test: `plugin/api/src/test/java/io/casehub/desiredstate/plugin/api/StepContextTest.java`

**Interfaces:**
- Produces: `StepPrimitive` SPI — `String name()`, `StepResult execute(StepParameters, StepContext)`
- Produces: `StepResult` — wraps `Map<String, Object>` with typed accessors and deep path traversal
- Produces: `StepContext` — accumulates `spec`, `auth`, `result`, `param` bindings
- Produces: `YamlNodeSpec implements NodeSpec` — wraps `Map<String, Object>` with `get(fieldName)`
- Produces: `StepParameters` — wraps step YAML map with typed accessors

- [ ] **Step 1: Create module pom.xml files**

  Create `plugin/pom.xml` as parent with `<packaging>pom</packaging>` and three submodules.
  `plugin/api/pom.xml` depends on `casehub-desiredstate-api`.
  `plugin/runtime/pom.xml` depends on `casehub-desiredstate-plugin-api`, `casehub-desiredstate`,
  `casehub-platform-api`, `casehub-ras-api`, `quarkus-arc`, `quarkus-rest-client-jackson`.
  `plugin/deployment/pom.xml` depends on `casehub-desiredstate-plugin` (runtime),
  `casehub-desiredstate-yaml-deployment`, `quarkus-arc-deployment`.
  Add `<module>plugin</module>` to root `pom.xml`.

  Follow the existing yaml/ module pattern — copy `yaml/pom.xml` structure as template.
  Use `mvn --batch-mode install -pl plugin/api -am` to verify the module resolves.

- [ ] **Step 2: Write YamlNodeSpec test**

  ```java
  @Test
  void implementsNodeSpec() {
      var spec = new YamlNodeSpec(
          NodeType.of("k8s-deployment"),
          HumanGating.NONE,
          Map.of("namespace", "default", "replicas", 3));
      assertThat(spec.nodeType()).isEqualTo(NodeType.of("k8s-deployment"));
      assertThat(spec.humanGating()).isEqualTo(HumanGating.NONE);
      assertThat(spec.get("namespace")).isEqualTo("default");
      assertThat(spec.get("replicas")).isEqualTo(3);
      assertThat(spec.get("nonexistent")).isNull();
      assertThat(spec.fields()).hasSize(2);
  }
  ```

- [ ] **Step 3: Implement YamlNodeSpec**

  ```java
  public final class YamlNodeSpec implements NodeSpec {
      private final NodeType nodeType;
      private final HumanGating humanGating;
      private final Map<String, Object> fields;

      public YamlNodeSpec(NodeType nodeType, HumanGating humanGating,
                          Map<String, Object> fields) {
          this.nodeType = Objects.requireNonNull(nodeType);
          this.humanGating = Objects.requireNonNull(humanGating);
          this.fields = Map.copyOf(fields);
      }

      @Override public NodeType nodeType() { return nodeType; }
      @Override public HumanGating humanGating() { return humanGating; }
      public Object get(String fieldName) { return fields.get(fieldName); }
      public Map<String, Object> fields() { return fields; }
  }
  ```

- [ ] **Step 4: Write StepResult and StepContext tests**

  Test `StepResult` deep path traversal: `result.get("body.status.replicas")` → `3`.
  Test `StepContext` accumulation: add spec bindings, add result binding, verify
  `resolve("spec.namespace")` and `resolve("result.response.status")` work.

- [ ] **Step 5: Implement StepResult, StepContext, StepParameters, StepPrimitive**

  `StepResult`: wraps `Map<String, Object>`, `get(String dotPath)` splits on `.` and
  walks the nested map. Returns `null` for missing paths.

  `StepContext`: holds `Map<String, Object> spec`, `Map<String, Map<String, String>> auth`,
  `Map<String, StepResult> results`, `Map<String, Object> params`. Provides
  `resolve(String prefixedRef)` that dispatches by prefix.

  `StepParameters`: wraps `Map<String, Object>` from YAML step block. Typed accessors:
  `getString(key)`, `getInt(key)`, `getMap(key)`, `getList(key)`.

  `StepPrimitive`: interface with `name()` and `execute(StepParameters, StepContext)`.

  `PluginSchemaRegistry`: placeholder interface — `Map<String, PluginSchema>` keyed by type name.
  `PluginSchema` record: `String type`, `int version`, `Map<String, FieldDefinition> fields`.
  `FieldDefinition` record: `String type`, `boolean required`, `Object defaultValue`,
  `Map<String, Object> constraints`.

- [ ] **Step 6: Run tests, verify pass**

  Run: `mvn --batch-mode test -pl plugin/api`

- [ ] **Step 7: Commit**

  ```
  feat(#87): add plugin SPI foundation — StepPrimitive, StepResult, StepContext, YamlNodeSpec
  Refs casehubio/casehub-ops#87
  ```

### Task 2: Expression evaluator

Standalone expression evaluation engine for conditions in `compare-state`,
`assert`, and `when:` directives.

**Files:**
- Create: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/expr/ExpressionEvaluator.java`
- Create: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/expr/ExpressionParseException.java`
- Test: `plugin/api/src/test/java/io/casehub/desiredstate/plugin/api/expr/ExpressionEvaluatorTest.java`

**Interfaces:**
- Consumes: nothing (standalone)
- Produces: `ExpressionEvaluator.evaluate(String expression, Map<String, Object> bindings) → boolean`

- [ ] **Step 1: Write expression evaluator tests**

  Test all operators from spec §6.3:
  ```java
  @Test void equality() {
      assertThat(eval("200 == 200", Map.of())).isTrue();
      assertThat(eval("200 == 404", Map.of())).isFalse();
  }
  @Test void inequality() { assertThat(eval("200 != 404", Map.of())).isTrue(); }
  @Test void numericComparison() {
      assertThat(eval("3 > 0", Map.of())).isTrue();
      assertThat(eval("0 >= 1", Map.of())).isFalse();
  }
  @Test void setMembership() {
      assertThat(eval("200 in [200, 201]", Map.of())).isTrue();
      assertThat(eval("404 in [200, 201]", Map.of())).isFalse();
  }
  @Test void contains() {
      assertThat(eval("\"Ready or not\" contains \"Ready\"", Map.of())).isTrue();
  }
  @Test void booleanCombinators() {
      assertThat(eval("200 == 200 and 3 > 0", Map.of())).isTrue();
      assertThat(eval("200 == 404 or 3 > 0", Map.of())).isTrue();
  }
  @Test void negation() { assertThat(eval("not 200 == 404", Map.of())).isTrue(); }
  @Test void nullHandling() {
      assertThat(eval("null == 200", Map.of())).isFalse();
      assertThat(eval("null == null", Map.of())).isTrue();
      assertThat(eval("null > 0", Map.of())).isFalse();
  }
  @Test void interpolatedValues() {
      assertThat(eval("${status} == 200", Map.of("status", 200))).isTrue();
      assertThat(eval("${count} < ${max}", Map.of("count", 2, "max", 5))).isTrue();
  }
  @Test void invalidSyntax() {
      assertThrows(ExpressionParseException.class, () -> eval("200 ===", Map.of()));
  }
  ```

- [ ] **Step 2: Run tests to verify they fail**

  Run: `mvn --batch-mode test -pl plugin/api -Dtest=ExpressionEvaluatorTest`

- [ ] **Step 3: Implement ExpressionEvaluator**

  Recursive descent parser. Tokenizer splits on whitespace respecting quoted strings
  and `${}` references. Grammar:
  ```
  expr     → or_expr
  or_expr  → and_expr ('or' and_expr)*
  and_expr → not_expr ('and' not_expr)*
  not_expr → 'not' not_expr | cmp_expr
  cmp_expr → value (('==' | '!=' | '<' | '>' | '<=' | '>=' | 'in' | 'contains') value)?
  value    → NUMBER | STRING | 'null' | INTERPOLATION | '[' value (',' value)* ']'
  ```

  Interpolation: `${name}` resolved against the bindings map before evaluation.
  Interpolated values are atomic — they cannot inject operators.

  Null semantics: any comparison with null → false. null == null → true.

- [ ] **Step 4: Run tests, verify pass**

  Run: `mvn --batch-mode test -pl plugin/api -Dtest=ExpressionEvaluatorTest`

- [ ] **Step 5: Commit**

  ```
  feat(#87): add expression evaluator — condition vocabulary for plugin steps
  Refs casehubio/casehub-ops#87
  ```

### Task 3: Plugin interpolation resolver

Extends the expression evaluator with plugin-specific `${prefix.*}` resolution
against a `StepContext`.

**Files:**
- Create: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/PluginInterpolator.java`
- Test: `plugin/api/src/test/java/io/casehub/desiredstate/plugin/api/PluginInterpolatorTest.java`

**Interfaces:**
- Consumes: `StepContext` (from Task 1), `ExpressionEvaluator` (from Task 2)
- Produces: `PluginInterpolator.interpolate(String template, StepContext ctx) → String`
- Produces: `PluginInterpolator.evaluateCondition(String condition, StepContext ctx) → boolean`
- Produces: `PluginInterpolator.interpolateMap(Map<String, Object> template, StepContext ctx) → Map<String, Object>`

- [ ] **Step 1: Write interpolation tests**

  ```java
  @Test void specInterpolation() {
      var ctx = StepContext.builder()
          .spec(Map.of("namespace", "default", "replicas", 3))
          .build();
      assertThat(interpolate("ns=${spec.namespace}", ctx)).isEqualTo("ns=default");
  }
  @Test void resultInterpolation() {
      var ctx = StepContext.builder()
          .spec(Map.of())
          .addResult("response", StepResult.of(Map.of("status", 200, "body",
              Map.of("status", Map.of("replicas", 3)))))
          .build();
      assertThat(interpolate("${result.response.status}", ctx)).isEqualTo("200");
      assertThat(interpolate("${result.response.body.status.replicas}", ctx)).isEqualTo("3");
  }
  @Test void authInterpolation() {
      var ctx = StepContext.builder()
          .spec(Map.of())
          .addAuth("k8s", Map.of("endpoint", "api.k8s.io", "token", "secret"))
          .build();
      assertThat(interpolate("https://${auth.k8s.endpoint}/api", ctx))
          .isEqualTo("https://api.k8s.io/api");
  }
  @Test void conditionWithContext() {
      var ctx = StepContext.builder()
          .spec(Map.of("replicas", 3))
          .addResult("r", StepResult.of(Map.of("status", 200)))
          .build();
      assertThat(evaluateCondition("${result.r.status} == 200", ctx)).isTrue();
  }
  @Test void deepMapInterpolation() {
      var ctx = StepContext.builder().spec(Map.of("name", "app")).build();
      var template = Map.of("metadata", Map.of("name", "${spec.name}"));
      var resolved = interpolateMap(template, ctx);
      assertThat(((Map<?,?>)resolved.get("metadata")).get("name")).isEqualTo("app");
  }
  @Test void unknownPrefix() {
      var ctx = StepContext.builder().spec(Map.of()).build();
      assertThrows(InterpolationException.class,
          () -> interpolate("${unknown.field}", ctx));
  }
  ```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement PluginInterpolator**

  `interpolate(String, StepContext)`: regex `\$\{([^}]+)\}` finds all references.
  For each match, split on first `.` to get prefix. Dispatch:
  - `spec` → `ctx.spec().get(remainder)` with deep path traversal
  - `auth` → split remainder on first `.` to get auth name + key → `ctx.auth(name).get(key)`
  - `result` → split remainder on first `.` to get result name + path → `ctx.result(name).get(path)`
  - `param` → `ctx.params().get(remainder)`
  - other → `InterpolationException`

  `interpolateMap`: recursive walk of Map/List values, interpolating string values.
  Non-string values pass through unchanged.

  `evaluateCondition`: interpolate all `${}` references in the condition string,
  then delegate to `ExpressionEvaluator.evaluate()`.

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Commit**

  ```
  feat(#87): add plugin interpolator — ${spec.*}, ${auth.*}, ${result.*}, ${param.*}
  Refs casehubio/casehub-ops#87
  ```

---

## Batch 2: YAML Model + Step Executor + Built-in Primitives

After this batch: plugin YAML files can be parsed into structured model types,
and step pipelines can be executed with built-in primitives (rest-call,
json-extract, compare-state, assert). The core runtime engine works.

### Task 4: Plugin YAML model types + parser

Define the structured model types for plugin YAML and a Jackson-based parser.

**Files:**
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginModel.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginHeader.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginSpecSchema.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginFieldDef.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginStepDef.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginProvisionerDef.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginAuthStanza.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginCbrDef.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginRasDef.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginParser.java`
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/model/PluginParserTest.java`
- Test resource: `plugin/runtime/src/test/resources/META-INF/desiredstate/plugins/test-resource.yaml`

**Interfaces:**
- Consumes: nothing (standalone parsing)
- Produces: `PluginParser.parse(InputStream yaml) → PluginModel`
- Produces: `PluginModel` record — `PluginHeader header`, `PluginSpecSchema spec`,
  `List<PluginStepDef> actualStateSteps`, `PluginProvisionerDef provisioner`,
  `List<YamlFaultPolicy> faultPolicies`, `PluginCbrDef cbr`, `PluginRasDef ras`

- [ ] **Step 1: Create test YAML resource**

  Create `test-resource.yaml` with the K8s deployment plugin from spec §4
  (simplified — no actual K8s API, just structural validation).

- [ ] **Step 2: Write parser test**

  ```java
  @Test void parsesPluginYaml() {
      var model = PluginParser.parse(
          getClass().getResourceAsStream("/META-INF/desiredstate/plugins/test-resource.yaml"));
      assertThat(model.header().type()).isEqualTo("test-resource");
      assertThat(model.header().version()).isEqualTo(1);
      assertThat(model.spec().fields()).containsKey("name");
      assertThat(model.actualStateSteps()).isNotEmpty();
      assertThat(model.provisioner().provisionSteps()).isNotEmpty();
      assertThat(model.provisioner().deprovisionSteps()).isNotEmpty();
  }
  ```

- [ ] **Step 3: Implement model records and parser**

  All model types are Java records. `PluginParser` uses Jackson `YAMLFactory` with
  YAML 1.2 Core Schema boolean resolution. The parser maps the top-level YAML
  structure to `PluginModel` via `ObjectMapper.readValue()` with `@JsonProperty`
  annotations.

  `PluginStepDef`: `String primitiveName`, `Map<String, Object> parameters`, `String resultName`,
  `String when`, `String onError`, `int maxRetries`, `String backoff`.

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Commit**

  ```
  feat(#87): add plugin YAML model + parser
  Refs casehubio/casehub-ops#87
  ```

### Task 5: StepPipelineExecutor + built-in primitives

The core runtime engine that executes step pipelines and the essential
built-in primitives.

**Files:**
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/StepPipelineExecutor.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PrimitiveRegistry.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/StepExecutionException.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/primitives/RestCallPrimitive.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/primitives/JsonExtractPrimitive.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/primitives/CompareStatePrimitive.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/primitives/AssertPrimitive.java`
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/StepPipelineExecutorTest.java`
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/primitives/CompareStatePrimitiveTest.java`
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/primitives/AssertPrimitiveTest.java`

**Interfaces:**
- Consumes: `StepPrimitive` (Task 1), `PluginInterpolator` (Task 3), `StepContext` (Task 1),
  `ExpressionEvaluator` (Task 2), `PluginStepDef` (Task 4)
- Produces: `StepPipelineExecutor.execute(List<PluginStepDef>, StepContext) → StepResult`
- Produces: `StepPipelineExecutor.executeActualState(List<PluginStepDef>, StepContext) → NodeStatus`
- Produces: `PrimitiveRegistry.resolve(String name) → StepPrimitive`

- [ ] **Step 1: Write CompareStatePrimitive test**

  ```java
  @Test void mapsToPresent() {
      var params = StepParameters.of(Map.of(
          "present-when", "${result.r.status} == 200",
          "drifted-when", "${result.r.replicas} < 3",
          "absent-when", "${result.r.status} == 404"));
      var ctx = StepContext.builder()
          .spec(Map.of())
          .addResult("r", StepResult.of(Map.of("status", 200, "replicas", 3)))
          .build();
      var result = new CompareStatePrimitive().execute(params, ctx);
      assertThat(result.get("nodeStatus")).isEqualTo("PRESENT");
  }
  @Test void mapsToDrifted() {
      // status=200 but replicas=1 < 3 → DRIFTED (absent checked first, then drifted)
      var params = StepParameters.of(Map.of(
          "present-when", "${result.r.status} == 200",
          "drifted-when", "${result.r.replicas} < 3",
          "absent-when", "${result.r.status} == 404"));
      var ctx = StepContext.builder()
          .spec(Map.of())
          .addResult("r", StepResult.of(Map.of("status", 200, "replicas", 1)))
          .build();
      var result = new CompareStatePrimitive().execute(params, ctx);
      assertThat(result.get("nodeStatus")).isEqualTo("DRIFTED");
  }
  @Test void mapsToAbsent() {
      var params = StepParameters.of(Map.of(
          "present-when", "${result.r.status} == 200",
          "absent-when", "${result.r.status} == 404"));
      var ctx = StepContext.builder()
          .spec(Map.of())
          .addResult("r", StepResult.of(Map.of("status", 404)))
          .build();
      var result = new CompareStatePrimitive().execute(params, ctx);
      assertThat(result.get("nodeStatus")).isEqualTo("ABSENT");
  }
  @Test void mapsToUnknownWhenNoConditionMatches() {
      var params = StepParameters.of(Map.of("present-when", "${result.r.status} == 200"));
      var ctx = StepContext.builder()
          .spec(Map.of())
          .addResult("r", StepResult.of(Map.of("status", 500)))
          .build();
      var result = new CompareStatePrimitive().execute(params, ctx);
      assertThat(result.get("nodeStatus")).isEqualTo("UNKNOWN");
  }
  ```

- [ ] **Step 2: Implement CompareStatePrimitive**

  Evaluation order: `absent-when` → `drifted-when` → `present-when`. First true → that status.
  None true → `UNKNOWN`. Uses `PluginInterpolator.evaluateCondition()`.

- [ ] **Step 3: Write AssertPrimitive test**

  ```java
  @Test void passesWhenTrue() {
      var params = StepParameters.of(Map.of("condition", "200 in [200, 201]"));
      var ctx = StepContext.builder().spec(Map.of()).build();
      var result = new AssertPrimitive().execute(params, ctx);
      assertThat(result.get("passed")).isEqualTo(true);
  }
  @Test void failsWithMessage() {
      var params = StepParameters.of(Map.of(
          "condition", "404 in [200, 201]",
          "message", "Provision failed: HTTP 404"));
      var ctx = StepContext.builder().spec(Map.of()).build();
      assertThrows(StepExecutionException.class,
          () -> new AssertPrimitive().execute(params, ctx));
  }
  ```

- [ ] **Step 4: Implement AssertPrimitive, JsonExtractPrimitive, RestCallPrimitive**

  `AssertPrimitive`: evaluates condition, throws `StepExecutionException` with message on false.

  `JsonExtractPrimitive`: takes `input` (Map or string) and `path` (JSONPath expression),
  extracts value. Uses Jackson for path traversal.

  `RestCallPrimitive`: uses `java.net.http.HttpClient` to make HTTP calls. Parameters:
  `method`, `url`, `headers` (Map), `body` (Map → JSON), `auth` (resolved from context).
  Returns `StepResult` with `status`, `headers`, `body` (parsed JSON or raw string).
  Auth header injected from `${auth.<name>.token}` as `Bearer` token if present.

- [ ] **Step 5: Write StepPipelineExecutor test**

  ```java
  @Test void executesSequentialSteps() {
      var registry = PrimitiveRegistry.of(Map.of(
          "assert", new AssertPrimitive(),
          "compare-state", new CompareStatePrimitive()));
      var executor = new StepPipelineExecutor(registry, new PluginInterpolator());
      var steps = List.of(
          new PluginStepDef("compare-state", Map.of(
              "present-when", "${spec.value} == 1",
              "absent-when", "${spec.value} == 0"), "state", null, null, 0, null));
      var ctx = StepContext.builder().spec(Map.of("value", 1)).build();
      var status = executor.executeActualState(steps, ctx);
      assertThat(status).isEqualTo(NodeStatus.PRESENT);
  }
  @Test void accumulatesResults() {
      // Step 1 produces result "r", Step 2 references ${result.r.*}
      // ... validates context accumulation across steps
  }
  @Test void respectsWhenCondition() {
      // Step with when: "false" is skipped
  }
  @Test void retriesOnError() {
      // Step with on-error: retry, max-retries: 2
  }
  ```

- [ ] **Step 6: Implement StepPipelineExecutor**

  Iterates steps sequentially. For each step:
  1. Check `when:` condition — skip if false
  2. Resolve primitive from `PrimitiveRegistry`
  3. Interpolate parameters via `PluginInterpolator`
  4. Execute primitive
  5. If step has `result:` name, add `StepResult` to context under that name
  6. On error: check `on-error` directive (retry/fail/skip)

  `executeActualState()` runs the pipeline, finds the `compare-state` step's result,
  and maps `nodeStatus` string to `NodeStatus` enum.

  `execute()` for provisioner pipelines returns the final `StepResult` (or throws on failure).

  Error reporting: `StepExecutionException` includes plugin type, step index, primitive name,
  interpolated parameters (with auth values masked as `***`), and cause.

- [ ] **Step 7: Run all tests, verify pass**

  Run: `mvn --batch-mode test -pl plugin/api,plugin/runtime`

- [ ] **Step 8: Commit**

  ```
  feat(#87): add StepPipelineExecutor + built-in primitives (compare-state, assert, json-extract, rest-call)
  Refs casehubio/casehub-ops#87
  ```

---

## Batch 3: SPI Wiring — Provisioner + Adapter + NodeSpec Registry

After this batch: YAML-declared plugins work end-to-end — a plugin's provision,
deprovision, and actual-state pipelines execute through the standard
NodeProvisioner and ActualStateAdapter SPIs. The reconciliation loop can
manage YAML-declared node types.

### Task 6: YamlPluginProvisioner + YamlPluginActualStateAdapter

Wire the step pipeline executor to the existing NodeProvisioner and
ActualStateAdapter SPIs.

**Files:**
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginDescriptor.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/YamlPluginProvisioner.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/YamlPluginActualStateAdapter.java`
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/YamlPluginProvisionerTest.java`
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/YamlPluginActualStateAdapterTest.java`

**Interfaces:**
- Consumes: `StepPipelineExecutor` (Task 5), `PluginDescriptor`, `DesiredNode`, `ProvisionContext`,
  `DeprovisionContext`, `DesiredStateGraph`, `CredentialResolver`
- Produces: `YamlPluginProvisioner implements NodeProvisioner` — routes by NodeType to step pipelines
- Produces: `YamlPluginActualStateAdapter implements ActualStateAdapter` — routes by NodeType

- [ ] **Step 1: Write PluginDescriptor record**

  ```java
  public record PluginDescriptor(
      String type,
      int version,
      Duration resyncInterval,
      Map<String, String> authCredentialRefs,
      PluginSpecSchema specSchema,
      List<PluginStepDef> actualStateSteps,
      List<PluginStepDef> provisionSteps,
      List<PluginStepDef> deprovisionSteps,
      List<YamlFaultPolicy> faultPolicies,
      PluginCbrDef cbr,
      PluginRasDef ras) {}
  ```

- [ ] **Step 2: Write YamlPluginProvisioner test**

  ```java
  @Test void provisionDelegatesToStepPipeline() {
      var descriptor = createTestDescriptor(/* provision steps that call assert with true */);
      var provisioner = new YamlPluginProvisioner(
          Map.of(NodeType.of("test"), descriptor),
          executor, credentialResolver);
      assertThat(provisioner.handledTypes()).containsExactly(NodeType.of("test"));
      var node = DesiredNode.of(NodeId.of("n1"), NodeType.of("test"),
          new YamlNodeSpec(NodeType.of("test"), HumanGating.NONE, Map.of("name", "x")));
      var result = provisioner.provision(node, ProvisionContext.of("tenant1", graph));
      assertThat(result).isInstanceOf(ProvisionResult.Success.class);
  }
  @Test void deprovisionDelegatesToStepPipeline() { /* similar */ }
  @Test void failedStepReturnsFailedResult() { /* assert that fails → ProvisionResult.Failed */ }
  @Test void resyncIntervalFromDescriptor() {
      var descriptor = createTestDescriptor(Duration.ofSeconds(30));
      var provisioner = new YamlPluginProvisioner(
          Map.of(NodeType.of("test"), descriptor), executor, credentialResolver);
      assertThat(provisioner.resyncInterval()).isEqualTo(Duration.ofSeconds(30));
  }
  ```

- [ ] **Step 3: Implement YamlPluginProvisioner**

  `provision()`: build `StepContext` from `DesiredNode.spec()` (extract fields via
  `YamlNodeSpec.fields()` or reflective record accessors for Java NodeSpec), resolve
  auth credentials, execute provision step pipeline. Map `StepResult` → `ProvisionResult.Success`.
  On `StepExecutionException` → `ProvisionResult.Failed(e.getMessage())`.

  `deprovision()`: same pattern with deprovision steps → `DeprovisionResult`.

  `handledTypes()`: returns all registered plugin types.

  `resyncInterval()`: returns the shortest resync interval across all plugins (for the
  router). Override `resyncIntervalFor(NodeType)` to return per-plugin intervals.

- [ ] **Step 4: Write YamlPluginActualStateAdapter test**

  ```java
  @Test void readsActualStateForPluginTypes() {
      var descriptor = createTestDescriptor(/* compare-state steps */);
      var adapter = new YamlPluginActualStateAdapter(
          Map.of(NodeType.of("test"), descriptor), executor, credentialResolver);
      var graph = createGraph(DesiredNode.of(NodeId.of("n1"), NodeType.of("test"), spec));
      var actual = adapter.readActual(graph, "tenant1");
      assertThat(actual.statusOf(NodeId.of("n1"))).isEqualTo(NodeStatus.PRESENT);
  }
  ```

- [ ] **Step 5: Implement YamlPluginActualStateAdapter**

  `readActual()`: iterates graph nodes, for each node whose type is in the plugin map,
  builds `StepContext`, executes actual-state steps, collects `NodeStatus`. Returns
  `ActualState.of(statusMap)`.

- [ ] **Step 6: Run tests, verify pass**

- [ ] **Step 7: Commit**

  ```
  feat(#87): add YamlPluginProvisioner + YamlPluginActualStateAdapter — SPI wiring
  Refs casehubio/casehub-ops#87
  ```

### Task 7: NodeSpecRegistry extension

Extend `NodeSpecRegistry` to support factory-based resolution for YAML plugin types
alongside the existing class-based resolution.

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/registry/NodeSpecRegistry.java`
- Test: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/registry/NodeSpecRegistryTest.java`

**Interfaces:**
- Consumes: `NodeSpecFactory` (existing SPI), `YamlNodeSpec` (Task 1)
- Produces: `NodeSpecRegistry.resolveFactory(typeName) → Optional<NodeSpecFactory>`
- Produces: `NodeSpecRegistry.isFactoryType(typeName) → boolean`

- [ ] **Step 1: Write tests for factory resolution path**

  ```java
  @Test void resolveFactoryForYamlType() {
      var factory = (NodeSpecFactory) specMap ->
          new YamlNodeSpec(NodeType.of("yaml-type"), HumanGating.NONE, specMap);
      var registry = NodeSpecRegistry.of(Map.of("java-type", JavaSpec.class.getName()),
          Map.of("yaml-type", factory));
      assertThat(registry.resolveFactory("yaml-type")).isPresent();
      assertThat(registry.isFactoryType("yaml-type")).isTrue();
      assertThat(registry.isFactoryType("java-type")).isFalse();
      assertThat(registry.resolve("java-type")).isEqualTo(JavaSpec.class);
  }
  @Test void factoryProducesYamlNodeSpec() {
      var factory = registry.resolveFactory("yaml-type").orElseThrow();
      var spec = factory.create(Map.of("name", "test"));
      assertThat(spec).isInstanceOf(YamlNodeSpec.class);
      assertThat(spec.nodeType()).isEqualTo(NodeType.of("yaml-type"));
  }
  ```

- [ ] **Step 2: Extend NodeSpecRegistry**

  Add `Map<String, NodeSpecFactory> factoryMap` field alongside existing `typeMap`.
  Add `resolveFactory(String)`, `isFactoryType(String)`.
  Extend static `of()` factory method to accept both maps.

- [ ] **Step 3: Run tests, verify pass**

- [ ] **Step 4: Commit**

  ```
  feat(#87): extend NodeSpecRegistry with factory resolution for YAML plugin types
  Refs casehubio/casehub-ops#87
  ```

---

## Batch 4: Build-Time Validation + Compound Primitives

After this batch: plugin YAML files are discovered at Quarkus build time,
validated exhaustively (spec schema, primitives, interpolation references,
auth refs, type conflicts), and YAML compound primitives compose hierarchically.

### Task 8: YamlPluginProcessor — build-time validation

The Quarkus build extension that discovers and validates plugin YAML files.

**Files:**
- Create: `plugin/deployment/src/main/java/io/casehub/desiredstate/plugin/deployment/YamlPluginProcessor.java`
- Create: `plugin/deployment/src/main/java/io/casehub/desiredstate/plugin/deployment/PluginBuildItem.java`
- Create: `plugin/deployment/src/main/java/io/casehub/desiredstate/plugin/deployment/PluginValidationException.java`
- Test: `plugin/deployment/src/test/java/io/casehub/desiredstate/plugin/deployment/YamlPluginProcessorTest.java`
- Test resource: `plugin/deployment/src/test/resources/META-INF/desiredstate/plugins/valid-plugin.yaml`
- Test resource: `plugin/deployment/src/test/resources/META-INF/desiredstate/plugins/invalid-spec.yaml`

**Interfaces:**
- Consumes: `PluginParser` (Task 4), `PluginModel` (Task 4), `DesiredStateGraphBuildItem` (existing)
- Produces: `PluginBuildItem` — carries validated `PluginDescriptor` through the build pipeline
- Produces: Synthetic bean registrations: `YamlPluginProvisioner`, `YamlPluginActualStateAdapter`,
  `NodeSpecFactory` per type

- [ ] **Step 1: Write build-time validation tests**

  ```java
  @Test void validPluginPassesValidation() {
      // Load valid-plugin.yaml, verify no exceptions
  }
  @Test void unknownFieldTypeFailsBuild() {
      // Plugin with spec field type "complex" → build error
  }
  @Test void inconsistentConstraintsFailsBuild() {
      // Plugin with min > max → build error
  }
  @Test void unknownPrimitiveFailsBuild() {
      // Plugin with step "rest-callz" → build error with "did you mean rest-call?"
  }
  @Test void invalidInterpolationRefFailsBuild() {
      // Plugin with ${spec.namespce} → build error with "did you mean namespace?"
  }
  @Test void typeConflictWithJavaFailsBuild() {
      // Plugin type that also has @NodeTypeId → build error
  }
  ```

- [ ] **Step 2: Implement YamlPluginProcessor**

  `@BuildStep` methods implementing spec §10 validation pipeline:
  1. `discoverPlugins()` — classpath scan `META-INF/desiredstate/plugins/*.yaml`
  2. `validatePlugins()` — parse, validate schema, primitives, interpolation, auth
  3. `registerBeans()` — `SyntheticBeanBuildItem` for provisioner, adapter, factories

  Typo detection: Levenshtein distance ≤ 2 against known field/primitive names.

  Integration with existing `DesiredStateGraphBuildItem` for cross-file type coverage
  validation (step 12 in spec §10).

- [ ] **Step 3: Run tests, verify pass**

- [ ] **Step 4: Commit**

  ```
  feat(#87): add YamlPluginProcessor — build-time plugin discovery and validation
  Refs casehubio/casehub-ops#87
  ```

### Task 9: Compound primitive model + expansion

YAML-defined compound primitives that compose other primitives.

**Files:**
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/CompoundPrimitiveDef.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/CompoundPrimitiveExpander.java`
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PrimitiveRegistry.java`
- Modify: `plugin/deployment/src/main/java/io/casehub/desiredstate/plugin/deployment/YamlPluginProcessor.java`
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/CompoundPrimitiveExpanderTest.java`
- Test resource: `plugin/runtime/src/test/resources/META-INF/desiredstate/primitives/test-compound.yaml`

**Interfaces:**
- Consumes: `PrimitiveRegistry` (Task 5), `PluginStepDef` (Task 4)
- Produces: `CompoundPrimitiveExpander.expand(CompoundPrimitiveDef, PrimitiveRegistry) → List<PluginStepDef>`
- Produces: `CompoundPrimitiveDef` record — `name`, `parameters`, `steps`, `resultBinding`

- [ ] **Step 1: Write compound primitive tests**

  ```java
  @Test void expandsCompoundToFlatSteps() {
      // compound "k8s-call" wrapping "rest-call" → expanded to one rest-call step
      // with ${param.*} → parameter values substituted
  }
  @Test void detectsCycles() {
      // A references B, B references A → CyclicPrimitiveException
  }
  @Test void enforcesMaxDepth() {
      // A→B→C→D→E→F (depth 6 > limit 5) → MaxPrimitiveDepthException
  }
  @Test void parameterScopingInnermostWins() {
      // A(x=1) invokes B(x=2) which uses ${param.x} → resolves to 2
  }
  ```

- [ ] **Step 2: Implement CompoundPrimitiveExpander**

  Build-time expansion: recursively inline compound primitives' steps at each
  invocation site. Track visited primitives for cycle detection. Count depth.
  `${param.*}` scoping: each expansion level pushes a new parameter scope.

- [ ] **Step 3: Extend PrimitiveRegistry and YamlPluginProcessor**

  Registry gains `registerCompound(name, CompoundPrimitiveDef)`.
  Processor gains `discoverCompoundPrimitives()` build step — scans
  `META-INF/desiredstate/primitives/*.yaml`.

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Commit**

  ```
  feat(#87): add compound primitive composition — YAML over YAML over Java
  Refs casehubio/casehub-ops#87
  ```

---

## Batch 5: Integration — Fault Policy + CBR + RAS + End-to-End Test

After this batch: the full plugin system works end-to-end. Fault policies
reuse #116 syntax, CBR metadata is registered, RAS situations are registered,
and an integration test validates the complete pipeline.

### Task 10: Fault policy + CBR + RAS integration

Wire the plugin's fault-policy, cbr, and ras sections to existing infrastructure.

**Files:**
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/YamlPluginFaultPolicyRegistrar.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/CbrPluginMetadata.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/YamlPluginRasRegistrar.java`
- Modify: `plugin/deployment/src/main/java/io/casehub/desiredstate/plugin/deployment/YamlPluginProcessor.java` — add fault policy and RAS bean registration
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/YamlPluginRasRegistrarTest.java`

**Interfaces:**
- Consumes: `ThresholdFaultPolicy.builder()` (existing), `SituationDefinition` (ras-api),
  `SituationDefinitionProvider` (ras-api), `PluginCbrDef` (Task 4), `PluginRasDef` (Task 4)
- Produces: `YamlPluginFaultPolicyRegistrar` — creates `ThresholdFaultPolicy` beans from plugin fault-policy
- Produces: `CbrPluginMetadata` record — feature declarations + outcome signals per type
- Produces: `YamlPluginRasRegistrar` — builds `SituationDefinition` records, implements `SituationDefinitionProvider`

- [ ] **Step 1: Write RAS registration test**

  ```java
  @Test void registersSituationFromPluginYaml() {
      var rasDef = new PluginRasDef(List.of(
          new PluginRasSituation("crash-loop",
              List.of("NODE_FAULTED", "NODE_RECOVERED"),
              Duration.ofMinutes(10),
              Map.of("streak", 3),
              "create-case", "fire-once", "${spec.namespace}")));
      var registrar = new YamlPluginRasRegistrar("k8s-deployment", rasDef);
      var situations = registrar.provide();
      assertThat(situations).hasSize(1);
      var sit = situations.get(0);
      assertThat(sit.situationId()).isEqualTo("plugin.k8s-deployment.crash-loop");
      assertThat(sit.correlationWindow()).isEqualTo(Duration.ofMinutes(10));
  }
  ```

- [ ] **Step 2: Implement integrations**

  `YamlPluginFaultPolicyRegistrar`: reuses `YamlFaultPolicyBuilder` from yaml/runtime
  (same code path as #116 YAML fault policies).

  `CbrPluginMetadata`: record carrying `List<CbrFeature>` and `Map<String, String> outcomeSignals`.
  Registered as a synthetic bean — consumed by future CBR evolution.

  `YamlPluginRasRegistrar implements SituationDefinitionProvider`: maps YAML chain modes
  to `ChainMode` variants. Maps event names to `DesiredStateEventTypes` constants.
  Infers ganglion IDs from event types.

- [ ] **Step 3: Wire registrations in YamlPluginProcessor**

  Add build steps for: fault policy bean registration, CBR metadata bean registration,
  RAS situation provider bean registration.

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Commit**

  ```
  feat(#87): add fault policy + CBR metadata + RAS situation registration
  Refs casehubio/casehub-ops#87
  ```

### Task 11: End-to-end integration test

A `@QuarkusTest` that validates the complete plugin pipeline: plugin YAML
discovered at build time, provisioner invoked, actual-state checked, fault
policy fires.

**Files:**
- Create: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/PluginIntegrationTest.java`
- Create: `plugin/runtime/src/test/resources/META-INF/desiredstate/plugins/mock-resource.yaml`
- Create: `plugin/runtime/src/test/resources/META-INF/desiredstate/mock-resource-graph.yaml`

**Interfaces:**
- Consumes: all prior tasks — validates end-to-end integration

- [ ] **Step 1: Create mock-resource plugin YAML**

  A plugin that provisions via a mock HTTP endpoint (WireMock or similar
  in-process stub), reads actual state, and has a fault policy. Uses
  `compare-state` to map responses to PRESENT/ABSENT.

- [ ] **Step 2: Create mock-resource graph YAML**

  A graph declaring one node of type `mock-resource` with spec fields.

- [ ] **Step 3: Write integration test**

  ```java
  @QuarkusTest
  class PluginIntegrationTest {
      @Inject YamlPluginProvisioner provisioner;
      @Inject YamlPluginActualStateAdapter adapter;

      @Test void pluginTypeIsRegistered() {
          assertThat(provisioner.handledTypes())
              .contains(NodeType.of("mock-resource"));
      }
      @Test void provisionExecutesStepPipeline() {
          var node = createNode("mock-resource", Map.of("name", "test"));
          var result = provisioner.provision(node, ctx);
          assertThat(result).isInstanceOf(ProvisionResult.Success.class);
      }
      @Test void actualStateReturnsCorrectStatus() {
          var graph = createGraph(createNode("mock-resource", Map.of("name", "test")));
          var actual = adapter.readActual(graph, "tenant1");
          assertThat(actual.statusOf(NodeId.of("n1"))).isIn(
              NodeStatus.PRESENT, NodeStatus.ABSENT);
      }
  }
  ```

- [ ] **Step 4: Run integration test**

  Run: `mvn --batch-mode test -pl plugin/runtime -Dtest=PluginIntegrationTest`

- [ ] **Step 5: Run full build**

  Run: `mvn --batch-mode install` — verify all modules compile and all tests pass.

- [ ] **Step 6: Commit**

  ```
  feat(#87): add end-to-end plugin integration test
  Refs casehubio/casehub-ops#87
  ```

---

## References

- `specs/issue-87-yaml-plugin-architecture/2026-09-09-yaml-plugin-architecture-design.md` — design spec
- `specs/issue-87-yaml-plugin-architecture/decisions.md` — decision log (D1–D16)
- `api/src/main/java/io/casehub/desiredstate/api/NodeSpec.java` — NodeSpec SPI
- `api/src/main/java/io/casehub/desiredstate/api/NodeProvisioner.java` — Provisioner SPI
- `api/src/main/java/io/casehub/desiredstate/api/ActualStateAdapter.java` — ActualStateAdapter SPI
- `api/src/main/java/io/casehub/desiredstate/api/NodeSpecFactory.java` — NodeSpecFactory SPI
- `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/registry/NodeSpecRegistry.java` — registry to extend
- `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java` — build pattern reference
- `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java` — runtime pattern reference
- `ras-adapter/src/main/java/io/casehub/desiredstate/ras/DesiredStateSituationDefinitionProvider.java` — RAS pattern reference
- GE-20260806-272a90 — per-type component overhead this eliminates
- casehubio/casehub-ops#87 — source issue
