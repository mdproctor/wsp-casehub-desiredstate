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

## Terrain Model

A shared 10x10 grid foundation all three scenarios build on.

**`TerrainGrid`** — 10x10 grid of `TerrainCell` records. Each cell has:
- `row`, `col` (0-indexed coordinates)
- `height` (0, 1, 2)
- `terrainType` — `OPEN`, `CLIFF`, `RAMP`

**Movement rule:** A unit can move from cell A to adjacent cell B (orthogonal only)
if B is not CLIFF and `|A.height - B.height| <= 1`.

**Fog of war:** `FogOfWar` tracks revealed cells. Vision range is configurable
(default 2). When a unit occupies a cell, all cells within `visionRange` manhattan
distance with line-of-sight are revealed. Height affects vision — a unit at height 2
can see over height 1; a unit at height 0 cannot see past height 2.

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

**`UnitSpec`** — `cellId` (NodeId of occupied cell), `strength` (int)

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
   DRIFTED/ABSENT → reconciliation detects drift → zone rebalances.
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
5. **Inject: "high ground at (7,8) revealed as occupied"** → waypoint spec requires more
   force. Ripple through column allocation.

**Evaluates:** Dependency chains for sequential advance (layer 1). Orphan removal + new
node creation for path rerouting (layer 1). Zone group nodes for linear chain allocation
(layer 2). Dynamic rebalancing — can a fault policy redistribute forces along the attack
column without knowing zone internals? (layer 3). Expected: layers 1–2 work, layer 3
shows coupling between fault policy and zone structure.

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
   Zone ratios change → every unit gets new strength.
4. **Inject: zone split** — frontier too wide for one zone. Single ZONE replaced by two ZONEs,
   each with subset of cells and own ratio policy. Structural graph change.

**Layer 3 test (fault policy coupling):**

5. **Inject: losses at (5,1)** — unit destroyed. Fault policy fires. To redistribute
   remaining force, the policy must understand zone membership, current ratios, and
   recompute allocations. Test: can a fault policy do this from `(FaultEvent, DesiredStateGraph)`
   alone, or does it need zone-specific knowledge injected?

**Layer 4 tests (strategic pivot):**

6. **Inject: repeated losses across frontier** — 3 consecutive cycles, each destroying
   units in the same zone. Individually, each is handled by the fault policy (rebuild).
   But cumulatively, the approach is failing. Test: assert that no existing mechanism
   detects the pattern or suggests an alternative.
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

**3. Fault policies need zone knowledge:** When losses occur, a fault policy that wants
to redistribute forces must emit N+1 mutations (update zone spec + update each child
unit spec). It needs intimate knowledge of zone internals — which cells belong to which
zone, what the current ratios are, what the new ratios should be. The fault policy
interface receives `(FaultEvent, DesiredStateGraph)` — the graph is opaque, with no
concept of zone membership or ratio semantics. The policy must reverse-engineer the
zone structure from raw graph traversal.

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

Layers 3–4 provide the concrete evidence for a constraint/evaluation model. The
gap is not in the graph's ability to represent spatial state, but in the system's
ability to reason about aggregate outcomes and strategic alternatives.

## ANSI Terminal Renderer

**`GridRenderer`** — renders `BattlefieldWorld` as ASCII art to the terminal. Uses
ANSI escape codes (`\033[H`) to repaint in place. Each reconciliation step overwrites
the previous frame.

Configurable step delay (default 500ms) for animation. Set to 0 for CI. Can dump
frames to log for static record.

```
Step 3: Scout reveals northeast (after reconciliation)
   0  1  2  3  4  5  6  7  8  9
0 [B] .  .  .  ░░ ░░ ░░ ░░ ░░ ░░
1  .  .  .  .  ░░ ░░ ░░ ░░ ░░ ░░
2  . U5  . ▲2  .  ░░ ░░ ░░ ░░ ░░
3  .  . ▲2 ▲2  .  .  ░░ ░░ ░░ ░░
4 U3  . ▲2  .  .  .  S  ░░ ░░ ░░
  ...
Legend: [B]=base  U5=unit(str:5)  S=scout  ▲2=height:2  ░░=fog  .=revealed
Zone: north-perimeter [U5,U3] ratio:0.6/0.4  budget:8
```

## Module Structure

**Maven artifact:** `casehub-desiredstate-example-spatial`
**Root package:** `io.casehub.desiredstate.example.spatial`

```
io.casehub.desiredstate.example.spatial
├── terrain/
│   ├── TerrainGrid
│   ├── TerrainCell          — record: row, col, height, terrainType
│   ├── TerrainType          — enum: OPEN, CLIFF, RAMP
│   └── FogOfWar             — revealed cells, vision range, line-of-sight
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
│   └── DistributionGoalCompiler
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

## Out of Scope

- No constraint model implementation — the POC identifies the need
- No runtime changes — uses runtime as-is
- No pathfinding algorithm — waypoint paths hand-placed in blueprints
- No combat resolution or AI opponent — injected events only
- No persistence — in-memory per test
