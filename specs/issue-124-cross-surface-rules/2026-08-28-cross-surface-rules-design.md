# Cross-Surface Rule and Invariant Resolution — Design Spec

**Issue:** #124 — cross-surface rule and invariant resolution for YAML graphs
**Date:** 2026-08-28
**Status:** Draft

## 1. Summary

Standalone `@GraphRule` and `@GraphInvariant` classes with `graph={"*:*"}` or
namespace patterns should fire against YAML-declared graphs, not just
annotation-declared graphs. A new Quarkus build step bridges both declaration
surfaces through the existing `DesiredStateGraphBuildItem` infrastructure.

## 2. Background

The annotation processor discovers standalone `@GraphRule` classes (annotated at
the class level, not inside a `@DesiredState` interface) and matches them against
annotation-declared graphs using `GraphPatternMatcher`. The YAML processor
discovers YAML graph files and produces `DesiredStateGraphBuildItem` entries. Both
surfaces produce build items with `namespace:name` keys, but standalone rules are
only matched against annotation graphs — YAML graphs are invisible to them.

## 3. Architecture

### 3.1 New Build Items

**`StandaloneRuleBuildItem`** — produced by the annotation processor, carries:
- `List<ResolvedRule>` — the standalone rules (all three variants: Imperative,
  ParameterizedReflective, DeclarativeRule)
- `String[] graphPatterns` — the `graph={}` attribute from `@GraphRule`

**`StandaloneInvariantBuildItem`** — same shape:
- `List<ResolvedInvariant>` — the standalone invariants
- `String[] graphPatterns` — the `graph={}` attribute from `@GraphInvariant`

### 3.2 CrossSurfaceRuleResolutionStep

A new `@BuildStep` method in its own processor class. Runs after both
`DesiredStateAnnotationsProcessor` and `YamlDesiredStateProcessor` (declared via
`@Consume(DesiredStateGraphBuildItem.class)` or explicit ordering).

**Consumes:**
- `List<DesiredStateGraphBuildItem>` — all graphs from both surfaces
- `List<StandaloneRuleBuildItem>` — standalone rules from annotation processor
- `List<StandaloneInvariantBuildItem>` — standalone invariants from annotation processor

**Produces:**
- `AdditionalRulesBuildItem` — per YAML graph, carries matched rules and invariants

**Logic:**
1. Filter `DesiredStateGraphBuildItem` to YAML-sourced entries (`source.startsWith("yaml:")`)
2. For each YAML graph, check each standalone rule/invariant's `graphPatterns` against
   the graph's `qualifiedName()` via `GraphPatternMatcher.matches()`
3. Collect matched rules and invariants
4. Produce `AdditionalRulesBuildItem(namespace, name, matchedRules, matchedInvariants)`

### 3.3 AdditionalRulesBuildItem

Carries cross-surface rules/invariants matched to a specific YAML graph:
- `String namespace` — the YAML graph's namespace
- `String name` — the YAML graph's name
- `List<ResolvedRule> rules` — matched standalone rules
- `List<ResolvedInvariant> invariants` — matched standalone invariants

### 3.4 YAML Recorder Integration

`YamlDesiredStateProcessor.discoverYamlGraphs()` gains a new parameter:
`List<AdditionalRulesBuildItem> additionalRules`. For each YAML graph, it finds
the matching `AdditionalRulesBuildItem` (by namespace + name) and passes the
rules/invariants to `YamlGraphRecorder.createYamlGoalCompiler()`.

The GoalCompiler merges them:
- YAML-declared rules first, then cross-surface annotation rules
- YAML-declared invariants first, then cross-surface annotation invariants

This ordering matches the annotation processor's convention where graph-local
rules precede standalone rules.

### 3.5 Annotation Processor Changes

The annotation processor currently handles standalone rules internally
(lines 110-122 in `DesiredStateAnnotationsProcessor`). It needs to:

1. **Export** standalone rules/invariants as `StandaloneRuleBuildItem` /
   `StandaloneInvariantBuildItem` so the cross-surface step can consume them
2. **Continue** to merge standalone rules into annotation graphs as before —
   the export is additive, not a replacement

## 4. Rule Ordering

When YAML declarative rules, module-promoted rules, and cross-surface annotation
rules are combined in the GoalCompiler:

1. YAML-declared rules (from the graph's `rules:` section)
2. Module-promoted rules (from imported modules)
3. Cross-surface annotation rules (from `AdditionalRulesBuildItem`)

The `GraphRuleEngine` fixed-point loop ensures convergence is independent of
evaluation order for well-formed (monotonic, non-conflicting) rules. Ordering
affects convergence speed only.

## 5. Testing

- Unit test: `CrossSurfaceRuleResolutionStep` with mock build items
- Integration test: a standalone `@GraphRule` class in the pipeline-yaml example
  that fires against the YAML-declared medallion pipeline graph
- Integration test: a standalone `@GraphInvariant` class that validates a
  structural property across YAML graphs

## References

- `specs/issue-116-yaml-language-design/2026-08-27-yaml-language-extensions-design.md` — §8.4
- `annotations/deployment/.../DesiredStateAnnotationsProcessor.java:554-578` — standalone rule scan
- `annotations/deployment/.../DesiredStateAnnotationsProcessor.java:110-122` — standalone rule matching loop
- `annotations/deployment/.../DesiredStateGraphBuildItem.java` — build item with namespace:name
- `annotations/runtime/.../GraphPatternMatcher.java` — surface-agnostic pattern matching
- `yaml/deployment/.../YamlDesiredStateProcessor.java:114` — YAML graph build item production
- `yaml/runtime/.../YamlGraphRecorder.java` — GoalCompiler rule/invariant merging
- #124 — cross-surface rule and invariant resolution
