# Scaling Artisanal Craft Store's Database Migration to CloudNativePG

This case study examines how the migration of the Artisanal Craft Store's database from a Kubernetes StatefulSet-based PostgreSQL deployment to a CloudNativePG (CNPG)-managed PostgreSQL solution on Amazon Elastic Kubernetes Service (EKS) would differ if the company were twice as large, with annual revenue of approximately $2 million (up from under $1 million). The Artisanal Craft Store specializes in handmade crafts like pottery, jewelry, and custom art pieces, relying on a custom e-commerce microservice architecture to meet unique design, SEO, and personalization needs. The migration aims to enhance reliability, high availability (HA), and disaster recovery (DR) capabilities, addressing operational challenges from their previous setup.

## Company Overview (Scaled to 2x Size)

**Business Description:** The Artisanal Craft Store sells unique, handmade crafts, focusing on pottery, jewelry, and custom art pieces. With a growing customer base, the store emphasizes organic traffic, personalized shopping experiences, and scalability to support expansion.

**Revenue:** ~$2 million annually (previously under $1 million).

**E-commerce Needs (Scaled):**
- A bespoke online presence reflecting artisanal products, requiring advanced customization beyond platforms like Shopify
- Advanced SEO control for niche, long-tail keywords (e.g., "handmade ceramic vases for plant lovers"), critical for doubled organic traffic
- Enhanced personalized shopping experiences, such as AI-driven product recommendations and complex checkout flows, to handle increased customer volume
- Scalability to support doubled transaction volume and anticipated growth into new markets without platform migration

**Reason for Custom Solution:** The store chose a custom e-commerce microservice architecture to overcome Shopify's limitations in design flexibility, SEO control, and bespoke features. With 2x growth, the need for a robust, scalable infrastructure is even more critical to handle increased traffic, data, and operational complexity.

**Team Size:** Expanded to 2 full-time developers and 1 full-time DevOps engineer (previously 1 full-time developer, 1 part-time DevOps engineer), reflecting increased resources.

## Current Infrastructure (Scaled to 2x Size)

With doubled revenue, the store's infrastructure has grown to support higher traffic, data volume, and operational demands. The platform remains a custom-built, microservices-based system on Amazon EKS, but with enhanced resources.

### Infrastructure Components

#### Kubernetes Cluster
- **Platform:** Amazon EKS, running Kubernetes version 1.24
- **Nodes:** 6 EC2 instances (t3.large, 4 vCPU, 8 GB RAM each) across three Availability Zones (us-east-1a, us-east-1b, us-east-1c) for improved HA, up from 3 t3.medium nodes
- **Storage:** Amazon EBS with gp3 storage class, increased to 2 TB total capacity (from 1 TB) to handle doubled data growth
- **Networking:** AWS VPC with public and private subnets, using AWS CNI plugin, with network policies partially implemented for microservices

#### Database
- **Type:** PostgreSQL 14, deployed via Kubernetes StatefulSet
- **Configuration:** Single primary instance with one manual replica (added due to growth), using two EBS volumes (200 GB primary, 200 GB replica, 4000 IOPS each)
- **Issues:**
  - Manual failover, causing ~10–15 minutes downtime during failures
  - Inconsistent replication, risking data divergence during high traffic
  - Manual backups to S3 (twice weekly), with no point-in-time recovery (PITR)
  - Scaling limitations, as StatefulSet struggles with increased transaction volume

#### Microservices
- **Services:** Product Catalog, Inventory, Order Management, Customer Profile, Payment Processing, Recommendation Engine, plus a new Analytics Service for customer behavior tracking
- **Deployment:** Each service runs as a Kubernetes Deployment with 3–5 replicas (up from 2–3), using Docker containers
- **Communication:** gRPC for internal communication, HTTP/REST for external APIs, with increased API calls due to doubled traffic

#### Load Balancing
- **External:** AWS Application Load Balancer (ALB) with auto-scaling enabled for public traffic
- **Internal:** Kubernetes Service with ClusterIP, now handling doubled inter-service requests

#### Monitoring and Logging
- **Tools:** AWS CloudWatch with Prometheus and Grafana partially deployed, but not fully utilized for database monitoring
- **Issues:** Insufficient alerts for database replication issues, manual log analysis slowing incident response

#### CI/CD Pipeline
- **Tools:** GitHub Actions for CI/CD, Docker images in Amazon ECR
- **Process:** Automated builds, tests, and deployments to EKS, but database schema migrations remain manual, causing delays during updates
- **Issues:** Lack of automated rollback, increasing risk during deployments

### DevOps Processes

#### Deployment
- Code commits trigger GitHub Actions workflows for builds and deployments
- Manual rollouts with kubectl, with basic canary deployments for critical services

#### Scaling
- Manual pod and node scaling during peak events (e.g., holiday sales), with basic Horizontal Pod Autoscaler (HPA) for some services
- No cluster auto-scaling, leading to overprovisioning during low traffic

#### Backup and Recovery
- Manual EBS snapshots to S3 (twice weekly), with no automated restore testing
- Limited DR, as backups lack PITR and are not replicated cross-region

