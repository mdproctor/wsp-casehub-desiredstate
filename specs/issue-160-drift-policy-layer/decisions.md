# Decisions — #160 Drift Policy Layer

## D1: DriftPolicy interception point in reconciliation cycle

**Choice:** Dual interception — suppress FaultEvents in `detectDrift()` AND pass exempt node set to the planner so it skips PROVISION for exempt DRIFTED nodes
**Alternatives:**
- New `filterExemptDrift()` step between detectDrift and plan — cleaner SRP but requires refactoring detectDrift to split detection from fault event creation, unnecessary churn
- Intercept in TransitionPlanner only — fault events already fired, breaks pre-planning principle
- Intercept in detectDrift only — suppresses fault path but planner independently sees DRIFTED in actual state and plans PROVISION (see R1-01)
- TargetStatus extension (DRIFT_TOLERATED) — conflates desired state with operational policy (see R1-02 rejection)
**Rationale:** The reconciliation cycle has two independent paths for DRIFTED nodes: (1) the fault path in `detectDrift()` creates NODE_DEGRADED FaultEvents → FaultPolicyEngine, and (2) the planning path where `TransitionPlanner.decideAction(DRIFTED, ACTIVE)` returns `StepAction.PROVISION`. Both must be aware of exemptions. `detectDrift()` collects the exempt node set during policy evaluation and passes it to `plan()`. The planner skips PROVISION for exempt DRIFTED nodes via a `Set<NodeId> exemptNodes` parameter.
**Trade-offs:** detectDrift() grows in responsibility (detection + policy evaluation + exempt set collection), and the planner gains a new parameter. The alternative (TargetStatus extension) would avoid the parameter threading but conflates desired state with operational policy — see D1 alternatives and R1-02 rejection rationale.
**Sources:** ReconciliationLoop.java:706-741 (detectDrift method), TransitionPlanner.java:95-107 (decideAction — DRIFTED→PROVISION for ACTIVE target), FaultPolicyEngine.java (existing fault evaluation pattern)
**Exploration:** quick
**Status:** revised — R1-01 correctly identified the planner gap; both paths need exemption awareness

## D2: ExemptionStore SPI design

**Choice:** Thin store — key=(tenancyId, nodeId), value=Exemption record with declarative `RevertCondition`. grant/revoke/get/evict. Loop checks expiry each cycle.
**Alternatives:**
- Rich store with internal expiration timers — unnecessary machinery since the loop already re-evaluates each cycle
- Exemptions as graph metadata — conflates desired state with operational policy, semantically wrong
- Predicate<ActualState> for revert conditions — not serializable, not inspectable, complicates JPA persistence and api/ SPI contracts (see R1-04)
**Rationale:** Follows the established FaultCountStore pattern. No namespace needed — unlike fault counts, exemptions don't need multi-policy scoping because there's one active exemption per node (first-EXEMPT-wins evaluation means only one policy grants per node per cycle). The reconciliation loop checks exemption expiry naturally during each cycle — expired exemptions simply result in RECONCILE on the next pass. Revert conditions use a declarative `RevertCondition` sealed type instead of `Predicate<ActualState>` — serializable, inspectable, and consistent with the platform's sealed type patterns.
**Trade-offs:** The declarative RevertCondition is less flexible than arbitrary predicates — new condition types require extending the sealed type. This is acceptable because: (a) the stated use cases (status-change revert, never-revert) are covered, (b) new variants are added by extending the sealed interface, and (c) serializability and inspectability outweigh flexibility for an api/ SPI.
**Sources:** FaultCountStore.java (SPI pattern), InMemoryFaultCountStore.java (in-memory impl pattern), ReconciliationStateStore.java (store SPI pattern)
**Exploration:** quick
**Status:** revised — R1-04 correctly identified Predicate<ActualState> as incompatible with api/ SPI patterns; replaced with declarative RevertCondition

## D3: DriftPolicy SPI shape — chain via DriftPolicyEngine

