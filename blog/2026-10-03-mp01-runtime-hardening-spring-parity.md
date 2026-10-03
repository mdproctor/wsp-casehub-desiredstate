---
layout: post
title: "Runtime Hardening — The Test Coverage That Was Missing and the Spring Parity That Wasn't"
date: 2026-10-03
entry_type: note
subtype: diary
projects: [casehubio/casehub-desiredstate]
tags: [testing, spring, cdi, runtime, hardening]
series: issue-164-runtime-hardening-batch
---

# Runtime Hardening — The Test Coverage That Was Missing and the Spring Parity That Wasn't

A hardening audit turned up two kinds of gap in the desiredstate runtime. The first was coverage: `NodeStepExecutor` had 2 of roughly 15 behavioural paths tested. `ReconciliationEventEmitter` was missing methods entirely. Three listener and lifecycle components — `ExemptionEvictionListener`, `CdiTransitionActionHandler`, `StatefulNodeProvisioner` — had zero test coverage despite being wired into the reconciliation loop.

The second gap was structural. The Spring auto-configuration had an exclusion in its test `application.properties` that masked a fundamental problem: it couldn't start. Eight beans that CDI provides via `@DefaultBean` — `HumanNodeHandler`, `PendingApprovalHandler`, `LifecycleStepExecutor`, `NotificationSink`, `ConfigurationRetriever`, `ConfigurationAdapter`, `MergedEventSource`, `ActualStateAdapterRouter` — had no Spring equivalents. The `LifecycleManager` was CDI-only. `NodeProvisionerRouter` didn't inject `PreferenceProvider`, so resync-interval overrides were silently ignored on Spring.

The exclusion in `application.properties` was the interesting discovery. It meant the integration test had been passing for the wrong reason — it was testing that a Spring Boot app loads without any desiredstate runtime, not that the runtime actually wires up. The four tests that passed (context loads, health check, Jackson, platform beans) would pass for any Spring Boot app with those starters.

## The architectural fix

The NoOp implementations lived in the CDI `runtime` module with Quarkus annotations (`@DefaultBean`, `@ApplicationScoped`). They were framework-neutral logic wearing framework-specific clothes. Moving them to `runtime-core` as plain POJOs — the same module that already held `DefaultFaultCountStore`, `DefaultExemptionStore`, and friends — let both CDI and Spring wire the same classes. CDI registers them via `@Produces @DefaultBean` in `RuntimeBeans`. Spring registers them via `@Bean @ConditionalOnMissingBean` in the auto-configuration.

The pattern was already established. I was just applying it to the classes that had been missed.

`SituationRecompilerDispatch` stays CDI-only. It depends on `@Observes StartupEvent` for cold-start recovery and `@ObservesAsync SituationChangeEvent` for reactive dispatch — both are CDI event infrastructure with no Spring RAS bridge. Documenting it as CDI-only is accurate; a Spring equivalent would require a Spring RAS adapter module that doesn't exist yet.

## What the tests found

`NodeStepExecutor` is the per-node execution unit — human gating, approval lifecycle, lifecycle step execution, and all four step actions (provision, deprovision, suspend, resume). Testing all paths confirmed the design works as intended: human gating takes priority over approval checks, approval re-entry carries `PlanApproval` through the context, and `AlreadyConverged` propagates correctly as a first-class outcome.

The Spring integration test went from 5 tests that proved nothing about the runtime to 11 tests that verify every runtime bean resolves, every fallback is the correct type, and the routers and stores are wired. Removing the auto-config exclusion and having the context load cleanly is the real verification — individual bean checks are just documentation.

The `annotations/deployment` module has a pre-existing failure unrelated to this work — Quarkus bytecode recording can't serialise a `namespace` field on `GraphDescriptor`. That's a separate issue.
