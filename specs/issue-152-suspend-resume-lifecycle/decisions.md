# Decisions — #152 Suspend/Resume Lifecycle Verbs

## D1: Idle signal mechanism

**Choice:** Desired graph annotation — nodes annotated with a target lifecycle status
**Alternatives:**
- External idle request API — separate API call to suspend/resume; decouples from graph but adds out-of-band state
- Timer/policy-driven — runtime auto-suspends idle nodes; removes explicit control from GoalCompiler
**Rationale:** Keeps the desired graph as the single source of truth for intent. The GoalCompiler already owns the declaration of what should exist — extending it to declare lifecycle state is natural. No new APIs, no out-of-band state.
**Trade-offs:** GoalCompiler must be re-invoked to change lifecycle state, which adds a recompilation step. Acceptable because goal changes already trigger recompilation.
**Sources:** NodeProvisioner.java, TransitionPlanner.java, issue #152
**Exploration:** quick
**Status:** captured

## D2: NodeStatus extension

**Choice:** Add SUSPENDED to the NodeStatus enum
**Alternatives:**
- Separate LifecycleState enum — more precise but doubles the state surface the planner must reason about
- SUSPENDED as sub-state of PRESENT — avoids changing NodeStatus but makes planner logic indirect
**Rationale:** The planner's core job is comparing actual status vs desired intent. SUSPENDED is a distinct observable state — the resource exists but is dormant. The planner needs to see this in the same dimension it already reasons about.
**Trade-offs:** All existing switch statements on NodeStatus need a new case. But Java's sealed/exhaustive switch will flag every missing case at compile time — safe to extend.
**Sources:** NodeStatus.java, TransitionPlanner.java (switch on NodeStatus)
**Exploration:** quick
**Status:** captured

## D3: Target status on DesiredNode

**Choice:** Add a targetStatus field to DesiredNode record
**Alternatives:**
- NodeSpec.targetStatus() default method — keeps DesiredNode unchanged but couples spec to lifecycle intent
- Graph-level Map<NodeId, TargetStatus> — decouples from node but adds parallel data structure
**Rationale:** DesiredNode is the planner's primary input type. Adding targetStatus directly makes the planner logic straightforward — compare node.targetStatus() vs actual NodeStatus. Default ACTIVE preserves backward compatibility.
**Trade-offs:** DesiredNode record gains a new field. All constructors need updating but the existing 3-arg constructor continues to work with a default.
**Sources:** DesiredNode.java, TransitionPlanner.java
**Exploration:** quick
**Status:** captured

## D4: Result types for suspend/resume

**Choice:** New sealed interfaces SuspendResult and ResumeResult
**Alternatives:**
- Reuse ProvisionResult/DeprovisionResult — fewer types but semantically misleading
- Simpler Success/Failed only — no PendingApproval support; less extensible
**Rationale:** Follows the established pattern exactly. Each lifecycle verb has its own result type. PendingApproval support means human-gated suspend/resume works out of the box. The types are small (3 variants each) and the pattern is proven.
**Trade-offs:** Six new types total (2 sealed interfaces × 3 variants). Worth it for type safety and consistency.
**Sources:** ProvisionResult.java, DeprovisionResult.java
**Exploration:** quick
**Status:** captured

## D5: TransitionPlan structure

**Choice:** Add suspensions and resumptions lists to TransitionPlan
**Alternatives:**
- Unified step list with StepAction — more flexible but loses explicit ordering guarantees
- Encode in existing lists via StepAction — reuses structure but naming becomes misleading
**Rationale:** Execution order matters: removals → suspensions → resumptions → additions. Separate lists make this order explicit and enforceable. The existing pattern (removals + additions) extends naturally.
**Trade-offs:** TransitionPlan grows from 4 fields to 6. All consumers must handle the new lists (even if empty). Acceptable since consumers are few and well-known (SimpleTransitionExecutor, CaseTransitionExecutor).
**Sources:** TransitionPlan.java, SimpleTransitionExecutor.java, CaseTransitionExecutor.java
**Exploration:** quick
**Status:** captured

## D6: ABSENT + SUSPENDED target handling

**Choice:** Attempt resume — the provisioner knows if persisted state is recoverable
**Alternatives:**
- No-op (treat as already suspended) — ignores crashed-but-stateful resources
- Provision fresh then suspend — wasteful, throws away persisted state
**Rationale:** The provisioner has domain knowledge the planner doesn't. For a crashed tmux session with conversation history on disk, resume() can detect and restore the persisted state. If state is gone, the provisioner returns Failed and the node faults — the fault policy decides what to do next.
**Trade-offs:** Resume on an ABSENT node may fail if state is truly gone. But that's the correct behavior — it surfaces the problem instead of silently creating a fresh resource.
**Sources:** Issue #152 (Claudony forcing function), TransitionPlanner.java
**Exploration:** quick
**Status:** captured

## D7: Context types for suspend/resume

**Choice:** Mirror existing ProvisionContext/DeprovisionContext pattern
**Alternatives:**
- Add stateReference field to ResumeContext — more explicit but couples generic runtime to specific stateful pattern
**Rationale:** SuspendContext(tenancyId, graph) and ResumeContext(tenancyId, graph) with optional PlanApproval. The provisioner reads state references from NodeSpec — same pattern as all existing context types. Generic runtime stays domain-agnostic.
**Trade-offs:** No explicit state reference handle in the context type. Provisioners must know where to find state references in their NodeSpec. This is the same pattern used for all other domain data today.
**Sources:** ProvisionContext.java, DeprovisionContext.java
**Exploration:** quick
**Status:** captured
