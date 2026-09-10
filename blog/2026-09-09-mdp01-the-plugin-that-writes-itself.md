---
layout: post
title: "The Plugin That Writes Itself"
date: 2026-09-09
entry_type: note
subtype: diary
projects: [casehubio/casehub-desiredstate]
tags: [desiredstate, yaml, plugin-architecture, spi]
series: issue-87-yaml-plugin-architecture
---

The desired-state runtime knows how to provision, monitor, and self-heal any resource type — as long as someone writes the Java. A `NodeProvisioner`, an `ActualStateAdapter`, a `NodeSpec` record, fault policies, and at least four other files. Every time ops needed a new resource type — a Kubernetes Deployment, a Cloudflare DNS record, a Supabase database — someone had to write all of that from scratch.

The YAML language extensions (#116) solved half the problem. An operator can declare a graph of nodes, dependencies, fault policies, and structural invariants in YAML. But "what should exist" is only half of desired-state management. The other half — "how to provision it, how to check if it exists, how to detect when it drifts" — stayed in Java.

I wanted to close that gap entirely. One YAML file per resource type. Five sections: spec schema, actual-state detection, provisioning, fault escalation, CBR learning surface, RAS situation definitions. No Java. An operator who knows REST APIs should be able to write a plugin that provisions a Kubernetes Deployment the same way a platform engineer writes one in Java — same reconciliation loop, same fault policies, same CBR/RAS integration.

The design took fourteen decisions to lock down. The interesting ones: dual NodeSpec declaration (Java records and YAML schemas coexist — an operator picks whichever fits), a step pipeline model for composing REST calls within a provisioner (sequential, no workflow semantics — the plan-level orchestration already handles parallelism), and full YAML-over-YAML-over-Java primitive composition from day one. I was tempted to defer composition, but the issue explicitly requires it, and retrofitting a composition model onto a primitive contract that wasn't designed for it would mean a breaking change.

The step pipeline deliberately stays simple. Each step invokes a named primitive, receives parameters, produces a named result. No parallelism, no branching, no durability — those concerns live one level up in the `CaseTransitionExecutor`'s Serverless Workflow. A plugin author sees only their sequential pipeline; the fact that independent nodes provision concurrently is invisible to them.

We built the foundation: the `StepPrimitive` SPI, a `StepResult` type with deep path traversal, a `StepContext` that dispatches `${spec.*}`, `${auth.*}`, `${result.*}`, and `${param.*}` references by prefix, an expression evaluator (recursive descent parser for the condition vocabulary), and a `PluginInterpolator` that bridges interpolation to condition evaluation. The expression evaluator handles SQL-style null semantics — any comparison with null evaluates to false, null equals null evaluates to true — which makes `compare-state` safe: indeterminate values fall through all conditions to `NodeStatus.UNKNOWN`.

The next piece wires the step executor to the existing `NodeProvisioner` and `ActualStateAdapter` SPIs via generic routing beans — one `YamlPluginProvisioner` handles all YAML-declared types, dispatching internally by `NodeType` to the right step pipeline. The same pattern the `DefaultNodeProvisionerRouter` already uses, just with YAML descriptors instead of Java beans.

The thing I keep coming back to: this is the layer where "no Java required" either works or doesn't. The spec schema, the interpolation, the expression evaluator — those are mechanical. The real test is whether the step pipeline model is expressive enough for real provisioners. A K8s Deployment is three REST calls. A multi-vendor setup with DNS, load balancer, and application server is twelve. If the model handles both without falling back to Java, the architecture is right.
