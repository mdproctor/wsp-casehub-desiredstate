# Brownfield Discovery — Scan Environments and Generate Desired-State YAML

**Epic:** casehubio/casehub-desiredstate#221
**Date:** 2026-11-04
**Status:** Draft

## Problem

The desired-state runtime requires operators to write YAML before the system can manage an
environment. For greenfield deployments this is natural — the operator declares what should
exist and the system provisions it. For brownfield environments (existing infrastructure
provisioned outside CaseHub), the operator must discover every resource ID, type, and
configuration manually and transcribe it into YAML before CaseHub can adopt the environment.

The import path (`ImportCapable`) reduces this to writing YAML with `import: "i-abc123"` per
resource, but the operator still needs to know every resource ID upfront. For a staging
environment with 30 EC2 instances, 5 RDS databases, and 12 S3 buckets, this is prohibitive.

## Solution

Invert the normal desired-state flow. Instead of YAML → graph → reconcile, the system
enumerates existing resources, builds a graph, and serializes it as YAML for operator review.

```
Normal:    YAML → GoalCompiler → DesiredStateGraph → reconcile vs actual
Discovery: enumerate actual → build graph → serialize YAML → operator reviews → adopt
```

First reconciliation after adoption makes no changes — the system treats the discovered state
as the baseline.

## Architecture

Five components across four modules:

| Component | Module | Issue |
|-----------|--------|-------|
| `DiscoveryCapable` mixin + `DiscoveredResource` | `bridge/api` | #222 |
| `DiscoveryAdapter` SPI + `DiscoveryAdapterRouter` + `DiscoveredNode` | `api/` + `runtime-core/` | #223 |
| `YamlGraphSerializer` | `yaml/runtime` | #224 |
| `DiscoveryOrchestrator` + `DiscoveryRequest` + `DiscoveryResult` | `runtime-core/` | #225, #226 |
| CDI/Spring wiring | `runtime/`, `runtime-spring/`, `bridge/runtime` | #225 |

### Data flow

```
BridgeProvider (DiscoveryCapable)
    .discover(typeId, tenancyId) → List<DiscoveredResource>

BridgeDiscoveryAdapter (DiscoveryAdapter impl in bridge/runtime)
    translates DiscoveredResource → DiscoveredNode (state→spec filtering)
    
DiscoveryAdapterRouter (runtime-core/)
    aggregates multiple DiscoveryAdapter beans by NodeType

DiscoveryOrchestrator (runtime-core/)
    calls router.discover() → builds DesiredStateGraph
    returns DiscoveryResult(resources, graph)

Caller (CLI / CDI bean)
    passes resources to YamlGraphSerializer.serialize() → YAML string
    optionally calls orchestrator.adoptBaseline() → seeds ReconciliationStateStore
```

## Component Details

### 1. DiscoveryCapable — bridge/api (#222)

New mixin on `BridgeProvider`, following `ImportCapable`/`PlanCapable`/`SuspendCapable` pattern.

```java
package io.casehub.desiredstate.bridge;

import java.util.List;

public interface DiscoveryCapable {
    List<DiscoveredResource> discover(String typeId, String tenancyId);
}
```

```java
package io.casehub.desiredstate.bridge;

import java.util.Map;

public record DiscoveredResource(
    String resourceId,
    String typeId,
    Map<String, Object> state
) {}
```

Per-type enumeration — the caller iterates `supportedTypes()` and calls `discover()` for
each type. Returns raw cloud state per resource.

**Terraform implementation:** `TerraformBridgeProvider implements DiscoveryCapable`. The
Terraform provider protocol has no native list API. Enumeration delegates to a
`ResourceEnumerator` SPI:

```java
package io.casehub.desiredstate.bridge.terraform;

import java.util.List;

public interface ResourceEnumerator {
    List<String> enumerate(String resourceType, String tenancyId);
}
```

