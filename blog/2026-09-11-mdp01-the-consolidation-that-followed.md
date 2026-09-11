---
layout: post
title: "The Consolidation That Followed"
date: 2026-09-11
entry_type: note
subtype: diary
projects: [casehubio/casehub-desiredstate]
tags: [desiredstate, yaml, yaml-step-core, platform, consolidation]
series: issue-87-yaml-plugin-architecture
---

The plugin architecture landed two days ago and immediately told me what to do next. The step pipeline's interpolation engine — prefix-based `${spec.*}`, `${auth.*}`, recursive map walking, expression evaluation — is the same algorithm as yaml-core's `VariableResolver`. Different source types, identical mechanism. The compound primitive expander does the same parameterised expansion as yaml-core's `ModuleExpander`. I'd designed the plugin system as a peer to the YAML surface, but the infrastructure underneath them is the same infrastructure.

The question was whether to consolidate now or let it sit. The case for now: an LLM implementing a compliance or IoT step pipeline next month won't discover the desiredstate-specific module. It'll build its own third copy. The case for later: the plugin system just landed, the interfaces are fresh, let them settle. I went with now — the types are already generic (zero domain imports in the SPI layer), and the migration pattern from #128 is well-established.

## What moved

A new platform module, `casehub-platform-yaml-step-core`, takes ownership of the generic step pipeline infrastructure. Sixteen types moved: `StepPrimitive`, `StepResult`, `StepParameters`, `StepContext`, `StepDef`, `CompoundStepDef`, `StepPipelineExecutor`, `PrimitiveRegistry`, `CompoundStepExpander`, `ExpressionEvaluator`, and the three generic primitives (`RestCallPrimitive`, `JsonExtractPrimitive`, `AssertPrimitive`).

The interesting design decision was `StepContext`. My first instinct was to have it implement yaml-core's `VariableSource` — one class handling all four prefixes. But `VariableSource` is a `@FunctionalInterface` with `String resolve(String name)` — it resolves within a single prefix. The `VariableResolver` does the prefix routing. So `StepContext` became a source *provider* instead: `specSource()`, `authSource()`, `resultSource()`, `paramSource()` each return a `VariableSource` closure, and `toResolver()` wires them into a `VariableResolver`. Domain layers extend with `.withScope("var", variableSource)` for graph variables and fault context.

The other shift was where interpolation happens. Previously, every primitive had a `private final PluginInterpolator interpolator` field and resolved `${...}` references in its own parameters. Now the executor pre-resolves all parameters via `VariableResolver.resolveMap()` before passing them to the primitive. Primitives receive fully resolved values — they don't need to know about interpolation at all. `PluginInterpolator` is deleted entirely.

## What stayed

Domain-specific wiring stays in desiredstate: `CompareStatePrimitive` (maps step results to `NodeStatus`), `YamlPluginProvisioner`, `YamlPluginActualStateAdapter`, and a new `ActualStateStepExecutor` that wraps the generic executor and maps `StepResult` → `NodeStatus`. The `plugin/api` module survives as a thin re-export — it depends on yaml-step-core (transitive) and owns only `YamlNodeSpec`.

The net effect is 2,200 lines removed from desiredstate, 1,800 added to platform. Single implementation for all `${...}` resolution across the platform. The next domain that needs a step pipeline gets it from yaml-step-core and never touches desiredstate.
