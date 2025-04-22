# Implementing mTLS Service Mesh for Niche Tech Gadget Seller's Custom E-commerce Platform on Amazon EKS

This case study outlines the implementation of a mutual TLS (mTLS) service mesh for the Niche Tech Gadget Seller, a small business with annual revenue under $2 million, specializing in custom mechanical keyboards, retro gaming consoles, and niche electronics. The company operates a custom e-commerce microservice architecture on Amazon Elastic Kubernetes Service (EKS) to meet unique inventory management, product configuration, and scalability needs. The project aimed to secure service-to-service communication using a service mesh (Linkerd), certificate management (cert-manager), and secret management (Vault), ensuring compliance with security audits and supporting the company's growth.

## Company Overview

**Business Description**: The Niche Tech Gadget Seller offers specialized tech products, focusing on customizable items like mechanical keyboards and retro gaming consoles. With a dedicated niche market, the store emphasizes seamless inventory integration, highly configurable products, and scalability for rapid growth.

**Revenue**: ~$2 million annually.

**E-commerce Needs**:
- Seamless integration with a specialized inventory management system for real-time tracking of low-stock, unique items
- Advanced product configuration options (e.g., custom keyboard switches, keycaps, console modifications) not easily supported by platforms like Shopify
- Scalable microservice architecture to handle increased traffic and sales without vendor lock-in
- Long-term cost efficiency by avoiding recurring subscription and plugin fees

**Reason for Custom Solution**: The store opted for a custom e-commerce microservice architecture to overcome Shopify's limitations in inventory integration, product customization, and scalability. The microservices approach ensures flexibility and supports anticipated growth in a competitive niche market.

**Team Size**: 2 full-time developers and 1 full-time DevOps engineer, reflecting resources suitable for a $2M business.

## Current Infrastructure

The Niche Tech Gadget Seller's e-commerce platform is a custom-built, microservices-based system running on a self-managed Kubernetes cluster on Amazon EKS. The infrastructure supports their online store, handling product catalog, inventory, orders, customer data, payments, and custom configurations.

### Infrastructure Components

#### Kubernetes Cluster
- **Platform**: Amazon EKS, running Kubernetes version 1.24
- **Nodes**: 6 EC2 instances (t3.large, 4 vCPU, 8 GB RAM each) across three Availability Zones (us-east-1a, us-east-1b, us-east-1c) for high availability
- **Storage**: Amazon EBS with gp3 storage class, providing 2 TB total capacity for application and database persistent volumes
- **Networking**: AWS VPC with public and private subnets, using AWS CNI plugin for pod networking. Limited network policies, exposing some services to internal risks

#### Database
- **Type**: PostgreSQL 14, deployed via Kubernetes StatefulSet
- **Configuration**: Primary instance with one manual replica, using two EBS volumes (200 GB primary, 200 GB replica, 4000 IOPS each)
- **Issues**: Manual failover and replication, with basic backups to S3, not critical for this project but noted for future upgrades

#### Microservices
- **Services**: Product Catalog, Inventory Management, Order Processing, Customer Profile, Payment Gateway, Configuration Engine (for custom products), Recommendation Service, Analytics Service
- **Deployment**: Each service runs as a Kubernetes Deployment with 3–5 replicas, using Docker containers
- **Communication**: gRPC for internal service communication, HTTP/REST for external APIs. Currently unencrypted, posing security risks

#### Load Balancing
- **External**: AWS Application Load Balancer (ALB) with auto-scaling for public-facing traffic
- **Internal**: Kubernetes Service with ClusterIP for inter-service communication, lacking encryption

#### Monitoring and Logging
- **Tools**: AWS CloudWatch for basic metrics and logs, partial Prometheus/Grafana setup for Kubernetes metrics
- **Issues**: Limited visibility into service communication security, no alerts for unauthorized access attempts

#### CI/CD Pipeline
- **Tools**: GitHub Actions for CI/CD, with Docker images stored in Amazon ECR
- **Process**: Code commits trigger automated builds, tests, and deployments to EKS via kubectl apply. Manual schema migrations for database updates
- **Issues**: No security checks in the pipeline for service communication, manual deployment validations

### DevOps Processes

#### Deployment
- Developers push code to GitHub, triggering GitHub Actions workflows for builds and deployments
- Manual rollouts with kubectl, with basic canary deployments for critical services like Payment Gateway

#### Scaling
- Manual pod scaling during peak traffic (e.g., product launches), with Horizontal Pod Autoscaler (HPA) for some services
- No cluster auto-scaling, leading to overprovisioning during low traffic

#### Backup and Recovery
- Manual EBS snapshots to S3 (twice weekly), with basic database backups not critical for this project

#### Security
- IAM roles for EKS nodes, with RBAC for Kubernetes access
- Limited network policies, allowing unrestricted internal service communication
- Secrets stored in Kubernetes Secrets, with manual rotation, posing security risks

