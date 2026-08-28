# Does It Scale Down? — Competitive Comparison

**Issue:** #116 — operator-first declaration language
**Date:** 2026-08-28
**Purpose:** Honest assessment of whether the YAML language extensions keep simple
things simple, and where CaseHub stands against Terraform, Helm, Ansible,
Crossplane, and Pulumi at every complexity level.

## 1. Progressive Complexity Ladder

The YAML surface layers additively. Each level adds optional top-level blocks —
nothing at a higher level changes the syntax of lower levels.

| Level | What you write | New concepts introduced |
|-------|---------------|------------------------|
| L0 | `nodes:` + `dependsOn:` | desiredState, type, spec, dependsOn |
| L1 | + `variables:` + `when:` | `${var.}`, boolean condition |
| L2 | + `faultPolicy:` + `invariants:` | tiers, threshold, pattern vocabulary |
| L3 | + `rules:` + `lifecycle:` | `${match.}`, actions, phases |
| L4 | + `forEach:` + `imports:` | `${each.}`, module parameters |

An operator at L0 never encounters `${match.}`, `${fault.}`, `${each.}`,
rules, invariants, fault policies, lifecycle phases, forEach, or modules.
The full power of the system is invisible until needed.

## 2. Scenario 1 — Deploy a Web App + Database

The simplest meaningful deployment: a web application and a PostgreSQL database,
with the app depending on the database.

### Terraform (~35 lines, 1-3 files, 7 concepts)

```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}
provider "aws" { region = "us-east-1" }

resource "aws_db_instance" "postgres" {
  identifier        = "myapp-db"
  engine            = "postgres"
  engine_version    = "15"
  instance_class    = "db.t3.micro"
  allocated_storage = 20
  db_name           = "myapp"
  username          = "admin"
  password          = var.db_password
  skip_final_snapshot = true
}

resource "aws_elastic_beanstalk_application" "app" {
  name = "myapp"
}

resource "aws_elastic_beanstalk_environment" "env" {
  name                = "myapp-env"
  application         = aws_elastic_beanstalk_application.app.name
  solution_stack_name = "64bit Amazon Linux 2023 v4.0.0 running Docker"
  setting {
    namespace = "aws:elasticbeanstalk:application:environment"
    name      = "DATABASE_URL"
    value     = "postgresql://${aws_db_instance.postgres.endpoint}/myapp"
  }
}

variable "db_password" { type = string; sensitive = true }
```

**Boilerplate:** ~40% (provider block, variable declaration, instance_class,
engine_version, skip_final_snapshot are ceremony).

### Helm (~35 lines, 3-5 files, 10+ concepts)

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 0.1.0
dependencies:
  - name: postgresql
    version: "12.1.9"
    repository: https://charts.bitnami.com/bitnami

# values.yaml
postgresql:
  auth:
    postgresPassword: "secret"
    database: myapp

# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: app
          image: myapp:latest
          env:
            - name: DATABASE_URL
              value: "postgresql://postgres:{{ .Values.postgresql.auth.postgresPassword }}@{{ .Release.Name }}-postgresql:5432/{{ .Values.postgresql.auth.database }}"
```

**Boilerplate:** ~55% (Kubernetes API: apiVersion, kind, metadata, spec.selector,
matchLabels, labels, template). Go template syntax adds noise.

### Ansible (~25 lines, 1-3 files, 9 concepts)

```yaml
- hosts: database
  become: yes
  roles:
    - role: geerlingguy.postgresql
      vars:
        postgresql_databases:
          - name: myapp
        postgresql_users:
          - name: appuser
            password: "secret"
            db: myapp

- hosts: webservers
  become: yes
  tasks:
    - name: Deploy app container
      community.docker.docker_container:
        name: myapp
        image: myapp:latest
        env:
          DATABASE_URL: "postgresql://appuser:secret@{{ hostvars[groups['database'][0]]['ansible_host'] }}:5432/myapp"
        ports:
          - "8080:8080"
