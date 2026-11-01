# HANDOFF — casehub-desiredstate

## Last Session

Post-bridge code review of Ansible and Terraform adapters. Found 5 bugs, 1 missing feature, 1 doc gap. Issues #189, #187, #188 were already landed from a prior session.

**Review findings — 7 issues created:**

| # | Title | Priority | Scale | Complexity |
|---|-------|----------|-------|------------|
| 200 | bug: Terraform applyPlan() uses resourceId as tenancyId and typeId | P1 | XS | Low |
| 201 | bug: Ansible plan() returns CREATE when no changes needed | P2 | XS | Low |
| 202 | bug: Spring bridge auto-config discovery opens directory URL as stream | P1 | S | Low |
| 203 | feat: complete Quarkus bridge deployment extension | P2 | M | Med |
| 204 | bug: Terraform doReadState treats warnings as drift | P2 | XS | Low |
| 205 | fix: AnsibleErrorClassifier never returns PERMANENT | P3 | XS | Low |
| 206 | docs: bridge adapter documentation | P3 | S | Low |

**Key findings:**
- Terraform `applyPlan()` path is broken at runtime — `planId` carries resourceId instead of tenancyId/typeId
- Spring manifest discovery is broken for ALL bridge adapters — directory URL opened as stream, silently returns empty
- Quarkus `BridgeProcessor` is a stub — only registers native-image resources, no bean wiring
- Both adapters' core protocol stacks are solid (52 Ansible tests, 133 Terraform tests all passing)

## Next Session

**Recommended order:** 200 → 204 → 201 → 205 → 202 → 203 → 206. Bug fixes first (small, targeted), then Spring discovery, then Quarkus deployment (largest), then docs last.

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
