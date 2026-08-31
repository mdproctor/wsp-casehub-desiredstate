## D1: Cardinality attribute placement

**Choice:** Add `minCount`/`maxCount` directly to each pattern annotation (`@Match`, `@DirectDep`, `@Reaches`) and to `PatternParameterDescriptor`
**Alternatives:**
- Separate `@Count` annotation on the same parameter — clean separation but noisy (two annotations per param), doesn't map naturally to YAML
- Method-level `@Cardinality(param = "x", min = N)` — indirection via string param name, fragile to refactoring, doesn't work for YAML at all
**Rationale:** Cardinality is a property of the pattern match, not an orthogonal concern. "Find nodes of type X with at least N matches" is one thought, not two. Keeps the model flat, maps cleanly to YAML `{ type: sink, minCount: 2 }`, extends `PatternParameterDescriptor` with two fields.
**Trade-offs:** Java annotation attributes require primitive defaults — need `-1` sentinel for "not specified" since `Integer` is not allowed.
**Sources:** `@Match` annotation, `PatternParameterDescriptor` record, `YamlPattern` record, `GraphInvariantEngine.buildExpectedAnchors()`
**Exploration:** quick
**Status:** captured

## D2a: Defaults

**Choice:** `@Match` default `minCount=0` (vacuous truth — universal quantification). `@DirectDep`/`@Reaches` default `minCount=1` (relationship must exist). `maxCount` unlimited everywhere. These are the clean semantics regardless of backward compatibility (pre-release).
**Alternatives:** None considered — these are the mathematically correct defaults for each pattern kind.
**Rationale:** Universal quantification ("for all X") is vacuously true with no X. Relationship patterns ("X depends on Y") are meaningless if Y need not exist.
**Trade-offs:** None.
**Sources:** `GraphInvariantEngine.buildExpectedAnchors()`, issue #115 spec D3 (vacuous truth)
**Exploration:** quick
**Status:** captured

## D2: Match-level cardinality semantics

**Choice:** Count-only assertion — when `@Match` has `minCount` or `maxCount`, the invariant is a pure count check. No per-anchor expansion, no method body invocation. Separate invariants for count vs structural checks.
**Alternatives:**
- Cardinality as pre-check before normal evaluation — allows combining count and structure in one invariant but conflates two concerns, creates dual-mode methods, and produces ambiguous violation reporting
**Rationale:** Two separate invariants are clearer than one doing double duty. Count invariants assert "enough of X exist." Structural invariants assert "each X has property Y." When `minCount=0` (default) and no `maxCount`, the invariant remains a normal per-anchor structural check — cardinality mode activates only when explicitly specified.
**Trade-offs:** "At least 3 compute instances, each with a monitor" requires two invariants instead of one. This is a feature — each invariant has a single, clear violation mode.
**Depends on:** D1 (cardinality on pattern annotations)
**Sources:** `GraphInvariantEngine.buildExpectedAnchors()`, `GraphInvariantEngine.validateParameterized()`
**Exploration:** quick
**Status:** captured

## D3: Expansion-level cardinality semantics

**Choice:** `minCount`/`maxCount` on `@DirectDep`/`@Reaches` count expansions per anchor. Count check is independent of method body invocation — cardinality violation if count is outside bounds, method body still invoked per expansion for value checks.
**Alternatives:** None — this is the natural generalization of the existing boolean existence check.
**Rationale:** Today zero expansions = structural violation (implicit `minCount=1`). Explicit `minCount`/`maxCount` generalizes this. `maxCount` enables "at most N" constraints (e.g., "each service must have exactly one database" via `minCount=1, maxCount=1`).
**Trade-offs:** None. Symmetric with match-level cardinality.
**Depends on:** D1 (cardinality on pattern annotations)
**Sources:** `GraphInvariantEngine.validateParameterized()`, `PatternEvaluator.expandChain()`
**Exploration:** quick
**Status:** captured
