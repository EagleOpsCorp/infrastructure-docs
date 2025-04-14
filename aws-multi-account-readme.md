# AWS Multi-Account Architecture

This document provides a comprehensive overview of our enterprise AWS multi-account setup, incorporating **sandbox environments**, **ArgoCD for Kubernetes GitOps**, **AWS Service Catalog**, a **platform team**, and a **hybrid sandbox model** with dedicated accounts and Kubernetes namespaces.

## Table of Contents
1. [Architecture Overview](#1-overview-of-the-architecture)
2. [Account Responsibilities](#2-account-responsibilities)
3. [Provisioning Responsibilities](#3-provisioning-responsibilities)
4. [Team Cooperation](#4-team-cooperation)
5. [Cross-Account Access Management](#5-cross-account-access-management)
6. [Sandbox-to-Prod Evaluation Process](#6-sandbox-to-prod-evaluation-process)
7. [GitHub Structure](#7-github-structure)
8. [Example End-to-End Flow](#8-example-end-to-end-flow)
9. [Sandbox Setup](#9-sandbox-setup)
10. [Best Practices](#10-best-practices)
11. [Key Considerations](#11-key-considerations)

## 1. Overview of the Architecture

### Accounts and OUs
- **Operations**, **Workloads**, **Data**, **Networking**, **Observability**, **Security**: Each has **dev**, **stage**, **prod** accounts (e.g., `workloads-dev`, `workloads-prod`).
- **Sandbox**: A dedicated **Sandbox OU** with accounts (e.g., `sandbox-workloads`, `sandbox-data`, `sandbox-dev1`) and namespaces in `workloads-dev` EKS (e.g., `sandbox-user1`).
- Organized under AWS Organizations with OUs like `DevOU`, `ProdOU`, `SandboxOU`.

### GitHub Structure
- Directory-per-account-type repos: `workloads-infra`, `data-infra`, etc., with `environments/dev`, `stage`, `prod`.
- `k8s-manifests` for Kubernetes manifests (apps and sandboxes).
- `terraform-modules`, `aws-configs`, `infrastructure-docs` for shared resources.

### Tools
- **Terraform**: Provisions AWS infrastructure.
- **ArgoCD**: Manages Kubernetes apps in EKS.
- **Service Catalog**: Enables self-service provisioning in sandboxes.
- **GitHub Actions**: Automates IaC deployment.

### Teams
- **Platform Team**: Manages sandboxes, Service Catalog, coordination.
- **DevOps**, **Data**, **Networking**, **SRE**, **Security**, **App Teams** (e.g., **backend**), **Developers**.

## 2. Account Responsibilities

Each account has specific roles, supporting both infrastructure and Kubernetes-based applications.

### Operations Account (Dev, Stage, Prod)
- **Purpose**: Centralized hub for automation, self-service, and operational governance.
- **Responsibilities**:
  - **Dev**: Tests automation scripts, Service Catalog products (e.g., EKS templates).
  - **Stage**: Validates operational workflows (e.g., backup policies).
  - **Prod**:
    - Hosts **AWS Service Catalog** for provisioning sandbox and shared resources (e.g., S3, EKS).
    - Manages cross-account backups (AWS Backup).
    - Automates sandbox cleanup (Lambda with TTL, e.g., 7 days).
    - Runs patching and compliance checks (AWS Systems Manager).
    - Monitors costs across accounts (AWS Cost Explorer, Budgets).
- **Services**:
  - Service Catalog, Systems Manager, AWS Backup, Lambda, Cost Explorer, AWS Config.

### Workloads Account (Dev, Stage, Prod)
- **Purpose**: Hosts application workloads, primarily on **EKS clusters** for Kubernetes apps.
- **Responsibilities**:
  - **Dev**:
    - Runs EKS cluster for development.
    - Hosts sandbox namespaces (e.g., `sandbox-user1`, `sandbox-teamA`) for app testing.
    - Tests new app features (e.g., **backend** deployments).
  - **Stage**: Runs EKS for production-like testing, validates app scaling.
  - **Prod**: Hosts production EKS for customer-facing apps (e.g., **frontend**, **backend**).
  - **All**: Runs **ArgoCD** to sync Kubernetes manifests from `k8s-manifests`.
- **Services**:
  - EKS, EC2 (if non-Kubernetes needed), Lambda, Application Load Balancer (ALB).

### Data Account (Dev, Stage, Prod)
- **Purpose**: Manages data storage, processing, and analytics.
- **Responsibilities**:
  - **Dev**: Tests data pipelines, storage solutions (e.g., S3, RDS).
  - **Stage**: Validates data workflows with synthetic or anonymized data.
  - **Prod**:
    - Stores production data in S3, RDS, DynamoDB, Redshift.
    - Runs data pipelines (Glue, Athena, SageMaker).
    - Supports apps in `workloads` accounts (e.g., S3 for **backend**).
- **Services**:
  - S3, RDS, DynamoDB, Redshift, Glue, Athena, Lake Formation.

### Networking Account (Dev, Stage, Prod)
- **Purpose**: Centralizes network configuration for EKS and other services.
- **Responsibilities**:
  - **Dev**: Tests VPCs, subnets, Transit Gateway for sandboxes and EKS.
  - **Stage**: Simulates production networking.
  - **Prod**:
    - Configures VPCs, subnets, Transit Gateway for EKS connectivity.
    - Manages DNS (Route 53), load balancers (ALB for EKS ingresses).
    - Supports sandbox accounts with VPC peering or Transit Gateway.
- **Services**:
  - VPC, Transit Gateway, Route 53, ALB, CloudFront, Direct Connect.

### Observability Account (Dev, Stage, Prod)
- **Purpose**: Centralizes monitoring, logging, and tracing for AWS and Kubernetes.
- **Responsibilities**:
  - **Dev**: Tests monitoring tools (e.g., Prometheus, CloudWatch).
  - **Stage**: Validates dashboards, alerts for EKS and apps.
  - **Prod**:
    - Aggregates EKS metrics (Prometheus, Grafana in EKS; CloudWatch).
    - Centralizes logs (CloudWatch Logs, OpenSearch).
    - Monitors ArgoCD syncs, app performance, sandbox usage.
    - Manages alerts (SNS, EventBridge).
- **Services**:
  - CloudWatch, Prometheus, Grafana, X-Ray, SNS, EventBridge.

### Security Account (Dev, Stage, Prod)
- **Purpose**: Enforces security, identity management, and compliance.
- **Responsibilities**:
  - **Dev**: Tests IAM roles, KMS keys, security policies.
  - **Stage**: Validates security configurations for production readiness.
  - **Prod**:
    - Manages IAM roles for EKS, ArgoCD (IRSA), Service Catalog, and cross-account access.
    - Configures encryption (KMS, ACM).
    - Audits compliance (GuardDuty, Security Hub, Config).
    - Enforces SCPs for all accounts, including sandboxes.
- **Services**:
  - IAM, KMS, GuardDuty, Security Hub, AWS Config, ACM.

### Sandbox Accounts (Multiple)
- **Purpose**: Provides isolated environments for developer and team experimentation.
- **Responsibilities**:
  - Host temporary AWS resources (e.g., EKS, S3, RDS) via **Service Catalog**.
  - Support Kubernetes namespaces in `workloads-dev` EKS (e.g., `sandbox-user1`).
  - Test apps, infrastructure, or data pipelines (e.g., **backend** feature, Redshift cluster).
  - Auto-deleted after TTL (e.g., 7 days) by `operations-prod` Lambda.
  - **Specific Accounts**:
    - `sandbox-workloads`: Tests EKS clusters for **Workloads** team.
    - `sandbox-data`: Tests S3, RDS for **Data** team.
    - `sandbox-networking`: Tests VPCs for **Networking** team.
    - `sandbox-security`: Tests IAM (optional, for **Security** team).
    - `sandbox-dev1`: Shared account for **Observability**, **Operations**, and general use.
- **Services**:
  - EKS, S3, RDS, Lambda, SageMaker, EC2 (per team/developer needs).

## 3. Provisioning Responsibilities

Infrastructure and applications are provisioned by specific teams, using **Terraform** for AWS resources and **ArgoCD** for Kubernetes manifests.

### Operations Account
- **Infrastructure**:
  - Service Catalog products (e.g., EKS, S3 templates).
  - Automation tools (Systems Manager, Lambda for sandbox cleanup).
  - Backup policies (AWS Backup).
  - Cost monitoring (Cost Explorer).
- **Who Provisions**: **Platform Team**.
  - Uses `operations-infra/environments/prod` for IaC.
  - Example: `operations-infra/environments/prod/service-catalog/s3-product.yaml`:
    ```yaml
    AWSTemplateFormatVersion: "2010-09-09"
    Resources:
      S3Bucket:
        Type: AWS::S3::Bucket
        Properties:
          BucketName: !Sub "sandbox-${AWS::AccountId}-${AWS::Region}"
          Tags:
            - Key: Environment
              Value: sandbox
    ```
- **Applications**: None (infrastructure-focused).

### Workloads Account
- **Infrastructure**:
  - EKS clusters for **dev**, **stage**, **prod**.
  - Sandbox namespaces in `workloads-dev` EKS (e.g., `sandbox-user1`).
  - Occasional non-Kubernetes resources (e.g., EC2, if needed).
- **Who Provisions**: **DevOps Team**.
  - Uses `workloads-infra` for Terraform.
  - Example: `workloads-infra/environments/prod/main.tf`:
    ```hcl
    module "eks" {
      source          = "terraform-aws-modules/eks/aws"
      cluster_name    = "web-app-prod"
      cluster_version = "1.27"
      vpc_id          = data.terraform_remote_state.networking.outputs.vpc_id
      subnet_ids      = data.terraform_remote_state.networking.outputs.private_subnets
      enable_irsa     = true
    }
    ```
- **Applications**:
  - Kubernetes apps (e.g., **frontend**, **backend**) in EKS.
  - **Who Provisions**: **App Teams** (e.g., **backend** team).
    - Uses `k8s-manifests/apps/backend` for manifests.
    - Example: `k8s-manifests/apps/backend/prod/deployment.yaml`:
      ```yaml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: backend
        namespace: backend
      spec:
        replicas: 3
        template:
          spec:
            containers:
            - name: backend
              image: acmecorp/backend:prod-1.0.0
      ```
  - **ArgoCD** syncs manifests to EKS.

### Data Account
- **Infrastructure**:
  - S3 buckets, RDS, DynamoDB, Redshift, Glue jobs.
  - Data resources for sandbox accounts (via Service Catalog).
- **Who Provisions**: **Data Engineering Team**.
  - Uses `data-infra` for Terraform.
  - Example: `data-infra/environments/prod/main.tf`:
    ```hcl
    resource "aws_s3_bucket" "backend_data" {
      bucket = "acme-backend-data-prod"
    }
    ```
- **Applications**:
  - Data pipelines (e.g., Glue jobs, Kubernetes-based ETL).
  - **Who Provisions**: **Data Engineering Team**.
    - Uses `k8s-manifests/apps/data-pipeline` for Kubernetes pipelines.

### Networking Account
- **Infrastructure**:
  - VPCs, subnets, Transit Gateway, Route 53, ALB.
  - Networking for sandbox accounts (e.g., VPC peering).
- **Who Provisions**: **Networking Team**.
  - Uses `networking-infra` for Terraform.
  - Example: `networking-infra/environments/prod/main.tf`:
    ```hcl
    module "vpc" {
      source      = "terraform-aws-modules/vpc/aws"
      name        = "prod-vpc"
      cidr        = "10.0.0.0/16"
      subnets     = ["10.0.1.0/24", "10.0.2.0/24"]
    }
    ```
- **Applications**: None (infrastructure-focused).

### Observability Account
- **Infrastructure**:
  - CloudWatch dashboards, log groups, metric alarms.
  - Prometheus/Grafana infrastructure (if not in EKS).
- **Who Provisions**: **SRE Team**.
  - Uses `observability-infra` for Terraform.
  - Example: `observability-infra/environments/prod/main.tf`:
    ```hcl
    resource "aws_cloudwatch_dashboard" "backend" {
      dashboard_name = "backend-prod"
      dashboard_body = jsonencode({...})
    }
    ```
- **Applications**:
  - Monitoring tools (e.g., Prometheus, Grafana) in EKS.
  - **Who Provisions**: **SRE Team**.
    - Uses `k8s-manifests/apps/prometheus`.

### Security Account
- **Infrastructure**:
  - IAM roles, KMS keys, GuardDuty, Security Hub.
  - IRSA for EKS pods and ArgoCD.
  - SCPs for all accounts, including sandboxes.
- **Who Provisions**: **Security Team**.
  - Uses `security-infra` for Terraform.
  - Example: `security-infra/environments/prod/main.tf`:
    ```hcl
    module "irsa_backend" {
      source      = "terraform-aws-modules/iam/aws//modules/iam-assumable-role-with-oidc"
      role_name   = "backend-prod"
      oidc_sub    = "system:serviceaccount:backend:backend-sa"
      policy_json = jsonencode({
        Statement = [{
          Effect   = "Allow"
          Action   = ["s3:GetObject"]
          Resource = "arn:aws:s3:::acme-backend-data-prod/*"
        }]
      })
    }
    ```
- **Applications**: None (infrastructure-focused).

### Sandbox Accounts
- **Infrastructure**:
  - Temporary resources (e.g., EKS, S3, RDS) in accounts like `sandbox-workloads`, `sandbox-data`, `sandbox-dev1`.
  - Provisioned via **Service Catalog**.
- **Applications**:
  - Test apps in Kubernetes namespaces (e.g., `sandbox-user1` in `workloads-dev` EKS).
- **Who Provisions**:
  - **Developers**: Deploy resources via Service Catalog or `k8s-manifests/sandboxes`.
    - Example: `k8s-manifests/sandboxes/user1/deployment.yaml`:
      ```yaml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: backend
        namespace: sandbox-user1
      spec:
        template:
          spec:
            containers:
            - name: backend
              image: acmecorp/backend:test-1.0.0
      ```
  - **Platform Team**: Provisions sandbox accounts, maintains Service Catalog.
    - Uses `operations-infra/environments/sandbox`.

## 4. Team Cooperation

Teams collaborate via **GitHub**, **Service Catalog**, **ArgoCD**, and communication tools to manage infrastructure, apps, and sandbox-to-prod workflows.

### Platform Team
- **Role**: Orchestrates sandboxes, Service Catalog, and cross-team coordination.
- **Tasks**:
  - Maintains `operations-infra` for Service Catalog, sandbox automation (e.g., account creation, cleanup Lambda).
  - Reviews sandbox requests (e.g., PRs to `k8s-manifests/sandboxes`).
  - Evaluates developer sandbox findings for prod changes.
  - Creates PRs/tickets for infrastructure updates (e.g., `data-infra` for S3).
  - Syncs with teams via Jira/Slack.
- **Collaboration**:
  - Works with **developers** to scope sandbox experiments.
  - Submits PRs to `workloads-infra`, `data-infra` for approved changes.
  - Engages **SRE** for monitoring, **security** for IAM.

### DevOps Team
- **Role**: Manages EKS and app infrastructure in `workloads` accounts.
- **Tasks**:
  - Provisions EKS clusters via `workloads-infra`.
  - Scales EKS based on platform team requests (e.g., new nodes for **backend**).
  - Reviews ArgoCD app deployments in `k8s-manifests`.
- **Collaboration**:
  - Reviews platform team PRs for EKS changes.
  - Works with **app teams** to ensure app stability in EKS.
  - Coordinates with **networking** for ALB, VPC peering.

### App Teams (e.g., frontend, backend)
- **Role**: Develops and deploys Kubernetes apps.
- **Tasks**:
  - Updates `k8s-manifests/apps/frontend` for app deployments.
  - Tests features in sandbox namespaces (e.g., `k8s-manifests/sandboxes/user1`).
  - Proposes prod changes to platform team.
- **Collaboration**:
  - Submits sandbox findings to **platform team**.
  - Works with **DevOps** for EKS access.
  - Relies on **data team** for S3, RDS access.

### Data Engineering Team
- **Role**: Manages data infrastructure and pipelines.
- **Tasks**:
  - Provisions S3, RDS, Redshift via `data-infra`.
  - Supports sandbox data needs (e.g., S3 in `sandbox-data`).
  - Deploys pipelines via `k8s-manifests/apps/data-pipeline`.
- **Collaboration**:
  - Reviews platform team PRs for data changes.
  - Works with **app teams** to provide data.
  - Coordinates with **security** for encryption.

### Networking Team
- **Role**: Manages network infrastructure.
- **Tasks**:
  - Provisions VPCs, subnets, Transit Gateway via `networking-infra`.
  - Configures networking for sandboxes (e.g., `sandbox-networking`).
  - Supports EKS ALBs.
- **Collaboration**:
  - Reviews platform team PRs for network updates.
  - Works with **DevOps** for EKS connectivity.
  - Coordinates with **security** for firewall rules.

### SRE Team
- **Role**: Ensures reliability, observability.
- **Tasks**:
  - Provisions monitoring via `observability-infra`.
  - Deploys Prometheus/Grafana via `k8s-manifests/apps/prometheus`.
  - Monitors sandboxes, EKS, and apps.
- **Collaboration**:
  - Reviews platform team PRs for monitoring.
  - Works with **DevOps** to optimize EKS.
  - Coordinates with **security** for compliance.

### Security Team
- **Role**: Enforces security, compliance.
- **Tasks**:
  - Provisions IAM, KMS via `security-infra`.
  - Reviews Service Catalog products, sandbox IAM.
  - Audits with GuardDuty, Security Hub.
- **Collaboration**:
  - Reviews all PRs for IAM, RBAC (`security-infra`, `k8s-manifests`).
  - Works with **platform team** to secure sandboxes.
  - Coordinates with **SRE** for incidents.

### Developers
- **Role**: Experiments in sandboxes, develops apps.
- **Tasks**:
  - Provisions resources via Service Catalog (e.g., S3 in `sandbox-user1`).
  - Tests apps in `k8s-manifests/sandboxes/user1`.
  - Submits prod proposals to platform team.
- **Collaboration**:
  - Works with **platform team** to promote features.
  - Integrates with **app teams** for deployment.
  - Relies on **DevOps** for EKS.

### Collaboration Tools
- **GitHub**: PRs for IaC (`workloads-infra`), manifests (`k8s-manifests`).
- **Jira/Slack**: Tracks sandbox findings, prod changes.
- **Service Catalog**: Self-service for developers.
- **ArgoCD**: Automates Kubernetes deployments.
- **infrastructure-docs**: Runbooks, sandbox guides.

## 5. Cross-Account Access Management

Cross-account access is managed securely to enable collaboration while enforcing least privilege.

### AWS Organizations
- **OUs**: `DevOU`, `StageOU`, `ProdOU`, `SandboxOU`.
- **SCPs** (from `security-prod`):
  - **Sandbox OU**: Restricts actions (e.g., no public S3 buckets, `t3.micro` only).
    ```json
    {
      "Effect": "Deny",
      "Action": ["s3:MakeBucketPublic", "ec2:RunInstances"],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "ec2:InstanceType": ["t3.micro", "t3.nano"]
        }
      }
    }
    ```
  - **Prod OU**: Limits regions (e.g., `us-east-1`).
  - Managed in `security-infra`.

### IAM Roles
- **Cross-Account Roles**:
  - Each account defines roles for specific access.
  - Example: `workloads-prod` role (`WorkloadsAccessRole`) allows `operations-prod` to manage backups:
    ```hcl
    resource "aws_iam_role" "workloads_access" {
      name = "WorkloadsAccessRole"
      assume_role_policy = jsonencode({
        Statement = [{
          Effect    = "Allow"
          Principal = { AWS = "arn:aws:iam::${var.operations_account_id}:root" }
          Action    = "sts:AssumeRole"
        }]
      })
      inline_policy {
        name = "backup_policy"
        policy = jsonencode({
          Statement = [{
            Effect   = "Allow"
            Action   = ["backup:*"]
            Resource = "*"
          }]
        })
      }
    }
    ```
  - **ArgoCD** in `workloads-prod` uses IRSA (from `security-prod`) to access S3 in `data-prod`.
- **Sandbox Roles**:
  - Limited roles in `sandbox-user1` (e.g., `SandboxDeveloperRole`) allow Service Catalog access.
- **Centralized IAM**:
  - `security-prod` defines policies, delegates role creation to accounts.

### Resource-Based Policies
- **S3**: `data-prod` bucket policy allows `workloads-prod` role.
- **KMS**: Keys in `security-prod` allow decryption by `workloads`, `data` accounts.
- **Service Catalog**: `operations-prod` portfolio shared with sandbox accounts.

### Kubernetes RBAC
- **ArgoCD** manages namespaces in `workloads` EKS.
- Example: `sandbox-user1` RBAC:
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    namespace: sandbox-user1
    name: user1-role
  rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "create", "update"]
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    namespace: sandbox-user1
    name: user1-binding
  subjects:
  - kind: User
    name: user1
  roleRef:
    kind: Role
    name: user1-role
  ```
- **ArgoCD RBAC**:
  - Restricts teams to their namespaces (e.g., `@backend` edits `backend`).

### Networking
- **Transit Gateway**: Connects all accounts (`workloads`, `data`, `sandboxes`).
- **VPC Peering**: Links `workloads-prod` to `data-prod` for EKS-to-S3 access.
- **Security Groups**: Restrict traffic (e.g., EKS pods to RDS).
- Managed in `networking-infra`.

### Service Catalog
- Developers assume `SandboxDeveloperRole` to access `operations-prod` Service Catalog.
- Example: Allows `servicecatalog:ProvisionProduct` for sandbox resources.

## 6. Sandbox-to-Prod Evaluation Process

A key workflow is evaluating whether developer updates in **sandboxes** (provisioned via Service Catalog) require **production infrastructure** changes. Here's how it works:

### Step 1: Developer Tests in Sandbox
- Developer uses Service Catalog to deploy resources (e.g., S3 in `sandbox-user1`).
- Tests apps in `k8s-manifests/sandboxes/user1` (e.g., **backend** feature).
- Example: Deploys `backend:test-1.0.0` pod, queries `s3://sandbox-user1-test`.
- Outcome: Identifies prod needs (e.g., S3 bucket in `data-prod`, IRSA role).

### Step 2: Developer Submits Findings
- Documents results in Jira, GitHub issue, or Slack.
- Example Jira: "Backend feature needs `data-prod` S3, IRSA for `workloads-prod` EKS."
- Provides sandbox logs, manifests, metrics.

### Step 3: Platform Team Reviews
- Evaluates based on:
  - **Technical Feasibility**: Does the feature work (e.g., S3 queries succeed)?
  - **Business Value**: Is it prioritized (e.g., confirmed by **backend** team lead)?
  - **Prod Compatibility**: Fits prod standards (e.g., KMS encryption)?
  - **Cost**: Estimates S3, EKS costs (via Cost Explorer).
  - **Security**: Compliant with SCPs, IAM policies?
- Meets with developer, reviews sandbox artifacts (e.g., `k8s-manifests/sandboxes/user1`).
- Outcome: Approves need for prod changes (e.g., S3, IRSA).

### Step 4: Engage Teams
- Platform team creates PRs/tickets:
  - `data-infra/environments/prod` for S3.
  - `security-infra/environments/prod` for IRSA.
  - Jira for **DevOps** to assess EKS scaling.
- Consults **data**, **security**, **DevOps**, **SRE**, **networking** teams.
- Example: **Data** team suggests S3 lifecycle policy; **DevOps** confirms EKS capacity.

### Step 5: Technical/Risk Assessment
- Teams validate changes:
  - **Data**: Tests S3 in `data-dev`.
  - **DevOps**: Simulates EKS scaling in `workloads-stage`.
  - **Security**: Ensures IRSA is scoped.
  - **SRE**: Plans metrics.
- Assess risks (e.g., S3 downtime, EKS limits).
- Outcome: Refined plan (e.g., S3 with lifecycle, no EKS scaling needed).

### Step 6: Approval
- Teams approve PRs (e.g., `@data`, `@security` for `data-infra`).
- GitHub Environments (`prod`) require manual approval.
- Product manager confirms priority.
- Outcome: Changes greenlit.

### Step 7: Deployment
- PRs merged, `terraform apply` deploys S3, IRSA.
- Developer updates `k8s-manifests/apps/backend/prod`, ArgoCD syncs to `workloads-prod` EKS.
- **SRE** monitors via `observability-prod`.

### Who's Involved
- **Developer**: Submits findings.
- **Platform Team**: Leads evaluation, PRs.
- **DevOps**, **Data**, **Security**, **SRE**, **Networking**: Review, deploy.
- **App Team**: Updates manifests.
- **Product Manager**: Validates priority.

## 7. GitHub Structure

The GitHub organization supports all workflows:

```
AcmeCorp/
├── terraform-modules/        # Reusable modules (e.g., eks, s3)
├── aws-configs/             # Account IDs, SCPs
├── infrastructure-docs/      # Runbooks, sandbox guide
├── workloads-infra/         # EKS IaC
│   ├── environments/
│   │   ├── dev/
│   │   ├── stage/
│   │   └── prod/
├── data-infra/              # S3, RDS IaC
├── networking-infra/        # VPC IaC
├── operations-infra/        # Service Catalog, cleanup
│   ├── environments/
│   │   ├── prod/
│   │   │   ├── service-catalog/
│   │   │   └── cleanup_lambda/
│   │   └── sandbox/
├── observability-infra/     # Monitoring IaC
├── security-infra/          # IAM, KMS IaC
├── k8s-manifests/           # Kubernetes manifests
│   ├── apps/
│   │   ├── frontend/
│   │   ├── backend/
│   │   └── prometheus/
│   ├── sandboxes/
│   │   ├── user1/
│   │   ├── teamA/
│   │   └── data-team/
│   └── bootstrap/
├── app-frontend/            # App code
├── app-backend/
└── data-pipeline/
```

- **Terraform**:
  - `*-infra` repos manage IaC with `environments/<env>`.
  - Example: `workloads-infra/environments/prod/main.tf` for EKS.
- **ArgoCD**:
  - `k8s-manifests` manages apps (`apps/backend`) and sandboxes (`sandboxes/user1`).
  - Example: `k8s-manifests/apps/backend/prod/deployment.yaml`.
- **Workflows**:
  - **IaC**: GitHub Actions run `terraform apply` on PR merge.
    - Example: `workloads-infra/.github/workflows/terraform-prod.yml`.
  - **Apps**: PRs to `k8s-manifests` trigger ArgoCD syncs.
  - **Sandboxes**: Service Catalog or PRs to `k8s-manifests/sandboxes`.

## 8. Example End-to-End Flow

**Scenario**: Developer tests a new **backend** feature, promotes to prod.

1. **Sandbox**:
   - Developer uses Service Catalog to deploy S3 in `sandbox-user1`.
   - Creates `k8s-manifests/sandboxes/user1/deployment.yaml`:
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: backend
       namespace: sandbox-user1
     spec:
       template:
         spec:
           containers:
           - name: backend
             image: acmecorp/backend:test-1.0.0
             env:
             - name: S3_BUCKET
               value: "sandbox-user1-test"
     ```
   - **ArgoCD** syncs to `workloads-dev` EKS.
   - Developer confirms feature needs prod S3.

2. **Platform Team**:
   - Reviews sandbox findings (Jira ticket: "Need `data-prod` S3").
   - Evaluates feasibility, cost, security.
   - Submits PR to `data-infra/environments/dev` for S3:
     ```hcl
     resource "aws_s3_bucket" "backend_data" {
       bucket = "acme-backend-data-dev"
     }
     ```
   - Creates ticket for **DevOps** to assess EKS.

3. **DevOps**:
   - Confirms EKS capacity, no scaling needed.
   - Reviews `workloads-infra` PRs if required.

4. **Data Team**:
   - Merges `data-infra` PR, deploys S3 in `data-dev`.

5. **App Team**:
   - Submits PR to `k8s-manifests/apps/backend/dev`:
     ```yaml
     spec:
       containers:
       - name: backend
         image: acmecorp/backend:dev-1.0.0
         env:
         - name: S3_BUCKET
           value: "acme-backend-data-dev"
     ```
   - **ArgoCD** syncs to `workloads-dev` EKS.

6. **SRE**:
   - Updates `observability-infra` for S3, **backend** metrics.

7. **Security**:
   - Designs IRSA in `security-infra`.

8. **Promotion**:
   - Feature tested in **dev**, **stage**.
   - PRs to `data-infra`, `security-infra`, `k8s-manifests/apps/backend/prod`.
   - **ArgoCD** deploys to `workloads-prod` EKS.

## 9. Sandbox Setup

- **Hybrid Model**:
  - **Accounts**:
    - `sandbox-workloads`: EKS testing.
    - `sandbox-data`: S3, RDS testing.
    - `sandbox-networking`: VPC testing.
    - `sandbox-security` (optional): IAM testing.
    - `sandbox-dev1`: Shared for **Observability**, **Operations**, general use.
  - **Namespaces**: In `workloads-dev` EKS (e.g., `sandbox-user1`, `sandbox-data-team`).
- **Provisioning**:
  - **Service Catalog** in `operations-prod` for AWS resources.
  - **ArgoCD** for namespaces via `k8s-manifests/sandboxes`.
- **Cleanup**:
  - Lambda in `operations-prod` deletes sandboxes after 7 days.
  - Tags enforce TTL (e.g., `TTL: 7d`).

## 10. Best Practices

- **Isolation**:
  - Sandboxes prevent interference with **dev**/**prod**.
  - SCPs, RBAC limit sandbox scope.
- **Cost**:
  - Budgets per sandbox account (e.g., $200/month).
  - Quotas for namespaces (e.g., 4 CPUs).
- **Security**:
  - IRSA for EKS pods.
  - GuardDuty audits sandboxes.
- **Automation**:
  - Service Catalog for self-service.
  - ArgoCD for app deployment.
  - Lambda for cleanup.
- **Documentation**:
  - `infrastructure-docs/sandbox-guide.md` for usage.
  - Runbooks for prod promotion.

## 11. Key Considerations

- **Scalability**:
  - Add sandboxes via Service Catalog, ArgoCD.
  - Scale EKS via `workloads-infra`.
- **Governance**:
  - **Security** enforces SCPs, audits.
  - **Platform team** ensures compliance.
- **Developer Empowerment**:
  - Sandboxes enable safe experimentation.
  - Platform team bridges to prod.