Cloud-specific deployments provide implementations (e.g. AWS SDK `describe-instances`). The
Terraform bridge imports each discovered ID via `TerraformProtocol.importResource()` to
hydrate state. This keeps the Terraform bridge generic — no AWS/GCP/Azure SDK dependencies
in the bridge module itself.

**Ansible implementation:** `AnsibleBridgeProvider implements DiscoveryCapable`. Ansible's
info/facts modules (`ec2_instance_info`, `rds_instance_info`) natively enumerate resources.
The `discover()` call runs the appropriate info module (configured per type in the manifest's
`ansible:` block) and parses JSON output.

**Bridge manifest extension:** Bridge manifests gain an optional `discovery:` block per type:

```yaml
bridge:
  protocol: mock-cloud
  types:
    - id: "cloud:vm"
      stateMapping:
        running: PRESENT
      discovery:
        enabled: true
        specFields: [name, instance_type, ami, subnet_id]
```

`specFields` declares which state fields map to spec (configurable) fields. Fields not
listed are treated as computed/read-only and excluded from generated YAML. If omitted, all
non-computed fields are included.

### 2. DiscoveryAdapter — api/ + runtime-core/ (#223)

New SPI in `api/`, symmetric with `ActualStateAdapter`.

```java
package io.casehub.desiredstate.api;

import java.util.List;
import java.util.Set;

public interface DiscoveryAdapter {
    Set<NodeType> handledTypes();
    List<DiscoveredNode> discover(Set<NodeType> types, String tenancyId);
}
```

```java
package io.casehub.desiredstate.api;

import java.util.Map;

public record DiscoveredNode(
    String resourceId,
    NodeType type,
    Map<String, Object> state,
    Map<String, Object> spec
) {}
```

`state` carries the full cloud state (computed fields, IDs, timestamps). `spec` carries only
the configurable fields that belong in YAML — the subset the operator would write by hand.

`DiscoveredNode` provides a `toNodeSpec()` convenience method that wraps `type` and `spec`
into a `MapNodeSpec` — a simple `NodeSpec` implementation in `api/` backed by
`Map<String, Object>`. This avoids coupling the orchestrator (runtime-core/) to bridge-specific
types like `BridgeNodeSpec`.

```java
package io.casehub.desiredstate.api;

import java.util.Map;

public record MapNodeSpec(NodeType nodeType, Map<String, Object> fields)
        implements NodeSpec {
    @Override
    public NodeType nodeType() { return nodeType; }
}
```

**DiscoveryAdapterRouter** in `runtime-core/`:

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.DiscoveredNode;
import io.casehub.desiredstate.api.DiscoveryAdapter;
import io.casehub.desiredstate.api.NodeType;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;

public class DefaultDiscoveryAdapterRouter {

    private final Map<NodeType, DiscoveryAdapter> routing;

    public DefaultDiscoveryAdapterRouter(List<DiscoveryAdapter> adapters) {
        this.routing = new HashMap<>();
        for (DiscoveryAdapter adapter : adapters) {
            for (NodeType type : adapter.handledTypes()) {
                if (routing.containsKey(type)) {
                    throw new IllegalStateException(
                        "Duplicate DiscoveryAdapter for type: " + type);
                }
                routing.put(type, adapter);
            }
        }
    }

    public List<DiscoveredNode> discover(Set<NodeType> types, String tenancyId) {
        List<DiscoveredNode> results = new ArrayList<>();
        for (NodeType type : types) {
            DiscoveryAdapter adapter = routing.get(type);
            if (adapter != null) {
                results.addAll(adapter.discover(Set.of(type), tenancyId));
            }
        }
        return results;
    }

    public Set<NodeType> allHandledTypes() {
        return Set.copyOf(routing.keySet());
    }
}
```

**BridgeDiscoveryAdapter** in `bridge/runtime/`:

```java
package io.casehub.desiredstate.bridge.runtime;

