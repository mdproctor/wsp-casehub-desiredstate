# Design Decisions — #105 class-based @DesiredNode

## D1: @DependsOn gains type-safe Class references via dual attributes

**Choice:** Add `Class<? extends NodeSpec>[] nodes() default {}` to existing `@DependsOn` alongside `String[] value()`. Both can be used together. Processor resolves class refs to node IDs at build time.
**Alternatives:**
- Separate `@DependsOnNodes(Class...)` annotation — simpler per-annotation, but two annotations for the same concept
- Class-only `@DependsOn` for class targets — breaks cross-model references
**Rationale:** Single annotation, two reference forms. Class-to-class dependencies are type-safe and compile-time checked. Cross-model references (class → interface node, or vice versa) use the string form. Both merge into the same `DependencyDescriptor(from, to)` at build time.
**Trade-offs:** `@DependsOn` API is slightly more complex with two attributes. Users must know which form to use for cross-model vs same-model references.
**Prerequisites:** `@DependsOn` needs `@Target` expanded to include `TYPE`. Contributing modules must generate Jandex indexes. Validation pipeline restructured for cross-model reference resolution.
**Depends on:** D3 (class reference resolution uses the target class's explicit `id`)
**Sources:** DependsOn.java, DesiredStateAnnotationsProcessor.java, DependencyDescriptor.java
**Exploration:** quick
**Status:** captured

## D2: @DesiredNode uses separate namespace/name attributes

**Choice:** `@DesiredNode(namespace = "infra", name = "multi-zone")` — separate attributes matching `@DesiredState(namespace, name)`.
**Alternatives:**
- Single `graph = "infra:multi-zone"` string — compact but requires parsing, inconsistent with @DesiredState API
- Class reference `graph = MedallionPipeline.class` — type-safe but defeats cross-module composition
**Rationale:** Consistency with `@DesiredState`. Direct match for merge — the processor matches class-based nodes to interface-declared graphs by comparing (namespace, name) pairs with no parsing.
**Trade-offs:** Slightly more verbose than a single `graph` string.
**Sources:** DesiredState.java, DesiredStateQualifier.java
**Exploration:** quick
**Status:** captured

## D3: Explicit node ID required on @DesiredNode

**Choice:** `id` attribute is mandatory on `@DesiredNode`. No derivation from class name. `@DesiredNode(namespace = "infra", name = "zones", id = "load-balancer")`.
**Alternatives:**
- Derive from class name via kebab-case (`LoadBalancer` → `"load-balancer"`) — ergonomic but fragile: class renames silently change node identity
- Derivation as opt-in (`id = DesiredNode.DERIVED`) — best of both but adds API surface
**Rationale:** Node IDs are persistence-critical — FaultCountStore, PendingApproval lifecycle, CloudEvents, ActualState mapping all key on them. A class rename silently changing the derived ID would cause deprovision + reprovision of actual infrastructure. The interface model always requires explicit IDs (`@Node("csv-source")`). Consistency and safety argue for the same requirement.
**Trade-offs:** More verbose — every `@DesiredNode` must specify `id`. This is the safe default for identity-critical values.
**Sources:** MedallionPipeline.java (explicit node IDs), NodeId.java, FaultCountStore SPI, PendingApprovalHandler SPI, reviewer R1-03
**Exploration:** quick → revised after decision review (R1-03)
**Status:** revised

## D4: No factory pattern — interface model handles multi-instance

**Choice:** No factory method support. One class = one node. If multiple nodes of the same NodeSpec type are needed, use the interface model (`@DesiredState` with multiple `@Node` methods returning the same type).
**Alternatives:**
- Factory class with `@Produces` method — handles multi-instance and records from other modules. Adds complexity and a new annotation.
- `@Repeatable` @DesiredNode — same class, two identities, same spec data. Weird semantics.
**Rationale:** The two models complement each other: class model for cross-module composition (one class per node), interface model for multi-instance and full-graph declarations. No need to duplicate the interface model's multi-instance capability.
**Trade-offs:** Records in external modules that can't be annotated must be wrapped in a NodeSpec subclass, or declared via the interface model.
**Sources:** Issue #105 scope, MedallionPipeline.java
**Exploration:** quick
**Status:** captured

## D5: HumanGating via NodeSpec.humanGating() — no annotation attribute

**Choice:** No `humanGating` attribute on `@DesiredNode`. The class IS the NodeSpec — override `humanGating()` directly.
**Alternatives:**
- `@DesiredNode(humanGating = ...)` attribute — visible in annotation, consistent with @Node's attribute. But duplicates NodeSpec.humanGating().
- Both with merge — maximum flexibility but two places to configure.
**Rationale:** `@Node` needs the attribute because the method body returns a NodeSpec instance — the gating is external to the spec. `@DesiredNode` IS the NodeSpec — the gating is intrinsic. No duplication.
**Trade-offs:** Gating is less visible (in method override vs annotation attribute). But the class is right there.
**Sources:** NodeSpec.java (humanGating default), DesiredNode.java (merge logic), HumanGating.java
**Exploration:** quick
**Status:** captured

## D6: @FaultPolicyDef supported on @DesiredNode classes

**Choice:** `@FaultPolicyDef` on a `@DesiredNode` class scopes the policy to that node's type. Review methods (`@Tier(review = "...")`) reference methods on the same class. `nodeTypes` is auto-inferred from the class's `nodeType()`.
**Alternatives:**
- Interface-level only — simpler processor, but class-based nodes can't declare node-specific fault policies via annotations
- Standalone @FaultPolicyDef class — fully decoupled, but policies without visible graph context are confusing
**Rationale:** Consistent with method-level @FaultPolicyDef on @Node. Self-contained — each @DesiredNode class carries its own fault policy and review methods. Interface-level @FaultPolicyDef remains for cross-type policies.
**Trade-offs:** Processor must handle FaultPolicyDef on both class targets and interface targets. Review method validation differs (on the class vs on the interface). No multi-class conflict: each @DesiredNode class has exactly one nodeType(), so ThresholdFaultPolicy scoping is unambiguous — two classes with @FaultPolicyDef for the same faultType but different nodeTypes are independent policies.
**Depends on:** D4 (one class = one node, so nodeTypes can be auto-inferred)
**Sources:** FaultPolicyDef.java, ThresholdFaultPolicy.java, AnnotationValidationStep.java
**Exploration:** quick
**Status:** captured

## D7: Sealed NodeDescriptor — InterfaceNode | ClassNode

**Choice:** Refactor `NodeDescriptor` into a sealed interface with two record variants. Single `List<NodeDescriptor> nodes` in `GraphDescriptor`. Recorder uses pattern matching for exhaustive handling.
```java
sealed interface NodeDescriptor permits NodeDescriptor.InterfaceNode, NodeDescriptor.ClassNode {
    String id();
    record InterfaceNode(String id, String methodName, String returnTypeName,
                         HumanGating humanGating) implements NodeDescriptor {}
    record ClassNode(String id, String className) implements NodeDescriptor {}
}
```
**Alternatives:**
- Parallel lists (`List<NodeDescriptor> nodes` + `List<ClassNodeDescriptor> classNodes`) — additive but introduces unnecessary complexity: two iteration paths in the recorder, every consumer must handle both lists, future node forms add yet another list
- Separate pipeline — class-based nodes produce own GoalCompiler beans. Defeats "same graph" semantics.
**Rationale:** Pre-release platform — cost of refactoring is always worth paying for the right design. A single sealed interface gives exhaustive switch at compile time, natural interleaving, single loop in the recorder. Quarkus recorder serialization supports sealed record variants.
**Trade-offs:** Existing code using `NodeDescriptor` record must migrate to `NodeDescriptor.InterfaceNode`. Mechanical change — rename + add `implements NodeDescriptor`.
**Depends on:** D2 (namespace/name matching for merge)
**Sources:** GraphDescriptor.java, NodeDescriptor.java, DesiredStateGraphRecorder.java, reviewer R1-02
**Exploration:** quick → revised after decision review (R1-02)
**Status:** revised

## D8: Class-only graphs are static — graph-level annotations require @DesiredState interface

**Choice:** `@Customize`, `@GoalMethod`, and interface-level `@FaultPolicyDef` remain interface-only. For class-only graphs, add a `@DesiredState` interface (even with no @Node methods) to use these features.
**Alternatives:**
- Support @Customize/@GoalMethod on @DesiredNode classes — blurs node/graph distinction, confusing that a class is both a node and graph-level logic
**Rationale:** Clean separation: classes contribute nodes, interfaces contribute graph-level behavior. A class-only graph is inherently static — no composition, no graph-level customization via annotations. Programmatic FaultPolicy beans and GoalCompiler beans still work via CDI for class-only graphs.
**Trade-offs:** Pure class-only graphs cannot use @Customize/@GoalMethod without adding an interface. This is the intended design boundary — class-based composition is for distributed node discovery, not full graph control.
**Depends on:** D7 (GraphDescriptor supports null interfaceName for class-only graphs)
**Sources:** Customize.java, GoalMethod.java, DesiredStateGraphRecorder.createGoalCompiler()
**Exploration:** quick
**Status:** captured
