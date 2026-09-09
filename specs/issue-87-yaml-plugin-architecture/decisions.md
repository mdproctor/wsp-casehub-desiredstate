# Decisions — YAML Plugin Architecture (#87)

## D1: Scope — plugin schema + interpreter first

**Choice:** Design the 5-section YAML plugin format and the runtime that maps it to NodeProvisioner/ActualStateAdapter/FaultPolicy SPIs. Other subsystems (Java extension primitives, 10 concrete plugins, full CBR/RAS integration) follow as separate specs.
**Alternatives:**
- Java extension primitives first — building blocks before consumers, but the plugin schema defines what building blocks are needed
- Full architecture overview — complete picture but too large for one spec (XL/High issue)
- Bottom-up from one concrete plugin — discovers abstractions but risks over-fitting to one domain
**Rationale:** Everything else depends on the plugin schema — the primitives, the concrete plugins, and the CaseHub integrations all consume this format.
**Trade-offs:** Deferring the concrete plugins means the schema is validated against sketched examples, not real-world usage. Mitigated by designing with K8s Deployment as the reference plugin throughout.
**Sources:** casehubio/casehub-ops#87 issue body, YAML language extensions spec (#116)
**Exploration:** quick
**Status:** captured

## D2: Execution model — build-time validation, runtime interpretation

