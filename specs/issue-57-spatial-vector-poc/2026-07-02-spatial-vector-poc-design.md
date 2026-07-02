# Spatial/Vector State Representation — Graph Model Stress Test

**Issue:** casehubio/casehub-desiredstate#57
**Date:** 2026-07-02
**Status:** Draft

## Purpose

Evaluate whether the existing graph model can handle spatial and continuous state
problems — or whether a constraint/evaluation model is needed. Build three standalone
scenarios on a shared terrain foundation, each pushing progressively harder on the
graph model's limits.

**Evaluation layers** — each scenario exercises all layers up to its ceiling:

1. **Region-as-node, strength-as-spec** — does the basic cell/unit/zone graph model
   work for spatial problems at all?
2. **Group nodes for ratio distribution** — can zone nodes maintain cross-node
   invariants through the provisioner pattern?
3. **Dynamic rebalancing via fault policy** — can fault policies redistribute forces
   without intimate knowledge of zone internals?
4. **Strategic pivot** — can the system recognise "this whole approach is failing"
   and switch to an alternative?

Defense posture exercises layers 1–2. Attack waypoints exercises 1–3. Force
distribution exercises 1–4, with the strategic pivot as the capstone.

The hypothesis: group nodes will handle topology and sequencing (layers 1–2) but
the model has no mechanism for aggregate subgraph evaluation or strategic
alternatives (layers 3–4). The documented failure modes become evidence for a
constraint/evaluation model — not speculation, but test cases.

## Relationship to Other POCs

Issue #57 recommends ordering: #58 (lifecycle) → #56 (GoalCompiler) → #57 (spatial).
This POC is designed to be evaluated independently, but its findings inform both:

- **#58 (lifecycle):** Scenario 2's path rerouting — old waypoints orphaned, deprovisioned,
  new nodes provisioned — exercises a lightweight form of lifecycle transition. If #58
  establishes a formal completion/successor mechanism, scenario 2's graph swap would use
  it rather than raw `updateDesired()`.
- **#56 (GoalCompiler):** This POC tests the graph model at the tactical level — cells,
  units, zones. Epic #24's strategic level (nodes as strategic goals, provisioners as
  tactical plans) is #56's domain. This POC's findings feed #56: if spatial state needs
  a constraint/evaluation model, then strategic nodes managing spatial resources need it too.

The POC does not depend on #58 or #56's conclusions. All scenarios use `updateDesired()`
directly for graph swaps, and GoalCompiler compilation is hand-coded domain logic (not
planner-backed). If #58 or #56 produce different conclusions than expected, this POC's
spatial findings remain independently valid.

## Terrain Model

A shared 10x10 grid foundation all three scenarios build on.

**`TerrainGrid`** — 10x10 grid of `TerrainCell` records. Each cell has:
- `row`, `col` (0-indexed coordinates)
- `height` (0, 1, 2)
- `terrainType` — `OPEN`, `CLIFF`, `RAMP`

**Fog of war:** `FogOfWar` tracks revealed cells. Vision range is configurable
(default 2). When a unit occupies a cell, all cells within `visionRange` manhattan
distance are revealed.

The grid is pre-built (the map exists), but knowledge of the grid is progressive
(fog). Scenarios translate terrain into graph nodes.

## Shared Domain Types

### Node Types

| Type | Purpose |
|------|---------|
| `CELL` | A revealed terrain cell |
| `UNIT` | A force unit placed on a cell |
| `SCOUT` | A unit whose purpose is revealing fog |
| `ZONE` | A group node for force distribution (composite pattern) |

### NodeSpec Implementations

**`CellSpec`** — `row`, `col`, `height`, `terrainType`

**`UnitSpec`** — `cellId` (NodeId of occupied cell), `strength` (int — computed by
GoalCompiler as `zone.totalForce × zone.allocation.get(cellId)`, not independently set.
The zone allocation is authoritative; UnitSpec carries the provisioned value for
ActualStateAdapter comparison.)

**`ScoutSpec`** — `cellId` (NodeId of occupied cell), `visionRange` (int)

**`ZoneSpec`** — `zoneName`, `Map<NodeId, Double> allocation` (cell → ratio, sums to 1.0),
`totalForce` (int)

### Dependency Structure

- `UNIT → CELL` — unit depends on its cell (cell provisioned first)
- `SCOUT → CELL` — same
- `ZONE → CELL` — zone depends on all its member cells
- `UNIT → ZONE` — units allocated by a zone depend on that zone

