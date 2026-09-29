# Plugin Testing Framework Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #135 — feat: plugin testing framework — test provisioners in isolation  
**Issue group:** #135

**Goal:** Build a declarative YAML-first plugin testing framework that lets YAML plugin authors write `*.test.yaml` files and run them via `mvn test` with zero Java authoring.

**Architecture:** A `plugin/testing/` module provides `PluginTestExtension` (JUnit5 `@RegisterExtension`) that discovers `*.test.yaml` files, starts test infrastructure (embedded WireMock or temp-dir sandbox), exercises the real `YamlPluginProvisioner` / `YamlPluginActualStateAdapter` production code path, and evaluates declarative assertions. Validation is extracted from `YamlPluginProcessor` into a framework-neutral `PluginValidator` in `plugin/runtime/`.

**Tech Stack:** Java 21+, JUnit5 (`@TestFactory` + `DynamicTest`), WireMock 3.x (embedded, no Docker), Jackson YAML, AssertJ

## Global Constraints

- No Quarkus dependencies in `plugin/testing/` — plain JUnit5 only
- No Testcontainers in v1 — WireMock runs embedded in-process
- Test YAML files live at `src/test/resources/META-INF/desiredstate/tests/`
- Plugin YAML files at `META-INF/desiredstate/plugins/` (existing convention)
- `PluginValidator` must remain static-method-only, no CDI, no Spring
- `YamlPluginProcessor` must delegate to `PluginValidator` — not duplicate
- `StepRunner` interface: method is `run()` returning `Result` (not `execute()`/`StepResult`)
- `DesiredNode` must use `YamlNodeSpec` (not plain `NodeSpec`) — `extractSpecFields()` pattern-matches on it

---

## Batch 1: Validation extraction — PluginValidator in plugin/runtime

### Task 1: Extract PluginValidator from YamlPluginProcessor

**Files:**
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginValidator.java`
- Create: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginValidationException.java`
- Modify: `plugin/deployment/src/main/java/io/casehub/desiredstate/plugin/deployment/YamlPluginProcessor.java`
- Modify: `plugin/deployment/src/main/java/io/casehub/desiredstate/plugin/deployment/PluginValidationException.java`
- Modify: `plugin/deployment/src/test/java/io/casehub/desiredstate/plugin/deployment/YamlPluginProcessorTest.java`
- Test: `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/PluginValidatorTest.java`

**Interfaces:**
- Produces: `PluginValidator.validatePlugin(PluginModel, Map<String,String>, Set<String>)`, `PluginValidator.BUILT_IN_PRIMITIVES`, `PluginValidator.SUPPORTED_FIELD_TYPES`, `PluginValidationException(String pluginType, String message)`

- [ ] **Step 1: Write failing test for PluginValidator**

Create `plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/PluginValidatorTest.java`:

```java
package io.casehub.desiredstate.plugin.runtime;

import io.casehub.desiredstate.plugin.model.PluginFieldDef;
import io.casehub.desiredstate.plugin.model.PluginModel;
import io.casehub.desiredstate.plugin.model.PluginHeader;
import io.casehub.desiredstate.plugin.model.PluginProvisionerDef;
import io.casehub.desiredstate.plugin.model.PluginSpecSchema;
import io.casehub.yaml.step.CatalogEntry;
import io.casehub.yaml.step.catalog.ResolvedStep;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThatCode;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class PluginValidatorTest {

    @Test
    void validPluginPassesValidation() {
        var plugin = createValidPlugin();
        assertThatCode(() ->
            PluginValidator.validatePlugin(plugin, Map.of(), PluginValidator.BUILT_IN_PRIMITIVES))
            .doesNotThrowAnyException();
    }

    @Test
    void unknownFieldTypeThrows() {
        var spec = new PluginSpecSchema(Map.of(
            "data", new PluginFieldDef("complex", false, null, null, null, null,
                null, null, null, null, null, null, null)));
        var plugin = createPlugin("bad-type", spec);

        assertThatThrownBy(() ->
            PluginValidator.validatePlugin(plugin, Map.of(), PluginValidator.BUILT_IN_PRIMITIVES))
            .isInstanceOf(PluginValidationException.class)
            .hasMessageContaining("unsupported type 'complex'");
    }

    @Test
    void unknownPrimitiveSuggestsSimilar() {
        var step = new ResolvedStep.PluginStep("rest-cll",
            new CatalogEntry("rest-cll", null, null), Map.of(), Map.of());
        var spec = new PluginSpecSchema(Map.of());
        var plugin = createPlugin("typo", spec,
            List.of(step), List.of(), List.of(step));

        assertThatThrownBy(() ->
            PluginValidator.validatePlugin(plugin, Map.of(), PluginValidator.BUILT_IN_PRIMITIVES))
            .isInstanceOf(PluginValidationException.class)
            .hasMessageContaining("rest-call");
    }

    @Test
    void actualStateMissingCompareStateThrows() {
        var step = new ResolvedStep.PluginStep("assert",
            new CatalogEntry("assert", null, null),
            Map.of("condition", "true"), Map.of());
        var spec = new PluginSpecSchema(Map.of(
            "name", new PluginFieldDef("string", true, null, null, null, null,
                null, null, null, null, null, null, null)));
        var plugin = createPlugin("no-compare", spec,
            List.of(step), List.of(step), List.of(step));

        assertThatThrownBy(() ->
            PluginValidator.validatePlugin(plugin, Map.of(), PluginValidator.BUILT_IN_PRIMITIVES))
            .isInstanceOf(PluginValidationException.class)
            .hasMessageContaining("compare-state");
    }

    @Test
    void typeConflictWithJavaNodeTypeThrows() {
        var plugin = createValidPlugin();
        var typeRegistry = Map.of("test-resource", "com.example.TestResource");

        assertThatThrownBy(() ->
            PluginValidator.validatePlugin(plugin, typeRegistry, PluginValidator.BUILT_IN_PRIMITIVES))
            .isInstanceOf(PluginValidationException.class)
            .hasMessageContaining("Type conflict");
    }

    private static PluginModel createValidPlugin() {
        return createPlugin("test-resource",
            new PluginSpecSchema(Map.of(
                "name", new PluginFieldDef("string", true, null, null, null, null,
                    null, null, null, null, null, null, null))));
    }

    private static PluginModel createPlugin(String type, PluginSpecSchema spec) {
        var compareStep = new ResolvedStep.PluginStep("compare-state",
            new CatalogEntry("compare-state", null, null),
            Map.of("present-when", "true"), Map.of());
        var assertStep = new ResolvedStep.PluginStep("assert",
            new CatalogEntry("assert", null, null),
            Map.of("condition", "true"), Map.of());
        return createPlugin(type, spec,
            List.of(compareStep), List.of(assertStep), List.of(assertStep));
    }

    private static PluginModel createPlugin(String type, PluginSpecSchema spec,
                                             List<ResolvedStep> actualState,
                                             List<ResolvedStep> provision,
                                             List<ResolvedStep> deprovision) {
        return new PluginModel(
            new PluginHeader(type, 1, "30s", Map.of()),
            spec, actualState,
            new PluginProvisionerDef(provision, deprovision),
            List.of(), null, null);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl plugin/runtime -Dtest=PluginValidatorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `PluginValidator` class does not exist

- [ ] **Step 3: Create PluginValidationException in plugin/runtime**

Create `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginValidationException.java`:

```java
package io.casehub.desiredstate.plugin.runtime;

