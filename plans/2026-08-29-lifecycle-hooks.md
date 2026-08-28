# YAML Lifecycle Hooks — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #121 — YAML lifecycle hooks — imperative steps within transitions
**Issue group:** #124, #121, #122

**Goal:** Per-node `provision:` and `deprovision:` hooks with `pre:` and `post:`
step lists (verify, notify, wait) that execute in `SimpleTransitionExecutor`
before/after `NodeProvisioner` calls.

**Architecture:** New YAML model types (`YamlHooks`, `YamlStep`) deserialize
hook declarations. `LifecycleStep` sealed interface and `HookDescriptor` record
carry resolved steps at runtime. `DesiredNode` gains an optional `HookDescriptor`.
`SimpleTransitionExecutor` executes pre-steps before provisioning (hard gate on
failure) and post-steps after (warning on failure). Three step executor
implementations: `VerifyStepExecutor`, `NotifyStepExecutor`, `WaitStepExecutor`.

**Tech Stack:** Java 21, Quarkus 3.x, Jackson YAML, Maven

**Design spec:** `specs/issue-124-cross-surface-rules/2026-08-29-lifecycle-hooks-design.md`

## Global Constraints

- Pre-step failure → node provisioning skipped (StepOutcome.Failed)
- Post-step failure → logged as warning, not blocking
- Step params support `${var.*}` interpolation (resolved at compile time)
- No shell/script execution — deferred (security model needed)
- No CaseTransitionExecutor integration — separate design
- Pre-release: DesiredNode record field changes are acceptable

---

## Batch 1: Model Types + DesiredNode Extension

Safe wrap point: YAML hooks deserialize correctly, `DesiredNode` carries
`HookDescriptor`, all existing tests pass with the new field.

### Task 1: LifecycleStep + HookDescriptor + DesiredNode extension

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/LifecycleStep.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/HookDescriptor.java`
- Modify: `api/src/main/java/io/casehub/desiredstate/api/DesiredNode.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlHooks.java`
- Create: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlStep.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/YamlNode.java`
- Create: `yaml/runtime/src/test/java/io/casehub/desiredstate/yaml/model/YamlHooksDeserializationTest.java`

**Interfaces:**
- Produces: `LifecycleStep` sealed interface: `Verify(url, timeoutSeconds)`,
  `Notify(channel, message)`, `Wait(seconds)`. `HookDescriptor(provisionPre,
  provisionPost, deprovisionPre, deprovisionPost)` with `List<LifecycleStep>`
  fields. `DesiredNode` gains `HookDescriptor hooks` (nullable, defaults null
  in compact constructor). `YamlHooks(pre, post)` and `YamlStep` for YAML
  deserialization.

- [ ] **Step 1: Create LifecycleStep sealed interface**

```java
package io.casehub.desiredstate.api;

public sealed interface LifecycleStep {
    record Verify(String url, int timeoutSeconds) implements LifecycleStep {
        public Verify { if (timeoutSeconds <= 0) timeoutSeconds = 30; }
    }
    record Notify(String channel, String message) implements LifecycleStep {}
    record Wait(int seconds) implements LifecycleStep {}
}
```

- [ ] **Step 2: Create HookDescriptor record**

```java
package io.casehub.desiredstate.api;

import java.util.List;

public record HookDescriptor(
        List<LifecycleStep> provisionPre,
        List<LifecycleStep> provisionPost,
        List<LifecycleStep> deprovisionPre,
        List<LifecycleStep> deprovisionPost) {

    public HookDescriptor {
        if (provisionPre == null) provisionPre = List.of();
        if (provisionPost == null) provisionPost = List.of();
        if (deprovisionPre == null) deprovisionPre = List.of();
        if (deprovisionPost == null) deprovisionPost = List.of();
    }

    public boolean isEmpty() {
        return provisionPre.isEmpty() && provisionPost.isEmpty()
                && deprovisionPre.isEmpty() && deprovisionPost.isEmpty();
    }
}
```

