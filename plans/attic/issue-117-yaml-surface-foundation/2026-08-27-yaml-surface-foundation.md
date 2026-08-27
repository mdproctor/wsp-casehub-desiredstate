# YAML Surface Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #117 — epic: YAML surface foundation — deserializer, type registry, variable substitution
**Issue group:** #117

**Goal:** Build the YAML → GraphDescriptor pipeline so operators can declare desired-state graphs in YAML and have them compile and reconcile identically to the annotation path.

**Architecture:** YAML files on classpath are discovered at build time by a Quarkus build extension, parsed via Jackson, validated, and transformed into `GraphDescriptor` records with `InlineNode` descriptors. A YAML-specific recorder creates `GoalCompiler<Void>` beans that resolve variables (Map → Preferences → Config) and deserialize spec values into typed `NodeSpec` records via a Jandex-discovered type registry.

**Tech Stack:** Java 21, Quarkus (build extension pattern: runtime + deployment), Jackson YAML, Jandex, JUnit 5, AssertJ

## Global Constraints

- All new code in `io.casehub.desiredstate.*` packages
- `@NodeTypeId` annotation lives in `api/` (not `annotations/`) — it's metadata on NodeSpec, not an annotation-surface concept
- YAML files discovered at `META-INF/desiredstate/*.yaml` on classpath
- Module naming: `casehub-desiredstate-yaml` (runtime), `casehub-desiredstate-yaml-deployment` (deployment)
- Variable syntax: `${key}` — simple lookup, no expressions, no defaults
- `GoalCompiler<Void>` — YAML graphs are fully declared, goals parameter is always null
- All IDE operations use `mcp__intellij-index__*` tools — never bash grep/find on source files

---

## Batch 1: IR Extension and Foundation Types

### Task 1: @NodeTypeId annotation, NodeDescriptor.InlineNode, and pipeline @NodeTypeId annotations

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/NodeTypeId.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/NodeDescriptor.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java` (buildNodes switch)
- Modify: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/DataSourceSpec.java`
- Modify: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/IngestionSpec.java`
- Modify: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/SchemaSpec.java`
- Modify: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/CleanserSpec.java`
- Modify: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/EnricherSpec.java`
- Modify: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/ValidatorSpec.java`
- Modify: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/TransformerSpec.java`
- Modify: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/SinkSpec.java`
- Modify: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/AiReviewSpec.java`
- Modify: `examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/HumanReviewSpec.java`
- Modify: `examples/pipeline-annotated/src/main/java/io/casehub/desiredstate/example/pipeline/annotated/MonitorSpec.java`
- Test: `api/src/test/java/io/casehub/desiredstate/api/NodeTypeIdTest.java`
- Test: `annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/NodeDescriptorTest.java`

**Interfaces:**
- Produces: `@NodeTypeId` annotation (retention=RUNTIME, target=TYPE), `NodeDescriptor.InlineNode` record
- Produces: All pipeline NodeSpec classes annotated with `@NodeTypeId`

- [ ] **Step 1: Write failing test for @NodeTypeId annotation**

```java
package io.casehub.desiredstate.api;

import org.junit.jupiter.api.Test;
import java.lang.annotation.ElementType;
import java.lang.annotation.RetentionPolicy;
import static org.assertj.core.api.Assertions.assertThat;

class NodeTypeIdTest {

    @Test
    void annotationHasRuntimeRetention() {
        assertThat(NodeTypeId.class.getAnnotation(java.lang.annotation.Retention.class).value())
                .isEqualTo(RetentionPolicy.RUNTIME);
    }

    @Test
    void annotationTargetsType() {
        assertThat(NodeTypeId.class.getAnnotation(java.lang.annotation.Target.class).value())
                .containsExactly(ElementType.TYPE);
    }

    @NodeTypeId("test-type")
    record TestSpec() {}

    @Test
    void valueIsReadable() {
        assertThat(TestSpec.class.getAnnotation(NodeTypeId.class).value())
                .isEqualTo("test-type");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=NodeTypeIdTest`
Expected: compilation failure — `NodeTypeId` does not exist

- [ ] **Step 3: Create @NodeTypeId annotation**

Create `api/src/main/java/io/casehub/desiredstate/api/NodeTypeId.java`:

```java
package io.casehub.desiredstate.api;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface NodeTypeId {
    String value();
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl api -Dtest=NodeTypeIdTest`
Expected: 3 tests PASS

- [ ] **Step 5: Write failing test for NodeDescriptor.InlineNode**

```java
package io.casehub.desiredstate.annotations.runtime;

import io.casehub.desiredstate.api.HumanGating;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class NodeDescriptorTest {

    @Test
    void inlineNodeImplementsNodeDescriptor() {
        NodeDescriptor node = new NodeDescriptor.InlineNode(
                "test-node",
                "io.casehub.desiredstate.example.pipeline.DataSourceSpec",
                Map.of("name", "test", "format", "CSV", "uri", "s3://test"),
                HumanGating.NONE);

        assertThat(node.id()).isEqualTo("test-node");
        assertThat(node).isInstanceOf(NodeDescriptor.InlineNode.class);

        var inline = (NodeDescriptor.InlineNode) node;
        assertThat(inline.specClassName()).isEqualTo("io.casehub.desiredstate.example.pipeline.DataSourceSpec");
        assertThat(inline.specValues()).containsEntry("name", "test");
        assertThat(inline.humanGating()).isEqualTo(HumanGating.NONE);
    }

    @Test
    void sealedHierarchyPermitsInlineNode() {
        assertThat(NodeDescriptor.class.getPermittedSubclasses())
                .extracting(Class::getSimpleName)
                .contains("InlineNode", "InterfaceNode", "ClassNode");
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=NodeDescriptorTest`
Expected: compilation failure — `InlineNode` does not exist

- [ ] **Step 7: Add InlineNode to NodeDescriptor sealed interface**

Use `ide_insert_member` to add the `InlineNode` record to `NodeDescriptor.java`. The sealed interface's `permits` clause also needs updating:

```java
public sealed interface NodeDescriptor
        permits NodeDescriptor.InterfaceNode, NodeDescriptor.ClassNode,
                NodeDescriptor.InlineNode {

    String id();

    record InterfaceNode(String id, String methodName, String returnTypeName,
                         HumanGating humanGating) implements NodeDescriptor {}

    record ClassNode(String id, String className) implements NodeDescriptor {}

    record InlineNode(String id, String specClassName,
                      Map<String, Object> specValues,
                      HumanGating humanGating) implements NodeDescriptor {}
}
```

Add `import java.util.Map;` to the imports.

- [ ] **Step 8: Fix DesiredStateGraphRecorder.buildNodes() exhaustive switch**

The `buildNodes()` method at line ~292 has an exhaustive switch over `NodeDescriptor` variants. Add a case for `InlineNode` that throws — annotation-path graphs should never contain inline nodes:

In `DesiredStateGraphRecorder.buildNodes()`, add after the `ClassNode` case:

```java
case NodeDescriptor.InlineNode ignored ->
    throw new IllegalStateException("InlineNode cannot appear in annotation-path graphs");
```

- [ ] **Step 9: Run test to verify InlineNode works**

Run: `mvn --batch-mode test -pl annotations/runtime -Dtest=NodeDescriptorTest`
Expected: 2 tests PASS

- [ ] **Step 10: Add @NodeTypeId to all pipeline NodeSpec classes**

Add `import io.casehub.desiredstate.api.NodeTypeId;` and `@NodeTypeId("<type>")` to each:

| File | Annotation |
|------|-----------|
| `DataSourceSpec.java` | `@NodeTypeId("data-source")` |
| `IngestionSpec.java` | `@NodeTypeId("ingestion")` |
| `SchemaSpec.java` | `@NodeTypeId("schema")` |
| `CleanserSpec.java` | `@NodeTypeId("cleanser")` |
| `EnricherSpec.java` | `@NodeTypeId("enricher")` |
| `ValidatorSpec.java` | `@NodeTypeId("validator")` |
| `TransformerSpec.java` | `@NodeTypeId("transformer")` |
| `SinkSpec.java` | `@NodeTypeId("sink")` |
| `AiReviewSpec.java` | `@NodeTypeId("ai-review")` |
| `HumanReviewSpec.java` | `@NodeTypeId("human-review")` |
| `MonitorSpec.java` (pipeline-annotated) | `@NodeTypeId("monitor")` |

The `@NodeTypeId` value must match `nodeType().value()` for each class. Cross-check against `PipelineNodeTypes` constants.

- [ ] **Step 11: Verify full build passes**

Run: `mvn --batch-mode install`
Expected: all existing tests pass — no behavior changes, only additive annotations

- [ ] **Step 12: Commit**

```bash
git add api/src/main/java/io/casehub/desiredstate/api/NodeTypeId.java
git add api/src/test/java/io/casehub/desiredstate/api/NodeTypeIdTest.java
git add annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/NodeDescriptor.java
git add annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java
git add annotations/runtime/src/test/java/io/casehub/desiredstate/annotations/runtime/NodeDescriptorTest.java
git add examples/pipeline/src/main/java/io/casehub/desiredstate/example/pipeline/
git add examples/pipeline-annotated/src/main/java/io/casehub/desiredstate/example/pipeline/annotated/MonitorSpec.java
git commit -m "feat(#117): @NodeTypeId annotation, NodeDescriptor.InlineNode, pipeline @NodeTypeId annotations"
```

---

## Batch 2: YAML Runtime Module

### Task 2: Module scaffolding, YAML model types, and VariableResolver

**Files:**
- Create: `yaml/pom.xml`
- Create: `yaml/runtime/pom.xml`
- Create: `yaml/deployment/pom.xml`
- Modify: `pom.xml` (add `<module>yaml</module>`, `<module>examples/pipeline-yaml</module>`)
- Modify: `pom.xml` dependencyManagement (add yaml artifacts)
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlGraph.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlDesiredState.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlNode.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/resolver/VariableResolver.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/resolver/UnresolvedVariableException.java`
- Test: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/model/YamlGraphDeserializationTest.java`
- Test: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/resolver/VariableResolverTest.java`

**Interfaces:**
- Produces: `YamlGraph`, `YamlDesiredState`, `YamlNode` records (Jackson-deserializable)
- Produces: `VariableResolver` with `resolve(Object)`, `resolveMap(Map)`, `resolveString(String)` methods
- Produces: `UnresolvedVariableException` with variable name, node context, and layers searched

- [ ] **Step 1: Create yaml/ parent POM**

Create `yaml/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-desiredstate-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-desiredstate-yaml-parent</artifactId>
    <packaging>pom</packaging>
    <name>CaseHub Desired State :: YAML</name>

    <modules>
        <module>runtime</module>
        <module>deployment</module>
    </modules>
</project>
```

- [ ] **Step 2: Create yaml/runtime/ POM**

Create `yaml/runtime/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-desiredstate-yaml-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-desiredstate-yaml</artifactId>
    <name>CaseHub Desired State :: YAML :: Runtime</name>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-annotations</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-core</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.dataformat</groupId>
            <artifactId>jackson-dataformat-yaml</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>org.eclipse.microprofile.config</groupId>
            <artifactId>microprofile-config-api</artifactId>
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

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 3: Create yaml/deployment/ POM (minimal)**

Create `yaml/deployment/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-desiredstate-yaml-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-desiredstate-yaml-deployment</artifactId>
    <name>CaseHub Desired State :: YAML :: Deployment</name>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-yaml</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-annotations-deployment</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc-deployment</artifactId>
        </dependency>

        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5-internal</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate</artifactId>
            <version>${project.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>io.quarkus</groupId>
                            <artifactId>quarkus-extension-processor</artifactId>
                            <version>${version.quarkus.platform}</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <systemPropertyVariables>
                        <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
                        <maven.home>${maven.home}</maven.home>
                    </systemPropertyVariables>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 4: Add modules to parent POM**

In the root `pom.xml`, add `<module>yaml</module>` to `<modules>`. Also add dependency management entries for the new artifacts:

```xml
<module>yaml</module>
<module>examples/pipeline-yaml</module>
```

In `<dependencyManagement>` add:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-desiredstate-yaml</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 5: Create source directories**

```bash
mkdir -p yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model
mkdir -p yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/resolver
mkdir -p yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/registry
mkdir -p yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/model
mkdir -p yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/resolver
mkdir -p yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment
```

- [ ] **Step 6: Write failing test for YamlGraph deserialization**

Create `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/model/YamlGraphDeserializationTest.java`:

```java
package io.casehub.desiredstate.yaml.model;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.desiredstate.api.HumanGating;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class YamlGraphDeserializationTest {

    private final ObjectMapper mapper = new ObjectMapper(new YAMLFactory());

    @Test
    void deserializesMinimalGraph() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: simple
                nodes:
                  my-node:
                    type: data-source
                    spec:
                      name: test-source
                """;

        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);

        assertThat(graph.desiredState().namespace()).isEqualTo("test");
        assertThat(graph.desiredState().name()).isEqualTo("simple");
        assertThat(graph.nodes()).hasSize(1);
        assertThat(graph.nodes().get("my-node").type()).isEqualTo("data-source");
        assertThat(graph.nodes().get("my-node").spec()).containsEntry("name", "test-source");
    }

    @Test
    void deserializesVariablesAndDependencies() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: deps
                variables:
                  batch: 500
                nodes:
                  source:
                    type: data-source
                    spec:
                      name: src
                  ingest:
                    type: ingestion
                    dependsOn: [source]
                    spec:
                      batchSize: ${batch}
                """;

        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);

        assertThat(graph.variables()).containsEntry("batch", "500");
        assertThat(graph.nodes().get("ingest").dependsOn()).containsExactly("source");
    }

    @Test
    void deserializesHumanGating() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: gated
                nodes:
                  gated-node:
                    type: transformer
                    humanGating: PROVISION_ONLY
                    spec:
                      name: test
                """;

        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);

        assertThat(graph.nodes().get("gated-node").humanGating())
                .isEqualTo(HumanGating.PROVISION_ONLY);
    }

    @Test
    void defaultsHumanGatingToNone() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: default
                nodes:
                  plain:
                    type: data-source
                    spec:
                      name: test
                """;

        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);

        assertThat(graph.nodes().get("plain").humanGating()).isEqualTo(HumanGating.NONE);
    }

    @Test
    void specDefaultsToEmptyMap() throws Exception {
        String yaml = """
                desiredState:
                  namespace: test
                  name: nospec
                nodes:
                  marker:
                    type: marker
                """;

        YamlGraph graph = mapper.readValue(yaml, YamlGraph.class);

        assertThat(graph.nodes().get("marker").spec()).isNotNull().isEmpty();
    }
}
```

- [ ] **Step 7: Run test to verify it fails**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlGraphDeserializationTest`
Expected: compilation failure — model classes don't exist

