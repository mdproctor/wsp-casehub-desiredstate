# YAML Lifecycle Hooks — Design Spec

**Issue:** #121 — YAML lifecycle hooks — imperative steps within transitions
**Date:** 2026-08-29
**Status:** Draft

## 1. Summary

Per-node `provision:` and `deprovision:` hook blocks with `pre:` and `post:` step
lists. Three step types: `verify` (URL health check), `notify` (channel + message),
`wait` (timed delay). Steps execute in `SimpleTransitionExecutor` before/after
`NodeProvisioner` calls. YAML-only feature — no annotation equivalent.

## 2. Background

The YAML surface declares nodes, dependencies, rules, invariants, fault policies,
forEach, modules, and lifecycle phases. The missing piece: imperative actions during
node provisioning. Operators need to verify dependencies are healthy before starting
a node, drain connections before removing one, and notify teams after deployment.

The design spec §11 deferred this: "Lifecycle hooks require imperative step execution
within phase transitions, which is structurally different from the declarative features."

## 3. YAML Syntax

```yaml
nodes:
  api-server:
    type: app
    dependsOn: [database]
    provision:
      pre:
        - verify: { url: "http://${var.db_host}:5432/health", timeout: 30 }
        - wait: { seconds: 5 }
      post:
        - notify: { channel: email, message: "api-server deployed" }
    deprovision:
      pre:
        - notify: { channel: ops, message: "draining api-server" }
        - wait: { seconds: 30 }
      post:
        - notify: { channel: ops, message: "api-server removed" }
    spec:
      image: "api:latest"
```

### 3.1 Step Types

| Step | Parameters | Purpose |
|------|-----------|---------|
| `verify` | `url` (required), `timeout` (seconds, default: 30) | HTTP GET health check. Success: 2xx response. Failure: non-2xx or timeout. |
| `notify` | `channel` (required: email/sms/webhook), `message` (required) | Send notification. Fire-and-forget — failure logged, not blocking. |
| `wait` | `seconds` (required) | Timed delay. Used for drain periods or startup grace. |

All step parameters support `${var.*}` interpolation (resolved at compile time).

### 3.2 Failure Semantics

| Phase | Step | On failure |
|-------|------|-----------|
| `provision.pre` | Any step fails | Node provisioning **skipped** — `StepOutcome.Failed` with reason |
| `provision.post` | Any step fails | **Logged as warning** — provisioning already succeeded |
| `deprovision.pre` | Any step fails | Node deprovisioning **skipped** — `StepOutcome.Failed` with reason |
| `deprovision.post` | Any step fails | **Logged as warning** — deprovisioning already succeeded |

Pre-step failure is a hard gate: if the database health check fails, the API server
is not provisioned. Post-step failure is informational: the node is already
provisioned/deprovisioned, the notification failure doesn't change that.

## 4. Architecture

### 4.1 Model Types

```
YamlHooks(pre: List<YamlStep>, post: List<YamlStep>)
YamlStep — sealed: VerifyStep(url, timeout) | NotifyStep(channel, message) | WaitStep(seconds)
```

`YamlNode` gains two optional fields: `provision: YamlHooks`, `deprovision: YamlHooks`.

### 4.2 Runtime SPI

```java
public interface LifecycleStepExecutor {
    StepOutcome execute(LifecycleStep step, String tenancyId);
}
```

Three implementations registered via CDI:
- `VerifyStepExecutor` — HTTP client, checks 2xx
- `NotifyStepExecutor` — delegates to a `NotificationSink` SPI (email/sms/webhook)
- `WaitStepExecutor` — `Thread.sleep` (simple, sufficient for reconciliation loop context)

`NotificationSink` is an SPI — the runtime provides a `LoggingNotificationSink` default
that logs the message. Real implementations (email, Slack, webhook) are domain-provided.

### 4.3 HookDescriptor

Compile-time artifact carried alongside the `DesiredNode`:

```java
public record HookDescriptor(
    List<LifecycleStep> provisionPre,
    List<LifecycleStep> provisionPost,
    List<LifecycleStep> deprovisionPre,
    List<LifecycleStep> deprovisionPost) {}
```

`LifecycleStep` is a sealed interface mirroring `YamlStep` but with resolved values
(no `${var.*}` placeholders — all resolved at compile time).

### 4.4 SimpleTransitionExecutor Integration

The executor checks for `HookDescriptor` on each `DesiredNode`. If present:

**Provision path:**
1. Execute `provisionPre` steps sequentially. If any fails → `StepOutcome.Failed`.
2. Call `NodeProvisioner.provision()` (existing path).
3. Execute `provisionPost` steps sequentially. Failures logged, not blocking.

**Deprovision path:**
1. Execute `deprovisionPre` steps sequentially. If any fails → `StepOutcome.Failed`.
2. Call `NodeProvisioner.deprovision()` (existing path).
3. Execute `deprovisionPost` steps sequentially. Failures logged, not blocking.

### 4.5 Carrying HookDescriptor

`HookDescriptor` needs to travel from the GoalCompiler to the TransitionExecutor.
Options:

**Option A (Recommended):** Attach to `DesiredNode` via a new optional field.
`DesiredNode` gains `HookDescriptor hooks()` (default: null). The GoalCompiler
sets it during node construction. The executor reads it.

**Option B:** Side-channel map `Map<NodeId, HookDescriptor>` passed through
`ProvisionContext`. More isolated but requires plumbing through multiple layers.

Option A is simpler — `DesiredNode` already carries `humanGating` which is a
similar per-node behavioral annotation.

### 4.6 Build-Time Validation

- `verify.url` must be a non-empty string (may contain `${var.*}`)
- `verify.timeout` must be positive (default: 30)
- `notify.channel` must be one of: `email`, `sms`, `webhook`
- `notify.message` must be non-empty
- `wait.seconds` must be positive
- Unknown step types → build-time error

## 5. What This Does NOT Cover

- **Shell/script execution** — deferred (security model needed)
- **Bean-reference steps** — Java escape hatch (future)
- **Phase-to-phase hooks** — different concern (transition callbacks between phases)
- **CaseTransitionExecutor integration** — CTE has its own Worker(Workflow) model;
  hooks would become additional workflow steps, which is a separate design
- **Retry/backoff on verify** — v1 is single-attempt. Retry policy can be added later.

## 6. Testing

- Unit tests: YAML model deserialization, step type validation
- Unit tests: `VerifyStepExecutor`, `NotifyStepExecutor`, `WaitStepExecutor`
- Unit tests: `SimpleTransitionExecutor` hook integration (mock provisioner + mock step executor)
- Integration test: webapp-yaml tutorial with hooks on the payment node
  (verify database health before payment, notify after confirmation)

## References

- `runtime/.../LifecycleManager.java` — phase transition via CAS
- `runtime/.../SimpleTransitionExecutor.java` — provision/deprovision execution
- `api/.../DesiredNode.java` — per-node behavioral fields
- `api/.../StepOutcome.java` — Succeeded/Failed/Skipped sealed interface
- `yaml/runtime/.../model/YamlNode.java` — YAML node model
- `yaml/runtime/.../YamlGraphRecorder.java` — GoalCompiler node construction
- Design spec §6.5 — lifecycle phases
- Design spec §11 — deferral note
- #121 — YAML lifecycle hooks