### World Layer

**`BattlefieldWorld`** — in-memory simulation state. Holds terrain grid, fog of war,
placed units, zone allocations. Shared across scenarios, configured differently by each.

**`BattlefieldActualStateAdapter`** — reads `BattlefieldWorld`:
- Cell revealed → PRESENT; not revealed → ABSENT
- Unit placed and matching spec → PRESENT; strength differs → DRIFTED
- Zone: all member units at correct ratios → PRESENT; any deviation → DRIFTED

**`BattlefieldProvisioner`** — writes `BattlefieldWorld`. Handles CELL, UNIT, SCOUT, ZONE.
- CELL: marks cell revealed, triggers fog update
- SCOUT: places scout, updates fog (reveals within vision range)
- ZONE: validates ratios, computes absolute values from totalForce × ratios
- UNIT: places unit with strength from zone allocation

### Graph Swap Behavior

When scenarios call `ReconciliationLoop.updateDesired()` during active reconciliation,
the runtime's atomic reference guarantees: the current cycle completes against the
previous graph, and the next cycle picks up the new graph. Orphaned nodes (provisioned
from the old graph, absent in the new) are detected as "actual without desired" and
deprovisioned in the subsequent reconciliation cycle — not atomically with the graph
swap. Scenario test flows that swap graphs (scenario 1 step 2, scenario 2 steps 2–3,
scenario 3 step 2) verify this orphan-cleanup behavior.

## Scenario 1: Defense Posture

**Goal:** Distribute defensive forces across revealed cells near a base. As fog lifts
and new cells are discovered, rebalance the distribution.

**Setup:** Base at corner (0,0). Cluster of cells initially revealed. Surrounding
terrain has height variation creating chokepoints.

**`DefenseGoalCompiler`** takes `DefenseBlueprint` (base position, scout positions,
defense budget, zone definitions) and compiles to CELL + SCOUT + ZONE + UNIT nodes.

**Test flow — scripted events, no opponent simulation:**

1. Compile initial graph → provision base area + scouts.
2. Scouts reveal new terrain → recompile with updated knowledge → reconcile (graph growth).
3. **Inject: "8 of 10 units at north-perimeter destroyed"** → ActualStateAdapter reports
   units as ABSENT → planner re-provisions them with original specs. Zone reports DRIFTED
   until all units are restored. This is restoration, not rebalancing — the default runtime
   behavior for ABSENT nodes. Rebalancing (redistributing among fewer units) would require
   GoalCompiler recompilation with updated zone definitions.
4. **Inject: "enemy fortification spotted at (4,3)"** → cell spec gains tactical weight →
   zone priorities shift, forces redistribute.
5. **Inject: "ramp collapsed at (2,5)"** → terrain change → forces depending on that route
   need reallocation.

**Evaluates:** Discovery-driven graph growth via GoalCompiler recompilation. Zone group
nodes for static rebalancing. Expected: works, recompilation pattern is verbose but functional.

## Scenario 2: Attack with Waypoints

**Goal:** Plan an attack path through revealed terrain toward a target, with force
allocation at each waypoint that adapts to discoveries and injected events.

**Setup:** Base at known position, target direction (northeast). Path discovered
through scouting.

**`AttackGoalCompiler`** takes `AttackBlueprint` (origin, target direction, budget,
waypoint spacing) and compiles waypoints as a dependency chain — waypoint N+1
depends on waypoint N.

**Test flow:**

1. Initial compile → scouts heading northeast.
2. Scouts reveal terrain: height-2 ridge blocks direct path, ramp at (3,6). Recompile →
   waypoints route through ramp.
3. **Inject: "ramp at (3,6) collapsed"** → route invalidated. Recompile → new path. Old
   waypoint nodes orphaned → deprovisioned. New nodes added → provisioned.
4. **Inject: "heavy losses at waypoint (5,7)"** → zone rebalances attack column.
   Test via both paths: GoalCompiler recompilation (structural) and direct `UpdateNode`
   mutations on zone spec + child unit specs (parametric). If incremental mutation works
   cleanly, ratio changes don't require full recompilation.
5. **Inject: "high ground at (7,8) revealed as occupied"** → waypoint spec requires more
   force. Ripple through column allocation.