import io.casehub.desiredstate.api.DiscoveredNode;
import io.casehub.desiredstate.api.DiscoveryAdapter;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.bridge.BridgeDescriptor;
import io.casehub.desiredstate.bridge.BridgeProvider;
import io.casehub.desiredstate.bridge.DiscoveryCapable;
import io.casehub.desiredstate.bridge.DiscoveredResource;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Set;

public class BridgeDiscoveryAdapter implements DiscoveryAdapter {

    private final BridgeDescriptor descriptor;
    private final DiscoveryCapable provider;

    public BridgeDiscoveryAdapter(BridgeDescriptor descriptor,
                                   BridgeProvider provider) {
        if (!(provider instanceof DiscoveryCapable dc)) {
            throw new IllegalArgumentException(
                "BridgeProvider must implement DiscoveryCapable");
        }
        this.descriptor = descriptor;
        this.provider = dc;
    }

    @Override
    public Set<NodeType> handledTypes() {
        return descriptor.handledTypes();
    }

    @Override
    public List<DiscoveredNode> discover(Set<NodeType> types, String tenancyId) {
        List<DiscoveredNode> nodes = new ArrayList<>();
        for (NodeType type : types) {
            if (!descriptor.handledTypes().contains(type)) continue;
            var typeDesc = descriptor.typeDescriptors().get(type.value());
            if (typeDesc == null) continue;

            List<DiscoveredResource> resources =
                provider.discover(type.value(), tenancyId);
            for (DiscoveredResource r : resources) {
                Map<String, Object> spec = filterSpecFields(r.state(), typeDesc);
                nodes.add(new DiscoveredNode(r.resourceId(), type, r.state(), spec));
            }
        }
        return nodes;
    }

    private Map<String, Object> filterSpecFields(
            Map<String, Object> state,
            io.casehub.desiredstate.bridge.BridgeTypeDescriptor typeDesc) {
        // Filter to spec-worthy fields using discovery config from manifest
        // Default: include all fields from state
        return Map.copyOf(state);
    }
}
```

CDI wiring: `CdiDiscoveryAdapterRouter` in `runtime/`, injecting `Instance<DiscoveryAdapter>`.
`BridgeDiscovery` in `bridge/runtime/` registers `BridgeDiscoveryAdapter` beans alongside
`BridgeProvisioner` and `BridgeActualStateAdapter` when the provider implements
`DiscoveryCapable`.

Spring wiring: `DesiredStateAutoConfiguration` in `runtime-spring/` produces
`DefaultDiscoveryAdapterRouter` from `List<DiscoveryAdapter>`.

### 3. YamlGraphSerializer — yaml/runtime (#224)

Standalone utility class — no framework dependencies.

```java
package io.casehub.desiredstate.yaml;

import io.casehub.desiredstate.api.DiscoveredNode;

import java.util.List;

public final class YamlGraphSerializer {

    private YamlGraphSerializer() {}

    public static String serialize(String namespace, String name,
                                    List<DiscoveredNode> resources) { ... }
}
```

Builds a `YamlGraph` model from discovered resources:

- `desiredState.namespace` = provided namespace
- `desiredState.name` = provided name
- Each `DiscoveredNode` → `YamlNode`:
  - `type` = `NodeType.value()`
  - `spec` = `DiscoveredNode.spec()`
  - Node ID = sanitized `resourceId` (lowercase, hyphens for special chars, max 63 chars)
- No dependencies inferred — flat graph. Operators add dependencies during review.
- No fault policy, rules, or invariants — operator adds during review.
- Serializes via Jackson `YAMLFactory` with `DumperOptions` for readable output (block style,
  sorted keys, no anchors).

**Output format:**

```yaml
desiredState:
  namespace: staging
  name: discovered-infrastructure

nodes:
  i-abc123:
    type: "cloud:vm"
    spec:
      name: web-server-1
      instance_type: t3.medium
      ami: ami-0abcdef1234567890
      subnet_id: subnet-12345

  i-def456:
    type: "cloud:vm"
    spec:
      name: web-server-2
      instance_type: t3.medium
      ami: ami-0abcdef1234567890
      subnet_id: subnet-12345

  mydb:
    type: "cloud:rds"
    spec:
      engine: postgres
      instance_class: db.r5.large
      storage_gb: 100
