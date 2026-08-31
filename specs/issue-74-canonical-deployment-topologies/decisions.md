# Decisions — Issue #82: Reconciliation + Live K8s Tests

## D1: Test level — synchronous planner+executor vs. async ReconciliationLoop

**Choice:** Synchronous TransitionPlanner + SimpleTransitionExecutor (no async ReconciliationLoop)
**Alternatives:**
- Full async ReconciliationLoop — tests the complete machinery but adds timing complexity, non-determinism, and re-tests behaviour already covered by desiredstate runtime tests
**Rationale:** Topology tests should prove topology correctness (ordering, drift, fault escalation), not re-test the reconciliation scheduling infrastructure. Synchronous tests are fast, deterministic, and focused.
**Trade-offs:** Won't test event-driven reconciliation triggering for topology exemplars — acceptable since that's generic machinery tested elsewhere.
**Sources:** `runtime/src/test/java/.../ReconciliationLoopTest.java` (existing async tests), `runtime/src/main/java/.../SimpleTransitionExecutor.java`
**Exploration:** quick
**Status:** captured

## D2: FailableNodeProvisioner — location and abstraction level

**Choice:** Implements `NodeProvisioner` (desiredstate SPI), lives in `topology-tests` module
**Alternatives:**
- Implement at InfraBackend/ResourceProvisioner level — too coupled to ops internals, not reusable
- Place in desiredstate/testing module — premature generalisation; MockNodeProvisioner already serves generic test needs
**Rationale:** NodeProvisioner is the right abstraction for transition executor tests. Records provision order, supports per-NodeId failure injection (fail N times then succeed). Test-specific — no need to generalise yet.
**Trade-offs:** If other modules need failable provisioning, they'll create their own or this gets promoted to testing module later.
**Sources:** `api/src/main/java/.../NodeProvisioner.java`, `testing/src/main/java/.../MockNodeProvisioner.java`
**Exploration:** quick
**Status:** captured

## D3: Maven profile gating mechanism

**Choice:** JUnit 5 `@Tag` annotations + surefire `<groups>`/`<excludedGroups>` with `combine.self="override"`
**Alternatives:**
- Package-based surefire `<includes>`/`<excludes>` — works but couples test location to build config
- Separate Maven modules — over-engineering for profile-gated tests in one module
**Rationale:** Tags are declarative, independent of package structure, and compose cleanly. `combine.self="override"` prevents the silent merge failure documented in GE-20260516-3a27dc.
**Trade-offs:** All surefire config fields must be repeated in the profile override block.
**Sources:** GE-20260516-3a27dc (combine.self="override" gotcha), GE-20260416-ca1c71 (*IT.java silent skip)
**Exploration:** quick
**Status:** captured

## D4: Live K8s test scope

**Choice:** Structured tests with `@EnabledIf` skip when K8s unavailable; real assertions for when cluster is present
**Alternatives:**
- Skip live tests entirely — loses the verification goal
- Require K8s for build — breaks the "mvn test passes without K8s" acceptance criterion
**Rationale:** Tests are written with proper structure and assertions. They're gated by both Maven profile AND runtime K8s availability check. This satisfies the acceptance criterion while allowing live testing when infrastructure exists.
**Trade-offs:** Live tests won't run in most CI environments until K8s is configured.
**Sources:** Issue #82 acceptance criteria: "Both test layers gated by Maven profiles (not in default build)"
**Exploration:** quick
**Status:** captured
