# Artisanal Craft Store: Enhanced Database and Infrastructure Migration to CNPG with AWS Services

This `README.md` outlines the process of enhancing the Artisanal Craft Store’s Kubernetes-based e-commerce platform (aligned with the OpenTelemetry demo: https://github.com/open-telemetry/opentelemetry-demo) by:
1. Deploying AWS ElastiCache (instead of Valkey) and Amazon MSK (instead of simple Kafka) from the AWS Service Catalog, and updating manifests to connect to these services.
2. Deploying AWS Secrets Manager and implementing a secret management system using External Secrets Operator for the EKS cluster.
3. Installing CloudNativePG (CNPG) CRDs into the Helmfile used in the GitHub Actions (GHA) pipeline.
4. Creating CNPG YAMLs to replace StatefulSet-based PostgreSQL deployments.
5. Providing migration scripts to transfer schemas, accounts, and data from the StatefulSet database to CNPG, including endpoint updates.
6. Validating CNPG functionality (HA, DR, failover, backups).
7. Creating ArgoCD YAMLs and a GHA pipeline for deploying the application with CNPG.
8. Adding Velero CRDs and YAMLs to the platform GHA pipeline for backups, working in synergy with ArgoCD and CNPG.

This guide assumes the scaled infrastructure for a $2M revenue Artisanal Craft Store running on Amazon EKS.

---

## Prerequisites
- **EKS Cluster**: Kubernetes 1.24, 6–8 t3.large nodes, 3 AZs (us-east-1).
- **Tools**: Helm, Helmfile, kubectl, GitHub Actions, ArgoCD, AWS CLI, Terraform (optional).
- **AWS Services**: Service Catalog, ElastiCache, MSK, Secrets Manager, S3, EBS (gp3), CloudWatch.
- **Existing Setup**: StatefulSet-based PostgreSQL 14, microservices (Product Catalog, Inventory, etc.), partial Prometheus/Grafana monitoring.
- **Namespaces**: `cnpg-system` (CNPG operator), `prod-db` (CNPG cluster), `monitoring` (Prometheus/Grafana), `velero` (Velero), `external-secrets` (External Secrets Operator).
- **Access**: IAM roles for Service Catalog, S3, ElastiCache, MSK, Secrets Manager, and EKS (IRSA configured).

---

## 1. Deploying AWS ElastiCache and MSK from Service Catalog

Deploy **AWS ElastiCache** (Redis) and **Amazon MSK** (Kafka) via the AWS Service Catalog for managed caching and messaging services.

### 1.1 Deploying ElastiCache (Redis)
- **Service Catalog Product**: Use AWS Service Catalog to deploy an ElastiCache Redis cluster.
- **Configuration**:
  - **Engine**: Redis 7.0.
  - **Node Type**: `cache.t3.medium` (2 vCPU, 3.22 GB RAM).
  - **Replicas**: 2 replicas across 3 AZs for HA.
  - **Security**: Enable encryption in-transit and at-rest.
  - **Cost**: ~$0.068/hour per node, ~$150/month for 3 nodes.
- **Steps**:
  1. In AWS Console, navigate to Service Catalog > Portfolios.
  2. Select an ElastiCache Redis product (or create via CloudFormation).
  3. Launch the product, specifying:
     - Cluster name: `craft-store-redis-2x`.
     - VPC: Same as EKS (`craft-store-vpc`).
     - Subnets: Private subnets across 3 AZs.
     - Security Group: Allow port 6379 from EKS nodes.
  4. Retrieve the endpoint (e.g., `craft-store-redis-2x.abcd.0001.use1.cache.amazonaws.com:6379`).
  5. Store credentials in AWS Secrets Manager (`craft-store-redis-auth-2x`).

### 1.2 Deploying MSK (Kafka)
- **Service Catalog Product**: Use AWS Service Catalog to deploy an MSK cluster.
- **Configuration**:
  - **Broker Type**: `kafka.t3.small` (2 vCPU, 2 GB RAM).
  - **Brokers**: 3 brokers across 3 AZs.
  - **Storage**: 100 GB per broker.
  - **Security**: SASL/SCRAM authentication, TLS encryption.
  - **Cost**: ~$0.07/hour per broker, ~$151/month for 3 brokers.
- **Steps**:
  1. In Service Catalog, select an MSK product.
  2. Launch the product, specifying:
     - Cluster name: `craft-store-msk-2x`.
     - VPC: Same as EKS.
     - Subnets: Private subnets.
     - Security Group: Allow port 9094 (TLS) from EKS nodes.
  3. Retrieve bootstrap servers (e.g., `b-1.craft-store-msk-2x.abcd.kafka.use1.amazonaws.com:9094`).
  4. Store SASL/SCRAM credentials in Secrets Manager (`craft-store-msk-auth-2x`).

### 1.3 Modifying Manifests to Connect to ElastiCache and MSK
Update microservices manifests (e.g., `cartService`) to use ElastiCache and MSK endpoints, with secrets managed via External Secrets Operator (see Section 2).

#### cartService Deployment YAML
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cart-service
  namespace: prod
spec:
  replicas: 4
  selector:
    matchLabels:
      app: cart-service
  template:
    metadata:
      labels:
        app: cart-service
    spec:
      containers:
      - name: cart-service
        image: otel-demo-cart:latest
        env:
        - name: REDIS_HOST
          value: "craft-store-redis-2x.abcd.0001.use1.cache.amazonaws.com"
        - name: REDIS_PORT
          value: "6379"
        - name: REDIS_AUTH
          valueFrom:
            secretKeyRef:
              name: redis-auth
              key: password
        - name: KAFKA_BOOTSTRAP_SERVERS
          value: "b-1.craft-store-msk-2x.abcd.kafka.use1.amazonaws.com:9094"
        - name: KAFKA_SASL_USERNAME
          valueFrom:
            secretKeyRef:
              name: msk-auth
              key: username
        - name: KAFKA_SASL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: msk-auth
              key: password
```

---

## 2. Deploying AWS Secrets Manager and Secret Management System

Deploy AWS Secrets Manager and implement the **External Secrets Operator** to manage all secrets in the EKS cluster, including CNPG, ElastiCache, MSK, and application credentials.

### 2.1 Deploying AWS Secrets Manager
- **Purpose**: Centralize storage of sensitive data (e.g., database passwords, API keys) with encryption and access control.
- **Configuration**:
  - **Region**: us-east-1.
  - **Encryption**: AWS-managed KMS key (default) or custom KMS key for enhanced control.
  - **Access**: IAM roles for EKS workloads (IRSA) to read secrets.
  - **Secrets**:
    - `craft-store-redis-auth-2x`: Redis password.
    - `craft-store-msk-auth-2x`: MSK SASL/SCRAM username and password.
    - `craft-store-cnpg-app-auth-2x`: CNPG application user credentials.
    - `craft-store-cnpg-superuser-auth-2x`: CNPG superuser credentials.
    - `craft-store-cnpg-barman-auth-2x`: S3 backup credentials.
- **Steps**:
  1. In AWS Console, navigate to Secrets Manager.
  2. Create secrets for each component, e.g.:
     - **Secret Name**: `craft-store-cnpg-app-auth-2x`.
     - **Key/Value**: `username: app_user`, `password: password123`.
  3. Create an IAM policy for External Secrets Operator:
     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Action": [
             "secretsmanager:GetSecretValue",
             "secretsmanager:DescribeSecret"
           ],
           "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:craft-store-*"
         }
       ]
     }
     ```
  4. Attach the policy to an IRSA role (`craft-store-external-secrets-irsa-2x`) for the External Secrets Operator service account.

### 2.2 Deploying External Secrets Operator
- **Purpose**: Sync AWS Secrets Manager secrets to Kubernetes Secrets for use by CNPG, microservices, and Velero.
- **Helmfile Configuration**:
  Add External Secrets Operator to `helmfile.yaml`:
  ```yaml
  repositories:
    - name: external-secrets
      url: https://charts.external-secrets.io

  releases:
    - name: external-secrets
      namespace: external-secrets
      chart: external-secrets/external-secrets
      version: 0.10.4
      values:
        - serviceAccount:
            annotations:
              eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/craft-store-external-secrets-irsa-2x
  ```
- **SecretStore Configuration**:
  Create a cluster-wide `SecretStore` to access AWS Secrets Manager:
  ```yaml
  apiVersion: external-secrets.io/v1beta1
  kind: ClusterSecretStore
  metadata:
    name: aws-secrets-manager
  spec:
    provider:
      aws:
        service: SecretsManager
        region: us-east-1
        auth:
          jwt:
            serviceAccountRef:
              name: external-secrets
              namespace: external-secrets
  ```

### 2.3 Managing Secrets with External Secrets
Create `ExternalSecret` resources to sync AWS Secrets Manager secrets to Kubernetes Secrets.

#### ExternalSecret for CNPG
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: cnpg-app-auth
  namespace: prod-db
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-auth
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: craft-store-cnpg-app-auth-2x
      property: username
  - secretKey: password
    remoteRef:
      key: craft-store-cnpg-app-auth-2x
      property: password
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: cnpg-superuser-auth
  namespace: prod-db
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: superuser-auth
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: craft-store-cnpg-superuser-auth-2x
      property: username
  - secretKey: password
    remoteRef:
      key: craft-store-cnpg-superuser-auth-2x
      property: password
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: cnpg-barman-auth
  namespace: prod-db
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: barman-auth
    creationPolicy: Owner
  data:
  - secretKey: ACCESS_KEY_ID
    remoteRef:
      key: craft-store-cnpg-barman-auth-2x
      property: access_key_id
  - secretKey: ACCESS_SECRET_KEY
    remoteRef:
      key: craft-store-cnpg-barman-auth-2x
      property: secret_access_key
```

#### ExternalSecret for ElastiCache and MSK
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: redis-auth
  namespace: prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: redis-auth
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: craft-store-redis-auth-2x
      property: password
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: msk-auth
  namespace: prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: msk-auth
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: craft-store-msk-auth-2x
      property: username
  - secretKey: password
    remoteRef:
      key: craft-store-msk-auth-2x
      property: password
```

- **Steps**:
  1. Create secrets in AWS Secrets Manager for all components.
  2. Deploy External Secrets Operator via Helmfile.
  3. Apply `ClusterSecretStore` and `ExternalSecret` YAMLs.
  4. Verify Kubernetes Secrets: `kubectl -n prod-db get secret app-auth`.

---

## 3. Installing CNPG CRDs into Helmfile for GHA Pipeline

Integrate CNPG CRDs and operator into the Helmfile used by the GHA pipeline.

### Helmfile Configuration
Update `helmfile.yaml` to include CNPG and External Secrets Operator.

```yaml
repositories:
  - name: cnpg
    url: https://cloudnative-pg.github.io/charts
  - name: external-secrets
    url: https://charts.external-secrets.io

releases:
  - name: cnpg
    namespace: cnpg-system
    chart: cnpg/cloudnative-pg
    version: 0.22.0
    values:
      - monitoring:
          podMonitorEnabled: true
      - config:
          data:
            POSTGRESQL_SHARED_PRELOAD_LIBRARIES: "pgaudit,pg_stat_statements"
  - name: external-secrets
    namespace: external-secrets
    chart: external-secrets/external-secrets
    version: 0.10.4
    values:
      - serviceAccount:
          annotations:
            eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/craft-store-external-secrets-irsa-2x
```

### GHA Pipeline Update
Modify the GHA workflow (`.github/workflows/deploy.yaml`) to apply the Helmfile.

```yaml
name: Deploy Platform
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up Helm
      uses: azure/setup-helm@v3
      with:
        version: v3.12.0
    - name: Install Helmfile
      run: |
        curl -L https://github.com/helmfile/helmfile/releases/download/v0.167.0/helmfile_0.167.0_linux_amd64.tar.gz | tar xz
        sudo mv helmfile /usr/local/bin/
    - name: Deploy Helmfile
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        AWS_REGION: us-east-1
      run: |
        helmfile --file helmfile.yaml sync
```

- **Steps**:
  1. Add CNPG and External Secrets Operator to `helmfile.yaml`.
  2. Update GHA workflow to run `helmfile sync`.
  3. Commit changes to trigger deployment.
  4. Verify: `kubectl get pods -n cnpg-system` and `kubectl get pods -n external-secrets`.

---

## 4. Creating CNPG YAMLs to Replace StatefulSet Database

Replace StatefulSet-based PostgreSQL YAMLs with CNPG-managed PostgreSQL cluster YAMLs, using secrets from External Secrets Operator.

### CNPG Cluster YAML
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: craft-store-db
  namespace: prod-db
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgres:15.4
  primaryUpdateStrategy: unsupervised
  storage:
    storageClass: cnpg-ebs
    size: 200Gi
  walStorage:
    storageClass: cnpg-ebs
    size: 100Gi
  bootstrap:
    initdb:
      database: craft_store
      owner: app_user
      secret:
        name: app-auth
  superuserSecret:
    name: superuser-auth
  enableSuperuserAccess: true
  postgresql:
    parameters:
      max_connections: "300"
      work_mem: "8MB"
      shared_preload_libraries: "pgaudit,pg_stat_statements"
  affinity:
    topologyKey: topology.kubernetes.io/zone
  backup:
    barmanObjectStore:
      destinationPath: s3://craft-store-cnpg-backup-bucket-2x/
      endpointURL: https://s3.us-east-1.amazonaws.com
      s3Credentials:
        accessKeyId:
          name: barman-auth
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: barman-auth
          key: ACCESS_SECRET_KEY
      wal:
        compression: gzip
        maxParallel: 4
    retentionPolicy: "14d"
  monitoring:
    enablePodMonitor: true
```

- **Changes from StatefulSet**:
  - Replaces StatefulSet with CNPG `Cluster` CRD.
  - Configures 3 instances (1 primary, 2 replicas).
  - Adds automated failover, WAL archiving, and S3 backups.
  - Uses secrets managed by External Secrets Operator.
- **Steps**:
  1. Delete old StatefulSet YAMLs.
  2. Add CNPG YAML to `manifests/db/`.
  3. Apply via ArgoCD (see Section 7).

---

## 5. Migration Scripts for Database, Schemas, and Accounts

Migrate data, schemas, and accounts from StatefulSet-based PostgreSQL to CNPG with near-zero downtime using logical replication.

### Migration Script
```bash
#!/bin/bash
set -e

# Variables
OLD_DB_HOST="statefulset-pg.prod-db.svc.cluster.local"
OLD_DB_PORT="5432"
OLD_DB_NAME="craft_store"
OLD_DB_USER="postgres"
OLD_DB_PASS="statefulset_superuser123"
NEW_DB_HOST="craft-store-db-rw.prod-db.svc.cluster.local"
NEW_DB_PORT="5432"
NEW_DB_NAME="craft_store"
NEW_DB_USER="postgres"
NEW_DB_PASS="superuser123"
S3_BUCKET="craft-store-cnpg-backup-bucket-2x"
BACKUP_DIR="/tmp/pg_backup"

# Step 1: Backup existing database
mkdir -p $BACKUP_DIR
export PGPASSWORD=$OLD_DB_PASS
pg_dump -h $OLD_DB_HOST -p $OLD_DB_PORT -U $OLD_DB_USER -d $OLD_DB_NAME --schema-only > $BACKUP_DIR/schema.sql
pg_dump -h $OLD_DB_HOST -p $OLD_DB_PORT -U $OLD_DB_USER -d $OLD_DB_NAME --data-only > $BACKUP_DIR/data.sql

# Step 2: Create database and user in CNPG
export PGPASSWORD=$NEW_DB_PASS
psql -h $NEW_DB_HOST -p $NEW_DB_PORT -U $NEW_DB_USER -c "CREATE DATABASE $NEW_DB_NAME;"
psql -h $NEW_DB_HOST -p $NEW_DB_PORT -U $NEW_DB_USER -d $NEW_DB_NAME -c "CREATE USER app_user WITH PASSWORD 'password123';"
psql -h $NEW_DB_HOST -p $NEW_DB_PORT -U $NEW_DB_USER -d $NEW_DB_NAME -c "GRANT ALL PRIVILEGES ON DATABASE $NEW_DB_NAME TO app_user;"

# Step 3: Restore schema and data
psql -h $NEW_DB_HOST -p $NEW_DB_PORT -U $NEW_DB_USER -d $NEW_DB_NAME -f $BACKUP_DIR/schema.sql
psql -h $NEW_DB_HOST -p $NEW_DB_PORT -U $NEW_DB_USER -d $NEW_DB_NAME -f $BACKUP_DIR/data.sql

# Step 4: Set up logical replication
psql -h $OLD_DB_HOST -p $OLD_DB_PORT -U $OLD_DB_USER -d $OLD_DB_NAME -c "ALTER SYSTEM SET wal_level = logical;"
psql -h $OLD_DB_HOST -p $OLD_DB_PORT -U $OLD_DB_USER -c "SELECT pg_reload_conf();"
psql -h $OLD_DB_HOST -p $OLD_DB_PORT -U $OLD_DB_USER -d $OLD_DB_NAME -c "CREATE PUBLICATION craft_store_pub FOR ALL TABLES;"
psql -h $OLD_DB_HOST -p $OLD_DB_PORT -U $OLD_DB_USER -d $OLD_DB_NAME -c "ALTER TABLE <table_name> REPLICA IDENTITY FULL;" # Repeat for each table
psql -h $NEW_DB_HOST -p $NEW_DB_PORT -U $NEW_DB_USER -d $NEW_DB_NAME -c "CREATE SUBSCRIPTION craft_store_sub CONNECTION 'host=$OLD_DB_HOST port=$OLD_DB_PORT user=$OLD_DB_USER password=$OLD_DB_PASS dbname=$OLD_DB_NAME' PUBLICATION craft_store_pub;"

# Step 5: Validate data consistency
echo "Validate data manually or with application tests"

# Step 6: Update application endpoints
kubectl -n prod patch deployment cart-service -p '{"spec":{"template":{"spec":{"containers":[{"name":"cart-service","env":[{"name":"DB_HOST","value":"craft-store-db-rw.prod-db.svc.cluster.local"}]}]}}}}'
kubectl -n prod patch deployment order-service -p '{"spec":{"template":{"spec":{"containers":[{"name":"order-service","env":[{"name":"DB_HOST","value":"craft-store-db-rw.prod-db.svc.cluster.local"}]}]}}}}'
# Repeat for other services

# Step 7: Monitor replication
psql -h $NEW_DB_HOST -p $NEW_DB_PORT -U $NEW_DB_USER -d $NEW_DB_NAME -c "SELECT * FROM pg_stat_subscription;"

# Step 8: Switch and decommission
psql -h $NEW_DB_HOST -p $NEW_DB_PORT -U $NEW_DB_USER -d $NEW_DB_NAME -c "ALTER SUBSCRIPTION craft_store_sub DISABLE;"
psql -h $NEW_DB_HOST -p $NEW_DB_PORT -U $NEW_DB_USER -d $NEW_DB_NAME -c "DROP SUBSCRIPTION craft_store_sub;"
kubectl -n prod-db delete statefulset statefulset-pg
aws s3 cp $BACKUP_DIR/ s3://$S3_BUCKET/backups/ --recursive
rm -rf $BACKUP_DIR
```

### Endpoint Changes
- **Old Endpoint**: `statefulset-pg.prod-db.svc.cluster.local:5432`.
- **New Endpoint**: `craft-store-db-rw.prod-db.svc.cluster.local:5432` (read-write).
- **Read-Only Endpoint**: `craft-store-db-ro.prod-db.svc.cluster.local:5432` (for Analytics Service).
- **Steps**:
  1. Update `DB_HOST` in microservices via `kubectl patch` or manifests.
  2. Use CNPG’s read-write (`-rw`) and read-only (`-ro`) services.
  3. Test connectivity: `psql -h craft-store-db-rw.prod-db.svc.cluster.local -U app_user -d craft_store`.

---

## 6. Validating CNPG Functionality

Validate CNPG’s HA, DR, failover, and backup features.

### Validation Steps
1. **High Availability**:
   - Simulate failure: `kubectl -n prod-db delete pod craft-store-db-1`.
   - Verify replica promotion: `kubectl -n prod-db get pods` (<15s).
   - Test application connectivity with `cartService` tests.
2. **Automated Failover**:
   - Check Grafana metrics (`cnpg_cluster_failover_count`).
   - Query replication: `psql -h craft-store-db-rw -U postgres -c "SELECT * FROM pg_stat_replication;"`.
3. **Backup and PITR**:
   - Trigger backup: `kubectl -n prod-db annotate cluster/craft-store-db cnpg.io/immediateBackup=true`.
   - Verify in S3: `aws s3 ls s3://craft-store-cnpg-backup-bucket-2x/`.
   - Test PITR:
     ```yaml
     apiVersion: postgresql.cnpg.io/v1
     kind: Backup
     metadata:
       name: craft-store-pitr
       namespace: prod-db
     spec:
       cluster:
         name: craft-store-db
       method: recovery
       targetTime: "2025-04-22T12:00:00Z"
     ```
4. **Performance**:
   - Run load tests (5,000 transactions/day) using OpenTelemetry demo scripts.
   - Monitor latency (<40ms) in Grafana.
5. **Data Integrity**:
   - Compare row counts and tables.
   - Run end-to-end tests.

- **Expected Outcomes**:
  - Failover: <15s downtime.
  - Backup: Hourly WAL, 14-day PITR.
  - Performance: <40ms latency.
  - Uptime: 99.95%.

---

## 7. ArgoCD YAMLs and GHA Pipeline for CNPG Deployment

Deploy the application and CNPG via ArgoCD with a GHA pipeline.

### ArgoCD Application YAML
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: craft-store-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/craft-store.git
    targetRevision: main
    path: manifests/
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: craft-store-db
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/craft-store.git
    targetRevision: main
    path: manifests/db/
  destination:
    server: https://kubernetes.default.svc
    namespace: prod-db
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### Directory Structure
```
manifests/
├── cart-service.yaml
├── order-service.yaml
├── ...
└── db/
    ├── cnpg-cluster.yaml
    ├── external-secrets.yaml
```

### GHA Pipeline for ArgoCD
```yaml
name: Deploy with ArgoCD
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: v1.24.0
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1
    - name: Install ArgoCD CLI
      run: |
        curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/download/v2.12.0/argocd-linux-amd64
        sudo mv argocd /usr/local/bin/
    - name: Deploy to ArgoCD
      env:
        ARGOCD_SERVER: ${{ secrets.ARGOCD_SERVER }}
        ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_AUTH_TOKEN }}
      run: |
        argocd app sync craft-store-app --force
        argocd app sync craft-store-db --force
        argocd app wait craft-store-app craft-store-db --timeout 600
```

- **Steps**:
  1. Create ArgoCD Applications.
  2. Organize manifests in the repository.
  3. Update GHA to sync ArgoCD deployments.
  4. Store ArgoCD credentials in GitHub Secrets.

---

## 8. Adding Velero CRDs and YAMLs for Backups

Integrate Velero for Kubernetes backups, using secrets from External Secrets Operator.

### Helmfile Update for Velero
```yaml
repositories:
  - name: vmware-tanzu
    url: https://vmware-tanzu.github.io/helm-charts

releases:
  - name: velero
    namespace: velero
    chart: vmware-tanzu/velero
    version: 7.3.0
    values:
      - configuration:
          provider: aws
          backupStorageLocation:
            bucket: craft-store-cnpg-backup-bucket-2x
            config:
              region: us-east-1
          volumeSnapshotLocation:
            config:
              region: us-east-1
      - credentials:
          useSecret: true
          secretContents: {} # Managed by ExternalSecret
      - snapshotsEnabled: true
      - initContainers:
        - name: velero-plugin-for-aws
          image: velero/velero-plugin-for-aws:v1.10.0
          volumeMounts:
            - mountPath: /target
              name: plugins
```

### ExternalSecret for Velero
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: velero-auth
  namespace: velero
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: velero-auth
    creationPolicy: Owner
  data:
  - secretKey: cloud
    remoteRef:
      key: craft-store-velero-auth-2x
      property: cloud
```

### Velero Backup Schedule
```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: craft-store-db-backup
  namespace: velero
spec:
  schedule: "0 * * * *" # Hourly
  template:
    includedNamespaces:
    - prod-db
    includedResources:
    - cluster.postgresql.cnpg.io
    - persistentvolumeclaims
    - secrets
    storageLocation: default
    ttl: 336h # 14 days
```

### GHA Pipeline Update
```yaml
name: Deploy Platform with Velero
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up Helm
      uses: azure/setup-helm@v3
      with:
        version: v3.12.0
    - name: Install Helmfile
      run: |
        curl -L https://github.com/helmfile/helmfile/releases/download/v0.167.0/helmfile_0.167.0_linux_amd64.tar.gz | tar xz
        sudo mv helmfile /usr/local/bin/
    - name: Deploy Helmfile
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        AWS_REGION: us-east-1
      run: |
        helmfile --file helmfile.yaml sync
    - name: Verify Velero Backup
      run: |
        kubectl -n velero wait --for=condition=ready pod -l app.kubernetes.io/name=velero --timeout=300s
        kubectl -n velero exec -it $(kubectl -n velero get pod -l app.kubernetes.io/name=velero -o jsonpath='{.items[0].metadata.name}') -- velero backup describe craft-store-db-backup
```

- **Steps**:
  1. Add Velero to `helmfile.yaml`.
  2. Create `External 3. Apply Velero Schedule and ExternalSecret.
  4. Update GHA to deploy Velero and verify backups.
  5. Test restore: `velero restore create --from-backup craft-store-db-backup-<timestamp>`.

### Synergy with CNPG and ArgoCD
- **CNPG**: Velero backs up CNPG `Cluster` CRs, PVCs, and Secrets.
- **ArgoCD**: Ensures declarative deployments.
- **Backup Process**:
  - Velero captures Kubernetes resources hourly.
  - CNPG handles database WAL backups and PITR.
  - Secrets are managed by External Secrets Operator.

---

## Conclusion

This guide enhances the Artisanal Craft Store’s platform by:
- Deploying managed ElastiCache, MSK, and Secrets Manager.
- Implementing CNPG for PostgreSQL.
- Automating secret management with External Secrets Operator.
- Using ArgoCD and GHA for deployments.
- Ensuring DR with Velero and CNPG backups.

The solution supports a scaled workload (100 GB data, 5,000 peak transactions/day), achieves 99.95% uptime, and stays within budget (~$450/month with Spot Instances). For assistance, refer to CNPG documentation or contact the DevOps team.