#### Security
- IAM roles for EKS nodes, with RBAC for Kubernetes access
- Partial network policies, leaving some services exposed
- Database credentials in Kubernetes Secrets, with infrequent manual rotation

#### Challenges
- Increased operational complexity with doubled infrastructure
- Manual database management strains the expanded team
- Limited HA and DR, risking revenue loss during outages
- Inadequate monitoring for doubled transaction volume

## Migration Objectives (Scaled)

The store's growth amplifies the need for a robust database solution. Previously migrating from Amazon RDS to a StatefulSet-based PostgreSQL to align with microservices, they now face intensified issues with manual failover, inconsistent replication, and scaling. The migration to CNPG aims to:

1. **Improve Reliability:** Automate failover and replication for near-zero downtime
2. **Enhance High Availability:** Deploy multiple replicas across AZs for robust HA
3. **Strengthen Disaster Recovery:** Implement automated backups and PITR with cross-region replication
4. **Simplify Management:** Use CNPG's operator to reduce administrative overhead
5. **Support Scalability:** Handle doubled transaction volume and future growth
6. **Optimize Costs:** Balance performance with budget constraints for a $2M business

## Solution: Stage 1 - Planning and Environment Setup (Scaled)

The first phase involves planning the migration and preparing the EKS environment for CNPG, tailored to the store's doubled scale.

### Requirement Analysis

#### Collaboration
The engineering team, now with 2 developers and 1 full-time DevOps engineer, collaborated with stakeholders (store owner, marketing team, customer support) via workshops.

**Stakeholders emphasized:**
- **Owner:** Zero downtime during peak sales (e.g., Black Friday)
- **Marketing:** Fast database queries for SEO-driven page loads
- **Support:** Robust DR to prevent data loss affecting customer trust

#### Workload Assessment
- **Data Size:** ~100 GB (doubled from 50 GB), with 15% monthly growth due to expanded product catalog and customer data
- **Transaction Volume:** ~1,000 transactions/day, peaking at 5,000 during sales events (up from 500/2,000)
- **Availability Requirements:** 99.95% uptime, with <2 minutes downtime during failures
- **Performance Needs:** <50ms latency for read/write operations to support doubled traffic and AI recommendations

#### Key Goals
- **Automated Failover:** Zero manual intervention, <15s switchover
- **Replication:** Two standby replicas for HA and read scalability
- **Backup/Restore:** Hourly WAL backups to S3, 14-day PITR retention
- **Simplified Management:** Minimize administrative tasks for the small team
- **Cost Optimization:** Keep monthly costs under $500 while meeting performance needs

#### Constraints
- Budget capped at ~$500/month for infrastructure, reflecting $2M revenue
- Small team requires high automation to manage increased complexity
- No downtime during peak periods, necessitating near-zero downtime migration

### Infrastructure Setup

#### Cluster Verification
- Confirmed EKS cluster (version 1.24) compatibility with CNPG, per CloudNativePG documentation
- Verified compute resources: 6 t3.large nodes (~24 vCPU, 48 GB RAM total) support CNPG with two replicas, with plans to add 2 nodes (t3.large) if needed
- Validated storage: 2 TB EBS gp3 volumes with 5000 IOPS, sufficient for database and backups

#### Namespace Configuration
- Created `cnpg-system` for CNPG operator and `prod-db` for database clusters
- Applied RBAC policies to restrict `prod-db` access to DevOps engineer and CI/CD pipeline

#### Storage Setup
- Used gp3 storage class for PVCs, with cnpg-ebs storage class (Retain policy) for data persistence
- Allocated 200 GB for primary, 200 GB per replica (2 replicas), and 100 GB for WAL archiving

#### IAM Roles for Service Accounts (IRSA)
Created IAM role for CNPG's Barman to access S3 bucket (craft-store-cnpg-backup-bucket-2x) for backups and WAL, with cross-region replication to us-west-2.

**Policies:** `s3:PutObject`, `s3:GetObject`, `s3:ListBucket`, `s3:ReplicateObject`

**Terraform output (hypothetical):**
```bash
terraform output
barman_backup_irsa = "arn:aws:iam::123456789012:role/cnpg-on-eks-prod-irsa-2x"
barman_s3_bucket = "craft-store-cnpg-backup-bucket-2x"
```

#### Monitoring and Logging Preparation
- Planned full Prometheus/Grafana deployment in monitoring namespace, with CNPG metrics and custom dashboards for replication and backup status
- Configured CloudWatch for EKS logs and CNPG logs, with alarms for failover, storage, and latency

#### Security
- Created Kubernetes Secrets (app-auth-2x) for database credentials, with AWS Secrets Manager integration planned
- Defined network policies to restrict database access to Product Catalog, Order Management, and Recommendation services
- Planned Pod Security Standards for CNPG pods

### Objective
- **Migration Strategy:** Near-zero downtime using PostgreSQL logical replication to migrate from StatefulSet to CNPG, inspired by CloudNativePG migration guide
- **Environment Readiness:** Prepared EKS cluster with expanded resources, namespaces, storage, IAM roles, and monitoring to support CNPG for doubled workload

## Post-Migration Infrastructure (Expected Outcomes, Scaled)

