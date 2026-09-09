# Decisions — YAML Plugin Architecture (#87)

## D1: Scope — plugin schema + interpreter first

**Choice:** Design the 5-section YAML plugin format and the runtime that maps it to NodeProvisioner/ActualStateAdapter/FaultPolicy SPIs. Other subsystems (Java extension primitives, 10 concrete plugins, full CBR/RAS integration) follow as separate specs.
**Alternatives:**
- Java extension primitives first — building blocks before consumers, but the plugin schema defines what building blocks are needed
- Full architecture overview — complete picture but too large for one spec (XL/High issue)
- Bottom-up from one concrete plugin — discovers abstractions but risks over-fitting to one domain
**Rationale:** Everything else depends on the plugin schema — the primitives, the concrete plugins, and the CaseHub integrations all consume this format.
**Trade-offs:** Deferring the concrete plugins means the schema is validated against sketched examples, not real-world usage. Mitigated by validating the schema against structurally distinct provisioning patterns: K8s Deployment (REST API), a database provisioner (DDL + migration), and a multi-endpoint orchestrator (DNS + load balancer + app). K8s Deployment is the primary reference plugin; the other two are validation scenarios ensuring the schema accommodates non-REST patterns.
**Sources:** casehubio/casehub-ops#87 issue body, YAML language extensions spec (#116)
**Exploration:** quick
**Status:** captured

## D2: Execution model — build-time validation, runtime interpretation