#### Challenges
- Unencrypted service communication risks data interception, failing security audits
- Manual security management strains the small team
- Limited monitoring for security events, delaying incident response
- Growing complexity with niche product configurations requires secure, scalable communication

## Project Objectives

The primary objective is to implement a mutual TLS (mTLS) service mesh to secure service-to-service communication within the Kubernetes-based e-commerce platform, ensuring compliance with security audits. The implementation leverages:

- **Linkerd**: Service mesh for mTLS enforcement and observability
- **cert-manager**: Automated TLS certificate issuance and renewal
- **Vault**: Secure storage and management of secrets (e.g., private keys, certificates)

The project addresses unencrypted gRPC and HTTP communication, enhances security for niche product configurations, and supports scalability for rapid growth.

## Solution: Stage 1 - Planning and Environment Setup

The first phase focused on planning the mTLS implementation and preparing the EKS environment for Linkerd, cert-manager, and Vault deployment. This stage ensured compatibility with the existing application and laid the foundation for secure communication.

### Requirement Analysis

#### Collaboration
The engineering team (2 developers, 1 DevOps engineer) collaborated with stakeholders, including the store owner, product manager, and a contracted security consultant, via workshops and audits.

**Stakeholders emphasized**:
- **Owner**: Compliance with security audits to maintain customer trust
- **Product Manager**: No disruption to custom product configuration workflows
- **Security Consultant**: Encrypted communication and automated certificate management

#### Workload Assessment
- **Traffic**: ~1,000 API requests/day, peaking at 5,000 during product launches, driven by custom configurations and inventory queries
- **Services**: 8 microservices with ~30–50 daily gRPC/HTTP calls per service, all unencrypted
- **Security Requirements**: mTLS for all internal communication, with certificates valid for 90 days and auto-renewed
- **Performance Needs**: <10ms added latency from mTLS, maintaining <100ms end-to-end API response time
- **Availability**: 99.95% uptime, with no downtime during mTLS rollout

#### Key Goals
- **Secure Communication**: Enforce mTLS for all service-to-service traffic
- **Automation**: Automate certificate issuance/renewal and secret management
- **Audit Compliance**: Pass security audits with encrypted communication and secure key storage
- **Observability**: Monitor mTLS health and performance via service mesh metrics
- **Scalability**: Support doubled traffic without security bottlenecks

#### Constraints
- Budget: ~$600/month for infrastructure, reflecting $2M revenue
- Small team requires high automation to manage complexity
- No downtime during peak sales (e.g., product launches)

### Infrastructure Setup

#### Cluster Verification
- Confirmed EKS cluster (version 1.24) compatibility with Linkerd, cert-manager, and Vault, per Linkerd documentation and cert-manager documentation
- Verified compute resources: 6 t3.large nodes (~24 vCPU, 48 GB RAM) support service mesh overhead (~10% CPU/memory per pod)
- Validated storage: 2 TB EBS gp3 with 4000 IOPS, sufficient for application and Vault storage

#### Namespace Configuration
- Created namespaces: `linkerd` (Linkerd control plane), `cert-manager` (certificate management), `vault` (secret storage), and `prod-app` (application services)
- Applied RBAC policies to restrict access to `linkerd`, `cert-manager`, and `vault` namespaces to the DevOps engineer

#### Application Deployment
- Deployed microservices (Product Catalog, Inventory Management, etc.) into `prod-app` namespace, using existing Kubernetes manifests
- Verified service functionality (e.g., gRPC calls, HTTP APIs) via automated tests in a sandbox EKS cluster, ensuring no pre-existing issues

#### Service Mesh Preparation
- Installed Linkerd CLI (`linkerd check`) to validate cluster readiness, per Linkerd EKS guide
- Planned Linkerd control plane deployment in `linkerd` namespace, with mTLS enabled for all services
- Configured Linkerd's proxy sidecar injection for application pods, ensuring minimal configuration changes

#### Certificate Management
- Planned cert-manager deployment in `cert-manager` namespace, using Let's Encrypt for public certificates and a self-signed CA for internal mTLS, per cert-manager EKS guide
- Configured ClusterIssuer for mTLS certificates with 90-day validity and auto-renewal

#### Secret Management
- Planned Vault deployment in `vault` namespace, using EBS for persistent storage (10 GB, gp3)
- Configured Vault with AWS KMS for auto-unsealing and secret encryption, per Vault AWS guide
- Created IAM role for Vault to access KMS and S3 for backup, with policies: `kms:Encrypt`, `kms:Decrypt`, `s3:PutObject`, `s3:GetObject`

**Terraform output (hypothetical)**:
```bash
terraform output
vault_irsa = "arn:aws:iam::123456789012:role/vault-on-eks-prod-irsa"
vault_s3_bucket = "tech-gadget-vault-backup-bucket"
```

#### Monitoring and Logging Preparation
- Planned Prometheus/Grafana in `monitoring` namespace, integrating Linkerd's built-in metrics for mTLS health, latency, and success rates
- Configured CloudWatch for EKS and Vault logs, with alarms for certificate failures or unauthorized access