```

**Boilerplate:** ~30% (hosts, become, role structure).

### Crossplane (~55 lines, 3 files, 8 concepts)

The developer writes a 10-line Claim. But a platform engineer must first write
the XRD (~25 lines) and Composition (~20 lines). Total upfront investment is high;
per-deployment cost is low.

**Boilerplate:** ~65% (openAPIV3Schema, compositeTypeRef, API versioning, metadata).

### Pulumi TypeScript (~22 lines, 1 file, 4 concepts)

```typescript
import * as aws from "@pulumi/aws";

const db = new aws.rds.Instance("myapp-db", {
    engine: "postgres",
    engineVersion: "15",
    instanceClass: "db.t3.micro",
    allocatedStorage: 20,
    dbName: "myapp",
    username: "admin",
    password: "secret",
    skipFinalSnapshot: true,
});

const app = new aws.elasticbeanstalk.Application("myapp", {});
const env = new aws.elasticbeanstalk.Environment("myapp-env", {
    application: app.name,
    solutionStackName: "64bit Amazon Linux 2023 v4.0.0 running Docker",
    settings: [{
        namespace: "aws:elasticbeanstalk:application:environment",
        name: "DATABASE_URL",
        value: pulumi.interpolate`postgresql://${db.endpoint}/myapp`,
    }],
});
```

**Boilerplate:** ~20% (import, Beanstalk solution stack name).

### CaseHub YAML (~20 lines, 1 file, 6 concepts)

```yaml
desiredState:
  namespace: myapp
  name: webapp

variables:
  db_password: "secret"

nodes:
  database:
    type: postgresql
    spec:
      name: myapp
      version: "15"
      username: admin
      password: "${var.db_password}"

  app:
    type: web-app
    dependsOn: [database]
    spec:
      image: myapp:latest
      databaseRef: database
```

**Boilerplate:** ~15% (desiredState header, variables block).

### Scenario 1 Summary

| Tool | Lines | Files | Concepts | Boilerplate |
|------|-------|-------|----------|-------------|
| Terraform | ~35 | 1-3 | 7 | ~40% |
| Helm | ~35 | 3-5 | 10+ | ~55% |
| Ansible | ~25 | 1-3 | 9 | ~30% |
| Crossplane | ~55 | 3 | 8 | ~65% |
| Pulumi | ~22 | 1 | 4 | ~20% |
| **CaseHub YAML** | **~20** | **1** | **6** | **~15%** |

**Honest assessment:** CaseHub matches or beats every tool on simplicity for this
baseline case. Pulumi is the closest competitor (22 vs 20 lines, 4 vs 6 concepts).
Terraform and Helm are significantly more verbose. But Pulumi's 4 concepts are
all transferable programming skills (import, constructor, variable, interpolation),
while CaseHub's 6 include domain-specific concepts (desiredState, type, dependsOn).

## 3. Scenario 2 — Add Monitoring

Same deployment, plus a monitoring agent that watches the app. This tests how each
tool handles the "add monitoring to everything" pattern.

| Tool | Additional lines | Approach | Scales to N apps? |
|------|-----------------|----------|-------------------|
| Terraform | ~15 | Manual CloudWatch alarm per resource | No — must copy per resource |
| Helm | ~25 | New template or subchart dependency | No — template per monitored resource |
| Ansible | ~15-20 | New play or role per host group | No — must duplicate per group |
| Crossplane | ~15-20 | New resource in Composition | Partial — applies to all Claims of type |
| Pulumi | ~10 | New resource constructor + reference | No — must write per resource |
| **CaseHub manual** | **~4** | Node with `dependsOn` | No — must copy per app |
| **CaseHub rule** | **~14** | Structural graph rule | **Yes — fires for every web-app node** |

This is where differentiation appears. The CaseHub graph rule:

```yaml
rules:
  ensure-monitoring:
    match:
      app: { type: web-app }
    notExists:
      guard: { type: monitor, of: app, direction: dependents }
    actions:
      - addNode:
          id: "monitor-${match.app.id}"
          type: monitor
          spec:
            target: "${match.app.id}"
      - addDependency:
          from: "monitor-${match.app.id}"
          to: "${match.app.id}"
