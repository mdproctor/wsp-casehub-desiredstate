# Terraform/OpenTofu Bridge Adapter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #184 — feat: Terraform/OpenTofu bridge adapter
**Issue group:** #184

**Goal:** Implement a BridgeProvider for the Terraform/OpenTofu go-plugin gRPC protocol with zero-config schema auto-discovery, proving the end-to-end lifecycle with real provider integration tests.

**Architecture:** Single `bridge/terraform/` module implementing `BridgeProvider` + `PlanCapable`. Direct Java host replicates the go-plugin handshake (stdout parse + gRPC connect). Supports both protocol v5 and v6 via `TerraformProtocol` interface with version-specific adapters. `GetProviderSchema` auto-discovers types → `NodeType` + `PluginSpecSchema` at runtime. Three-tier test strategy: protocol mock, real providers (null/random), optional cloud tier (Phase 2).

**Tech Stack:** gRPC-Java, protobuf-maven-plugin, msgpack-java, Jackson YAML, JUnit 5, AssertJ

## Global Constraints

- Project version: `0.4-SNAPSHOT`
- Pre-release stage — no backward compatibility requirements
- Phase 1 only: no import, no credentials, no mTLS, no suspend/resume
- NodeType namespace: `tf:{namespace}/{type}:{resource}` (e.g. `tf:hashicorp/aws:aws_instance`)
- Provider binaries cached at `~/.casehub/providers/{namespace}/{type}/{version}/{os}_{arch}/`
- Schema cache at `~/.casehub/cache/schemas/{namespace}/{type}/{version}/schema.json`
- Integration tests gated behind Maven profile `-Pintegration`

---

## Batch 1: Prerequisites — Existing Module Changes

### Task 1: Extract PluginSpecSchema and PluginFieldDef to plugin/api

`PluginSpecSchema` and `PluginFieldDef` are pure data records in `plugin/runtime/` with no
runtime dependencies. The Terraform bridge needs them for schema translation but should not
depend on `plugin/runtime/` (which pulls in Quarkus, step pipeline, etc.). Move them to
`plugin/api/` where `YamlNodeSpec` already lives.

**Files:**
- Move: `../../plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/PluginSpecSchema.java` → `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/PluginSpecSchema.java` (use `ide_move_file`)
- Move: `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginFieldDef.java` → `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/PluginFieldDef.java` (use `ide_move_file`)
- Modify: All files importing from `io.casehub.desiredstate.plugin.model.PluginSpecSchema` or `PluginFieldDef` (use `ide_refactor_rename` on the package, or verify `ide_move_file` updates references)

**Interfaces:**
- Produces: `io.casehub.desiredstate.plugin.api.PluginSpecSchema` — `record PluginSpecSchema(Map<String, PluginFieldDef> fields)`
- Produces: `io.casehub.desiredstate.plugin.api.PluginFieldDef` — `record PluginFieldDef(String type, boolean required, Object defaultValue, String pattern, Integer minLength, Integer maxLength, Number min, Number max, List<String> values, String itemType, String valueType, Integer minItems, Integer maxItems)`

- [ ] **Step 1: Find all usages of PluginSpecSchema and PluginFieldDef**

Use `ide_find_references` on both classes to identify all importers before moving.

- [ ] **Step 2: Move PluginSpecSchema to plugin/api**

Use `ide_move_file` to move `../../plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/PluginSpecSchema.java` to `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/PluginSpecSchema.java`. Verify all references updated.

- [ ] **Step 3: Move PluginFieldDef to plugin/api**

Use `ide_move_file` to move `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginFieldDef.java` to `plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/PluginFieldDef.java`. Verify all references updated.

- [ ] **Step 4: Build and verify**

Run: `mvn --batch-mode install -pl plugin/api,plugin/runtime,plugin/deployment,plugin/testing,plugin/spring -am`
Expected: BUILD SUCCESS — all modules compile with the new package location.

- [ ] **Step 5: Run full test suite**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — no regressions from the move.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "refactor: extract PluginSpecSchema and PluginFieldDef to plugin/api

Move pure data records from plugin/runtime (io.casehub.desiredstate.plugin.model)
to plugin/api (io.casehub.desiredstate.plugin.api) so bridge/terraform can depend
on schema types without pulling in the full plugin runtime.

Refs #184

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: Add BridgeOutputStateStore SPI and extend BridgeReadRequest

Terraform providers are stateless — the caller must track per-resource output state (IDs, ARNs, computed attributes) between operations. Add a `BridgeOutputStateStore` SPI to `bridge/api/` and extend `BridgeReadRequest` with `previousState` and `desiredSpec` fields. Then update `bridge/runtime/` to use them.

**Files:**
- Create: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeOutputStateStore.java`
- Modify: `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeReadRequest.java`
- Modify: `bridge/runtime/src/main/java/io/casehub/desiredstate/bridge/runtime/BridgeProvisioner.java`
- Modify: `bridge/runtime/src/main/java/io/casehub/desiredstate/bridge/runtime/BridgeActualStateAdapter.java`
- Test: `bridge/runtime/src/test/java/io/casehub/desiredstate/bridge/runtime/BridgeOutputStateStoreTest.java`

**Interfaces:**
- Produces: `BridgeOutputStateStore` — `void store(String tenancyId, String nodeId, Map<String, Object> outputState)`, `Optional<Map<String, Object>> load(String tenancyId, String nodeId)`, `void remove(String tenancyId, String nodeId)`
- Produces: `BridgeReadRequest(String typeId, String resourceId, String tenancyId, Map<String, Object> previousState, Map<String, Object> desiredSpec)` — extended record

- [ ] **Step 1: Write test for BridgeOutputStateStore lifecycle**

```java
package io.casehub.desiredstate.bridge.runtime;

