# Decisions — YAML Lifecycle Hooks (#121)

## D1: Pre/post provisioner steps (not inline sequences)

**Choice:** Hooks run before or after the NodeProvisioner — they wrap it without changing the provisioning contract.
**Alternatives:**
- Inline step sequences — provisioner becomes one step among others. More powerful but changes the provisioning SPI contract for all surfaces.
- Phase transition hooks only — not per-node. Too narrow for real use cases (health checks, drain, notify).
**Rationale:** Pre/post wrapping preserves the NodeProvisioner SPI. Existing provisioners work unchanged. The hook model is additive.
**Trade-offs:** Cannot interleave custom steps with provisioner internals. Complex multi-step provisioning still requires a custom NodeProvisioner.
**Sources:** LifecycleManager.java, SimpleTransitionExecutor, NodeProvisioner SPI
**Exploration:** quick
**Status:** captured

## D2: Three initial step types — verify, notify, wait

**Choice:** `verify` (URL health check with timeout), `notify` (channel + message), `wait` (timed delay). Shell/script execution deferred.
**Alternatives:**
- verify + notify + wait + shell — adds arbitrary command execution with security implications
- Bean reference only — maximum flexibility but operators must write Java for every step
**Rationale:** These three cover the most common deployment patterns without introducing security concerns. Shell execution requires sandbox/security decisions that shouldn't gate the initial feature. Bean reference is the Java escape hatch (can be added later).
**Trade-offs:** No custom logic without Java. Data migration scripts can't run as hooks in v1.
**Sources:** Design spec §11 deferral note
**Exploration:** quick
**Status:** captured

## D3: Both provision and deprovision hooks

**Choice:** Symmetric — both `provision:` and `deprovision:` support `pre:` and `post:` step lists.
**Alternatives:**
- Provision only — simpler, but operators need drain/teardown hooks in practice
**Rationale:** Deprovision hooks are essential for graceful shutdown (connection draining, notification). The model is identical — no additional complexity.
**Trade-offs:** None significant.
**Sources:** DesiredNode, SimpleTransitionExecutor deprovision path
**Exploration:** quick
**Status:** captured