```

Add 10 more web-app nodes → 10 monitors appear automatically. No other tool
can express this declaratively. Terraform requires 10 explicit alarm resources.
Helm requires Go template loops. Ansible requires explicit tasks per host.

**Honest caveat:** The rule is 14 lines vs 4 for a manual node. For a single app,
the manual approach is simpler. The rule pays off at 3+ apps. Operators who never
have more than one instance of a type gain nothing from rules.

## 4. Greenfield Lifecycle Comparison

The three-tool stack (Terraform + Helm + Ansible) is not a deliberate best
practice — it's accumulated tooling from different eras. Even FSIs starting
greenfield today typically pick 1-2 tools, not 3:

| FSI profile | Greenfield choice | Why |
|-------------|-------------------|-----|
| K8s-native | Terraform + ArgoCD | Terraform for infra, GitOps for apps. 2 tools. |
| Serverless | Terraform only | No machines, no K8s. 1 tool. |
| VM-heavy | Terraform + Ansible | Terraform provisions, Ansible hardens. 2 tools. |
| Platform eng | Crossplane | Unify infra + K8s. 1 tool. |

The fair comparison is greenfield vs greenfield — CaseHub against the tools
a sophisticated team would actually choose for a new platform today.

**Greenfield: CaseHub vs Terraform + ArgoCD (most common FSI choice)**

| Concern | Terraform + ArgoCD | CaseHub YAML |
|---------|-------------------|--------------|
| Day 0 — provision | ✅ Terraform | ✅ lifecycle phase 1 |
| Day 1 — platform | ⚠️ mix of Terraform + ArgoCD | ✅ lifecycle phase 2 |
| Day 2 — deploy app | ✅ ArgoCD (GitOps) | ✅ lifecycle phase 3 |
| Drift detection | ✅ ArgoCD continuous, Terraform on-demand | ✅ continuous reconciliation |
| Fault handling | ❌ neither tool | ✅ multi-tier escalation |
| Structural rules | ❌ not a concept | ✅ graph rules |
| Live invariants | ❌ OPA at plan time only | ✅ continuous enforcement |
| Audit separation | ✅ separate state stores | ⚠️ per-phase audit events (needs design) |
| Ecosystem | ✅ 4,000 providers + Helm charts | ❌ write your own NodeSpecs |
| Preview | ✅ terraform plan + ArgoCD diff | ❌ not yet |

CaseHub covers Day 0 through Day N in one declaration. Terraform + ArgoCD
covers Day 0 and Day 2 well but leaves fault handling, structural rules,
and continuous invariants to you. The trade is lifecycle completeness vs
ecosystem breadth.

**The three-tool stack for context** (documented by LOAD Digital, Scalr,
Spacelift — included for reference, not as the primary comparison):

| Phase | Tool | Files | Lines | What it does |
|-------|------|-------|-------|-------------|
| Day 0 | Terraform | 5-8 .tf | 300-600 | VPC, EKS, RDS, IAM |
| Day 1 | Ansible | 3-5 playbooks | 200-400 | OS packages, Docker, certs |
| Day 2+ | Helm | 8-12 templates | 400-800 | Deployments, services, ingress |
| Glue | CI/CD pipeline | 1-2 files | 100-200 | Stages, artifact passing |
| **Total** | **3 tools** | **~20-27 files** | **~1000-2000** | 3 languages, 3 state models |

This combination exists because each tool chose to be excellent at one
interface (cloud API, SSH, K8s API) rather than mediocre at all of them.
It is not a deliberate architecture — it is accumulated tool specialisation.

**CaseHub YAML equivalent** (lifecycle phases):

```yaml
desiredState:
  namespace: platform
  name: full-stack

