# Summarisation→RAS Integration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #74 — design: logistics example with blocks summarisation feeding RAS — integration scope
**Issue group:** #74

**Goal:** Close the design issue by filing scoped child issues on casehub-blocks and casehub-ops, then closing #74 with the design decision.

**Architecture:** Issue #74 is a design/scoping issue. The design spec establishes that all implementation work belongs in casehub-blocks (bridge module, YAML surface, logistics example) and casehub-ops (deployment + summarisation enhancement). No code changes in casehub-desiredstate. The deliverables are the spec, filed child issues, and closing #74.

**Tech Stack:** GitHub CLI (`gh`), Markdown

## Global Constraints

- Pre-release platform — child issues should reference the design spec for full context
- Each child issue must be self-contained with its own acceptance criteria
- Issues filed against the correct repos (blocks vs ops)
- Scale and complexity estimated per child issue

---

## Batch 1: File child issues and close #74

### Task 1: File casehub-blocks child issues

**Files:**
- Reference: `specs/issue-74-topology-implementation/2026-09-02-summarisation-ras-integration-design.md`

**Interfaces:**
- Consumes: design spec §7 (Deliverables and Issue Map)
- Produces: 4 GitHub issues on casehubio/casehub-blocks

- [ ] **Step 1: File API extraction issue**

```bash
gh issue create --repo casehubio/casehub-blocks \
  --title "feat: extract summarisation types to casehub-blocks-summarisation-api" \
  --body "$(cat <<'EOF'
## Context

Design spec: casehubio/casehub-desiredstate#74 — Summarisation→RAS Integration

## Scope

Extract pure-Java summarisation types from the monolithic `casehub-blocks` jar into `casehub-blocks-summarisation-api`:

- `Summariser<IN,OUT>`, `LevelEvent<E>`, `EventStreamBus<E>`, `WindowPolicy`
- `EventAccumulator<E>`, `Compactor<E>`, `EventLevel`, `SummarisationRunner<IN,OUT>`
- `KeyedAccumulator<K,E>`, `KeyedSummarisationRunner<K,IN,OUT>`

**NOT extracted** (remain in casehub-blocks): `ContentSummariser`, `TieredContentSummariser`, `LlmContentSummariser`, observation package — these depend on qhorus-api/platform-agent-api.

**Breaking change:** `LevelEvent` gains a `@Nullable String tenancyId` component. Existing callers must be updated.

## Acceptance Criteria

- [ ] New `summarisation-api/` module with zero external dependencies (pure Java)
- [ ] All 10 types extracted and compiling
- [ ] `casehub-blocks` depends on `summarisation-api/` — no duplicate classes
- [ ] Existing blocks tests pass unchanged (except LevelEvent constructor updates)
- [ ] Module published to Maven

## Scale / Complexity

S / Low — mechanical extraction of existing types
EOF
)"
```

- [ ] **Step 2: File CloudEvent bridge module issue**

```bash
gh issue create --repo casehubio/casehub-blocks \
  --title "feat: CloudEvent↔Summarisation bridge module (casehub-blocks-cloudevents)" \
  --body "$(cat <<'EOF'
## Context

Design spec: casehubio/casehub-desiredstate#74 — Summarisation→RAS Integration

Depends on: summarisation-api extraction (sibling issue)

## Scope

New module `casehub-blocks-cloudevents` with two adapters:

- **CloudEventIngestionAdapter** — catches CloudEvents by type pattern, extracts payload, wraps as `LevelEvent<T>` with tenancyId, feeds into `EventStreamBus<T>`
- **CloudEventEmitter** — subscribes to output `EventStreamBus<T>`, wraps as CloudEvents with configurable type URI, propagates tenancyId, fires via CDI `Event<CloudEvent>.fireAsync()`
- **PipelineTickScheduler** — periodic tick driver for SummarisationRunner chains (Nyquist interval)

Dependencies: `casehub-blocks-summarisation-api`, `cloudevents-api`, `quarkus-arc`

Plain Java utilities, not CDI beans — instantiated by domain wiring code. ~30 lines of CDI setup per domain.

## Acceptance Criteria

- [ ] CloudEventIngestionAdapter: filters by type, extracts payload, propagates tenancyId
- [ ] CloudEventEmitter: wraps output, sets type URI + tenancyId extension, fires async
- [ ] PipelineTickScheduler: drives tick() on runners at configurable interval
- [ ] EventAccumulator partitions by tenancyId — tenant-homogeneous batches
- [ ] Integration test: CloudEvent → ingestion → summarisation → emission → CloudEvent
- [ ] Tenancyid propagation test: multi-tenant events never cross-contaminate

## Scale / Complexity

M / Med — new module with multi-tenant pipeline design
EOF
)"
```

