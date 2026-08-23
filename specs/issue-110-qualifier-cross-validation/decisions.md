# Design Decisions — #110/#111 qualifier + cross-model validation

## D1: Two-pass conditional @Default on GoalCompiler beans

**Choice:** Restructure `generateDesiredStateGraphs()` into a collect-then-register pattern. Count all distinct graph keys first. If exactly one graph: add both `@DesiredStateQualifier(namespace, name)` and `@Default`. If >1 graph: add only `@DesiredStateQualifier(namespace, name)` — no `@Default`.
**Alternatives:**
- Always both qualifiers — simpler (no two-pass), but multi-graph unqualified injection gives `AmbiguousResolutionException` instead of the clearer `UnsatisfiedResolutionException`
- Always qualifier only — clean and uniform but breaks every existing single-graph injection point (`@Inject GoalCompiler`) for no practical benefit
**Rationale:** Preserves zero-qualifier single-graph experience. Multi-graph apps must explicitly qualify each injection point — `UnsatisfiedResolutionException` is a clearer error than `AmbiguousResolutionException`. The qualifier is always present on every bean, so apps can opt into qualified injection at any time. The two-pass restructure is minimal — collect descriptors into a list, then iterate to register.
**Trade-offs:** Processor restructuring from single-pass to collect-then-register. Adding a second graph to an app changes injection semantics (existing unqualified injection points stop resolving). This is intentional — the developer must choose which graph to inject.
**Prerequisites:** None
**Sources:** DesiredStateAnnotationsProcessor.java:177-187, DesiredStateQualifier.java, CDI 4.1 spec (qualifier semantics)
**Exploration:** quick
**Status:** captured