lifecycle:
  phases:
    - id: infrastructure
      completionCondition: allPresent
      nodes:
        vpc: { type: network, spec: { cidr: "10.0.0.0/16" } }
        database: { type: postgresql, spec: { version: "15", ha: true } }
        cluster: { type: kubernetes, dependsOn: [vpc], spec: { version: "1.29" } }

    - id: platform
      completionCondition: allPresent
      nodes:
        ingress: { type: ingress-controller, dependsOn: [cluster], spec: { class: nginx } }
        certs: { type: cert-manager, dependsOn: [cluster], spec: { issuer: letsencrypt } }

    - id: application
      completionCondition: never
      nodes:
        api: { type: web-app, dependsOn: [database, ingress, certs], spec: { image: "api:latest" } }
        worker: { type: worker, dependsOn: [database], spec: { image: "worker:latest" } }

faultPolicy:
  - faultTypes: [PROVISION_FAILED]
    nodeTypes: [web-app, worker]
    tiers:
      - threshold: 3
        reviewNode:
          type: ai-review
          spec: { target: "${fault.nodeId}", detail: "${fault.detail}" }
      - threshold: 5
        reviewNode:
          type: human-review
          humanGating: ALL
          spec: { target: "${fault.nodeId}", instruction: "Manual intervention required" }

invariants:
  every-app-has-ingress:
    match:
      app: { type: web-app }
    directDep:
      ingress: { type: ingress-controller, of: app, direction: dependencies }

rules:
  ensure-monitoring:
    match:
      service: { type: web-app }
    notExists:
      guard: { type: monitor, of: service, direction: dependents }
    actions:
      - addNode:
          id: "monitor-${match.service.id}"
          type: monitor
          spec: { target: "${match.service.id}" }
      - addDependency:
          from: "monitor-${match.service.id}"
          to: "${match.service.id}"
