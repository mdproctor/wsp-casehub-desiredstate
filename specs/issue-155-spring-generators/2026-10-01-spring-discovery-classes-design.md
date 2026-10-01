# Spring Discovery Classes — Design Spec

**Issue:** #155 — Spring: yaml/annotations/plugin/ts-dsl Spring modules could use generators
**Branch:** issue-155-spring-generators
**Date:** 2026-10-01

## Problem

Four desiredstate Spring auto-config modules (annotations/spring, yaml/spring, plugin/spring, ts-dsl/spring) are hand-written, each 75–207 lines. They share duplicated infrastructure (Jandex index loading, `@NodeTypeId` scanning, classpath resource discovery) and contain domain-specific factory invocations that are identical to what the Quarkus `@Recorder` classes call.

The platform `spring-generator` cannot handle these modules — it scans `@Produces` methods in CDI classes, but these modules use `SmartInitializingSingleton` with dynamic `GenericApplicationContext.registerBean()`.

## Solution

Extract domain-specific discovery + factory logic into **framework-neutral discovery classes** in each runtime module. Spring auto-configs become trivial glue (~15 lines each). Add a **verify goal** extending `AbstractVerifyMojo` for drift detection. File a follow-up issue for **generating the glue code** from the discovery classes.

## Architecture

### Layer structure

```
┌─────────────────────────────────────────────────────┐
│  api/                                                │
│  BeanRegistration record (shared return type)        │
├─────────────────────────────────────────────────────┤
│  annotations/runtime/  yaml/runtime/  ts-dsl/runtime/  plugin/runtime/ │
│  AnnotationsDiscovery  YamlDiscovery  TsDslDiscovery   PluginDiscovery │
│  (calls factories, returns List<BeanRegistration>)                     │
├─────────────────────────────────────────────────────┤
│  annotations/spring/   yaml/spring/   ts-dsl/spring/   plugin/spring/  │
│  ~15 lines: load index → discovery.discover() → registerBean()          │
├─────────────────────────────────────────────────────┤
│  verify module (extends AbstractVerifyMojo)           │
│  Scans discovery return types vs Spring registerBean  │
└─────────────────────────────────────────────────────┘
```

### BeanRegistration (api module)

```java
package io.casehub.desiredstate.api;

public record BeanRegistration(String name, Class<?> type, Object instance) {}
```

Simple value type. The discovery classes return `List<BeanRegistration>`. Spring auto-configs iterate and call `registerBean()` for each.

### Discovery classes (one per surface)

Each discovery class lives in its surface's runtime module, alongside the factory it calls.

**AnnotationsDiscovery** (`annotations/runtime/`):
- Input: `IndexView` (Jandex composite index)
- Calls: `DescriptorScanner.scanGraphs()` → `GoalCompilerFactory.create()` per graph
- Calls: `DescriptorScanner.scanFaultPolicies()` → `FaultPolicyFactory.create()` per policy
- Returns: `List<BeanRegistration>` with GoalCompiler and ThresholdFaultPolicy beans

**YamlDiscovery** (`yaml/runtime/`):
- Input: `IndexView`, `ClassLoader` (for resource scanning)
- Calls: scans `@NodeTypeId` for type registry, discovers YAML graphs + modules from classpath
- Calls: `YamlGoalCompilerFactory.create()` or `createLifecycle()` per graph
- Calls: `YamlFaultPolicyBuilder.build()` per fault policy
- Returns: `List<BeanRegistration>` with GoalCompiler and FaultPolicy beans

**TsDslDiscovery** (`ts-dsl/runtime/`):
- Input: `IndexView`, `ClassLoader`
- Calls: scans `@NodeTypeId`, discovers `.ds.json` files from classpath
- Calls: `TsGoalCompilerFactory.create()` or `createLifecycle()` per envelope
- Returns: `List<BeanRegistration>`

**PluginDiscovery** (`plugin/runtime/`):
- Input: `ClassLoader`
- Calls: discovers plugin YAML from `META-INF/desiredstate/plugins/*.yaml`
- Calls: `PluginParser.parse()` + builds `PluginDescriptor` per plugin
- Returns: `List<BeanRegistration>`

### SpringJandexSupport (shared utility)

