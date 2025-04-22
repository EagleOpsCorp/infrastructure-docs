# Artisanal Craft Store: CNPG Migration Plan (2x Scale)

## 1. Introduction
This document outlines the migration of the Artisanal Craft Store’s PostgreSQL database from a Kubernetes StatefulSet to CloudNativePG (CNPG) on Amazon EKS, scaled for a $2M revenue business. The goal is to enhance reliability, HA, and DR while supporting doubled workload.

## 2. Current Infrastructure
- **EKS Cluster**: Kubernetes 1.24, 6 t3.large nodes (4 vCPU, 8 GB RAM), 3 AZs.
- **Database**: PostgreSQL 14, StatefulSet, primary + 1 replica, 400 GB EBS.
- **Microservices**: Product Catalog, Inventory, Order Management, Customer Profile, Payment, Recommendation, Analytics; 3–5 replicas each.
- **Issues**: Manual failover (10–15m downtime), inconsistent replication, manual backups, scaling limits.

## 3. Objectives
- **Reliability**: Automated failover, <15s downtime.
- **HA**: Primary + 2 replicas across AZs.
- **DR**: Hourly S3 backups, 14-day PITR, cross-region.
- **Management**: Operator-based automation.
- **Scalability**: Handle 5,000 peak transactions/day.
- **Cost**: < $500/month with optimization.

## 4. Stage 1: Planning and Environment Setup
### 4.1 Requirement Analysis
- **Workload**:
  - Data: 100 GB, 15% monthly growth.
  - Transactions: 1,000/day, 5,000 peak.
  - Latency: <50ms read/write.
  - Uptime: 99.95%.
- **Goals**:
  - Automated failover/replication.
  - Hourly backups, 14-day PITR.
  - Minimal management overhead.
- **Constraints**:
  - $500/month budget.
  - Small team (2 devs, 1 DevOps).
  - No downtime during peaks.

### 4.2 Infrastructure Setup
- **Cluster**:
  - Verify EKS 1.24 compatibility.
  - Scale to 8 t3.large nodes, 3 AZs.
- **Namespaces**:
  - `cnpg-system`: CNPG operator.
  - `prod-db`: Database cluster.
- **Storage**:
  - `gp3` EBS, 200 GB primary, 400 GB replicas, 100 GB WAL.
  - `cnpg-ebs` storage class, `Retain` policy.
- **IAM**:
  - IRSA for Barman, S3 access (`craft-store-cnpg-backup-bucket-2x`).
- **Monitoring**:
  - Prometheus/Grafana in `monitoring`.
  - CloudWatch for logs/alerts.
- **Security**:
  - RBAC for `prod-db`.
  - Secrets with AWS Secrets Manager.
  - Network policies.

## 5. Migration Strategy
- **Method**: Near-zero downtime with logical replication.
- **Steps**:
  1. Deploy CNPG operator.
  2. Bootstrap PostgreSQL 15 cluster.
  3. Set up replication from StatefulSet to CNPG.
  4. Validate data consistency.
  5. Switch connections to CNPG.
  6. Decommission StatefulSet.

## 6. Expected Outcomes
- **Database**: PostgreSQL 15, primary + 2 replicas, <15s failover.
- **Performance**: <40ms latency.
- **Cost**: ~$638/month (EC2: $576, EBS: $40, S3: $6.60, CloudWatch: $15), ~$450 with Spot Instances.
- **Processes**:
  - Automated backups/PITR.
  - Cluster Autoscaler/HPA.
  - Grafana dashboards.
- **Business Impact**:
  - Enhanced SEO/customer experience.
  - Supports market expansion.
  - Reduced operational overhead.

## 7. Next Steps
- Deploy CNPG operator.
- Execute migration.
- Monitor performance.
- Optimize costs with Spot Instances.