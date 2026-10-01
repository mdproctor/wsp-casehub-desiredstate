# Decisions — #155 Spring Module Generators

## D1: Overall approach

**Choice:** Discovery classes + verify — extract domain logic into framework-neutral discovery classes per surface, reduce Spring auto-configs to ~15-line glue, add verify goal for drift detection
**Alternatives:**
- Verify-only + shared infra — less refactoring, catches drift, but modules stay ~100+ lines each
- Template DSL generator — YAML-descriptor-driven generation; highest long-term ROI but over-engineered for 4 surfaces
- Close as won't-do — accept hand-written code; misses the opportunity to eliminate duplication and enable future generation
**Rationale:** The current spring-generator scans @Produces methods — these modules use SmartInitializingSingleton with dynamic registerBean(). Rather than forcing the generator to handle a fundamentally different pattern, we extract domain logic into framework-neutral discovery classes that both Quarkus @Recorder and Spring auto-configs can share. This makes Spring modules trivially simple (~15 lines), makes the domain logic independently testable, and enables future generation of the trivial glue code.
**Trade-offs:** Requires refactoring existing working code. The discovery classes add a new abstraction layer between factories and framework wiring.
**Sources:** annotations/spring, yaml/spring, plugin/spring, ts-dsl/spring auto-configs; GoalCompilerFactory, YamlGoalCompilerFactory, TsGoalCompilerFactory, FaultPolicyFactory; spring-generator/JandexProducerScanner, AbstractVerifyMojo
**Exploration:** deep-analysis
**Status:** captured

## D2: Discovery class location

**Choice:** In each runtime module — AnnotationsDiscovery in annotations/runtime, YamlDiscovery in yaml/runtime, etc.
**Alternatives:**
- New shared module — cleaner dependency but adds a module for 4 small classes
- In each spring module — simpler but prevents Quarkus @Recorder from sharing the logic
**Rationale:** Colocated with the factories they call. Spring modules already depend on these runtime modules. Enables Quarkus @Recorder to also delegate to the same discovery class.
**Trade-offs:** Discovery classes in runtime modules have access to Jandex but not Spring — they must be framework-neutral by construction.
**Sources:** Module structure in CLAUDE.md
**Exploration:** quick
**Depends on:** D1 (overall approach)
**Status:** captured

## D3: BeanRegistration type

**Choice:** Shared `BeanRegistration` record in api module
**Alternatives:**
- Per discovery class — more independent but duplicated
**Rationale:** Consistent return type across all 4 discovery classes. Enables a shared Spring base class that iterates registrations and calls `registerBean()`.
**Trade-offs:** api module gains a type that's only used by discovery classes and Spring wiring. Not a domain concept per se.
**Sources:** api module structure
**Exploration:** quick
**Depends on:** D1 (overall approach)
**Status:** captured

## D4: Verify goal implementation

**Choice:** Extend `AbstractVerifyMojo` from platform's generator-common
**Alternatives:**
- Standalone verifier — self-contained but duplicates drift-detection framework
**Rationale:** Reuses the platform's existing drift-detection framework. Custom `collectSourceTypes()` scans discovery class return types, `collectTargetTypes()` scans Spring `registerBean()` calls.
**Trade-offs:** Adds dependency on generator-common from desiredstate's verify module.
**Sources:** spring-generator/SpringVerifyMojo, generator-common/AbstractVerifyMojo
**Exploration:** quick
**Depends on:** D1 (overall approach)
**Status:** captured
