# Design Decisions — #112 TypedFaultPolicy tier() redundancy

## D1: Exclusive TypedFaultPolicy — no overloaded tier()

**Choice:** `tier()` exclusively accepts `TypedFaultPolicy`. Tests that use raw `FaultPolicy` lambdas wrap via `TypedFaultPolicy.of(NodeType, FaultPolicy)`. The old 3-arg `tier(int, FaultPolicy, NodeType)` is removed entirely.
**Alternatives:**
- Overloaded with deprecated 3-arg — gradual migration, but old path stays available and callers can still mismatch NodeType
- Overloaded without deprecation — permanent escape hatch, defeats the purpose of the type-safe change
- Tier accepts ReviewSpecFactory directly (R1-05) — eliminates mismatch structurally but constrains tiers to node-producing actions only, reshapes the Tier/FaultPolicy relationship too deeply for this issue
**Rationale:** Pre-release platform. No production caller uses raw lambdas for `tier()` — all go through `addReviewNode`. Raw-lambda pattern is test-only (7 callers) for threshold validation mechanics. `TypedFaultPolicy.of()` is a one-line wrapper. The improvement is **parameter bundling** — concentrating NodeType and FaultPolicy at one construction site rather than two separate arguments. The mismatch is concentrated, not compile-time-eliminated. But locality makes errors easier to spot in review and harder to introduce during refactoring.
**Trade-offs:** 7 test callers need `TypedFaultPolicy.of()` wrapping. Ops callers (casehub-ops#70, when they migrate) will also need to drop the trailing `NodeType` from `tier()`. The mismatch is still possible via `TypedFaultPolicy.of(WRONG_TYPE, ...)` — this is parameter bundling, not type-level prevention.
**Sources:** ThresholdFaultPolicy.java (api), ThresholdFaultPolicyTest.java (17 callers total), DesiredStateGraphRecorder.java (annotations/runtime), PipelineTest.java
**Exploration:** quick
**Review:** R1-01 (naming), R1-04 (compile-time claim), R1-05 (unconsidered alternative)
**Status:** revised

## D2: TypedFaultPolicy in api module — generic naming

**Choice:** `TypedFaultPolicy` is a top-level interface in `io.casehub.desiredstate.api`, in its own file `TypedFaultPolicy.java`. Extends `FaultPolicy`, adds `outputNodeType()` and a static factory `of(NodeType, FaultPolicy)`.
**Alternatives:**
- `ReviewNodePolicy` with `reviewNodeType()` — hardcodes review-node use case into structural abstraction. Tier NodeType is used for auto-ignore and predecessor gating, neither specific to "review"
- `TierPolicy` — too tightly coupled to ThresholdFaultPolicy
- `EscalationPolicy` — broader than "review" but still use-case-specific
**Rationale:** The structural contract is "a FaultPolicy that declares its associated output NodeType." `TypedFaultPolicy` names the contract, not the use case. Matches CDI `Typed` convention. Same package as FaultPolicy, visible to all api consumers.
**Trade-offs:** One more file in the api package. Minimal — the interface is small (~10 lines).
**Sources:** FaultPolicy.java, ThresholdFaultPolicy.java, decision review R1-01
**Exploration:** quick
**Review:** R1-01 (naming)
**Depends on:** D1 (exclusive tier() signature requires the type to exist)
**Status:** revised

## D3: ReviewSpecFactory gains default nodeType()

**Choice:** `ReviewSpecFactory` gains a `default NodeType nodeType()` method. The default implementation probes by calling `create(FaultEvent.probe(), null).nodeType()`. Domain factories that are concrete classes can override with a constant — eliminating the probe entirely. Lambda callers fall back to the default probe.
**Alternatives:**
- No default method, probe stays in addReviewNode — works but domain factories can't opt out of probing
- Require nodeType() as abstract (not default) — breaks all existing lambda callers since @FunctionalInterface needs exactly one abstract method
**Rationale:** The default preserves backward compatibility with lambdas while giving concrete factories a clean path to declare their type statically. `addReviewNode` calls `factory.nodeType()` once at construction time to build the `TypedFaultPolicy`. The probe is encapsulated in one place.
**Trade-offs:** Lambda callers still use the probe (null graph — if review method touches graph, NPE with fallback to method name). This is acknowledged as a trade-off, not solved. Concrete factories in production code should override `nodeType()`.
**Sources:** FaultPolicy.java:8-19, ReviewSpecFactory.java, DesiredStateGraphRecorder.java:230-241
**Exploration:** quick
**Review:** R1-02, R1-03, R1-05
**Depends on:** D2 (addReviewNode return type)
**Status:** captured

## D4: Annotation recorder — probeReviewNodeType() eliminated

**Choice:** `DesiredStateGraphRecorder.probeReviewNodeType()` is removed. The recorder calls `FaultPolicy.addReviewNode(factory)` which returns `TypedFaultPolicy` — NodeType is inside the return value. The recorder passes it directly to `tier(threshold, typedPolicy)`.
**Alternatives:**
- Path B from R1-02: add `nodeType` to `@Tier` annotation for build-time validation — genuine improvement but expands scope to annotation model + deployment processor changes. Filed as follow-up consideration.
**Rationale:** The probe moves inside `ReviewSpecFactory.nodeType()` default — encapsulated rather than scattered in the recorder. The recorder code simplifies: no separate NodeType discovery step. For annotation-driven factories (lambdas wrapping reflective method calls), the probe still fires via the default, but it's the factory's responsibility, not the recorder's.
**Trade-offs:** The probe is still fragile for lambda callers (null graph). A future `@Tier(nodeType="...")` attribute with build-time validation would eliminate it entirely for the annotation path.
**Sources:** DesiredStateGraphRecorder.java:134-148 (tier construction), DesiredStateGraphRecorder.java:230-241 (probeReviewNodeType)
**Exploration:** quick
**Review:** R1-02
**Depends on:** D3 (ReviewSpecFactory.nodeType())
**Status:** captured
