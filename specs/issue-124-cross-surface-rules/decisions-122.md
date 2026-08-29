# Decisions — #122 TypeScript DSL

## D1: Scope

**Choice:** Full parity + programmatic generation + LLM generation target (all three)
**Alternatives:**
- Programmatic generation only — targeted complement for YAML gaps
- LLM generation only — .d.ts as context, SDK as generation target
**Rationale:** The TS DSL should be a complete alternative surface. Operators choose based on preference and use case, not capability gaps.
**Trade-offs:** Largest implementation scope — every YAML feature must have a TS equivalent
**Sources:** Issue #122, research doc §5.2, YAML language extensions spec §6
**Exploration:** quick
**Status:** captured

## D2: Integration path

**Choice:** TSJ (ts2jvm) — TypeScript compiled to JVM bytecode, returns JSON. Both build-time and runtime evaluation.
**Alternatives:**
- GraalJS + transpile — mature JS engine but indirect TS support
- Javet (V8 binding) — full Node.js but heavyweight JNI dependency
- REST endpoint — TS compiles offline, posts JSON to Quarkus app
- Build-time classpath only — .ts files at META-INF, no runtime eval
**Rationale:** TSJ provides native TS execution on JVM without external toolchain. Build-time for static declarations (classpath .ts files); runtime for dynamic/LLM-generated.
**Trade-offs:** TSJ maturity — experimental, may have gaps. Fallback to JSON serialization boundary if TSJ proves inadequate.
**Sources:** User input (TSJ library identified)
**Exploration:** quick
**Status:** captured

## D3: DSL style

**Choice:** Hybrid — imperative graph construction with native TS (loops, conditionals, functions), declarative pattern-matching API for rules and invariants
**Alternatives:**
- Fully imperative — rules/invariants expressed as TS functions
- Fully declarative — mirror YAML's forEach/when/modules as TS DSL concepts
**Rationale:** TS already has loops/conditionals — re-inventing them as DSL concepts adds complexity without value. But rules/invariants are structural graph concepts (pattern matching, rewriting) — they need a declarative vocabulary, not arbitrary TS functions.
**Trade-offs:** Two authoring styles in one SDK — imperative for graph construction, declarative for rules
**Sources:** YAML language extensions spec §6.4 (rules), §6.2 (invariants)
**Exploration:** quick
**Status:** captured

## D4: SDK API design

**Choice:** Object literal / `defineGraph()` with NodeTypeMap discriminated unions
**Alternatives:**
- Functional composition — nodes as first-class values, composed with helper functions
- Fluent builder — method chaining (Java-style)
**Rationale:** Object literal shape mirrors YAML (reduces cognitive load across surfaces). TypeScript structural typing provides deep validation via discriminated unions on the `type` field — spec autocomplete narrows automatically. LLMs generate object literals more reliably than function chains. Native TS (spread, map, conditionals) provides programmatic power without DSL-specific constructs.
**Trade-offs:** Less composable than functional approach for very complex programmatic patterns. Node ID cross-references validated at build time, not compile time.
**Sources:** Zod, tRPC, Drizzle (TS ecosystem precedent for object literal APIs)
**Exploration:** deep-analysis
**Status:** captured

## D5: Type safety mechanism

**Choice:** Generated .d.ts from Java @NodeTypeId-annotated types, producing a NodeTypeMap discriminated union
**Alternatives:**
- Manual TS types — hand-written, drift risk
- Untyped spec maps — Map<string, unknown>, weakest safety
**Rationale:** Auto-generated types stay in sync with Java. The NodeTypeMap enables spec-level autocomplete keyed on node type — the killer feature over YAML.
**Trade-offs:** Requires a type generation tool/build step. Generated types may need manual review for complex Java types.
**Sources:** GraphDescriptor.java, NodeDescriptor.java, @NodeTypeId
**Exploration:** quick
**Status:** captured

## D6: Positioning relative to YAML

**Choice:** Programmatic power and type safety are the primary value propositions. LLM generation is a secondary benefit. Consumer decides which surface.
**Alternatives:**
- Position TS as the primary LLM target, YAML for humans
- Position TS as strictly superior to YAML
**Rationale:** For static declarations, YAML is simpler and sufficient. TS enables declarations YAML can't express (programmatic graph construction). The .d.ts files add value as LLM context regardless of output format.
**Trade-offs:** Neither surface is positioned as "preferred" — consumer must understand when each is appropriate
**Sources:** Research doc §2.3 (Ansible observation about psychological contract), §5.2
**Exploration:** quick
**Status:** captured
