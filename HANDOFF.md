# HANDOFF — casehub-desiredstate

## Last Session

Closed #180 (plugin catalog and versioning — foundation layer). Landed as `1eabf1d` on main.

**What shipped:**
- **PluginHeader evolution** — version int→semver string with backward compat, description field, `PluginRequires` record for dependency declaration (runtime version range, required primitives, required types)
- **Policy interface extraction** — `FaultPolicyEvaluator` and `DriftPolicyEvaluator` interfaces in api/, ReconciliationLoop/Builder/CDI/Spring migrated from concrete classes to interfaces
- **Dynamic infrastructure** — `DynamicNodeProvisionerRouter`, `DynamicActualStateAdapterRouter`, `DynamicFaultPolicyEngine`, `DynamicDriftPolicyEngine` with ConcurrentHashMap/CopyOnWriteArrayList backing, atomic register with collision rollback
- **ReconciliationStateStore.allTenancyIds()** — SPI + InMemory, JPA, Spring JPA implementations
- **ExtensionRegistry** — combined query surface for static (classpath) and dynamic (catalog) types
- **Code review** fixed partial-registration bug on multi-type collision, added 4 missing test files

**What's deferred:**
- DesiredStatePluginManifest, DesiredStatePluginActivator, PluginCatalog integration, filesystem watcher, reload endpoint — all blocked on casehubio/platform#662 (generic catalog SPI + filesystem repository)

## Cross-Module

#161 still pending — Spring auto-config codegen requires work in `casehubio/parent` repo.

## References

| Artifact | Location |
|----------|----------|
| Epic | casehubio/casehub-desiredstate#173 |
| #180 commit | 1eabf1d |
| Design spec | specs/issue-180-plugin-catalog-versioning/2026-10-26-plugin-catalog-versioning-design.md |
| Decisions | specs/issue-180-plugin-catalog-versioning/decisions.md (11 decisions, reviewed) |
| Implementation plan | plans/2026-10-26-plugin-catalog-versioning.md |
| Blog entry | blog/2026-10-26-mdp01-the-infrastructure-before-the-infrastructure.md |
| Platform blocker | casehubio/platform#662 |
