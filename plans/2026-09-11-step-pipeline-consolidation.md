# Step Pipeline Consolidation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/casehub-ops#88 — Consolidate step pipeline infrastructure into yaml-core
**Issue group:** #88

**Goal:** Extract generic step pipeline types from `casehub-desiredstate-plugin` into a new
`casehub-platform-yaml-step-core` module, eliminating duplication with yaml-core's
`VariableResolver` and making the step pipeline reusable by other domains.

**Architecture:** Two-phase migration following the #128 pattern. Phase 1 creates
`yaml-step-core` in casehub-platform with all generic types. Phase 2 migrates
desiredstate to consume the new module, deleting local copies and refactoring
`PluginInterpolator` to use `VariableResolver`. Primitives no longer self-interpolate —
the executor pre-resolves parameters via `VariableResolver.resolveMap()`.

**Tech Stack:** Java 26, Maven, casehub-platform-yaml-core (VariableResolver, VariableSource),
java.net.http (RestCallPrimitive), Jackson (JSON parsing)

## Global Constraints

- yaml-step-core package: `io.casehub.yaml.step`, subpackages `expr` and `primitives`
- yaml-step-core depends on `casehub-platform-yaml-core` only (plus java.net.http, jackson-databind)
- Zero desiredstate imports in yaml-step-core — any `io.casehub.desiredstate.*` import is a failure
- StepExecutionException field `pluginType` is renamed to `pipelineId` — generic, not plugin-specific
- All `PluginStepDef` → `StepDef`, `CompoundPrimitiveDef` → `CompoundStepDef` renames
- `VariableSource` is `@FunctionalInterface` with `String resolve(String name)` — per-prefix resolver
- Build: `mvn --batch-mode install` in platform repo, then `mvn --batch-mode install` in desiredstate repo
- **Repo targets:** Batch 1 targets `casehub-platform`. Batches 2-3 target `casehub-desiredstate`.

---

## Batch 1: yaml-step-core module in casehub-platform

**Prerequisite:** casehub-platform must be accessible. If not in the slot, run
`work-slot add-repo platform` to add it, or clone alongside desiredstate.

### Task 1: Create yaml-step-core module with SPI types

**Files:**
- Create: `yaml-step-core/pom.xml`
- Modify: `pom.xml` (platform root — add module entry)
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/StepPrimitive.java`
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/StepResult.java`
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/StepParameters.java`
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/StepDef.java`
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/CompoundStepDef.java`
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/StepExecutionException.java`
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/InterpolationException.java`
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/StepContext.java`
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/expr/ExpressionEvaluator.java`
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/expr/ExpressionParseException.java`
- Test: `yaml-step-core/src/test/java/io/casehub/yaml/step/StepContextTest.java`
- Test: `yaml-step-core/src/test/java/io/casehub/yaml/step/expr/ExpressionEvaluatorTest.java`

**Interfaces:**
- Consumes: `io.casehub.yaml.core.resolver.VariableSource` (`@FunctionalInterface`, `String resolve(String name)`)
- Consumes: `io.casehub.yaml.core.resolver.VariableResolver` (constructor: `Map<String, VariableSource>`, `Set<String>`)
- Produces: `StepPrimitive` interface (`name()`, `execute(StepParameters, StepContext)`)
- Produces: `StepResult` value type (`of(Map)`, `get(String dotPath)`, `data()`)
- Produces: `StepParameters` value type (`of(Map)`, `getString(key)`, `getInt(key)`, `getMap(key)`, `getList(key)`, `get(key)`, `asMap()`)
- Produces: `StepContext` with `specSource()`, `authSource()`, `resultSource()`, `paramSource()` → `VariableSource`, and `toResolver()` → `VariableResolver`, `addResult(name, StepResult)`, `resolve(prefixedRef)` → `Object`
- Produces: `StepDef` record (`primitiveName`, `parameters`, `resultName`, `when`, `onError`, `maxRetries`, `backoff`)
- Produces: `CompoundStepDef` record (`name`, `parameters` (Map of PluginFieldDef-equivalent), `steps`, `resultBinding`)
- Produces: `ExpressionEvaluator.evaluate(String, Map<String,Object>)` → `boolean`

- [ ] **Step 1: Create pom.xml for yaml-step-core**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-platform</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-platform-yaml-step-core</artifactId>
    <name>CaseHub Platform :: YAML Step Core</name>
    <description>Generic step pipeline infrastructure — SPI, executor, registry,
        expression evaluator, and built-in primitives.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-yaml-core</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>

        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-api</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

Add `<module>yaml-step-core</module>` to platform root `pom.xml`.

- [ ] **Step 2: Write StepContextTest — VariableSource integration**