**Evaluates:** Dependency chains for sequential advance (layer 1). Orphan removal + new
node creation for path rerouting (layer 1). Zone group nodes for linear chain allocation
(layer 2). Rebalancing in a linear dependency chain requires either full recompilation or
N+1 incremental mutations — the same "verbose but functional" finding from layer 2,
applied to a sequential topology. Expected: layers 1–2 work.

## Scenario 3: Force Distribution as Ratios

**Goal:** Maintain force distribution across a frontier where ratios are the constraint.
As the frontier shifts, ratios must be re-evaluated across all participating nodes.

**Setup:** Player controls a band of cells across the map middle. Frontier is the forward
edge closest to unexplored territory.

**`DistributionGoalCompiler`** takes `DistributionBlueprint` (territory, frontier,
budget, priority function) and compiles CELL + ZONE + UNIT nodes. Priority function
assigns weight to cells based on height, adjacency to cliffs, unexplored neighbour count.

**Test flow — progressively harder:**

**Layer 2 tests (ratio distribution):**

1. Initial compile → frontier of 5 cells, zone allocates 100 force by priority ratios.
2. **Inject: frontier expands** — scouting reveals 3 new cells beyond frontier. Old frontier
   becomes interior. Zone spec must be rebuilt for new frontier.
3. **Inject: priority shift** — enemy fortification spotted, adjacent cell weights double.
   Zone ratios change → every unit gets new strength. Test via both paths: GoalCompiler
   recompilation (full graph rebuild) and incremental `UpdateNode` mutations (zone spec +
   N child unit specs). If incremental mutation works cleanly for parameter changes, the
   "verbose but functional" finding narrows to topology changes only.
4. **Inject: zone split** — frontier too wide for one zone. Single ZONE replaced by two ZONEs,
   each with subset of cells and own ratio policy. Structural graph change.

**Layer 3 test (fault policy coupling):**

5. **Inject: losses at (5,1)** — unit destroyed in BattlefieldWorld. The reconciliation
   loop's actual path: ActualStateAdapter reports the unit as ABSENT and the zone as DRIFTED.
   `detectDrift()` creates `FaultEvent(zoneId, NODE_DEGRADED, ...)` — the fault is on the
   **zone**, not the unit. `ZoneRebalanceFaultPolicy` receives this event and attempts
   redistribution from `(FaultEvent, DesiredStateGraph)` alone:

   (a) The policy knows the zone is degraded but NOT why — the FaultEvent carries the zone's
   NodeId and a generic detail string, not which child unit was lost. The `FaultPolicy` SPI
   receives `(FaultEvent, DesiredStateGraph)` but not `ActualState`. To determine which units
   are missing, the policy would need actual state — which it doesn't have.

   (b) Even if the policy could identify the missing unit, it faces a planner conflict:
   the `TransitionPlanner` independently detects the destroyed unit as ABSENT and schedules
   it for re-provisioning. The policy's redistribution mutations and the planner's restoration
   compete. The policy might update the zone's allocation to exclude cell (5,1), but the
   planner will still re-provision the unit with whatever `UnitSpec` is in the desired graph.

   (c) The policy cannot remove the destroyed unit from the desired graph — `RemoveNode`
   destroys dependency edges (§8 anti-pattern in ARC42STORIES), making the topology
   unrecoverable without re-invoking the GoalCompiler.

   This step produces three concrete findings: the fault policy SPI lacks actual state
   access (information gap), fault policies and the planner conflict on the same nodes
   (coordination gap), and the graph model cannot express "this node should no longer
   exist" without destroying topology (mutation gap). These are stronger evidence for a
   constraint/evaluation model than the originally hypothesized "reverse-engineering zone
   structure" difficulty.

**Layer 4 tests (strategic pivot):**

6. **Inject: repeated losses across frontier** — 3 consecutive cycles, each destroying
   units in the same zone. Individually, each loss is handled by the runtime — the planner
   restores destroyed units with original specs (step 5 showed the fault policy cannot
   redistribute). But cumulatively, the approach is failing. Test: assert that no existing
   mechanism detects the pattern or suggests an alternative.
7. **Inject: "enemy has fortified heavily, approach from south instead"** — the entire
   frontier subgraph must be abandoned and a new frontier compiled in a different area.
   The GoalCompiler can produce the new graph. The reconciliation loop can transition.
   But the *decision* to pivot has no home — test asserts that the pivot requires
   external intervention (manual recompilation), not an autonomous system response.

### Expected Failure Modes

**Layer 2 — ratio invariants (may hold):**