- [ ] **Step 3: File YAML surface issue**

```bash
gh issue create --repo casehubio/casehub-blocks \
  --title "feat: summarisation YAML surface with composable runtime (casehub-blocks-summarisation-yaml)" \
  --body "$(cat <<'EOF'
## Context

Design spec: casehubio/casehub-desiredstate#74 — Summarisation→RAS Integration

Depends on: summarisation-api extraction, CloudEvent bridge module (sibling issues)

## Scope

New module `casehub-blocks-summarisation-yaml` with composable runtime:

**Tier 1 — YAML standalone:** Built-in summariser types, no Java required:
- `threshold-classify` — MVEL3 boolean predicates, 1:1 output cardinality
- `phase-detect` — state machine, emit-on-transition-only, per-tenant state, first-match-wins
- `count` — per-category counts within window
- `field-extract` — JQ document transformation
- `pass-through` — identity rebatching

**Tier 2 — YAML + Java:** `@SummariserTypeId` annotation + `SummariserRegistry` (Jandex discovery at build time). Custom Java Summarisers referenced by type string in YAML.

**YAML model:** Pipeline with sources, levels (window or keyed grouping), summariser references, CloudEvent output. Linear topology only — non-linear requires Tier 2.

**Build-time processing** (`casehub-blocks-summarisation-yaml-deployment`):
- Classpath scan for `META-INF/summarisation/*.yaml`
- `@SummariserTypeId` registry scan
- Validation: unknown types, state reachability, cross-level consistency, loop prevention
- CDI bean registration: PipelineCompiler + @Scheduled tick driver

**Expression contexts:** Documented per position (threshold-classify when, keyed keyExpression, keyed completionTest, phase-detect when).

Dependencies: `casehub-blocks-summarisation-api`, `casehub-blocks-cloudevents`, `casehub-platform-expression`, Jackson YAML

## Acceptance Criteria

- [ ] YAML pipeline parsed and compiled to SummarisationRunner chains
- [ ] All 5 built-in summariser types implemented with tests
- [ ] @SummariserTypeId discovery works (custom Java type referenced from YAML)
- [ ] Build-time validation catches: unknown types, unreachable states, cross-level mismatches, loops
- [ ] phase-detect: per-tenant state, emit-on-transition-only, first-match-wins
- [ ] Keyed grouping mode: keyExpression + completionTest + staleTimeout
- [ ] Standard CloudEvent type URIs for built-in output
- [ ] Stateful summariser tenant-keying contract enforced

## Scale / Complexity

L / High — new YAML surface with 5 built-in types, Quarkus build extension, expression engine integration
EOF
)"
```

- [ ] **Step 4: File logistics example issue**

```bash
gh issue create --repo casehubio/casehub-blocks \
  --title "feat: logistics hub example — YAML-first summarisation→RAS pipeline" \
  --body "$(cat <<'EOF'
## Context

Design spec: casehubio/casehub-desiredstate#74 — Summarisation→RAS Integration

Depends on: summarisation YAML surface (sibling issue)

## Scope

New example module `examples/logistics/` demonstrating the full summarisation→RAS pipeline:

**YAML-first approach:**
- Pipeline YAML with `threshold-classify` (L1→L2 anomaly classification) and `phase-detect` (L2→L3 phase transitions)
- RAS situation YAML with `expression-rules` Ganglia consuming L3 phase CloudEvents
- Java only for: event simulation, test assertions

**Domain model:**
- L1: Simulated package scan events (delay, misroute, capacity breach)
- L2: Classified anomalies (DELAY, MISROUTE, CAPACITY_BREACH) via threshold-classify
- L3: Hub phases (NORMAL, CONGESTION, RECOVERY) via phase-detect

**No desiredstate dependency.** Pure blocks + RAS example.

**Validation goal:** Proves Tier 1 YAML standalone is sufficient for a non-trivial multi-level pipeline. If Java escape hatches are needed beyond simulation/testing, that signals the built-in types need extension.

Dependencies: `casehub-blocks-summarisation-yaml`, `casehub-ras-api`

## Acceptance Criteria

- [ ] Pipeline declared in YAML — no Java summariser implementations
- [ ] RAS situations declared in YAML — expression-rules Ganglia
- [ ] L1→L2→L3 pipeline works end-to-end (simulated events → phase detection)
- [ ] Multi-tenant test: two tenants with independent phase state
- [ ] Integration test: congestion situation detected and case trigger fires

## Scale / Complexity

M / Med — example wiring existing infrastructure, YAML-first validation
EOF
)"
```

- [ ] **Step 5: Record the issue numbers**

Note the 4 issue numbers returned by `gh issue create`. These will be referenced in the ops issue and in the #74 closure comment.

### Task 2: File casehub-ops child issue

**Files:**
- Reference: design spec §6

**Interfaces:**
- Consumes: blocks issue numbers from Task 1
- Produces: 1 GitHub issue on casehubio/casehub-ops

- [ ] **Step 1: File ops enhancement issue**

```bash
gh issue create --repo casehubio/casehub-ops \
  --title "feat: summarisation-enhanced RAS detection for deployment topologies" \
  --body "$(cat <<'EOF'
## Context

Design spec: casehubio/casehub-desiredstate#74 — Summarisation→RAS Integration

Depends on: casehub-blocks summarisation YAML surface + CloudEvent bridge

## Scope

Add a summarisation layer between ReconciliationLoop CloudEvents and existing RAS Ganglia in ops deployment topologies. Instead of detecting raw L1 events (individual NODE_FAULTED), RAS detects summarised L2/L3 phases (east-region degraded, cluster recovery).

Uses the casehub-blocks CloudEvent bridge and YAML surface — no new framework code needed.

**Deliverable:** YAML pipeline + RAS situation definitions for deployment topology monitoring.

## Acceptance Criteria

- [ ] Summarisation pipeline declared in YAML consuming desiredstate CloudEvents
- [ ] L2 anomaly classification for deployment events (provision failure types)
- [ ] L3 regional/zone phase detection (healthy, degraded, recovering)
- [ ] RAS situation definitions consuming L3 phase events
- [ ] Integration test: deployment fault → summarisation → situation detection

## Scale / Complexity

M / Med — wiring existing infrastructure with deployment domain knowledge
EOF
)"
```

### Task 3: Close #74 with scope decision

**Files:**
- Modify: GitHub issue #74 on casehubio/casehub-desiredstate

**Interfaces:**
- Consumes: all issue numbers from Tasks 1-2
- Produces: closed issue #74

- [ ] **Step 1: Comment on #74 with scope decision and child issues**

```bash
gh issue comment 74 --repo casehubio/casehub-desiredstate \
  --body "$(cat <<'EOF'
## Scope Decision

Design spec: `specs/issue-74-topology-implementation/2026-09-02-summarisation-ras-integration-design.md`

**Integration pattern:** CloudEvent re-entry via CDI bus. Summarised L2/L3 events fire as new CloudEvents — RAS consumes them identically to raw L1 events. No new SPI needed.

**What lives where:**
- **casehub-blocks** — all reusable plumbing: summarisation API extraction, CloudEvent bridge, YAML surface with composable runtime (Tier 1 standalone / Tier 2 + Java)
- **casehub-ops** — deployment topology + summarisation enhancement (the genuine desiredstate use case)
- **casehub-desiredstate** — no code changes needed

**No desiredstate in the logistics example.** The logistics scenario is an event processing + situation detection problem, not a desired-state reconciliation problem. The genuine desiredstate + summarisation use case is ops deployment topologies.

**Child issues filed:**
- casehubio/casehub-blocks#[API_EXTRACTION] — summarisation API extraction
- casehubio/casehub-blocks#[BRIDGE] — CloudEvent bridge module
- casehubio/casehub-blocks#[YAML] — summarisation YAML surface
- casehubio/casehub-blocks#[EXAMPLE] — logistics hub example
- casehubio/casehub-ops#[OPS] — summarisation-enhanced RAS detection
EOF
)"
```

Replace `[API_EXTRACTION]`, `[BRIDGE]`, `[YAML]`, `[EXAMPLE]`, `[OPS]` with actual issue numbers from Tasks 1-2.

- [ ] **Step 2: Close #74**

```bash
gh issue close 74 --repo casehubio/casehub-desiredstate \
  --reason completed \
  --comment "Scope decision made, child issues filed. See design spec for details."
```

- [ ] **Step 3: Commit workspace artifacts**

```bash
git -C /Users/mdproctor/claude/casehub/slots/170/wsp-casehub-desiredstate add specs/issue-74-topology-implementation/ plans/
git -C /Users/mdproctor/claude/casehub/slots/170/wsp-casehub-desiredstate commit -m "feat(#74): scope decision — summarisation→RAS integration design Closes #74"
```

---

## References

- `specs/issue-74-topology-implementation/2026-09-02-summarisation-ras-integration-design.md` — design spec
- `specs/issue-74-topology-implementation/decisions.md` — 9 design decisions (D1-D9)
- `blocks/summarisation/` — existing summarisation framework types
- `ras-adapter/` — existing RAS Ganglia pattern
- GE-20260730-d761e5, GE-20260629-e8b16d, GE-20260817-ce1de5, GE-20260616-02d0a7 — garden entries consulted
- `parent/docs/platform/capability-ownership.md`, `boundary-rules.md`, `overview.md` — platform coherence docs
- casehubio/casehub-desiredstate#74 — focal issue
