# Decisions — #130 Plan Preview + #159 Edge Handling

## D1: Approval gate model — skip-and-recheck

**Choice:** Skip-and-recheck pattern. Plan is generated, approval requested, loop returns. Next cycle checks approval status before re-planning.
**Alternatives:**
- Blocking within cycle — simpler but holds the reconciliation thread; only viable for short approval windows
- Separate approval executor — wrapping TransitionExecutor, but loop would wastefully re-plan every cycle
**Rationale:** Consistent with the existing PendingApproval pattern (per-node approval already uses skip-and-recheck via PendingApprovalHandler). Non-blocking keeps the reconciliation loop responsive. Each cycle either checks pending or plans fresh.
**Trade-offs:** More complex state machine than blocking. Pending plan must be stored and invalidated when desired state changes.
**Sources:** ReconciliationLoop.java:624-671, PendingApprovalHandler.java, ApprovalCheckResult.java
**Exploration:** quick
**Status:** captured

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
**Rationale:** Follows the existing injection pattern. ReconciliationLoop already has injected components for orthogonal concerns: FaultPolicyEngine, DriftPolicyEngine, CbrProposalTracker. PlanApprovalGate fits the same pattern — ReconciliationLoop gets minimal changes (one if-block), gate is testable in isolation.
**Trade-offs:** One more component to wire. But this is the established pattern and CDI handles wiring automatically.
**Sources:** ReconciliationLoop.java:104-118 (existing injected components), DriftPolicyEngine (pattern precedent)
**Exploration:** quick
**Depends on:** D1 (skip-and-recheck drives the gate's state machine)
**Status:** captured

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

**Choice:** TransitionPlanner adds virtual in-degree entries for constraint-matching node pairs before BFS starts. The existing topological sort algorithm is unchanged — constraints are just virtual edges resolved once per plan cycle.
**Alternatives:**
- Pre-sort constraint injection (expand into synthetic Dependency edges) — simpler code but creates temporary graph copies with fake edges that could confuse callers
- Post-sort layer merge — doesn't touch the sort algorithm but complex post-processing with edge cases
**Rationale:** Zero change to the BFS algorithm. Virtual edges are added to the in-degree map during the scan phase, exactly like real dependencies. O(n·m) per cycle (n constrained nodes, m constraints) — negligible for realistic graph sizes (10-200 nodes, 1-5 constraints).
**Trade-offs:** Slightly more code in topologicalSort() to scan for constraint matches. But the algorithm stays identical — just more edges.
**Sources:** TransitionPlanner.java:134-186 (topologicalSort method)
**Exploration:** quick
**Depends on:** D5 (type-level scope determines what pairs to match)
**Status:** captured

## D7: Constraint location — on DesiredStateGraph (revised)

**Choice:** Add `Set<OrderingConstraint> orderingConstraints()` to `DesiredStateGraph` with default `Set.of()`. ImmutableDesiredStateGraph stores and propagates constraints through all mutation methods, overlay, connect, filterByTypes.
**Alternatives:**
- CompilationResult carries constraints — conceptually cleaner separation but requires plumbing through 5+ methods (LifecycleManager.start, ReconciliationLoop.start/updateDesired/compareAndSetDesired, TransitionPlanner.plan)
- Graph-level metadata (Map<String,Object>) — untyped, hidden API
**Rationale:** Zero plumbing. Constraints travel with the graph naturally through every layer (GoalCompiler → LifecycleManager → ReconciliationLoop → TransitionPlanner). The graph already has `dependencies()` which is execution ordering — constraints are the type-level equivalent. `filterByTypes()` can automatically filter constraints (keep only those where both types are in the filter set). `overlay()` merges constraints from both graphs.
**Trade-offs:** Graph carries execution-ordering metadata. But dependencies() already does this — ordering constraints are semantically parallel.
**Sources:** DesiredStateGraph.java, ImmutableDesiredStateGraph.java, LifecycleManager.java:24-52 (graph extraction)
**Exploration:** quick (initially), revised after plumbing analysis
**Depends on:** D5 (type-level scope), D6 (virtual edges need to read constraints from the graph)
**Status:** captured (revised from CompilationResult approach)

## D8: Plan presentation — CloudEvent + PlanPreviewHandler SPI

**Choice:** Runtime emits a CloudEvent with plan data (additions, removals, node specs). New SPI `PlanPreviewHandler` handles presentation and approval tracking. CaseTransitionExecutor can implement it via Flow.
**Alternatives:**
- Runtime generates human-readable diff — TransitionPlan.toDiff() producing structured output; runtime owns format. But limits presentation flexibility.
- Direct Flow integration — tightest integration but couples runtime to engine, violating Foundation tier boundary
**Rationale:** Keeps runtime domain-agnostic and Foundation-tier clean. Consumers decide how to present plans — CLI, web UI, Flow workflow, Slack notification. The CloudEvent pattern is already used throughout the runtime (ReconciliationEventEmitter emits CloudEvents for faults, drift, recovery, etc.).
**Trade-offs:** Domain implementors must provide a PlanPreviewHandler to get approval workflows. But NoOp default auto-approves, so it's opt-in.
**Sources:** ReconciliationEventEmitter (CloudEvent pattern), DesiredStateEventTypes (event type constants), CaseTransitionExecutor (engine-adapter bridge pattern)
**Exploration:** quick
**Depends on:** D4 (gate uses PlanPreviewHandler for submit/check/cancel)
**Status:** captured