public class PluginValidationException extends RuntimeException {

    private final String pluginType;

    public PluginValidationException(String pluginType, String message) {
        super("Plugin '" + pluginType + "': " + message);
        this.pluginType = pluginType;
    }

    public String pluginType() { return pluginType; }
}
```

- [ ] **Step 4: Create PluginValidator with extracted methods**

Create `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginValidator.java`.

Move the following static methods and constants from `YamlPluginProcessor` into `PluginValidator`:
- `BUILT_IN_PRIMITIVES`, `SUPPORTED_FIELD_TYPES`, `INTERPOLATION_REF`, `KNOWN_PREFIXES`
- `validatePlugin()`, `validateSpecSchema()`, `validateSteps()`, `validateInterpolationRefs()`, `validateInterpolationRefsInValue()`, `validateInterpolationRefsInString()`, `validateActualStateHasCompareState()`, `validateActualStateNoApprovalGate()`, `suggestSimilar()`, `levenshtein()`

Change all `PluginValidationException` references to use the new `io.casehub.desiredstate.plugin.runtime.PluginValidationException`.

```java
package io.casehub.desiredstate.plugin.runtime;

import io.casehub.desiredstate.plugin.model.PluginFieldDef;
import io.casehub.desiredstate.plugin.model.PluginModel;
import io.casehub.desiredstate.plugin.model.PluginSpecSchema;
import io.casehub.yaml.step.catalog.ResolvedStep;

