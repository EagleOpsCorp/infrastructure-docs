# Secret Management for CloudNativePG in Kubernetes

This guide covers secure secret management strategies for CloudNativePG running in Kubernetes environments, focusing on two popular external secret management solutions:

1. AWS Secrets Manager
2. HashiCorp Vault

Both solutions enhance security by avoiding the storage of sensitive information directly in Kubernetes secrets, which are only base64-encoded and stored in etcd.

## Table of Contents

- [Why Use External Secret Management](#why-use-external-secret-management)
- [Solution 1: AWS Secrets Manager Integration](#solution-1-aws-secrets-manager-integration)
- [Solution 2: HashiCorp Vault Integration](#solution-2-hashicorp-vault-integration)
- [Comparison of Approaches](#comparison-of-approaches)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

## Why Use External Secret Management

Kubernetes secrets have several limitations:

- They are only base64-encoded, not encrypted by default
- They're stored in etcd, potentially accessible to anyone with API server access
- No built-in versioning or rotation capabilities
- Limited audit trail for secret access and changes
- Challenging to manage across multiple environments and clusters

External secret management solutions address these limitations by:

- Providing actual encryption for secret values
- Supporting secret versioning and rotation
- Offering detailed audit trails for compliance
- Implementing fine-grained access control
- Centralizing secrets across multiple applications/environments

## Solution 1: AWS Secrets Manager Integration

This solution leverages AWS Secrets Manager to store sensitive information with the AWS Secrets Store CSI Driver or External Secrets Operator for Kubernetes integration.

### Prerequisites

- EKS cluster with IAM OIDC provider enabled
- AWS CLI with appropriate permissions
- `kubectl` configured to access your EKS cluster
- `eksctl` (optional)

### Implementation Steps

#### 1. Create Secrets in AWS Secrets Manager

```bash
# Create secret for database credentials
aws secretsmanager create-secret \
    --name "prod/cnpg/postgres-credentials" \
    --description "PostgreSQL credentials for CNPG cluster" \
    --secret-string '{"username":"postgres","password":"YOUR_SECURE_PASSWORD"}'

# Create secret for backup S3 credentials (if needed)
aws secretsmanager create-secret \
    --name "prod/cnpg/s3-credentials" \
    --description "S3 credentials for CNPG backups" \
    --secret-string '{"accessKeyId":"YOUR_ACCESS_KEY","secretAccessKey":"YOUR_SECRET_KEY"}'
```

#### 2. Create IAM Policy for Secret Access

```bash
cat > secrets-manager-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": [
                "arn:aws:secretsmanager:${AWS_REGION}:${AWS_ACCOUNT_ID}:secret:prod/cnpg/*"
            ]
        }
    ]
}
EOF

aws iam create-policy \
    --policy-name CNPGSecretsManagerPolicy \
    --policy-document file://secrets-manager-policy.json
```

#### 3. Create Service Account with IAM Role

With eksctl:

```bash
eksctl create iamserviceaccount \
    --name cnpg-sa \
    --namespace cnpg-system \
    --cluster your-eks-cluster \
    --attach-policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/CNPGSecretsManagerPolicy \
    --approve \
    --override-existing-serviceaccounts
```

#### 4. Install the AWS Secrets Store CSI Driver

```bash
# Add the Helm repository
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo update

# Install the CSI driver
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
    --namespace kube-system \
    --set syncSecret.enabled=true

# Install the AWS provider
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
```

#### 5. Create a SecretProviderClass

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: postgres-aws-secrets
  namespace: cnpg-system
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/cnpg/postgres-credentials"
        objectType: "secretsmanager"
        jmesPath:
          - path: username
            objectAlias: db-username
          - path: password
            objectAlias: db-password
      - objectName: "prod/cnpg/s3-credentials"
        objectType: "secretsmanager"
        jmesPath:
          - path: accessKeyId
            objectAlias: s3-access-key
          - path: secretAccessKey
            objectAlias: s3-secret-key
  secretObjects:
    - secretName: postgres-credentials
      type: Opaque
      data:
        - objectName: db-username
          key: username
        - objectName: db-password
          key: password
    - secretName: s3-credentials
      type: Opaque
      data:
        - objectName: s3-access-key
          key: access-key
        - objectName: s3-secret-key
          key: secret-key
```

#### 6. Configure CloudNativePG to Use the Service Account

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-cluster
  namespace: cnpg-system
spec:
  instances: 3
  primaryUpdateStrategy: unsupervised
  serviceAccountName: cnpg-sa  # Use the service account with IAM role
  postgresql:
    parameters:
      shared_buffers: 256MB
      max_connections: 100
  storage:
    size: 20Gi
    storageClass: gp3
  superuserSecret:
    name: postgres-credentials
  backup:
    barmanObjectStore:
      destinationPath: s3://your-backup-bucket
      s3Credentials:
        accessKeyId:
          name: s3-credentials
          key: access-key
        secretAccessKey:
          name: s3-credentials
          key: secret-key
      endpointURL: https://s3.amazonaws.com
  walBackup:
    enableArchiveTimeout: true
```

#### 7. Alternative: External Secrets Operator (ESO)

For a more Kubernetes-native approach:

```bash
# Install External Secrets Operator using Helm
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace

# Create SecretStore that connects to AWS Secrets Manager
kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: cnpg-system
spec:
  provider:
    aws:
      service: SecretsManager
      region: ${AWS_REGION}
      auth:
        jwt:
          serviceAccountRef:
            name: cnpg-sa
EOF

# Create ExternalSecret resources to fetch secrets
kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: postgres-credentials
  namespace: cnpg-system
spec:
  refreshInterval: "15m"
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: postgres-credentials
  data:
  - secretKey: username
    remoteRef:
      key: prod/cnpg/postgres-credentials
      property: username
  - secretKey: password
    remoteRef:
      key: prod/cnpg/postgres-credentials
      property: password
EOF
```

## Solution 2: HashiCorp Vault Integration

This solution uses HashiCorp Vault for secret management with the Vault Secrets Operator for Kubernetes integration.

### Prerequisites

- Kubernetes cluster (any provider)
- `kubectl` configured to access your cluster
- Helm installed

### Implementation Steps

#### 1. Install HashiCorp Vault in Kubernetes

```bash
# Add the HashiCorp Helm repository
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Create a namespace for Vault
kubectl create namespace vault-system

# Install Vault
helm install vault hashicorp/vault \
  --namespace vault-system \
  --set server.dev.enabled=false \
  --set server.ha.enabled=true \
  --set server.ha.replicas=3
```

#### 2. Initialize and Unseal Vault

```bash
# Initialize Vault (only on the first pod)
kubectl exec -n vault-system vault-0 -- vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -format=json > vault-init.json

# Store the unseal keys and root token safely
VAULT_UNSEAL_KEY1=$(cat vault-init.json | jq -r ".unseal_keys_b64[0]")
VAULT_UNSEAL_KEY2=$(cat vault-init.json | jq -r ".unseal_keys_b64[1]")
VAULT_UNSEAL_KEY3=$(cat vault-init.json | jq -r ".unseal_keys_b64[2]")
VAULT_ROOT_TOKEN=$(cat vault-init.json | jq -r ".root_token")

# Unseal each Vault pod
for i in {0..2}; do
  kubectl exec -n vault-system vault-$i -- vault operator unseal $VAULT_UNSEAL_KEY1
  kubectl exec -n vault-system vault-$i -- vault operator unseal $VAULT_UNSEAL_KEY2
  kubectl exec -n vault-system vault-$i -- vault operator unseal $VAULT_UNSEAL_KEY3
done
```

#### 3. Configure Kubernetes Authentication

```bash
# Set environment variables
export VAULT_TOKEN=$VAULT_ROOT_TOKEN
kubectl exec -it vault-0 -n vault-system -- /bin/sh

# Inside the pod, configure Vault
vault auth enable kubernetes

# Get Kubernetes information for Vault authentication
KUBE_CA_CERT=$(cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt | base64)
KUBE_HOST="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT"

# Configure Kubernetes auth method
vault write auth/kubernetes/config \
  kubernetes_host="$KUBE_HOST" \
  kubernetes_ca_cert="$KUBE_CA_CERT" \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"

exit
```

#### 4. Create Secret Engines and Policies

```bash
kubectl exec -it vault-0 -n vault-system -- /bin/sh

# Enable the KV secrets engine
vault secrets enable -path=cnpg kv-v2

# Create PostgreSQL superuser credentials
vault kv put cnpg/postgres-credentials \
  username="postgres" \
  password="your-secure-password"

# Create S3 backup credentials
vault kv put cnpg/s3-credentials \
  access_key="your-access-key" \
  secret_key="your-secret-key"

# Create a policy for CNPG to access these secrets
vault policy write cnpg-policy - <<EOF
path "cnpg/data/postgres-credentials" {
  capabilities = ["read"]
}

path "cnpg/data/s3-credentials" {
  capabilities = ["read"]
}
EOF

# Create a Kubernetes auth role for CNPG
vault write auth/kubernetes/role/cnpg \
  bound_service_account_names=cnpg-sa \
  bound_service_account_namespaces=cnpg-system \
  policies=cnpg-policy \
  ttl=1h

exit
```

#### 5. Install the Vault Secrets Operator

```bash
# Install the Vault Secrets Operator
helm install vault-secrets-operator hashicorp/vault-secrets-operator \
  --namespace vault-system \
  --set defaultVaultConnection.enabled=true \
  --set defaultVaultConnection.address=http://vault.vault-system:8200
```

#### 6. Create a Service Account for CNPG

```bash
# Create the service account for CNPG
kubectl create -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cnpg-sa
  namespace: cnpg-system
EOF
```

#### 7. Create VaultAuth and VaultConnection Resources

```bash
kubectl apply -f - <<EOF
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: vault-auth
  namespace: cnpg-system
spec:
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: cnpg
    serviceAccount: cnpg-sa
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: vault-connection
  namespace: cnpg-system
spec:
  address: http://vault.vault-system:8200
EOF
```

#### 8. Create VaultStaticSecret Resources

```bash
kubectl apply -f - <<EOF
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: postgres-credentials
  namespace: cnpg-system
spec:
  vaultAuthRef: vault-auth
  vaultConnectionRef: vault-connection
  mount: cnpg
  type: kv-v2
  path: postgres-credentials
  refreshAfter: 30s
  destination:
    name: postgres-credentials
    create: true
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: s3-credentials
  namespace: cnpg-system
spec:
  vaultAuthRef: vault-auth
  vaultConnectionRef: vault-connection
  mount: cnpg
  type: kv-v2
  path: s3-credentials
  refreshAfter: 30s
  destination:
    name: s3-credentials
    create: true
EOF
```

#### 9. Configure CloudNativePG to Use These Secrets

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-cluster
  namespace: cnpg-system
spec:
  instances: 3
  primaryUpdateStrategy: unsupervised
  serviceAccountName: cnpg-sa
  postgresql:
    parameters:
      shared_buffers: 256MB
      max_connections: 100
  storage:
    size: 20Gi
    storageClass: gp3
  superuserSecret:
    name: postgres-credentials
  backup:
    barmanObjectStore:
      destinationPath: s3://your-backup-bucket
      s3Credentials:
        accessKeyId:
          name: s3-credentials
          key: access_key
        secretAccessKey:
          name: s3-credentials
          key: secret_key
      endpointURL: https://s3.amazonaws.com
  walBackup:
    enableArchiveTimeout: true
```

#### 10. Optional: Dynamic Database Credentials

```bash
kubectl exec -it vault-0 -n vault-system -- /bin/sh

# Enable the database secrets engine
vault secrets enable database

# Configure PostgreSQL connection
vault write database/config/pg-cluster \
  plugin_name=postgresql-database-plugin \
  allowed_roles="cnpg-app-role" \
  connection_url="postgresql://{{username}}:{{password}}@pg-cluster-rw.cnpg-system:5432/postgres?sslmode=require" \
  username="postgres" \
  password="$(kubectl get secret -n cnpg-system postgres-credentials -o jsonpath='{.data.password}' | base64 -d)"

# Create a role for application access
vault write database/roles/cnpg-app-role \
  db_name=pg-cluster \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
                       GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

exit
```

Create a VaultDynamicSecret:

```bash
kubectl apply -f - <<EOF
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultDynamicSecret
metadata:
  name: app-db-credentials
  namespace: cnpg-system
spec:
  vaultAuthRef: vault-auth
  vaultConnectionRef: vault-connection
  mount: database
  path: creds/cnpg-app-role
  destination:
    name: app-db-credentials
    create: true
  rolloutRestartTargets:
  - kind: Deployment
    name: your-application
EOF
```

## Comparison of Approaches

| Feature | AWS Secrets Manager | HashiCorp Vault |
|---------|---------------------|-----------------|
| **Management** | Managed service by AWS | Self-managed (can be deployed in K8s) |
| **Availability** | AWS-managed high availability | Requires HA configuration |
| **Integration** | Native with AWS services | Multi-cloud, platform-agnostic |
| **Dynamic Secrets** | Limited | Extensive support |
| **Secret Rotation** | Native rotation for some services | Comprehensive rotation capabilities |
| **Pricing** | Pay per secret and API call | Free open source (paid enterprise version) |
| **Access Control** | IAM policies | Fine-grained ACL policies |
| **Audit** | CloudTrail integration | Built-in audit logging |
| **Identity Sources** | AWS IAM, limited external IDPs | Multiple auth methods (LDAP, OIDC, etc.) |
| **Multi-Region** | Cross-region replication | Manual replication setup required |

## Best Practices

### General Best Practices

1. **Least privilege access**: Grant minimal permissions to access secrets
2. **Secret rotation**: Regularly rotate credentials
3. **Audit access**: Enable comprehensive audit logging
4. **Encryption**: Ensure secrets are encrypted at rest and in transit
5. **Backup**: Regularly back up your secret storage
6. **Monitoring**: Set up alerting for unauthorized access attempts
7. **Immutable infrastructure**: Use GitOps approaches for secret management

### AWS Secrets Manager Specific

1. **Use IAM Roles for Service Accounts**: Avoid using static credentials
2. **Implement cross-region replication**: For disaster recovery
3. **Set up CloudWatch alerts**: Monitor for unauthorized access
4. **Use resource tags**: For better organization and cost allocation
5. **Enable automatic rotation**: For long-lived secrets

### HashiCorp Vault Specific

1. **Use auto-unseal**: Configure with a cloud KMS provider
2. **Implement HA storage backend**: Use Consul or Raft storage
3. **Token TTLs**: Use appropriate time-to-live settings for tokens
4. **Key rotation**: Regularly rotate encryption keys
5. **Network segmentation**: Restrict network access to Vault
6. **Use response wrapping**: For sensitive operations
7. **Implement transit secrets engine**: For encryption as a service

## Troubleshooting

### AWS Secrets Manager Issues

1. **Access denied errors**:
   - Check IAM policies and OIDC provider configuration
   - Verify service account annotations and IAM role trust relationships
   - Ensure the EKS cluster has the IAM OIDC provider enabled

2. **SecretProviderClass not working**:
   - Check CSI driver and provider logs
   - Verify the SecretProviderClass parameters format
   - Ensure the service account has the correct IAM role

3. **External Secrets Operator issues**:
   - Check SecretStore and ExternalSecret resources for errors
   - Verify IRSA setup and permissions

### HashiCorp Vault Issues

1. **Authentication failures**:
   - Check Vault server logs
   - Verify Kubernetes auth configuration
   - Ensure the service account exists and has the correct name

2. **Vault Secrets Operator issues**:
   - Check VaultAuth and VaultConnection resources
   - Verify the VaultStaticSecret or VaultDynamicSecret configuration
   - Check operator logs for detailed error messages

3. **Unsealing problems**:
   - Ensure all Vault pods are properly unsealed
   - Configure auto-unseal for production environments

4. **Policy issues**:
   - Verify that the policy grants appropriate access to the secret paths
   - Check for typos in path names or missing capabilities
