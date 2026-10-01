# Decisions — #160 Drift Policy Layer

## D1: DriftPolicy interception point in reconciliation cycle

**Choice:** Intercept inside `detectDrift()` — check DriftPolicy before creating FaultEvents
**Alternatives:**
- New `filterExemptDrift()` step between detectDrift and plan — cleaner SRP but requires refactoring detectDrift to split detection from fault event creation, unnecessary churn
- Intercept in TransitionPlanner — wrong layer, fault events already fired, breaks pre-planning principle
**Rationale:** `detectDrift()` is where both drift detection AND fault event creation happen. Inserting the policy check before FaultEvent creation means exempt nodes never enter the fault pipeline at all. Minimal change surface — one method modified. The planner already handles only what detectDrift passes through, so no planner changes needed.
**Trade-offs:** detectDrift() grows slightly in responsibility (detection + policy evaluation), but the alternative (splitting) adds unnecessary indirection for a check that is inherently part of "what do we do with this drift?"
**Sources:** ReconciliationLoop.java:706-741 (detectDrift method), FaultPolicyEngine.java (existing fault evaluation pattern)
**Exploration:** quick
**Status:** captured

## D2: ExemptionStore SPI design

**Choice:** Thin store — key=(tenancyId, nodeId), value=Exemption record. grant/revoke/get/evict. Loop checks expiry each cycle.
**Alternatives:**
- Rich store with internal expiration timers — unnecessary machinery since the loop already re-evaluates each cycle
- Exemptions as graph metadata — conflates desired state with operational policy, semantically wrong
**Rationale:** Follows the established FaultCountStore pattern. No namespace needed — unlike fault counts, exemptions don't need multi-policy scoping because there's one active exemption per node. The reconciliation loop checks exemption expiry naturally during each cycle — expired exemptions simply result in RECONCILE on the next pass.
**Trade-offs:** Predicate<ActualState> for event-based revert is not serializable — JPA persistence must store revert mode metadata and reconstruct predicates. This is acceptable since predicates are domain-specific and domains own their DriftPolicy implementations.
**Sources:** FaultCountStore.java (SPI pattern), InMemoryFaultCountStore.java (in-memory impl pattern), ReconciliationStateStore.java (store SPI pattern)
**Exploration:** quick
**Status:** captured

## D3: DriftPolicy SPI shape — chain via DriftPolicyEngine

**Choice:** Multiple DriftPolicy beans aggregated by DriftPolicyEngine, first-EXEMPT-wins semantics
**Alternatives:**
- Single policy, no chain — simpler but breaks multi-domain composition (CrossDomainCompositionEngine already supports multiple domains contributing SPIs)
- Priority-ordered chain with RECONCILE overriding EXEMPT — overly complex, the default IS reconcile, policies can only escalate to exempt
**Rationale:** Mirrors the FaultPolicy/FaultPolicyEngine pattern. Multiple domains can contribute policies. First-EXEMPT-wins is intuitive: "anyone can grant permission." RECONCILE is the default state — policies only grant exemptions, they never force reconciliation (that's already the baseline).
**Trade-offs:** Multiple policies evaluating per node per cycle has a performance cost, but this is bounded by the number of DriftPolicy beans (typically 1-3 per deployment) and is negligible compared to actual state reads and provisioning.
**Sources:** FaultPolicyEngine.java (chain evaluation pattern), CrossDomainCompositionEngine (multi-domain SPI contribution)
**Exploration:** quick
**Status:** captured

## D4: Observability — new event type for permitted drift

**Choice:** Emit a distinct `NodeDriftExemptedData` CloudEvent for drift-exempt nodes
**Alternatives:**
- Silent — exempt nodes produce no events, but operators can't observe exemptions without querying the store
- Reuse NodeDriftedData with an `exempt` flag — conflates two different semantic signals in one event type
**Rationale:** Dashboards and operators need visibility into permitted drift. A separate event type keeps the semantic contract clean — NodeDriftedData means "unexpected drift requiring action," NodeDriftExemptedData means "drift detected but permitted by policy."
**Trade-offs:** Additional event type adds to the CloudEvent surface. Acceptable — the emitter pattern is established and adding one more is trivial.
**Sources:** ReconciliationEventEmitter.java (event emission pattern), NodeDriftedData.java (existing drift event), DesiredStateEventTypes.java (event type constants)
**Exploration:** quick
**Status:** captured

## D5: Exemption granting — both SPI and imperative

**Choice:** DriftPolicy.evaluate() for declarative per-cycle evaluation, plus ExemptionStore.grant()/revoke() for imperative use by external event handlers
**Alternatives:**
- SPI only — external code would need to influence policy inputs indirectly, less flexible for reactive domains like IoT
- Imperative only — no per-cycle evaluation hook, loses the ability for policies to react to changing conditions each cycle
**Rationale:** IoT triggers (motion sensors, manual overrides) need imperative grant when the trigger event fires. Policies need per-cycle evaluation for rule-based exemptions (e.g., "exempt this node type during business hours"). Both paths write to the same ExemptionStore — the loop just checks the store, agnostic to who wrote the exemption.
**Trade-offs:** Two paths to the same store could lead to conflicts (policy grants, external revokes same cycle). Resolved by: imperative revoke always wins — if external code revokes, the policy must re-evaluate and re-grant on the next cycle if the exemption should persist.
**Sources:** casehubio/platform#486 § Drift policy model, casehubio/iot#120 (IoT trigger use case)
**Exploration:** quick
**Status:** captured

## D6: Duration-based retriggering via DriftPolicy re-evaluation

**Choice:** DriftPolicy returns fresh EXEMPT(duration) each cycle if drift is still permitted. Store overwrites old exemption.
**Alternatives:**
- ExemptionStore.extend() resets expiry on repeated grant calls — splits retriggering logic between two components
**Rationale:** Natural fit — the policy decides whether drift is still permitted, the store just records the latest decision. Retriggering is simply the policy saying "still exempt" each cycle with a fresh duration. No special extend() semantics needed.
**Trade-offs:** Policy is called every cycle for every drifted node, even if the exemption hasn't changed. Acceptable — policy evaluation should be lightweight (check conditions, return decision).
**Sources:** casehubio/platform#486 § duration revert mode (retriggerable)
**Exploration:** quick
**Status:** captured

## D7: Event-based revert via Predicate<ActualState>

**Choice:** ExemptionSpec carries a Predicate<ActualState> checked each reconciliation cycle
**Alternatives:**
- Separate EventSource listener for immediate revert — adds bidirectional wiring complexity between exemptions and event system
- External revoke only — simplest but pushes event wiring to every consumer
**Rationale:** The reconciliation loop already reads ActualState each cycle. Checking a predicate against it is zero additional infrastructure. Covers the primary use case: "revert when device state changes" (e.g., no-motion sensor returns to PRESENT). Simple, testable, works within existing loop data.
**Trade-offs:** Revert is not immediate — it happens on the next reconciliation cycle after the predicate becomes true. For most domains (IoT resync every few minutes), this is acceptable. Domains needing sub-second revert should use imperative revoke via EventSource handlers.
**Sources:** casehubio/platform#486 § event-based revert mode
**Exploration:** quick
**Status:** captured