```java
package io.casehub.desiredstate.runtime.spring;

public final class SpringJandexSupport {
    public static IndexView loadCompositeIndex() { ... }
    public static Map<String, String> scanNodeTypes(IndexView index) { ... }
}
```

Extracted from the duplicated code in 3 of 4 existing Spring modules. Lives in `runtime-spring/` since it uses Spring's classpath but the result (`IndexView`, `Map`) is framework-neutral.

### Simplified Spring auto-configs

After refactoring, each auto-config follows this pattern:

```java
@AutoConfiguration
@ConditionalOnClass(GoalCompilerFactory.class)
public class DesiredStateAnnotationsAutoConfiguration implements SmartInitializingSingleton {
    private final GenericApplicationContext context;

    public DesiredStateAnnotationsAutoConfiguration(GenericApplicationContext ctx) {
        this.context = ctx;
    }

    @Override
    public void afterSingletonsInstantiated() {
        IndexView index = SpringJandexSupport.loadCompositeIndex();
        new AnnotationsDiscovery().discover(index)
            .forEach(reg -> context.registerBean(reg.name(), reg.type(), reg::instance));
    }
}
```

~15 lines. The entire domain logic is in the discovery class.

### Verify goal

New Maven module: `verify/` (or added to an existing module). Extends `AbstractVerifyMojo` from platform's `generator-common`:

- `collectSourceTypes()` — scans discovery class methods' return types (via Jandex on the runtime modules, looking for methods returning `List<BeanRegistration>` and examining what `BeanRegistration.type()` values they produce)
- `collectTargetTypes()` — scans Spring module `registerBean()` calls (via `ManualBeanScanner` or a custom regex scanner on the Spring source)

Reports drift when a discovery class registers a bean type that the Spring module doesn't cover.

### Follow-up: Glue code generation

After discovery classes land, the Spring auto-configs are ~15 lines of mechanical glue following a fixed pattern. A follow-up issue should create a generator (or extend `spring-generator`) that:

1. Scans for classes with a `discover(...)` method returning `List<BeanRegistration>`
2. Infers the `@ConditionalOnClass` type from the discovery class's package/imports
3. Generates the `SmartInitializingSingleton` glue with the correct discovery method call

This is trivial to generate because the discovery classes have a uniform interface.

## Scope

**In scope — this issue (#155):**
- `BeanRegistration` record in api module
- 4 discovery classes (one per surface runtime module)
- `SpringJandexSupport` shared utility in runtime-spring
- Refactor 4 Spring auto-configs to use discovery classes
- Verify goal extending `AbstractVerifyMojo`
- All existing tests must pass (no behavior change)

**Follow-up issue (filed, not implemented here):**
- Generator for the ~15-line glue code from discovery classes

**Not in scope:**
- Refactoring Quarkus `@Recorder` classes to use discovery classes (desirable but separate)
- Changes to the platform `spring-generator` itself
- `runtime-spring/DesiredStateRuntimeAutoConfiguration` — noted as "CDI bridging too complex," stays hand-written

## Testing

- Unit tests for each discovery class (framework-neutral — no Spring context needed)
- Existing Spring auto-config behavior must be preserved (regression)
- Verify goal integration test (introduce deliberate drift, confirm detection)

## References

- `annotations/spring/DesiredStateAnnotationsAutoConfiguration.java` — 75 lines, current hand-written
- `yaml/spring/DesiredStateYamlAutoConfiguration.java` — 207 lines, current hand-written
- `plugin/spring/DesiredStatePluginAutoConfiguration.java` — 82 lines, current hand-written
- `ts-dsl/spring/DesiredStateTsDslAutoConfiguration.java` — 148 lines, current hand-written
- `spring-generator/JandexProducerScanner.java` — current generator scanner (scans @Produces)
- `spring-generator/SpringVerifyMojo.java` — existing verify goal pattern
- `generator-common/AbstractVerifyMojo.java` — verify framework extension point
- `GoalCompilerFactory`, `YamlGoalCompilerFactory`, `TsGoalCompilerFactory`, `FaultPolicyFactory` — framework-neutral factories
- `DesiredStateGraphRecorder`, `YamlGraphRecorder`, `TsGraphRecorder` — Quarkus recorders wrapping factories
- Blog: 2026-09-14 "The Epic That Shrank" — context on spring-generator design
