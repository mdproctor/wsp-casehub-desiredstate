# Class-Based @DesiredNode — Design Spec

**Date:** 2026-08-22
**Issue:** casehubio/casehub-desiredstate#105
**Status:** Draft

## Motivation

The interface-based `@DesiredState` model (#102) requires the entire graph in one file.
For graphs assembled from multiple modules (e.g., base infrastructure + per-tenant extensions),
nodes need to be declared as separate classes discovered across the classpath — the same pattern
CDI uses for bean discovery.

The class-based model complements the interface model:
- **Interface model** (`@DesiredState` + `@Node`): centralized graph — entire topology in one file
- **Class model** (`@DesiredNode`): distributed graph — individual nodes scattered across modules

Both produce the same `GraphDescriptor` → `DesiredStateGraphRecorder` pipeline output. When they
share the same (namespace, name), their nodes merge into a single graph.

---

## Part 1: New Annotation — @DesiredNode

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface DesiredNode {
    String namespace() default "";
    String name() default "";
    String id();
}
```

- `namespace` + `name`: graph identity — matches `@DesiredState(namespace, name)` for merge
- `id`: **mandatory** — node identity (persistence-critical: FaultCountStore, PendingApproval,
  CloudEvents, ActualState all key on node ID). No derivation from class name — class renames
  must not silently change node identity.

### Annotated class constraints

The annotated class must:
1. Implement `NodeSpec` (directly or via supertype)
2. Have a public no-arg constructor (for recorder instantiation)
3. Be concrete (not abstract, not an interface)

### Programming model

```java
@DesiredNode(namespace = "infra", name = "multi-zone", id = "load-balancer")
@DependsOn(nodes = DnsRecord.class)
public class LoadBalancer implements NodeSpec {
    @Override
    public NodeType nodeType() { return NodeType.of("lb"); }

    @Override
    public HumanGating humanGating() { return HumanGating.PROVISION_ONLY; }
}
```

### HumanGating

No `humanGating` attribute on `@DesiredNode`. The class IS the NodeSpec — override
`NodeSpec.humanGating()` directly. This differs from `@Node(humanGating = ...)` because
`@Node` methods return a NodeSpec *instance* where the gating is external to the spec.

### One class = one node

No factory method pattern. Each `@DesiredNode` class declares exactly one node.
For multiple nodes of the same `NodeSpec` type, use the interface model (`@DesiredState`
with multiple `@Node` methods returning the same type).

---

## Part 2: @DependsOn Extended for Type-Safe References

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
public @interface DependsOn {
    String[] value() default {};
    Class<? extends NodeSpec>[] nodes() default {};
}
```

Changes from current:
1. `@Target` expanded: adds `ElementType.TYPE` alongside `ElementType.METHOD`
2. New attribute: `Class<? extends NodeSpec>[] nodes()` — type-safe, compile-time checked
3. Both attributes can be used together on the same annotation

### Resolution

The processor resolves `nodes` class references to node IDs at build time:
- Find the `@DesiredNode` annotation on the referenced class
- Extract the `id` attribute → use as the dependency target
- Error if the referenced class has no `@DesiredNode` annotation

### Cross-model references

| Source | Target | Reference form |
|--------|--------|---------------|
| Class → Class | `@DependsOn(nodes = DnsRecord.class)` | Type-safe |
| Class → Interface node | `@DependsOn("csv-source")` | String ID |
| Interface node → Class | `@DependsOn("load-balancer")` | String ID |
| Interface node → Interface node | `@DependsOn("csv-source")` | String ID (unchanged) |

Cross-model string references are validated across all nodes in the merged graph.

---

## Part 3: Sealed NodeDescriptor

Refactor `NodeDescriptor` from a flat record into a sealed interface with two variants:

```java
public sealed interface NodeDescriptor
        permits NodeDescriptor.InterfaceNode, NodeDescriptor.ClassNode {

    String id();

    record InterfaceNode(String id, String methodName, String returnTypeName,
                         HumanGating humanGating) implements NodeDescriptor {}

    record ClassNode(String id, String className) implements NodeDescriptor {}
}
```

`ClassNode` carries only `id` and `className`:
- No `humanGating` field — derived from `NodeSpec.humanGating()` at runtime
- No `returnTypeName` — the class IS the NodeSpec (not a method returning one)

`GraphDescriptor` keeps a single `List<NodeDescriptor> nodes`. The recorder uses
pattern matching for exhaustive handling:

```java
switch (nd) {
    case NodeDescriptor.InterfaceNode in -> {
        Method method = implClass.getMethod(in.methodName());
        NodeSpec spec = (NodeSpec) method.invoke(instance);
        nodes.add(new DesiredNode(NodeId.of(in.id()), spec, in.humanGating()));
    }
    case NodeDescriptor.ClassNode cn -> {
        Class<?> nodeClass = classLoader.loadClass(cn.className());
        NodeSpec spec = (NodeSpec) nodeClass.getDeclaredConstructor().newInstance();
        nodes.add(new DesiredNode(NodeId.of(cn.id()), spec, HumanGating.NONE));
    }
}
```

For `ClassNode`, `humanGating` in the `DesiredNode` record is `NONE` — the merge with
`NodeSpec.humanGating()` happens in `DesiredNode.requiresHuman(StepAction)` which
already ORs `humanGating` with `spec.humanGating()`.

---

## Part 4: @FaultPolicyDef on @DesiredNode Classes

`@FaultPolicyDef` already targets `{ElementType.TYPE, ElementType.METHOD}`. On a
`@DesiredNode` class, it scopes the policy to that node's type:

```java
@DesiredNode(namespace = "infra", name = "zones", id = "load-balancer")
@FaultPolicyDef(
    faultTypes = {"PROVISION_FAILED"},
    tiers = {
        @Tier(threshold = 3, review = "createReview")
    }
)
public class LoadBalancer implements NodeSpec {
    @Override
    public NodeType nodeType() { return NodeType.of("lb"); }

    public AiReviewSpec createReview(FaultEvent event, DesiredStateGraph graph) {
        return new AiReviewSpec(event.node(), event.detail());
    }
}
```

- `nodeTypes` is **auto-inferred** from the class's `nodeType()` — no need to specify
- `@Tier(review = "...")` references a method on the same class (not an interface)
- Each `@DesiredNode` class has exactly one `nodeType()`, so scoping is unambiguous

### Processor changes

When collecting fault policies from `@DesiredNode` classes:
1. Scan class-level `@FaultPolicyDef` / `@FaultPolicies`
2. For each, if `nodeTypes` is empty, mark for auto-inference at runtime
3. Validate review methods exist on the class with correct signature
4. Create `FaultPolicyDescriptor` with `sourceClassName` for the recorder

### Recorder changes

When creating `ThresholdFaultPolicy` from a class-sourced descriptor:
1. Instantiate the `@DesiredNode` class (reuse the instance from node construction)
2. Probe `nodeType()` → auto-fill `nodeTypes` if empty
3. Find review methods on the class instance (not the interface impl)
4. Build `ThresholdFaultPolicy` as normal

---

## Part 5: Graph Merge Semantics

When `@DesiredNode` classes and a `@DesiredState` interface share the same
(namespace, name), the processor merges them into a single `GraphDescriptor`:

1. **Node merge:** Class-based `ClassNode` descriptors are appended to the interface's
   `InterfaceNode` descriptors. Duplicate ID detection spans both sets.
2. **Dependency merge:** String-based dependencies from both sources are combined.
   Class-based `nodes` references are resolved to string IDs and added to the same
   dependency list.
3. **Fault policy merge:** Policies from both sources are combined. Interface-level
   policies apply to matching node types from both sources.
4. **GoalCompiler:** The merged graph is wrapped in the same `GoalCompiler` bean.
   If the interface has a `@GoalMethod`, it receives the merged base graph.

### Class-only graphs

When `@DesiredNode` classes share a (namespace, name) but no `@DesiredState` interface
exists with that name:
- The processor creates a `GraphDescriptor` with `null` interfaceName and no implClassName
- No Gizmo impl class is generated (no interface to implement)
- The recorder creates a static `GoalCompiler` that ignores goals and returns the graph
- `@Customize`, `@GoalMethod`, and interface-level `@FaultPolicyDef` are unavailable

---

## Part 6: Build Extension Changes

### DesiredStateAnnotationsProcessor

New `@BuildStep` or expanded `generateDesiredStateGraphs`:

1. **Scan phase:**
   - Scan `@DesiredState` interfaces (existing)
   - Scan `@DesiredNode` classes (new)
   - Group class-based nodes by (namespace, name)

2. **Merge phase:**
   - For each (namespace, name) pair, merge interface and class-based nodes
   - For class-only graphs, create a GraphDescriptor without interface fields

3. **Resolve class references:**
   - For each `@DependsOn(nodes = ...)`, look up the target class's `@DesiredNode(id = ...)`
   - Convert to `DependencyDescriptor(from, to)` string pairs

4. **Register beans:**
   - One `GoalCompiler` per merged graph (unchanged)
   - `ThresholdFaultPolicy` beans from both sources (unchanged)

### AnnotationValidationStep

New validations:

| Check | Error message |
|-------|---------------|
| @DesiredNode on non-NodeSpec class | `@DesiredNode on 'Foo' which does not implement NodeSpec` |
| @DesiredNode on interface | `@DesiredNode on interface 'Bar' — use @DesiredState for interfaces` |
| @DesiredNode on abstract class | `@DesiredNode on abstract class 'Baz' — must be concrete` |
| Missing no-arg constructor | `@DesiredNode class 'Qux' must have a public no-arg constructor` |
| Duplicate node ID (cross-model) | `Duplicate node id 'lb' — declared on LoadBalancer and interface method lbNode` |
| @DependsOn(nodes) target missing @DesiredNode | `@DependsOn(nodes) on 'Vpc' references 'DnsRecord' which has no @DesiredNode annotation` |
| @DependsOn(nodes) target not NodeSpec | `@DependsOn(nodes) references 'NotASpec' which does not implement NodeSpec` |
| @FaultPolicyDef review method missing on class | `@Tier review 'createReview' not found on class LoadBalancer` |
| @FaultPolicyDef review method bad signature on class | `Review method 'createReview' on LoadBalancer must accept (FaultEvent, DesiredStateGraph)` |

Existing validations remain unchanged for the interface model.

---

## Part 7: GraphDescriptor Changes

```java
public record GraphDescriptor(
        String namespace,
        String name,
        String interfaceName,       // null for class-only graphs
        String implClassName,       // null for class-only graphs
        List<NodeDescriptor> nodes, // InterfaceNode + ClassNode interleaved
        List<DependencyDescriptor> dependencies,
        List<FaultPolicyDescriptor> faultPolicies,
        GoalMethodDescriptor goalMethod) {}
```

No structural changes — `interfaceName` and `implClassName` become nullable for class-only
graphs. `List<NodeDescriptor>` already handles both variants via the sealed interface.

### FaultPolicyDescriptor changes

Add `sourceClassName` for class-sourced fault policies (reviewer method lookup at runtime):

```java
public record FaultPolicyDescriptor(
        List<String> faultTypes,
        List<String> nodeTypes,
        List<String> ignoreTypes,
        String namespace,
        List<TierDescriptor> tiers,
        String sourceClassName) {}  // null for interface-sourced
```

Existing constructor compatibility: add a compact constructor that defaults `sourceClassName`
to `null`, or update all call sites. Since pre-release, update all call sites.

---

## Testing Strategy

### Unit tests (deployment/)

**ClassBasedNodeTest** — core class-based node functionality:
- `@DesiredNode` class → GoalCompiler bean → graph contains the node
- Node ID matches `@DesiredNode(id = ...)`
- NodeType from `nodeType()` method
- NodeSpec data preserved
- HumanGating from `NodeSpec.humanGating()` override

**ClassBasedDependencyTest** — type-safe dependencies:
- `@DependsOn(nodes = {A.class})` → Dependency(from, to) wired
- Mixed `@DependsOn(value = "x", nodes = {A.class})` → both dependencies wired
- `@DependsOn(nodes = ...)` on class, `@DependsOn("...")` on interface methods → all wired

**MergedGraphTest** — interface + class merge:
- `@DesiredState` interface + `@DesiredNode` classes with same (namespace, name) → single graph
- Duplicate node ID across models → build error
- Cross-model string references resolve

**ClassOnlyGraphTest** — no interface:
- Multiple `@DesiredNode` classes with same (namespace, name), no `@DesiredState` → graph
- GoalCompiler ignores goals, returns static graph

**ClassBasedFaultPolicyTest** — fault policy on classes:
- `@FaultPolicyDef` on `@DesiredNode` class → ThresholdFaultPolicy bean
- `nodeTypes` auto-inferred from `nodeType()`
- Review method on class invoked correctly

**ClassBasedValidationTest** — error cases:
- Each validation error from Part 6 verified with a negative-test class
- @DesiredNode on non-NodeSpec → error
- @DesiredNode on interface → error
- @DesiredNode on abstract → error
- @DependsOn(nodes) target missing @DesiredNode → error
- Missing no-arg constructor → error

### Sealed NodeDescriptor migration

Existing tests (`DesiredStateAnnotationsProcessorTest`, `FaultPolicyWiringTest`,
`GoalMethodCompositionTest`, `ValidationErrorTest`) must pass after `NodeDescriptor`
refactoring — the interface model is unchanged functionally.

---

## Migration Impact

| Component | Change |
|-----------|--------|
| `NodeDescriptor` | Record → sealed interface. `InterfaceNode` replaces the flat record. |
| `GraphDescriptor` | `interfaceName`/`implClassName` become nullable. `sourceClassName` added to `FaultPolicyDescriptor`. |
| `DesiredStateGraphRecorder` | Pattern matching on `NodeDescriptor` variants. Class instantiation for `ClassNode`. |
| `DesiredStateAnnotationsProcessor` | Expanded scan + merge logic. Class reference resolution. |
| `AnnotationValidationStep` | New validation rules for class-based nodes. Cross-model duplicate detection. |
| `@DependsOn` | `@Target` expanded. `nodes` attribute added. |

No changes to: `@DesiredState`, `@Node`, `@FaultPolicyDef`, `@Tier`, `@Customize`,
`@GoalMethod`, `@DesiredStateQualifier`, `DesiredNode` (api record), `NodeSpec`,
`GoalCompiler`, `ThresholdFaultPolicy`, or any runtime module.

---

## References

- [DesiredState.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DesiredState.java) — existing interface annotation
- [DependsOn.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/DependsOn.java) — current string-only dependency
- [NodeDescriptor.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/NodeDescriptor.java) — current flat record
- [DesiredStateAnnotationsProcessor.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/DesiredStateAnnotationsProcessor.java) — build extension
- [DesiredStateGraphRecorder.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/runtime/src/main/java/io/casehub/desiredstate/annotations/runtime/DesiredStateGraphRecorder.java) — runtime recorder
- [AnnotationValidationStep.java](/Users/mdproctor/claude/casehub/desiredstate/annotations/deployment/src/main/java/io/casehub/desiredstate/annotations/deployment/AnnotationValidationStep.java) — build-time validation
- [DesiredNode.java](/Users/mdproctor/claude/casehub/desiredstate/api/src/main/java/io/casehub/desiredstate/api/DesiredNode.java) — API record (unchanged)
- [NodeSpec.java](/Users/mdproctor/claude/casehub/desiredstate/api/src/main/java/io/casehub/desiredstate/api/NodeSpec.java) — NodeSpec interface (unchanged)
- [#102 annotations design spec](/Users/mdproctor/claude/casehub/desiredstate/docs/specs/issue-102-desiredstate-annotations/2026-08-20-desiredstate-annotations-design.md) — foundation spec
- decisions.md — 8 design decisions with rationale (D3, D7 revised after decision review)
