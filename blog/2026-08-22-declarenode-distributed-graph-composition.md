---
title: "@DeclareNode — distributed graph composition via class-based annotations"
date: 2026-08-22
entry_type: note
subtype: diary
series: issue-105-class-based-desirednode
projects: casehub-desiredstate
tags: [annotations, quarkus-extension, graph-composition, cdi]
---

# @DeclareNode — distributed graph composition via class-based annotations

The desiredstate annotation model from #102 gives you `@DesiredState` on an interface with `@Node` methods — your entire graph declared in one file. That works for single-module graphs. It doesn't work when the graph is assembled from nodes scattered across multiple modules — base infrastructure in one JAR, tenant extensions in another, each discovered at build time via Jandex scanning. CDI bean discovery, applied to desired-state graphs.

That's what `@DeclareNode` is. A class implements `NodeSpec`, carries `@DeclareNode(namespace, name, id)`, and the build extension discovers it, groups it by graph name, and assembles it alongside any interface-declared nodes with matching coordinates. Two models feeding the same pipeline: interface for centralized graphs, class for distributed composition.

We renamed the annotation from `@DesiredNode` (the issue's original name) to `@DeclareNode` during design review. The API record `io.casehub.desiredstate.api.DesiredNode` would have collided — any test asserting on graph nodes AND annotating inner classes would need fully-qualified imports for one of them. `@DeclareNode` avoids it cleanly.

The foundation change was refactoring `NodeDescriptor` from a flat record to a sealed interface: `InterfaceNode | ClassNode`. Pattern matching in the recorder handles both exhaustively. This was the right prerequisite — it makes the merge path clean and gives compile-time guarantees that every node form is handled.

`@DependsOn` gained a `nodes` attribute — `Class<? extends NodeSpec>[]` alongside the existing `String[] value()`. Type-safe, compile-time checked. The processor resolves class references to node IDs at build time by looking up the target's `@DeclareNode(id)`. Cross-model string references still work for interface-to-class and class-to-interface edges.

The CDI qualifier story surfaced a genuine gotcha: adding `@DesiredStateQualifier` to `SyntheticBeanBuildItem` via `.addQualifier()` silently removes `@Default`, breaking every unqualified `@Inject GoalCompiler`. The CDI spec documents this behavior, but the Quarkus API reads like "also add this qualifier" rather than "replace @Default with this." We deferred qualifier support to #110 — it needs a two-pass approach where you count graphs first and only apply qualifiers when multiple exist.

The cross-model validation phase (duplicate ID detection across models, cycle detection spanning interface and class nodes) was partially implemented — per-class validations are in, but the global merge-then-validate Phase 2 needs its own issue (#111). The runtime is correct; it's only the build-time error messages that are missing for cross-model misconfigurations.

Five commits, 14 files, 771 lines — five new test classes covering class-only graphs, merged graphs, type-safe dependencies, class-based fault policies, and annotation misuse detection. The full build passes across all modules.

Two follow-up issues filed: #110 for CDI qualifier support, #111 for cross-model validation Phase 2. The annotation model now supports both centralized and distributed graph declarations — the next issue in the series (#106, `@GraphRule`) builds on this for parameterized graph growth rules.
