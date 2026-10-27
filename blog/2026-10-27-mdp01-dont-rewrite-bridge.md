---
layout: post
title: "Don't Rewrite — Bridge"
date: 2026-10-27
entry_type: article
subtype: diary
projects: [casehubio/casehub-desiredstate]
tags: [bridge-adapter, plugin-architecture, terraform, ansible, spi-design]
series: issue-183-bridge-adapter-spi-design
---

# Don't Rewrite — Bridge

There are roughly three thousand Terraform providers in the wild. Each one knows how to create, read, update, and delete a specific type of infrastructure resource. Ansible has thousands more collection modules covering everything from network switches to cloud APIs.

When I started thinking about how CaseHub's desired-state runtime should handle infrastructure provisioning, the tempting path was clear: write adapters. Take the AWS SDK, wrap it in a `NodeProvisioner`, teach it how to create EC2 instances, S3 buckets, RDS databases. Repeat for GCP, Azure, every SaaS API anyone might need.

That way lies madness. The infrastructure ecosystem is vast, and it's already been mapped — by Terraform, by Ansible, by tools that have been refining their provider models for a decade. Rewriting that knowledge would be a multi-year project that starts obsolete the day it ships.

## The Bridge Idea

The better question: what if CaseHub could use those existing providers directly?

Not as a Terraform wrapper — CaseHub isn't an orchestration frontend for `terraform apply`. The reconciliation loop has its own opinions about ordering, fault escalation, drift detection, plan approval, and lifecycle management. Those opinions are the value. What it doesn't need is its own knowledge of how to create an AWS instance. That's what the Terraform AWS provider already does.

A bridge adapter translates between CaseHub's SPI contract and an external provider's native protocol. CaseHub says "provision this node." The bridge translates that into a Terraform gRPC call, gets the result back, and translates the response into something the reconciliation loop understands. The loop never knows it's talking to Terraform. It sees a `NodeProvisioner` like any other.

## What We Actually Built

The bridge is a new SPI tier — Tier 3 in the plugin architecture, after Java plugins (Tier 1) and YAML plugins (Tier 2). The SPI has three layers:

**`BridgeProvider`** is what protocol-specific implementations provide. It's deliberately thin: `apply()`, `destroy()`, `readState()`. A Terraform bridge implements this by speaking the go-plugin gRPC protocol. An Ansible bridge implements it by shelling out to `ansible-playbook`. The bridge runtime doesn't care which.

**Optional mixins** let providers declare extra capabilities without forcing them on everyone. `PlanCapable` adds `plan()` and `applyPlan()` for providers with native plan-apply cycles — Terraform's plan/apply, Ansible's check mode. `SuspendCapable` adds `suspend()` and `resume()` for providers that can stop a resource without destroying it. If a provider doesn't implement these, the runtime falls through to the default path.

**Manifest-driven configuration** keeps bridge adapters declarative. A YAML manifest at `META-INF/desiredstate/bridges/*.yaml` declares what types a bridge handles, how provider states map to CaseHub's `NodeStatus`, and how errors classify into fault categories. An operator can customise state mapping without writing Java.

The design mirrors the YAML plugin pattern that's already proven. One `BridgeProvisioner` handles all types for a given protocol — the same single-adapter-many-types approach that `YamlPluginProvisioner` uses. `BridgeDiscovery` follows the same `BeanRegistration` pattern. Both Quarkus and Spring wiring work identically to the plugin surface.

## The Licensing Line

This matters and is easy to get wrong.

OpenTofu — the open-source Terraform fork — is MPL 2.0. Terraform providers are mostly MPL 2.0. These can be used freely, including distribution.

Ansible is GPL 3.0. CaseHub never bundles or distributes ansible-core. The bridge adapter shells out to a user-supplied `ansible-playbook` binary. For testing, ansible-core is a test-scope dependency — installed via pip in CI, never shipped. The bridge distributes only the adapter code, which is ours.

This is not a hypothetical distinction. Getting it wrong would contaminate the entire project's licence.

## What This Opens Up

The SPI and runtime shipped today. Concrete bridges — Terraform and Ansible — are next. They'll implement `BridgeProvider`, plug into the runtime we built, and prove the design against real provider protocols.

The interesting part isn't the bridge itself. It's what CaseHub layers on top. Terraform can provision infrastructure, but it doesn't have fault escalation that routes through CBR pattern matching, doesn't have drift policies with exemption windows, doesn't have plan approval gates backed by WorkItems, doesn't have cross-domain composition that coordinates infrastructure changes with application deployment in a single reconciliation cycle.

The bridge gives CaseHub reach. The runtime gives it depth. Together, an operator could declare a desired state that spans cloud infrastructure, application services, and data pipelines — managed in one graph, reconciled in one loop, with one fault policy.

That's the theory. Next step is to prove it with a real Terraform provider and a real Ansible collection.
