# Cross-Surface Rule Resolution — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #124 — cross-surface rule and invariant resolution for YAML graphs
**Issue group:** #124, #121, #122

**Goal:** Standalone `@GraphRule` and `@GraphInvariant` classes fire against
YAML-declared graphs, not just annotation-declared graphs.

**Architecture:** The annotation processor exports standalone rules/invariants
via new build items. A new `CrossSurfaceRuleResolutionStep` matches them
against YAML graphs using `GraphPatternMatcher`. The YAML recorder merges
matched rules/invariants into its GoalCompiler alongside YAML-declared ones.

**Tech Stack:** Java 21, Quarkus 3.x (build extensions), Jandex, Maven

**Design spec:** `specs/issue-124-cross-surface-rules/2026-08-28-cross-surface-rules-design.md`

## Global Constraints

- Rule ordering: YAML-declared → module-promoted → cross-surface annotation (spec §4)
- `GraphPatternMatcher.matches()` is surface-agnostic — no changes needed
- Build step ordering: annotation processor → YAML processor → cross-surface step
- All build items are `MultiBuildItem` (multiple instances per build)

---

## Batch 1: Build Items + Cross-Surface Step + Integration

This is a single-batch plan — the feature is small and all parts must
exist together to be testable.

### Task 1: Standalone build items + export from annotation processor

**Files:**
- Create: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/StandaloneRuleBuildItem.java`
- Create: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/StandaloneInvariantBuildItem.java`
- Modify: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java:107-108`

**Interfaces:**
- Consumes: `GraphRuleDescriptor`, `GraphInvariantDescriptor`, existing `scanStandaloneGraphRules()` / `scanStandaloneGraphInvariants()` methods
- Produces: `StandaloneRuleBuildItem(String[] graphPatterns, List<GraphRuleDescriptor> rules)`,
  `StandaloneInvariantBuildItem(String[] graphPatterns, List<GraphInvariantDescriptor> invariants)`.
  Annotation processor produces one build item per standalone class.

- [ ] **Step 1: Create StandaloneRuleBuildItem**

```java
package io.casehub.desiredstate.annotations.deployment;

import io.casehub.desiredstate.annotations.runtime.GraphRuleDescriptor;
import io.quarkus.builder.item.MultiBuildItem;

import java.util.List;

public final class StandaloneRuleBuildItem extends MultiBuildItem {
    private final String[] graphPatterns;
    private final List<GraphRuleDescriptor> rules;

    public StandaloneRuleBuildItem(String[] graphPatterns, List<GraphRuleDescriptor> rules) {
        this.graphPatterns = graphPatterns;
        this.rules = rules;
    }

    public String[] graphPatterns() { return graphPatterns; }
    public List<GraphRuleDescriptor> rules() { return rules; }
}
```

- [ ] **Step 2: Create StandaloneInvariantBuildItem**

```java
package io.casehub.desiredstate.annotations.deployment;

import io.casehub.desiredstate.annotations.runtime.GraphInvariantDescriptor;
import io.quarkus.builder.item.MultiBuildItem;

import java.util.List;

public final class StandaloneInvariantBuildItem extends MultiBuildItem {
    private final String[] graphPatterns;
    private final List<GraphInvariantDescriptor> invariants;

    public StandaloneInvariantBuildItem(String[] graphPatterns,
                                        List<GraphInvariantDescriptor> invariants) {
        this.graphPatterns = graphPatterns;
        this.invariants = invariants;
    }