These were the original hypothesis for breakage but may be weaker than expected.
The transition planner processes zone before children (dependency order), and
GoalCompiler recompilation replaces the graph atomically. The POC tests these
honestly — if they pass clean, that's a valid finding.

**1. Invisible invariant:** Nothing in the graph enforces that unit strengths sum to
the zone's totalForce, or that allocation ratios sum to 1.0. A rogue GraphMutation
bypassing the GoalCompiler can silently break the invariant. Real concern, but more
of a code-safety issue than a model limitation.

**2. Expensive restructuring:** Zone split is deprovision-all + provision-all — a
complete teardown and rebuild, not a rebalance. Correct but costly and disruptive.

**Layer 3 — fault policy coupling (likely breaks):**

**3. Fault policy information gap:** The `FaultPolicy` SPI receives
`(FaultEvent, DesiredStateGraph)` but not `ActualState`. When the zone is DRIFTED, the
policy knows something is wrong but cannot determine which child unit was lost without
actual state access. The FaultEvent carries the zone's NodeId and a generic detail
string — not enough to diagnose the specific failure.

**9. Fault policy / planner conflict:** The fault policy and the `TransitionPlanner`
independently respond to the same situation. The policy fires during `detectDrift()`
and may emit mutations to redistribute force. The planner runs AFTER drift detection
and sees the destroyed unit as ABSENT → schedules re-provisioning. The policy's
redistribution and the planner's restoration compete: the policy may update the zone's
allocation, but the planner will still re-provision the destroyed unit with whatever
spec is in the desired graph. There is no mechanism for the fault policy to signal
"do not re-provision this node" — the planner has no concept of policy intent.

**10. RemoveNode destroys topology:** A fault policy that wants to exclude a destroyed
unit from the zone cannot use `RemoveNode` — it destroys all dependency edges (per §8
anti-pattern in ARC42STORIES), making the topology unrecoverable without re-invoking
the GoalCompiler. The policy cannot express "this node should no longer exist" without
collateral damage to the graph structure.

**Layer 4 — strategic pivot (genuinely breaks):**

**4. No aggregate subgraph evaluation:** The fault policy sees individual node failures
in isolation. Five failures in the same zone are five independent fault events — there's
no mechanism to detect "this whole approach is failing." The system cannot evaluate
cumulative cost or success rate of an approach.

**5. No strategic alternatives:** The fault policy can rebuild what failed but cannot
say "stop rebuilding and try a different approach." It has no concept of alternative
plans. The decision "this attack path is too costly, try from the south instead"
requires reasoning about the graph as a whole, comparing alternative subgraphs, and
making a cost/benefit evaluation that lives outside any existing SPI.

**6. No correlated fault detection:** Related failures (multiple units lost along the
same approach) are treated as independent events. There's no mechanism to correlate
them and recognise a pattern — "we keep losing units on this approach" is invisible
to the per-node fault model.

**7. Group-membership gap:** The graph API has no concept of group membership.
`DesiredStateGraph` offers `dependenciesOf(node)` and `dependentsOf(node)` for edge
traversal, but no `membersOf(group)` or `groupOf(member)` query. Composite node
semantics (zone → member cells, zone → allocated units) are enforced entirely by
domain convention — fragile and convention-dependent. This is a specific capability
a constraint/evaluation model would need.

**8. ActualStateAdapter coupling:** The adapter for composite nodes inherits the same
coupling problem as fault policies. Computing zone status (PRESENT vs DRIFTED) requires
the adapter to understand zone structure, ratio semantics, and the relationship between
zone specs and unit specs. The "opaque graph" problem extends beyond fault policies to
any component that reasons about composite node state.

Layers 3–4 provide the concrete evidence for a constraint/evaluation model. The
gap is not in the graph's ability to represent spatial state, but in the system's
ability to reason about aggregate outcomes and strategic alternatives. Findings 3,
7–10 identify specific capabilities that model would need: fault policy access to
actual state (3), group-membership queries (7), composite-aware state reading (8),
coordinated policy/planner intent (9), and safe node removal (10).

## ANSI Terminal Renderer

`GridRenderer` provides ASCII art visualization of `BattlefieldWorld` for debugging,
with step-by-step animation of reconciliation cycles. Configurable step delay (default
500ms, 0 for CI).

## Module Structure

**Maven artifact:** `casehub-desiredstate-example-spatial`
**Root package:** `io.casehub.desiredstate.example.spatial`

