---
layout: post
title: "Suspend, Resume, and a Plugin That Teaches Itself"
date: 2026-10-25
entry_type: note
subtype: diary
projects: [casehubio/casehub-desiredstate]
tags: [yaml-plugins, lifecycle, suspend-resume, teaching-examples]
---

# Suspend, Resume, and a Plugin That Teaches Itself

The YAML plugin surface now supports the full node lifecycle. Not just provision and deprovision — suspend and resume too. A plugin author declares four step lists in their YAML, and the runtime handles state machines, fault escalation, and the rest.

The runtime work threaded two new fields through six layers — `PluginProvisionerDef`, `PluginParser`, `PluginDescriptor`, `PluginDiscovery`, `YamlPluginProvisioner`, `PluginValidator` — and every layer already had the pattern from provision/deprovision to follow. The interesting design decision: what happens when a plugin declares suspend steps but not resume, or vice versa? We made each independent. `supportsStatefulLifecycle()` returns true if *any* plugin in the provisioner's map has suspend or resume steps defined. The runtime's `StatefulNodeProvisioner` decorator sees that flag and wraps accordingly. A plugin that only suspends but never resumes is a valid lifecycle — unusual, but the runtime doesn't impose opinions about what combinations make sense. That's domain policy, not infrastructure.

The monitoring-endpoint example was more revealing than the runtime work. A plugin that registers health checks, suspends them during maintenance windows, and resumes after seemed straightforward. The YAML came together quickly — spec schema with URL, name, interval, severity; provisioner steps for all four operations; actual-state with `compare-state`; fault policy with three-strike escalation to human review; CBR features on severity and interval; RAS situation for endpoint flapping detection.

But writing tests surfaced a gap. The `PluginTestExtension` framework resolves `${wiremock.url}` in spec fields but not in auth variables. Plugins whose steps reference `${auth.monitoring-api.endpoint}` — which is how any real plugin would wire its HTTP calls — can't route those calls to the WireMock mock. The existing `mock-resource` tests sidestep this because they use `assert` steps that don't need HTTP at all. The integration test covers the full auth chain programmatically, so nothing is untested. But the declarative test YAML path — the one operators would actually use — has a resolution gap for auth-scoped infrastructure bindings. Worth closing.

The YAML plugin surface is effectively feature-complete for Tier 2. Six of seven sub-issues from the epic are closed. What remains is plugin catalog and versioning — design-heavy, not implementation. The question I keep circling: how much of the catalog model can live in the plugin YAML itself? The header already carries `type`, `version`, and `resyncInterval`. Adding dependency declarations and compatibility ranges would keep the plugin self-describing — no separate manifest, no external registry schema to learn. The plugin IS the package definition. That's the direction I'm leaning, but it needs brainstorming before I commit to it.
