## D1: Protocol version support

**Choice:** Both v5 and v6
**Alternatives:**
- v6 only — simpler but excludes many current providers (e.g. AWS)
- v5 only — covers majority today but leaves out newer providers (e.g. Azure)
**Rationale:** Protocol differences are mainly at schema/state serialization, not RPC structure. A shared ProtocolTranslator with two thin serialization adapters keeps complexity low. Shipping one without the other frustrates users immediately.
**Trade-offs:** More initial implementation work, two code paths to maintain
**Sources:** Terraform plugin protocol docs, issue #184
**Exploration:** quick
**Status:** captured

## D2: Integration approach

**Choice:** Direct Java host — replicate go-plugin host-side behavior in Java
**Alternatives:**
- Go shim/proxy — uses real go-plugin library, but adds a hop and Go build dependency
- CLI wrapper (terraform plan/apply) — simplest but slow, stateful, poor error granularity
**Rationale:** go-plugin handshake is well-documented and stable. Single stdout line parse + standard gRPC connect. Go shim adds operational complexity (two managed processes per provider) and Go toolchain requirement. CLI wrapping too coarse for per-resource reconciliation.
**Trade-offs:** Must track go-plugin protocol changes ourselves (rare — hasn't changed in years)
**Sources:** HashiCorp go-plugin protocol spec, issue #182 design principle
**Exploration:** quick
**Status:** captured

## D3: Schema auto-discovery and YAML plugin leverage

**Choice:** Auto-generate NodeTypes and PluginSpecSchema from Terraform's GetProviderSchema at startup
**Alternatives:**
- Manual manifest declaration per resource type — explicit but defeats zero-config
- Code-gen at build time — generates Java types, but requires rebuild per provider version
**Rationale:** Zero-code provider adoption is the key differentiator. CaseHub's YAML plugin PluginSpecSchema provides native spec validation. Combined with Terraform's schema introspection, adopting a new provider requires no manifests, no code — point at the binary and go. Competitive advantage over Crossplane (hand-written CRDs) and Pulumi (SDK code generation).
**Trade-offs:** Startup cost for schema discovery; schema must be cached/persisted for performance
**Sources:** Terraform GetProviderSchema RPC, plugin/ module PluginSpecSchema, issue #182 design principle
**Exploration:** quick
**Status:** captured

## D4: Provider binary discovery

**Choice:** Registry auto-download with binary path override
**Alternatives:**
- Registry only — excludes air-gapped/custom providers
- Binary path only — requires manual provider management, weakens zero-config
**Rationale:** Registry auto-download makes zero-config real ("declare hashicorp/aws and go"). Binary path override handles enterprise/air-gapped scenarios and custom providers. Registry download is a well-documented HTTP API.
**Trade-offs:** Registry download adds network dependency at startup; must handle registry unavailability gracefully
**Sources:** Terraform registry API (registry.terraform.io), bridge manifest BridgeLifecycleConfig
**Exploration:** quick
**Status:** captured

## D5: Phase 1 scope

**Choice:** Core: provider lifecycle, schema discovery, plan/apply, state read, test harness
**Alternatives:**
- Full scope (add import, credentials, suspend/resume) — too much for one issue at L/High
**Rationale:** Phase 1 proves the full end-to-end loop with credential-free providers (null, random). Import and credential resolution are additive and don't change core architecture.
**Trade-offs:** Cloud provider testing deferred until credential resolution lands in Phase 2
**Sources:** Issue #184 scope, bridge SPI capabilities
**Exploration:** quick
**Status:** captured

## D6: Module structure

**Choice:** Single bridge/terraform/ module
**Alternatives:**
- Two modules (terraform-protocol + terraform) — cleaner separation but premature
- Three modules (generic process manager + protocol + bridge) — maximum reuse but YAGNI
**Rationale:** One Terraform bridge today. Splitting before a second go-plugin bridge exists is premature. Protocol layer is internal, not a published API. Extract shared code if/when Packer or Vault bridges arrive.
**Trade-offs:** If multiple go-plugin bridges emerge, will need extraction refactor
**Sources:** Bridge runtime module structure, YAGNI principle
**Exploration:** quick
**Status:** captured

## D7: Testing strategy

**Choice:** Three-tier: protocol mock + real providers (null, random) + optional cloud tier
**Alternatives:**
- Mock-only — fast but doesn't prove zero-config with real providers
- Real providers only — slow, harder to test error paths
**Rationale:** Protocol mock for fast unit tests of handshake, v5/v6 translation, error paths. Real null provider proves protocol plumbing. Real random provider proves full lifecycle (schema discovery, plan, apply, state read). Together they prove zero-config without cloud dependency in CI.
**Trade-offs:** Real provider tests require downloading binaries in CI; slightly slower than pure mocks
**Sources:** hashicorp/null and hashicorp/random provider binaries
**Exploration:** quick
**Status:** captured
