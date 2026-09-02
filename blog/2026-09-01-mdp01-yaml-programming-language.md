---
layout: post
title: "We Accidentally Built a YAML Programming Language"
date: 2026-09-01
entry_type: article
subtype: diary
projects: [casehubio/casehub-desiredstate]
tags: [yaml-core, graph-core, module-system, architecture, platform]
---

# We Accidentally Built a YAML Programming Language

I started this session planning to add count constraints to graph invariants. I ended it designing a generic graph pattern matching engine and realising we'd built something that looks a lot like a programming language.

## Cardinality — the quick win

The starting point was straightforward. Graph invariants could check existence — "every sink must have an upstream transformer" — but not count. The casehub-ops team needed "HA requires at least 3 compute instances" and "exactly one control plane." The fix: `minCount`/`maxCount` on `@Match`, `@DirectDep`, and `@Reaches`, plus the YAML equivalent. Two levels of counting — match-level (how many nodes of this type exist globally) and expansion-level (how many neighbors per anchor). The design fell out naturally from the existing pattern matching architecture, and the implementation landed in a few hours.

One wrinkle: adding a second constructor to a Java record breaks Quarkus bytecode recording. The recorder sees multiple constructors and falls back to bean-style serialisation, which doesn't work with records (read-only properties). `@RecordableConstructor` on the canonical constructor fixed it. Worth remembering for any record that needs a convenience constructor alongside Quarkus.

## The migration question

Then I moved to module parameter validation — `minLength`, `maxLength`, `pattern` for YAML module parameters. Brainstorming surfaced something I should have noticed earlier: we had a `yaml-core` module in platform that was extracted from desiredstate, but desiredstate never actually migrated to use it. We were still running local copies of `VariableResolver` and `ForEachExpander`, adding new YAML features on top of diverging code.

The right sequence was obvious once I saw it: migrate first, then build new features on the shared foundation. But the migration wasn't simple — the APIs had diverged. yaml-core's `VariableResolver` uses generic `VariableSource` prefix composition. The local version had hardcoded sources. yaml-core's `ForEachExpander` is generic with `ForEachAdapter<E>`. The local version was coupled to `DesiredNode`, `NodeSpec`, and `ObjectMapper`.

I did a systematic comparison — twelve potential regressions across VariableResolver, ForEachExpander, and ModuleExpander. Some were API gaps (no way to chain a module scope without carrying the base source around). Some were capability gaps (JSON array parsing for dynamic iteration values). One was a genuine type safety loss from the generic sections model.

Each gap became a platform issue. Three rounds of improvements went into yaml-core before the migration could proceed without regressions:

- `sourceFor()` and `withChainedScope()` — module scope without plumbing
- `IterationValueExpander` — pluggable iteration value parsing
- `SectionDeserializer` + `SectionContentRewriter` + typed accessor — type safety during module expansion
- `DeferredPrefixHandler` — context-dependent error messages for template variables

## Prior art

Somewhere in the middle of designing parameter constraints, I asked the obvious question: does this already exist? We searched. The answer: yes, extensively.

CloudFormation has had `MinLength`, `MaxLength`, `AllowedPattern`, `MinValue`, `MaxValue` on parameters since approximately 2011. Terraform has module parameters with validation blocks and custom error messages. CUE treats types and constraints as the same thing — `minLength: 3` is part of the type definition, not a separate annotation. VMware's ytt does structural YAML templating with a sandboxed Starlark scripting language.

What we'd built in yaml-core — variable resolution, forEach expansion, conditional inclusion, modules with parameters, parameter validation with type-aware constraints — is the same feature set, reimplemented in Java because we need it embedded in a Quarkus build pipeline. The standalone tools can't embed. CUE and ytt are Go-based. But the concepts are identical.

The genuinely novel parts are the graph-aware features: pattern matching rules with a fixed-point loop, structural invariants with universal quantification, cardinality constraints. No existing YAML tool has these. They justify the custom system. The rest is table stakes that we built because embedding requirements ruled out adoption.

## Module outputs — the Terraform lesson

I pushed yaml-core's module system further. Terraform's most powerful module feature isn't parameters — it's outputs. A module that provisions a database outputs `connection_url`. A module that provisions an application consumes it. Without outputs, modules are isolated node factories. With them, they're composable abstractions with defined interfaces.

We designed this for yaml-core:

```yaml
module:
  name: database
  parameters:
    engine: { type: string, required: true }
    port: { type: integer, default: 5432 }
  outputs:
    connection_url: { type: string, value: "jdbc:${var.engine}://db:${var.port}/app" }
```

Consumer:
```yaml
imports:
  - module: database
    as: app-db
    parameters: { engine: postgres }
  - module: cache
    as: app-cache
    parameters:
      backend_url: "${module.app-db.connection_url}"
```

Import order determines resolution order. Outputs are typed — the build step catches type mismatches between an output's declared type and the consuming parameter's type before any expansion runs. Design-time validation, not runtime safety nets. This distinction matters: ForEach uses `ForEachAdapter<E>` for compile-time type safety, SectionDeserializer gives typed access during expansion. Outputs follow the same principle — validate at design time, catch at build time, runtime ParameterValidator is the safety net for values that escape static analysis.

## GraphView — the abstraction that almost went wrong

The biggest architectural question was whether to extract the graph engines to platform. The pattern matching, rule evaluation, and invariant checking are generic — any typed directed graph could use them. Engine case plans, ops deployment topologies, eidos component models — they all have the same structural validation needs.

Claude initially proposed requiring domain types to implement a `Graph<N>` interface. I pushed back: that forces desiredstate's foundation-tier `api/` module to depend on platform's graph API. The reader/adapter pattern is better — engines work through `GraphReader<G, N>` adapters, domain types stay untouched.

But raw reader threading creates ceremony — every internal method gains `<G, N>` type parameters plus separate `graph` and `reader` parameters, threaded through 25+ methods. The fix: `GraphView<N>` wraps graph + reader at the API boundary. Engine internals operate on `GraphView<N>` — single type parameter, single parameter. The method bodies are actually simpler than today because `NodeType.of(string).equals(node.type())` becomes `graph.nodeType(node).equals(string)` — direct string comparison, no wrapper objects.

The design supports two consumer patterns without the engine knowing which one is in use. If your domain separates graph structure from content (generic nodes with a payload property), the reader extracts through the generic structure. If your domain unifies them (desiredstate's `DesiredNode` IS the graph node), the reader extracts through domain accessors. Same engine, same algorithms, different reader implementation. When platform eventually gets concrete graph data structures, they're just another reader — the engines don't change.

## What's next

The yaml-core migration (#128) is unblocked — all platform improvements have landed. The plan needs rewriting against the final API and then execution. After that, #126 (module parameter validation in desiredstate) becomes an adoption exercise rather than a greenfield build.

The graph-core extraction is logged but not imminent. Step one is an in-place refactor in desiredstate — introduce `GraphView` and readers, verify the engines have zero domain imports, then the extraction to platform is a mechanical package move. The important thing was getting the architecture right before writing code, and I think we did.

The `@RecordableConstructor` gotcha for multi-constructor records is probably worth a garden entry. And I still owe an ADR documenting why we didn't adopt CUE or ytt — the answer is "Java embedding requirements" but it should be on record so the question doesn't get re-asked.