#### Security
- Created Kubernetes Secrets for initial Vault and cert-manager configurations, with plans for Vault-managed secrets post-deployment
- Planned network policies to restrict service-to-service access, enforced by Linkerd's mTLS

### Objective
- **mTLS Strategy**: Deploy Linkerd, cert-manager, and Vault in a sandbox environment, validate mTLS for all microservices, and ensure no performance or functional issues
- **Environment Readiness**: Prepare EKS cluster with namespaces, IAM roles, storage, and monitoring to support secure communication without disrupting application workflows

## Post-Implementation Infrastructure (Expected Outcomes)

After implementing the mTLS service mesh, the Niche Tech Gadget Seller's infrastructure and DevOps processes are significantly enhanced, securing communication and supporting growth.

### Infrastructure Components

#### Kubernetes Cluster
- **Platform**: Amazon EKS (version 1.24, planned upgrade to 1.26)
- **Nodes**: 7 t3.large instances (4 vCPU, 8 GB RAM each) across three AZs, adding 1 node for service mesh overhead, costing $0.10/hour per node ($504/month)
- **Storage**: EBS gp3 with 2 TB capacity, adding 10 GB for Vault, maintaining 4000 IOPS
- **Networking**: Network policies restrict all service-to-service traffic to mTLS-verified connections, using AWS CNI and ALB

#### Database
- Unchanged (PostgreSQL 14, StatefulSet), as mTLS focuses on service communication, with future CNPG migration planned

#### Microservices
- **Configuration**: All services (Product Catalog, Inventory Management, etc.) injected with Linkerd proxy sidecars, enforcing mTLS
- **Communication**: gRPC and HTTP traffic encrypted with mTLS, verified by cert-manager-issued certificates
- **Replicas**: 4–6 per service, handling doubled peak traffic with <10ms mTLS overhead

#### Load Balancing
ALB with auto-scaling, internal Services using Linkerd's mTLS for secure communication

#### Monitoring and Logging
- Prometheus/Grafana with Linkerd dashboards for mTLS success rates, latency, and errors
- CloudWatch with alarms for certificate expirations, unauthorized access, and service failures, costing ~$15/month

#### Security
- **Certificates**: cert-manager issues 90-day mTLS certificates, auto-renewed 7 days before expiry
- **Secrets**: Vault stores private keys and certificates, with automated rotation every 30 days
- **Cost**: $0.08/GB/month for 10 GB EBS ($0.80/month), $0.023/GB/month for 50 GB S3 backups ($1.15/month)

### DevOps Processes

#### Deployment
- Linkerd, cert-manager, and Vault deployed via Helm in respective namespaces
- Application deployments automated via GitHub Actions, with Linkerd sidecar injection enabled
- Canary rollouts with Linkerd's traffic splitting, automated rollbacks based on mTLS metrics

#### Scaling
- Cluster Autoscaler adjusts nodes (50–80% CPU utilization)
- HPA for microservices, targeting 70% CPU/memory, with Linkerd metrics for traffic-based scaling
- Spot Instances for non-critical services, saving ~30% on EC2 costs

#### Backup and Recovery
- Vault backups to S3, with daily snapshots and cross-region replication for DR
- cert-manager state backed up to S3, ensuring certificate recovery

#### Security
- Network policies enforce mTLS-only communication, minimizing attack surface
- Vault manages all secrets, with IRSA for KMS and S3 access
- Pod Security Standards for Linkerd, cert-manager, and Vault pods

#### Monitoring and Alerts
- Grafana dashboards for mTLS health, service latency, and certificate status
- CloudWatch Alarms with Slack notifications, reducing response time to <5 minutes

### Outcomes

**Security**: mTLS encrypts all service communication, passing security audits

**Automation**: cert-manager and Vault automate certificate and secret management, reducing manual tasks by ~70%

**Observability**: Linkerd metrics provide real-time insights, improving incident response

**Scalability**: Supports 5,000 peak API requests/day, with capacity for 2x growth

**Performance**: <10ms mTLS overhead, maintaining <100ms API responses

**Cost Efficiency**: Total ~$521/month (EC2: $504, EBS: $0.80, S3: $1.15, CloudWatch: $15), reducible to ~$400 with Spot Instances

**Business Impact**:
- Secure communication builds trust for niche customers configuring high-value products
- Scalable architecture supports rapid growth in competitive markets
- Reduced security overhead enables focus on new features like advanced product configurators

## Conclusion

The mTLS service mesh implementation transforms the Niche Tech Gadget Seller's e-commerce platform, securing service communication and ensuring audit compliance. By leveraging Linkerd, cert-manager, and Vault, the solution supports the company's unique inventory and configuration needs while enabling scalability for rapid growth. This case study serves as a blueprint for other small e-commerce businesses adopting custom microservice architectures, demonstrating how to implement secure communication in Kubernetes environments.