- [ ] **Step 8: Create YAML model types**

Create `YamlDesiredState.java`:

```java
package io.casehub.desiredstate.yaml.model;

public record YamlDesiredState(String namespace, String name) {}
```

Create `YamlNode.java`:

```java
package io.casehub.desiredstate.yaml.model;

import com.fasterxml.jackson.annotation.JsonProperty;
import io.casehub.desiredstate.api.HumanGating;
import java.util.List;
import java.util.Map;

public record YamlNode(
        String type,
        @JsonProperty(defaultValue = "{}") Map<String, Object> spec,
        List<String> dependsOn,
        HumanGating humanGating) {

    public YamlNode {
        if (spec == null) spec = Map.of();
        if (dependsOn == null) dependsOn = List.of();
        if (humanGating == null) humanGating = HumanGating.NONE;
    }
}
```

Create `YamlGraph.java`:

```java
package io.casehub.desiredstate.yaml.model;

import java.util.Map;

public record YamlGraph(
        YamlDesiredState desiredState,
        Map<String, String> variables,
        Map<String, YamlNode> nodes) {

    public YamlGraph {
        if (variables == null) variables = Map.of();
        if (nodes == null) nodes = Map.of();
    }
}
```

- [ ] **Step 9: Run deserialization tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlGraphDeserializationTest`
Expected: 5 tests PASS

- [ ] **Step 10: Write failing test for VariableResolver**

Create `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/resolver/VariableResolverTest.java`:

```java
package io.casehub.desiredstate.yaml.resolver;

import org.junit.jupiter.api.Test;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class VariableResolverTest {

    @Test
    void resolvesFromInlineMap() {
        var resolver = new VariableResolver(Map.of("batch", "500"), null, null);
        assertThat(resolver.resolveString("${batch}", "test-node"))
                .isEqualTo("500");
    }

    @Test
    void passesNonVariableStringsThrough() {
        var resolver = new VariableResolver(Map.of(), null, null);
        assertThat(resolver.resolve("plain-string"))
                .isEqualTo("plain-string");
    }

    @Test
    void resolvesNestedMapValues() {
        var resolver = new VariableResolver(Map.of("uri", "s3://data"), null, null);
        Map<String, Object> input = new LinkedHashMap<>();
        input.put("destination", "${uri}");
        input.put("count", 42);

        @SuppressWarnings("unchecked")
        Map<String, Object> resolved = (Map<String, Object>) resolver.resolveMap(input, "node");

        assertThat(resolved).containsEntry("destination", "s3://data");
        assertThat(resolved).containsEntry("count", 42);
    }

    @Test
    void resolvesListValues() {
        var resolver = new VariableResolver(Map.of("field", "email"), null, null);
        List<?> resolved = (List<?>) resolver.resolveList(
                List.of("name", "${field}"), "node");

        assertThat(resolved).containsExactly("name", "email");
    }

    @Test
    void throwsOnUnresolvedVariable() {
        var resolver = new VariableResolver(Map.of("batch_size", "100"), null, null);

        assertThatThrownBy(() -> resolver.resolveString("${bacth_size}", "csv-ingest"))
                .isInstanceOf(UnresolvedVariableException.class)
                .hasMessageContaining("bacth_size")
                .hasMessageContaining("csv-ingest");
    }

    @Test
    void wholeStringVariablePreservesType() {
        var resolver = new VariableResolver(Map.of("count", "42"), null, null);
        assertThat(resolver.resolveString("${count}", "node")).isEqualTo("42");
    }

    @Test
    void embeddedVariableInLargerString() {
        var resolver = new VariableResolver(Map.of("bucket", "prod"), null, null);
        assertThat(resolver.resolveString("s3://${bucket}/data", "node"))
                .isEqualTo("s3://prod/data");
    }
}
```

- [ ] **Step 11: Run test to verify it fails**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=VariableResolverTest`
Expected: compilation failure — VariableResolver/UnresolvedVariableException don't exist

- [ ] **Step 12: Create UnresolvedVariableException**

Create `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/resolver/UnresolvedVariableException.java`:

```java
package io.casehub.desiredstate.yaml.resolver;

public class UnresolvedVariableException extends RuntimeException {

    private final String variableName;
    private final String nodeContext;

    public UnresolvedVariableException(String variableName, String nodeContext, String detail) {
        super("Unresolved variable '" + variableName + "' in node '" + nodeContext + "'. " + detail);
        this.variableName = variableName;
        this.nodeContext = nodeContext;
    }

    public String variableName() { return variableName; }
    public String nodeContext() { return nodeContext; }
}
```

- [ ] **Step 13: Create VariableResolver**

Create `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/resolver/VariableResolver.java`:

