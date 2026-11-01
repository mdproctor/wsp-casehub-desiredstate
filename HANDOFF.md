# HANDOFF — casehub-desiredstate

## Last Session

Closed #185 (Ansible collection bridge adapter) and epic #182 (bridge adapter architecture). Landed as `25b1b9f` on main, audit fixes as `05f6ff0`.

**What shipped:**
- **`bridge/ansible/` module** — `AnsibleBridgeProvider` implements `BridgeProvider` + `ImportCapable` + `PlanCapable` via `ansible-playbook` CLI with JSON callback. Eight production classes: `PlaybookGenerator`, `AnsibleOutputParser` (drift detection via `drift_keys`, output extraction via `output_keys`), `InventoryGenerator`, `AnsibleProcess` (0700 temp dirs, virtual-thread stderr drain, shutdown hook), `AnsibleErrorClassifier` (TRANSIENT default), `AnsibleManifestExtensionParser`, `AnsibleModuleConfig`, `ProcessExecutor`
- **Spring auto-config** — `AnsibleBridgeAutoConfiguration` with `@ConditionalOnMissingBean`
- **Audit fixes** — `applyPlan()` was returning cached check-mode result instead of executing real provisioning (stored original request in `AnsiblePlanContext`, delegates to `apply()`); stderr/stdout separation via virtual thread; null `resultKey` NPE guard; `splitSpec()` type-safe casts
- **Epic #182 closed** — all four children complete (#183 SPI, #184 Terraform, #185 Ansible, #186 testing)
- 52 tests (unit + TCK contract tests for all three SPIs)

**Blog entry:**
- Diary: workspace `blog/2026-10-31-mdp01-ansible-bridge-and-the-bug-that-looked-right.md`

## Next Session

Start with `work start 189` — all three issues are follow-ons from #134 rollback work.

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|-------|-------|------------|------------|-------|
| 189 | fix: GraphSerializer ordering constraints | XS | Low | — | Bug fix, do first — unblocks correct history round-trips |
| 187 | feat: PreferenceProvider for history prune depth | XS | Low | — | Mechanical wiring, same session as 189 |
| 188 | feat: per-node rollback | S | Med | — | Builds on history store from #134, needs design for dependency semantics |

**Recommended order:** 189 → 187 → 188. The two XS issues are quick wins on the same code paths. #188 is the meatiest and benefits from a fresh context window.

## Cross-Module

#161 still pending — Spring auto-config codegen requires work in `casehubio/parent` repo.

## References

| Artifact | Location |
|----------|----------|
| Issue | casehubio/casehub-desiredstate#185 |
| Epic | casehubio/casehub-desiredstate#182 (closed) |
| Commits | `25b1b9f` (feature), `05f6ff0` (audit fixes) on main |
| Design spec | specs/issue-185-ansible-bridge-adapter/2026-10-31-ansible-bridge-adapter-design.md |
| Decisions | specs/issue-185-ansible-bridge-adapter/decisions.md (7 decisions) |
| Implementation plan | plans/2026-10-31-ansible-bridge-adapter.md |
| Design review | ~/reviews/casehub-desiredstate/ansible-bridge-adapter-design-20261031-050508/ |
