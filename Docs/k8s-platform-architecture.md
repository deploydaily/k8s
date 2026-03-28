# Standardized Kubernetes Platform — Architecture & Implementation Guide

> **Purpose**: This document is the single source of truth for building a standardized, multi-cluster AWS EKS platform from scratch. It is designed so that any AI assistant (or engineer) can read it, understand the full architecture, and know exactly what to do next based on which steps are marked complete.
>
> **How to use**: Check off each step as you complete it. Share this file in any new conversation to instantly provide full context.

---

## Table of contents

1. [Architecture overview](#1-architecture-overview)
2. [Repository structure](#2-repository-structure)
3. [Implementation phases](#3-implementation-phases)
   - Phase 0: Prerequisites
   - Phase 1: Core Terraform module
   - Phase 2: First cluster (dev)
   - Phase 3: GitOps with ArgoCD
   - Phase 4: Core components
   - Phase 5: CI/CD pipelines
   - Phase 6: Cluster automation
   - Phase 7: Observability & security
   - Phase 8: Multi-environment rollout
   - Phase 9: Operational maturity
4. [Key design decisions](#4-key-design-decisions)
5. [Component reference](#5-component-reference)

---

## 1. Architecture overview

### What we are building

A platform that provisions and manages many AWS EKS clusters across multiple environments (dev, staging, prod) using:

- **Terraform / Terragrunt** for infrastructure-as-code, with a single EKS module versioned via git tags
- **ArgoCD** running on a dedicated operations cluster, managing all workload clusters via ApplicationSets
- **Phased rollout model** where changes propagate dev → staging → prod over time
- **~47 core components** deployed to every cluster (service mesh, autoscaler, secrets, monitoring, etc.)
- **Python automation** that generates Terragrunt configs from YAML templates for self-service cluster creation
- **GitHub Actions** for CI/CD (cluster deploys, helm packaging, container builds)

### High-level data flow

```
GitHub Repo (IaC + Helm + Configs)
        │
        ▼
GitHub Actions (CI/CD)
   ├── Terragrunt plan/apply ──► AWS EKS Clusters
   ├── Helm package + push ───► Helm Registry (OCI/ECR)
   └── Docker build + push ───► ECR (container images)
        │
        ▼
Operations Cluster
   └── ArgoCD + ApplicationSets
        ├── sync ──► Dev clusters    (phase: dev-*)
        ├── sync ──► Staging clusters (phase: staging-*)
        └── sync ──► Prod clusters   (phase: prod-*)
```

### Cluster architecture (per cluster)

Each EKS cluster gets:

- **EKS control plane**: Managed by AWS, version 1.32+, private + public endpoints
- **Managed node groups**: Graviton ARM instances (m6g/m7g), AL2023 AMI, operations-labeled nodes
- **Karpenter**: Autoscaler for workload nodes (IRSA + SQS interruption queue)
- **VPC CNI**: Custom networking with prefix delegation, eniConfig per AZ
- **Cilium**: CNI chaining mode alongside VPC CNI, replaces kube-proxy
- **Istio**: Service mesh (base, istiod, CNI, ztunnel, ingress gateway)
- **RBAC**: 6 role types (platform-admin, operator, developer-dev, developer-nondev, readonly, service-account)
- **IAM roles**: IRSA-based for Karpenter, secrets store, Velero, ALB controller, EBS CSI, external-dns
- **S3 buckets**: ALB access logs, Velero backups
- **Secrets**: AWS Secrets Manager with cluster-scoped tag-based access

---

## 2. Repository structure

```
k8s-platform/
├── .github/workflows/              # CI/CD pipelines
│   ├── cluster-deploy.yml          # Terragrunt plan/apply
│   ├── helm-package.yml            # Chart lint + publish to registry
│   ├── container-build.yml         # Docker build + ECR push
│   └── version-dashboard.yml       # Scheduled cluster inventory report
│
├── infrastructure/
│   ├── terragrunt/
│   │   ├── modules/eks/            # THE core Terraform module (~30 files)
│   │   │   ├── eks.tf              # EKS cluster (terraform-aws-modules/eks/aws v20.33)
│   │   │   ├── eks-managed-node-group.tf
│   │   │   ├── eks-addons.tf       # EKS managed addons (VPC CNI, CoreDNS, kube-proxy)
│   │   │   ├── eks-helm-addons.tf  # Helm-managed addons (VPC CNI, CoreDNS, Cilium)
│   │   │   ├── karpenter.tf        # Karpenter IRSA + SQS + IAM
│   │   │   ├── rbac-*.tf           # 6 RBAC ClusterRoles + bindings
│   │   │   ├── secrets-store-iam-role.tf
│   │   │   ├── velero-iam-role.tf
│   │   │   ├── aws-load-balancer-controller-iam-role.tf
│   │   │   ├── ebs-csi-driver-iam-role.tf
│   │   │   ├── exernal-dns-iam-role.tf
│   │   │   ├── load-balancer-log-s3-bucket.tf
│   │   │   ├── velero-backup-s3-bucket.tf
│   │   │   ├── secret-manager.tf
│   │   │   ├── aws-auth.tf
│   │   │   ├── asg_scaling_policies.tf
│   │   │   ├── _variables.tf       # ~300 lines of input variables
│   │   │   ├── _locals.tf          # Computed values, tags, CNI config
│   │   │   ├── _data.tf            # Data sources (AMI, subnets, caller identity)
│   │   │   ├── _outputs.tf         # Cluster name, IAM ARNs, subnet IDs
│   │   │   ├── _providers.tf       # AWS + Kubernetes + Helm providers
│   │   │   └── _versions.tf        # Provider version constraints
│   │   │
│   │   └── deploy/                 # Per-cluster Terragrunt configs
│   │       ├── 1-dev/
│   │       │   ├── {cluster-name}/terragrunt.hcl
│   │       │   └── environment_specific.hcl  # Shared vars for this phase
│   │       ├── 2-staging/
│   │       │   └── {cluster-name}/terragrunt.hcl
│   │       └── 3-prod/
│   │           └── {cluster-name}/terragrunt.hcl
│   │
│   ├── helm/
│   │   ├── appset-{phase-id}/      # ArgoCD ApplicationSet charts per phase
│   │   │   ├── Chart.yaml
│   │   │   ├── values.yaml
│   │   │   └── templates/          # One YAML per core component
│   │   │       ├── istio-base.yaml
│   │   │       ├── karpenter.yaml
│   │   │       ├── datadog.yaml
│   │   │       └── ... (~47 templates)
│   │   │
│   │   ├── _values/
│   │   │   ├── global/             # Default helm values (apply to ALL clusters)
│   │   │   │   ├── karpenter.yaml
│   │   │   │   ├── istio-istiod.yaml
│   │   │   │   ├── datadog.yaml
│   │   │   │   └── ...
│   │   │   └── cluster-specific/   # Per-cluster overrides
│   │   │       └── {phase}/{sub-phase}/{cluster-name}/
│   │   │           └── {component}.yaml
│   │   │
│   │   ├── {chart-name}/           # Vendored/custom helm charts
│   │   │   ├── Chart.yaml
│   │   │   ├── values.yaml
│   │   │   └── templates/
│   │   └── z_archive/              # Deprecated charts
│   │
│   ├── container_image/            # Dockerfiles for all core component images
│   │   ├── karpenter-controller/Dockerfile
│   │   ├── istio/
│   │   ├── datadog/
│   │   └── ... (~120 Dockerfiles)
│   │
│   ├── _config/                    # Cluster definition YAMLs (automation input)
│   │   └── {cluster-name}.yaml
│   │
│   └── _template/                  # Jinja2 templates for Terragrunt generation
│       └── _cluster/
│           ├── terragrunt.hcl
│           ├── vpc_cni_config_template.hcl
│           └── cluster_config_template.yaml
│
├── automation/                     # Self-service cluster provisioning
│   ├── Dockerfile
│   ├── requirements.txt            # boto3, gitpython, jinja2, pyyaml
│   ├── secrets-config.yaml         # Secret NAME references (not values)
│   ├── scr/
│   │   ├── main.py                 # Orchestrator entry point
│   │   ├── clusterProcessor/       # Config parsing, template rendering
│   │   ├── gitProcessor/           # Clone, branch, PR creation via GitHub API
│   │   ├── vpc_cni_constructor/    # Dynamic VPC CNI eniConfig from AWS API
│   │   ├── preReqClusterCheck/     # Pre-flight checks (EKS exists? IAM ready?)
│   │   └── utilityHelper/          # Shared utilities
│   └── pipeline/                   # CI pipeline scripts
│
├── resources/                      # Supporting Terraform modules (standalone)
│   ├── ecr/                        # ECR repository management
│   ├── route53/                    # DNS zone + record management
│   ├── route53-cname-weighted/     # Weighted DNS for multi-cluster routing
│   ├── route53_resolver/           # DNS resolver endpoints
│   ├── acm_certificates/           # TLS certificate provisioning
│   ├── deployment-role/            # Cross-account IAM assume roles
│   ├── terraform_pre_reqs/         # S3 state bucket + DynamoDB lock table
│   ├── terraform_custom_iam_role/  # Custom IAM role module
│   └── coralogix-alerts/           # Monitoring alert definitions
│
├── docs/
│   ├── architecture/decisions/     # Architecture Decision Records (ADRs)
│   ├── runbooks/                   # Operational procedures
│   └── version-dashboard.md        # Auto-generated cluster version matrix
│
├── README.md
├── CONTRIBUTING.md                 # Branching strategy: main + develop
├── DEPLOYMENT_STANDARDS.md         # Naming conventions, coding standards
└── CHANGELOG.md                    # Module version history
```

---

## 3. Implementation phases

### Phase 0: Prerequisites

> Set up the AWS accounts, networking, and state management before any Terraform runs.

- [ ] **0.1** Create AWS accounts (or decide on account structure)
  - Minimum: 1 dev account, 1 staging account, 1 prod account, 1 shared-services account
  - Shared-services hosts: ECR repos, Route53 zones, operations cluster
- [ ] **0.2** Set up VPCs in each account
  - Private subnets (for EKS nodes), internal subnets (for VPC CNI secondary IPs), public subnets (optional, for public ALBs)
  - VPC peering or Transit Gateway between accounts (needed for ArgoCD to reach workload clusters)
  - Tag subnets: `Access: Private`, `Access: Internal`
- [ ] **0.3** Create the GitHub repository (`k8s-platform`)
  - Set up branch protection: `main` (production), `develop` (testing)
  - Feature branches created from `main`
- [ ] **0.4** Deploy Terraform prerequisites (use `resources/terraform_pre_reqs/`)
  - Per account: S3 bucket for TF state (`{prefix}-tfstate-{region}-{account_id}`)
  - Per account: DynamoDB table for state locking (`{prefix}-terraform-state-lock`)
  - Per account: Secrets in AWS Secrets Manager (API keys, tokens — names only, values via console)
- [ ] **0.5** Create IAM deployment role in each account
  - A role that can be assumed by CI/CD (GitHub Actions OIDC) and by the Terraform runner
  - Permissions: EKS full, EC2/VPC read, IAM create roles/policies, S3, DynamoDB, SecretsManager, KMS
- [ ] **0.6** Set up ECR repositories (use `resources/ecr/`)
  - One repo per core component image (karpenter, istio-pilot, datadog-agent, etc.)
  - Cross-account pull policies so all accounts can pull from the shared ECR account
- [ ] **0.7** Configure GitHub Actions OIDC provider
  - In each AWS account, create an OIDC identity provider for `token.actions.githubusercontent.com`
  - Create a role that GitHub Actions can assume via OIDC

### Phase 1: Core Terraform module

> Build the single EKS module that every cluster will use. This is the foundation — get it right.

- [ ] **1.1** Create `infrastructure/terragrunt/modules/eks/` directory structure
- [ ] **1.2** Implement `_versions.tf` — pin provider versions
  - AWS provider ~> 5.x, Kubernetes ~> 2.x, Helm ~> 2.x
- [ ] **1.3** Implement `_variables.tf` — define all input variables
  - Key variables: `cluster_name`, `cluster_version`, `vpc_id`, `region`, `asset_id`, `ami_name`, `phase`, `is_dev_cluster`, `is_fedramp`, `eks_managed_node_groups`, `use_helm_for_addons`, `helm_addons`
- [ ] **1.4** Implement `_data.tf` — data sources
  - `aws_availability_zones`, `aws_caller_identity`, `aws_subnets` (private, internal, operations), `aws_ami` (by name filter)
- [ ] **1.5** Implement `_locals.tf` — computed values
  - `cluster_name` = `{asset_id}-{cluster_name}`, tags map, VPC CNI config merge, KMS key administrators, default aws_auth roles, CoreDNS config
- [ ] **1.6** Implement `_providers.tf` — AWS, Kubernetes, Helm providers
- [ ] **1.7** Implement `eks.tf` — main EKS cluster
  - Use `terraform-aws-modules/eks/aws` v20.33.0
  - Authentication mode: `API_AND_CONFIG_MAP`
  - IRSA enabled, recommended node SG rules enabled
  - Conditional: Istio SG rules, FedRAMP hardening (private-only endpoint, all log types, 365-day retention)
  - Conditional: `bootstrap_self_managed_addons` logic for addon migration path
- [ ] **1.8** Implement `eks-managed-node-group.tf` — operations node groups
  - Graviton instances (m6g.2xlarge, m7g.2xlarge, etc.)
  - AL2023 ARM AMI, custom userdata for metadata lockdown
  - Labels: `deployment.group: operations`
  - Separate ON_DEMAND and SPOT groups
- [ ] **1.9** Implement `eks-addons.tf` — EKS managed addons
  - CoreDNS, kube-proxy, VPC CNI via `aws-ia/eks-blueprints-addons/aws` v1.21
  - Only active when `use_helm_for_addons = false`
- [ ] **1.10** Implement `eks-helm-addons.tf` — Helm-managed addons
  - VPC CNI, CoreDNS, kube-proxy, Cilium via `helm_release` resources
  - Only active when `use_helm_for_addons = true`
  - Dynamic VPC CNI eniConfig construction from subnet data
- [ ] **1.11** Implement `karpenter.tf` — Karpenter autoscaler
  - IRSA role, instance profile, SQS interruption queue
  - Additional IAM policy for instance profile management
  - v1 permissions enabled
- [ ] **1.12** Implement `aws-auth.tf` — Kubernetes aws-auth ConfigMap
  - Default roles: Karpenter node role, platform-admin, operator, readonly
  - Configurable additional roles via `aws_auth_roles` variable
- [ ] **1.13** Implement RBAC files (6 files)
  - `rbac-adfs-devopsadmin.tf` — full cluster admin
  - `rbac-adfs-sK8sOperator.tf` — platform operator (almost-admin)
  - `rbac-developer-dev.tf` — developer role for dev clusters (exec allowed)
  - `rbac-developer-nondev.tf` — developer role for non-dev (no exec)
  - `rbac-readonly.tf` — global read-only
  - `rbac-massdriver-platform.tf` — service account for platform orchestrator
  - `rbac-developer-role-binding.tf` — bindings
- [ ] **1.14** Implement IAM role files
  - `secrets-store-iam-role.tf` — IRSA for external-secrets (tag-based secret access)
  - `velero-iam-role.tf` — IRSA for Velero backup/restore
  - `aws-load-balancer-controller-iam-role.tf` — IRSA for ALB controller
  - `ebs-csi-driver-iam-role.tf` — IRSA for EBS CSI driver
  - `exernal-dns-iam-role.tf` — IRSA for external-dns
- [ ] **1.15** Implement S3 + logging files
  - `load-balancer-log-s3-bucket.tf` — ALB access logs with lifecycle rules
  - `velero-backup-s3-bucket.tf` — Velero backup storage
- [ ] **1.16** Implement `secret-manager.tf` — create Secrets Manager entries with dummy values
- [ ] **1.17** Implement `asg_scaling_policies.tf` — CPU/memory-based ASG scaling for ops nodes
- [ ] **1.18** Implement `_outputs.tf` — export cluster name, IAM ARNs, subnet IDs, AMI ID
- [ ] **1.19** Tag the module: `git tag v1.0.0`
- [ ] **1.20** Write `CHANGELOG.md` entry for v1.0.0

### Phase 2: First cluster (dev)

> Deploy your first EKS cluster using the module. Prove it works end-to-end.

- [ ] **2.1** Create `infrastructure/terragrunt/deploy/1-dev/{cluster-name}/terragrunt.hcl`
  - Reference module via git tag: `source = "git::https://...?ref=v1.0.0"`
  - Configure S3 backend
  - Set all required inputs (VPC, region, account, AMI, versions)
- [ ] **2.2** Run `terragrunt plan` — review all resources
- [ ] **2.3** Run `terragrunt apply` — create the cluster
- [ ] **2.4** Verify access: `aws eks update-kubeconfig --name {cluster} --region {region}`
- [ ] **2.5** Verify: `kubectl get nodes` shows operations nodes running
- [ ] **2.6** Verify: VPC CNI pods running in `kube-system`
- [ ] **2.7** Verify: CoreDNS pods running
- [ ] **2.8** Verify: Karpenter controller pod running (after helm install in Phase 4)

### Phase 3: GitOps with ArgoCD

> Deploy ArgoCD on a dedicated operations cluster and register workload clusters.

- [ ] **3.1** Deploy the operations cluster (dedicated for ArgoCD)
  - Same Terraform module, but with `install_argocd = true` or manual install
  - This cluster does NOT run customer workloads
- [ ] **3.2** Install ArgoCD via Helm
  - Namespace: `argocd` (or `argo-cd`)
  - Configure HA mode for production readiness
- [ ] **3.3** Set up ArgoCD repo connection
  - Connect GitHub repo via HTTPS + deploy key or GitHub App
- [ ] **3.4** Create ArgoCD project: `core-components`
  - Allow all cluster destinations, all namespaces, all source repos
- [ ] **3.5** Register the first dev cluster with ArgoCD
  - `argocd cluster add {cluster-arn} --name {cluster-name} --label phase="dev-01"`
  - The `phase` label is critical — ApplicationSets use it to target clusters
- [ ] **3.6** Create the first ApplicationSet chart (`infrastructure/helm/appset-1-dev-01/`)
  - `Chart.yaml`, `values.yaml`, `templates/` directory
  - Each template YAML is an ArgoCD ApplicationSet with a cluster generator matching `phase: dev-01`
- [ ] **3.7** Verify: ArgoCD UI shows the registered cluster and applications

### Phase 4: Core components

> Deploy the essential platform components to every cluster via ArgoCD.

Deploy in this order (dependencies flow top to bottom):

- [ ] **4.1** **Metrics server** — required by HPA and Karpenter decisions
- [ ] **4.2** **AWS EBS CSI driver** — required for persistent volumes
- [ ] **4.3** **AWS Load Balancer Controller** — required for ingress
- [ ] **4.4** **External Secrets Operator** — required before any secret-dependent components
- [ ] **4.5** **External Secrets Injector** — cluster-scoped SecretStore + ExternalSecret resources
- [ ] **4.6** **Cert-manager** — required for TLS certificates (Istio, webhooks)
- [ ] **4.7** **Karpenter** + NodePool/EC2NodeClass CRDs — workload node autoscaling
- [ ] **4.8** **Istio base** (CRDs) → **Istio CNI** → **Istiod** → **Istio ingress gateway**
- [ ] **4.9** **External DNS** — automatic DNS record management
- [ ] **4.10** **Kyverno** + policies — security guardrails and policy enforcement
- [ ] **4.11** **Reloader** — automatic rollout restart on ConfigMap/Secret changes
- [ ] **4.12** **Velero** — cluster backup and disaster recovery
- [ ] **4.13** **Default namespace quotas** — resource limits per namespace
- [ ] **4.14** **Network policies** — default deny + allow rules

For each component:
1. Create/vendor the Helm chart in `infrastructure/helm/{chart-name}/`
2. Create global values in `infrastructure/helm/_values/global/{chart-name}.yaml`
3. Add the ApplicationSet template to each `appset-{phase}/templates/{chart-name}.yaml`
4. Apply and verify via ArgoCD sync

### Phase 5: CI/CD pipelines

> Automate everything that was manual in Phases 1–4.

- [ ] **5.1** Create `.github/workflows/cluster-deploy.yml`
  - Trigger: PR to `main` touching `infrastructure/terragrunt/deploy/`
  - Steps: Assume role via OIDC → `terragrunt plan` → comment plan on PR → manual approve → `terragrunt apply`
- [ ] **5.2** Create `.github/workflows/helm-package.yml`
  - Trigger: PR to `main` touching `infrastructure/helm/`
  - Steps: Lint charts → package → push to OCI registry (ECR or Artifactory)
  - Parallel execution for multiple charts
- [ ] **5.3** Create `.github/workflows/container-build.yml`
  - Trigger: PR to `main` touching `infrastructure/container_image/`
  - Steps: Build multi-arch (ARM64) → scan for vulnerabilities → push to ECR
- [ ] **5.4** Create `.github/workflows/version-dashboard.yml`
  - Trigger: Cron (twice daily)
  - Steps: Parse all `terragrunt.hcl` files → extract versions, AMIs, regions → commit markdown dashboard
- [ ] **5.5** Set up branch protection rules
  - `main`: Require PR, require CI pass, require 1 approval
  - `develop`: Require CI pass

### Phase 6: Cluster automation

> Self-service cluster creation from a YAML config file.

- [ ] **6.1** Create `infrastructure/_template/_cluster/terragrunt.hcl` (Jinja2 template)
- [ ] **6.2** Create `infrastructure/_template/_cluster/cluster_config_template.yaml` (example config)
- [ ] **6.3** Implement `automation/scr/main.py` — orchestrator
- [ ] **6.4** Implement `automation/scr/clusterProcessor/` — YAML parsing + Jinja2 rendering
- [ ] **6.5** Implement `automation/scr/gitProcessor/` — GitHub API for branches + PRs
- [ ] **6.6** Implement `automation/scr/vpc_cni_constructor/` — AWS API to discover subnets + build eniConfig
- [ ] **6.7** Implement `automation/scr/preReqClusterCheck/` — validate prerequisites
- [ ] **6.8** Build automation container: `automation/Dockerfile`
- [ ] **6.9** Create GitHub Actions workflow to trigger automation on config YAML changes
- [ ] **6.10** Test end-to-end: create `infrastructure/_config/test-cluster.yaml` → automation creates PR with generated `terragrunt.hcl`

### Phase 7: Observability & security

> Add monitoring, logging, and security scanning to every cluster.

- [ ] **7.1** Deploy **Datadog** agent + cluster-agent (or your preferred APM)
- [ ] **7.2** Deploy **Prometheus** + Grafana for cluster-level metrics
- [ ] **7.3** Deploy **logging stack** (Coralogix, Splunk, or CloudWatch)
- [ ] **7.4** Deploy **Jaeger** for distributed tracing
- [ ] **7.5** Deploy **Kubecost** for cost monitoring
- [ ] **7.6** Deploy **Kiali** for Istio service mesh observability
- [ ] **7.7** Deploy **Twistlock / Prisma** (or Trivy) for container security scanning
- [ ] **7.8** Set up alerting rules (Terraform for alert definitions)
- [ ] **7.9** Create monitoring dashboards

### Phase 8: Multi-environment rollout

> Scale from one dev cluster to full dev/staging/prod with phased rollouts.

- [ ] **8.1** Deploy staging operations cluster (or reuse shared ops cluster)
- [ ] **8.2** Deploy first staging workload cluster
- [ ] **8.3** Create staging ApplicationSets (`appset-2-staging-*`)
- [ ] **8.4** Deploy prod operations cluster
- [ ] **8.5** Deploy first prod workload cluster
- [ ] **8.6** Create prod ApplicationSets (`appset-3-prod-*`)
- [ ] **8.7** Document the phased rollout process
  - Update dev phase → wait N days → update staging → wait N days → update prod
  - Same pattern for: EKS version upgrades, AMI rotations, helm chart updates, Terraform module updates
- [ ] **8.8** Create EKS upgrade workflow
  - GitHub Actions pipeline to auto-create work items per cluster for version upgrades

### Phase 9: Operational maturity

> Harden, document, and scale.

- [ ] **9.1** Write Architecture Decision Records (ADRs) for each major choice
  - Terragrunt for versioned deployments
  - Secrets management strategy
  - ArgoCD for GitOps
  - Istio for service mesh
  - Kyverno for policy enforcement
  - Network policy enforcement approach
  - Cilium CNI chaining decision
- [ ] **9.2** Write operational runbooks
  - Cluster provisioning, EKS upgrades, AMI rotation, disaster recovery (Velero), ArgoCD rollout procedures
- [ ] **9.3** Set up Backstage catalog entry (optional)
- [ ] **9.4** Implement FedRAMP compliance mode (if needed)
  - Private-only endpoints, full audit logging, 365-day log retention, FIPS images, CodeCommit credential rotation
- [ ] **9.5** Implement weighted DNS routing for multi-cluster traffic splitting
- [ ] **9.6** Set up cluster cleanup automation for deprovisioning

---

## 4. Key design decisions

### Why Terragrunt over plain Terraform?

The EKS module is written once in `modules/eks/` and versioned via git tags. Each cluster references a specific tag, so you can upgrade clusters independently. Terragrunt handles the S3 backend config, IAM role assumption, and DRY variable injection without duplicating provider blocks across 75+ cluster directories.

### Why a single module, not per-component modules?

Everything that constitutes "a cluster" lives in one module: the EKS cluster, node groups, IAM roles, RBAC, S3 buckets, secrets. This means `terragrunt apply` gives you a complete, working cluster in one shot. Components that are shared across clusters (ECR, Route53, ACM) live in `resources/` as separate modules.

### Why ArgoCD ApplicationSets over plain Applications?

With 75+ clusters and 47+ components, you'd need 3,500+ ArgoCD Application manifests. ApplicationSets with cluster generators reduce this to ~47 templates per phase. Add a cluster with the right `phase` label and it automatically gets all components.

### Why phase labels instead of environment labels?

Phases are more granular than environments. You might have `dev-01` (bleeding edge), `dev-10` (stable dev), `staging-05`, `prod-01` (canary prod), `prod-10` (main prod). This lets you roll changes through sub-phases within an environment.

### Addon migration: EKS managed vs Helm-managed

The module supports both modes via the `use_helm_for_addons` flag. Existing clusters use EKS managed addons. New clusters (or clusters being migrated) can use Helm-managed addons, which enables Cilium CNI chaining and finer control. The `bootstrap_self_managed_addons` logic handles the transition without cluster recreation.

### Container images from ECR, never public registries

All container images are pulled from ECR in the same AWS account as the cluster. The `infrastructure/container_image/` directory contains Dockerfiles that pull from upstream public registries and push to internal ECR. This ensures air-gapped clusters work and eliminates rate limit issues.

---

## 5. Component reference

### Core components deployed via ArgoCD (~47)

| Category | Components |
|----------|-----------|
| **Networking** | Istio base, Istio CNI, Istiod, Istio ingress gateway, Istio ztunnel, Cilium, AWS VPC CNI, AWS Load Balancer Controller (internal + public), External DNS |
| **Autoscaling** | Karpenter, Karpenter NodePool CRDs, KEDA, Metrics server |
| **Security** | Kyverno, Kyverno policies, Cert-manager, Network policies, Default namespace quotas |
| **Secrets** | External Secrets Operator, External Secrets Injector (ClusterSecretStore), Reloader |
| **Storage** | AWS EBS CSI driver |
| **Backup** | Velero |
| **Observability** | Datadog (agent + cluster-agent), Prometheus stack, Coralogix/Splunk collector, Jaeger operator + instance, Kiali operator + server, Kubecost |
| **AI/ML** | GPU operator, AWS Neuron device plugin, KubeRay operator, Knative serving |
| **CI/CD** | Argo Rollouts, ArgoCD bind credentials |
| **Debugging** | Chaos Mesh, Image cacher |
| **Auth** | OAuth2 proxy |

### Deployment standards

- snake_case for Terraform variables, kebab-case for resource names
- All resources named with prefix: `{asset_id}-{cluster_name}-{resource}`
- All images from same-account ECR
- Helm charts kept as vendor-provided; customizations via separate charts
- Minimum 3 replicas + PDB (minAvailable: 1) for all deployments
- Core components deployed to `operations` namespace (DaemonSets to `ops-{name}` namespace)
- `terraform fmt` before every commit
- 2-space indentation

---

## Quick start: Where to begin

**If starting from absolute zero, focus on these first:**

1. **Phase 0.2** (VPC) and **Phase 0.4** (TF state) — you can't do anything without networking and state storage
2. **Phase 1.2–1.8** (core module files) — get to a point where `terragrunt plan` succeeds
3. **Phase 2.1–2.5** (first cluster) — prove the module works
4. **Phase 3.1–3.5** (ArgoCD) — get GitOps running
5. **Phase 4.1–4.7** (essential components) — metrics, storage, secrets, autoscaling

Everything else builds on top of a working cluster with ArgoCD. Don't try to implement all 47 components at once — add them incrementally as teams need them.