**Choice:** Comprehensive build-time validation at Quarkus build time (types, references, primitives, composition cycles). Runtime interpretation by generic Java beans (YamlPluginProvisioner, YamlPluginActualStateAdapter).
**Alternatives:**
- Build-time code generation (Gizmo) — fastest runtime but complex build pipeline, harder to debug
- Pure runtime interpretation — simpler build but errors surface only at execution time
**Rationale:** Plugin steps make REST/GraphQL calls to external APIs — interpretation overhead is negligible compared to network latency. Build-time validation catches typos, missing primitives, and reference errors before deployment. Pattern proven by existing YAML surface (#116).
**Trade-offs:** Interpretation has marginally higher per-call overhead than generated code. Acceptable because external API latency dominates. Interpretation requires explicit error context propagation — when a step fails at runtime, the error must include: plugin file path, step name/index, primitive being executed, and interpolated parameter values. Build-time code generation provides natural stack traces; the interpreter must construct equivalent diagnostic context. The #116 YAML surface solved this for graph compilation errors — the plugin surface needs the same treatment for runtime step execution errors.
**Sources:** YamlDesiredStateProcessor (existing build-time validation), YamlGraphRecorder (existing runtime interpretation)
**Exploration:** quick
**Status:** captured

## D3: NodeSpec — dual Java/YAML declaration, mixed freely

**Choice:** Support both Java NodeSpec records (via @NodeTypeId) and YAML-defined spec schemas. Mixed and matched per type — a plugin can use either. YAML schemas define fields with types, constraints, and defaults. YAML-declared types produce a `YamlNodeSpec` adapter that implements `NodeSpec` by wrapping `Map<String, Object>` — deriving `nodeType()` from the plugin's declared type and `humanGating()` from the YAML schema's `humanGating` field (defaulting to `HumanGating.NONE`). The `NodeSpecFactory` SPI provides the bridge: `create(Map<String, Object>) → NodeSpec`. `NodeSpecRegistry` gains a second resolution path: YAML-declared types resolve to a `NodeSpecFactory` (producing `YamlNodeSpec` wrappers), while Java-declared types resolve to `Class<? extends NodeSpec>` (existing path, using Jackson `convertValue`).
**Alternatives:**
- YAML-only schemas — achieves "no Java" but forces YAML on types that benefit from Java type safety
- Java-only NodeSpec — simplest runtime but defeats the "no Java required" goal
- JSON Schema — stricter validation but more verbose, harder for operators to author
**Rationale:** Different plugin authors have different needs. Platform developers writing complex types want Java records. Operators adding a new REST-managed resource want YAML. `NodeSpec` is NOT a marker interface — it declares `nodeType()` and `humanGating()`. A `Map<String, Object>` cannot implement it directly. `YamlNodeSpec` bridges this: it implements `NodeSpec`, wraps the raw map for field access by the step pipeline via `${spec.*}` interpolation, and provides `nodeType()` and `humanGating()` from the plugin's declared metadata. The `NodeSpecFactory` SPI already exists for this purpose. Machine-readable YAML schemas enable future IDE plugins for autocomplete and refactoring.
**Trade-offs:** Two code paths in NodeSpecRegistry — one for Java classes (existing), one using `NodeSpecFactory` to produce `YamlNodeSpec` wrappers for YAML-declared types. Both paths produce a `NodeSpec` that `DesiredNode`, routing, and provisioning work with identically.
**Sources:** NodeSpecRegistry, NodeSpecFactory, @NodeTypeId annotation, user requirement for IDE plugin support
**Exploration:** quick
**Status:** revised — explicit `YamlNodeSpec` wrapper design replaces inaccurate "same runtime representation" claim

## D4: Composition model — step pipeline with named bindings

**Choice:** Ordered list of steps. Each step invokes a named primitive, receives parameters, produces a result bound to a name. Subsequent steps reference prior results via ${<name>.<path>}. Sequential execution only — no parallelism, no workflow semantics at the per-node level.
**Alternatives:**
- DAG with declarative bindings — allows parallel execution but harder to validate, debug, and visualize
- Nested expressions (Helm-style) — compact but harder to debug and validate
- Serverless Workflow YAML — overkill for per-node provisioning; workflow orchestration already handled by CaseTransitionExecutor at the plan level
**Rationale:** Per-node provisioning is 1-3 sequential REST calls to one external system. Parallelism within a single node doesn't make sense. Simplicity aids debugging and validation. The graph's node-level parallelism handles the case where independent nodes should provision concurrently.
**Trade-offs:** Cannot express parallel API calls within one node's provisioning. Acceptable because this pattern is rare and can fall back to a Java NodeProvisioner. The full execution nesting model is: case → Worker(Workflow) → fork (parallel independent nodes) → per-node step pipeline (sequential). Per-node sequentiality is this decision. Graph-level parallelism is CaseTransitionExecutor's concern. Plugin authors see only their sequential pipeline; the parallelism across nodes is invisible to them.
**Sources:** GitHub Actions step model, Ansible tasks model
**Exploration:** quick
**Status:** captured

## D5: Primitive composition — YAML over YAML over Java from day one

**Choice:** Full hierarchical composition from the start. Java primitives (rest-call, json-extract, etc.) are leaf nodes. YAML compound primitives compose other primitives (Java or YAML) into reusable named operations. Discovered at build time from META-INF/desiredstate/primitives/. Cycle detection and max nesting depth (default 5) enforced at build time.
**Alternatives:**
- Flat first, compose later — simpler first iteration but defers the composition model, risking a retrofit
- Java-only primitives — simplest but limits the "no Java" story
**Rationale:** The issue explicitly requires "YAML over YAML over Java" composition. Deferring it risks designing a primitive contract that doesn't support composition, requiring a breaking change later. Build-time expansion of YAML compound primitives into step sequence templates keeps runtime simple. The expanded result is a template — `${spec.*}` and `${auth.*}` values are runtime-resolved during step execution, not at expansion time. Expansion is macro-style: the compound primitive's steps are inlined at each invocation site. Runtime-conditional sub-primitive selection (choosing between sub-primitives based on runtime state) is not supported — that logic belongs in the step pipeline (D4) or falls back to Java.
**Trade-offs:** More complex build-time validation (cycle detection, depth checking, parameter propagation across composition boundaries). Parameter scoping uses innermost-wins: when primitive A invokes compound primitive B which invokes leaf primitive C, C's `${param.*}` resolves against B's parameter declarations. A's parameters are not visible to C unless B explicitly passes them through as its own parameters. Max nesting depth is 5 (vs #116 D10's module nesting cap of 2). The difference is justified: primitive expansion produces a flat runtime artifact (step sequence template). Deep nesting increases authoring complexity but not debugging complexity — the expanded result is flat, and error reporting traces back to the source primitive chain. Module nesting at depth 2+ creates nested graph structures that operators must navigate at runtime.
**Sources:** casehubio/casehub-ops#87 "Composable: YAML over YAML over Java", #116 module composition model
**Exploration:** quick
**Status:** captured

