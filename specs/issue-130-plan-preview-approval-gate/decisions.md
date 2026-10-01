# Decisions — #130 Plan Preview + #159 Edge Handling

## D1: Approval gate model — skip-and-recheck

**Choice:** Skip-and-recheck pattern. Plan is generated, approval requested, loop returns. Next cycle checks approval status before re-planning. While a plan is pending approval, the cycle performs ONLY approval status polling — drift detection, planning, execution, fault feedback, and event emission are all skipped. This prevents drift-triggered fault mutations (via FaultPolicyEngine in detectDrift) from invalidating the pending plan. Type-filtered `reconcileTypes()` continues independently as it serves a different concern (provisioner resync).
**Alternatives:**
- Blocking within cycle — simpler but holds the reconciliation thread; only viable for short approval windows
- Separate approval executor — wrapping TransitionExecutor, but loop would wastefully re-plan every cycle
**Rationale:** Consistent with the existing PendingApproval pattern (per-node approval already uses skip-and-recheck via PendingApprovalHandler). Non-blocking keeps the reconciliation loop responsive. Each cycle either checks pending or plans fresh. The complete operational skip during pending prevents pathological invalidation loops where drift detection mutates the desired graph, invalidating the pending plan, triggering re-plan, re-submit, and repeat.
**Trade-offs:** More complex state machine than blocking. Pending plan must be stored and invalidated when desired state changes. Drift is not detected during the approval window for full-graph cycles, but type-filtered resync continues independently, and drift detection resumes immediately upon approval resolution.
**Sources:** ReconciliationLoop.java:624-671, ReconciliationLoop.java:735-799 (detectDrift calls faultPolicyEngine.evaluate which mutates desiredRef), PendingApprovalHandler.java, ApprovalCheckResult.java
**Exploration:** quick
**Status:** revised (R1-01: clarified that all operations are skipped during pending to prevent drift-triggered invalidation loops)

## D2: Policy model — new SPI PlanApprovalPolicy

**Choice:** New SPI `PlanApprovalPolicy` in api/ that evaluates the whole TransitionPlan and returns auto-approve or require-review.
**Alternatives:**
- Extend TransitionExecutor SPI — mixes execution with policy evaluation
- Configuration-only (property-driven) — less flexible for complex domain-specific approval rules
**Rationale:** Clean separation of concerns. Plan-level approval is independent of per-node approval and independent of execution mechanics. Domain implementors can make nuanced decisions (e.g., auto-approve drift corrections, require review for topology changes).
**Trade-offs:** One more SPI for domain implementors to learn. But it's optional — NoOp default auto-approves everything.
**Sources:** FaultPolicy.java (pattern precedent), DriftPolicy.java (pattern precedent)
**Exploration:** quick
**Status:** captured

## D3: Staleness handling — invalidate and re-plan

**Choice:** When desired state changes while a plan is awaiting approval, invalidate the pending plan. Next cycle generates a fresh plan from the new desired state.
**Alternatives:**
- Keep pending, merge on approval — risk executing a stale plan that no longer matches reality
- Keep pending, warn on diff — more informative but significantly more complex
**Rationale:** Safety first. An approved plan that doesn't match current desired state is dangerous. Re-planning is cheap (milliseconds for realistic graph sizes). The approval workflow restarts cleanly.
**Trade-offs:** Approvers may see plans invalidated mid-review if desired state is changing rapidly. Mitigated: most desired-state changes are infrequent (topology changes, not continuous).
**Sources:** ReconciliationLoop.java:801-821 (plan method), DesiredStateGraph version tracking
**Exploration:** quick
**Status:** captured

## D4: Gate architecture — PlanApprovalGate as injectable component

