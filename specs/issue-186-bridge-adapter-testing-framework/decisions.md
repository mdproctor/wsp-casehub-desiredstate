# Decisions — Bridge Adapter Testing Framework (#186)

## D1: Primary consumer

**Choice:** Bridge implementors — people writing new BridgeProvider implementations
**Alternatives:**
- Bridge manifest authors — testing YAML manifest wiring; secondary concern, not primary
- Both equally — would dilute focus; manifest testing can use BridgeManifestParserTest patterns
**Rationale:** The framework exists to help provider authors verify their implementation against the BridgeProvider/PlanCapable/SuspendCapable/ImportCapable contracts without wiring up a full reconciliation loop.
**Trade-offs:** Manifest authors don't get a dedicated testing path; they use existing BridgeManifestParserTest and BridgeDiscoveryTest patterns.
**Sources:** BridgeProvider.java, BridgeProvisioner.java, PluginTestExtension.java (analogue)
**Exploration:** quick
**Status:** captured

## D2: Contract test style

**Choice:** TCK-style abstract test classes per capability tier
**Alternatives:**
- Smoke verification — helper methods for happy/error paths; less prescriptive, easier to miss cases
- Both TCK + helpers — adds complexity without clear benefit; TCK already covers the contracts
**Rationale:** Abstract test classes enforce contract coverage. Provider implementors extend the tier-appropriate class, override a factory method, and get full contract verification for free. Standard TCK pattern (JPA TCK, Jakarta EE TCK).
**Trade-offs:** More rigid — provider authors must follow the extension pattern. Can't cherry-pick individual contract tests without extending the class.
**Sources:** PluginTestExtension (analogue), JPA TCK pattern
**Exploration:** quick
**Status:** captured

## D3: Framework structure

**Choice:** Abstract TCK classes (per-tier) + BridgeTestExtension (runtime wiring) + extracted MockBridgeProvider (with fault injection)
**Alternatives:**
- Unified extension — auto-discovers contracts via interface detection; more magic, harder to debug
- Assertion library — pure helpers with no enforced structure; maximum flexibility but no coverage guarantee
**Rationale:** Clean separation: TCK tests the provider contract (BridgeProvider interface compliance), extension tests the runtime wiring (BridgeProvisioner/BridgeActualStateAdapter integration), mock provides programmable behavior for both. Matches existing testing/ module pattern.
**Trade-offs:** Three distinct components to maintain. More files than a unified approach.
**Sources:** testing/ (MockGoalCompiler etc.), plugin/testing/ (PluginTestExtension), bridge/runtime/src/test/ (MockBridgeProvider — extraction source)
**Exploration:** quick
**Depends on:** D1 (consumer focus determines what the framework optimizes for), D2 (TCK style determines the abstract class structure)
**Status:** captured

## D4: Fault injection approach

**Choice:** Configurable mock provider — MockBridgeProvider gains fault injection configs
**Alternatives:**
- Decorator wrapper — FaultInjectingBridgeProvider wraps any BridgeProvider; more flexible for integration tests
- Both — adds complexity without clear benefit at this stage
**Rationale:** Configurable mock is simpler, covers the common case (unit testing fault paths), matches how FaultInjectingStepRunner works in plugin testing. Decorator can be added later if integration test needs emerge.
**Trade-offs:** Can't inject faults into real providers (e.g., TerraformBridgeProvider) during integration tests.
**Sources:** FaultInjectingStepRunner (plugin/testing), MockBridgeProvider (bridge/runtime/src/test)
**Exploration:** quick
**Depends on:** D3 (mock is a component of the chosen structure)
**Status:** captured
