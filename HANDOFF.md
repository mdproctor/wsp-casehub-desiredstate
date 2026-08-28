# Handoff — casehub-desiredstate

## Last Session

Completed Phase 1 of #116 (YAML language extensions). Batches 3-5 implemented this session: PatternEvaluator extraction, sealed interface hierarchy (ResolvedRule/ResolvedInvariant with Imperative/ParameterizedReflective/Declarative variants), YAML invariants with structural pattern assertions, and conditional inclusion (when:) with dependency safety.

**Implementation this session (8 commits, Batches 3-5):**
- **Batch 3 — Engine Infrastructure:** PatternEvaluator extracted from GraphRuleEngine and GraphInvariantEngine (~80 lines of duplicated expandChain logic eliminated). Wildcard `*` type matching in PatternMatchingSupport. Sealed interface hierarchy: `ResolvedRule` and `ResolvedInvariant` with three variants each (Imperative, ParameterizedReflective, Declarative). Old `ResolvedGraphRule`/`ResolvedGraphInvariant` records removed.
- **Batch 4 — Declarative Invariants:** YamlInvariant/YamlPattern model types, build-time validation (match required, of-references checked, type validated). YamlInvariantConverter bridges YAML model to DeclarativeInvariant. GoalCompiler validates invariants via GraphInvariantEngine after graph construction. Custom message templates with `${match.*}` resolution. Pipeline-yaml integration test.
- **Batch 5 — Conditional Inclusion:** `when:` field on YamlNode. `List<Object>` dependsOn supporting `{ node: "id", optional: true }` syntax. Build-time error when unconditional node depends on conditional node. Compile-time when: evaluation (truthy: true/yes/on/y/1, falsy: false/no/off/n/0). Optional dependencies to excluded nodes silently removed. Pipeline-yaml integration test with debug-validator node.

**Phase 1 complete.** An operator can write a single YAML file with nodes, dependencies, fault policies (template-based escalation), structural invariants (pattern assertions), and conditional nodes — no Java required. 41 new tests across the session.

**Next: Phase 2** — Declarative graph rules (structural rewriting in YAML) and lifecycle phases (build-then-operate). The sealed interface and PatternEvaluator infrastructure from Batch 3 directly enables this — `DeclarativeRule` variant is already in place but not yet wired for YAML.

## Branch

`issue-116-yaml-language-design` — project + workspace

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-116-yaml-language-design/2026-08-27-yaml-language-extensions-design.md` |
| Decisions (16) | `specs/issue-116-yaml-language-design/decisions.md` |
| Competitive comparison | `specs/issue-116-yaml-language-design/2026-08-28-scales-down-comparison.md` |
| Phase 1 plan | `plans/2026-08-28-phase1-yaml-extensions.md` |
| Blog | `docs/blog/2026-08-28-mdp01-the-operator-surface-that-scales-down.md` |
| Review workspaces | `~/reviews/casehub-desiredstate/yaml-language-extensions-*` |