    public String[] graphPatterns() { return graphPatterns; }
    public List<GraphInvariantDescriptor> invariants() { return invariants; }
}
```

- [ ] **Step 3: Export standalone rules/invariants from annotation processor**

In `DesiredStateAnnotationsProcessor.discoverDesiredStates()`, add a
`BuildProducer<StandaloneRuleBuildItem>` and `BuildProducer<StandaloneInvariantBuildItem>`
parameter. After the existing `standaloneRules` / `standaloneInvariants` are scanned
(lines 107-108), produce build items:

```java
// After line 108, before the @DesiredState loop:
for (var srEntry : standaloneRules) {
    standaloneRuleItems.produce(new StandaloneRuleBuildItem(
            srEntry.getKey(), srEntry.getValue()));
}
for (var siEntry : standaloneInvariants) {
    standaloneInvariantItems.produce(new StandaloneInvariantBuildItem(
            siEntry.getKey(), siEntry.getValue()));
}
```

The existing loop at lines 117-128 continues to merge standalone rules into
annotation graphs — this export is additive.

- [ ] **Step 4: Verify annotation module compiles**

Run: `mvn --batch-mode test -pl annotations/deployment`
Expected: ALL PASS (build items are passive — they don't affect existing behaviour)

- [ ] **Step 5: Commit**

```
feat(#124): standalone rule/invariant build items

StandaloneRuleBuildItem and StandaloneInvariantBuildItem carry
standalone @GraphRule and @GraphInvariant descriptors for
cross-surface consumption. Annotation processor exports them
alongside existing in-surface merging.

Refs #124
```

---

### Task 2: CrossSurfaceRuleResolutionStep + AdditionalRulesBuildItem

**Files:**
- Create: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AdditionalRulesBuildItem.java`
- Create: `annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/CrossSurfaceRuleResolutionStep.java`
- Create: `annotations/deployment/src/test/java/io/casehub/desiredstate/annotations/deployment/CrossSurfaceRuleResolutionStepTest.java`

**Interfaces:**
- Consumes: `List<DesiredStateGraphBuildItem>`, `List<StandaloneRuleBuildItem>`,
  `List<StandaloneInvariantBuildItem>`, `GraphPatternMatcher.matches()`
- Produces: `AdditionalRulesBuildItem(String namespace, String name,
  List<GraphRuleDescriptor> rules, List<GraphInvariantDescriptor> invariants)`.
  One per YAML graph with matched standalone rules/invariants.

- [ ] **Step 1: Create AdditionalRulesBuildItem**

```java
package io.casehub.desiredstate.annotations.deployment;

import io.casehub.desiredstate.annotations.runtime.GraphInvariantDescriptor;
import io.casehub.desiredstate.annotations.runtime.GraphRuleDescriptor;
import io.quarkus.builder.item.MultiBuildItem;

import java.util.List;

public final class AdditionalRulesBuildItem extends MultiBuildItem {
    private final String namespace;
    private final String name;
    private final List<GraphRuleDescriptor> rules;
    private final List<GraphInvariantDescriptor> invariants;

    public AdditionalRulesBuildItem(String namespace, String name,
                                    List<GraphRuleDescriptor> rules,
                                    List<GraphInvariantDescriptor> invariants) {
        this.namespace = namespace;
        this.name = name;
        this.rules = rules;
        this.invariants = invariants;
    }

    public String namespace() { return namespace; }
    public String name() { return name; }
    public List<GraphRuleDescriptor> rules() { return rules; }
    public List<GraphInvariantDescriptor> invariants() { return invariants; }
    public String qualifiedName() { return namespace + ":" + name; }
}
```

- [ ] **Step 2: Write CrossSurfaceRuleResolutionStep test**

```java
package io.casehub.desiredstate.annotations.deployment;

import io.casehub.desiredstate.annotations.runtime.GraphInvariantDescriptor;
import io.casehub.desiredstate.annotations.runtime.GraphRuleDescriptor;
import io.casehub.desiredstate.annotations.runtime.PatternKind;
import io.casehub.desiredstate.annotations.runtime.PatternParameterDescriptor;
import io.casehub.desiredstate.annotations.runtime.Direction;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class CrossSurfaceRuleResolutionStepTest {

    @Test
    void matchesStandaloneRuleToYamlGraph() {
        var graphs = List.of(
                new DesiredStateGraphBuildItem("pipeline", "medallion", "yaml:medallion.yaml"),
                new DesiredStateGraphBuildItem("tutorial", "store", "annotation:Store"));

        var ruleDesc = new GraphRuleDescriptor.ReflectiveRuleDescriptor(
                "ensureMonitoring", false,
                List.of(new PatternParameterDescriptor(
                        PatternKind.MATCH, "sink", "", Direction.DEPENDENCIES)),
                "com.example.MonitorRule");

        var standaloneRules = List.of(
                new StandaloneRuleBuildItem(new String[]{"*:*"}, List.of(ruleDesc)));

        List<AdditionalRulesBuildItem> results = new ArrayList<>();
        CrossSurfaceRuleResolutionStep.resolve(graphs, standaloneRules, List.of(), results);

        // Should match only YAML graph, not annotation graph
        assertThat(results).hasSize(1);
        assertThat(results.get(0).qualifiedName()).isEqualTo("pipeline:medallion");
        assertThat(results.get(0).rules()).hasSize(1);
    }

    @Test
    void namespacePatternFiltersCorrectly() {
        var graphs = List.of(
                new DesiredStateGraphBuildItem("pipeline", "medallion", "yaml:medallion.yaml"),
                new DesiredStateGraphBuildItem("tutorial", "store", "yaml:store.yaml"));

        var ruleDesc = new GraphRuleDescriptor.ReflectiveRuleDescriptor(
                "pipelineOnly", false, List.of(), "com.example.PipelineRule");

        var standaloneRules = List.of(
                new StandaloneRuleBuildItem(new String[]{"pipeline:*"}, List.of(ruleDesc)));

        List<AdditionalRulesBuildItem> results = new ArrayList<>();
        CrossSurfaceRuleResolutionStep.resolve(graphs, standaloneRules, List.of(), results);

        assertThat(results).hasSize(1);
        assertThat(results.get(0).qualifiedName()).isEqualTo("pipeline:medallion");
    }

    @Test
    void noMatchProducesNoItems() {
        var graphs = List.of(
                new DesiredStateGraphBuildItem("tutorial", "store", "yaml:store.yaml"));

        var ruleDesc = new GraphRuleDescriptor.ReflectiveRuleDescriptor(
                "pipelineOnly", false, List.of(), "com.example.PipelineRule");

        var standaloneRules = List.of(
                new StandaloneRuleBuildItem(new String[]{"pipeline:*"}, List.of(ruleDesc)));

        List<AdditionalRulesBuildItem> results = new ArrayList<>();
        CrossSurfaceRuleResolutionStep.resolve(graphs, standaloneRules, List.of(), results);

        assertThat(results).isEmpty();
    }

    @Test
    void invariantsMatchedAlongside() {
        var graphs = List.of(
                new DesiredStateGraphBuildItem("pipeline", "medallion", "yaml:medallion.yaml"));

        var invDesc = new GraphInvariantDescriptor.ReflectiveInvariantDescriptor(
                "sinkHasUpstream", false, List.of(), "com.example.SinkInvariant");

        var standaloneInvariants = List.of(
                new StandaloneInvariantBuildItem(new String[]{"*:*"}, List.of(invDesc)));

        List<AdditionalRulesBuildItem> results = new ArrayList<>();
        CrossSurfaceRuleResolutionStep.resolve(graphs, List.of(), standaloneInvariants, results);

        assertThat(results).hasSize(1);
        assertThat(results.get(0).invariants()).hasSize(1);
        assertThat(results.get(0).rules()).isEmpty();
    }
}
```

- [ ] **Step 3: Implement CrossSurfaceRuleResolutionStep**

```java
package io.casehub.desiredstate.annotations.deployment;

import io.casehub.desiredstate.annotations.runtime.GraphInvariantDescriptor;
import io.casehub.desiredstate.annotations.runtime.GraphPatternMatcher;
import io.casehub.desiredstate.annotations.runtime.GraphRuleDescriptor;
import io.quarkus.deployment.annotations.BuildProducer;
import io.quarkus.deployment.annotations.BuildStep;

import java.util.ArrayList;
import java.util.List;

public class CrossSurfaceRuleResolutionStep {

    @BuildStep
    void resolveRulesAcrossSurfaces(
            List<DesiredStateGraphBuildItem> graphs,
            List<StandaloneRuleBuildItem> standaloneRules,
            List<StandaloneInvariantBuildItem> standaloneInvariants,
            BuildProducer<AdditionalRulesBuildItem> additionalRules) {

        List<AdditionalRulesBuildItem> results = new ArrayList<>();
        resolve(graphs, standaloneRules, standaloneInvariants, results);
        results.forEach(additionalRules::produce);
    }

    static void resolve(List<DesiredStateGraphBuildItem> graphs,
                        List<StandaloneRuleBuildItem> standaloneRules,
                        List<StandaloneInvariantBuildItem> standaloneInvariants,
                        List<AdditionalRulesBuildItem> results) {
        for (DesiredStateGraphBuildItem graph : graphs) {
            if (!graph.source().startsWith("yaml:")) {continue;}

            String graphKey = graph.qualifiedName();
            List<GraphRuleDescriptor> matchedRules = new ArrayList<>();
            List<GraphInvariantDescriptor> matchedInvariants = new ArrayList<>();

            for (StandaloneRuleBuildItem sr : standaloneRules) {
                if (GraphPatternMatcher.matches(sr.graphPatterns(), graphKey)) {
                    matchedRules.addAll(sr.rules());
                }
            }
            for (StandaloneInvariantBuildItem si : standaloneInvariants) {
                if (GraphPatternMatcher.matches(si.graphPatterns(), graphKey)) {
                    matchedInvariants.addAll(si.invariants());
                }
            }

            if (!matchedRules.isEmpty() || !matchedInvariants.isEmpty()) {
                results.add(new AdditionalRulesBuildItem(
                        graph.namespace(), graph.name(),
                        matchedRules, matchedInvariants));
            }
        }
    }
}
```

- [ ] **Step 4: Run tests**

Run: `mvn --batch-mode test -pl annotations/deployment -Dtest=CrossSurfaceRuleResolutionStepTest`
Expected: ALL PASS

- [ ] **Step 5: Run all annotation/deployment tests**

Run: `mvn --batch-mode test -pl annotations/deployment`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```
feat(#124): CrossSurfaceRuleResolutionStep with pattern matching

Matches standalone @GraphRule and @GraphInvariant classes against
YAML graphs via GraphPatternMatcher. Produces AdditionalRulesBuildItem
per YAML graph with matched rules/invariants.

Refs #124
```

---

### Task 3: YAML recorder integration + pipeline-yaml integration test

**Files:**
- Modify: `yaml/deployment/src/main/java/io/casehub/desiredstate/yaml/deployment/YamlDesiredStateProcessor.java`
- Modify: `yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlGraphRecorder.java`
- Create: `examples/pipeline-yaml/src/main/java/io/casehub/desiredstate/example/pipeline/yaml/StandaloneMonitorRule.java`
- Modify: `examples/pipeline-yaml/src/test/java/io/casehub/desiredstate/example/pipeline/yaml/PipelineYamlTest.java`

**Interfaces:**
- Consumes: `List<AdditionalRulesBuildItem>`, existing GoalCompiler rule merging
- Produces: YAML GoalCompiler applies cross-surface rules after YAML-declared
  and module-promoted rules. Integration test proves a standalone annotation
  rule fires against a YAML graph.

- [ ] **Step 1: Wire AdditionalRulesBuildItem into YamlDesiredStateProcessor**

Add `List<AdditionalRulesBuildItem>` parameter to `discoverYamlGraphs()`.
For each YAML graph, find matching additional rules and pass them to the recorder:

```java
// After the existing compiler creation, find additional rules for this graph
List<GraphRuleDescriptor> crossSurfaceRules = List.of();
List<GraphInvariantDescriptor> crossSurfaceInvariants = List.of();
for (AdditionalRulesBuildItem additional : additionalRuleItems) {
    if (additional.namespace().equals(ns) && additional.name().equals(name)) {
        crossSurfaceRules = additional.rules();
        crossSurfaceInvariants = additional.invariants();
        break;
    }
}
```

Pass these to the recorder method (new parameter).

- [ ] **Step 2: Update YamlGraphRecorder to accept cross-surface descriptors**

Add parameters to `createYamlGoalCompiler` for cross-surface rule/invariant
descriptors. In the GoalCompiler lambda, after YAML-declared + module-promoted
rules, resolve cross-surface descriptors into `ResolvedRule` instances via the
annotation recorder's existing resolution logic, then merge them:

```java
// After effectiveRules are applied...
// Cross-surface annotation rules (from standalone @GraphRule classes)
if (crossSurfaceRuleDescriptors != null && !crossSurfaceRuleDescriptors.isEmpty()) {
    // These are already ResolvedRule from the annotation processor
    // Merge after YAML rules
    List<ResolvedRule> allRules = new ArrayList<>(resolvedRules);
    allRules.addAll(crossSurfaceResolvedRules);
    graph = new GraphRuleEngine().evaluate(graph, allRules);
}
```

Note: The cross-surface rules are `GraphRuleDescriptor` (build-time descriptors),
not `ResolvedRule` (runtime). The recorder needs to resolve them at runtime init.
This is the same resolution the annotation `DesiredStateGraphRecorder` does.
The simplest approach: pass the resolved rules directly. The annotation processor
can resolve standalone rules at build time and carry `ResolvedRule` in the build
item. Alternatively, carry descriptors and resolve in the YAML recorder.

**Recommended:** Pass `List<ResolvedRule>` and `List<ResolvedInvariant>` in
`AdditionalRulesBuildItem` instead of descriptors. The annotation processor
already resolves them. Update `AdditionalRulesBuildItem` fields accordingly,
and update `CrossSurfaceRuleResolutionStep` to resolve descriptors via
`DesiredStateGraphRecorder` before producing the build item.

**Simpler alternative:** Since the cross-surface step runs at build time and
`ResolvedRule` contains runtime objects (Method references), pass the
descriptors and resolve them in the YAML recorder at runtime init. The YAML
recorder already has the pattern for resolving `GraphRuleDescriptor` →
`ResolvedRule` from the annotation recorder.

- [ ] **Step 3: Create standalone rule for integration test**

Create `StandaloneMonitorRule.java` in the pipeline-yaml example:

```java
package io.casehub.desiredstate.example.pipeline.yaml;

import io.casehub.desiredstate.annotations.GraphRule;
import io.casehub.desiredstate.annotations.Match;
import io.casehub.desiredstate.annotations.NotExists;
import io.casehub.desiredstate.annotations.runtime.Direction;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.GraphMutation;
import io.casehub.desiredstate.api.GraphMutations;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.example.pipeline.MonitorSpec;

import java.util.List;

@GraphRule(graph = {"pipeline-yaml:*"})
public class StandaloneMonitorRule {

    @GraphRule
    public static List<GraphMutation> ensureMonitoring(
            @Match(type = "sink") DesiredNode sink,
            @NotExists(type = "monitor", of = "sink",
                    direction = Direction.DEPENDENTS) Void guard) {
        return GraphMutations.addNodeDependingOn(
                new DesiredNode(NodeId.of("xsurface-monitor-" + sink.id().value()),
                        new MonitorSpec(sink.id().value()), HumanGating.NONE),
                sink.id());
    }
}
```

This rule has `graph = {"pipeline-yaml:*"}` — it matches YAML graphs in the
`pipeline-yaml` namespace. If cross-surface resolution works, the rule fires
and adds `xsurface-monitor-warehouse-sink` to the YAML graph.

- [ ] **Step 4: Write integration test**

Add to `PipelineYamlTest.java`:

```java
@Test
void crossSurface_standaloneRuleFiresAgainstYamlGraph() {
    DesiredStateGraph graph = compileSingleGraph();
    // The standalone @GraphRule with graph={"pipeline-yaml:*"} should fire
    assertThat(graph.nodes()).containsKey(NodeId.of("xsurface-monitor-warehouse-sink"));
}
```

- [ ] **Step 5: Run integration test**

Run: `mvn --batch-mode test -pl examples/pipeline-yaml`
Expected: ALL PASS

- [ ] **Step 6: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```
feat(#124): YAML recorder consumes cross-surface rules

YamlDesiredStateProcessor passes AdditionalRulesBuildItem to the
GoalCompiler. Cross-surface annotation rules merge after YAML-declared
and module-promoted rules. Integration test: standalone @GraphRule
fires against pipeline-yaml medallion graph.

Closes #124
```

---

## Summary

| Batch | Tasks | What's working after |
|-------|-------|---------------------|
| 1: Build Items + Step + Integration | 1-3 | Standalone @GraphRule and @GraphInvariant fire against YAML graphs |

**Total:** 1 batch, 3 tasks

**What #124 delivers:** A standalone Java class with `@GraphRule(graph={"*:*"})`
or `@GraphRule(graph={"pipeline:*"})` fires against YAML-declared graphs. Same
for `@GraphInvariant`. The existing annotation-surface behaviour is unchanged.

## References

- `specs/issue-124-cross-surface-rules/2026-08-28-cross-surface-rules-design.md` — design spec
- `specs/issue-116-yaml-language-design/2026-08-27-yaml-language-extensions-design.md` — §8.4
- `annotations/deployment/.../DesiredStateAnnotationsProcessor.java:107-108` — standalone scan
- `annotations/deployment/.../DesiredStateAnnotationsProcessor.java:117-128` — standalone merge loop
- `annotations/deployment/.../DesiredStateGraphBuildItem.java` — namespace:name carrier
- `annotations/runtime/.../GraphPatternMatcher.java` — pattern matching
- `yaml/deployment/.../YamlDesiredStateProcessor.java:114` — YAML graph production
- `yaml/runtime/.../YamlGraphRecorder.java` — GoalCompiler rule merging
- #124 — cross-surface rule and invariant resolution
