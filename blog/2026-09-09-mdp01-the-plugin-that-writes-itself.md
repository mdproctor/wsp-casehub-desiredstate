---
layout: post
title: "The Plugin That Writes Itself"
date: 2026-09-09
entry_type: note
subtype: diary
projects: [casehubio/casehub-desiredstate]
tags: [desiredstate, yaml, plugin-architecture, spi, step-pipeline]
series: issue-87-yaml-plugin-architecture
---

The desired-state runtime knows how to provision, monitor, and self-heal any resource type — as long as someone writes the Java. A `NodeProvisioner`, an `ActualStateAdapter`, a `NodeSpec` record, fault policies, and at least four other files. Every time ops needed a new resource type — a Kubernetes Deployment, a Cloudflare DNS record, a Supabase database — someone had to write all of that from scratch.

The YAML language extensions (#116) solved half the problem. An operator can declare a graph of nodes, dependencies, fault policies, and structural invariants in YAML. But "what should exist" is only half of desired-state management. The other half — "how to provision it, how to check if it exists, how to detect when it drifts" — stayed in Java.

I wanted to close that gap entirely. One YAML file per resource type. Five sections: spec schema, actual-state detection, provisioning, fault escalation, CBR learning surface, RAS situation definitions. No Java. An operator who knows REST APIs should be able to write a plugin that provisions a Kubernetes Deployment the same way a platform engineer writes one in Java — same reconciliation loop, same fault policies, same CBR/RAS integration.

The design took fourteen decisions to lock down. The interesting ones: dual NodeSpec declaration (Java records and YAML schemas coexist — an operator picks whichever fits), a step pipeline model for composing REST calls within a provisioner (sequential, no workflow semantics — the plan-level orchestration already handles parallelism), and full YAML-over-YAML-over-Java primitive composition from day one. I was tempted to defer composition, but the issue explicitly requires it, and retrofitting a composition model onto a primitive contract that wasn't designed for it would mean a breaking change.

The step pipeline deliberately stays simple. Each step invokes a named primitive, receives parameters, produces a named result. No parallelism, no branching, no durability — those concerns live one level up in the `CaseTransitionExecutor`'s Serverless Workflow. A plugin author sees only their sequential pipeline; the fact that independent nodes provision concurrently is invisible to them.

We built the foundation: the `StepPrimitive` SPI, a `StepResult` type with deep path traversal, a `StepContext` that dispatches `${spec.*}`, `${auth.*}`, `${result.*}`, and `${param.*}` references by prefix, an expression evaluator (recursive descent parser for the condition vocabulary), and a `PluginInterpolator` that bridges interpolation to condition evaluation. The expression evaluator handles SQL-style null semantics — any comparison with null evaluates to false, null equals null evaluates to true — which makes `compare-state` safe: indeterminate values fall through all conditions to `NodeStatus.UNKNOWN`.

Then the implementation proper. A Jackson tree-based parser that handles the irregular step structure — each step is a single-key map where the key is the primitive name and the value is a parameter block, with control-flow directives (`when`, `on-error`, `result`) extracted from the parameter map before it reaches the primitive. The `StepPipelineExecutor` runs steps sequentially, accumulates result bindings across steps, evaluates `when:` conditions for conditional execution, and handles retry with configurable backoff. Four built-in primitives ship with the runtime: `rest-call` (HTTP via `java.net.http.HttpClient`), `json-extract`, `compare-state` (absent → drifted → present evaluation order — the ordering matters because a resource can be present but drifted), and `assert`.

The SPI wiring turned out clean. A single `YamlPluginProvisioner` implements `NodeProvisioner` and routes by `NodeType` to the right step pipeline — the same dispatch pattern `DefaultNodeProvisionerRouter` uses. `YamlPluginActualStateAdapter` does the same for actual-state detection. Both resolve credentials via `CredentialResolver` at pipeline-entry time, never during step execution. The build processor validates everything at Quarkus build time: spec schemas, primitive existence (with Levenshtein-based "did you mean?" suggestions), interpolation references against the spec schema, result binding forward references, type conflicts against Java `@NodeTypeId` declarations. If it compiles, it runs.

The compound primitive expander is where composition gets interesting. A YAML-defined primitive like `k8s-api-call` wraps `rest-call` with auth and URL boilerplate. At build time, the expander inlines the compound's steps at each invocation site with parameter bindings — macro expansion, not function calls. `${param.*}` scopes innermost-wins, so a compound primitive's parameters shadow its parent's. Cycle detection and a max nesting depth of five prevent runaway composition.

The part I didn't expect to surface during implementation: the plugin system's interpolation engine is a reimplementation of yaml-core's `VariableResolver`. Same prefix dispatch model, same recursive map walking, different source types. The compound primitive expander is a reimplementation of yaml-core's module expansion. I'd designed the plugin system as a peer to the YAML surface, but the infrastructure underneath them is the same infrastructure. That's either an accidental duplication or a signal that the step pipeline belongs in yaml-core as a generic capability. I filed #88 to consolidate — not because a second consumer exists today, but because an LLM implementing a compliance or IoT plugin system next month won't discover the desiredstate-specific module. It'll build its own.
