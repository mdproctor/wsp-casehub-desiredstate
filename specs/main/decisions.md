# Decisions — desiredstate#139 Core Extraction

## D1: Scope

**Choice:** Full scope — runtime-core + persistence-jpa-core + Spring adapters + annotations/spring-boot + yaml/spring-boot + ts-dsl/spring-boot + plugin/spring-boot
**Alternatives:**
- Runtime only — simpler but leaves Spring users without graph declaration surfaces
- Runtime + stubs — skeleton modules without implementation, defers real work
**Rationale:** The issue explicitly scopes all surfaces. The yaml/ts-dsl/plugin Spring adapters are easy (classpath resource discovery). Only annotations needs design work.
**Trade-offs:** Larger PR, more modules to maintain
**Sources:** casehubio/casehub-desiredstate#139
**Exploration:** quick
**Status:** captured

## D2: Annotation scanning in Spring

**Choice:** Jandex at startup — read META-INF/jandex.idx at Spring startup using the same descriptor extraction logic as the Quarkus deployment module
**Alternatives:**
- Spring ClassPathScanning — standard Spring but reimplements scan logic
- APT — zero startup cost but requires new build-time module
**Rationale:** Reuses existing Jandex plugin output (all modules already produce jandex.idx). Same descriptor records, zero duplicated logic. ~50ms startup cost is negligible.
**Trade-offs:** Runtime Jandex dependency in Spring auto-config module
**Sources:** annotations/deployment/DesiredStateAnnotationsProcessor.java, annotations/runtime/DesiredStateGraphRecorder.java
**Exploration:** quick
**Status:** captured

## D3: @DesiredState stub generation in Spring

**Choice:** JDK Proxy at startup — dynamic proxy for @DesiredState interfaces
**Alternatives:**
- Skip entirely — only support @DeclareNode in Spring, loses API parity
- ByteBuddy — more capable but adds runtime dependency
**Rationale:** The @DesiredState interface impl is purely declarative (no-arg ctor, methods return NodeSpec). JDK Proxy handles this trivially. No build tooling needed.
**Trade-offs:** Slightly different reflective behavior vs concrete Gizmo class; negligible in practice since the impl is only used for instance.getMethod() calls
**Sources:** annotations/deployment/DesiredStateAnnotationsProcessor.java (Gizmo generation)
**Exploration:** quick
**Status:** captured
