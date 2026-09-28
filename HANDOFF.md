# Handoff — casehub-desiredstate

## Last Session

Completed #149 (YamlGraph CSV forEach + ObjectVariableSource typed resolution) end-to-end:
resumed from pause stack, implemented ObjectVariableSource wiring, widened YamlGraph.variables
to Map<String,Object>, wired CSV data source dispatch, added PARSE_BOOLEAN_LIKE_WORDS_AS_STRINGS
to all YAML ObjectMapper instances. Branch audit caught a Spring parity gap (Factory missing
CSV wiring) — fixed before landing. Squashed to 2 commits, landed on main.

Key finding: widening a Jackson YAML record field from Map<String,String> to Map<String,Object>
silently coerces YAML 1.1 boolean-like words (yes/no/on/off) from String to Boolean. Fix:
YAMLParser.Feature.PARSE_BOOLEAN_LIKE_WORDS_AS_STRINGS. Captured as garden entry GE-20260928-1e3a2e.

## Branch State

On `main`. No active branch. #149 closed, #150 recommended next.

## Cross-Module

Pre-existing `work-adapter` test failure (WorkItemRef constructor mismatch) — unrelated.

## References

| Artifact | Location |
|----------|----------|
| #149 spec | `specs/issue-149-yamlgraph-csv-foreach/` |
| #149 diary | `blog/2026-09-28-mdp01-yaml-typed-variables.md` |
| Garden entry | `~/.hortora/garden/jvm/GE-20260928-1e3a2e.md` |