## D6: Auth model — CredentialResolver SPI directly

**Choice:** Plugin steps reference credentials by name via `auth:` stanzas, each mapping a logical name to a `credentialRef` string. At runtime, `CredentialResolver.resolve(credentialRef)` returns `Map<String, String>` credential properties. Plugin YAML uses `${auth.<name>.<key>}` interpolation to inject credential values (e.g., `${auth.k8s.token}`). No additional auth provider registration mechanism — the existing `CredentialResolver` SPI handles resolution directly. Plugin YAML is credential-free. Endpoint URLs are a separate concern, provided via `${spec.*}` fields or `${var.*}` variables per plugin — not conflated with credential resolution.
**Alternatives:**
- Inline credential config — simpler for single-use plugins but mixes concerns, risks credential leakage
- Auth provider abstraction layer — adds indirection over CredentialResolver without architectural benefit; `CredentialResolver` already supports named refs, environment-specific resolution, and shared providers
- EndpointRegistry for URLs — platform endpoint resolution is path-based and tenancy-aware, designed for platform-internal services; external API endpoints (K8s API server, Cloudflare, etc.) are better modeled as spec fields or variables
**Rationale:** Separation of concerns — plugin logic describes behavior, not credentials. `CredentialResolver.resolve(credentialRef)` is exactly the right abstraction: named credential lookup with environment-specific implementations. No additional layer needed.
**Trade-offs:** None significant — this is direct reuse of an existing platform SPI. Endpoint resolution is deferred to per-plugin design (spec fields, variables, or future EndpointRegistry integration for platform-managed services).
**Sources:** casehub-platform CredentialResolver SPI (`Map<String, String> resolve(String credentialRef)`)
**Exploration:** quick
**Status:** revised — removed unnecessary auth provider abstraction layer; CredentialResolver SPI used directly

## D7: SPI mapping — single generic provisioner/adapter per surface

**Choice:** One YamlPluginProvisioner bean handles all YAML-declared types (routes internally by NodeType to the appropriate step pipeline descriptor). Same pattern for YamlPluginActualStateAdapter. Existing DefaultNodeProvisionerRouter sees one YAML provisioner alongside N Java provisioners.
**Alternatives:**
- Per-plugin synthetic beans — each YAML plugin generates a separate NodeProvisioner/ActualStateAdapter bean at build time. More aligned with annotation surface but creates N beans instead of 1.
**Rationale:** The router already handles multi-type provisioners via handledTypes(). A single generic bean is simpler — one bean, one routing table. Per-plugin beans would create unnecessary CDI complexity. Build-time conflict detection catches Java/YAML type overlap.
**Trade-offs:** A single bean handling many types has a larger routing table. Negligible impact at expected scale (10-50 plugin types). Build-time conflict detection between Java and YAML type declarations requires a cross-surface validation pass: the annotations processor knows about Java @NodeTypeId types, the YAML plugin processor knows about YAML plugin types — a new validation step must union both sets and detect overlaps. This cross-surface validation is an explicit build-time requirement, not an implicit assumption. Dynamic CDI dependency resolution for different plugin types is handled at the primitive level, not the provisioner level — primitives are registered CDI beans with their own injection.
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

