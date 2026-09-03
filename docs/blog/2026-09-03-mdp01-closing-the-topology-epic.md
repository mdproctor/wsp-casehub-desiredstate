---
title: "Closing the topology epic"
date: 2026-09-03
author: mdp
tags: [desiredstate, ops, topology, yaml, comparison, roadmap]
entry_type: note
subtype: diary
---

# Closing the topology epic

Epic ops#74 — canonical deployment topologies — is done. All ten sub-issues
verified and closed. The YAML frontend can express a personal blog, a
multi-region banking core, and everything in between.

The session started with a stale branch and a closed issue. The slot had
been set up for ops#74 but never started in this CLI instance. Once oriented,
the picture was clearer than expected: nine of the ten issues were already
implemented from a previous session. The one remaining piece was ops#72 —
the TransitionPlanner's orphan spec problem.

## The orphan spec problem

When a node is removed from the desired graph, the reconciliation loop needs
to deprovision it. The planner detects it as an orphan (present in actual
state, absent from desired) and creates a removal step. But with what spec?
The node is gone from the desired graph. The planner was wrapping orphans in
`UnknownSpec` — a placeholder with nodeType "unknown" that no provisioner
could handle. Every orphan deprovision failed silently.

The fix: a `ReconciliationStateStore` SPI that persists the last-reconciled
desired graph per tenant. When the planner detects an orphan, it looks up
the node in the stored previous graph and gets the real spec — correct type
for routing, correct spec for the provisioner, correct humanGating for
approval gates. Cold start falls back to UnknownSpec, which is correct
behaviour — the system shouldn't auto-delete things it never created.

The SPI follows the FaultCountStore pattern: interface in api/,
`InMemoryReconciliationStateStore` as default, `@DefaultBean` in runtime/,
JPA implementation overridable via classpath activation. The pattern is
becoming a convention worth naming.

## The comparison document

After closing the epic, I wanted to know: did we actually prove the
hypothesis? The fourteen YAML exemplars compile and reconcile correctly.
But "can express" is not the same as "would choose to express." I wrote a
side-by-side comparison — the same e-commerce deployment in CaseHub YAML,
Terraform HCL, and Ansible YAML.

The first draft was dishonest. CaseHub's 42-line count hid eighty-five lines
of Java per NodeSpec type and twenty lines of module YAML. Claude's
Terraform example was deliberately verbose — inline resources instead of
modules. The Ansible example used raw K8s manifests instead of Helm.

I ran two adversarial reviewers — one arguing for Terraform, one for Ansible.
Claude playing Terraform advocate caught real factual errors: Terraform has
`precondition/postcondition` blocks for free (not just Sentinel), Terraform
Cloud has drift detection, and `terraform plan` is a safety feature CaseHub
doesn't have yet. Claude playing Ansible advocate pointed out that AWX
provides continuous reconciliation, `block/rescue/always` nesting is tiered
escalation, and idiomatic Ansible with Helm is eight lines.

The revised document is honest about the trade-offs. CaseHub's advantage is
real — lifecycle phases, declarative invariants, auto-wiring rules, built-in
continuous reconciliation with fault escalation. But the cost is real too:
a running JVM, Java developers for new types, and a handful of NodeSpec types
versus thousands of Terraform providers.

## The roadmap

The adversarial reviews surfaced genuine product gaps. Seven issues filed:

- **#130** — plan preview before execution (the `terraform plan` gap)
- **#131** — plugin architecture with three tiers: Java plugins (exist),
  YAML plugins (next), bridge adapters for Terraform providers (future)
- **#132** — durable ReconciliationStateStore via JPA
- **#133** — import/adoption of existing unmanaged resources
- **#134** — rollback to previous desired-state graph
- **#135** — plugin testing framework with YAML test definitions
- **#136** — dry-run reconciliation mode

The plugin architecture conversation was the most interesting. The
NodeSpecFactory SPI from ops#75 already decoupled spec creation from Java
records — a YAML-defined type can produce valid NodeSpec instances today
via factories. The remaining gap is the provisioner side. YAML plugins
would define provisioning behaviour in YAML (HTTP calls, K8s API, shell
commands) without touching Java. And the architecture is deliberately open
to future bridge adapters that wrap external provider protocols — the SPI
contract doesn't assume Java as the implementation language.

The Terraform provider bridge is not what I initially thought. Pulumi's
terraform-bridge is a Go library, not a runtime protocol adapter — it
imports Terraform providers as Go module dependencies and links them at
compile time. CaseHub is Java, so the options are gRPC client (heavy),
Go sidecar (medium), or CLI backend (pragmatic). Not investing now, but
the architecture doesn't preclude it.

## What opens up

The topology matrix proved the YAML primitives compose. The roadmap
identified the gaps. The plugin architecture sketched how to close them
without redesigning the core. The next real test is whether someone
outside the team can write a YAML deployment and a YAML plugin without
asking for help. That's the usability proof the exemplars can't provide.
