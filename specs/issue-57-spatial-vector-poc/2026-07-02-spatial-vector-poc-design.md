# Spatial/Vector State Representation — Graph Model Stress Test

**Issue:** casehubio/casehub-desiredstate#57
**Date:** 2026-07-02
**Status:** Draft

## Purpose

Evaluate whether the existing graph model can handle spatial and continuous state
problems — or whether a constraint model is needed. Build three standalone scenarios
on a shared terrain foundation, push the graph model until it breaks, and document
the concrete failure modes.

The hypothesis: group nodes (Approach 2) will handle topology and sequencing but
fail on cross-node invariants. The documented failure modes become the evidence
for a constraint model — not speculation, but test cases.

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

**Evaluates:** Dependency chains for sequential advance. Orphan removal + new node creation
for path rerouting. Zone group nodes for linear chain allocation. Expected: works.

## Scenario 3: Force Distribution as Ratios

**Goal:** Maintain force distribution across a frontier where ratios are the constraint.
As the frontier shifts, ratios must be re-evaluated across all participating nodes.

**Setup:** Player controls a band of cells across the map middle. Frontier is the forward
edge closest to unexplored territory.

**`DistributionGoalCompiler`** takes `DistributionBlueprint` (territory, frontier,
budget, priority function) and compiles CELL + ZONE + UNIT nodes. Priority function
assigns weight to cells based on height, adjacency to cliffs, unexplored neighbour count.

**Test flow — designed to break:**

1. Initial compile → frontier of 5 cells, zone allocates 100 force by priority ratios.
2. **Inject: frontier expands** — scouting reveals 3 new cells beyond frontier. Old frontier
   becomes interior. Zone spec must be rebuilt for new frontier.
3. **Inject: losses at (5,1)** — zone total force drops. All units need proportional reduction.
   Zone update and child unit updates are separate transition steps.
4. **Inject: priority shift** — enemy fortification spotted, adjacent cell weights double.
   Zone ratios change → every unit gets new strength.
5. **Inject: zone split** — frontier too wide for one zone. Single ZONE replaced by two ZONEs,
   each with subset of cells and own ratio policy. Structural graph change.

### Expected Failure Modes

**1. Atomicity gap:** Zone spec update and child unit spec updates are separate graph
mutations. Between zone update and last child update, the graph is in an inconsistent
state. Reconciliation sees drift on children not yet updated.

**2. Invisible invariant:** Nothing in the graph enforces that unit strengths sum to
the zone's totalForce, or that allocation ratios sum to 1.0. A concurrent mutation
(fault policy, manual override) can silently break the invariant.

**3. Expensive restructuring:** Zone split is deprovision-all + provision-all — a
complete teardown and rebuild, not a rebalance. Correct but costly and disruptive.

These three failure modes are the concrete evidence for a constraint model.

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

**Scenario 1 (Defense Posture):** Graph model handles it. Zone group nodes work for
static rebalancing. GoalCompiler recompilation is viable for discovery-driven growth
but verbose.

**Scenario 2 (Attack Waypoints):** Graph model handles it. Dependency chains model
sequential advance. Orphan removal + new node creation handles path rerouting cleanly.

**Scenario 3 (Force Distribution):** Graph model breaks. Three documented failure modes
(atomicity gap, invisible invariant, expensive restructuring) provide concrete evidence
for a constraint model.

## Out of Scope

- No constraint model implementation — the POC identifies the need
- No runtime changes — uses runtime as-is
- No pathfinding algorithm — waypoint paths hand-placed in blueprints
- No combat resolution or AI opponent — injected events only
- No persistence — in-memory per test