The scaled migration enhances the store's infrastructure to handle doubled scale, improving reliability, HA, and DR while optimizing costs.

### Infrastructure Components

#### Kubernetes Cluster
- **Platform:** Amazon EKS (version 1.24, planned upgrade to 1.26)
- **Nodes:** 8 t3.large instances (4 vCPU, 8 GB RAM each) across three AZs, costing $0.10/hour per node ($576/month)
- **Storage:** EBS gp3 with 2 TB capacity, using CNPG-managed PVCs (200 GB primary, 400 GB replicas, 100 GB WAL)
- **Networking:** Network policies restrict database and service access, maintaining AWS CNI and ALB

#### Database
- **Type:** PostgreSQL 15, managed by CNPG
- **Configuration:**
  - Primary instance with 2 standby replicas in prod-db namespace, using streaming replication
  - 200 GB EBS per instance, 100 GB WAL, 5000 IOPS
  - Automated failover with <15s switchover
  - Hourly WAL backups to S3, 14-day PITR, cross-region replication to us-west-2
- **Performance:** <40ms read/write latency, supporting SEO and recommendation services
- **Cost:** $0.08/GB/month for 500 GB EBS ($40/month), $0.023/GB/month for 200 GB S3 backups ($4.60/month), $0.01/GB/month for replication ($2/month)

#### Microservices
- Connect to CNPG via prod-rw (read-write) and prod-ro (read-only) Services
- Analytics Service leverages read replicas for faster customer insights
- Replicas increased to 4–6 per service to handle doubled traffic

#### Load Balancing
ALB with auto-scaling, internal Services optimized for doubled API calls

#### Monitoring and Logging
- Prometheus/Grafana with CNPG dashboards for database health, replication, and backups
- CloudWatch with alarms for failover, storage, and latency, costing ~$15/month

#### CI/CD Pipeline
- Automated schema migrations via CNPG's initdb and logical replication
- GitHub Actions with canary deployments and automated rollbacks for all services

### DevOps Processes

#### Deployment
- CNPG operator via Helm in cnpg-system, managing database clusters declaratively
- Application and database deployments fully automated via GitHub Actions
- Canary rollouts with automated rollback based on Prometheus metrics

#### Scaling
- CNPG manages up to 3 replicas with kubectl apply
- Cluster Autoscaler adjusts nodes (50–80% CPU utilization)
- HPA for microservices, targeting 70% CPU/memory
- Spot Instances for non-critical services, saving ~40% on EC2 costs

#### Backup and Recovery
- Hourly WAL backups to S3, 14-day PITR, cross-region replication
- Monthly restore tests via CNPG's Barman, ensuring DR readiness

#### Security
- Network policies for all services, minimizing attack surface
- Automated credential rotation with AWS Secrets Manager
- Pod Security Standards and IRSA for least privilege

#### Monitoring and Alerts
- Grafana dashboards for database and application metrics
- CloudWatch Alarms with Slack notifications, reducing response time to <5 minutes

### Outcomes

**Reliability:** Automated failover ensures <15s downtime, achieving 99.95% uptime

**High Availability:** Three-node cluster (primary + 2 replicas) across AZs prevents outages

**Disaster Recovery:** Hourly backups and cross-region PITR enable recovery in <10 minutes

**Simplified Management:** CNPG reduces database administration by ~60%, freeing the team for feature development

**Scalability:** Supports 5,000 peak transactions/day, with capacity for 2x further growth

**Cost Efficiency:** Total $638/month (EC2: $576, EBS: $40, S3: $6.60, CloudWatch: $15), under $500 with Spot Instances ($450)

**Business Impact:**
- <40ms database queries enhance SEO rankings and page loads
- Scalable recommendations drive conversions for doubled customer base
- Reduced overhead supports new market expansion (e.g., international sales)

## Key Differences from Original ($1M) Case

### Infrastructure
- **Nodes:** 8 t3.large vs. 3 t3.large, supporting doubled workload
- **Database:** 2 replicas vs. 1, 500 GB EBS vs. 250 GB, hourly backups vs. daily
- **Storage:** 2 TB vs. 1 TB, 5000 IOPS vs. 3000

### Processes
- **Auto-scaling:** Cluster Autoscaler and HPA fully enabled vs. manual scaling
- **Backups:** Hourly WAL with cross-region vs. daily without replication
- **Monitoring:** Full Prometheus/Grafana vs. partial, with advanced alerts

### Costs
~$638/month vs. ~$250, reducible to ~$450 with Spot Instances

### Team
2 developers + 1 DevOps vs. 1 developer + part-time DevOps, enabling more automation

### Performance
<40ms latency vs. <50ms, supporting doubled traffic and SEO needs

## Conclusion

For a 2x larger Artisanal Craft Store ($2M revenue), the CNPG migration scales infrastructure and processes to handle doubled data, traffic, and complexity. Enhanced HA, DR, and automation ensure reliability and scalability, while cost optimization keeps expenses within budget. This case study provides a blueprint for small e-commerce businesses scaling custom microservice platforms, demonstrating how to adapt Kubernetes and CNPG for growth.
