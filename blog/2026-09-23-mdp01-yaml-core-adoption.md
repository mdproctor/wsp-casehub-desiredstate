---
title: "Stopping the Drift: Adopting yaml-core Constructs in Desired State"
author: mdp
entry_type: note
subtype: diary
date: 2026-09-23
tags: [yaml-core, refactoring, variable-resolution, condition-evaluation, module-outputs]
projects: [casehub-desiredstate]
---

# Stopping the Drift: Adopting yaml-core Constructs in Desired State

Platform's yaml-core keeps growing — conditions, CSV forEach, module outputs, typed variable resolution — and desiredstate hasn't been keeping up. We extracted yaml-core *from* desiredstate months ago, but the consumer side still had local duplicates and hand-rolled workarounds that the shared library now handles better.

I wanted to know exactly where the gaps were, so Claude ran a parallel audit: one fork mapping yaml-core's new API surface, another mapping how desiredstate currently consumes it. The overlap was bigger than I expected — seven concrete opportunities, ranging from trivial dedup to meaningful capability unlocks.

## What Landed

**Fault template resolution.** `YamlFaultPolicyBuilder` had its own `resolveFaultString()` — three chained `String.replace()` calls for `${fault.nodeId}`, `${fault.type}`, `${fault.detail}`. That's exactly what `VariableResolver` does, except the resolver also handles nested maps, lists, default values, and error messages. We replaced the hand-rolled version with a `VariableResolver` using `fault` as an active prefix. The old code silently passed through typos like `${fault.noeId}`; the new code throws `UnresolvedVariableException`. Better.

**ForEachDirective.parse().** `YamlNodeForEachAdapter` had a local `toDirective()` that duplicated the canonical `ForEachDirective.parse()` factory. The upstream version handles more cases — string-as-GroupRef-with-as, for instance — and stays in sync as the sealed interface evolves. The local copy was 10 lines waiting to drift.

**ConditionEvaluator.** Two call sites in `YamlGraphRecorder` used `Truthiness.isTruthy()` directly — which means `when:` conditions only supported boolean literals. Swapping to `ConditionEvaluator` wraps the same truthiness check but adds an expression delegate hook. Right now the delegate is null (same boolean behaviour), but the abstraction is in place for richer expressions later — `when: "${var.batch_size} > 500"` instead of `when: "true"`.

**DeferredPrefixHandler for match reference validation.** This was the most satisfying change. When YAML rules use `${match.sink.id}` in actions, `sink` must be a declared binding in the rule's `match:` section. Previously, a typo like `${match.snk.id}` passed through silently — the `match` prefix was deferred, so the resolver left it as-is without complaint. yaml-core's `DeferredPrefixHandler` fires a callback every time a deferred reference is encountered. We wired one into `YamlRuleConverter` that collects all `match.*` binding names, then cross-checks against the declared bindings after resolution. Typos now throw at compile time with a clear error: *"Rule 'bad-ref-rule': action references ${match.snk.*} but no binding 'snk' exists. Available: [sink]."*

The enabling change was subtle: `resolveVarInActionParams` previously only passed strings containing `${var.` through the resolver, so `${match.*}` references never reached it and the handler never fired. Broadening to `${` means all variable references go through resolution — deferred ones still pass through unchanged, but the handler sees them.

**Module outputs wiring.** yaml-core modules can declare `outputs:` — resolved values that downstream imports reference via `${module.alias.outputName}`. The module expander already resolves these, but desiredstate's `YamlGraphRecorder` never connected `outputSource()` into the variable resolver. One conditional `withScope("module", ...)` call after module expansion, and cross-module data flow works.

## What Didn't Land

Two items deferred: CSV-backed forEach and ObjectVariableSource for typed resolution. Both require adding a field to the `YamlGraph` record, which has 12 direct constructor call sites across tests. Mechanically straightforward but invasive for a branch focused on adopting existing APIs. They'll get their own branch where the model change is the focused concern.

## Looking Forward

The `ConditionEvaluator` delegate is the interesting open question. Right now it's null — boolean-only, same as before. But yaml-core provides the hook for an expression engine. The natural fit is a lightweight comparator — string equality, numeric comparison, maybe regex matching — so YAML authors can write `when: "${var.environment} == production"` without needing a full scripting engine. That's a yaml-core concern, not a desiredstate one, but desiredstate would be the first consumer to benefit.