Port from `desiredstate/plugin/api/src/test/java/.../StepContextTest.java`.
Add new test for `toResolver()` → `VariableResolver` integration:

```java
package io.casehub.yaml.step;

import io.casehub.yaml.core.resolver.VariableResolver;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class StepContextTest {

    @Test
    void specSource_resolvesFlatField() {
        var ctx = StepContext.builder()
            .spec(java.util.Map.of("namespace", "prod"))
            .build();
        assertThat(ctx.specSource().resolve("namespace")).isEqualTo("prod");
    }

    @Test
    void resultSource_reflectsMutableAdditions() {
        var ctx = StepContext.builder().build();
        ctx.addResult("r1", StepResult.of(java.util.Map.of("status", 200)));
        assertThat(ctx.resultSource().resolve("r1.status")).isEqualTo("200");
    }

    @Test
    void toResolver_interpolatesTemplate() {
        var ctx = StepContext.builder()
            .spec(java.util.Map.of("name", "my-app"))
            .build();
        VariableResolver resolver = ctx.toResolver();
        String resolved = resolver.resolveString("app-${spec.name}", "<test>");
        assertThat(resolved).isEqualTo("app-my-app");
    }

    @Test
    void toResolver_withDomainScope() {
        var ctx = StepContext.builder()
            .spec(java.util.Map.of("name", "app"))
            .build();
        VariableResolver resolver = ctx.toResolver()
            .withScope("var", name -> "v1".equals(name) ? "hello" : null);
        assertThat(resolver.resolveString("${var.v1}", "<test>")).isEqualTo("hello");
        assertThat(resolver.resolveString("${spec.name}", "<test>")).isEqualTo("app");
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl yaml-step-core -Dtest=StepContextTest`
Expected: FAIL — classes don't exist yet.

- [ ] **Step 4: Create SPI types**

Copy from desiredstate plugin/api, changing package to `io.casehub.yaml.step`:

- `StepPrimitive.java` — interface, change package only
- `StepResult.java` — value type, change package only
- `StepParameters.java` — value type, change package only
- `StepDef.java` — renamed from `PluginStepDef`, change package
- `CompoundStepDef.java` — renamed from `CompoundPrimitiveDef`, uses `Map<String, Object>` for parameters (field definitions stay in desiredstate's plugin model), uses `StepDef` for steps
- `StepExecutionException.java` — rename `pluginType` field to `pipelineId`
- `InterpolationException.java` — change package only
- `ExpressionEvaluator.java` — change package to `io.casehub.yaml.step.expr`
- `ExpressionParseException.java` — change package to `io.casehub.yaml.step.expr`

- [ ] **Step 5: Create StepContext with VariableSource factory methods**

```java
package io.casehub.yaml.step;

import io.casehub.yaml.core.resolver.VariableResolver;
import io.casehub.yaml.core.resolver.VariableSource;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;

public final class StepContext {

    private final Map<String, Object> spec;
    private final Map<String, Map<String, String>> auth;
    private final Map<String, StepResult> results;
    private final Map<String, Object> params;

    private StepContext(Map<String, Object> spec,
                        Map<String, Map<String, String>> auth,
                        Map<String, StepResult> results,
                        Map<String, Object> params) {
        this.spec = Collections.unmodifiableMap(spec);
        this.auth = Collections.unmodifiableMap(auth);
        this.results = new HashMap<>(results);
        this.params = Collections.unmodifiableMap(params);
    }

    public VariableSource specSource() {
        return name -> {
            Object value = resolveDeep(spec, name);
            return value != null ? value.toString() : null;
        };
    }

    public VariableSource authSource() {
        return name -> {
            Object value = resolveAuth(name);
            return value != null ? value.toString() : null;
        };
    }

    public VariableSource resultSource() {
        return name -> {
            Object value = resolveResult(name);
            return value != null ? value.toString() : null;
        };
    }

    public VariableSource paramSource() {
        return name -> {
            Object value = params.get(name);
            return value != null ? value.toString() : null;
        };
    }

    public VariableResolver toResolver() {
        return new VariableResolver(
            Map.of("spec", specSource(), "auth", authSource(),
                   "result", resultSource(), "param", paramSource()),
            Set.of());
    }

    public Map<String, Object> spec() { return spec; }

    public Map<String, String> auth(String name) {
        return auth.getOrDefault(name, Map.of());
    }

    public Map<String, Map<String, String>> allAuth() { return auth; }

    public StepResult result(String name) { return results.get(name); }

    public void addResult(String name, StepResult result) {
        results.put(name, result);
    }

    public Map<String, Object> params() { return params; }

    public Object resolve(String prefixedRef) {
        int dot = prefixedRef.indexOf('.');
        if (dot < 0) {
            throw new IllegalArgumentException(
                "Reference must be prefixed: " + prefixedRef);
        }
        String prefix = prefixedRef.substring(0, dot);
        String remainder = prefixedRef.substring(dot + 1);
        return switch (prefix) {
            case "spec" -> resolveDeep(spec, remainder);
            case "auth" -> resolveAuth(remainder);
            case "result" -> resolveResult(remainder);
            case "param" -> params.get(remainder);
            default -> throw new IllegalArgumentException(
                "Unknown prefix '" + prefix + "' in reference: " + prefixedRef);
        };
    }

    @SuppressWarnings("unchecked")
    private Object resolveDeep(Map<String, Object> map, String dotPath) {
        String[] segments = dotPath.split("\\.");
        Object current = map;
        for (String segment : segments) {
            if (current instanceof Map<?, ?> m) {
                current = m.get(segment);
            } else {
                return null;
            }
        }
        return current;
    }

    private Object resolveAuth(String remainder) {
        int dot = remainder.indexOf('.');
        if (dot < 0) return auth.get(remainder);
        String authName = remainder.substring(0, dot);
        String key = remainder.substring(dot + 1);
        Map<String, String> creds = auth.get(authName);
        return creds != null ? creds.get(key) : null;
    }

    private Object resolveResult(String remainder) {
        int dot = remainder.indexOf('.');
        if (dot < 0) {
            StepResult r = results.get(remainder);
            return r != null ? r.data() : null;
        }
        String resultName = remainder.substring(0, dot);
        String path = remainder.substring(dot + 1);
        StepResult r = results.get(resultName);
        return r != null ? r.get(path) : null;
    }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private Map<String, Object> spec = Map.of();
        private final Map<String, Map<String, String>> auth = new HashMap<>();
        private final Map<String, StepResult> results = new HashMap<>();
        private Map<String, Object> params = Map.of();

        public Builder spec(Map<String, Object> spec) { this.spec = spec; return this; }
        public Builder addAuth(String name, Map<String, String> creds) { auth.put(name, creds); return this; }
        public Builder addResult(String name, StepResult result) { results.put(name, result); return this; }
        public Builder params(Map<String, Object> params) { this.params = params; return this; }
        public StepContext build() { return new StepContext(spec, auth, results, params); }
    }
}
```

- [ ] **Step 6: Write ExpressionEvaluatorTest**

Port from `desiredstate/plugin/api/src/test/java/.../expr/ExpressionEvaluatorTest.java`,
changing package to `io.casehub.yaml.step.expr`.

- [ ] **Step 7: Run all tests and verify they pass**

Run: `mvn --batch-mode test -pl yaml-step-core`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add yaml-step-core/ pom.xml
git commit -m "feat(ops#88): add yaml-step-core module — SPI types, StepContext, ExpressionEvaluator

Refs casehubio/casehub-ops#88"
```

### Task 2: Add executor, registry, and expander

**Files:**
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/StepPipelineExecutor.java`
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/PrimitiveRegistry.java`
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/CompoundStepExpander.java`
- Test: `yaml-step-core/src/test/java/io/casehub/yaml/step/StepPipelineExecutorTest.java`
- Test: `yaml-step-core/src/test/java/io/casehub/yaml/step/CompoundStepExpanderTest.java`

**Interfaces:**
- Consumes: `StepPrimitive`, `StepResult`, `StepParameters`, `StepDef`, `StepContext` (from Task 1)
- Consumes: `VariableResolver` (from yaml-core — `resolveString()`, `resolveMap()`)
- Produces: `StepPipelineExecutor(PrimitiveRegistry)` with `execute(List<StepDef>, StepContext, VariableResolver) → StepResult`
- Produces: `PrimitiveRegistry.of(Map<String, StepPrimitive>)` with `resolve(name) → StepPrimitive`, `contains(name) → boolean`
- Produces: `CompoundStepExpander(Map<String, CompoundStepDef>, Set<String> javaPrimitives)` with `expand(StepDef) → List<StepDef>`

- [ ] **Step 1: Write StepPipelineExecutorTest**

Port from `desiredstate/plugin/runtime/src/test/java/.../StepPipelineExecutorTest.java`.
Key change: `execute()` now takes `(steps, context, resolver)`. Tests construct
resolver via `context.toResolver()`.

```java
package io.casehub.yaml.step;

import io.casehub.yaml.core.resolver.VariableResolver;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class StepPipelineExecutorTest {

    private final StepPrimitive echoPrimitive = new StepPrimitive() {
        @Override public String name() { return "echo"; }
        @Override public StepResult execute(StepParameters params, StepContext ctx) {
            return StepResult.of(Map.of("echoed", params.getString("value")));
        }
    };

    @Test
    void executesSequentialSteps() {
        PrimitiveRegistry registry = PrimitiveRegistry.of(Map.of("echo", echoPrimitive));
        StepPipelineExecutor executor = new StepPipelineExecutor(registry);
        StepContext ctx = StepContext.builder()
            .spec(Map.of("name", "test"))
            .build();
        VariableResolver resolver = ctx.toResolver();
        List<StepDef> steps = List.of(
            new StepDef("echo", Map.of("value", "${spec.name}"), "r1",
                null, null, 0, null)
        );
        StepResult result = executor.execute(steps, ctx, resolver);
        assertThat(ctx.result("r1").get("echoed")).isEqualTo("test");
    }

    @Test
    void skipsStep_whenConditionFalse() {
        PrimitiveRegistry registry = PrimitiveRegistry.of(Map.of("echo", echoPrimitive));
        StepPipelineExecutor executor = new StepPipelineExecutor(registry);
        StepContext ctx = StepContext.builder().spec(Map.of("skip", "true")).build();
        VariableResolver resolver = ctx.toResolver();
        List<StepDef> steps = List.of(
            new StepDef("echo", Map.of("value", "x"), "r1",
                "${spec.skip} == false", null, 0, null)
        );
        executor.execute(steps, ctx, resolver);
        assertThat(ctx.result("r1")).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl yaml-step-core -Dtest=StepPipelineExecutorTest`
Expected: FAIL

- [ ] **Step 3: Implement StepPipelineExecutor**

Port from `desiredstate/plugin/runtime/src/main/java/.../StepPipelineExecutor.java`.
Key changes:
- Constructor takes `PrimitiveRegistry` only (no interpolator)
- `execute(List<StepDef>, StepContext, VariableResolver)` — resolver passed in
- Condition evaluation: `resolver.resolveString(condition, "step-" + i)` → `ExpressionEvaluator.evaluate()`
- Parameter resolution: `resolver.resolveMap(step.parameters(), "step-" + i)` before passing to primitive
- Uses `StepDef` instead of `PluginStepDef`
- No `executeActualState()` method

```java
package io.casehub.yaml.step;

import io.casehub.yaml.core.resolver.VariableResolver;
import io.casehub.yaml.step.expr.ExpressionEvaluator;

import java.util.List;
import java.util.Map;

public class StepPipelineExecutor {

    private final PrimitiveRegistry registry;

    public StepPipelineExecutor(PrimitiveRegistry registry) {
        this.registry = registry;
    }

    public StepResult execute(List<StepDef> steps, StepContext context,
                              VariableResolver resolver) {
        StepResult lastResult = StepResult.empty();
        for (int i = 0; i < steps.size(); i++) {
            StepDef step = steps.get(i);
            if (step.when() != null) {
                String resolved = resolver.resolveString(step.when(), "step-" + i);
                if (!ExpressionEvaluator.evaluate(resolved, Map.of())) {
                    continue;
                }
            }
            StepPrimitive primitive = registry.resolve(step.primitiveName());
            Map<String, Object> resolvedParams =
                resolver.resolveMap(step.parameters(), "step-" + i);
            StepParameters params = StepParameters.of(resolvedParams);
            lastResult = executeWithRetry(primitive, params, context, step, i);
            if (step.resultName() != null) {
                context.addResult(step.resultName(), lastResult);
            }
        }
        return lastResult;
    }

    private StepResult executeWithRetry(StepPrimitive primitive,
                                        StepParameters params,
                                        StepContext context,
                                        StepDef step, int stepIndex) {
        String onError = step.onError();
        int maxRetries = step.maxRetries();
        String backoff = step.backoff();
        if (!"retry".equals(onError)) {
            return executeSingle(primitive, params, context, step, stepIndex);
        }
        StepExecutionException lastError = null;
        for (int attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                return executeSingle(primitive, params, context, step, stepIndex);
            } catch (StepExecutionException e) {
                lastError = e;
                if (attempt < maxRetries) { applyBackoff(backoff, attempt); }
            }
        }
        throw lastError;
    }

    private StepResult executeSingle(StepPrimitive primitive,
                                     StepParameters params,
                                     StepContext context,
                                     StepDef step, int stepIndex) {
        try {
            return primitive.execute(params, context);
        } catch (RuntimeException e) {
            if ("skip".equals(step.onError())) { return StepResult.empty(); }
            throw new StepExecutionException(
                "Step " + stepIndex + " (" + step.primitiveName() + ") failed: "
                    + e.getMessage(),
                e, null, stepIndex, step.primitiveName());
        }
    }

    private void applyBackoff(String backoff, int attempt) {
        long delayMs;
        if (backoff != null && backoff.startsWith("exponential:")) {
            long base = parseDurationMs(backoff.substring("exponential:".length()));
            delayMs = base * (1L << attempt);
        } else if (backoff != null && backoff.startsWith("fixed:")) {
            delayMs = parseDurationMs(backoff.substring("fixed:".length()));
        } else {
            delayMs = 1000;
        }
        try { Thread.sleep(delayMs); }
        catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new StepExecutionException("Retry interrupted");
        }
    }

    private long parseDurationMs(String duration) {
        if (duration.endsWith("ms"))
            return Long.parseLong(duration.substring(0, duration.length() - 2));
        if (duration.endsWith("s"))
            return Long.parseLong(duration.substring(0, duration.length() - 1)) * 1000;
        return Long.parseLong(duration);
    }
}
```

- [ ] **Step 4: Implement PrimitiveRegistry**

Port from desiredstate, change package. Uses `StepExecutionException` for unknown primitives.

- [ ] **Step 5: Write CompoundStepExpanderTest and implement**

Port from `desiredstate/.../CompoundPrimitiveExpanderTest.java`. Rename classes,
use `StepDef`/`CompoundStepDef`. Verify cycle detection and max depth.

- [ ] **Step 6: Run all tests**

Run: `mvn --batch-mode test -pl yaml-step-core`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add yaml-step-core/
git commit -m "feat(ops#88): add StepPipelineExecutor, PrimitiveRegistry, CompoundStepExpander

Refs casehubio/casehub-ops#88"
```

### Task 3: Add generic primitives

**Files:**
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/primitives/RestCallPrimitive.java`
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/primitives/JsonExtractPrimitive.java`
- Create: `yaml-step-core/src/main/java/io/casehub/yaml/step/primitives/AssertPrimitive.java`
- Test: `yaml-step-core/src/test/java/io/casehub/yaml/step/primitives/AssertPrimitiveTest.java`

**Interfaces:**
- Consumes: `StepPrimitive`, `StepResult`, `StepParameters`, `StepContext` (from Task 1)
- Produces: `RestCallPrimitive` (name: `"rest-call"`), `JsonExtractPrimitive` (name: `"json-extract"`), `AssertPrimitive` (name: `"assert"`)

- [ ] **Step 1: Write AssertPrimitiveTest**

Port from `desiredstate/.../primitives/AssertPrimitiveTest.java`.

Key change: primitives no longer self-interpolate. Parameters arrive pre-resolved
from the executor. Tests pass raw (already resolved) values:

```java
package io.casehub.yaml.step.primitives;

import io.casehub.yaml.step.*;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class AssertPrimitiveTest {

    private final AssertPrimitive prim = new AssertPrimitive();

    @Test
    void passesWhenConditionTrue() {
        StepParameters params = StepParameters.of(Map.of("condition", "200 == 200"));
        StepContext ctx = StepContext.builder().build();
        StepResult result = prim.execute(params, ctx);
        assertThat(result.get("passed")).isEqualTo(true);
    }

    @Test
    void throwsWhenConditionFalse() {
        StepParameters params = StepParameters.of(Map.of(
            "condition", "404 == 200",
            "message", "Expected 200"));
        StepContext ctx = StepContext.builder().build();
        assertThatThrownBy(() -> prim.execute(params, ctx))
            .isInstanceOf(StepExecutionException.class)
            .hasMessage("Expected 200");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl yaml-step-core -Dtest=AssertPrimitiveTest`
Expected: FAIL

- [ ] **Step 3: Implement generic primitives**

Port from desiredstate, remove all `PluginInterpolator` usage. Parameters arrive
pre-resolved from the executor.

**AssertPrimitive** — evaluates condition via `ExpressionEvaluator.evaluate()` directly
on the already-resolved condition string. No interpolation needed.

**RestCallPrimitive** — uses pre-resolved params directly. No `interpolator.interpolate()`
calls. Constructor takes `HttpClient` only (no interpolator). Auth lookup still
uses `context.auth(authName)`.

**JsonExtractPrimitive** — `context.resolve(inputRef)` for object access remains.
Path is already resolved (no interpolation). No interpolator field.

- [ ] **Step 4: Run full test suite**

Run: `mvn --batch-mode test -pl yaml-step-core`
Expected: PASS

- [ ] **Step 5: Install to local Maven repo**

Run: `mvn --batch-mode install -pl yaml-step-core`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add yaml-step-core/
git commit -m "feat(ops#88): add generic primitives — rest-call, json-extract, assert

Refs casehubio/casehub-ops#88"
```

---

## Batch 2: Desiredstate — dependency swap and import migration

### Task 4: Add yaml-step-core dependency and update plugin/api

**Files:**
- Modify: `plugin/api/pom.xml` — add yaml-step-core dependency
- Delete: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/StepPrimitive.java` (use `ide_refactor_safe_delete`)
- Delete: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/StepResult.java`
- Delete: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/StepParameters.java`
- Delete: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/StepContext.java`
- Delete: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/PluginInterpolator.java`
- Delete: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/InterpolationException.java`
- Delete: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/expr/ExpressionEvaluator.java`
- Delete: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/expr/ExpressionParseException.java`
- Delete: `plugin/api/src/test/java/io/casehub/desiredstate/plugin/api/StepContextTest.java`
- Delete: `plugin/api/src/test/java/io/casehub/desiredstate/plugin/api/PluginInterpolatorTest.java`
- Delete: `plugin/api/src/test/java/io/casehub/desiredstate/plugin/api/expr/ExpressionEvaluatorTest.java`
- Keep: `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/YamlNodeSpec.java`
- Keep: `plugin/api/src/test/java/io/casehub/desiredstate/plugin/api/YamlNodeSpecTest.java`

**Interfaces:**
- Consumes: `casehub-platform-yaml-step-core` (all SPI types re-exported transitively)
- Produces: `YamlNodeSpec` (unchanged, domain type)

- [ ] **Step 1: Add yaml-step-core dependency to plugin/api pom.xml**

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-yaml-step-core</artifactId>
    <version>${casehub-platform.version}</version>
</dependency>
```

- [ ] **Step 2: Delete migrated source files from plugin/api**

Use `ide_refactor_safe_delete` for each file to check for remaining references:
- `StepPrimitive.java`, `StepResult.java`, `StepParameters.java`, `StepContext.java`
- `PluginInterpolator.java`, `InterpolationException.java`
- `expr/ExpressionEvaluator.java`, `expr/ExpressionParseException.java`

Safe-delete will flag references. Update imports in referencing files from
`io.casehub.desiredstate.plugin.api.*` → `io.casehub.yaml.step.*`.

- [ ] **Step 3: Delete migrated test files**

Delete `StepContextTest.java`, `PluginInterpolatorTest.java`,
`ExpressionEvaluatorTest.java` — these are now in yaml-step-core.

- [ ] **Step 4: Verify YamlNodeSpec still compiles**

`YamlNodeSpec` imports `NodeSpec`, `NodeType`, `HumanGating` from desiredstate-api.
It should NOT import any `io.casehub.yaml.step.*` types. Verify:

Run: `mvn --batch-mode compile -pl plugin/api`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add plugin/api/
git commit -m "refactor(ops#88): swap plugin/api SPI types for yaml-step-core dependency

Delete local StepPrimitive, StepResult, StepParameters, StepContext,
PluginInterpolator, ExpressionEvaluator. Re-exported transitively
via yaml-step-core. YamlNodeSpec remains as sole domain type.

Refs casehubio/casehub-ops#88"
```

### Task 5: Migrate plugin/runtime — imports, renames, domain adapter

**Files:**
- Delete: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/StepPipelineExecutor.java`
- Delete: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PrimitiveRegistry.java`
- Delete: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/CompoundPrimitiveExpander.java`
- Delete: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/StepExecutionException.java`
- Delete: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/primitives/RestCallPrimitive.java`
- Delete: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/primitives/JsonExtractPrimitive.java`
- Delete: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/primitives/AssertPrimitive.java`
- Delete: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginStepDef.java`
- Delete: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/CompoundPrimitiveDef.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/ActualStateStepExecutor.java`
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/YamlPluginProvisioner.java`
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/YamlPluginActualStateAdapter.java`
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginDescriptor.java`
- Modify: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginParser.java`
- Delete: `plugin/runtime/src/test/java/.../StepPipelineExecutorTest.java`
- Delete: `plugin/runtime/src/test/java/.../CompoundPrimitiveExpanderTest.java`
- Delete: `plugin/runtime/src/test/java/.../primitives/AssertPrimitiveTest.java`
- Test: `plugin/runtime/src/test/java/.../ActualStateStepExecutorTest.java` (new)

**Interfaces:**
- Consumes: `io.casehub.yaml.step.StepPipelineExecutor`, `PrimitiveRegistry`, `CompoundStepExpander`, `StepDef`, `CompoundStepDef`, `StepContext` (from yaml-step-core)
- Consumes: `io.casehub.yaml.core.resolver.VariableResolver` (from yaml-core)
- Produces: `ActualStateStepExecutor` — wraps `StepPipelineExecutor`, maps `StepResult` → `NodeStatus`

- [ ] **Step 1: Write ActualStateStepExecutorTest**

```java
package io.casehub.desiredstate.plugin.runtime;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.yaml.step.*;
import io.casehub.yaml.core.resolver.VariableResolver;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;

class ActualStateStepExecutorTest {

    @Test
    void mapsStepResultToNodeStatus() {
        StepPrimitive statusPrimitive = new StepPrimitive() {
            @Override public String name() { return "status"; }
            @Override public StepResult execute(StepParameters p, StepContext c) {
                return StepResult.of(Map.of("nodeStatus", "PRESENT"));
            }
        };
        PrimitiveRegistry registry = PrimitiveRegistry.of(Map.of("status", statusPrimitive));
        StepPipelineExecutor executor = new StepPipelineExecutor(registry);
        ActualStateStepExecutor adapter = new ActualStateStepExecutor(executor);

        StepContext ctx = StepContext.builder().build();
        VariableResolver resolver = ctx.toResolver();
        List<StepDef> steps = List.of(
            new StepDef("status", Map.of(), "r", null, null, 0, null));

        NodeStatus result = adapter.execute(steps, ctx, resolver);
        assertThat(result).isEqualTo(NodeStatus.PRESENT);
    }

    @Test
    void returnsUnknown_onException() {
        StepPrimitive failing = new StepPrimitive() {
            @Override public String name() { return "fail"; }
            @Override public StepResult execute(StepParameters p, StepContext c) {
                throw new StepExecutionException("boom");
            }
        };
        PrimitiveRegistry registry = PrimitiveRegistry.of(Map.of("fail", failing));
        StepPipelineExecutor executor = new StepPipelineExecutor(registry);
        ActualStateStepExecutor adapter = new ActualStateStepExecutor(executor);

        StepContext ctx = StepContext.builder().build();
        VariableResolver resolver = ctx.toResolver();
        List<StepDef> steps = List.of(
            new StepDef("fail", Map.of(), null, null, null, 0, null));

        NodeStatus result = adapter.execute(steps, ctx, resolver);
        assertThat(result).isEqualTo(NodeStatus.UNKNOWN);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl plugin/runtime -Dtest=ActualStateStepExecutorTest`
Expected: FAIL

- [ ] **Step 3: Create ActualStateStepExecutor**

```java
package io.casehub.desiredstate.plugin.runtime;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.yaml.core.resolver.VariableResolver;
import io.casehub.yaml.step.StepContext;
import io.casehub.yaml.step.StepDef;
import io.casehub.yaml.step.StepExecutionException;
import io.casehub.yaml.step.StepPipelineExecutor;
import io.casehub.yaml.step.StepResult;

import java.util.List;

final class ActualStateStepExecutor {

    private final StepPipelineExecutor executor;

    ActualStateStepExecutor(StepPipelineExecutor executor) {
        this.executor = executor;
    }

    NodeStatus execute(List<StepDef> steps, StepContext context,
                       VariableResolver resolver) {
        try {
            StepResult result = executor.execute(steps, context, resolver);
            Object status = result.get("nodeStatus");
            if (status instanceof String s) {
                return NodeStatus.valueOf(s);
            }
            return NodeStatus.UNKNOWN;
        } catch (StepExecutionException e) {
            return NodeStatus.UNKNOWN;
        }
    }
}
```

- [ ] **Step 4: Delete migrated runtime files**

Use `ide_refactor_safe_delete` for each. Update imports in remaining files:
- `StepPipelineExecutor.java`, `PrimitiveRegistry.java`, `CompoundPrimitiveExpander.java`,
  `StepExecutionException.java` → now from `io.casehub.yaml.step.*`
- `RestCallPrimitive.java`, `JsonExtractPrimitive.java`, `AssertPrimitive.java` → now from `io.casehub.yaml.step.primitives.*`
- `PluginStepDef.java` → `io.casehub.yaml.step.StepDef`
- `CompoundPrimitiveDef.java` → `io.casehub.yaml.step.CompoundStepDef`

- [ ] **Step 5: Update PluginDescriptor to use StepDef**

Change all `PluginStepDef` references to `StepDef` in `PluginDescriptor.java`.
Use `ide_refactor_rename` if IntelliJ handles the type change; otherwise
update the record component types directly.

- [ ] **Step 6: Update PluginParser to produce StepDef**

Change `PluginParser` to construct `StepDef` and `CompoundStepDef` instead of
`PluginStepDef` and `CompoundPrimitiveDef`. The record fields are identical —
only the type name and package change.

- [ ] **Step 7: Refactor YamlPluginProvisioner**

Update to construct `VariableResolver` from `StepContext` + domain sources:

```java
// Replace PluginInterpolator usage with VariableResolver construction
StepContext context = buildContext(node, ctx, plugin);
VariableResolver resolver = context.toResolver()
    .withScope("var", variableSource)   // from graph YAML variables
    .withScope("fault", faultSource);   // from fault context, if applicable

StepResult result = executor.execute(plugin.provisionSteps(), context, resolver);
```

Remove `PluginInterpolator` field. Inject or construct `StepPipelineExecutor`
with `PrimitiveRegistry` only.

- [ ] **Step 8: Refactor YamlPluginActualStateAdapter**

Use `ActualStateStepExecutor` instead of calling
`StepPipelineExecutor.executeActualState()`:

```java
ActualStateStepExecutor actualStateExecutor = new ActualStateStepExecutor(executor);
// In readActual():
StepContext context = buildContext(node, tenancyId, plugin);
VariableResolver resolver = context.toResolver();
NodeStatus status = actualStateExecutor.execute(
    plugin.actualStateSteps(), context, resolver);
```

- [ ] **Step 9: Delete migrated test files**

Delete `StepPipelineExecutorTest.java`, `CompoundPrimitiveExpanderTest.java`,
`AssertPrimitiveTest.java` — now in yaml-step-core.

- [ ] **Step 10: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS (all 28+ modules)

- [ ] **Step 11: Commit**

```bash
git add plugin/
git commit -m "refactor(ops#88): migrate plugin/runtime to yaml-step-core

Delete local StepPipelineExecutor, PrimitiveRegistry,
CompoundPrimitiveExpander, generic primitives. Add ActualStateStepExecutor
domain adapter. Refactor YamlPluginProvisioner and
YamlPluginActualStateAdapter to use VariableResolver.

Refs casehubio/casehub-ops#88"
```

---

## Batch 3: Desiredstate — deployment processor and regression verification

### Task 6: Update plugin/deployment and run full regression

**Files:**
- Modify: `plugin/deployment/src/main/java/io/casehub/desiredstate/plugin/deployment/YamlPluginProcessor.java`
- Modify: `plugin/deployment/src/test/java/.../YamlPluginProcessorTest.java`

**Interfaces:**
- Consumes: `io.casehub.yaml.step.StepDef` (replaces `PluginStepDef`)
- Consumes: `io.casehub.yaml.step.CompoundStepDef` (replaces `CompoundPrimitiveDef`)

- [ ] **Step 1: Update YamlPluginProcessor imports**

Replace all `PluginStepDef` → `StepDef` and `CompoundPrimitiveDef` → `CompoundStepDef`
references in `YamlPluginProcessor.java`. Update imports from
`io.casehub.desiredstate.plugin.model.*` → `io.casehub.yaml.step.*` for these types.

- [ ] **Step 2: Update YamlPluginProcessorTest**

Same import changes in the test file.

- [ ] **Step 3: Run deployment module tests**

Run: `mvn --batch-mode test -pl plugin/deployment`
Expected: PASS

- [ ] **Step 4: Run end-to-end plugin integration test**

Run: `mvn --batch-mode test -pl plugin/runtime -Dtest=PluginIntegrationTest`
Expected: PASS

- [ ] **Step 5: Run full project build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules, all tests green.

- [ ] **Step 6: Verify zero desiredstate imports in yaml-step-core**

```bash
grep -r "io.casehub.desiredstate" platform/yaml-step-core/src/ && echo "FAIL: domain imports found" || echo "PASS: no domain imports"
```

- [ ] **Step 7: Commit**

```bash
git add plugin/deployment/
git commit -m "refactor(ops#88): update plugin/deployment for StepDef/CompoundStepDef

Refs casehubio/casehub-ops#88"
```

## References

- `specs/issue-88-step-pipeline-yaml-core/2026-09-11-step-pipeline-consolidation-design.md` — design spec
- `specs/issue-88-step-pipeline-yaml-core/decisions.md` — D1-D5 decision log
- `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/StepPrimitive.java` — SPI interface
- `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/StepContext.java:48-65` — prefix dispatch
- `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/PluginInterpolator.java` — deletion target
- `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/expr/ExpressionEvaluator.java` — expression parser
- `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/StepPipelineExecutor.java` — executor
- `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/CompoundPrimitiveExpander.java` — expander
- `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/primitives/RestCallPrimitive.java:26` — PluginInterpolator field
- `io.casehub.yaml.core.resolver.VariableResolver` — yaml-core resolver (193 lines)
- `io.casehub.yaml.core.resolver.VariableSource` — `@FunctionalInterface`, `String resolve(String name)`
- `docs/specs/issue-128-migrate-yaml-core/2026-09-04-module-migration-design.md` — prior migration pattern
- casehubio/casehub-ops#88 — source issue