```

### 4. DiscoveryOrchestrator — runtime-core/ (#225, #226)

Composes the discovery pipeline.

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.Dependency;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.DesiredStateGraphFactory;
import io.casehub.desiredstate.api.DiscoveredNode;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeType;
import io.casehub.desiredstate.api.ReconciliationStateStore;

import java.util.List;
import java.util.Set;

public class DiscoveryOrchestrator {

    private final DefaultDiscoveryAdapterRouter router;
    private final DesiredStateGraphFactory graphFactory;
    private final ReconciliationStateStore stateStore;

    public DiscoveryOrchestrator(DefaultDiscoveryAdapterRouter router,
                                  DesiredStateGraphFactory graphFactory,
                                  ReconciliationStateStore stateStore) {
        this.router = router;
        this.graphFactory = graphFactory;
        this.stateStore = stateStore;
    }

    public DiscoveryResult discover(DiscoveryRequest request) {
        Set<NodeType> types = request.types().isEmpty()
            ? router.allHandledTypes()
            : request.types();
        List<DiscoveredNode> resources = router.discover(types, request.tenancyId());
        DesiredStateGraph graph = buildGraph(resources);
        return new DiscoveryResult(resources, graph);
    }

    public void adoptBaseline(String tenancyId, DesiredStateGraph graph) {
        stateStore.store(tenancyId, graph);
    }

    private DesiredStateGraph buildGraph(List<DiscoveredNode> resources) {
        List<DesiredNode> nodes = resources.stream()
            .map(this::toDesiredNode)
            .toList();
        return graphFactory.of(nodes, List.of());
    }

    private DesiredNode toDesiredNode(DiscoveredNode discovered) {
        return new DesiredNode(
            NodeId.of(sanitizeId(discovered.resourceId())),
            discovered.toNodeSpec());
    }

    private String sanitizeId(String resourceId) {
        return resourceId.toLowerCase()
            .replaceAll("[^a-z0-9-]", "-")
            .replaceAll("-+", "-")
            .replaceFirst("^-", "")
            .replaceFirst("-$", "");
    }
}
```

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.NodeType;
import java.util.Set;

public record DiscoveryRequest(
    Set<NodeType> types,
    String tenancyId,
    String namespace,
    String name
) {
    public DiscoveryRequest {
        if (types == null) types = Set.of();
    }
}
```

```java
package io.casehub.desiredstate.runtime;

import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.DiscoveredNode;
import java.util.List;

