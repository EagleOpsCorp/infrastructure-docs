# CloudNativePG Migration Project

This repository contains the necessary scripts and configurations to implement a migration from a traditional PostgreSQL StatefulSet deployment to a CloudNativePG (CNPG) operator-based deployment in Kubernetes.

## Project Overview

The migration is divided into stages:

1. **Stage 1: Planning and Environment Setup**
   - Setup Kubernetes sandbox environment
   - Configure namespaces and storage classes
   - Prepare environment for CNPG deployment

2. **Stage 2: CNPG Deployment in Test Environment**
   - Install CNPG operator
   - Deploy PostgreSQL cluster with CNPG
   - Migrate data from existing StatefulSet database
   - Test and validate the deployment

## Prerequisites

- `kubectl` CLI
- `kind` or `minikube` for local Kubernetes cluster
- `helm` for deploying the CNPG operator
- Docker
- Basic understanding of Kubernetes and PostgreSQL

## Stage 1: Planning and Environment Setup

### Local Kubernetes Cluster Setup

The `setup-kubernetes-cluster.sh` script sets up a local Kubernetes cluster using kind, creates a dedicated namespace, and configures a storage class for persistent volumes.

```bash
#!/bin/bash
set -e

echo "Setting up Kubernetes cluster using kind..."
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF

kind create cluster --name cnpg-sandbox --config kind-config.yaml
kubectl cluster-info

echo "Creating cnpg-system namespace..."
kubectl create namespace cnpg-system

echo "Creating storage class for persistent volumes..."
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cnpg-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF

echo "Kubernetes cluster setup complete!"
```

## Stage 2: CNPG Deployment in Test Environment

### 1. CNPG Operator Installation and PostgreSQL Deployment

The `deploy-cnpg.sh` script installs the CNPG operator using Helm, deploys a MinIO instance for backups, and sets up a CNPG PostgreSQL cluster.

```bash
#!/bin/bash
set -e

echo "Installing CNPG operator with Helm..."
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update
helm install cnpg-controller cnpg/cloudnative-pg --namespace cnpg-system

echo "Deploying MinIO for S3-compatible backup storage..."
kubectl create namespace minio-system
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: minio-system
spec:
  selector:
    matchLabels:
      app: minio
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio
        args:
        - server
        - /data
        env:
        - name: MINIO_ACCESS_KEY
          value: "minioadmin"
        - name: MINIO_SECRET_KEY
          value: "minioadmin"
        ports:
        - containerPort: 9000
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: minio-system
spec:
  type: ClusterIP
  ports:
  - port: 9000
    targetPort: 9000
  selector:
    app: minio
EOF

echo "Creating S3 bucket for backups..."
kubectl wait --namespace minio-system --for=condition=available deployment/minio --timeout=120s
kubectl run -n minio-system minio-client --rm --tty -i --restart='Never' \
  --image="minio/mc" \
  --command -- bash -c "mc config host add myminio http://minio:9000 minioadmin minioadmin && mc mb myminio/cnpg-backups"

echo "Deploying CNPG PostgreSQL cluster..."
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-cluster
  namespace: cnpg-system
spec:
  instances: 3
  primaryUpdateStrategy: unsupervised
  postgresql:
    parameters:
      shared_buffers: 256MB
      max_connections: 100
      log_statement: all
  storage:
    size: 1Gi
    storageClass: cnpg-storage
  backup:
    barmanObjectStore:
      destinationPath: s3://cnpg-backups
      s3Credentials:
        accessKeyId:
          name: minio-creds
          key: access-key
        secretAccessKey:
          name: minio-creds
          key: secret-key
      endpointURL: http://minio.minio-system.svc.cluster.local:9000
      s3ForcePathStyle: true
    retentionPolicy: 7d
  walBackup:
    enableArchiveTimeout: true
  enableSuperuserAccess: true
EOF

# Create MinIO credentials secret
kubectl create secret generic minio-creds -n cnpg-system \
  --from-literal=access-key=minioadmin \
  --from-literal=secret-key=minioadmin

echo "CNPG PostgreSQL cluster deployment complete!"
```

### 2. Mock StatefulSet Database

The `mock-statefulset-db.yaml` file defines a simple PostgreSQL StatefulSet to simulate the existing database.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-statefulset
  namespace: default
spec:
  serviceName: postgres-statefulset
  replicas: 1
  selector:
    matchLabels:
      app: postgres-statefulset
  template:
    metadata:
      labels:
        app: postgres-statefulset
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          value: postgres
        - name: POSTGRES_DB
          value: testdb
        ports:
        - containerPort: 5432
          name: postgres
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-statefulset
  namespace: default
spec:
  selector:
    app: postgres-statefulset
  ports:
  - port: 5432
    targetPort: 5432
  clusterIP: None
```

### 3. Data Migration and Testing

The `migrate-and-test.sh` script migrates data from the mock StatefulSet database to the CNPG cluster and performs validation tests.

```bash
#!/bin/bash
set -e

echo "Creating test data in mock StatefulSet database..."
kubectl exec -it postgres-statefulset-0 -- bash -c "psql -U postgres -d testdb -c 'CREATE TABLE test_table (id SERIAL PRIMARY KEY, name VARCHAR(100), value INTEGER);' && for i in {1..100}; do psql -U postgres -d testdb -c \"INSERT INTO test_table (name, value) VALUES ('test_\$i', \$i);\"; done"