**Choice:** Multiple DriftPolicy beans aggregated by DriftPolicyEngine, first-EXEMPT-wins semantics with `@Priority`-based evaluation ordering (highest priority evaluates first)
**Alternatives:**
- Single policy, no chain — simpler but breaks multi-domain composition (CrossDomainCompositionEngine already supports multiple domains contributing SPIs)
- Priority-ordered chain with RECONCILE overriding EXEMPT — overly complex, the default IS reconcile, policies can only escalate to exempt
- All-policies-run with EXEMPT/RECONCILE arbitration (FaultPolicyEngine model) — FaultPolicyEngine merges MUTATIONS (additive, can conflict); DriftPolicyEngine evaluates PERMISSIONS (binary decision). These are different semantic domains — applying the FaultPolicyEngine composition model to a permission decision adds complexity without benefit
- Non-deterministic ordering — would make exemption properties (duration, revert condition) unpredictable when multiple policies can EXEMPT the same node
**Rationale:** Follows the FaultPolicy/FaultPolicyEngine structural pattern (multiple SPI beans composed by an engine). The composition semantics differ because the domains differ: FaultPolicyEngine merges additive mutations with conflict detection; DriftPolicyEngine evaluates a binary permission where RECONCILE is the default and any policy can grant EXEMPT. Multiple domains can contribute policies. First-EXEMPT-wins is the natural model for permission grants — analogous to RBAC where any granted role suffices. DriftPolicy beans MUST declare `@Priority` — the engine evaluates in priority order (highest first), and the first EXEMPT terminates evaluation. This is required because CDI `Instance<T>.stream()` ordering is not guaranteed by the CDI specification; without `@Priority`, first-EXEMPT-wins produces non-deterministic results. This follows CDI conventions and the platform's established priority-based ordering patterns.
**Trade-offs:** Multiple policies evaluating per node per cycle has a performance cost, but this is bounded by the number of DriftPolicy beans (typically 1-3 per deployment) and is negligible compared to actual state reads and provisioning. First-EXEMPT-wins means a single policy can override the default for any node — cross-domain integrity depends on drift exemption not violating provides/requires contracts (it doesn't — a DRIFTED node is still PRESENT from a dependency perspective). Requiring `@Priority` adds a declaration burden but makes the evaluation contract explicit — a deployment where two policies can EXEMPT the same node has deterministic, predictable behaviour.
**Sources:** FaultPolicyEngine.java (chain evaluation pattern), CrossDomainCompositionEngine (multi-domain SPI contribution), RuntimeBeans.java:37-41 (CDI Instance wiring pattern)
**Exploration:** quick
**Status:** revised — R1-05 clarified composition semantics divergence; R2-02 correctly identified that first-EXEMPT-wins requires deterministic ordering via @Priority

## D4: Observability — new event type for permitted drift

**Choice:** Emit a distinct `NodeDriftExemptedData` CloudEvent for drift-exempt nodes. Document as breaking change for existing `NodeDriftedData` consumers.
**Alternatives:**
- Silent — exempt nodes produce no events, but operators can't observe exemptions without querying the store
- Reuse NodeDriftedData with an `exempt` flag — conflates two different semantic signals in one event type
- Emit both NodeDriftedData + NodeDriftExemptedData for exempt nodes — preserves backward compatibility but dilutes NodeDriftedData semantics ("drift requiring action" would include nodes where no action is taken)
**Rationale:** Dashboards and operators need visibility into permitted drift. A separate event type keeps the semantic contract clean — NodeDriftedData means "unexpected drift requiring action," NodeDriftExemptedData means "drift detected but permitted by policy." This is a breaking semantic contract change: existing consumers counting NodeDriftedData for total drift visibility will undercount. The spec must document this as a breaking change and provide migration guidance (consumers wanting total drift counts should subscribe to both event types).
**Trade-offs:** Breaking change for existing NodeDriftedData consumers. Acceptable — the semantic clarity is worth the migration cost, and emitting both events would make NodeDriftedData semantically incorrect (it would include nodes where no action is required).
**Sources:** ReconciliationEventEmitter.java (event emission pattern), NodeDriftedData.java (existing drift event), DesiredStateEventTypes.java (event type constants)
**Exploration:** quick
**Status:** revised — R1-08 correctly identified the contract change must be documented as breaking

## D5: Exemption granting — both SPI and imperative

**Choice:** DriftPolicy.evaluate() for declarative per-cycle evaluation, plus ExemptionStore.grant()/revoke() for imperative use by external event handlers
**Alternatives:**
- SPI only — external code would need to influence policy inputs indirectly, less flexible for reactive domains like IoT
- Imperative only — no per-cycle evaluation hook, loses the ability for policies to react to changing conditions each cycle
**Rationale:** IoT triggers (motion sensors, manual overrides) need imperative grant when the trigger event fires. Policies need per-cycle evaluation for rule-based exemptions (e.g., "exempt this node type during business hours"). Both paths write to the same ExemptionStore — the loop just checks the store, agnostic to who wrote the exemption.
**Trade-offs:** Two paths to the same store could lead to conflicts (policy grants, external revokes same cycle). Resolved by: imperative revoke always wins — if external code revokes, the policy must re-evaluate and re-grant on the next cycle if the exemption should persist. Implementation responsibility: when an external handler calls `revoke()`, it must also update the state that the DriftPolicy reads (e.g., set a flag, update a condition source) so the policy returns RECONCILE on the next cycle. If the handler only calls `revoke()` without updating policy inputs, the policy will re-grant on the next cycle — this is a usage error, not a framework defect. The SPI Javadoc must document this responsibility clearly.
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

## D7: Event-based revert via declarative RevertCondition

**Choice:** ExemptionSpec carries a `RevertCondition` (sealed type) checked each reconciliation cycle
**Alternatives:**
- Predicate<ActualState> — maximum flexibility but not serializable, not inspectable, complicates JPA persistence and api/ SPI contracts
- Separate EventSource listener for immediate revert — adds bidirectional wiring complexity between exemptions and event system
- External revoke only — simplest but pushes event wiring to every consumer
**Rationale:** The reconciliation loop already reads ActualState each cycle. Checking a declarative RevertCondition against it is zero additional infrastructure. `RevertCondition` is a sealed type: `OnStatusChange(Set<NodeStatus>)` reverts when the node transitions to any of the specified statuses (e.g., `OnStatusChange(Set.of(PRESENT))` reverts when a drifted node becomes conformant); `Never` means the exemption never auto-reverts (only expires via duration or imperative revoke). Covers the primary use cases while remaining serializable, inspectable, and loggable. Follows the platform's pattern of sealed types over opaque callbacks.
**Trade-offs:** Less flexible than `Predicate<ActualState>` — conditions beyond NodeStatus change require extending the sealed type. Acceptable: the stated use cases are covered, and new variants (e.g., `OnPropertyChange`) are straightforward extensions. Revert is not immediate — it happens on the next reconciliation cycle after the condition becomes true. Domains needing sub-second revert should use imperative revoke via EventSource handlers.
**Sources:** casehubio/platform#486 § event-based revert mode
**Exploration:** quick
**Status:** revised — R1-04 correctly identified Predicate<ActualState> as a category error in an api/ SPI; replaced with declarative RevertCondition sealed type
