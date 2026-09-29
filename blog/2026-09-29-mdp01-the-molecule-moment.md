---
layout: post
title: "The Molecule Moment — Plugin Testing Without Java"
date: 2026-09-29
entry_type: note
subtype: diary
projects: [casehubio/casehub-desiredstate]
tags: [testing, yaml-plugins, wiremock, junit5, design-review]
---

This session built a complete test framework for YAML plugin authors. The problem: plugin authors who write zero Java have no way to verify their plugins work. They declare a resource type in YAML — spec schema, provisioning steps, actual-state detection — and then trust. The existing `MockNodeProvisioner` and friends test the reconciliation loop, not individual plugins. `PluginIntegrationTest` shows how to test a plugin, but it requires assembling a `YamlPluginProvisioner` by hand with a custom `StepRunner` and `ConditionEvaluator`. If you're the "no Java required" audience, that's not a test — it's a wall.

The design went through brainstorming with 14 decisions and two rounds of design review. The decision review pushed back on several assumptions from the original issue — the Java fluent API was deferred (the "low marginal cost" claim didn't survive scrutiny), WireMock was moved from Testcontainers to embedded (Docker is overkill for an in-process mock), and K3s was phased to v2.

The core insight is that the framework doesn't need its own test engine. `YamlPluginProvisioner` and `YamlPluginActualStateAdapter` already implement the provision-then-check cycle. The framework's value is everything around that: parsing test YAML, managing WireMock lifecycle, constructing `DesiredNode` with the right `YamlNodeSpec` (pattern-match failure otherwise silently drops all spec fields), pre-processing infrastructure variable bindings (the `VariableResolver` doesn't recurse), and evaluating declarative assertions.

The implementation hit a pre-existing build break — the platform's step API had migrated (`StepResult` to `Result`, `StepAction` to `Action`, `CatalogEntry` to `Definition`) but `plugin/runtime` hadn't been updated. We fixed that as part of the extraction work, which moved validation logic from `YamlPluginProcessor` (Quarkus deployment module) into a framework-neutral `PluginValidator` in `plugin/runtime`.

What a plugin author writes now:

```java
class MyPluginTest {
    @RegisterExtension
    static PluginTestExtension ext = PluginTestExtension.forPlugin("my-plugin");

    @TestFactory
    Stream<DynamicTest> tests() { return ext.discoverTests(); }
}
```

And a YAML file alongside their plugin definition. Five lines of Java. The rest is declarative — spec values, WireMock expectations, assertions. `mvn test` runs everything.

The work-end pipeline exposed a sequencing issue — the orchestrator's `verify_recover` step expects the merge to happen during the pipeline, not after. When it returned `META_STATE=idle`, the state machine lost its position and looped on `arc42_scan`. The work landed correctly, but the orchestrator couldn't confirm it. Worth investigating whether the state machine's recovery path needs a "merge already happened" detection.