```
io.casehub.desiredstate.example.spatial
├── terrain/
│   ├── TerrainGrid
│   ├── TerrainCell          — record: row, col, height, terrainType
│   ├── TerrainType          — enum: OPEN, CLIFF, RAMP
│   └── FogOfWar             — revealed cells, vision range
├── world/
│   ├── BattlefieldWorld
│   ├── BattlefieldActualStateAdapter
│   └── BattlefieldProvisioner
├── specs/
│   ├── CellSpec
│   ├── UnitSpec
│   ├── ScoutSpec
│   ├── ZoneSpec
│   └── SpatialNodeTypes
├── defense/
│   ├── DefenseBlueprint
│   └── DefenseGoalCompiler
├── attack/
│   ├── AttackBlueprint
│   └── AttackGoalCompiler
├── distribution/
│   ├── DistributionBlueprint
│   ├── DistributionGoalCompiler
│   └── ZoneRebalanceFaultPolicy
└── render/
    └── GridRenderer         — ANSI in-place repaint
```

**Tests:**

```
├── defense/
│   └── DefensePostureTest
├── attack/
│   └── AttackWaypointsTest
└── distribution/
    └── ForceDistributionTest
```

**Dependencies:** `casehub-desiredstate-api` (compile), `casehub-desiredstate` runtime (test),
`casehub-desiredstate-testing` (test), JUnit 5 (test). No Quarkus.

## Expected Findings

**Scenario 1 (Defense Posture) — layers 1–2:** Graph model handles it. Zone group nodes
work for static rebalancing. GoalCompiler recompilation is viable for discovery-driven
growth but verbose. No breakage expected.

**Scenario 2 (Attack Waypoints) — layers 1–3:** Graph model handles topology and
sequencing (layers 1–2). Layer 3 shows coupling: fault policies that need to redistribute
forces along the attack column require intimate knowledge of zone internals. The fault
policy interface is too opaque for spatial reasoning.

**Scenario 3 (Force Distribution) — layers 1–4:** Layers 1–2 may work better than
expected (ratio invariants enforced by GoalCompiler + provisioner dependency ordering).
Layer 3 confirms the fault-policy coupling finding from Scenario 2. Layer 4 is where
the model genuinely has no answer — no aggregate evaluation, no strategic alternatives,
no correlated fault detection. The system can handle any individual failure but cannot
reason about whether an entire approach is succeeding or failing.

**Overall finding:** The graph model handles spatial *topology* well. It fails on spatial
*strategy* — the layer above topology where the system needs to evaluate outcomes,
detect patterns across failures, and choose between alternative approaches. This is
the evidence for a constraint/evaluation model.

## Conditional Deliverables

If layers 3–4 fail as expected, the POC produces these deliverables as findings
(per issue #57 acceptance criteria):

**Interface sketch for constraint/evaluation model:** Based on the specific failure
points documented by the `ZoneRebalanceFaultPolicy` and strategic pivot tests, sketch
what an evaluation SPI would look like — aggregate subgraph health assessment,
correlated fault detection, and alternative-plan evaluation. The sketch targets the
gaps the POC identified, not a generic solution.

**Reconciliation/drift/fault semantics for the new model:** Define what drift means
for aggregate evaluation ("this approach is failing" vs "this node drifted"), what
fault means at the strategic level (pattern of failures vs individual failure), and
how reconciliation interacts with plan-level evaluation.

**Runtime factoring cost assessment:** Using epic #24's coupling analysis — 
`FaultPolicyEngine` (loosely coupled, interface-based), `EventSource`/`SituationSource`
(fully generic), `ReconciliationLoop` (tightly graph-coupled), `TransitionPlanner`
(assumes DAG, Kahn's algorithm) — assess what's shared across representations, what
needs replacement, and what the factoring boundary looks like.

If the graph model handles all layers cleanly, these deliverables are not produced.
The POC tests the hypothesis honestly — the deliverables are conditional on the
findings, not predetermined.

## Out of Scope

- No constraint model implementation — the POC identifies the need; follow-up
  tracked as #59 (conditional on POC findings)
- No runtime changes — uses runtime as-is; if findings indicate runtime factoring is
  needed, that work is tracked as #60
- No pathfinding algorithm — waypoint paths hand-placed in blueprints
- No combat resolution or AI opponent — injected events only
- No persistence — in-memory per test