```java
package io.casehub.desiredstate.yaml.resolver;

import org.eclipse.microprofile.config.Config;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class VariableResolver {

    private static final Pattern VAR_PATTERN = Pattern.compile("\\$\\{([^}]+)}");

    private final Map<String, String> inlineVariables;
    private final Object preferences;
    private final Config config;

    public VariableResolver(Map<String, String> inlineVariables,
                            Object preferences, Config config) {
        this.inlineVariables = inlineVariables != null ? inlineVariables : Map.of();
        this.preferences = preferences;
        this.config = config;
    }

    public Object resolve(Object value) {
        if (value instanceof String s) {
            return s.contains("${") ? resolveString(s, "") : s;
        }
        if (value instanceof Map<?, ?> map) {
            return resolveMap(map, "");
        }
        if (value instanceof List<?> list) {
            return resolveList(list, "");
        }
        return value;
    }

    public String resolveString(String template, String nodeContext) {
        Matcher matcher = VAR_PATTERN.matcher(template);
        StringBuilder sb = new StringBuilder();
        while (matcher.find()) {
            String key = matcher.group(1);
            String resolved = lookupVariable(key, nodeContext);
            matcher.appendReplacement(sb, Matcher.quoteReplacement(resolved));
        }
        matcher.appendTail(sb);
        return sb.toString();
    }

    @SuppressWarnings("unchecked")
    public Map<String, Object> resolveMap(Map<?, ?> input, String nodeContext) {
        Map<String, Object> result = new LinkedHashMap<>();
        for (Map.Entry<?, ?> entry : input.entrySet()) {
            String key = entry.getKey().toString();
            Object val = entry.getValue();
            if (val instanceof String s && s.contains("${")) {
                result.put(key, resolveString(s, nodeContext));
            } else if (val instanceof Map<?, ?> nested) {
                result.put(key, resolveMap(nested, nodeContext));
            } else if (val instanceof List<?> list) {
                result.put(key, resolveList(list, nodeContext));
            } else {
                result.put(key, val);
            }
        }
        return result;
    }

    public List<?> resolveList(List<?> input, String nodeContext) {
        return input.stream()
                .map(item -> {
                    if (item instanceof String s && s.contains("${")) {
                        return resolveString(s, nodeContext);
                    }
                    return item;
                })
                .toList();
    }

    private String lookupVariable(String key, String nodeContext) {
        String value = inlineVariables.get(key);
        if (value != null) return value;

        if (config != null) {
            Optional<String> configValue = config.getOptionalValue(key, String.class);
            if (configValue.isPresent()) return configValue.get();
        }

        throw new UnresolvedVariableException(key, nodeContext,
                "Not found in: inline variables " + inlineVariables.keySet()
                + ", MicroProfile Config.");
    }
}
```

Note: Preferences layer integration is stubbed as `Object preferences` for now — full CDI integration happens in the recorder (Task 3). The VariableResolver is a pure resolver; CDI wiring is the recorder's concern.

- [ ] **Step 14: Run VariableResolver tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=VariableResolverTest`
Expected: 7 tests PASS

- [ ] **Step 15: Commit**

```bash
git add yaml/ pom.xml
git commit -m "feat(#117): YAML module scaffolding, model types, and VariableResolver"
```

---

### Task 3: NodeSpecRegistry and YamlGraphRecorder

**Files:**
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/registry/NodeSpecRegistry.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`
- Test: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/registry/NodeSpecRegistryTest.java`
- Test: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlGraphRecorderTest.java`

**Interfaces:**
- Consumes: `NodeDescriptor.InlineNode` (from Task 1), `YamlGraph`/`YamlNode` (from Task 2), `VariableResolver` (from Task 2)
- Produces: `NodeSpecRegistry.resolve(String) → Class<? extends NodeSpec>`, `NodeSpecRegistry.availableTypes() → Set<String>`
- Produces: `YamlGraphRecorder.createYamlGoalCompiler(GraphDescriptor, Map<String,String>, Map<String,String>) → RuntimeValue<GoalCompiler>`

- [ ] **Step 1: Write failing test for NodeSpecRegistry**

Create `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/registry/NodeSpecRegistryTest.java`:

```java
package io.casehub.desiredstate.yaml.registry;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.NodeTypeId;
import org.junit.jupiter.api.Test;
import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class NodeSpecRegistryTest {

    @NodeTypeId("test-type")
    record TestNodeSpec() implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("test-type"); }
    }

    @Test
    void resolvesKnownType() {
        var registry = NodeSpecRegistry.of(
                Map.of("test-type", TestNodeSpec.class.getName()));
        assertThat(registry.resolve("test-type")).isEqualTo(TestNodeSpec.class);
    }

    @Test
    void throwsOnUnknownType() {
        var registry = NodeSpecRegistry.of(Map.of("test-type", TestNodeSpec.class.getName()));
        assertThatThrownBy(() -> registry.resolve("unknown"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("unknown")
                .hasMessageContaining("test-type");
    }

    @Test
    void reportsAvailableTypes() {
        var registry = NodeSpecRegistry.of(Map.of(
                "type-a", TestNodeSpec.class.getName(),
                "type-b", TestNodeSpec.class.getName()));
        assertThat(registry.availableTypes()).isEqualTo(Set.of("type-a", "type-b"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=NodeSpecRegistryTest`
Expected: compilation failure

- [ ] **Step 3: Create NodeSpecRegistry**

Create `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/registry/NodeSpecRegistry.java`:

```java
package io.casehub.desiredstate.yaml.registry;

import io.casehub.desiredstate.api.NodeSpec;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;

public class NodeSpecRegistry {

    private final Map<String, Class<? extends NodeSpec>> typeMap;

    private NodeSpecRegistry(Map<String, Class<? extends NodeSpec>> typeMap) {
        this.typeMap = Map.copyOf(typeMap);
    }

    @SuppressWarnings("unchecked")
    public static NodeSpecRegistry of(Map<String, String> typeToClassName) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        Map<String, Class<? extends NodeSpec>> resolved = new HashMap<>();
        for (Map.Entry<String, String> entry : typeToClassName.entrySet()) {
            try {
                Class<?> cls = cl.loadClass(entry.getValue());
                resolved.put(entry.getKey(), (Class<? extends NodeSpec>) cls);
            } catch (ClassNotFoundException e) {
                throw new RuntimeException("NodeSpec class not found: " + entry.getValue(), e);
            }
        }
        return new NodeSpecRegistry(resolved);
    }

    public Class<? extends NodeSpec> resolve(String typeName) {
        Class<? extends NodeSpec> cls = typeMap.get(typeName);
        if (cls == null) {
            throw new IllegalArgumentException("Unknown node type: '" + typeName
                    + "'. Available types: " + typeMap.keySet());
        }
        return cls;
    }

    public Set<String> availableTypes() {
        return Collections.unmodifiableSet(typeMap.keySet());
    }
}
```

- [ ] **Step 4: Run NodeSpecRegistry tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=NodeSpecRegistryTest`
Expected: 3 tests PASS

- [ ] **Step 5: Write failing test for YamlGraphRecorder**

Create `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/YamlGraphRecorderTest.java`:

```java
package io.casehub.desiredstate.yaml;