**Choice:** Comprehensive build-time validation at Quarkus build time (types, references, primitives, composition cycles). Runtime interpretation by generic Java beans (YamlPluginProvisioner, YamlPluginActualStateAdapter).
**Alternatives:**
- Build-time code generation (Gizmo) — fastest runtime but complex build pipeline, harder to debug
- Pure runtime interpretation — simpler build but errors surface only at execution time
**Rationale:** Plugin steps make REST/GraphQL calls to external APIs — interpretation overhead is negligible compared to network latency. Build-time validation catches typos, missing primitives, and reference errors before deployment. Pattern proven by existing YAML surface (#116).
**Trade-offs:** Interpretation has marginally higher per-call overhead than generated code. Acceptable because external API latency dominates.
**Sources:** YamlDesiredStateProcessor (existing build-time validation), YamlGraphRecorder (existing runtime interpretation)
**Exploration:** quick
**Status:** captured

## D3: NodeSpec — dual Java/YAML declaration, mixed freely

**Choice:** Support both Java NodeSpec records (via @NodeTypeId) and YAML-defined spec schemas. Mixed and matched per type — a plugin can use either. YAML schemas define fields with types, constraints, and defaults. Runtime representation is Map<String, Object> for YAML-declared types.
**Alternatives:**
- YAML-only schemas — achieves "no Java" but forces YAML on types that benefit from Java type safety
- Java-only NodeSpec — simplest runtime but defeats the "no Java required" goal
- JSON Schema — stricter validation but more verbose, harder for operators to author
**Rationale:** Different plugin authors have different needs. Platform developers writing complex types want Java records. Operators adding a new REST-managed resource want YAML. The registry supports both transparently. Machine-readable YAML schemas enable future IDE plugins for autocomplete and refactoring.
**Trade-offs:** Two code paths in NodeSpecRegistry — one for Java classes, one for YAML schemas. Complexity is bounded because both converge to the same runtime representation (the provisioner works with either).
**Sources:** NodeSpecRegistry, @NodeTypeId annotation, user requirement for IDE plugin support
**Exploration:** quick
**Status:** captured

## D4: Composition model — step pipeline with named bindings

**Choice:** Ordered list of steps. Each step invokes a named primitive, receives parameters, produces a result bound to a name. Subsequent steps reference prior results via ${<name>.<path>}. Sequential execution only — no parallelism, no workflow semantics at the per-node level.
**Alternatives:**
- DAG with declarative bindings — allows parallel execution but harder to validate, debug, and visualize
- Nested expressions (Helm-style) — compact but harder to debug and validate
- Serverless Workflow YAML — overkill for per-node provisioning; workflow orchestration already handled by CaseTransitionExecutor at the plan level
**Rationale:** Per-node provisioning is 1-3 sequential REST calls to one external system. Parallelism within a single node doesn't make sense. Simplicity aids debugging and validation. The graph's node-level parallelism handles the case where independent nodes should provision concurrently.
**Trade-offs:** Cannot express parallel API calls within one node's provisioning. Acceptable because this pattern is rare and can fall back to a Java NodeProvisioner.
**Sources:** GitHub Actions step model, Ansible tasks model
**Exploration:** quick
**Status:** captured

## D5: Primitive composition — YAML over YAML over Java from day one

**Choice:** Full hierarchical composition from the start. Java primitives (rest-call, json-extract, etc.) are leaf nodes. YAML compound primitives compose other primitives (Java or YAML) into reusable named operations. Discovered at build time from META-INF/desiredstate/primitives/. Cycle detection and max nesting depth (default 5) enforced at build time.
**Alternatives:**
- Flat first, compose later — simpler first iteration but defers the composition model, risking a retrofit
- Java-only primitives — simplest but limits the "no Java" story
**Rationale:** The issue explicitly requires "YAML over YAML over Java" composition. Deferring it risks designing a primitive contract that doesn't support composition, requiring a breaking change later. Build-time expansion of YAML primitives into flat step sequences keeps runtime simple.
**Trade-offs:** More complex build-time validation (cycle detection, depth checking, parameter propagation across composition boundaries). Worth it to avoid retrofit.
**Sources:** casehubio/casehub-ops#87 "Composable: YAML over YAML over Java", #116 module composition model
**Exploration:** quick
**Status:** captured

## D6: Auth model — named auth refs resolved at runtime

**Choice:** Plugin steps reference authentication by name (auth: k8s). Auth providers are registered separately (YAML config or Java CDI bean) and resolve credentials at runtime via CredentialResolver. Plugin YAML is credential-free.
**Alternatives:**
- Inline credential config — simpler for single-use plugins but mixes concerns, risks credential leakage
- EndpointRegistry integration — reuses existing platform infra but couples plugins to endpoint registration model
**Rationale:** Separation of concerns — plugin logic describes behavior, not credentials. Auth providers are environment-specific (dev vs prod). Named refs allow multiple plugins to share the same auth provider.
**Trade-offs:** Requires a separate auth provider registration mechanism. Builds on existing CredentialResolver SPI from casehub-platform.
**Sources:** casehub-platform CredentialResolver SPI, EndpointRegistry credentialRef pattern
**Exploration:** quick
**Status:** captured

## D7: SPI mapping — single generic provisioner/adapter per surface

**Choice:** One YamlPluginProvisioner bean handles all YAML-declared types (routes internally by NodeType to the appropriate step pipeline descriptor). Same pattern for YamlPluginActualStateAdapter. Existing DefaultNodeProvisionerRouter sees one YAML provisioner alongside N Java provisioners.
**Alternatives:**
- Per-plugin synthetic beans — each YAML plugin generates a separate NodeProvisioner/ActualStateAdapter bean at build time. More aligned with annotation surface but creates N beans instead of 1.
**Rationale:** The router already handles multi-type provisioners via handledTypes(). A single generic bean is simpler — one bean, one routing table. Per-plugin beans would create unnecessary CDI complexity. Build-time conflict detection catches Java/YAML type overlap.
**Trade-offs:** A single bean handling many types has a larger routing table. Negligible impact at expected scale (10-50 plugin types).
**Sources:** DefaultNodeProvisionerRouter, CdiNodeProvisionerRouter, handledTypes() SPI contract
**Exploration:** quick
**Status:** captured

## D8: Plugin file structure — one file per type at META-INF/desiredstate/plugins/

**Choice:** Each plugin is a separate YAML file at META-INF/desiredstate/plugins/<type>.yaml. One file declares one node type's complete self-healing behavior (spec, actual-state, provisioner, fault-policy, cbr, ras). Discovered at build time via classpath scan.
**Alternatives:**
- Multiple types per file — reduces file count but makes each file harder to navigate and validate
- Alongside graph files — simpler discovery but conflates declaration (what exists) with behavior (how to manage it)
**Rationale:** One-file-per-type matches the mental model — a plugin is a self-contained unit. Classpath scan at META-INF/desiredstate/plugins/ follows the established pattern for graph files (META-INF/desiredstate/) and modules (META-INF/desiredstate/modules/). Plugins can ship in library JARs.
**Trade-offs:** Many files for many types. Acceptable — operators typically manage 5-20 types, not hundreds.
**Sources:** META-INF/desiredstate/*.yaml discovery pattern, META-INF/desiredstate/modules/ pattern from #116
**Exploration:** quick
**Status:** captured

## D9: Fault policy — reuses #116 syntax verbatim

**Choice:** The plugin's fault-policy section uses identical syntax to the #116 YAML fault policy declarations. Same ThresholdFaultPolicy builder, same ${fault.*} interpolation, same FaultCountStore injection.
**Alternatives:**
- Plugin-specific fault policy format — could be more tightly integrated with plugin sections but creates unnecessary divergence
**Rationale:** No reason to reinvent. The #116 fault policy syntax maps directly to ThresholdFaultPolicy.builder() and is already validated and tested. Plugin authors who know the graph-level fault-policy syntax can reuse their knowledge.
**Trade-offs:** None meaningful — this is pure reuse.
**Sources:** #116 YAML language extensions design spec §6.1
**Exploration:** quick
**Status:** captured

## D10: CBR/RAS — declarative metadata, not runtime changes

**Choice:** The cbr and ras plugin sections are declarative metadata that register feature extractors, outcome signals, and situation definitions with existing CBR and RAS infrastructure. They don't change the CBR/RAS runtime — they configure it for the new node type.
**Alternatives:**
- Deep CBR/RAS integration — custom retrieval/adaptation logic per plugin type. More powerful but belongs in a separate spec after the core CBR/RAS SPIs are validated with YAML plugins.
**Rationale:** CBR and RAS already have well-defined SPIs. Plugins declare what features matter and what situations to detect — the retrieval, adaptation, and aggregation logic is generic. Deep integration (custom retrievers, custom adaptation) is a future iteration.
**Trade-offs:** Limited to feature extraction and outcome signals — cannot express custom similarity functions or adaptation strategies in YAML. Falls back to Java for advanced CBR use cases.
**Sources:** CbrFaultPolicy, CbrSituationRecompiler, ConfigurationRetriever SPI, RAS Ganglia
**Exploration:** quick
**Status:** captured

## D11: Interpolation namespaces — extending #116 model

**Choice:** Plugin YAML adds three new interpolation prefixes to the #116 namespace model: ${spec.*} (node spec fields), ${auth.<name>.*} (auth provider properties), and ${param.*} (compound primitive parameters). Step results use unqualified names (${response.body}, not ${step.response.body}). Build-time validates no name collision between result names and reserved prefixes.
**Alternatives:**
- All-qualified names (${step.response.body}) — more explicit but verbose for the common case
- Single flat namespace — simpler but collision-prone
**Rationale:** Short unqualified result names match the sequential pipeline mental model — each step names its output, subsequent steps reference it. Reserved prefix check at build time prevents the collision risk. Consistent with #116's prefix-based dispatch architecture.
**Trade-offs:** Unqualified result names could shadow reserved prefixes — mitigated by build-time validation.
**Depends on:** D4 (step pipeline model)
**Sources:** #116 §4 Interpolation Model
**Exploration:** quick
**Status:** captured