**Choice:** New `PlanApprovalGate` component injected into ReconciliationLoop, called between plan and execute. Gate encapsulates: pending plan storage, policy evaluation, approval status checking, plan invalidation.
**Alternatives:**
- ReconciliationLoop internal extension — most efficient but significantly increases loop complexity with approval state machine
- Approval-aware TransitionPlanner — breaks TransitionPlanner's pure-function nature; planning and approval are different concerns
**Rationale:** Follows the existing injection pattern. ReconciliationLoop already has injected components for orthogonal concerns: FaultPolicyEngine, DriftPolicyEngine, CbrProposalTracker. PlanApprovalGate fits the same pattern. The integration requires two if-blocks in `reconcile()`: one at cycle start to check pending approval status (skip remaining operations if pending), one after plan generation to evaluate approval policy (store pending and return early if review required). Plus a one-line invalidation call in `updateDesired()`. Gate is testable in isolation.
**Trade-offs:** One more component to wire. But this is the established pattern and CDI handles wiring automatically.
**Sources:** ReconciliationLoop.java:104-118 (existing injected components), DriftPolicyEngine (pattern precedent)
**Exploration:** quick
**Depends on:** D1 (skip-and-recheck drives the gate's state machine)
**Status:** revised (R1-03: corrected "one if-block" to accurately describe integration as two if-blocks plus invalidation hook)

## D5: Ordering constraint scope — type-level only

**Choice:** Ordering constraints are NodeType-level: "all nodes of type A must be provisioned before all nodes of type B."
**Alternatives:**
- Instance-level only — already exists as Dependency edges; no new concept needed but less expressive for the IoT use case
- Both type-level and instance-level — more flexible but over-complicates the model for the stated requirement
**Rationale:** Matches the IoT use case directly ("power-breaker before equipment" is a type-level truth, not per-instance). Instance-level ordering already exists as explicit Dependency edges. Type-level fills the gap.
**Trade-offs:** Cannot express "this specific A before that specific B" without a Dependency edge. Acceptable — instance-level is already covered.
**Sources:** #159 issue body, #153 (IoT consumer requirements), Dependency.java
**Exploration:** quick
**Status:** captured

## D6: Constraint consumption — virtual edges at plan time

**Choice:** TransitionPlanner adds virtual in-degree entries for constraint-matching node pairs before BFS starts. The BFS output processing (layer iteration, in-degree decrement) is unchanged — the input construction phase (in-degree counting) is extended to include virtual edges from ordering constraints.
**Alternatives:**
- Pre-sort constraint injection (expand into synthetic Dependency edges) — simpler code but creates temporary graph copies with fake edges that could confuse callers
- Post-sort layer merge — doesn't touch the sort algorithm but complex post-processing with edge cases
**Rationale:** BFS output processing is unchanged — it still processes layers by in-degree. The input construction phase scans for constraint-matching node pairs and adds virtual in-degree entries, exactly like real dependency edges. Complexity is O(|A|·|B|) per type-pair constraint where A and B are the constrained type's node sets. For the stated use case (1-5 constraints, 10-200 nodes per type), this is negligible. For large IoT fleets (hundreds of nodes per type), constraint resolution could produce thousands of virtual edges, but this is still dominated by the provisioning cost of the actual nodes.
**Trade-offs:** More code in topologicalSort() to scan for constraint matches. Virtual edge count scales quadratically with constrained-type node counts, which should be monitored in high-node-count deployments.
**Sources:** TransitionPlanner.java:134-186 (topologicalSort method)
**Exploration:** quick
**Depends on:** D5 (type-level scope determines what pairs to match)
**Status:** revised (R1-05: replaced "zero change to BFS" with accurate description; acknowledged quadratic scaling in high-node-count scenarios)

## D7: Constraint location — on DesiredStateGraph (revised)

**Choice:** Add `Set<OrderingConstraint> orderingConstraints()` to `DesiredStateGraph` with default `Set.of()`. ImmutableDesiredStateGraph stores and propagates constraints through all mutation methods, overlay, connect, filterByTypes.
**Alternatives:**
- CompilationResult carries constraints — semantically separates "what should exist" from "how to execute changes," but constraints would not participate in graph operations (filterByTypes, overlay, connect) and would require separate propagation through every layer
- Graph-level metadata (Map<String,Object>) — untyped, hidden API
- Provisioner-declared ordering via NodeProvisioner.prerequisiteTypes() — avoids new types but conflates provisioner capability declaration with domain-level execution strategy
**Rationale:** Constraints must participate in graph operations. `filterByTypes()` must filter constraints to only include types in the filter set. `overlay()` must merge constraints from both graphs. `connect()` must preserve constraints. These are structural requirements — if constraints live outside the graph, every graph operation must be accompanied by manual constraint propagation, which is error-prone and violates the principle that graph operations compose correctly. The graph already carries execution-ordering metadata via `dependencies()` (Dependency javadoc: "The runtime guarantees `to` is provisioned before `from`"). Ordering constraints are the type-level equivalent of instance-level dependency edges.
**Trade-offs:** Graph carries type-level ordering metadata alongside instance-level dependency edges. Every consumer of the graph receives ordering constraints whether they need them or not. This is the same trade-off as dependencies — consumers that don't need dependency information already ignore it.
**Sources:** DesiredStateGraph.java, ImmutableDesiredStateGraph.java, LifecycleManager.java:24-52 (graph extraction), Dependency.java javadoc ("The runtime guarantees to is provisioned before from")
**Exploration:** quick (initially), revised after plumbing and graph-operations analysis
**Depends on:** D5 (type-level scope), D6 (virtual edges need to read constraints from the graph)
**Status:** revised (R1-06: replaced cost-based dismissal of CompilationResult with architectural argument about graph-operation participation; added provisioner-declared alternative)

## D8: Plan presentation — CloudEvent + PlanPreviewHandler SPI

**Choice:** Runtime emits a CloudEvent with plan data (additions, removals, node specs). New SPI `PlanApprovalHandler` handles presentation and approval tracking. CaseTransitionExecutor can implement it via Flow.
**Alternatives:**
- Runtime generates human-readable diff — TransitionPlan.toDiff() producing structured output; runtime owns format. But limits presentation flexibility.
- Direct Flow integration — tightest integration but couples runtime to engine, violating Foundation tier boundary
- Collapse PlanApprovalPolicy and PlanApprovalHandler into a single SPI — reduces SPI count but conflates stateless policy evaluation with stateful lifecycle management (persistence, webhooks, UI integration)
**Rationale:** Keeps runtime domain-agnostic and Foundation-tier clean. Consumers decide how to present plans — CLI, web UI, Flow workflow, Slack notification. The CloudEvent pattern is already used throughout the runtime (ReconciliationEventEmitter emits CloudEvents for faults, drift, recovery, etc.). Naming follows the per-node convention: PendingApprovalHandler → PlanApprovalHandler.
**Trade-offs:** Domain implementors must provide a PlanApprovalHandler to get approval workflows. But NoOp default auto-approves, so it's opt-in. Two plan-level SPIs (PlanApprovalPolicy + PlanApprovalHandler) exist alongside the per-node PendingApprovalHandler, but they serve different architectural roles: Policy is stateless domain logic, Handler is stateful infrastructure.
**Sources:** ReconciliationEventEmitter (CloudEvent pattern), DesiredStateEventTypes (event type constants), CaseTransitionExecutor (engine-adapter bridge pattern), PendingApprovalHandler (naming convention)
**Exploration:** quick
**Depends on:** D4 (gate uses PlanApprovalHandler for submit/check/cancel)
**Status:** revised (R1-04: renamed PlanPreviewHandler to PlanApprovalHandler for naming consistency with PendingApprovalHandler; added collapsed-SPI alternative)

## D9: Plan-level and per-node approval interaction model

**Choice:** Plan-level approval (PlanApprovalGate in ReconciliationLoop) and per-node approval (PendingApprovalHandler in NodeStepExecutor) coexist as independent concerns at different layers. Plan approval gates the entire transition plan before execution begins. Per-node approval gates individual node operations during execution. Plan rejection fires a `FaultType.PLAN_REJECTED` fault event through the existing fault pipeline. Type-filtered `reconcileTypes()` runs independently of plan approval.
**Alternatives:**
- Plan approval subsumes per-node approval — simpler but removes per-node granularity needed for multi-team environments where different nodes have different authorization requirements
- Per-node approval only (no plan-level gate) — current state; sufficient for individual nodes but cannot gate coordinated multi-node changes
- CaseTransitionExecutor owns plan gating — tightest engine integration but couples plan approval to the engine tier, making it unavailable to SimpleTransitionExecutor deployments
**Rationale:** Plan approval and per-node approval serve different purposes. Plan approval answers "should this coordinated set of changes proceed?" — an operational review question. Per-node approval answers "is this specific node authorized to change?" — an authorization question. An approver might approve the plan (the overall change is correct) but individual nodes may still require per-node authorization. CaseTransitionExecutor never sees the plan until it's approved — this is intentional, as plan approval is a loop-level concern independent of the execution strategy (Simple vs. Case). Plan rejection produces a fault event so operators have visibility through the existing fault pipeline (CloudEvents, reconciliation state store).
**Trade-offs:** Approvers may encounter two approval layers (plan then per-node). This is the correct behavior for environments with different authorization scopes, but may be confusing in simple deployments — documentation should clarify the interaction.
**Sources:** NodeStepExecutor.java (per-node PendingApprovalHandler integration), CaseTransitionExecutor.java (per-node approval via checkApproval), ReconciliationLoop.java:624-671 (reconcile flow), ReconciliationLoop.java:677-721 (reconcileTypes flow), FaultType.java (existing fault type enum)
**Exploration:** implicit (surfaced by R1-08 review)
**Depends on:** D1 (skip-and-recheck model), D4 (PlanApprovalGate), D8 (PlanApprovalHandler)
**Status:** captured
