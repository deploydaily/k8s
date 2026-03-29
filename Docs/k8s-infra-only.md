# Kubernetes Platform — Infrastructure & Security Guide

> **Purpose**: Build a standardized, secure AWS EKS platform using Terraform/Terragrunt with GitOps. This guide covers only the **infrastructure and security layers** — the foundation that clusters run on. Application-layer components (service mesh, monitoring, tracing, cost tools, ML operators) are out of scope and can be added later by teams as needed.
>
> **How to use**: Check off steps as you complete them. Drop this file into any AI conversation for instant context.

---

## Table of contents

1. [What we are building](#1-what-we-are-building)
2. [Architecture](#2-architecture)
3. [Repo structure](#3-repo-structure)
4. [Implementation phases](#4-implementation-phases)
5. [Security practices](#5-security-practices)
6. [Design decisions](#6-design-decisions)

---

## 1. What we are building

A platform that provisions and manages AWS EKS clusters across environments with:

- **Single Terraform/Terragrunt EKS module** — versioned via git tags, one source of truth
- **Phased rollout** — changes propagate dev → staging → prod
- **ArgoCD on an operations cluster** — manages infrastructure components across all clusters
- **Least-privilege IAM** — IRSA for every service, tag-scoped secrets, no shared credentials
- **Network security** — VPC CNI custom networking, network policies, private endpoints
- **GitOps everything** — infrastructure state, RBAC, policies, and secrets references all live in git

### What is IN scope

| Layer | Components |
|-------|-----------|
| **Compute** | EKS cluster, managed node groups (Graviton ARM), Karpenter autoscaler |
| **Networking** | VPC CNI (custom networking, prefix delegation), ALB controller, external-dns |
| **IAM & RBAC** | IRSA roles, OIDC trust, 6 RBAC role types, aws-auth management |
| **Secrets** | External Secrets Operator + AWS Secrets Manager integration |
| **Storage** | EBS CSI driver |
| **Certificates** | Cert-manager |
| **Policy enforcement** | Kyverno + security policies |
| **Backup** | Velero + S3 |
| **State management** | S3 + DynamoDB for Terraform state, KMS encryption |
| **GitOps** | ArgoCD + ApplicationSets |
| **CI/CD** | GitHub Actions (cluster deploy, container build) |

### What is OUT of scope (add later)

Service mesh (Istio/Cilium L7), APM (Datadog/Prometheus), logging (Splunk/Coralogix), tracing (Jaeger), cost tools (Kubecost), AI/ML operators (GPU/Neuron/KubeRay), Knative, Chaos Mesh, OAuth2 proxy, Kiali, Argo Rollouts.

---

## 2. Architecture

### Data flow

```
GitHub Repo
    │
    ▼
GitHub Actions
    ├── terragrunt plan/apply ──► EKS Clusters (per account)
    ├── docker build + push ───► ECR (core infra images)
    └── helm package + push ───► Helm Registry
         │
         ▼
Operations Cluster (ArgoCD)
    ├── sync ──► Dev clusters     (label: phase=dev-*)
    ├── sync ──► Staging clusters (label: phase=staging-*)
    └── sync ──► Prod clusters    (label: phase=prod-*)
```

### What each cluster gets from Terraform

```
EKS Cluster (v1.32+, private + public endpoint)
├── Managed node groups (Graviton m6g/m7g, AL2023 ARM)
├── Karpenter (IRSA + SQS interruption queue + instance profile)
├── VPC CNI (custom networking, prefix delegation, eniConfig per AZ)
├── RBAC (6 ClusterRoles + bindings)
├── IAM IRSA roles
│   ├── Secrets store (tag-scoped access to Secrets Manager)
│   ├── Velero (S3 + EC2 snapshots)
│   ├── ALB controller
│   ├── EBS CSI driver
│   └── External DNS (Route53)
├── S3 buckets (ALB logs + Velero backups)
├── KMS encryption (cluster secrets)
├── Secrets Manager entries (placeholder secrets)
└── ASG scaling policies (CPU/memory based)
```

### What ArgoCD deploys to each cluster (infra only)

```
Core infrastructure components (~12)
├── Metrics server
├── AWS EBS CSI driver
├── AWS Load Balancer Controller
├── External Secrets Operator
├── External Secrets Injector (ClusterSecretStore)
├── Cert-manager
├── Karpenter + NodePool CRDs
├── External DNS
├── Kyverno + security policies
├── Reloader
├── Velero
└── Default namespace quotas + network policies
```

---

## 3. Repo structure

```
k8s-platform/
├── .github/workflows/
│   ├── cluster-deploy.yml          # TG plan/apply on PR
│   ├── container-build.yml         # Docker build → ECR
│   └── version-dashboard.yml       # Scheduled cluster inventory
│
├── infrastructure/
│   ├── terragrunt/
│   │   ├── modules/eks/            # Core module (~25 .tf files)
│   │   │   ├── eks.tf
│   │   │   ├── eks-managed-node-group.tf
│   │   │   ├── eks-addons.tf
│   │   │   ├── karpenter.tf
│   │   │   ├── rbac-*.tf           # 6 RBAC definitions
│   │   │   ├── secrets-store-iam-role.tf
│   │   │   ├── velero-iam-role.tf
│   │   │   ├── alb-controller-iam-role.tf
│   │   │   ├── ebs-csi-driver-iam-role.tf
│   │   │   ├── external-dns-iam-role.tf
│   │   │   ├── load-balancer-log-s3-bucket.tf
│   │   │   ├── velero-backup-s3-bucket.tf
│   │   │   ├── secret-manager.tf
│   │   │   ├── aws-auth.tf
│   │   │   ├── asg-scaling-policies.tf
│   │   │   ├── _variables.tf
│   │   │   ├── _locals.tf
│   │   │   ├── _data.tf
│   │   │   ├── _outputs.tf
│   │   │   ├── _providers.tf
│   │   │   └── _versions.tf
│   │   │
│   │   └── deploy/
│   │       ├── 1-dev/{cluster}/terragrunt.hcl
│   │       ├── 2-staging/{cluster}/terragrunt.hcl
│   │       └── 3-prod/{cluster}/terragrunt.hcl
│   │
│   ├── helm/
│   │   ├── appset-{phase}/
│   │   │   └── templates/          # ~12 infra component templates
│   │   └── _values/
│   │       ├── global/             # Shared values per component
│   │       └── cluster-specific/   # Per-cluster overrides
│   │
│   └── container_image/            # Dockerfiles for infra images only
│       ├── karpenter-controller/Dockerfile
│       ├── aws-ebs-csi-driver/
│       ├── aws-load-balancer-controller/Dockerfile
│       ├── external-dns/Dockerfile
│       ├── external-secrets/Dockerfile
│       ├── cert-manager/
│       ├── kyverno/
│       ├── velero/
│       ├── metrics-server/Dockerfile
│       └── reloader/Dockerfile
│
├── resources/                      # Standalone Terraform modules
│   ├── ecr/                        # ECR repos + cross-account policies
│   ├── route53/                    # DNS zones
│   ├── acm-certificates/           # TLS certs
│   ├── deployment-role/            # Cross-account IAM
│   └── terraform-pre-reqs/         # S3 state + DynamoDB lock
│
├── docs/
│   ├── architecture/decisions/     # ADRs
│   └── runbooks/
│
├── ARCHITECTURE.md                 # ← This file
├── CONTRIBUTING.md
├── DEPLOYMENT-STANDARDS.md
└── CHANGELOG.md
```

---

## 4. Implementation phases

### Phase 0: Prerequisites

- [ ] **0.1** AWS account structure decided
  - Minimum: 1 shared-services account (ECR, Route53, ops cluster) + 1 workload account
  - Scale: separate dev / staging / prod accounts
- [ ] **0.2** VPCs created in each account
  - Private subnets (EKS nodes) tagged `Access: Private`
  - Internal subnets (VPC CNI secondary IPs) tagged `Access: Internal`
  - VPC peering or Transit Gateway for ArgoCD → workload cluster connectivity
- [ ] **0.3** GitHub repo created with branch protection
  - `main` = production, `develop` = testing, features branch from `main`
- [ ] **0.4** Terraform state infrastructure deployed
  - Per account: S3 bucket `{prefix}-tfstate-{region}-{account}`
  - Per account: DynamoDB table `{prefix}-terraform-state-lock`
- [ ] **0.5** IAM deployment role created per account
  - Assumable by GitHub Actions OIDC
  - Permissions: EKS, EC2/VPC (read), IAM (create roles/policies), S3, DynamoDB, SecretsManager, KMS
- [ ] **0.6** GitHub Actions OIDC provider configured in each AWS account
- [ ] **0.7** ECR repositories created for infra images
  - Cross-account pull policies attached

---

### Phase 1: Core Terraform module

> The single EKS module that every cluster uses. This is the hardest and most important phase.

**Start here — skeleton files:**

- [ ] **1.1** `_versions.tf` — provider version constraints (AWS ~>5.x, Kubernetes ~>2.x, Helm ~>2.x)
- [ ] **1.2** `_variables.tf` — all input variables with types and defaults
- [ ] **1.3** `_data.tf` — data sources (AZs, caller identity, subnets by tag, AMI by name)
- [ ] **1.4** `_locals.tf` — computed values (cluster_name, tags, CNI config merge, KMS admins)
- [ ] **1.5** `_providers.tf` — AWS, Kubernetes (using cluster auth), Helm
- [ ] **1.6** `_outputs.tf` — cluster name, IAM ARNs, subnet IDs, kubectl command

**EKS cluster + compute:**

- [ ] **1.7** `eks.tf` — EKS cluster using `terraform-aws-modules/eks/aws` v20.33
  - Auth mode: `API_AND_CONFIG_MAP`
  - IRSA enabled, recommended SG rules
  - Conditional: private-only endpoint, full audit logging for hardened environments
- [ ] **1.8** `eks-managed-node-group.tf` — operations node groups
  - Graviton ARM (m6g/m7g), AL2023, labeled `deployment.group: operations`
  - ON_DEMAND + SPOT groups, metadata lockdown userdata
- [ ] **1.9** `eks-addons.tf` — VPC CNI, CoreDNS, kube-proxy via EKS managed addons
- [ ] **1.10** `karpenter.tf` — IRSA, instance profile, SQS queue, additional IAM policy

**IAM (IRSA) roles — one per service:**

- [ ] **1.11** `secrets-store-iam-role.tf` — OIDC trust, tag-scoped `secretsmanager:GetSecretValue`
- [ ] **1.12** `velero-iam-role.tf` — OIDC trust, EC2 snapshots + S3 access
- [ ] **1.13** `alb-controller-iam-role.tf` — OIDC trust, ELBv2 + EC2 + WAF permissions
- [ ] **1.14** `ebs-csi-driver-iam-role.tf` — OIDC trust, EC2 volume management
- [ ] **1.15** `external-dns-iam-role.tf` — OIDC trust, Route53 record management

**RBAC — least privilege by role:**

- [ ] **1.16** `rbac-platform-admin.tf` — full cluster access (for platform team)
- [ ] **1.17** `rbac-operator.tf` — almost-admin, scoped to operations namespace
- [ ] **1.18** `rbac-developer-dev.tf` — developer access for dev clusters (exec allowed)
- [ ] **1.19** `rbac-developer-nondev.tf` — developer access for staging/prod (no exec, no secrets read)
- [ ] **1.20** `rbac-readonly.tf` — global read-only for observers
- [ ] **1.21** `rbac-service-account.tf` — least-privilege for platform orchestrator tooling
- [ ] **1.22** `rbac-bindings.tf` — ClusterRoleBindings mapping IAM roles to k8s groups

**Security & storage:**

- [ ] **1.23** `aws-auth.tf` — aws-auth ConfigMap (Karpenter node role, platform roles, additional roles)
- [ ] **1.24** `secret-manager.tf` — create Secrets Manager entries with placeholder values
- [ ] **1.25** `load-balancer-log-s3-bucket.tf` — ALB logs bucket with lifecycle + encryption
- [ ] **1.26** `velero-backup-s3-bucket.tf` — backup bucket with encryption
- [ ] **1.27** `asg-scaling-policies.tf` — CPU/memory target tracking for ops nodes

**Finalize:**

- [ ] **1.28** Run `terraform fmt` on all files
- [ ] **1.29** Tag module: `git tag v1.0.0`
- [ ] **1.30** Write CHANGELOG entry

---

### Phase 2: First cluster

> Deploy one dev cluster. Prove the module works.

- [ ] **2.1** Create `infrastructure/terragrunt/deploy/1-dev/{cluster}/terragrunt.hcl`
  - Reference module: `source = "git::https://...//infrastructure/terragrunt/modules/eks?ref=v1.0.0"`
  - Configure S3 backend, IAM role, all required inputs
- [ ] **2.2** `terragrunt plan` — review output, fix issues
- [ ] **2.3** `terragrunt apply` — create the cluster
- [ ] **2.4** Verify: `kubectl get nodes` — ops nodes running
- [ ] **2.5** Verify: VPC CNI, CoreDNS, kube-proxy pods healthy in `kube-system`
- [ ] **2.6** Verify: IRSA roles exist in IAM console
- [ ] **2.7** Verify: S3 buckets created (ALB logs, Velero)
- [ ] **2.8** Verify: Secrets Manager entries created

---

### Phase 3: ArgoCD

> GitOps engine on a dedicated operations cluster.

- [ ] **3.1** Deploy operations cluster (same TF module, sized for ArgoCD only)
- [ ] **3.2** Install ArgoCD via Helm (HA mode)
- [ ] **3.3** Connect GitHub repo (HTTPS + deploy key or GitHub App)
- [ ] **3.4** Create ArgoCD project: `core-infrastructure`
- [ ] **3.5** Register dev workload cluster: `argocd cluster add ... --label phase="dev-01"`
- [ ] **3.6** Create `infrastructure/helm/appset-1-dev-01/` with templates for infra components
- [ ] **3.7** Verify: ArgoCD UI shows cluster and synced apps

---

### Phase 4: Infrastructure components via ArgoCD

> Deploy in dependency order. Each component needs: helm chart, global values file, ApplicationSet template.

- [ ] **4.1** **Metrics server** — HPA and Karpenter need this
- [ ] **4.2** **AWS EBS CSI driver** — persistent volumes
- [ ] **4.3** **AWS Load Balancer Controller** — ingress
- [ ] **4.4** **Cert-manager** — TLS for webhooks and ingress
- [ ] **4.5** **External Secrets Operator** — connects to Secrets Manager
- [ ] **4.6** **External Secrets Injector** — ClusterSecretStore + ExternalSecret CRs
- [ ] **4.7** **Karpenter** + NodePool + EC2NodeClass — workload node autoscaling
- [ ] **4.8** **External DNS** — Route53 record automation
- [ ] **4.9** **Kyverno** — policy engine
- [ ] **4.10** **Kyverno security policies** — see [Security practices](#5-security-practices)
- [ ] **4.11** **Reloader** — restart pods on ConfigMap/Secret change
- [ ] **4.12** **Velero** — backup and restore
- [ ] **4.13** **Default namespace quotas** — resource limits
- [ ] **4.14** **Default network policies** — deny-all ingress/egress baseline

---

### Phase 5: CI/CD

- [ ] **5.1** `cluster-deploy.yml` — PR trigger, TG plan → comment → approve → apply
- [ ] **5.2** `container-build.yml` — build multi-arch ARM images → push ECR
- [ ] **5.3** `version-dashboard.yml` — cron, parse TG configs → markdown report
- [ ] **5.4** Branch protection: `main` requires PR + CI pass + 1 approval

---

### Phase 6: Multi-environment rollout

- [ ] **6.1** Deploy staging cluster(s) + register with ArgoCD (label `phase=staging-*`)
- [ ] **6.2** Create staging ApplicationSets
- [ ] **6.3** Deploy prod cluster(s) + register (label `phase=prod-*`)
- [ ] **6.4** Create prod ApplicationSets
- [ ] **6.5** Document rollout process: update dev → wait → staging → wait → prod

---

### Phase 7: Hardening

- [ ] **7.1** Enable EKS audit logging (api, audit, authenticator, controllerManager, scheduler)
- [ ] **7.2** Enable KMS encryption for EKS secrets
- [ ] **7.3** Private endpoint only for production clusters
- [ ] **7.4** Enable GuardDuty EKS protection
- [ ] **7.5** Enable AWS Config rules for EKS compliance
- [ ] **7.6** Rotate ECR image tags to use digest pinning
- [ ] **7.7** Write ADRs for all key decisions
- [ ] **7.8** Write operational runbooks (provisioning, upgrades, DR, incident response)

---

## 5. Security practices

### IAM: Zero shared credentials

Every service gets its own IRSA role with the minimum permissions it needs. No service shares an IAM role with another. The trust policy locks each role to a specific namespace + service account.

```
Karpenter          → karpenter OIDC role     → EC2, SQS, IAM (instance profiles)
External Secrets   → secrets-store role      → SecretsManager:Get (tag-scoped)
Velero             → velero role             → S3, EC2 snapshots
ALB Controller     → alb-controller role     → ELBv2, EC2, WAF
EBS CSI            → ebs-csi role            → EC2 volumes
External DNS       → external-dns role       → Route53 records
```

### Secrets: Tag-scoped access

The secrets-store IAM role can only read secrets tagged with `sk8s_cluster: {cluster-name}` or `sk8s_cluster: global`. A dev cluster cannot read prod secrets even if it guesses the secret name.

```hcl
condition {
  test     = "StringEquals"
  variable = "aws:ResourceTag/sk8s_cluster"
  values   = ["global", local.cluster_name]
}
```

### RBAC: Six roles, least privilege

| Role | Can do | Cannot do |
|------|--------|-----------|
| **Platform admin** | Everything | — |
| **Operator** | Manage deployments, services, pods, configmaps, secrets, karpenter, networking | Delete namespaces, modify RBAC |
| **Developer (dev)** | Deploy, exec into pods, read secrets, port-forward | Modify cluster-scoped resources |
| **Developer (non-dev)** | Deploy, read logs, read configmaps | Exec into pods, read secrets |
| **Readonly** | Get/list/watch all resources | Create, update, delete anything |
| **Service account** | Create/manage deployments, services in allowed namespaces | Cluster-scoped operations |

### Kyverno policies (Phase 4.10)

| Policy | What it enforces |
|--------|-----------------|
| **Disallow privileged containers** | No `privileged: true` or `hostPID/hostNetwork` |
| **Require non-root** | `runAsNonRoot: true` on all pods |
| **Disallow latest tag** | Images must have explicit tags, not `:latest` |
| **Require resource limits** | All containers must set CPU + memory limits |
| **Restrict host paths** | No `hostPath` volume mounts |
| **Require labels** | All deployments must have `app`, `team`, `environment` labels |
| **Restrict image registries** | Images must come from allowed ECR repos only |
| **Disallow default namespace** | No workloads in the `default` namespace |

### Network policies (Phase 4.14)

Default deny-all baseline per namespace. Teams explicitly allow what they need.

```yaml
# Default deny all ingress and egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

Core infrastructure namespaces (`operations`, `kube-system`) get pre-configured allow rules for cross-component communication.

### Encryption

- **EKS secrets**: KMS-encrypted (key managed by Terraform, admin access controlled via `kms_key_administrators`)
- **S3 buckets**: SSE-S3 or SSE-KMS encryption enabled on all buckets
- **Terraform state**: Encrypted S3 bucket with DynamoDB locking
- **Node metadata**: Blocked via userdata — pods cannot access EC2 instance metadata

### Container image supply chain

All images pulled from same-account ECR. Never from public registries at runtime.

```
Upstream (public.ecr.aws, docker.io, ghcr.io)
    │
    ▼  (CI pipeline: build + scan + push)
ECR in shared-services account
    │
    ▼  (cross-account pull policy)
ECR cache in workload account ──► EKS nodes pull from here
```

---

## 6. Design decisions

### Single module, versioned via git tags

The entire cluster definition lives in `modules/eks/`. Each cluster's `terragrunt.hcl` references a specific git tag (`?ref=v1.0.0`). Upgrading a cluster = changing the tag. Rolling out = changing tags phase by phase.

### Terragrunt for DRY config

Each cluster is a single `terragrunt.hcl` file (~150 lines) that sets variables. The module source, backend config, and provider setup are all handled by Terragrunt. Without Terragrunt you'd need to copy provider blocks and backend configs across 75 directories.

### ArgoCD ApplicationSets with phase labels

Instead of creating one ArgoCD Application per cluster per component, ApplicationSets use a cluster generator with `matchLabels: { phase: "dev-01" }`. Register a cluster with the right label and it gets all infrastructure components automatically.

### Operations namespace convention

All infrastructure components deploy to the `operations` namespace. DaemonSets (if any) go to `ops-{name}` namespaces. This keeps infrastructure cleanly separated from workload namespaces.

### Why Karpenter over Cluster Autoscaler

Karpenter provisions nodes in seconds (not minutes), supports mixed instance types and purchase options (spot + on-demand), and consolidates underutilized nodes automatically. It's the standard for modern EKS.

---

## Quick start priority

```
Phase 0.2 (VPC) + 0.4 (TF state)     ← Can't do anything without these
     │
     ▼
Phase 1.1–1.10 (module skeleton)      ← Get terragrunt plan to succeed
     │
     ▼
Phase 2.1–2.5 (first cluster)         ← Prove it works
     │
     ▼
Phase 1.11–1.27 (IAM + RBAC + S3)     ← Add security layers
     │
     ▼
Phase 3 (ArgoCD)                       ← GitOps engine
     │
     ▼
Phase 4.1–4.8 (core infra components) ← Essential services
     │
     ▼
Phase 4.9–4.14 (policies + security)  ← Lock it down
     │
     ▼
Phase 5 (CI/CD) + Phase 6 (multi-env) ← Scale out
```

Start with the module skeleton + first cluster. Everything else layers on top.
