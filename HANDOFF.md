# Handoff — casehub-desiredstate

## Last Session

Closed #84 (CDI no-args constructors for Cdi* bridge classes) and #83
(overlay full node equality). Also closed epic #61 — all child issues and
cross-repo deps were already done. ARC42 stale scan fixed Chapters 8 and 9
(pending → delivered). Landed as `6d4d8eb` on main.

## Immediate Next Step

Pick next from What's Next table. #27 (managed pipeline mode) has a branch
`issue-27-managed-pipeline-mode` on the pause stack — resume with `/work`.

## What's Left

- `neocortex#142` — Wire CbrOutcomeConsumer to platform CloudEvent routing · open
- `neocortex#141` — Map<String, Object> → Map<String, FeatureValue> migration · open
- casehub-ops — remove App* workaround clones now that #84 shipped · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Managed pipeline mode — Quarkus Flow per stage | M | High | On pause stack |
| #25 | Desired-state as alternative case planning model | L | High | Depends on parent#233 |

## References

- Blog: `blog/2026-07-19-mdp01-two-bugs-and-an-orphaned-epic.md`
