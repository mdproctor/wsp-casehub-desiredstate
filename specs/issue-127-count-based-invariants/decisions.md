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