public record DiscoveryResult(
    List<DiscoveredNode> resources,
    DesiredStateGraph graph
) {}
```

**Adoption flow (#226):**

1. Operator runs discovery → reviews generated YAML → commits to project
2. Operator calls `adoptBaseline(tenancyId, graph)` (via CLI or API)
3. `ReconciliationStateStore.store()` persists the graph as last-reconciled state
4. First reconciliation: `TransitionPlanner` loads last-reconciled state, diffs against
   desired graph (identical), plans zero transitions
5. Subsequent reconciliations detect drift normally

### 5. CDI/Spring wiring (#225)

**CDI (runtime/):**

`RuntimeBeans` gains `@Produces` methods for:
- `DefaultDiscoveryAdapterRouter` (from `Instance<DiscoveryAdapter>`)
- `DiscoveryOrchestrator` (from router + `DesiredStateGraphFactory` + `ReconciliationStateStore`)

**Spring (runtime-spring/):**

`DesiredStateAutoConfiguration` gains `@Bean` methods for the same types,
using `List<DiscoveryAdapter>` injection.

**Bridge (bridge/runtime/):**

`BridgeDiscovery.discoverAll()` gains `BridgeDiscoveryAdapter` registration alongside
existing `BridgeProvisioner` and `BridgeActualStateAdapter` — conditional on
`provider instanceof DiscoveryCapable`.

## Module dependency additions

```
bridge/api  →  (no new deps — DiscoveredResource is pure Java)
api/        →  (no new deps — DiscoveredNode is pure Java)
runtime-core/ depends on api/ (existing)
yaml/runtime depends on api/ (existing — needs DiscoveredNode for serializer)
bridge/runtime depends on bridge/api + api/ (existing)
runtime/ depends on runtime-core/ + api/ (existing)
runtime-spring/ depends on runtime-core/ + api/ (existing)
```

No new module dependencies. All new types fit within existing dependency edges.

## Scope boundaries

**In scope:**
- `DiscoveryCapable` mixin and `DiscoveredResource` record
- `DiscoveryAdapter` SPI and `DiscoveredNode` record
- `DefaultDiscoveryAdapterRouter`
- `BridgeDiscoveryAdapter`
- `YamlGraphSerializer`
- `DiscoveryOrchestrator` with `DiscoveryRequest`/`DiscoveryResult`
- Adoption baseline via `ReconciliationStateStore`
- CDI and Spring wiring
- `MockCloudProvider` implementing `DiscoveryCapable` in bridge-cloud example
- Bridge manifest `discovery:` extension for `specFields`

**Out of scope (future work):**
- CLI command (shell/Quarkus CLI) — entry point exists as API; CLI is a consumer concern
- Dependency inference from discovered state (e.g. inferring subnet→VPC edges)
- Incremental re-discovery (only discover resources not already in the graph)
- Cloud SDK `ResourceEnumerator` implementations — these are deployment-specific
- Terraform state file parsing as a discovery source
- CMDB integration as a discovery source
- UI for reviewing/editing discovered YAML

## Testing strategy

| Test | Type | Module |
|------|------|--------|
| `YamlGraphSerializerTest` | Unit | `yaml/runtime` |
| `DiscoveryOrchestratorTest` | Unit | `runtime-core/` |
| `BridgeDiscoveryAdapterTest` | Unit | `bridge/runtime` |
| `DefaultDiscoveryAdapterRouterTest` | Unit | `runtime-core/` |
| `DiscoveryIntegrationTest` | Integration | `examples/bridge-cloud/` |

- **Serializer tests:** round-trip — serialize resources, parse back via Jackson, verify
  `YamlGraph` structure matches input. Edge cases: empty resource list, special characters in
  resource IDs, large spec maps.
- **Orchestrator tests:** mock `DiscoveryAdapter`, verify graph building (nodes present, no
  deps), verify `adoptBaseline()` calls `ReconciliationStateStore.store()`.
- **Bridge adapter tests:** mock `DiscoveryCapable` provider, verify `DiscoveredResource` →
  `DiscoveredNode` translation with spec field filtering.
- **Router tests:** multiple adapters with disjoint types, duplicate type detection.
- **Integration test:** `MockCloudProvider implements DiscoveryCapable` → full pipeline:
  discover → serialize → parse → adopt → reconcile (zero transitions).

## References

- `bridge/api/ImportCapable.java` — existing per-resource import pattern
- `bridge/api/BridgeProvider.java` — mixin pattern (PlanCapable, SuspendCapable)
- `bridge/terraform/SchemaTranslator.java` — type discovery from provider schema
- `bridge/terraform/TerraformBridgeProvider.java` — ImportCapable implementation
- `bridge/runtime/BridgeActualStateAdapter.java` — per-type routing + state mapping
- `api/ActualStateAdapter.java` — symmetric SPI pattern
- `runtime-core/DefaultActualStateAdapterRouter.java` — router pattern
- `yaml/runtime/model/YamlGraph.java`, `YamlNode.java` — serialization target model
- `examples/pipeline-yaml/medallion-pipeline.yaml` — example YAML format
- `examples/bridge-cloud/mock-cloud.yaml` — bridge manifest format
- casehubio/casehub-desiredstate#221 — epic
- casehubio/casehub-desiredstate#222–#226 — sub-issues