**Choice:** Plugin YAML adds four new interpolation prefixes to the #116 namespace model: `${spec.*}` (node spec fields), `${auth.<name>.*}` (credential properties via CredentialResolver), `${param.*}` (compound primitive parameters), and `${result.*}` (prior step results). All step results are qualified: `${result.response.body}`, `${result.extracted.items}`. Every interpolation reference uses an explicit prefix — no unqualified names. For YAML-declared types (D3), `${spec.*}` resolves via `Map.get()` on the `YamlNodeSpec` wrapper's underlying map. For Java NodeSpec records, `${spec.*}` resolves via record component accessors.
**Alternatives:**
- Unqualified step result names (`${response.body}`) — shorter but creates a frozen reserved prefix list. Adding any new top-level prefix (e.g., `${env.*}`, `${debug.*}`) would break existing plugins with a step result of that name. Build-time validation catches current collisions but cannot protect against future prefixes.
- Single flat namespace — simpler but collision-prone
**Rationale:** Consistent with #116 D1: "A variable named `sink` and a pattern binding named `sink` are indistinguishable without prefixes." The same principle applies to step results vs top-level namespaces. Qualified names provide permanent namespace isolation — no reserved prefix list to maintain or freeze. The verbosity cost is bounded (7 characters per reference) and consistent with the existing prefix convention.
**Trade-offs:** `${result.response.body}` is longer than `${response.body}`. Acceptable — all other namespaces (`${spec.*}`, `${auth.*}`, `${param.*}`, `${var.*}`, `${match.*}`, `${fault.*}`, `${each.*}`) are equally prefixed.
**Depends on:** D4 (step pipeline model), D3 (NodeSpec representation — affects `${spec.*}` resolution)
**Sources:** #116 §4 Interpolation Model, #116 D1 (explicit namespaces prevent ambiguity)
**Exploration:** quick
**Status:** revised — qualified `${result.*}` prefix replaces unqualified step result names for forward-compatibility and #116 D1 consistency

## D12: Plugin schema versioning — explicit version field

**Choice:** Plugin YAML files include an explicit `version:` field (initially `1`). Schema evolution defaults to backward-compatible (additive changes only). Non-backward-compatible changes increment the version. Build-time validation rejects unknown versions with a clear error. No migration transformers in v1 — migration tooling is a future concern if schema-breaking changes prove necessary.
**Alternatives:**
- No version field — implicit v1 forever. Works until the first breaking change, then requires out-of-band coordination
- Schema version with migration transformers — provides automated upgrade paths but adds build-time complexity before there is evidence of need
- Backward-compatible evolution only (never break) — constrains schema design indefinitely
**Rationale:** An explicit version field costs nothing and provides the escape hatch for future evolution. The default strategy (additive-only) avoids migration complexity. If a breaking change is needed later, the version field enables detection and clear error reporting.
**Trade-offs:** One extra line per plugin file (`version: 1`). Negligible cost.
**Sources:** #116 YAML surface (no version field — additive evolution worked so far)
**Exploration:** quick (surfaced by reviewer)
**Status:** captured

## D13: Testing model — deferred to companion spec

**Choice:** The YAML plugin testing story is architecturally significant and deferred to a companion spec. The testing model must support: (1) schema validation without Quarkus boot (CLI tool or standalone validator), (2) step pipeline testing against mock external APIs (WireMock-based), (3) interpolation verification with sample spec values, (4) fault policy behavior testing with simulated failures. The existing `casehub-desiredstate-testing` module (MockNodeProvisioner, MockActualStateAdapter) provides the SPI-level mocks; the plugin testing layer sits above this.
**Alternatives:**
- Test only via full Quarkus boot — too heavy for rapid iteration, defeats the "no Java required" goal
- Inline testing in this spec — expands scope beyond the plugin schema and interpreter design
**Rationale:** The inner development loop for YAML plugin authors is a first-class concern, but it depends on the schema and interpreter being designed first. A companion spec can design the testing model against the finalized plugin format.
**Trade-offs:** Plugin authors have no formal testing story until the companion spec is delivered. Mitigated by build-time validation catching structural errors.
**Sources:** casehub-desiredstate-testing module (existing SPI mocks), casehubio/casehub-ops#87 testing section
**Exploration:** quick (surfaced by reviewer)
**Status:** captured

## D14: Graph-to-plugin cross-file validation — build-time type coverage