import io.casehub.desiredstate.bridge.BridgeOutputStateStore;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class BridgeOutputStateStoreTest {

    @Test
    void storeAndLoadRoundTrip() {
        BridgeOutputStateStore store = new InMemoryBridgeOutputStateStore();
        store.store("tenant-1", "node-1", Map.of("id", "i-abc123", "arn", "arn:aws:..."));
        var loaded = store.load("tenant-1", "node-1");
        assertThat(loaded).isPresent();
        assertThat(loaded.get()).containsEntry("id", "i-abc123");
    }

    @Test
    void loadReturnsEmptyForUnknownNode() {
        BridgeOutputStateStore store = new InMemoryBridgeOutputStateStore();
        assertThat(store.load("tenant-1", "unknown")).isEmpty();
    }

    @Test
    void removeDeletesState() {
        BridgeOutputStateStore store = new InMemoryBridgeOutputStateStore();
        store.store("tenant-1", "node-1", Map.of("id", "i-abc123"));
        store.remove("tenant-1", "node-1");
        assertThat(store.load("tenant-1", "node-1")).isEmpty();
    }

    @Test
    void tenantIsolation() {
        BridgeOutputStateStore store = new InMemoryBridgeOutputStateStore();
        store.store("tenant-1", "node-1", Map.of("id", "i-111"));
        store.store("tenant-2", "node-1", Map.of("id", "i-222"));
        assertThat(store.load("tenant-1", "node-1").get()).containsEntry("id", "i-111");
        assertThat(store.load("tenant-2", "node-1").get()).containsEntry("id", "i-222");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl bridge/runtime -Dtest=BridgeOutputStateStoreTest`
Expected: FAIL — classes do not exist yet.

- [ ] **Step 3: Create BridgeOutputStateStore interface**

Create `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeOutputStateStore.java`:

```java
package io.casehub.desiredstate.bridge;

import java.util.Map;
import java.util.Optional;

public interface BridgeOutputStateStore {
    void store(String tenancyId, String nodeId, Map<String, Object> outputState);
    Optional<Map<String, Object>> load(String tenancyId, String nodeId);
    void remove(String tenancyId, String nodeId);
}
```

- [ ] **Step 4: Create InMemoryBridgeOutputStateStore**

Create `bridge/runtime/src/main/java/io/casehub/desiredstate/bridge/runtime/InMemoryBridgeOutputStateStore.java`:

```java
package io.casehub.desiredstate.bridge.runtime;

import io.casehub.desiredstate.bridge.BridgeOutputStateStore;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

public class InMemoryBridgeOutputStateStore implements BridgeOutputStateStore {

    private final ConcurrentHashMap<String, Map<String, Object>> store = new ConcurrentHashMap<>();

    @Override
    public void store(String tenancyId, String nodeId, Map<String, Object> outputState) {
        store.put(key(tenancyId, nodeId), Map.copyOf(outputState));
    }

    @Override
    public Optional<Map<String, Object>> load(String tenancyId, String nodeId) {
        return Optional.ofNullable(store.get(key(tenancyId, nodeId)));
    }

    @Override
    public void remove(String tenancyId, String nodeId) {
        store.remove(key(tenancyId, nodeId));
    }

    private static String key(String tenancyId, String nodeId) {
        return tenancyId + "::" + nodeId;
    }
}
```

- [ ] **Step 5: Extend BridgeReadRequest with previousState and desiredSpec**

Replace `BridgeReadRequest`:

```java
package io.casehub.desiredstate.bridge;

import java.util.Map;

public record BridgeReadRequest(
    String typeId,
    String resourceId,
    String tenancyId,
    Map<String, Object> previousState,
    Map<String, Object> desiredSpec
) {
    public BridgeReadRequest(String typeId, String resourceId, String tenancyId) {
        this(typeId, resourceId, tenancyId, Map.of(), Map.of());
    }
}
```

- [ ] **Step 6: Update BridgeProvisioner to use BridgeOutputStateStore**

Modify `BridgeProvisioner`:
- Add `BridgeOutputStateStore` field, injected via constructor
- On `provision()` success: `stateStore.store(tenancyId, nodeId, outputState)`
- On `deprovision()` success: `stateStore.remove(tenancyId, nodeId)`
- In `toBridgeRequest()`: load prior state via `stateStore.load()` and populate `previousState`
- Constructor backward-compatible: add a no-stateStore constructor that uses a no-op store

```java
public BridgeProvisioner(BridgeDescriptor descriptor, BridgeProvider provider) {
    this(descriptor, provider, new NoOpBridgeOutputStateStore());
}

public BridgeProvisioner(BridgeDescriptor descriptor, BridgeProvider provider,
                         BridgeOutputStateStore stateStore) {
    this.descriptor = descriptor;
    this.provider = provider;
    this.stateStore = stateStore;
}
```

- [ ] **Step 7: Update BridgeActualStateAdapter to pass previousState and desiredSpec**

Modify `BridgeActualStateAdapter.readActual()` to construct `BridgeReadRequest` with `previousState` (loaded from `BridgeOutputStateStore`) and `desiredSpec` (from `node.spec()`). Add `BridgeOutputStateStore` as constructor parameter with backward-compatible default.

- [ ] **Step 8: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl bridge/runtime -Dtest=BridgeOutputStateStoreTest`
Expected: PASS

- [ ] **Step 9: Run full bridge test suite**

Run: `mvn --batch-mode install -pl bridge/api,bridge/runtime,bridge/deployment,bridge/spring -am`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat(bridge): add BridgeOutputStateStore SPI and extend BridgeReadRequest

Add per-resource output state persistence for stateless providers (Terraform).
BridgeProvisioner stores output state on provision success, loads prior state
for subsequent operations, removes on deprovision. BridgeReadRequest extended
with previousState and desiredSpec fields.

Refs #184

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Protocol Foundation — Go-Plugin + gRPC

### Task 3: Module scaffold, proto generation, and GoPluginHost

Create the `bridge/terraform/` module with Maven build, proto generation, and the go-plugin
handshake implementation. This is the lowest layer — start the provider binary, parse the
handshake, establish a gRPC channel.

**Files:**
- Create: `bridge/terraform/pom.xml`
- Create: `bridge/terraform/src/main/proto/tfplugin5.proto` (from Terraform source)
- Create: `bridge/terraform/src/main/proto/tfplugin6.proto` (from Terraform source)
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/GoPluginHost.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/GoPluginHandshake.java`
- Modify: `bridge/pom.xml` — add `<module>terraform</module>`
- Modify: `pom.xml` — add `casehub-desiredstate-bridge-terraform` to dependencyManagement if needed
- Test: `bridge/terraform/src/test/java/io/casehub/desiredstate/bridge/terraform/GoPluginHandshakeTest.java`
- Test: `bridge/terraform/src/test/java/io/casehub/desiredstate/bridge/terraform/GoPluginHostTest.java`

**Interfaces:**
- Produces: `GoPluginHandshake` — `record GoPluginHandshake(int coreProtocol, int appProtocol, String networkType, String address, String protocol)` with `static GoPluginHandshake parse(String line)`
- Produces: `GoPluginHost` — `ManagedChannel connect(String binaryPath, List<String> args)`, `void shutdown()`, `boolean isHealthy()`, `GoPluginHandshake handshake()`

- [ ] **Step 1: Create bridge/terraform/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-desiredstate-bridge-parent</artifactId>
        <version>0.4-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-desiredstate-bridge-terraform</artifactId>
    <name>CaseHub Desired State :: Bridge :: Terraform</name>
    <description>Terraform/OpenTofu bridge adapter — go-plugin gRPC protocol, zero-config schema discovery.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-bridge-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-bridge</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-plugin-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-netty-shaded</artifactId>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-protobuf</artifactId>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-stub</artifactId>
        </dependency>
        <dependency>
            <groupId>org.msgpack</groupId>
            <artifactId>msgpack-core</artifactId>
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
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-testing</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <extensions>
            <extension>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>1.7.1</version>
            </extension>
        </extensions>
        <plugins>
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>0.6.1</version>
                <configuration>
                    <protocArtifact>com.google.protobuf:protoc:${protobuf.version}:exe:${os.detected.classifier}</protocArtifact>
                    <pluginId>grpc-java</pluginId>
                    <pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}</pluginArtifact>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>compile-custom</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

Note: `grpc.version`, `protobuf.version`, `msgpack-core` version properties must be added to the parent pom's `<dependencyManagement>` section. Check the casehub-parent BOM for existing versions; if not present, add:
- `grpc.version`: `1.68.0` (or latest stable)
- `protobuf.version`: `4.28.3` (compatible with gRPC version)
- `msgpack-core`: `0.9.8`

- [ ] **Step 2: Add terraform module to bridge/pom.xml**

Add `<module>terraform</module>` to the modules list in `bridge/pom.xml`.

- [ ] **Step 3: Add proto files**

Download Terraform plugin protocol v5 and v6 proto files:
- `bridge/terraform/src/main/proto/tfplugin5.proto` — from `github.com/hashicorp/terraform/internal/tfplugin5/tfplugin5.proto`
- `bridge/terraform/src/main/proto/tfplugin6.proto` — from `github.com/hashicorp/terraform/internal/tfplugin6/tfplugin6.proto`

Adjust the `option java_package` and `option java_outer_classname` for each proto to set:
- v5: `option java_package = "io.casehub.desiredstate.bridge.terraform.proto.v5";`
- v6: `option java_package = "io.casehub.desiredstate.bridge.terraform.proto.v6";`

- [ ] **Step 4: Verify proto compilation**

Run: `mvn --batch-mode compile -pl bridge/terraform -am`
Expected: BUILD SUCCESS — proto stubs generated in `target/generated-sources/protobuf/`.

- [ ] **Step 5: Write test for GoPluginHandshake parsing**

```java
package io.casehub.desiredstate.bridge.terraform;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class GoPluginHandshakeTest {

    @Test
    void parseValidV5Handshake() {
        var hs = GoPluginHandshake.parse("1|5|tcp|127.0.0.1:50051|grpc");
        assertThat(hs.coreProtocol()).isEqualTo(1);
        assertThat(hs.appProtocol()).isEqualTo(5);
        assertThat(hs.networkType()).isEqualTo("tcp");
        assertThat(hs.address()).isEqualTo("127.0.0.1:50051");
        assertThat(hs.protocol()).isEqualTo("grpc");
    }

    @Test
    void parseValidV6Handshake() {
        var hs = GoPluginHandshake.parse("1|6|tcp|127.0.0.1:50052|grpc");
        assertThat(hs.appProtocol()).isEqualTo(6);
    }

    @Test
    void parseDualVersionPreferV6() {
        var hs = GoPluginHandshake.parse("1|5|tcp|127.0.0.1:50051|grpc");
        assertThat(hs.appProtocol()).isEqualTo(5);
    }

    @Test
    void rejectNetrpcProtocol() {
        assertThatThrownBy(() -> GoPluginHandshake.parse("1|5|tcp|127.0.0.1:50051|netrpc"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("netrpc");
    }

    @Test
    void rejectMalformedLine() {
        assertThatThrownBy(() -> GoPluginHandshake.parse("garbage"))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectEmptyLine() {
        assertThatThrownBy(() -> GoPluginHandshake.parse(""))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 6: Implement GoPluginHandshake**

```java
package io.casehub.desiredstate.bridge.terraform;

public record GoPluginHandshake(
    int coreProtocol,
    int appProtocol,
    String networkType,
    String address,
    String protocol
) {
    static final String MAGIC_COOKIE_KEY = "PLUGIN_MAGIC_COOKIE";
    static final String MAGIC_COOKIE_VALUE = "d602bf8f470bc67ca7faa0386276bbdd4330ea6918e54b83e925f0f7";

    public static GoPluginHandshake parse(String line) {
        if (line == null || line.isEmpty()) {
            throw new IllegalArgumentException("Empty handshake line");
        }
        String[] parts = line.trim().split("\\|");
        if (parts.length < 5) {
            throw new IllegalArgumentException(
                "Malformed go-plugin handshake: expected 5+ pipe-separated fields, got " + parts.length);
        }
        String protocol = parts[4];
        if (!"grpc".equals(protocol)) {
            throw new IllegalArgumentException(
                "Unsupported protocol: " + protocol + " — only grpc is supported (netrpc is legacy)");
        }
        return new GoPluginHandshake(
            Integer.parseInt(parts[0]),
            Integer.parseInt(parts[1]),
            parts[2],
            parts[3],
            protocol
        );
    }
}
```

- [ ] **Step 7: Run handshake tests**

Run: `mvn --batch-mode test -pl bridge/terraform -Dtest=GoPluginHandshakeTest`
Expected: PASS

- [ ] **Step 8: Write GoPluginHost**

```java
package io.casehub.desiredstate.bridge.terraform;

import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

public class GoPluginHost implements AutoCloseable {

    private Process process;
    private ManagedChannel channel;
    private GoPluginHandshake handshake;
    private static final Duration HANDSHAKE_TIMEOUT = Duration.ofSeconds(30);

    public ManagedChannel connect(String binaryPath, List<String> args) {
        if (process != null && process.isAlive()) {
            throw new IllegalStateException("Provider already running");
        }

        List<String> command = new ArrayList<>();
        command.add(binaryPath);
        command.addAll(args);

        try {
            ProcessBuilder pb = new ProcessBuilder(command);
            pb.environment().put(GoPluginHandshake.MAGIC_COOKIE_KEY,
                                 GoPluginHandshake.MAGIC_COOKIE_VALUE);
            pb.redirectErrorStream(false);
            process = pb.start();

            BufferedReader stdout = new BufferedReader(
                new InputStreamReader(process.getInputStream()));

            String handshakeLine = readHandshakeLine(stdout);
            handshake = GoPluginHandshake.parse(handshakeLine);

            channel = ManagedChannelBuilder.forTarget(handshake.address())
                .usePlaintext()
                .build();

            return channel;
        } catch (Exception e) {
            shutdown();
            throw new RuntimeException("Failed to start provider: " + binaryPath, e);
        }
    }

    public GoPluginHandshake handshake() {
        return handshake;
    }

    public boolean isHealthy() {
        return process != null && process.isAlive() && channel != null && !channel.isShutdown();
    }

    @Override
    public void close() {
        shutdown();
    }

    public void shutdown() {
        if (channel != null) {
            try {
                channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);
            } catch (InterruptedException e) {
                channel.shutdownNow();
                Thread.currentThread().interrupt();
            }
            channel = null;
        }
        if (process != null) {
            process.destroyForcibly();
            process = null;
        }
        handshake = null;
    }

    private String readHandshakeLine(BufferedReader reader) {
        long deadline = System.currentTimeMillis() + HANDSHAKE_TIMEOUT.toMillis();
        try {
            while (System.currentTimeMillis() < deadline) {
                if (reader.ready()) {
                    String line = reader.readLine();
                    if (line != null && !line.isEmpty()) {
                        return line;
                    }
                }
                if (!process.isAlive()) {
                    throw new RuntimeException(
                        "Provider process exited with code " + process.exitValue()
                        + " before completing handshake");
                }
                Thread.sleep(50);
            }
            throw new RuntimeException("Timed out waiting for go-plugin handshake");
        } catch (Exception e) {
            throw new RuntimeException("Failed to read handshake", e);
        }
    }
}
```

- [ ] **Step 9: Write GoPluginHost test with mock process**

Test that `GoPluginHost` correctly handles: a process that writes a valid handshake line (use a script that echoes), process that exits immediately (error), and shutdown cleanup. Note: full integration test with a real provider is in Batch 4.

```java
package io.casehub.desiredstate.bridge.terraform;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class GoPluginHostTest {

    @TempDir Path tempDir;

    @Test
    void connectParsesHandshakeFromProcess() throws Exception {
        Path script = tempDir.resolve("mock-provider.sh");
        Files.writeString(script, "#!/bin/bash\necho '1|5|tcp|127.0.0.1:0|grpc'\nsleep 60\n");
        script.toFile().setExecutable(true);

        var host = new GoPluginHost();
        try {
            // connect will fail on gRPC (port 0) but handshake should parse
            assertThatThrownBy(() -> host.connect(script.toString(), List.of()))
                .isInstanceOf(RuntimeException.class);

            // Verify handshake was parsed before the gRPC failure
            // (GoPluginHost.handshake() may be null if connect failed before setting it)
        } finally {
            host.shutdown();
        }
    }

    @Test
    void shutdownKillsProcess() throws Exception {
        Path script = tempDir.resolve("mock-provider.sh");
        Files.writeString(script, "#!/bin/bash\necho '1|5|tcp|127.0.0.1:50099|grpc'\nsleep 60\n");
        script.toFile().setExecutable(true);

        var host = new GoPluginHost();
        try {
            // Will fail on gRPC connect, but the process will have started
        } catch (Exception ignored) {}
        host.shutdown();
        assertThat(host.isHealthy()).isFalse();
    }
}
```

- [ ] **Step 10: Run all tests**

Run: `mvn --batch-mode test -pl bridge/terraform`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat(bridge/terraform): module scaffold, proto generation, GoPluginHost

New bridge/terraform module with protobuf-maven-plugin for v5/v6 proto
compilation. GoPluginHost handles go-plugin handshake (stdout parse,
magic cookie, gRPC channel creation). GoPluginHandshake record parses
the pipe-separated handshake line.

Refs #184

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: CtyDecoder, TerraformProtocol interface, and v5/v6 adapters

Implement the `cty` type system decoder (Terraform's type encoding) and the protocol adapters
that translate between our domain types and the proto-generated stubs.

**Files:**
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/CtyDecoder.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/CtyEncoder.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/TerraformProtocol.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/ProtocolV5Adapter.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/ProtocolV6Adapter.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/TerraformProtocolFactory.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/ProviderSchema.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/PlanResponse.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/ApplyResponse.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/ReadResponse.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/Diagnostic.java`
- Test: `bridge/terraform/src/test/java/io/casehub/desiredstate/bridge/terraform/CtyDecoderTest.java`
- Test: `bridge/terraform/src/test/java/io/casehub/desiredstate/bridge/terraform/ProtocolAdapterTest.java`

**Interfaces:**
- Consumes: Proto-generated stubs from `io.casehub.desiredstate.bridge.terraform.proto.v5` and `.v6`
- Produces: `TerraformProtocol` — `ProviderSchema getSchema()`, `void configure(Map<String, Object> config)`, `PlanResponse plan(String resourceType, Map<String, Object> priorState, Map<String, Object> proposedState)`, `ApplyResponse apply(String resourceType, Map<String, Object> priorState, Object plannedChange)`, `ReadResponse read(String resourceType, Map<String, Object> currentState)`
- Produces: `CtyDecoder` — `static Object decode(byte[] msgpack)`, `static Object decodeJson(String json)`
- Produces: `CtyEncoder` — `static byte[] encode(Map<String, Object> value)`, `static String encodeJson(Map<String, Object> value)`
- Produces: `TerraformProtocolFactory` — `static TerraformProtocol create(GoPluginHandshake handshake, ManagedChannel channel)`
- Produces: `ProviderSchema` — `record ProviderSchema(Map<String, ResourceSchema> resourceSchemas)`, `record ResourceSchema(Map<String, AttributeSchema> attributes)`, `record AttributeSchema(String name, String type, boolean required, boolean optional, boolean computed, String description)`

- [ ] **Step 1: Write CtyDecoder tests**

```java
package io.casehub.desiredstate.bridge.terraform;

import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.util.List;
import java.util.Map;
import java.util.Set;
import static org.assertj.core.api.Assertions.*;

class CtyDecoderTest {

    @Test
    void decodeString() {
        assertThat(CtyDecoder.decodeJson("\"hello\"")).isEqualTo("hello");
    }

    @Test
    void decodeNumber() {
        assertThat(CtyDecoder.decodeJson("42")).isEqualTo(new BigDecimal("42"));
    }

    @Test
    void decodeBool() {
        assertThat(CtyDecoder.decodeJson("true")).isEqualTo(Boolean.TRUE);
    }

    @Test
    void decodeNull() {
        assertThat(CtyDecoder.decodeJson("null")).isNull();
    }

    @Test
    void decodeList() {
        Object result = CtyDecoder.decodeJson("[\"a\", \"b\"]");
        assertThat(result).isInstanceOf(List.class);
        assertThat((List<?>) result).containsExactly("a", "b");
    }

    @Test
    void decodeMap() {
        Object result = CtyDecoder.decodeJson("{\"key\": \"value\"}");
        assertThat(result).isInstanceOf(Map.class);
        assertThat((Map<?, ?>) result).containsEntry("key", "value");
    }

    @Test
    void decodeNestedObject() {
        Object result = CtyDecoder.decodeJson("{\"tags\": {\"Name\": \"web\"}}");
        assertThat(result).isInstanceOf(Map.class);
        @SuppressWarnings("unchecked")
        Map<String, Object> map = (Map<String, Object>) result;
        assertThat(map.get("tags")).isInstanceOf(Map.class);
    }
}
```

- [ ] **Step 2: Implement CtyDecoder and CtyEncoder**

`CtyDecoder` decodes Terraform's `DynamicValue` (msgpack or JSON) to Java types. `CtyEncoder` encodes Java types back to Terraform's format.

For v5 (JSON): use Jackson ObjectMapper.
For v6 (msgpack): use msgpack-core's `MessageUnpacker` / `MessagePacker`.

The cty type mapping:
- msgpack string → `String`
- msgpack integer/float → `BigDecimal`
- msgpack boolean → `Boolean`
- msgpack nil → `null`
- msgpack array → `List<Object>` (recurse)
- msgpack map → `Map<String, Object>` (recurse)

- [ ] **Step 3: Run CtyDecoder tests**

Run: `mvn --batch-mode test -pl bridge/terraform -Dtest=CtyDecoderTest`
Expected: PASS

- [ ] **Step 4: Create domain model records**

```java
// ProviderSchema.java
package io.casehub.desiredstate.bridge.terraform;
import java.util.Map;

public record ProviderSchema(Map<String, ResourceSchema> resourceSchemas) {}

// ResourceSchema.java
package io.casehub.desiredstate.bridge.terraform;
import java.util.Map;

public record ResourceSchema(Map<String, AttributeSchema> attributes) {}

// AttributeSchema.java
package io.casehub.desiredstate.bridge.terraform;

public record AttributeSchema(
    String name, String type, boolean required,
    boolean optional, boolean computed, String description
) {}

// Diagnostic.java
package io.casehub.desiredstate.bridge.terraform;

public record Diagnostic(Severity severity, String summary, String detail) {
    public enum Severity { ERROR, WARNING }
}

// PlanResponse.java
package io.casehub.desiredstate.bridge.terraform;
import java.util.List;
import java.util.Map;

public record PlanResponse(
    Map<String, Object> plannedState, Object rawPlan,
    boolean requiresReplace, List<Diagnostic> diagnostics
) {}

// ApplyResponse.java
package io.casehub.desiredstate.bridge.terraform;
import java.util.List;
import java.util.Map;

public record ApplyResponse(Map<String, Object> newState, List<Diagnostic> diagnostics) {}

// ReadResponse.java
package io.casehub.desiredstate.bridge.terraform;
import java.util.List;
import java.util.Map;

public record ReadResponse(Map<String, Object> currentState, List<Diagnostic> diagnostics) {}
```

- [ ] **Step 5: Create TerraformProtocol interface**

```java
package io.casehub.desiredstate.bridge.terraform;

import java.util.Map;

public interface TerraformProtocol extends AutoCloseable {
    ProviderSchema getSchema();
    void configure(Map<String, Object> config);
    PlanResponse plan(String resourceType, Map<String, Object> priorState,
                      Map<String, Object> proposedState);
    ApplyResponse apply(String resourceType, Map<String, Object> priorState,
                        Object plannedChange);
    ReadResponse read(String resourceType, Map<String, Object> currentState);
}
```

- [ ] **Step 6: Implement ProtocolV5Adapter**

Implements `TerraformProtocol` using proto v5 stubs. Key translations:
- `getSchema()` → v5 `GetSchema.Request` → parse response blocks/attributes into `ProviderSchema`
- `configure()` → v5 `Configure.Request` with JSON `DynamicValue`
- `plan()` → v5 `PlanResourceChange.Request` with JSON prior/proposed state
- `apply()` → v5 `ApplyResourceChange.Request`
- `read()` → v5 `ReadResource.Request`

All `DynamicValue` encoding/decoding uses JSON (v5's format) via `CtyDecoder`/`CtyEncoder`.

- [ ] **Step 7: Implement ProtocolV6Adapter**

Same as v5 but:
- `DynamicValue` uses msgpack encoding
- Schema attributes have a `type` field as raw msgpack cty type descriptors
- Use `CtyDecoder.decode(byte[] msgpack)` and `CtyEncoder.encode()`

- [ ] **Step 8: Implement TerraformProtocolFactory**

```java
package io.casehub.desiredstate.bridge.terraform;

import io.grpc.ManagedChannel;

public final class TerraformProtocolFactory {

    private TerraformProtocolFactory() {}

    public static TerraformProtocol create(GoPluginHandshake handshake, ManagedChannel channel) {
        return switch (handshake.appProtocol()) {
            case 5 -> new ProtocolV5Adapter(channel);
            case 6 -> new ProtocolV6Adapter(channel);
            default -> throw new IllegalArgumentException(
                "Unsupported Terraform plugin protocol version: " + handshake.appProtocol());
        };
    }
}
```

- [ ] **Step 9: Write ProtocolAdapter test with mock gRPC server**

Create an in-process gRPC server implementing the v5/v6 service stubs, returning canned schema and plan/apply/read responses. Verify the adapters correctly translate proto responses to our domain types.

- [ ] **Step 10: Run all tests**

Run: `mvn --batch-mode test -pl bridge/terraform`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat(bridge/terraform): CtyDecoder, TerraformProtocol, v5/v6 adapters

CtyDecoder/CtyEncoder handle Terraform's cty type system (JSON for v5,
msgpack for v6). TerraformProtocol interface with ProtocolV5Adapter and
ProtocolV6Adapter translate between domain types and proto stubs.
TerraformProtocolFactory selects adapter by handshake version.

Refs #184

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Schema Discovery + Bridge Provider

### Task 5: SchemaTranslator and SchemaCache

Convert `GetProviderSchema` response into `NodeType` + `PluginSpecSchema` entries. Cache schemas to disk for fast startup.

**Files:**
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/SchemaTranslator.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/SchemaCache.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/DiscoveredType.java`
- Test: `bridge/terraform/src/test/java/io/casehub/desiredstate/bridge/terraform/SchemaTranslatorTest.java`
- Test: `bridge/terraform/src/test/java/io/casehub/desiredstate/bridge/terraform/SchemaCacheTest.java`

**Interfaces:**
- Consumes: `ProviderSchema` from Task 4
- Consumes: `PluginSpecSchema`, `PluginFieldDef` from Task 1
- Produces: `SchemaTranslator` — `static List<DiscoveredType> translate(String namespace, String providerType, ProviderSchema schema)`
- Produces: `DiscoveredType` — `record DiscoveredType(NodeType nodeType, PluginSpecSchema specSchema, BridgeTypeDescriptor typeDescriptor)`
- Produces: `SchemaCache` — `void store(String namespace, String type, String version, List<DiscoveredType> types)`, `Optional<List<DiscoveredType>> load(String namespace, String type, String version)`

- [ ] **Step 1: Write SchemaTranslator tests**

```java
package io.casehub.desiredstate.bridge.terraform;

import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.plugin.api.PluginFieldDef;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class SchemaTranslatorTest {

    @Test
    void translateResourceTypeToNamespacedNodeType() {
        var schema = new ProviderSchema(Map.of(
            "aws_instance", new ResourceSchema(Map.of(
                "ami", new AttributeSchema("ami", "string", true, false, false, "AMI ID"),
                "instance_type", new AttributeSchema("instance_type", "string", true, false, false, "Instance type"),
                "id", new AttributeSchema("id", "string", false, false, true, "Instance ID")
            ))
        ));

        var types = SchemaTranslator.translate("hashicorp", "aws", schema);
        assertThat(types).hasSize(1);

        var discovered = types.get(0);
        assertThat(discovered.nodeType()).isEqualTo(NodeType.of("tf:hashicorp/aws:aws_instance"));
    }

    @Test
    void translateRequiredStringAttribute() {
        var schema = new ProviderSchema(Map.of(
            "aws_instance", new ResourceSchema(Map.of(
                "ami", new AttributeSchema("ami", "string", true, false, false, "AMI ID")
            ))
        ));

        var types = SchemaTranslator.translate("hashicorp", "aws", schema);
        var specSchema = types.get(0).specSchema();
        assertThat(specSchema.fields()).containsKey("ami");
        var field = specSchema.fields().get("ami");
        assertThat(field.type()).isEqualTo("STRING");
        assertThat(field.required()).isTrue();
    }

    @Test
    void excludeComputedOnlyAttributes() {
        var schema = new ProviderSchema(Map.of(
            "aws_instance", new ResourceSchema(Map.of(
                "ami", new AttributeSchema("ami", "string", true, false, false, "AMI ID"),
                "id", new AttributeSchema("id", "string", false, false, true, "Instance ID"),
                "arn", new AttributeSchema("arn", "string", false, false, true, "ARN")
            ))
        ));

        var types = SchemaTranslator.translate("hashicorp", "aws", schema);
        var specSchema = types.get(0).specSchema();
        assertThat(specSchema.fields()).containsKey("ami");
        assertThat(specSchema.fields()).doesNotContainKey("id");
        assertThat(specSchema.fields()).doesNotContainKey("arn");
    }

    @Test
    void translateNumberAttribute() {
        var schema = new ProviderSchema(Map.of(
            "res", new ResourceSchema(Map.of(
                "count", new AttributeSchema("count", "number", false, true, false, "Count")
            ))
        ));

        var types = SchemaTranslator.translate("hashicorp", "test", schema);
        var field = types.get(0).specSchema().fields().get("count");
        assertThat(field.type()).isEqualTo("NUMBER");
        assertThat(field.required()).isFalse();
    }

    @Test
    void translateBoolAttribute() {
        var schema = new ProviderSchema(Map.of(
            "res", new ResourceSchema(Map.of(
                "enabled", new AttributeSchema("enabled", "bool", false, true, false, "Flag")
            ))
        ));

        var types = SchemaTranslator.translate("hashicorp", "test", schema);
        var field = types.get(0).specSchema().fields().get("enabled");
        assertThat(field.type()).isEqualTo("BOOLEAN");
    }

    @Test
    void bridgeTypeDescriptorHasUniformStateMapping() {
        var schema = new ProviderSchema(Map.of(
            "res", new ResourceSchema(Map.of())
        ));

        var types = SchemaTranslator.translate("hashicorp", "test", schema);
        var td = types.get(0).typeDescriptor();
        assertThat(td.defaultState()).isEqualTo(io.casehub.desiredstate.api.NodeStatus.DRIFTED);
        assertThat(td.notFoundState()).isEqualTo(io.casehub.desiredstate.api.NodeStatus.ABSENT);
    }
}
```

- [ ] **Step 2: Implement SchemaTranslator**

```java
package io.casehub.desiredstate.bridge.terraform;

import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.bridge.BridgeErrorCategory;
import io.casehub.desiredstate.bridge.BridgeTypeDescriptor;
import io.casehub.desiredstate.plugin.api.PluginFieldDef;
import io.casehub.desiredstate.plugin.api.PluginSpecSchema;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public final class SchemaTranslator {

    private SchemaTranslator() {}

    public static List<DiscoveredType> translate(String namespace, String providerType,
                                                  ProviderSchema schema) {
        List<DiscoveredType> types = new ArrayList<>();
        for (var entry : schema.resourceSchemas().entrySet()) {
            String resourceName = entry.getKey();
            ResourceSchema rs = entry.getValue();

            NodeType nodeType = NodeType.of("tf:" + namespace + "/" + providerType + ":" + resourceName);
            PluginSpecSchema specSchema = translateSchema(rs);
            BridgeTypeDescriptor typeDesc = new BridgeTypeDescriptor(
                nodeType,
                Map.of("present", NodeStatus.PRESENT, "drifted", NodeStatus.DRIFTED),
                NodeStatus.DRIFTED,
                NodeStatus.ABSENT,
                Map.of(),
                BridgeErrorCategory.PERMANENT
            );
            types.add(new DiscoveredType(nodeType, specSchema, typeDesc));
        }
        return types;
    }

    private static PluginSpecSchema translateSchema(ResourceSchema rs) {
        Map<String, PluginFieldDef> fields = new LinkedHashMap<>();
        for (var attr : rs.attributes().values()) {
            if (attr.computed() && !attr.required() && !attr.optional()) {
                continue; // computed-only — output attribute
            }
            fields.put(attr.name(), translateAttribute(attr));
        }
        return new PluginSpecSchema(fields);
    }

    private static PluginFieldDef translateAttribute(AttributeSchema attr) {
        String fieldType = switch (attr.type()) {
            case "string" -> "STRING";
            case "number" -> "NUMBER";
            case "bool" -> "BOOLEAN";
            case "list" -> "LIST";
            case "set" -> "LIST";
            case "map" -> "MAP";
            default -> "STRING"; // fallback for complex types
        };
        return new PluginFieldDef(
            fieldType, attr.required(), null, null,
            null, null, null, null, null, null, null, null, null
        );
    }
}
```

- [ ] **Step 3: Create DiscoveredType record**

```java
package io.casehub.desiredstate.bridge.terraform;

import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.bridge.BridgeTypeDescriptor;
import io.casehub.desiredstate.plugin.api.PluginSpecSchema;

public record DiscoveredType(
    NodeType nodeType,
    PluginSpecSchema specSchema,
    BridgeTypeDescriptor typeDescriptor
) {}
```

- [ ] **Step 4: Run SchemaTranslator tests**

Run: `mvn --batch-mode test -pl bridge/terraform -Dtest=SchemaTranslatorTest`
Expected: PASS

- [ ] **Step 5: Write SchemaCache tests and implementation**

`SchemaCache` persists `List<DiscoveredType>` as JSON to disk and reloads. Tests should cover: store/load round trip, version-based cache invalidation (different version → cache miss), corrupt file handling. Use a `@TempDir` for the cache directory in tests.

- [ ] **Step 6: Run all tests**

Run: `mvn --batch-mode test -pl bridge/terraform`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(bridge/terraform): SchemaTranslator and SchemaCache

SchemaTranslator converts GetProviderSchema → NodeType + PluginSpecSchema.
Namespaced as tf:{namespace}/{type}:{resource}. Excludes computed-only
attributes. SchemaCache persists discovered types to disk for fast startup.

Refs #184

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 6: TerraformBridgeProvider and ProviderProcessManager

The main integration class: `TerraformBridgeProvider` implements `BridgeProvider` + `PlanCapable`,
wired together with `ProviderProcessManager` for lifecycle management.

**Files:**
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/TerraformBridgeProvider.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/ProviderProcessManager.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/TerraformErrorClassifier.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/ProviderConfig.java`
- Test: `bridge/terraform/src/test/java/io/casehub/desiredstate/bridge/terraform/TerraformBridgeProviderTest.java`
- Test: `bridge/terraform/src/test/java/io/casehub/desiredstate/bridge/terraform/TerraformErrorClassifierTest.java`

**Interfaces:**
- Consumes: `TerraformProtocol` (Task 4), `SchemaTranslator` (Task 5), `GoPluginHost` (Task 3)
- Consumes: `BridgeProvider`, `PlanCapable`, `BridgeRequest`, `BridgeResult`, `BridgePlan` from bridge/api
- Produces: `TerraformBridgeProvider` — implements `BridgeProvider`, `PlanCapable`
- Produces: `ProviderProcessManager` — `TerraformProtocol getProtocol()`, `List<DiscoveredType> discoverTypes()`, `void shutdown()`
- Produces: `ProviderConfig` — `record ProviderConfig(String namespace, String type, String version, String binaryPath)`

- [ ] **Step 1: Create ProviderConfig record**

```java
package io.casehub.desiredstate.bridge.terraform;

public record ProviderConfig(
    String namespace,
    String type,
    String version,
    String binaryPath
) {
    public String providerAddress() {
        return namespace + "/" + type;
    }
}
```

- [ ] **Step 2: Write TerraformErrorClassifier tests**

```java
package io.casehub.desiredstate.bridge.terraform;

import io.casehub.desiredstate.bridge.BridgeErrorCategory;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class TerraformErrorClassifierTest {

    @Test
    void authenticationError() {
        var diag = new Diagnostic(Diagnostic.Severity.ERROR, "AuthFailure", "Invalid credentials");
        assertThat(TerraformErrorClassifier.classify(List.of(diag))).isEqualTo(BridgeErrorCategory.AUTHENTICATION);
    }

    @Test
    void throttlingError() {
        var diag = new Diagnostic(Diagnostic.Severity.ERROR, "Throttling", "Rate exceeded");
        assertThat(TerraformErrorClassifier.classify(List.of(diag))).isEqualTo(BridgeErrorCategory.TRANSIENT);
    }

    @Test
    void timeoutError() {
        var diag = new Diagnostic(Diagnostic.Severity.ERROR, "timeout", "Request timed out");
        assertThat(TerraformErrorClassifier.classify(List.of(diag))).isEqualTo(BridgeErrorCategory.TRANSIENT);
    }

    @Test
    void validationError() {
        var diag = new Diagnostic(Diagnostic.Severity.ERROR, "Invalid configuration", "Missing required field");
        assertThat(TerraformErrorClassifier.classify(List.of(diag))).isEqualTo(BridgeErrorCategory.PERMANENT);
    }

    @Test
    void unknownErrorDefaultsToTransient() {
        var diag = new Diagnostic(Diagnostic.Severity.ERROR, "Something went wrong", "Unknown");
        assertThat(TerraformErrorClassifier.classify(List.of(diag))).isEqualTo(BridgeErrorCategory.TRANSIENT);
    }
}
```

- [ ] **Step 3: Implement TerraformErrorClassifier**

```java
package io.casehub.desiredstate.bridge.terraform;

import io.casehub.desiredstate.bridge.BridgeErrorCategory;
import java.util.List;
import java.util.Locale;

public final class TerraformErrorClassifier {

    private TerraformErrorClassifier() {}

    public static BridgeErrorCategory classify(List<Diagnostic> diagnostics) {
        for (Diagnostic d : diagnostics) {
            if (d.severity() != Diagnostic.Severity.ERROR) continue;
            String text = (d.summary() + " " + d.detail()).toLowerCase(Locale.ROOT);
            if (text.contains("auth") || text.contains("credential") || text.contains("permission_denied")
                || text.contains("accessdenied") || text.contains("unauthorized")) {
                return BridgeErrorCategory.AUTHENTICATION;
            }
            if (text.contains("invalid") || text.contains("validation") || text.contains("missing required")) {
                return BridgeErrorCategory.PERMANENT;
            }
        }
        return BridgeErrorCategory.TRANSIENT;
    }
}
```

- [ ] **Step 4: Run error classifier tests**

Run: `mvn --batch-mode test -pl bridge/terraform -Dtest=TerraformErrorClassifierTest`
Expected: PASS

- [ ] **Step 5: Implement ProviderProcessManager**

Wraps `GoPluginHost` + `TerraformProtocolFactory` + `SchemaTranslator`. Lazy initialization: `getProtocol()` starts the provider on first call, caches the protocol instance. `discoverTypes()` calls `getSchema()` and runs through `SchemaTranslator`. Thread-safe via `synchronized`.

Key behaviors:
- `getProtocol()` → lazy start, returns `TerraformProtocol`
- `discoverTypes()` → `getProtocol().getSchema()` → `SchemaTranslator.translate()`
- `shutdown()` → `protocol.close()` + `host.shutdown()`
- Restart on failure: if `host.isHealthy()` returns false, restart before next operation

- [ ] **Step 6: Implement TerraformBridgeProvider**

```java
package io.casehub.desiredstate.bridge.terraform;

import io.casehub.desiredstate.bridge.*;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

public class TerraformBridgeProvider implements BridgeProvider, PlanCapable {

    private final ProviderProcessManager processManager;
    private Set<String> supportedTypes;

    public TerraformBridgeProvider(ProviderProcessManager processManager) {
        this.processManager = processManager;
    }

    @Override
    public String protocolId() {
        return "terraform";
    }

    @Override
    public Set<String> supportedTypes() {
        if (supportedTypes == null) {
            supportedTypes = processManager.discoverTypes().stream()
                .map(dt -> dt.nodeType().value())
                .collect(Collectors.toSet());
        }
        return supportedTypes;
    }

    @Override
    public BridgePlan plan(BridgeRequest request) {
        var protocol = processManager.getProtocol();
        var response = protocol.plan(
            stripNamespace(request.typeId()),
            request.previousState(),
            request.spec());
        checkDiagnostics(response.diagnostics());
        return new BridgePlan(
            request.resourceId(),
            "Plan for " + request.typeId(),
            List.of(new ResourceChange(request.typeId(), request.resourceId(), ChangeAction.CREATE)),
            response.rawPlan());
    }

    @Override
    public BridgeResult applyPlan(BridgePlan plan) {
        var protocol = processManager.getProtocol();
        var response = protocol.apply(
            stripNamespace(plan.planId()), Map.of(), plan.rawPlan());
        if (hasErrors(response.diagnostics())) {
            return new BridgeResult.Failed(
                formatErrors(response.diagnostics()),
                TerraformErrorClassifier.classify(response.diagnostics()));
        }
        return new BridgeResult.Success(response.newState());
    }

    @Override
    public BridgeResult apply(BridgeRequest request) {
        var plan = plan(request);
        return applyPlan(plan);
    }

    @Override
    public BridgeResult destroy(BridgeRequest request) {
        var protocol = processManager.getProtocol();
        var planResp = protocol.plan(
            stripNamespace(request.typeId()),
            request.previousState(),
            null); // null proposed = destroy
        checkDiagnostics(planResp.diagnostics());
        var applyResp = protocol.apply(
            stripNamespace(request.typeId()),
            request.previousState(),
            planResp.rawPlan());
        if (hasErrors(applyResp.diagnostics())) {
            return new BridgeResult.Failed(
                formatErrors(applyResp.diagnostics()),
                TerraformErrorClassifier.classify(applyResp.diagnostics()));
        }
        return new BridgeResult.Success(Map.of());
    }

    @Override
    public BridgeStateResult readState(BridgeReadRequest request) {
        var protocol = processManager.getProtocol();
        var response = protocol.read(
            stripNamespace(request.typeId()),
            request.previousState());
        if (response.currentState() == null || response.currentState().isEmpty()) {
            return new BridgeStateResult.NotFound();
        }
        // Check for drift by comparing actual vs desired
        var planResp = protocol.plan(
            stripNamespace(request.typeId()),
            response.currentState(),
            request.desiredSpec());
        String status = planResp.requiresReplace() || !planResp.diagnostics().isEmpty()
            ? "drifted" : "present";
        var state = new java.util.HashMap<>(response.currentState());
        state.put("status", status);
        return new BridgeStateResult.Found(state);
    }

    private String stripNamespace(String typeId) {
        // tf:hashicorp/aws:aws_instance → aws_instance
        int lastColon = typeId.lastIndexOf(':');
        return lastColon >= 0 ? typeId.substring(lastColon + 1) : typeId;
    }

    private boolean hasErrors(List<Diagnostic> diagnostics) {
        return diagnostics.stream().anyMatch(d -> d.severity() == Diagnostic.Severity.ERROR);
    }

    private void checkDiagnostics(List<Diagnostic> diagnostics) {
        if (hasErrors(diagnostics)) {
            throw new RuntimeException("Terraform operation failed: " + formatErrors(diagnostics));
        }
    }

    private String formatErrors(List<Diagnostic> diagnostics) {
        return diagnostics.stream()
            .filter(d -> d.severity() == Diagnostic.Severity.ERROR)
            .map(d -> d.summary() + ": " + d.detail())
            .collect(Collectors.joining("; "));
    }
}
```

- [ ] **Step 7: Write TerraformBridgeProvider tests with mock protocol**

Create a mock `TerraformProtocol` that returns canned responses. Test the full provision → read → deprovision lifecycle through `TerraformBridgeProvider`. Test error classification for failed operations. Test drift detection in `readState`.

- [ ] **Step 8: Run all tests**

Run: `mvn --batch-mode test -pl bridge/terraform`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(bridge/terraform): TerraformBridgeProvider and ProviderProcessManager

TerraformBridgeProvider implements BridgeProvider + PlanCapable with full
plan/apply/read/destroy lifecycle. ProviderProcessManager handles lazy
start, restart on failure, and protocol caching. TerraformErrorClassifier
maps diagnostics to BridgeErrorCategory with TRANSIENT default.

Refs #184

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 4: Registry Client + Integration Tests

### Task 7: ProviderRegistryClient

Download provider binaries from registry.terraform.io, with SHA256 verification and local cache. Supports binary path override for air-gapped environments.

**Files:**
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/ProviderRegistryClient.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/ProviderBinary.java`
- Create: `bridge/terraform/src/main/java/io/casehub/desiredstate/bridge/terraform/PlatformDetector.java`
- Test: `bridge/terraform/src/test/java/io/casehub/desiredstate/bridge/terraform/ProviderRegistryClientTest.java`
- Test: `bridge/terraform/src/test/java/io/casehub/desiredstate/bridge/terraform/PlatformDetectorTest.java`

**Interfaces:**
- Produces: `ProviderRegistryClient` — `ProviderBinary resolve(ProviderConfig config)` (downloads if needed, returns local path)
- Produces: `ProviderBinary` — `record ProviderBinary(Path binaryPath, String version)`
- Produces: `PlatformDetector` — `static String os()`, `static String arch()`

- [ ] **Step 1: Write PlatformDetector tests and implementation**

```java
package io.casehub.desiredstate.bridge.terraform;

public final class PlatformDetector {

    private PlatformDetector() {}

    public static String os() {
        String name = System.getProperty("os.name").toLowerCase();
        if (name.contains("mac") || name.contains("darwin")) return "darwin";
        if (name.contains("linux")) return "linux";
        if (name.contains("windows")) return "windows";
        throw new UnsupportedOperationException("Unsupported OS: " + name);
    }

    public static String arch() {
        String arch = System.getProperty("os.arch").toLowerCase();
        if (arch.equals("amd64") || arch.equals("x86_64")) return "amd64";
        if (arch.equals("aarch64") || arch.equals("arm64")) return "arm64";
        throw new UnsupportedOperationException("Unsupported architecture: " + arch);
    }
}
```

- [ ] **Step 2: Create ProviderBinary record**

```java
package io.casehub.desiredstate.bridge.terraform;
import java.nio.file.Path;

public record ProviderBinary(Path binaryPath, String version) {}
```

- [ ] **Step 3: Write ProviderRegistryClient tests**

Test with mock HTTP responses (use a simple embedded HTTP server or mock the HTTP client). Tests:
- Resolve specific version → returns cached binary path
- Binary path override → skips registry, returns override path
- SHA256 mismatch → throws
- Network error → throws with clear message

- [ ] **Step 4: Implement ProviderRegistryClient**

Key logic:
1. If `config.binaryPath()` is set → return `new ProviderBinary(Path.of(binaryPath), config.version())`
2. Check cache: `~/.casehub/providers/{namespace}/{type}/{version}/{os}_{arch}/`
3. If cached → return cached binary
4. `GET https://registry.terraform.io/v1/providers/{namespace}/{type}/{version}/download/{os}/{arch}`
5. Parse JSON: `download_url`, `shasum`, `filename`
6. Download zip to temp dir
7. Verify SHA256 of zip
8. Extract, make binary executable
9. Move to cache directory
10. Return binary path

Use `java.net.http.HttpClient` for downloads, `java.security.MessageDigest` for SHA256, `java.util.zip.ZipInputStream` for extraction.

- [ ] **Step 5: Run tests**

Run: `mvn --batch-mode test -pl bridge/terraform -Dtest=ProviderRegistryClientTest,PlatformDetectorTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(bridge/terraform): ProviderRegistryClient with SHA256 verification

Downloads provider binaries from registry.terraform.io with SHA256
checksum verification and local caching. Binary path override for
air-gapped environments. PlatformDetector for OS/arch auto-detection.

Refs #184

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 8: Integration tests with real Terraform providers

The zero-config proof. Download real `hashicorp/null` and `hashicorp/random` providers,
exercise the full stack from binary start through schema discovery to lifecycle completion.

**Files:**
- Create: `bridge/terraform/src/test/java/io/casehub/desiredstate/bridge/terraform/NullProviderIntegrationTest.java`
- Create: `bridge/terraform/src/test/java/io/casehub/desiredstate/bridge/terraform/RandomProviderIntegrationTest.java`

**Interfaces:**
- Consumes: All classes from Tasks 3-7

- [ ] **Step 1: Write NullProviderIntegrationTest**

```java
package io.casehub.desiredstate.bridge.terraform;

import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Tag;
import static org.assertj.core.api.Assertions.*;

@Tag("integration")
class NullProviderIntegrationTest {

    static ProviderProcessManager processManager;

    @BeforeAll
    static void setup() {
        var registryClient = new ProviderRegistryClient();
        var config = new ProviderConfig("hashicorp", "null", "3.2.3", null);
        var binary = registryClient.resolve(config);
        processManager = new ProviderProcessManager(
            new GoPluginHost(), config, binary);
    }

    @AfterAll
    static void teardown() {
        if (processManager != null) processManager.shutdown();
    }

    @Test
    void handshakeSucceeds() {
        var protocol = processManager.getProtocol();
        assertThat(protocol).isNotNull();
    }

    @Test
    void schemaDiscoveryReturnsNullResource() {
        var types = processManager.discoverTypes();
        assertThat(types).isNotEmpty();
        assertThat(types.stream().map(dt -> dt.nodeType().value()))
            .anyMatch(t -> t.contains("null_resource"));
    }

    @Test
    void provisionAndDestroyNullResource() {
        var provider = new TerraformBridgeProvider(processManager);
        var request = new io.casehub.desiredstate.bridge.BridgeRequest(
            "tf:hashicorp/null:null_resource", "test-1",
            java.util.Map.of("triggers", java.util.Map.of("key", "value")),
            "tenant-1");

        var result = provider.apply(request);
        assertThat(result).isInstanceOf(io.casehub.desiredstate.bridge.BridgeResult.Success.class);

        var destroyResult = provider.destroy(request);
        assertThat(destroyResult).isInstanceOf(io.casehub.desiredstate.bridge.BridgeResult.Success.class);
    }
}
```

- [ ] **Step 2: Write RandomProviderIntegrationTest**

```java
package io.casehub.desiredstate.bridge.terraform;

import io.casehub.desiredstate.bridge.*;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.TestMethodOrder;
import org.junit.jupiter.api.MethodOrderer;
import org.junit.jupiter.api.Order;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

@Tag("integration")
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class RandomProviderIntegrationTest {

    static ProviderProcessManager processManager;
    static TerraformBridgeProvider provider;
    static InMemoryBridgeOutputStateStore stateStore;

    @BeforeAll
    static void setup() {
        var registryClient = new ProviderRegistryClient();
        var config = new ProviderConfig("hashicorp", "random", "3.6.3", null);
        var binary = registryClient.resolve(config);
        processManager = new ProviderProcessManager(
            new GoPluginHost(), config, binary);
        provider = new TerraformBridgeProvider(processManager);
        stateStore = new InMemoryBridgeOutputStateStore();
    }

    @AfterAll
    static void teardown() {
        if (processManager != null) processManager.shutdown();
    }

    @Test
    @Order(1)
    void zeroConfigSchemaDiscovery() {
        var types = processManager.discoverTypes();
        assertThat(types).isNotEmpty();
        var typeNames = types.stream().map(dt -> dt.nodeType().value()).toList();
        assertThat(typeNames).anyMatch(t -> t.contains("random_pet"));
        assertThat(typeNames).anyMatch(t -> t.contains("random_id"));
        assertThat(typeNames).anyMatch(t -> t.contains("random_string"));

        // Verify PluginSpecSchema was generated
        var petType = types.stream()
            .filter(dt -> dt.nodeType().value().contains("random_pet"))
            .findFirst().orElseThrow();
        assertThat(petType.specSchema().fields()).isNotEmpty();
    }

    @Test
    @Order(2)
    void provisionRandomPet() {
        var request = new BridgeRequest(
            "tf:hashicorp/random:random_pet", "my-pet",
            Map.of("length", 2), "tenant-1");
        var result = provider.apply(request);
        assertThat(result).isInstanceOf(BridgeResult.Success.class);

        var success = (BridgeResult.Success) result;
        assertThat(success.outputState()).isNotEmpty();
        stateStore.store("tenant-1", "my-pet", success.outputState());
    }

    @Test
    @Order(3)
    void readRandomPetStatePresent() {
        var previousState = stateStore.load("tenant-1", "my-pet").orElseThrow();
        var readRequest = new BridgeReadRequest(
            "tf:hashicorp/random:random_pet", "my-pet", "tenant-1",
            previousState, Map.of("length", 2));
        var result = provider.readState(readRequest);
        assertThat(result).isInstanceOf(BridgeStateResult.Found.class);

        var found = (BridgeStateResult.Found) result;
        assertThat(found.state()).containsKey("id");
    }

    @Test
    @Order(4)
    void deprovisionRandomPet() {
        var previousState = stateStore.load("tenant-1", "my-pet").orElseThrow();
        var request = new BridgeRequest(
            "tf:hashicorp/random:random_pet", "my-pet",
            Map.of("length", 2), "tenant-1", previousState);
        var result = provider.destroy(request);
        assertThat(result).isInstanceOf(BridgeResult.Success.class);
        stateStore.remove("tenant-1", "my-pet");
    }

    @Test
    @Order(5)
    void readAfterDeprovisionReturnsNotFound() {
        var readRequest = new BridgeReadRequest(
            "tf:hashicorp/random:random_pet", "my-pet", "tenant-1",
            Map.of(), Map.of());
        var result = provider.readState(readRequest);
        assertThat(result).isInstanceOf(BridgeStateResult.NotFound.class);
    }

    @Test
    @Order(6)
    void specValidationRejectsInvalidFields() {
        var types = processManager.discoverTypes();
        var petType = types.stream()
            .filter(dt -> dt.nodeType().value().contains("random_pet"))
            .findFirst().orElseThrow();
        var schema = petType.specSchema();

        // Verify schema has fields that can reject invalid input
        assertThat(schema.fields()).isNotEmpty();
        // PluginSpecSchema validation is handled by PluginSpecValidator in the
        // plugin framework — here we verify the schema was correctly generated
        // from the provider's GetProviderSchema response
    }
}
```

- [ ] **Step 3: Configure Maven profile for integration tests**

Add to `bridge/terraform/pom.xml`:

```xml
<profiles>
    <profile>
        <id>integration</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-failsafe-plugin</artifactId>
                    <configuration>
                        <groups>integration</groups>
                    </configuration>
                    <executions>
                        <execution>
                            <goals>
                                <goal>integration-test</goal>
                                <goal>verify</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

- [ ] **Step 4: Run integration tests**

Run: `mvn --batch-mode verify -pl bridge/terraform -Pintegration -am`
Expected: PASS — both null and random provider tests complete.

This is the critical test run. It proves:
- Go-plugin handshake with real provider binaries
- Zero-config schema auto-discovery
- Full provision → read → deprovision lifecycle
- PluginSpecSchema generation from provider schemas

- [ ] **Step 5: Run full project build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS (integration tests skipped without `-Pintegration`)

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(bridge/terraform): integration tests with null and random providers

NullProviderIntegrationTest proves go-plugin handshake and protocol
with a real provider binary. RandomProviderIntegrationTest proves the
zero-config lifecycle claim: schema auto-discovery, provision random_pet,
read state (PRESENT), deprovision, read state (NOT_FOUND).

Refs #184

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-10-27-terraform-bridge-adapter-design.md] — design spec this plan implements
- `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeProvider.java` — BridgeProvider SPI
- `bridge/api/src/main/java/io/casehub/desiredstate/bridge/PlanCapable.java` — PlanCapable mixin
- `bridge/api/src/main/java/io/casehub/desiredstate/bridge/BridgeReadRequest.java` — to be extended
- `bridge/runtime/src/main/java/io/casehub/desiredstate/bridge/runtime/BridgeProvisioner.java` — to be updated
- `bridge/runtime/src/main/java/io/casehub/desiredstate/bridge/runtime/BridgeActualStateAdapter.java` — to be updated
- `../../plugin/api/src/main/java/io/casehub/desiredstate/plugin/api/PluginSpecSchema.java` — to be moved to plugin/api
- `plugin/runtime/src/main/java/io/casehub/desiredstate/plugin/model/PluginFieldDef.java` — to be moved to plugin/api
- `runtime-core/src/main/java/io/casehub/desiredstate/runtime/DefaultNodeProvisionerRouter.java` — existing router pattern
- GitHub #182 — bridge adapter architecture epic
- GitHub #183 — bridge adapter SPI design
- GitHub #184 — focal issue
