# Decisions — Cross-Surface Rule Resolution (#124)

## D1: Scope includes both rules and invariants

**Choice:** The cross-surface build step handles both standalone `@GraphRule` and standalone `@GraphInvariant` classes.
**Alternatives:**
- Rules only — smaller scope, but invariants use the identical mechanism and leaving them out creates an inconsistency
- Rules + invariants + fault policies — broader, but fault policies are `@ApplicationScoped` beans (not graph-scoped), so cross-surface routing doesn't apply
**Rationale:** Rules and invariants share the same `GraphPatternMatcher` scoping and the same `DesiredStateGraphBuildItem` infrastructure. One build step handles both with no additional complexity. Fault policies are CDI beans scoped by `nodeTypes`/`faultTypes`, not by graph namespace — they already fire cross-surface via CDI discovery.
**Trade-offs:** None significant — this is the natural scope.
**Sources:** Design spec §8.4, `DesiredStateAnnotationsProcessor.java:554-578` (standalone rule scan), `GraphPatternMatcher.java` (surface-agnostic matching)
**Exploration:** quick
**Status:** captured
