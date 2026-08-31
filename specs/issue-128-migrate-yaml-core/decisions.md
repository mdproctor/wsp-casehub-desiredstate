## D1: ForEachExpander migration strategy — restructure YamlGraphRecorder

**Choice:** Restructure — separate expansion from spec resolution. Use yaml-core's `ForEachExpander<YamlNode>` with a `ForEachAdapter<YamlNode>` for forEach/when/stamping. Spec resolution, DesiredNode creation, and dependency wiring move to a second pass in `YamlGraphRecorder`.
**Alternatives:**
- Thin wrapper — keep local ForEachExpander, delegate to yaml-core internally. Smaller diff but keeps a local class that's just a wrapper, defeating the migration purpose.
**Rationale:** The whole point is to stop maintaining local copies. yaml-core's design separates generic expansion from domain transformation. Aligning with that design makes the codebase a proper consumer of the platform primitive.
**Trade-offs:** Larger diff in `YamlGraphRecorder` — restructures two call sites from single-pass (expand + resolve) to two-pass (expand, then resolve). Tests need updating to match the new structure.
**Sources:** `ForEachExpander.java` (local, 271 lines), `io.casehub.yaml.core.foreach.ForEachExpander` (yaml-core, 143 lines), `YamlGraphRecorder.java` lines 102 and 264 (call sites)
**Exploration:** quick
**Status:** captured