echo "Fetching PostgreSQL credentials for CNPG cluster..."
PGPASSWORD=$(kubectl get secret pg-cluster-superuser -n cnpg-system -o jsonpath='{.data.password}' | base64 -d)

echo "Migrating data to CNPG cluster..."
kubectl exec -it postgres-statefulset-0 -- bash -c "pg_dump -U postgres -d testdb" > /tmp/dump.sql
kubectl exec -it pg-cluster-1 -n cnpg-system -- bash -c "psql -U postgres -c 'CREATE DATABASE testdb;'"
cat /tmp/dump.sql | kubectl exec -i pg-cluster-1 -n cnpg-system -- bash -c "psql -U postgres -d testdb"

echo "Validating data integrity..."
ORIGINAL_COUNT=$(kubectl exec -it postgres-statefulset-0 -- bash -c "psql -U postgres -d testdb -t -c 'SELECT COUNT(*) FROM test_table;'" | tr -d '[:space:]')
MIGRATED_COUNT=$(kubectl exec -it pg-cluster-1 -n cnpg-system -- bash -c "psql -U postgres -d testdb -t -c 'SELECT COUNT(*) FROM test_table;'" | tr -d '[:space:]')

if [ "$ORIGINAL_COUNT" = "$MIGRATED_COUNT" ]; then
  echo "Data migration validated successfully! Count: $MIGRATED_COUNT"
else
  echo "Data validation failed! Original: $ORIGINAL_COUNT, Migrated: $MIGRATED_COUNT"
  exit 1
fi

echo "Testing application connectivity to CNPG cluster..."
cat <<EOF > test-app.js
const { Client } = require('pg');

async function main() {
  const client = new Client({
    host: 'pg-cluster-rw.cnpg-system.svc.cluster.local',
    port: 5432,
    database: 'testdb',
    user: 'postgres',
    password: process.env.PGPASSWORD,
  });
  
  await client.connect();
  const res = await client.query('SELECT COUNT(*) FROM test_table');
  console.log('Query result:', res.rows[0]);
  await client.end();
}

main().catch(e => {
  console.error('Error:', e);
  process.exit(1);
});
EOF

kubectl run -n cnpg-system test-app --image=node:16 --rm -i --env="PGPASSWORD=$PGPASSWORD" \
  --restart=Never -- bash -c "cat > app.js << 'EOL'
$(cat test-app.js)
EOL
npm init -y && npm install pg && node app.js"

echo "Testing CNPG failover..."
kubectl exec -n cnpg-system pg-cluster-1 -- bash -c 'pg_ctl stop -m immediate'
sleep 10
PRIMARY_POD=$(kubectl get pods -n cnpg-system -l cnpg.io/cluster=pg-cluster,cnpg.io/instanceRole=primary -o jsonpath='{.items[0].metadata.name}')
echo "New primary after failover: $PRIMARY_POD"

echo "Testing backup and restoration..."
# Create a backup
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: pg-cluster-backup
  namespace: cnpg-system
spec:
  cluster:
    name: pg-cluster
EOF

# Wait for backup to complete
kubectl wait --for=condition=completed backup/pg-cluster-backup -n cnpg-system --timeout=300s

echo "All tests completed successfully!"
```

## Execution Instructions

1. **Run Stage 1**:
   ```bash
   chmod +x setup-kubernetes-cluster.sh
   ./setup-kubernetes-cluster.sh
   ```

2. **Run Stage 2**:
   ```bash
   chmod +x deploy-cnpg.sh
   ./deploy-cnpg.sh
   
   kubectl apply -f mock-statefulset-db.yaml
   
   chmod +x migrate-and-test.sh
   ./migrate-and-test.sh
   ```

## Notes

- The scripts assume a local environment with kind, kubectl, and helm installed.
- For a cloud-based cluster (e.g., EKS), modify the storage class and MinIO setup to use actual cloud storage (e.g., AWS S3).
- The test application is a simple Node.js script that queries the database. Replace it with your actual application for real-world testing.
- The backup configuration uses MinIO as a mock S3 storage. Update the S3_ENDPOINT, S3_BUCKET, S3_ACCESS_KEY, and S3_SECRET_KEY for production use.
- The scripts include basic error handling and validation. Enhance them with additional checks as needed for robustness.

## Advanced Configuration Options

For more advanced configurations and production-grade deployments, consider:

- Implementing network policies for security
- Setting up monitoring with Prometheus and Grafana
- Configuring resource limits and requests
- Implementing SSL/TLS for database connections
- Configuring more sophisticated backup strategies
- Setting up maintenance windows for updates

## Troubleshooting

If you encounter issues:

1. Check the CNPG operator logs:
   ```bash
   kubectl logs -n cnpg-system -l app.kubernetes.io/name=cloudnative-pg
   ```

2. Check the PostgreSQL cluster logs:
   ```bash
   kubectl logs -n cnpg-system -l cnpg.io/cluster=pg-cluster
   ```

3. Verify PVC status:
   ```bash
   kubectl get pvc -n cnpg-system
   ```

4. Check backup status:
   ```bash
   kubectl get backup -n cnpg-system
   ```

## References

- [CloudNativePG Documentation](https://cloudnative-pg.io/documentation/)
- [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