**Choice:** Build-time validation that every node type referenced in graph files has a provisioner declaration — either a Java @NodeTypeId-annotated class or a YAML plugin at META-INF/desiredstate/plugins/<type>.yaml. Moves the current runtime check (`DefaultNodeProvisionerRouter`: "No provisioner for node type: X") to build time. The YAML plugin discovery (D8) is integrated with the graph validation from #116 to produce a unified type coverage check.
**Alternatives:**
- Runtime-only validation (current state) — errors surface only at execution time, which may be in production
- Partial build-time validation (graph types only, no plugin check) — catches undefined types but not missing provisioners
**Rationale:** Build-time validation is a core principle of this architecture (D2). Cross-file type coverage is the natural extension: if a graph declares a node of type "ingestion", and no provisioner exists for "ingestion," that's a build-time error, not a runtime surprise.
**Trade-offs:** Requires the build-time processor to union type information across three surfaces: Java annotations, YAML graph files, and YAML plugin files. The `YamlDesiredStateProcessor` already handles graph + annotation cross-validation (#116); this adds the plugin surface.
**Depends on:** D7 (SPI mapping), D8 (plugin file structure)
**Sources:** DefaultNodeProvisionerRouter runtime validation, YamlDesiredStateProcessor build-time validation
**Exploration:** quick (surfaced by reviewer)
**Status:** captured

## D15: CBR and RAS sections required — declarative metadata for v1

**Choice:** `cbr:` and `ras:` sections are required in every plugin YAML file, matching issue #87's explicit requirement. In v1 (this spec), the sections are declarative metadata — they declare features, outcome signals, and situation definitions without creating new runtime infrastructure. Full runtime integration (feature-aware ConfigurationRetriever for CBR, custom ganglia for RAS) is designed in companion specs under subsystem 5.
**Alternatives:**
- Optional cbr/ras (spec's original position) — contradicts issue #87's stated requirement and dilutes the architectural intent
- Required with full runtime integration — the CBR infrastructure (FeatureValue, similarity index) doesn't exist yet; RAS integration needs the SituationDefinition vocabulary, not simple expressions
**Rationale:** Issue #87 explicitly states "CBR: every plugin declares its learning surface (required, not optional)" and "RAS: every plugin declares detection situations (required, not optional)." Making them required enforces the architectural goal: every resource type is a self-healing, self-learning unit. The v1 declarative-metadata scope is honest about what the plugin schema spec delivers — the declarations are structurally correct and build-time validated, but the retrieval/feedback runtime is a separate concern.
**Trade-offs:** Plugin authors must fill out cbr and ras sections even before the full runtime consumes them. This is intentional — declarations baked in from day one are forward-compatible; bolting them on later risks gaps and inconsistency.
**Depends on:** D10 (CBR/RAS declarative metadata scope), R1-06, R1-07, R1-08
**Sources:** casehubio/casehub-ops#87 issue body ("5 required sections"), CBR integration design spec
**Exploration:** quick (surfaced by R1-06)
**Status:** captured

## D16: #117 D10 superseded — YamlNodeSpec enables Java-free types

**Choice:** Explicitly supersede #117 decision D10 ("YAML requires Java NodeSpec classes on classpath"). `YamlNodeSpec` implements `NodeSpec` by wrapping `Map<String, Object>`, enabling operators to define new node types purely in YAML without compiling Java classes.
**Alternatives:**
- Leave D10 in force — defeats the "no Java required" goal of #87
**Rationale:** D10 was appropriate for #117's scope (graph declaration — the graph declares WHAT exists, but provisioning HOW is still Java). #87 extends the surface to include behavior (provisioning, detection), making Java-free types both possible and desirable. The constraint is lifted, not violated — it's an intentional architectural evolution.
**Trade-offs:** `YamlNodeSpec` carries fields as `Map<String, Object>` — no compile-time type safety within the spec. Mitigated by build-time schema validation against the plugin's field definitions.
**Depends on:** D3 (dual declaration)
**Sources:** #117 decisions.md D10, casehubio/casehub-ops#87 issue body
**Exploration:** quick (surfaced by R1-14)
**Status:** captured