```

**~65 lines, 1 file, 1 tool, 1 state model.** Covers:
- Infrastructure provisioning (lifecycle phase 1)
- Platform setup (lifecycle phase 2)
- Application deployment with steady-state reconciliation (lifecycle phase 3, `never`)
- Multi-tier fault escalation (auto-fix → AI → human)
- Continuous invariant enforcement ("every app has ingress")
- Structural graph rules ("every app gets a monitor")
- Continuous drift detection (built into reconciliation loop)

**What the three-tool stack needs 1000-2000 lines and 20-27 files to achieve,
CaseHub expresses in ~65 lines and 1 file.**

## 5. Honest Weaknesses

This section is purely about where CaseHub is worse. No hedging.

### 5.1 Ecosystem — Zero

Terraform: ~4,000 providers, 1,300+ AWS resources alone. Helm: ~15,000 charts.
Ansible Galaxy: ~20,000+ roles. CaseHub: zero public NodeSpec implementations,
zero shared modules, zero community-contributed graphs.

The cold-start problem is severe. Even if the language is superior, an operator
choosing CaseHub must write every provisioner from scratch. Terraform's AWS
provider represents person-decades of work.

### 5.2 Tooling Gap — No Preview

Without dry-run/preview, operators can't see the expanded graph before it hits
the reconciliation loop. Terraform `plan` is not optional — it's the primary
operator interface. Every production Terraform workflow gates on plan review.
An operator making a config change that affects forEach expansion discovers the
impact at reconciliation time, not before. This is a production safety gap.

### 5.3 Community — Every Error Is a Dead End

When a Terraform user hits an error, they find 50+ StackOverflow answers.
When a CaseHub user hits `UnresolvedVariableException`, they find nothing.
Zero blog posts, zero forums, zero third-party tutorials.

### 5.4 Learning Curve — 4 Namespaces vs 1

Terraform: `var.name` (one interpolation context). CaseHub: `${var.}`,
`${match.binding.id}`, `${fault.nodeId}`, `${each.region}` (four contexts with
different access patterns). The conceptual surface area is 2-3× larger for
someone writing their first rule.

**Mitigating factor:** An L0 operator sees only `${var.}`. The other three
namespaces appear only at L2+ (fault policies, rules, forEach). The complexity
is layered, not front-loaded.

### 5.5 Java Dependency — JVM Is a Filter

Terraform: 50MB static binary. Helm: 45MB static binary. CaseHub: requires
Quarkus 3.x, JDK 21+, ~200MB+ runtime. For DevOps teams on Alpine-based CI
pipelines, the JVM requirement is a non-starter. This immediately filters out
the Python/Go-native infrastructure community.

### 5.6 The Switching Question

> "I have 500 Terraform modules in production. My team knows HCL. My CI
> pipeline runs `terraform plan` with automated policy checks. Every cloud
> resource I need has a maintained provider. You're asking me to rewrite
> all of that in a language with no ecosystem, no preview, no community,
> and a JVM dependency — because it has graph rules I've never needed."

CaseHub YAML is not a Terraform replacement for existing infrastructure teams.
It's a platform for teams already on JVM/Quarkus who need continuous reconciliation,
structural adaptation, and lifecycle management — capabilities no existing tool
provides. The target is greenfield platform engineering on CaseHub, not migration
from established IaC stacks.

## 6. Competitive Position Matrix

| Dimension | Terraform | Helm | Ansible | Crossplane | Pulumi | CaseHub YAML |
|-----------|-----------|------|---------|------------|--------|-------------|
| Simple case verbosity | Medium | High | Low | Very high | Low | **Low** |
| Concepts to learn | 7 | 10+ | 9 | 8 | 4 | 6 (L0) |
| Boilerplate ratio | 40% | 55% | 30% | 65% | 20% | **15%** |
| Structural rules | ✗ | ✗ | ✗ | ✗ | ✗ | **✓** |
| Continuous invariants | ✗ | ✗ | ✗ | ✗ | ✗ | **✓** |
| Fault escalation | ✗ | ✗ | Partial | ✗ | ✗ | **✓** |
| Lifecycle phases | Separate configs | Hooks | Separate playbooks | ✗ | ✗ | **✓** |
| Drift detection | On-demand | ✗ | ✗ | Continuous | ✗ | **Continuous** |
| Ecosystem | Massive | Large | Large | Growing | Medium | **None** |
| Preview/dry-run | ✓ | ✓ | ✓ | ✓ | ✓ | **✗** |
| Community | Massive | Large | Large | Medium | Medium | **None** |
| Runtime footprint | 50MB binary | 45MB binary | Python | Kubernetes | Node.js | **JVM ~200MB** |
| Multi-cloud providers | 4,000+ | N/A | 20,000+ | Growing | 100+ | **Write your own** |

## 7. Conclusion

**Does it scale down?** Yes — at the language level. The YAML surface is
competitive at L0 (20 lines, 15% boilerplate, 6 concepts) and layers power
progressively without polluting the simple case. An operator who never needs
rules, invariants, or fault policies never encounters them.

**Where it doesn't scale down** is at the ecosystem level. The Java NodeSpec
dependency, the absence of pre-built modules, and the lack of preview tooling
are real barriers that no amount of language elegance solves. These are platform
maturity issues, not design issues.

**The honest pitch:** CaseHub YAML is for teams who need what no existing tool
provides — structural graph rewriting, continuous invariants, multi-tier fault
escalation, and lifecycle phases — and are willing to write their own provisioners
in exchange. For teams who just need to deploy and forget, Terraform is fine.
For teams whose infrastructure needs to think, adapt, and self-heal, CaseHub
is the only declarative option.

## References

- Scalr: "Ultimate Guide to Using Terraform with Ansible"
- Spacelift: "Using Terraform and Ansible Together" / "Terraform Drift Detection"
- LOAD Digital: "How Terraform, Ansible, Kubernetes and Helm Work Together"
- Firefly: "What 3000+ Terraform Files Taught Us About Cloud Drift"
- InfoWorld: "The Terraform Scaling Problem"
- HashiCorp: Terraform Helm Provider Tutorial, Validated Patterns
- OneUptime: "Terraform Ansible CI/CD Pipelines"
- env zero: "Ultimate Guide to Terraform Drift Detection"
- Crossplane Blog: "Crossplane vs Terraform"
