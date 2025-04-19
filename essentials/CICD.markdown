# CI/CD Pipeline for CNPG Database Migration with Velero

This repository contains an end-to-end CI/CD pipeline for deploying and managing a **CloudNativePG (CNPG)**-managed PostgreSQL database and **Velero** for disaster recovery in a Kubernetes environment. The pipeline automates the setup, testing, and deployment of the CNPG operator, PostgreSQL clusters, and Velero backups, ensuring reliability, high availability (HA), and disaster recovery (DR) for the database workload. The pipeline is designed for a sandbox environment (using `kind`) and is extensible to development, staging, and production environments.

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [CI/CD Pipeline](#ci-cd-pipeline)
  - [GitHub Actions Workflow](#github-actions-workflow)
  - [ArgoCD GitOps Integration](#argocd-gitops-integration)
- [Setup Instructions](#setup-instructions)
- [Testing and Validation](#testing-and-validation)
- [Extending to Other Environments](#extending-to-other-environments)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## Overview
The CI/CD pipeline automates the deployment of a CNPG-managed PostgreSQL cluster, replacing a problematic StatefulSet-based database, as part of a migration project. It integrates **Velero** for cluster-wide backups and restores, complementing CNPG’s database-level backups and point-in-time recovery (PITR). The pipeline uses **GitHub Actions** for continuous integration and deployment, and **ArgoCD** for GitOps-based management, ensuring consistent and reproducible deployments across environments.

Key objectives:
- Deploy CNPG operator and PostgreSQL clusters with HA and DR capabilities.
- Configure Velero for backing up Kubernetes resources and persistent volumes (PVs).
- Automate testing of application connectivity, failover, backups, and restores.
- Support GitOps workflows with ArgoCD for declarative configuration management.

## Architecture
The pipeline operates in a Kubernetes cluster (e.g., `kind` for sandbox) and includes:
- **CNPG Operator**: Manages PostgreSQL clusters with automated failover, replication, and backups to MinIO (S3-compatible storage).
- **Velero**: Backs up Kubernetes resources (e.g., CNPG clusters, secrets, PVs) to MinIO, with pre-backup hooks to trigger CNPG backups.
- **MinIO**: Provides S3-compatible storage for CNPG and Velero backups.
- **GitHub Actions**: Builds, tests, and deploys the CNPG and Velero configurations.
- **ArgoCD**: Synchronizes Kubernetes manifests from the repository to the cluster.
- **Test Application**: A Node.js app to validate database connectivity.

## Prerequisites
- **Local Tools**:
  - `kind` (v0.20.0) for creating a local Kubernetes cluster.
  - `kubectl` for cluster interactions.
  - `helm` (v3.x) for installing CNPG and Velero.
  - `velero` CLI (v1.15.0) for managing backups.
  - `docker` for running containers.
- **GitHub Repository**:
  - Fork or clone this repository.
  - Configure GitHub Secrets for sensitive data (e.g., MinIO credentials).
- **Kubernetes Cluster**:
  - A running `kind` cluster or access to a cloud-based cluster (e.g., EKS).
  - Storage class supporting CSI snapshots (e.g., `rancher.io/local-path`).
- **ArgoCD**:
  - Installed in the cluster for GitOps management.

## Repository Structure
```
├── manifests/
│   ├── cnpg-cluster.yaml         # CNPG PostgreSQL cluster configuration
│   ├── velero-minio-credentials.yaml # Velero MinIO credentials secret
│   ├── velero-backup-schedule.yaml   # Velero backup schedule
├── scripts/
│   ├── setup-kubernetes-cluster.sh  # Sets up kind cluster
│   ├── deploy-cnpg.sh              # Deploys CNPG operator and cluster
│   ├── deploy-velero.sh            # Deploys Velero
│   ├── migrate-and-test.sh         # Migrates data and tests CNPG
│   ├── test-velero.sh             # Tests Velero backups/restores
├── .github/workflows/
│   ├── ci-cd.yaml                 # GitHub Actions workflow
├── README.md                      # This file
```

## CI/CD Pipeline

### GitHub Actions Workflow
The pipeline is defined in `.github/workflows/ci-cd.yaml` and includes the following stages:
1. **Setup**: Initialize the `kind` cluster and install dependencies (`kubectl`, `helm`, `velero`).
2. **Lint and Validate**: Check Kubernetes manifests and Helm charts for errors using `kubeval` and `helm lint`.
3. **Deploy CNPG**: Install the CNPG operator, deploy the PostgreSQL cluster, and migrate data from a mock StatefulSet database.
4. **Deploy Velero**: Install Velero and configure backup schedules.
5. **Test**: Validate application connectivity, CNPG failover, Velero backups, and restores.
6. **Push to ArgoCD**: Commit updated manifests to the repository for ArgoCD to synchronize.
7. **Cleanup**: Tear down the `kind` cluster (optional, for sandbox).

#### Example Workflow
```yaml
name: CI/CD Pipeline for CNPG and Velero
on:
  push:
    branches:
      - main
  pull_request:
jobs:
  ci-cd:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Set up kind cluster
        run: ./scripts/setup-kubernetes-cluster.sh
      - name: Install Velero CLI
        run: ./scripts/velero-cli-install.sh
      - name: Lint manifests
        run: |
          helm lint charts/cnpg
          kubeval manifests/*.yaml
      - name: Deploy CNPG
        run: ./scripts/deploy-cnpg.sh
      - name: Migrate and test CNPG
        run: |
          kubectl apply -f manifests/mock-statefulset-db.yaml
          ./scripts/migrate-and-test.sh
      - name: Deploy Velero
        run: ./scripts/deploy-velero.sh
      - name: Test Velero
        run: |
          kubectl apply -f manifests/velero-backup-schedule.yaml
          ./scripts/test-velero.sh
      - name: Commit manifests for ArgoCD
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add manifests/
          git commit -m "Update manifests from CI/CD"
          git push
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - name: Cleanup
        if: always()
        run: kind delete cluster --name cnpg-sandbox
```

### ArgoCD GitOps Integration
ArgoCD is used to synchronize the Kubernetes manifests from the `manifests/` directory to the cluster, ensuring declarative and version-controlled deployments.

#### ArgoCD Application Configuration
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cnpg-velero
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-repo>/cnpg-velero
    targetRevision: main
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: cnpg-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Setup Instructions
1. **Clone the Repository**:
   ```bash
   git clone https://github.com/<your-repo>/cnpg-velero
   cd cnpg-velero
   ```

2. **Configure GitHub Secrets**:
   - Add `GITHUB_TOKEN` for pushing manifests.
   - Add `MINIO_ACCESS_KEY` and `MINIO_SECRET_KEY` for MinIO access (default: `minioadmin`).

3. **Set Up ArgoCD**:
   - Install ArgoCD in the cluster:
     ```bash
     kubectl create namespace argocd
     kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
     ```
   - Apply the ArgoCD Application manifest:
     ```bash
     kubectl apply -f manifests/argocd-application.yaml
     ```

4. **Run the Pipeline**:
   - Push changes to the `main` branch or open a pull request to trigger the GitHub Actions workflow.
   - Monitor the pipeline in the GitHub Actions tab.

5. **Verify Deployment**:
   - Check ArgoCD UI for sync status:
     ```bash
     kubectl port-forward svc/argocd-server -n argocd 8080:443
     ```
   - Access `http://localhost:8080` (default password: `admin`/`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`).
   - Verify CNPG and Velero resources:
     ```bash
     kubectl get pods -n cnpg-system
     velero backup get -n cnpg-system
     ```

## Testing and Validation
The pipeline includes automated tests in the `migrate-and-test.sh` and `test-velero.sh` scripts:
- **CNPG Tests**:
  - Data migration from a mock StatefulSet database.
  - Application connectivity using a Node.js test app.
  - Failover simulation by deleting the primary pod.
  - Backup creation and validation.
- **Velero Tests**:
  - Manual backup of the `cnpg-system` namespace.
  - Disaster simulation by deleting the CNPG cluster.
  - Restore and data integrity validation.
  - CNPG PITR test post-restore.

Run tests manually:
```bash
./scripts/migrate-and-test.sh
./scripts/test-velero.sh
```

## Extending to Other Environments
To adapt the pipeline for development, staging, or production:
1. **Cluster Configuration**:
   - Replace `kind` with a cloud provider (e.g., EKS) in `setup-kubernetes-cluster.sh`.
   - Update storage classes and CSI drivers for production-grade storage.
2. **Storage Backend**:
   - Replace MinIO with AWS S3 or equivalent.
   - Update CNPG and Velero configurations (`cnpg-cluster.yaml`, `deploy-velero.sh`).
3. **Environment-Specific Settings**:
   - Use Helm values files or Kustomize for environment-specific configurations (e.g., resource limits, connection strings).
   - Example: `manifests/dev/`, `manifests/staging/`, `manifests/prod/`.
4. **Security**:
   - Use IAM roles for S3 access instead of static credentials.
   - Integrate with a secrets manager (e.g., AWS Secrets Manager, HashiCorp Vault).
5. **Monitoring**:
   - Integrate CNPG and Velero metrics with Prometheus and Grafana.
   - Set up alerts for backup failures or failover events.

## Troubleshooting
- **Pipeline Failures**:
  - Check GitHub Actions logs for errors.
  - Ensure GitHub Secrets are correctly configured.
- **CNPG Issues**:
  - Verify pod logs: `kubectl logs -n cnpg-system -l cnpg.io/cluster=test-cluster`.
  - Check CNPG cluster status: `kubectl get cluster -n cnpg-system`.
- **Velero Issues**:
  - Inspect backup logs: `velero backup logs <backup-name> -n cnpg-system`.
  - Ensure MinIO is accessible: `kubectl port-forward svc/minio 9000:9000 -n cnpg-system`.
- **ArgoCD Sync Failures**:
  - Check ArgoCD events: `kubectl describe application cnpg-velero -n argocd`.
  - Verify repository access in ArgoCD settings.

## License
This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.