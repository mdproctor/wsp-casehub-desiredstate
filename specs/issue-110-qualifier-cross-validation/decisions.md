# Design Decisions — #110/#111 qualifier + cross-model validation

## D1: Always @Default + @DesiredStateQualifier on every GoalCompiler bean

**Choice:** Always register both `@Default` and `@DesiredStateQualifier(namespace, name)` on every GoalCompiler synthetic bean. Single-graph apps work unqualified via `@Default`. Multi-graph apps with unqualified injection get ArC's `AmbiguousResolutionException` (which lists both beans with their qualifier values, telling the developer exactly how to fix it). No conditional logic, no two-pass restructure.
**Alternatives:**
- Conditional @Default (original D1) — single-graph gets @Default, multi-graph doesn't. Creates non-local coupling: adding a second graph in a different Maven module silently changes the CDI shape of the existing bean, breaking unqualified injection points elsewhere. `UnsatisfiedResolutionException` ("no bean found") is confusing when you just declared one.
- Always qualifier only — consistent but breaks every existing `@Inject GoalCompiler` for no practical benefit. Verbosity tax on single-graph apps.
**Rationale:** Eliminates non-local coupling — every bean always has the same CDI shape regardless of graph count. `AmbiguousResolutionException` is more informative than `UnsatisfiedResolutionException` for this scenario: "multiple GoalCompiler beans found: [ns1:name1, ns2:name2]" directly tells the developer to disambiguate, while "no bean found" is misleading. No processor restructuring needed — single-pass, same qualifiers always. Both are build-time errors in Quarkus ArC. Decision review (R1-02, R1-03) identified the non-local coupling flaw in the conditional approach.
**Trade-offs:** Multi-graph unqualified injection gives `AmbiguousResolutionException` instead of a domain-specific error. ArC's error message includes the qualifier values, which is sufficient for pre-release. Custom validation for richer error messages can be added later.
**Prerequisites:** None
**Depends on:** None
**Sources:** DesiredStateAnnotationsProcessor.java:177-187, DesiredStateQualifier.java, CDI 4.1 spec §2.3.5 (qualifier matching), decision review R1-02 (non-local coupling), R1-03 (unconsidered alternative)
**Exploration:** quick → revised after decision review (R1-02, R1-03)
**Status:** revised