import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public final class PluginValidator {

    public static final Set<String> BUILT_IN_PRIMITIVES = Set.of(
        "rest-call", "graphql-call", "json-extract", "compare-state", "assert", "approval-gate");

    public static final Set<String> SUPPORTED_FIELD_TYPES = Set.of(
        "string", "integer", "number", "boolean", "enum", "list", "map");

    static final Pattern INTERPOLATION_REF = Pattern.compile("\\$\\{([^}]+)}");

    static final Set<String> KNOWN_PREFIXES = Set.of(
        "spec", "auth", "result", "param", "var", "fault");

    private PluginValidator() {}

    // Copy all static validate* methods, suggestSimilar, levenshtein
    // from YamlPluginProcessor verbatim, changing only the
    // PluginValidationException import to the new package.
    // (Full method bodies identical to YamlPluginProcessor lines 86-303)
}
```

- [ ] **Step 5: Update YamlPluginProcessor to delegate to PluginValidator**

Modify `plugin/deployment/src/main/java/io/casehub/desiredstate/plugin/deployment/YamlPluginProcessor.java`:
- Remove all `static validate*` methods, `suggestSimilar()`, `levenshtein()`, constants (`BUILT_IN_PRIMITIVES`, `SUPPORTED_FIELD_TYPES`, `INTERPOLATION_REF`, `KNOWN_PREFIXES`)
- In `discoverAndValidatePlugins()`, change `validatePlugin(np.model, typeRegistry, BUILT_IN_PRIMITIVES)` to `PluginValidator.validatePlugin(np.model, typeRegistry, PluginValidator.BUILT_IN_PRIMITIVES)`
- Add import for `io.casehub.desiredstate.plugin.runtime.PluginValidator`
- Keep `PluginValidationException` in deployment package as a subclass extending `io.casehub.desiredstate.plugin.runtime.PluginValidationException` for backward compatibility, OR remove it and update imports (check if anything depends on the deployment-package exception)

- [ ] **Step 6: Update YamlPluginProcessorTest**

Modify `plugin/deployment/src/test/java/io/casehub/desiredstate/plugin/deployment/YamlPluginProcessorTest.java`:
- Remove `PRIMITIVES` constant, replace with `PluginValidator.BUILT_IN_PRIMITIVES`
- Change `YamlPluginProcessor.validatePlugin(...)` calls to `PluginValidator.validatePlugin(...)`
- Update `PluginValidationException` import to `io.casehub.desiredstate.plugin.runtime.PluginValidationException`

- [ ] **Step 7: Run all tests to verify extraction is correct**

Run: `mvn --batch-mode test -pl plugin/runtime,plugin/deployment`
Expected: All tests pass — both `PluginValidatorTest` and `YamlPluginProcessorTest`

- [ ] **Step 8: Commit**

```bash
git add plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginValidator.java
git add plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/runtime/PluginValidationException.java
git add plugin/runtime/src/test/java/io/casehub/desiredstate/plugin/runtime/PluginValidatorTest.java
git add plugin/deployment/src/main/java/io/casehub/desiredstate/plugin/deployment/YamlPluginProcessor.java
git add plugin/deployment/src/main/java/io/casehub/desiredstate/plugin/deployment/PluginValidationException.java
git add plugin/deployment/src/test/java/io/casehub/desiredstate/plugin/deployment/YamlPluginProcessorTest.java
git commit -m "refactor(#135): extract PluginValidator from YamlPluginProcessor to plugin/runtime"
```

---

## Batch 2: Module scaffolding and test infrastructure

### Task 2: Create plugin/testing module with POM and TestInfrastructure SPI

**Files:**
- Create: `plugin/testing/pom.xml`
- Modify: `plugin/pom.xml` (add `<module>testing</module>`)
- Create: `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/infrastructure/TestInfrastructure.java`
- Create: `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/infrastructure/InfrastructureFactory.java`
- Test: `plugin/testing/src/test/java/io/casehub/desiredstate/plugin/testing/infrastructure/InfrastructureFactoryTest.java`

**Interfaces:**
- Produces: `TestInfrastructure` interface (`start()`, `configure(List<StubExpectation>)`, `resetBetweenTests()`, `variableBindings()`, `stop()`), `InfrastructureFactory.create(String type) → TestInfrastructure`

- [ ] **Step 1: Write failing test for InfrastructureFactory**

Create `plugin/testing/src/test/java/io/casehub/desiredstate/plugin/testing/infrastructure/InfrastructureFactoryTest.java`:

```java
package io.casehub.desiredstate.plugin.testing.infrastructure;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class InfrastructureFactoryTest {

    @Test
    void createsHttpMockInfrastructure() {
        var infra = InfrastructureFactory.create("http-mock");
        assertThat(infra).isInstanceOf(HttpMockInfrastructure.class);
    }

    @Test
    void createsShellSandboxInfrastructure() {
        var infra = InfrastructureFactory.create("shell-sandbox");
        assertThat(infra).isInstanceOf(ShellSandboxInfrastructure.class);
    }

    @Test
    void unknownTypeThrows() {
        assertThatThrownBy(() -> InfrastructureFactory.create("k8s"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("k8s");
    }
}
```

- [ ] **Step 2: Create POM**

Create `plugin/testing/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-desiredstate-plugin-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-desiredstate-plugin-testing</artifactId>
    <name>CaseHub Desired State :: Plugin :: Testing</name>
    <description>YAML plugin test framework — declarative test YAML,
        embedded WireMock, shell sandbox. Test scope only.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-plugin</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-runtime-core</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-yaml-step-runtime</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-api</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.dataformat</groupId>
            <artifactId>jackson-dataformat-yaml</artifactId>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-api</artifactId>
        </dependency>
        <dependency>
            <groupId>org.wiremock</groupId>
            <artifactId>wiremock</artifactId>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

Add `<module>testing</module>` to `plugin/pom.xml` after `<module>spring</module>`.

- [ ] **Step 3: Create TestInfrastructure interface**

Create `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/infrastructure/TestInfrastructure.java`:

```java
package io.casehub.desiredstate.plugin.testing.infrastructure;

import java.util.List;
import java.util.Map;

public interface TestInfrastructure {
    void start();
    void configure(List<Map<String, Object>> expectations);
    void resetBetweenTests(List<Map<String, Object>> setupStubs);
    Map<String, Object> variableBindings();
    void stop();
}
```

- [ ] **Step 4: Create HttpMockInfrastructure stub**

Create `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/infrastructure/HttpMockInfrastructure.java`:

```java
package io.casehub.desiredstate.plugin.testing.infrastructure;

import java.util.List;
import java.util.Map;

public class HttpMockInfrastructure implements TestInfrastructure {
    @Override public void start() {}
    @Override public void configure(List<Map<String, Object>> expectations) {}
    @Override public void resetBetweenTests(List<Map<String, Object>> setupStubs) {}
    @Override public Map<String, Object> variableBindings() { return Map.of(); }
    @Override public void stop() {}
}
```

- [ ] **Step 5: Create ShellSandboxInfrastructure stub**

Create `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/infrastructure/ShellSandboxInfrastructure.java`:

```java
package io.casehub.desiredstate.plugin.testing.infrastructure;

import java.util.List;
import java.util.Map;

public class ShellSandboxInfrastructure implements TestInfrastructure {
    @Override public void start() {}
    @Override public void configure(List<Map<String, Object>> expectations) {}
    @Override public void resetBetweenTests(List<Map<String, Object>> setupStubs) {}
    @Override public Map<String, Object> variableBindings() { return Map.of(); }
    @Override public void stop() {}
}
```

- [ ] **Step 6: Create InfrastructureFactory**

Create `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/infrastructure/InfrastructureFactory.java`:

```java
package io.casehub.desiredstate.plugin.testing.infrastructure;

public final class InfrastructureFactory {

    private InfrastructureFactory() {}

    public static TestInfrastructure create(String type) {
        return switch (type) {
            case "http-mock" -> new HttpMockInfrastructure();
            case "shell-sandbox" -> new ShellSandboxInfrastructure();
            default -> throw new IllegalArgumentException(
                "Unknown infrastructure type: '" + type
                    + "'. Supported: http-mock, shell-sandbox");
        };
    }
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `mvn --batch-mode test -pl plugin/testing -Dtest=InfrastructureFactoryTest`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add plugin/testing/ plugin/pom.xml
git commit -m "feat(#135): scaffold plugin/testing module with TestInfrastructure SPI"
```

---

### Task 3: Implement HttpMockInfrastructure with embedded WireMock

**Files:**
- Modify: `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/infrastructure/HttpMockInfrastructure.java`
- Test: `plugin/testing/src/test/java/io/casehub/desiredstate/plugin/testing/infrastructure/HttpMockInfrastructureTest.java`

**Interfaces:**
- Consumes: `TestInfrastructure` interface
- Produces: Fully functional `HttpMockInfrastructure` — WireMock start/stop, stub configuration from YAML expectation maps, scenario support, reset with setup-stub preservation

- [ ] **Step 1: Write failing test for HttpMockInfrastructure**

```java
package io.casehub.desiredstate.plugin.testing.infrastructure;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class HttpMockInfrastructureTest {

    private HttpMockInfrastructure infra;

    @BeforeEach
    void setUp() {
        infra = new HttpMockInfrastructure();
        infra.start();
    }

    @AfterEach
    void tearDown() {
        infra.stop();
    }

    @Test
    void variableBindingsContainWiremockUrl() {
        var bindings = infra.variableBindings();
        assertThat(bindings).containsKey("wiremock.url");
        assertThat((String) bindings.get("wiremock.url")).startsWith("http://localhost:");
    }

    @Test
    void configureCreatesStub() throws Exception {
        infra.configure(List.of(Map.of(
            "request", Map.of("method", "GET", "path", "/api/test"),
            "response", Map.of("status", 200, "body", Map.of("ok", true))
        )));

        var client = HttpClient.newHttpClient();
        var response = client.send(
            HttpRequest.newBuilder()
                .uri(URI.create(infra.variableBindings().get("wiremock.url") + "/api/test"))
                .GET().build(),
            HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).contains("\"ok\"");
    }

    @Test
    void resetPreservesSetupStubs() throws Exception {
        var setupStub = Map.<String, Object>of(
            "request", Map.of("method", "GET", "path", "/setup"),
            "response", Map.of("status", 200, "body", Map.of("setup", true))
        );
        infra.configure(List.of(
            setupStub,
            Map.of("request", Map.of("method", "GET", "path", "/test-only"),
                   "response", Map.of("status", 200))
        ));

        infra.resetBetweenTests(List.of(setupStub));

        var client = HttpClient.newHttpClient();
        var setupResp = client.send(
            HttpRequest.newBuilder()
                .uri(URI.create(infra.variableBindings().get("wiremock.url") + "/setup"))
                .GET().build(),
            HttpResponse.BodyHandlers.ofString());
        assertThat(setupResp.statusCode()).isEqualTo(200);
    }

    @Test
    void scenarioSupport() throws Exception {
        infra.configure(List.of(
            Map.of(
                "scenario", "create-flow",
                "when-state", "Started",
                "set-state", "Created",
                "request", Map.of("method", "POST", "path", "/resources"),
                "response", Map.of("status", 201, "body", Map.of("id", "r1"))
            ),
            Map.of(
                "scenario", "create-flow",
                "when-state", "Created",
                "request", Map.of("method", "GET", "path", "/resources/r1"),
                "response", Map.of("status", 200, "body", Map.of("status", "active"))
            )
        ));

        var client = HttpClient.newHttpClient();
        var base = infra.variableBindings().get("wiremock.url");

        var createResp = client.send(
            HttpRequest.newBuilder().uri(URI.create(base + "/resources"))
                .POST(HttpRequest.BodyPublishers.ofString("{}")).build(),
            HttpResponse.BodyHandlers.ofString());
        assertThat(createResp.statusCode()).isEqualTo(201);

        var getResp = client.send(
            HttpRequest.newBuilder().uri(URI.create(base + "/resources/r1"))
                .GET().build(),
            HttpResponse.BodyHandlers.ofString());
        assertThat(getResp.statusCode()).isEqualTo(200);
        assertThat(getResp.body()).contains("active");
    }
}
```

- [ ] **Step 2: Implement HttpMockInfrastructure**

Replace the stub implementation with the full WireMock-backed version. Use `WireMockServer` directly (not `WireMockExtension`, since the extension's lifecycle is managed by `TestInfrastructure`, not JUnit).

Key behaviors:
- `start()`: create and start `WireMockServer` on random port
- `configure(expectations)`: translate YAML expectation maps to `stubFor()` calls, including scenario support (`inScenario`, `whenScenarioStateIs`, `willSetStateTo`)
- `resetBetweenTests(setupStubs)`: `resetAll()`, then re-apply setup stubs
- `variableBindings()`: return `{"wiremock.url": "http://localhost:<port>"}`
- `stop()`: stop the WireMock server

- [ ] **Step 3: Run tests**

Run: `mvn --batch-mode test -pl plugin/testing -Dtest=HttpMockInfrastructureTest`
Expected: All 4 tests pass

- [ ] **Step 4: Commit**

```bash
git add plugin/testing/src/
git commit -m "feat(#135): implement HttpMockInfrastructure with embedded WireMock"
```

---

### Task 4: Implement ShellSandboxInfrastructure

**Files:**
- Modify: `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/infrastructure/ShellSandboxInfrastructure.java`
- Test: `plugin/testing/src/test/java/io/casehub/desiredstate/plugin/testing/infrastructure/ShellSandboxInfrastructureTest.java`

**Interfaces:**
- Consumes: `TestInfrastructure` interface
- Produces: Fully functional `ShellSandboxInfrastructure` — temp dir creation, variable bindings, file preservation between tests, cleanup on stop

- [ ] **Step 1: Write failing test**

```java
package io.casehub.desiredstate.plugin.testing.infrastructure;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class ShellSandboxInfrastructureTest {

    private ShellSandboxInfrastructure infra;

    @BeforeEach
    void setUp() {
        infra = new ShellSandboxInfrastructure();
        infra.start();
    }

    @AfterEach
    void tearDown() {
        infra.stop();
    }

    @Test
    void variableBindingsContainSandboxDir() {
        var bindings = infra.variableBindings();
        assertThat(bindings).containsKey("sandbox.dir");
        assertThat(Path.of((String) bindings.get("sandbox.dir"))).exists().isDirectory();
    }

    @Test
    void resetPreservesFiles() throws IOException {
        var sandboxDir = Path.of((String) infra.variableBindings().get("sandbox.dir"));
        var testFile = sandboxDir.resolve("test.txt");
        Files.createDirectories(testFile.getParent());
        Files.writeString(testFile, "hello");

        infra.resetBetweenTests(List.of());

        assertThat(testFile).exists().hasContent("hello");
    }

    @Test
    void stopDeletesSandboxDir() {
        var sandboxDir = Path.of((String) infra.variableBindings().get("sandbox.dir"));
        assertThat(sandboxDir).exists();

        infra.stop();

        assertThat(sandboxDir).doesNotExist();
        infra = null; // prevent double-stop in tearDown
    }
}
```

- [ ] **Step 2: Implement ShellSandboxInfrastructure**

Replace stub with temp-dir implementation:
- `start()`: create temp directory via `Files.createTempDirectory("plugin-test-")`
- `variableBindings()`: return `{"sandbox.dir": tempDir.toString()}`
- `resetBetweenTests()`: no-op (preserve files between tests)
- `stop()`: recursively delete temp directory

- [ ] **Step 3: Run tests**

Run: `mvn --batch-mode test -pl plugin/testing -Dtest=ShellSandboxInfrastructureTest`
Expected: All 3 tests pass

- [ ] **Step 4: Commit**

```bash
git add plugin/testing/src/
git commit -m "feat(#135): implement ShellSandboxInfrastructure with temp directory"
```

---

## Batch 3: Test YAML parsing and test runner

### Task 5: Test YAML parser — PluginTestSuite model and Jackson parser

**Files:**
- Create: `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/PluginTestSuite.java`
- Create: `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/PluginTestCase.java`
- Create: `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/PluginTestYamlParser.java`
- Create: `plugin/testing/src/test/resources/META-INF/desiredstate/tests/parser-test.test.yaml`
- Test: `plugin/testing/src/test/java/io/casehub/desiredstate/plugin/testing/PluginTestYamlParserTest.java`

**Interfaces:**
- Produces: `PluginTestSuite` record (`pluginType`, `infrastructureType`, `setup`, `testCases`), `PluginTestCase` record (`name`, `spec`, `expectations`, `action`, `assertions`, `faultInjection`), `PluginTestYamlParser.parse(InputStream) → PluginTestSuite`

- [ ] **Step 1: Write failing test for parser**

Create test YAML at `plugin/testing/src/test/resources/META-INF/desiredstate/tests/parser-test.test.yaml`:

```yaml
plugin: mock-resource
infrastructure: http-mock

setup:
  stubs:
    - request:
        method: GET
        path: /auth
      response:
        status: 200
        body:
          token: test-token
  variables:
    auth:
      api:
        endpoint: "${wiremock.url}"

tests:
  - name: provision succeeds
    spec:
      name: test-app
      status-code: 200
    expectations:
      - request:
          method: POST
          path: /resources
        response:
          status: 201
    action: provision
    assert:
      provision: success

  - name: actual state is present
    spec:
      name: test-app
      status-code: 200
    action: actual-state
    assert:
      actual-state: PRESENT

  - name: validation catches missing name
    spec:
      status-code: 200
    action: validate
    assert:
      error-matches: "name is required"

  - name: fault injection
    spec:
      name: test-app
    fault-injection:
      action: provision
      fail-count: 3
      error: "Connection refused"
    assert:
      provision: failed
      error-matches: "Connection refused"
```

Write parser test:

```java
package io.casehub.desiredstate.plugin.testing;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class PluginTestYamlParserTest {

    @Test
    void parsesTestYaml() throws Exception {
        try (var is = getClass().getResourceAsStream(
                "/META-INF/desiredstate/tests/parser-test.test.yaml")) {
            var suite = PluginTestYamlParser.parse(is);

            assertThat(suite.pluginType()).isEqualTo("mock-resource");
            assertThat(suite.infrastructureType()).isEqualTo("http-mock");
            assertThat(suite.setup().stubs()).hasSize(1);
            assertThat(suite.setup().variables()).containsKey("auth");
            assertThat(suite.testCases()).hasSize(4);

            var provisionTest = suite.testCases().get(0);
            assertThat(provisionTest.name()).isEqualTo("provision succeeds");
            assertThat(provisionTest.action()).isEqualTo("provision");
            assertThat(provisionTest.assertions().get("provision")).isEqualTo("success");
            assertThat(provisionTest.expectations()).hasSize(1);

            var faultTest = suite.testCases().get(3);
            assertThat(faultTest.faultInjection()).isNotNull();
            assertThat(faultTest.faultInjection().failCount()).isEqualTo(3);
            assertThat(faultTest.faultInjection().error()).isEqualTo("Connection refused");
        }
    }
}
```

- [ ] **Step 2: Implement model records and parser**

Create `PluginTestSuite`, `PluginTestCase` (with nested `Setup`, `FaultInjection` records), and `PluginTestYamlParser` using Jackson `ObjectMapper` with `YAMLFactory`. The parser is a straightforward Jackson `readTree` + manual field extraction (same pattern as `PluginParser`).

- [ ] **Step 3: Run tests**

Run: `mvn --batch-mode test -pl plugin/testing -Dtest=PluginTestYamlParserTest`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add plugin/testing/src/
git commit -m "feat(#135): implement test YAML parser with PluginTestSuite model"
```

---

### Task 6: PluginTestRunner — execute actions against production code

**Files:**
- Create: `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/PluginTestRunner.java`
- Create: `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/TestCredentialResolver.java`
- Create: `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/FaultInjectingStepRunner.java`
- Test: `plugin/testing/src/test/java/io/casehub/desiredstate/plugin/testing/PluginTestRunnerTest.java`

**Interfaces:**
- Consumes: `PluginDescriptor`, `PluginTestCase`, `TestInfrastructure.variableBindings()`
- Produces: `PluginTestRunner.runProvision(...) → ProvisionResult`, `runDeprovision(...) → DeprovisionResult`, `runActualState(...) → NodeStatus`, `runValidation(...) → Optional<PluginValidationException>`

- [ ] **Step 1: Write failing tests**

Test the runner using the existing `mock-resource.yaml` plugin from `plugin/runtime/src/test/resources/`:

```java
package io.casehub.desiredstate.plugin.testing;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.ProvisionResult;
import io.casehub.desiredstate.api.DeprovisionResult;
import io.casehub.desiredstate.plugin.model.PluginModel;
import io.casehub.desiredstate.plugin.model.PluginParser;
import io.casehub.desiredstate.plugin.runtime.PluginDescriptor;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.time.Duration;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class PluginTestRunnerTest {

    private static PluginDescriptor descriptor;

    @BeforeAll
    static void loadPlugin() throws IOException {
        try (var is = PluginTestRunnerTest.class.getResourceAsStream(
                "/META-INF/desiredstate/plugins/mock-resource.yaml")) {
            PluginModel model = PluginParser.parse(is);
            descriptor = toDescriptor(model);
        }
    }

    @Test
    void provisionSucceeds() {
        var runner = new PluginTestRunner(descriptor, Map.of());
        var result = runner.runProvision(Map.of("name", "app1", "status-code", 200));
        assertThat(result).isInstanceOf(ProvisionResult.Success.class);
    }

    @Test
    void deprovisionSucceeds() {
        var runner = new PluginTestRunner(descriptor, Map.of());
        var result = runner.runDeprovision(Map.of("name", "app1", "status-code", 200));
        assertThat(result).isInstanceOf(DeprovisionResult.Success.class);
    }

    @Test
    void actualStatePresent() {
        var runner = new PluginTestRunner(descriptor, Map.of());
        var status = runner.runActualState(Map.of("name", "app1", "status-code", 200));
        assertThat(status).isEqualTo(NodeStatus.PRESENT);
    }

    @Test
    void actualStateAbsent() {
        var runner = new PluginTestRunner(descriptor, Map.of());
        var status = runner.runActualState(Map.of("name", "gone", "status-code", 404));
        assertThat(status).isEqualTo(NodeStatus.ABSENT);
    }

    @Test
    void faultInjectionCausesFailure() {
        var runner = new PluginTestRunner(descriptor, Map.of());
        var result = runner.runProvisionWithFaultInjection(
            Map.of("name", "app1", "status-code", 200), 1, "injected failure");
        assertThat(result).isInstanceOf(ProvisionResult.Failed.class);
        assertThat(((ProvisionResult.Failed) result).reason()).contains("injected failure");
    }

    private static PluginDescriptor toDescriptor(PluginModel model) {
        return new PluginDescriptor(
            model.header().type(), model.header().version(),
            Duration.ofSeconds(30), Map.of(), model.spec(),
            model.actualStateSteps(),
            model.provisioner().provisionSteps(),
            model.provisioner().deprovisionSteps(),
            model.faultPolicies(), model.cbr(), model.ras());
    }
}
```

- [ ] **Step 2: Implement PluginTestRunner, TestCredentialResolver, FaultInjectingStepRunner**

`PluginTestRunner`: constructs `DesiredNode` with `YamlNodeSpec`, builds `ConditionEvaluator` and `StructuralStepEvaluator`, creates `YamlPluginProvisioner`/`YamlPluginActualStateAdapter` per action call. Pre-processes infrastructure variable bindings in spec field string values before constructing `YamlNodeSpec`.

`TestCredentialResolver`: maps plugin auth stanza names to test-declared credential values via the plugin descriptor's `authCredentialRefs`.

`FaultInjectingStepRunner`: wraps `StepRunner`, decrements `AtomicInteger` failuresRemaining, throws `RuntimeException` when > 0.

- [ ] **Step 3: Run tests**

Run: `mvn --batch-mode test -pl plugin/testing -Dtest=PluginTestRunnerTest`
Expected: All 5 tests pass

- [ ] **Step 4: Commit**

```bash
git add plugin/testing/src/
git commit -m "feat(#135): implement PluginTestRunner with fault injection and credential resolution"
```

---

## Batch 4: PluginTestExtension and assertion engine

### Task 7: PluginTestAssertions — declarative assertion evaluation

**Files:**
- Create: `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/PluginTestAssertions.java`
- Test: `plugin/testing/src/test/java/io/casehub/desiredstate/plugin/testing/PluginTestAssertionsTest.java`

**Interfaces:**
- Consumes: `PluginTestCase.assertions()` map, action results (`ProvisionResult`, `DeprovisionResult`, `NodeStatus`)
- Produces: `PluginTestAssertions.evaluate(Map<String,String> assertions, ActionResult result)` — throws `AssertionError` with structured message on mismatch

- [ ] **Step 1: Write failing tests**

Test assertion evaluation for provision success, provision failure, actual-state match, error-matches regex, file-exists, file-absent.

- [ ] **Step 2: Implement PluginTestAssertions**

Evaluate each assertion key against the action result. Produce structured error messages including action, expected, actual, plugin path, test name, and spec values.

- [ ] **Step 3: Run tests and commit**

Run: `mvn --batch-mode test -pl plugin/testing -Dtest=PluginTestAssertionsTest`

```bash
git add plugin/testing/src/
git commit -m "feat(#135): implement declarative PluginTestAssertions engine"
```

---

### Task 8: PluginTestExtension — JUnit5 lifecycle orchestrator

**Files:**
- Create: `plugin/testing/src/main/java/io/casehub/desiredstate/plugin/testing/PluginTestExtension.java`
- Create: `plugin/testing/src/test/java/io/casehub/desiredstate/plugin/testing/PluginTestExtensionTest.java`
- Create: `plugin/testing/src/test/resources/META-INF/desiredstate/tests/mock-resource.test.yaml`

**Interfaces:**
- Consumes: All previous tasks — `PluginTestYamlParser`, `PluginTestRunner`, `PluginTestAssertions`, `TestInfrastructure`, `PluginValidator`, `PluginParser`
- Produces: `PluginTestExtension.forPlugin(String) → PluginTestExtension`, `discoverTests() → Stream<DynamicTest>`

- [ ] **Step 1: Create test YAML for mock-resource plugin**

`plugin/testing/src/test/resources/META-INF/desiredstate/tests/mock-resource.test.yaml`:

```yaml
plugin: mock-resource
infrastructure: http-mock

tests:
  - name: provision succeeds for valid spec
    spec:
      name: myapp
      status-code: 200
    action: provision
    assert:
      provision: success

  - name: actual state is PRESENT when status 200
    spec:
      name: myapp
      status-code: 200
    action: actual-state
    assert:
      actual-state: PRESENT

  - name: actual state is ABSENT when status 404
    spec:
      name: gone
      status-code: 404
    action: actual-state
    assert:
      actual-state: ABSENT

  - name: actual state is DRIFTED when status 206
    spec:
      name: partial
      status-code: 206
    action: actual-state
    assert:
      actual-state: DRIFTED

  - name: deprovision succeeds
    spec:
      name: myapp
      status-code: 200
    action: deprovision
    assert:
      deprovision: success
```

- [ ] **Step 2: Write the test class using PluginTestExtension**

```java
package io.casehub.desiredstate.plugin.testing;

import org.junit.jupiter.api.DynamicTest;
import org.junit.jupiter.api.TestFactory;
import org.junit.jupiter.api.extension.RegisterExtension;

import java.util.stream.Stream;

class PluginTestExtensionTest {

    @RegisterExtension
    static PluginTestExtension ext = PluginTestExtension.forPlugin("mock-resource");

    @TestFactory
    Stream<DynamicTest> tests() {
        return ext.discoverTests();
    }
}
```

- [ ] **Step 3: Implement PluginTestExtension**

The extension implements `BeforeAllCallback` and `AfterAllCallback`:
- `beforeAll()`: find plugin YAML at `META-INF/desiredstate/plugins/<type>.yaml`, parse via `PluginParser`, run `PluginValidator`, find test YAML at `META-INF/desiredstate/tests/*.test.yaml` matching the plugin type, parse all suites, determine infrastructure type, start infrastructure
- `discoverTests()`: for each test case across all suites, create a `DynamicTest` that: resets infrastructure, configures expectations, runs the action via `PluginTestRunner`, evaluates assertions via `PluginTestAssertions`
- `afterAll()`: stop infrastructure

- [ ] **Step 4: Run the end-to-end test**

Run: `mvn --batch-mode test -pl plugin/testing -Dtest=PluginTestExtensionTest`
Expected: 5 dynamic tests pass (provision success, 3 actual-state variants, deprovision)

- [ ] **Step 5: Commit**

```bash
git add plugin/testing/src/
git commit -m "feat(#135): implement PluginTestExtension with end-to-end YAML test discovery"
```

---

## Batch 5: Full build verification and documentation

### Task 9: Full build verification and consumer guide

**Files:**
- Modify: `docs/guides/consumer-guide.md` (add "Testing Plugins" section)
- Modify: `CLAUDE.md` (add `plugin/testing/` module entry)

**Interfaces:**
- Consumes: All previous tasks

- [ ] **Step 1: Run full build**

Run: `mvn --batch-mode install`
Expected: All modules compile and all tests pass, including the new `plugin/testing/` module

- [ ] **Step 2: Add Testing Plugins section to consumer guide**

Add a section to `docs/guides/consumer-guide.md` covering:
- Minimal test class (4 lines of Java)
- Test YAML format reference (plugin, infrastructure, setup, tests sections)
- Infrastructure types (http-mock, shell-sandbox)
- Common assertion patterns
- Fault injection example

- [ ] **Step 3: Update CLAUDE.md module table**

Add `plugin/testing/` entry to the module table in `CLAUDE.md`:
```
| `plugin/testing/` | `casehub-desiredstate-plugin-testing` | `io.casehub.desiredstate.plugin.testing` | YAML plugin test framework — PluginTestExtension (JUnit5), test YAML discovery, embedded WireMock HttpMockInfrastructure, ShellSandboxInfrastructure, PluginTestRunner, PluginTestAssertions, FaultInjectingStepRunner. Test scope only. |
```

- [ ] **Step 4: Commit**

```bash
git add docs/guides/consumer-guide.md CLAUDE.md
git commit -m "docs(#135): add Testing Plugins consumer guide section and CLAUDE.md module entry"
```

---

## References

- [2026-09-29-plugin-testing-framework-design.md] — design spec this plan implements
- [plugin/runtime/src/main/java/.../YamlPluginProvisioner.java] — production provisioner exercised by test runner
- [plugin/runtime/src/main/java/.../YamlPluginActualStateAdapter.java] — production adapter exercised by test runner
- [plugin/runtime/src/main/java/.../PluginDescriptor.java] — plugin descriptor record
- [plugin/runtime/src/main/java/.../ActualStateStepExecutor.java] — actual-state step execution
- [plugin/deployment/src/main/java/.../YamlPluginProcessor.java:86-303] — validation methods to extract
- [plugin/deployment/src/main/java/.../PluginValidationException.java] — exception to move
- [plugin/runtime/src/test/java/.../PluginIntegrationTest.java] — existing ad-hoc pattern
- [plugin/runtime/src/test/resources/META-INF/desiredstate/plugins/mock-resource.yaml] — test plugin fixture
- [plugin/api/src/main/java/.../YamlNodeSpec.java] — NodeSpec implementation required for spec field extraction
- [io.casehub.yaml.step.eval.StepRunner] — `Result run(ResolvedStep, VariableResolver)` — production step runner interface
- [io.casehub.yaml.core.condition.ConditionEvaluator] — condition evaluation for `when:` decorators
- [#87 D13] — deferred testing companion spec (origin requirement)
- [#135] — focal issue