- [ ] **Step 3: Extend DesiredNode with hooks field**

Add `HookDescriptor hooks` as 4th field (nullable, no requireNonNull):

```java
public record DesiredNode(NodeId id, NodeSpec spec, HumanGating humanGating,
                          HookDescriptor hooks) {
    public DesiredNode {
        Objects.requireNonNull(id);
        Objects.requireNonNull(spec);
        Objects.requireNonNull(humanGating);
    }

    public DesiredNode(NodeId id, NodeSpec spec, HumanGating humanGating) {
        this(id, spec, humanGating, null);
    }
    // ... existing methods unchanged
}
```

The 3-arg constructor preserves backward compatibility for all existing callers.

- [ ] **Step 4: Create YAML model types**

`YamlHooks.java`:
```java
package io.casehub.desiredstate.yaml.model;
import java.util.List;
import java.util.Map;

public record YamlHooks(List<Map<String, Object>> pre, List<Map<String, Object>> post) {
    public YamlHooks {
        if (pre == null) pre = List.of();
        if (post == null) post = List.of();
    }
}
```

Add `YamlHooks provision` and `YamlHooks deprovision` fields to `YamlNode`
(after `forEach`).

- [ ] **Step 5: Write deserialization test**

Test that YAML with `provision:` and `deprovision:` blocks deserializes correctly.

- [ ] **Step 6: Run all api and yaml/runtime tests**

Run: `mvn --batch-mode test -pl api,yaml/runtime`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```
feat(#121): lifecycle hook model types

LifecycleStep sealed interface (Verify, Notify, Wait). HookDescriptor
record. DesiredNode gains optional hooks field with 3-arg backward-
compatible constructor. YAML model types for hook deserialization.

Refs #121
```

---

## Batch 2: Step Executors + SimpleTransitionExecutor Integration

Safe wrap point: hooks execute during provisioning/deprovisioning with
correct failure semantics.

### Task 2: LifecycleStepExecutor + three implementations

**Files:**
- Create: `api/src/main/java/io/casehub/desiredstate/api/LifecycleStepExecutor.java`
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/VerifyStepExecutor.java`
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/NotifyStepExecutor.java`
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/WaitStepExecutor.java`
- Create: `api/src/main/java/io/casehub/desiredstate/api/NotificationSink.java`
- Create: `runtime/src/main/java/io/casehub/desiredstate/runtime/LoggingNotificationSink.java`
- Create: `runtime/src/test/java/io/casehub/desiredstate/runtime/LifecycleStepExecutorTest.java`

**Interfaces:**
- Consumes: `LifecycleStep`, `StepOutcome`
- Produces: `LifecycleStepExecutor.execute(LifecycleStep, String tenancyId) → StepOutcome`.
  `VerifyStepExecutor` does HTTP GET, returns Succeeded/Failed.
  `NotifyStepExecutor` delegates to `NotificationSink` SPI.
  `WaitStepExecutor` sleeps for the specified duration.

- [ ] **Step 1: Create LifecycleStepExecutor interface**

```java
package io.casehub.desiredstate.api;

public interface LifecycleStepExecutor {
    StepOutcome execute(LifecycleStep step, String tenancyId);
}
```

- [ ] **Step 2: Create NotificationSink SPI + LoggingNotificationSink**

```java
// api module
public interface NotificationSink {
    void send(String channel, String message, String tenancyId);
}