import io.casehub.desiredstate.annotations.runtime.DependencyDescriptor;
import io.casehub.desiredstate.annotations.runtime.GraphDescriptor;
import io.casehub.desiredstate.annotations.runtime.NodeDescriptor;
import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.NodeTypeId;
import io.casehub.desiredstate.runtime.ImmutableDesiredStateGraph;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class YamlGraphRecorderTest {

    @NodeTypeId("test-source")
    public record TestSourceSpec(String name, String uri) implements NodeSpec {
        @Override public NodeType nodeType() { return NodeType.of("test-source"); }
    }

    @Test
    void compilesInlineNodesToDesiredStateGraph() {
        var descriptor = new GraphDescriptor(
                "test", "simple", null, null,
                List.of(
                        new NodeDescriptor.InlineNode("src", TestSourceSpec.class.getName(),
                                Map.of("name", "my-source", "uri", "s3://data"),
                                HumanGating.NONE)
                ),
                List.of(), List.of(), null, List.of(), List.of());

        Map<String, String> typeRegistry = Map.of("test-source", TestSourceSpec.class.getName());
        Map<String, String> variables = Map.of();

        var recorder = new YamlGraphRecorder();
        @SuppressWarnings("unchecked")
        GoalCompiler<Void> compiler = recorder.createYamlGoalCompiler(
                descriptor, typeRegistry, variables).getValue();

        CompilationResult result = compiler.compile(null, ImmutableDesiredStateGraph::new);
        assertThat(result).isInstanceOf(CompilationResult.SingleGraph.class);

        DesiredStateGraph graph = ((CompilationResult.SingleGraph) result).graph();
        assertThat(graph.nodes()).hasSize(1);

        DesiredNode node = graph.nodes().iterator().next();
        assertThat(node.id()).isEqualTo(NodeId.of("src"));
        assertThat(node.type()).isEqualTo(NodeType.of("test-source"));
        assertThat(node.spec()).isInstanceOf(TestSourceSpec.class);

        TestSourceSpec spec = (TestSourceSpec) node.spec();
        assertThat(spec.name()).isEqualTo("my-source");
        assertThat(spec.uri()).isEqualTo("s3://data");
    }

    @Test
    void resolvesVariablesBeforeDeserialization() {
        var descriptor = new GraphDescriptor(
                "test", "vars", null, null,
                List.of(
                        new NodeDescriptor.InlineNode("src", TestSourceSpec.class.getName(),
                                Map.of("name", "source", "uri", "${data_uri}"),
                                HumanGating.NONE)
                ),
                List.of(), List.of(), null, List.of(), List.of());

        Map<String, String> typeRegistry = Map.of("test-source", TestSourceSpec.class.getName());
        Map<String, String> variables = Map.of("data_uri", "s3://resolved");

        var recorder = new YamlGraphRecorder();
        @SuppressWarnings("unchecked")
        GoalCompiler<Void> compiler = recorder.createYamlGoalCompiler(
                descriptor, typeRegistry, variables).getValue();

        CompilationResult result = compiler.compile(null, ImmutableDesiredStateGraph::new);
        DesiredStateGraph graph = ((CompilationResult.SingleGraph) result).graph();
        TestSourceSpec spec = (TestSourceSpec) graph.nodes().iterator().next().spec();
        assertThat(spec.uri()).isEqualTo("s3://resolved");
    }

    @Test
    void buildsDependencyEdges() {
        var descriptor = new GraphDescriptor(
                "test", "deps", null, null,
                List.of(
                        new NodeDescriptor.InlineNode("a", TestSourceSpec.class.getName(),
                                Map.of("name", "a", "uri", "x"), HumanGating.NONE),
                        new NodeDescriptor.InlineNode("b", TestSourceSpec.class.getName(),
                                Map.of("name", "b", "uri", "y"), HumanGating.NONE)
                ),
                List.of(new DependencyDescriptor("b", "a")),
                List.of(), null, List.of(), List.of());

        Map<String, String> typeRegistry = Map.of("test-source", TestSourceSpec.class.getName());

        var recorder = new YamlGraphRecorder();
        @SuppressWarnings("unchecked")
        GoalCompiler<Void> compiler = recorder.createYamlGoalCompiler(
                descriptor, typeRegistry, Map.of()).getValue();

        CompilationResult result = compiler.compile(null, ImmutableDesiredStateGraph::new);
        DesiredStateGraph graph = ((CompilationResult.SingleGraph) result).graph();

        assertThat(graph.dependenciesOf(NodeId.of("b"))).contains(NodeId.of("a"));
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlGraphRecorderTest`
Expected: compilation failure — YamlGraphRecorder doesn't exist

- [ ] **Step 7: Create YamlGraphRecorder**

Create `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`:

```java
package io.casehub.desiredstate.yaml;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.desiredstate.annotations.runtime.DependencyDescriptor;
import io.casehub.desiredstate.annotations.runtime.GraphDescriptor;
import io.casehub.desiredstate.annotations.runtime.NodeDescriptor;
import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.Dependency;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.yaml.registry.NodeSpecRegistry;
import io.casehub.desiredstate.yaml.resolver.VariableResolver;
import io.quarkus.runtime.RuntimeValue;
import io.quarkus.runtime.annotations.Recorder;
import org.jboss.logging.Logger;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

@Recorder
public class YamlGraphRecorder {

    private static final Logger LOG = Logger.getLogger(YamlGraphRecorder.class);

    @SuppressWarnings({"unchecked", "rawtypes"})
    public RuntimeValue<GoalCompiler> createYamlGoalCompiler(
            GraphDescriptor descriptor,
            Map<String, String> typeRegistryMap,
            Map<String, String> inlineVariables) {

        ObjectMapper mapper = new ObjectMapper();
        NodeSpecRegistry registry = NodeSpecRegistry.of(typeRegistryMap);

        return new RuntimeValue<>((GoalCompiler) (goals, factory) -> {
            VariableResolver resolver = new VariableResolver(inlineVariables, null, null);

            List<DesiredNode> nodes = new ArrayList<>();
            for (NodeDescriptor nd : descriptor.nodes()) {
                if (nd instanceof NodeDescriptor.InlineNode in) {
                    Class<? extends NodeSpec> specClass = registry.resolveByClassName(in.specClassName());
                    Map<String, Object> resolved = resolver.resolveMap(in.specValues(), in.id());
                    NodeSpec spec = mapper.convertValue(resolved, specClass);

                    String expectedType = findTypeNameForClass(typeRegistryMap, in.specClassName());
                    if (expectedType != null && !spec.nodeType().value().equals(expectedType)) {
                        throw new IllegalStateException(
                                "@NodeTypeId(\"" + expectedType + "\") diverges from nodeType()=\""
                                + spec.nodeType().value() + "\" on " + specClass.getName());
                    }

                    nodes.add(new DesiredNode(NodeId.of(in.id()), spec, in.humanGating()));
                }
            }

            List<Dependency> deps = new ArrayList<>();
            for (DependencyDescriptor dd : descriptor.dependencies()) {
                deps.add(new Dependency(NodeId.of(dd.from()), NodeId.of(dd.to())));
            }

            return CompilationResult.single(factory.of(nodes, deps));
        });
    }

    private static String findTypeNameForClass(Map<String, String> typeRegistry, String className) {
        for (Map.Entry<String, String> entry : typeRegistry.entrySet()) {
            if (entry.getValue().equals(className)) {
                return entry.getKey();
            }
        }
        return null;
    }
}
```

Add `resolveByClassName` to `NodeSpecRegistry`:

```java
public Class<? extends NodeSpec> resolveByClassName(String className) {
    for (Class<? extends NodeSpec> cls : typeMap.values()) {
        if (cls.getName().equals(className)) return cls;
    }
    throw new IllegalArgumentException("No NodeSpec registered with class: " + className);
}
```

- [ ] **Step 8: Run YamlGraphRecorder tests**

Run: `mvn --batch-mode test -pl yaml/runtime -Dtest=YamlGraphRecorderTest`
Expected: 3 tests PASS

- [ ] **Step 9: Run all yaml/runtime tests**

Run: `mvn --batch-mode test -pl yaml/runtime`
Expected: all tests PASS

- [ ] **Step 10: Commit**

```bash
git add yaml/runtime/
git commit -m "feat(#117): NodeSpecRegistry and YamlGraphRecorder with unit tests"
```

---

## Batch 3: Build Extension and Pipeline Example

### Task 4: YamlDesiredStateProcessor and build-time validation

**Files:**
- Create: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java`
- Create: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateGraphBuildItem.java`
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java` (emit build items)
- Test: `yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessorTest.java`

**Interfaces:**
- Consumes: `YamlGraph` model (Task 2), `NodeSpecRegistry` (Task 3), `YamlGraphRecorder` (Task 3), `@NodeTypeId` (Task 1)
- Produces: `DesiredStateGraphBuildItem(String namespace, String name, String source)` — emitted by both processors
- Produces: `YamlDesiredStateProcessor.discoverYamlGraphs()` @BuildStep
- Produces: Cross-surface collision validation @BuildStep (shared, in annotations/deployment)

- [ ] **Step 1: Create DesiredStateGraphBuildItem**

Create `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateGraphBuildItem.java`:

```java
package io.casehub.desiredstate.annotations.deployment;

import io.quarkus.builder.item.MultiBuildItem;

public final class DesiredStateGraphBuildItem extends MultiBuildItem {

    private final String namespace;
    private final String name;
    private final String source;

    public DesiredStateGraphBuildItem(String namespace, String name, String source) {
        this.namespace = namespace;
        this.name = name;
        this.source = source;
    }

    public String namespace() { return namespace; }
    public String name() { return name; }
    public String source() { return source; }
    public String qualifiedName() { return namespace + ":" + name; }
}
```

- [ ] **Step 2: Update DesiredStateAnnotationsProcessor to emit build items**

In `DesiredStateAnnotationsProcessor.generateDesiredStateGraphs()`, after each `registerGoalCompilerBean()` call, also produce a `DesiredStateGraphBuildItem`:

Add `BuildProducer<DesiredStateGraphBuildItem> graphItems` parameter to the method. After each graph registration, add:

```java
graphItems.produce(new DesiredStateGraphBuildItem(namespace, name,
        "annotation:" + dsClass.name().toString()));
```

- [ ] **Step 3: Add cross-surface collision validation to DesiredStateAnnotationsProcessor**

Add a new `@BuildStep` method:

```java
@BuildStep
void validateNoDuplicateGraphs(List<DesiredStateGraphBuildItem> graphs) {
    Map<String, List<String>> byQualifiedName = new HashMap<>();
    for (DesiredStateGraphBuildItem item : graphs) {
        byQualifiedName.computeIfAbsent(item.qualifiedName(), k -> new ArrayList<>())
                .add(item.source());
    }
    for (Map.Entry<String, List<String>> entry : byQualifiedName.entrySet()) {
        if (entry.getValue().size() > 1) {
            throw new RuntimeException("Graph '" + entry.getKey()
                    + "' declared by multiple sources: " + entry.getValue());
        }
    }
}
```

- [ ] **Step 4: Create YamlDesiredStateProcessor**

Create `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java`:

```java
package io.casehub.desiredstate.yaml.deployment;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.desiredstate.annotations.deployment.DesiredStateGraphBuildItem;
import io.casehub.desiredstate.annotations.runtime.DependencyDescriptor;
import io.casehub.desiredstate.annotations.runtime.GraphDescriptor;
import io.casehub.desiredstate.annotations.runtime.NodeDescriptor;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeTypeId;
import io.casehub.desiredstate.yaml.YamlGraphRecorder;
import io.casehub.desiredstate.yaml.model.YamlGraph;
import io.casehub.desiredstate.yaml.model.YamlNode;
import io.quarkus.arc.deployment.SyntheticBeanBuildItem;
import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.annotations.Record;
import io.quarkus.deployment.annotations.ExecutionTime;
import io.quarkus.deployment.builditem.CombinedIndexBuildItem;
import io.quarkus.runtime.RuntimeValue;
import org.jboss.jandex.AnnotationInstance;
import org.jboss.jandex.ClassInfo;
import org.jboss.jandex.DotName;
import org.jboss.jandex.IndexView;
import org.jboss.logging.Logger;

import java.io.IOException;
import java.io.InputStream;
import java.net.URL;
import java.util.ArrayList;
import java.util.Enumeration;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

public class YamlDesiredStateProcessor {

    private static final Logger LOG = Logger.getLogger(YamlDesiredStateProcessor.class);
    private static final String YAML_CLASSPATH = "META-INF/desiredstate/";
    private static final DotName NODE_SPEC = DotName.createSimple(NodeSpec.class.getName());
    private static final DotName NODE_TYPE_ID = DotName.createSimple(NodeTypeId.class.getName());

    @BuildStep
    @Record(ExecutionTime.RUNTIME_INIT)
    void discoverYamlGraphs(CombinedIndexBuildItem indexBuildItem,
                            YamlGraphRecorder recorder,
                            BuildProducer<SyntheticBeanBuildItem> syntheticBeans,
                            BuildProducer<DesiredStateGraphBuildItem> graphItems) throws IOException {

        IndexView index = indexBuildItem.getIndex();
        Map<String, String> typeRegistry = scanNodeTypes(index);

        ObjectMapper yamlMapper = new ObjectMapper(new YAMLFactory());
        List<YamlGraph> yamlGraphs = discoverYamlFiles(yamlMapper);

        for (YamlGraph yamlGraph : yamlGraphs) {
            validateYamlGraph(yamlGraph, typeRegistry);

            GraphDescriptor descriptor = toGraphDescriptor(yamlGraph, typeRegistry);

            RuntimeValue<GoalCompiler> compiler = recorder.createYamlGoalCompiler(
                    descriptor, typeRegistry,
                    yamlGraph.variables() != null ? yamlGraph.variables() : Map.of());

            String ns = yamlGraph.desiredState().namespace();
            String name = yamlGraph.desiredState().name();

            syntheticBeans.produce(SyntheticBeanBuildItem.configure(GoalCompiler.class)
                    .addQualifier()
                        .annotation(io.casehub.desiredstate.api.DesiredStateQualifier.class)
                        .addValue("namespace", ns)
                        .addValue("name", name)
                        .done()
                    .runtimeValue(compiler)
                    .done());

            graphItems.produce(new DesiredStateGraphBuildItem(ns, name,
                    "yaml:" + yamlGraph.desiredState().namespace() + ":" + yamlGraph.desiredState().name()));
        }
    }

    private Map<String, String> scanNodeTypes(IndexView index) {
        Map<String, String> registry = new HashMap<>();
        for (AnnotationInstance ann : index.getAnnotations(NODE_TYPE_ID)) {
            if (ann.target().kind() == org.jboss.jandex.AnnotationTarget.Kind.CLASS) {
                ClassInfo cls = ann.target().asClass();
                if (cls.interfaceNames().stream().anyMatch(n -> n.equals(NODE_SPEC))
                        || implementsNodeSpec(cls, index)) {
                    String typeId = ann.value().asString();
                    String existing = registry.put(typeId, cls.name().toString());
                    if (existing != null) {
                        throw new RuntimeException("NodeType '" + typeId
                                + "' claimed by both " + existing + " and " + cls.name());
                    }
                }
            }
        }
        return registry;
    }

    private boolean implementsNodeSpec(ClassInfo cls, IndexView index) {
        return index.getAllKnownImplementors(NODE_SPEC).contains(cls);
    }

    private List<YamlGraph> discoverYamlFiles(ObjectMapper mapper) throws IOException {
        List<YamlGraph> graphs = new ArrayList<>();
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        Enumeration<URL> resources = cl.getResources(YAML_CLASSPATH);
        while (resources.hasMoreElements()) {
            URL dir = resources.nextElement();
            // Scan for .yaml files — implementation handles JAR and file: URLs
        }
        // Also scan individual files
        Enumeration<URL> yamlFiles = cl.getResources(YAML_CLASSPATH);
        // Discovery implementation scans classpath for META-INF/desiredstate/*.yaml
        // Full implementation uses Quarkus ApplicationArchivesBuildItem for reliable classpath scanning
        return graphs;
    }

    private void validateYamlGraph(YamlGraph graph, Map<String, String> typeRegistry) {
        if (graph.desiredState() == null || graph.desiredState().namespace() == null
                || graph.desiredState().name() == null) {
            throw new RuntimeException("desiredState.namespace and desiredState.name are required");
        }

        Set<String> nodeIds = new HashSet<>();
        for (Map.Entry<String, YamlNode> entry : graph.nodes().entrySet()) {
            String nodeId = entry.getKey();
            YamlNode node = entry.getValue();

            if (!nodeIds.add(nodeId)) {
                throw new RuntimeException("Duplicate node ID: '" + nodeId + "'");
            }
            if (!typeRegistry.containsKey(node.type())) {
                throw new RuntimeException("Unknown node type '" + node.type()
                        + "' for node '" + nodeId + "'. Available: " + typeRegistry.keySet());
            }
        }

        // Validate dependsOn references
        for (Map.Entry<String, YamlNode> entry : graph.nodes().entrySet()) {
            for (String dep : entry.getValue().dependsOn()) {
                if (!nodeIds.contains(dep)) {
                    throw new RuntimeException("Node '" + entry.getKey()
                            + "' depends on '" + dep + "' which is not declared");
                }
            }
        }

        // Cycle detection via topological sort
        detectCycles(graph.nodes());
    }

    private void detectCycles(Map<String, YamlNode> nodes) {
        Map<String, Integer> inDegree = new HashMap<>();
        Map<String, List<String>> adjList = new HashMap<>();
        for (String id : nodes.keySet()) {
            inDegree.put(id, 0);
            adjList.put(id, new ArrayList<>());
        }
        for (Map.Entry<String, YamlNode> entry : nodes.entrySet()) {
            for (String dep : entry.getValue().dependsOn()) {
                adjList.get(dep).add(entry.getKey());
                inDegree.merge(entry.getKey(), 1, Integer::sum);
            }
        }

        List<String> queue = new ArrayList<>();
        for (Map.Entry<String, Integer> e : inDegree.entrySet()) {
            if (e.getValue() == 0) queue.add(e.getKey());
        }

        int processed = 0;
        int idx = 0;
        while (idx < queue.size()) {
            String node = queue.get(idx++);
            processed++;
            for (String dependent : adjList.get(node)) {
                if (inDegree.merge(dependent, -1, Integer::sum) == 0) {
                    queue.add(dependent);
                }
            }
        }

        if (processed < nodes.size()) {
            Set<String> cyclic = new HashSet<>(nodes.keySet());
            cyclic.removeAll(new HashSet<>(queue));
            throw new RuntimeException("Cyclic dependency detected involving nodes: " + cyclic);
        }
    }

    private GraphDescriptor toGraphDescriptor(YamlGraph yamlGraph, Map<String, String> typeRegistry) {
        List<NodeDescriptor> nodes = new ArrayList<>();
        List<DependencyDescriptor> deps = new ArrayList<>();

        for (Map.Entry<String, YamlNode> entry : yamlGraph.nodes().entrySet()) {
            String nodeId = entry.getKey();
            YamlNode yamlNode = entry.getValue();
            String specClassName = typeRegistry.get(yamlNode.type());

            nodes.add(new NodeDescriptor.InlineNode(
                    nodeId, specClassName,
                    yamlNode.spec() != null ? yamlNode.spec() : Map.of(),
                    yamlNode.humanGating()));

            for (String dep : yamlNode.dependsOn()) {
                deps.add(new DependencyDescriptor(nodeId, dep));
            }
        }

        return new GraphDescriptor(
                yamlGraph.desiredState().namespace(),
                yamlGraph.desiredState().name(),
                null, null, nodes, deps,
                List.of(), null, List.of(), List.of());
    }
}
```

Note: The `discoverYamlFiles` method needs full implementation using Quarkus `HotDeploymentWatchedFileBuildItem` or classpath scanning. The exact implementation depends on Quarkus build infrastructure — implement during coding.

- [ ] **Step 5: Verify annotations module still compiles**

Run: `mvn --batch-mode install -pl annotations/deployment`
Expected: compiles successfully with the new DesiredStateGraphBuildItem and updated processor

- [ ] **Step 6: Write build extension test**

Create `yaml/deployment/src/test/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessorTest.java`. This is a Quarkus deployment test that verifies the build extension discovers YAML files and creates GoalCompiler beans. The exact test pattern follows `DesiredStateAnnotationsProcessorTest` in `annotations/deployment/`.

```java
package io.casehub.desiredstate.yaml.deployment;

import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.GoalCompiler;
import io.quarkus.test.QuarkusUnitTest;
import jakarta.inject.Inject;
import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.asset.StringAsset;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import static org.assertj.core.api.Assertions.assertThat;

class YamlDesiredStateProcessorTest {

    // Test details depend on exact Quarkus test infrastructure setup —
    // implement during coding with the QuarkusUnitTest pattern from
    // annotations/deployment tests
}
```

- [ ] **Step 7: Verify yaml module builds**

Run: `mvn --batch-mode install -pl yaml/runtime,yaml/deployment`
Expected: builds successfully

- [ ] **Step 8: Commit**

```bash
git add yaml/deployment/ annotations/deployment/
git commit -m "feat(#117): YamlDesiredStateProcessor, DesiredStateGraphBuildItem, build-time validation"
```

---

### Task 5: Pipeline YAML example and integration test

**Files:**
- Create: `examples/pipeline-yaml/pom.xml`
- Create: `examples/pipeline-yaml/src/main/resources/META-INF/desiredstate/medallion-pipeline.yaml`
- Create: `examples/pipeline-yaml/src/test/java/io/casehub/desiredstate/example/pipeline/yaml/PipelineYamlTest.java`

**Interfaces:**
- Consumes: `YamlDesiredStateProcessor` (Task 4), all pipeline NodeSpec classes with `@NodeTypeId` (Task 1)
- Produces: Working end-to-end YAML pipeline with integration test

- [ ] **Step 1: Create pipeline-yaml POM**

Create `examples/pipeline-yaml/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-desiredstate-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
        <relativePath>../../pom.xml</relativePath>
    </parent>

    <artifactId>casehub-desiredstate-example-pipeline-yaml</artifactId>
    <name>CaseHub Desired State :: Example :: Pipeline YAML</name>
    <description>Teaching example — YAML-driven medallion architecture pipeline,
        side-by-side companion to pipeline-annotated.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-yaml</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-example-pipeline</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>

        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5-internal</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>${jandex-maven-plugin.version}</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Create medallion-pipeline.yaml**

Create `examples/pipeline-yaml/src/main/resources/META-INF/desiredstate/medallion-pipeline.yaml`:

```yaml
desiredState:
  namespace: pipeline-yaml
  name: medallion

variables:
  batch_size: "1000"
  source_uri: s3://data/customers.csv

nodes:
  csv-source:
    type: data-source
    spec:
      name: customers
      format: CSV
      uri: ${source_uri}

  customer-schema:
    type: schema
    spec:
      name: customer-schema
      fields: [id, name, email]
      version: 1

  csv-ingest:
    type: ingestion
    dependsOn: [csv-source]
    spec:
      sourceRef: csv-source
      batchSize: ${batch_size}
      format: CSV

  dedup-cleanser:
    type: cleanser
    dependsOn: [csv-ingest, customer-schema]
    spec:
      rules: [dedup, nullcheck]
      deduplication: true
      nullHandling: DROP

  geo-enricher:
    type: enricher
    dependsOn: [dedup-cleanser]
    spec:
      lookupSource: geo-lookup
      joinKeys: [address]
      enrichFields: [lat, lon]

  quality-validator:
    type: validator
    dependsOn: [geo-enricher, customer-schema]
    spec:
      schemaRef: customer-schema
      qualityThreshold: 0.95
      anomalyDetection: true

  aggregate-tx:
    type: transformer
    dependsOn: [quality-validator]
    humanGating: PROVISION_ONLY
    spec:
      aggregations: [sum, avg]
      reshapeRules: []
      outputFormat: parquet
      approvalRequired: true

  warehouse-sink:
    type: sink
    dependsOn: [aggregate-tx]
    humanGating: PROVISION_ONLY
    spec:
      destination: s3://warehouse/gold/
      format: parquet
      partitionKeys: [date]
      approvalRequired: true
```

Note: namespace is `pipeline-yaml` (not `pipeline`) to avoid collision with the annotation-path `MedallionPipeline` if both are ever on the same classpath.

- [ ] **Step 3: Write integration test**

Create `examples/pipeline-yaml/src/test/java/io/casehub/desiredstate/example/pipeline/yaml/PipelineYamlTest.java`:

```java
package io.casehub.desiredstate.example.pipeline.yaml;

import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.Dependency;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.example.pipeline.DataSourceSpec;
import io.casehub.desiredstate.example.pipeline.IngestionSpec;
import io.casehub.desiredstate.example.pipeline.SinkSpec;
import io.casehub.desiredstate.example.pipeline.TransformerSpec;
import io.casehub.desiredstate.runtime.ImmutableDesiredStateGraph;
import io.quarkus.test.QuarkusUnitTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import static org.assertj.core.api.Assertions.assertThat;

class PipelineYamlTest {

    @RegisterExtension
    static final QuarkusUnitTest config = new QuarkusUnitTest()
            .withApplicationRoot(jar -> jar
                    .addAsResource("META-INF/desiredstate/medallion-pipeline.yaml"));

    // GoalCompiler injected via @DesiredStateQualifier — exact injection
    // pattern depends on CDI qualifier setup, implement during coding

    @Test
    void yamlGraphHasAllEightNodes() {
        // Compile the YAML graph and verify 8 nodes
        // Implementation details depend on CDI wiring
    }

    @Test
    void dependencyEdgesMatchAnnotationPath() {
        // Verify csv-ingest depends on csv-source, etc.
    }

    @Test
    void specFieldsAreCorrectlyDeserialized() {
        // Verify DataSourceSpec fields match YAML values
    }

    @Test
    void variableSubstitutionWorks() {
        // Verify ${source_uri} resolved to s3://data/customers.csv
    }

    @Test
    void humanGatingIsApplied() {
        // Verify aggregate-tx has PROVISION_ONLY gating
    }
}
```

Note: Test method bodies are sketched — full assertions require CDI qualification wiring that will be determined during implementation.

- [ ] **Step 4: Run integration test**

Run: `mvn --batch-mode test -pl examples/pipeline-yaml`
Expected: all tests PASS

- [ ] **Step 5: Run full build**

Run: `mvn --batch-mode install`
Expected: full project builds, all tests pass across all modules

- [ ] **Step 6: Commit**

```bash
git add examples/pipeline-yaml/ pom.xml
git commit -m "feat(#117): pipeline YAML example with integration tests"
```

---

## References

- `specs/issue-117-yaml-surface-foundation/2026-08-27-yaml-surface-foundation-design.md` — design spec this plan implements
- `annotations/runtime/src/main/java/.../NodeDescriptor.java:5` — sealed interface to extend
- `annotations/runtime/src/main/java/.../DesiredStateGraphRecorder.java:292` — exhaustive switch to fix
- `annotations/runtime/src/main/java/.../GraphDescriptor.java:5` — IR record
- `annotations/deployment/src/main/java/.../DesiredStateAnnotationsProcessor.java:88` — build extension pattern
- `api/src/main/java/.../NodeSpec.java:3` — runtime contract
- `api/src/main/java/.../GoalCompiler.java:3` — compilation interface
- `examples/pipeline/src/main/java/.../DataSourceSpec.java:6` — NodeSpec to annotate
- `examples/pipeline-annotated/pom.xml` — example module POM pattern
- [GitHub #117](https://github.com/casehubio/casehub-desiredstate/issues/117)
- [GitHub #116](https://github.com/casehubio/casehub-desiredstate/issues/116) — parent epic
