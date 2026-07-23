---
layout: post
title: "The Class That Shouldn't Have Been There"
date: 2026-07-23
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub DesiredState]
tags: [ci, dependency-management, cross-repo]
---

CI was red. The error said `NoClassDefFoundError: io/micrometer/core/instrument/MeterRegistry` — a class nothing in the spatial module directly referenced. Every test in the module failed, not just the ones touching RAS integration.

The obvious fix was adding micrometer-core to the test classpath. I tried it. Tests passed. But that's symptom treatment — the next time casehub-ras adds a `scope:provided` dependency, the same thing breaks again.

The actual problem was a class called `TestSituationDefinitionRegistry` sitting in `examples/spatial/src/test/java/io/casehub/ras/runtime/`. It extended `SituationDefinitionRegistry` from the RAS runtime jar, using a package-access hack — placing a subclass in `io.casehub.ras.runtime` inside a completely different module — to reach the package-private constructors.

JUnit's class discovery doesn't care that this isn't a test class. It scans every class in the test source tree via `Class.getDeclaredMethods0()`, which reflectively resolves all inherited method parameter types. The `@Inject` constructor on `SituationDefinitionRegistry` references `Instance<MeterRegistry>`. Micrometer is `scope:provided` in casehub-ras — not transitive to consumers. So the reflective resolution fails before any test even runs.

The fix was cross-repo. We added a `public static SituationDefinitionRegistry.forTesting(providers, ganglia)` factory method to casehub-ras — a clean entry point that doesn't require subclassing. Then deleted the hack class from desiredstate entirely. No micrometer dependency needed. No future breakage when RAS adds more provided deps.

Once the spatial module passed, a second failure surfaced: `jackson-jq` version divergence. `casehub-engine-api` brings 1.6.0, `quarkus-flow` (via `serverlessworkflow-impl-jq`) brings 1.6.2. The Maven enforcer's `DependencyConvergence` rule caught it in engine-adapter. This had been lurking — previously masked because the spatial failure killed the build first. Pinned 1.6.2 in `dependencyManagement`.

Also wrote ADR 0001 this session — the combined findings from epic #24's three POCs (planner-backed GoalCompiler, spatial/vector state, lifecycle transitions). The decision: graph model handles spatial and planning state representationally, but aggregate reasoning requires RAS as a separate layer. Runtime gains lifecycle support via `CompilationResult.Lifecycle`. No alternative state representation needed. Epic closed.

The JUnit discovery gotcha is worth knowing. The symptom — `NoClassDefFoundError` on a type you never reference — points you toward adding the missing dependency. The real cause is two modules away, in a subclass that exists only to work around access control. The `forTesting()` pattern eliminates the problem class entirely rather than papering over its dependency leaks.