// runtime module — @DefaultBean
@DefaultBean @ApplicationScoped
public class LoggingNotificationSink implements NotificationSink {
    private static final Logger LOG = Logger.getLogger(LoggingNotificationSink.class);
    public void send(String channel, String message, String tenancyId) {
        LOG.infof("[%s] %s → %s: %s", tenancyId, channel, message);
    }
}
```

- [ ] **Step 3: Implement three step executors**

`VerifyStepExecutor`: HTTP GET with timeout via `java.net.http.HttpClient`.
`NotifyStepExecutor`: delegates to `NotificationSink`, catches exceptions.
`WaitStepExecutor`: `Thread.sleep(seconds * 1000)`.

- [ ] **Step 4: Write executor unit tests**

Test each executor in isolation. Verify returns Succeeded/Failed correctly.

- [ ] **Step 5: Commit**

```
feat(#121): lifecycle step executors — verify, notify, wait

LifecycleStepExecutor SPI with three implementations. NotificationSink
SPI with LoggingNotificationSink default. VerifyStepExecutor uses
HttpClient for health checks.

Refs #121
```

### Task 3: SimpleTransitionExecutor hook integration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/desiredstate/runtime/SimpleTransitionExecutor.java`
- Modify: `runtime/src/test/java/io/casehub/desiredstate/runtime/SimpleTransitionExecutorTest.java`

**Interfaces:**
- Consumes: `HookDescriptor` from `DesiredNode.hooks()`, `LifecycleStepExecutor`
- Produces: Pre-steps gate provisioning. Post-steps are fire-and-forget.

- [ ] **Step 1: Inject LifecycleStepExecutor into SimpleTransitionExecutor**

Add constructor parameter. CDI provides the bean.

- [ ] **Step 2: Add hook execution in executeProvision**

Before the human/approval/provisioner chain:
```java
if (node.hooks() != null && !node.hooks().provisionPre().isEmpty()) {
    for (LifecycleStep step : node.hooks().provisionPre()) {
        StepOutcome preResult = stepExecutor.execute(step, tenancyId);
        if (preResult instanceof StepOutcome.Failed f) {
            return new StepOutcome.Failed("pre-provision hook failed: " + f.reason());
        }
    }
}
```

After successful provisioning:
```java
if (node.hooks() != null && !node.hooks().provisionPost().isEmpty()) {
    for (LifecycleStep step : node.hooks().provisionPost()) {
        StepOutcome postResult = stepExecutor.execute(step, tenancyId);
        if (postResult instanceof StepOutcome.Failed f) {
            LOG.warnf("post-provision hook failed for %s: %s", node.id(), f.reason());
        }
    }
}
```

- [ ] **Step 3: Add hook execution in executeDeprovision**

Same pattern — pre gates, post warns.

- [ ] **Step 4: Write integration tests**

Test: node with pre-hook that fails → provisioning skipped.
Test: node with pre-hook that succeeds → provisioning proceeds.
Test: node with post-hook that fails → provisioning still succeeded.
Test: node without hooks → unchanged behavior.

- [ ] **Step 5: Run all runtime tests**

Run: `mvn --batch-mode test -pl runtime`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```
feat(#121): SimpleTransitionExecutor hook integration

Pre-provision/deprovision hooks gate execution — failure skips the node.
Post-provision/deprovision hooks are fire-and-forget — failure logged
as warning. LifecycleStepExecutor injected via CDI.

Refs #121
```

---

## Batch 3: YAML GoalCompiler Wiring + Integration Test

Safe wrap point: YAML operators can declare hooks on nodes and they
execute during reconciliation.

### Task 4: GoalCompiler hook resolution + build-time validation + integration test

**Files:**
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/ForEachExpander.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`
- Modify: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java`
- Modify: `examples/webapp-yaml/src/main/resources/META-INF/desiredstate/tutorial-2-smart-defaults.yaml`
- Modify: `examples/webapp-yaml/src/test/java/io/casehub/desiredstate/example/webapp/yaml/Tutorial2SmartDefaultsTest.java`

**Interfaces:**
- Consumes: `YamlHooks` from `YamlNode`, `VariableResolver` for step param resolution
- Produces: `HookDescriptor` set on `DesiredNode` during graph construction.
  Build-time validation for step types and parameters.

- [ ] **Step 1: Add hook resolution to ForEachExpander**

When building `DesiredNode`, if `yamlNode.provision()` or `yamlNode.deprovision()`
is non-null, resolve `${var.*}` in step params and build `HookDescriptor`:

```java
HookDescriptor hooks = resolveHooks(yamlNode, nodeResolver);
allNodes.add(new DesiredNode(NodeId.of(nodeId), spec, yamlNode.humanGating(), hooks));
```

`resolveHooks` converts `YamlHooks` → `HookDescriptor` by mapping each
`Map<String, Object>` step entry to a `LifecycleStep` variant.

- [ ] **Step 2: Add build-time validation**

In `YamlDesiredStateProcessor`, validate:
- Step type is one of: verify, notify, wait
- verify.url is non-empty, timeout is positive
- notify.channel is one of: email, sms, webhook
- wait.seconds is positive

- [ ] **Step 3: Add hooks to tutorial-2 YAML**

Add provision/deprovision hooks to the payment node:

```yaml
  payment:
    type: payment
    dependsOn: [shopping-cart]
    provision:
      pre:
        - verify: { url: "http://localhost:5432/health", timeout: 10 }
      post:
        - notify: { channel: email, message: "Payment processor deployed" }
    deprovision:
      pre:
        - wait: { seconds: 5 }
      post:
        - notify: { channel: email, message: "Payment processor removed" }
    spec:
      provider: ${var.payment_provider}
      currency: ${var.currency}
      maxRetries: 3
```

- [ ] **Step 4: Write integration test**

```java
@Test
void hooks_paymentNodeHasProvisionHooks() {
    DesiredStateGraph graph = compile();
    DesiredNode payment = graph.nodes().get(NodeId.of("payment"));
    assertThat(payment.hooks()).isNotNull();
    assertThat(payment.hooks().provisionPre()).hasSize(1);
    assertThat(payment.hooks().provisionPre().get(0))
            .isInstanceOf(LifecycleStep.Verify.class);
    assertThat(payment.hooks().provisionPost()).hasSize(1);
    assertThat(payment.hooks().deprovisionPre()).hasSize(1);
}

@Test
void hooks_nodesWithoutHooks_hooksIsNull() {
    DesiredStateGraph graph = compile();
    DesiredNode catalog = graph.nodes().get(NodeId.of("product-catalog"));
    assertThat(catalog.hooks()).isNull();
}
```

- [ ] **Step 5: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```
feat(#121): YAML lifecycle hooks — verify, notify, wait

Per-node provision: and deprovision: hook blocks with pre: and post:
step lists. Three step types: verify (HTTP health check), notify
(channel + message), wait (timed delay). Build-time validation.
Tutorial 2 demonstrates hooks on the payment node.

Closes #121
```

---

## Summary

| Batch | Tasks | What's working after |
|-------|-------|---------------------|
| 1: Model Types | 1 | Hook types deserialize, DesiredNode carries HookDescriptor |
| 2: Step Executors | 2-3 | Hooks execute in SimpleTransitionExecutor with correct failure semantics |
| 3: YAML Wiring | 4 | Operators declare hooks in YAML, tutorial demonstrates end-to-end |

**Total:** 3 batches, 4 tasks

## References

- `specs/issue-124-cross-surface-rules/2026-08-29-lifecycle-hooks-design.md` — design spec
- `api/.../DesiredNode.java` — per-node record (hooks field addition)
- `api/.../StepOutcome.java` — Succeeded/Failed/Skipped/Rejected
- `runtime/.../SimpleTransitionExecutor.java:72-115` — provision execution
- `runtime/.../SimpleTransitionExecutor.java:117-158` — deprovision execution
- `yaml/runtime/.../model/YamlNode.java` — YAML node model
- `yaml/runtime/.../ForEachExpander.java` — node construction
- #121 — YAML lifecycle hooks
