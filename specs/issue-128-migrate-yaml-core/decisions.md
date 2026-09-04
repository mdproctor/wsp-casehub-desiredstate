## D1: ForEachExpander migration strategy — restructure YamlGraphRecorder

**Choice:** Restructure — separate expansion from spec resolution. Use yaml-core's `ForEachExpander<YamlNode>` with a `ForEachAdapter<YamlNode>` for forEach/when/stamping. Spec resolution, DesiredNode creation, and dependency wiring move to a second pass in `YamlGraphRecorder`.
**Alternatives:**
- Thin wrapper — keep local ForEachExpander, delegate to yaml-core internally. Smaller diff but keeps a local class that's just a wrapper, defeating the migration purpose.
**Rationale:** The whole point is to stop maintaining local copies. yaml-core's design separates generic expansion from domain transformation. Aligning with that design makes the codebase a proper consumer of the platform primitive.
**Trade-offs:** Larger diff in `YamlGraphRecorder` — restructures two call sites from single-pass (expand + resolve) to two-pass (expand, then resolve). Tests need updating to match the new structure.
**Sources:** `ForEachExpander.java` (local, 271 lines), `io.casehub.yaml.core.foreach.ForEachExpander` (yaml-core, 143 lines), `YamlGraphRecorder.java` lines 102 and 264 (call sites)
**Exploration:** quick
**Status:** captured

## D2: Module migration strategy — full ModuleBridge<T> adoption

**Choice:** Full ModuleBridge<T> adoption with `DesiredStateModuleContent` record and `DesiredStateModuleBridge`. YamlGraphRecorder uses typed `expand()` → `TypedExpandedModule<DesiredStateModuleContent>`. Local model types and ModuleExpander deleted.
**Alternatives:**
- Raw expand() with ExpansionOptions — fewer new types but ignores the bridge pattern designed for this use case. Raw Map<String, Object> at call sites.
- Phased — raw expand first, bridge as follow-up. Smaller diffs but two passes through same code, intermediate state uses raw API.
**Rationale:** ModuleBridge<T> was designed and built (platform#270) specifically for this migration. Using the raw API means paying the migration cost without the typed benefit. Two new classes is a small price for typed `expanded.content().nodes()` throughout. Pre-release — one clean migration is better than two incremental ones.
**Trade-offs:** Introduces DesiredStateModuleContent record + DesiredStateModuleBridge class. Larger diff than raw API approach.
**Sources:** `ModuleBridge<T>` (platform yaml-core), `TypedExpandedModule<T>` (platform yaml-core), `ModuleExpander.java` (local, 98 lines), `YamlGraphRecorder.java` line 92 (call site), platform#270
**Exploration:** quick
**Status:** captured

## D3: Bridge deserialization — ObjectMapper.convertValue()

**Choice:** Use Jackson `ObjectMapper.convertValue()` in `DesiredStateModuleBridge.fromSections()` to convert raw `Map<String, Object>` section entries back to typed `YamlNode`, `YamlRule`, `YamlInvariant`.
**Alternatives:**
- Manual map construction — no Jackson dependency in bridge, but fragile to model changes and more code.
- Section-level readValue (YAML round-trip) — serialize back to YAML string then readValue. Unnecessary overhead; convertValue avoids the serialization step.
**Rationale:** `convertValue()` is the same mechanism YamlGraphRecorder already uses for spec deserialization. One mapper, consistent behavior, handles nested structures naturally.
**Trade-offs:** Bridge requires an ObjectMapper instance. Acceptable — the mapper is already available in both YamlGraphRecorder (runtime) and YamlDesiredStateProcessor (build-time).
**Sources:** `YamlGraphRecorder.java` — existing `mapper.convertValue()` usage for NodeSpec deserialization
**Exploration:** quick
**Status:** captured
