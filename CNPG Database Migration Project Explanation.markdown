# Project Explanation: Replacing StatefulSet Database with CloudNativePG (CNPG) for Enhanced Reliability

## Project Goal

The objective of this project was to replace a Kubernetes StatefulSet-based database deployment with **CloudNativePG (CNPG)** to improve reliability, high availability (HA), and disaster recovery (DR) capabilities for a client’s database workload. The client had previously migrated from Amazon RDS to a StatefulSet-based database but encountered operational issues, prompting the transition to CNPG for a more robust and production-ready PostgreSQL solution.

---

## Background

The client initially used **Amazon RDS** for their PostgreSQL database to leverage managed database services. However, to gain more control over the database environment and reduce costs, they migrated to a **StatefulSet-based PostgreSQL deployment** on Kubernetes. This approach introduced challenges, including:

- **Limited High Availability**: The StatefulSet setup lacked native support for automated failover and replication.
- **Complex Management**: Manual intervention was required for scaling, backups, and recovery processes.
- **Reliability Issues**: The deployment was prone to operational failures, impacting application availability.

Due to these issues, the client decided to adopt **CloudNativePG (CNPG)**, a Kubernetes-native operator for PostgreSQL, to address the shortcomings and achieve a more reliable, highly available, and disaster-resistant database solution.

---

## Stage 1: Planning and Environment Setup

The first phase involved planning the migration and preparing the environment for CNPG deployment. Key activities included:

- **Requirement Analysis**:

  - Collaborated with the client to understand the database workload, including data size, transaction volume, and availability requirements.
  - Identified key goals: automated failover, replication, backup/restore, and simplified management.

- **Infrastructure Setup**:

  - Verified the existing Kubernetes cluster (running on Amazon EKS) met CNPG prerequisites, such as sufficient storage and compute resources.
  - Configured a dedicated namespace for the CNPG operator and database instances.
  - Ensured compatibility with the client’s existing storage class for persistent volumes.

- **Objective**:

  - Establish a clear migration strategy and prepare the Kubernetes environment for CNPG deployment.

---

## Stage 2: CNPG Deployment in a Test Environment

To minimize risks, CNPG was first deployed in a test environment to validate its functionality and compatibility with the client’s application. The steps included:

- **CNPG Installation**:

  - Installed the CNPG operator using Helm charts in the test Kubernetes cluster.
  - Configured the operator to manage PostgreSQL clusters with the desired specifications (e.g., version, storage, replicas).

- **Database Cluster Setup**:

  - Deployed a CNPG-managed PostgreSQL cluster with:
    - **Primary and Replica Nodes**: Configured one primary and two replicas for high availability.
    - **Automated Failover**: Enabled CNPG’s built-in failover mechanism to promote a replica in case of primary failure.
    - **Backup Configuration**: Set up continuous backups to an S3-compatible storage backend for disaster recovery.
    - **Point-in-Time Recovery (PITR)**: Configured WAL archiving to support PITR for granular recovery.

- **Data Migration**:

  - Exported data from the existing StatefulSet-based PostgreSQL database using `pg_dump`.
  - Imported the data into the CNPG-managed PostgreSQL cluster.
  - Validated data integrity and consistency post-migration.

- **Testing**:

  - Conducted functional tests to ensure the application could connect to the CNPG cluster without issues.
  - Simulated failover scenarios to verify HA capabilities.
  - Tested backup and restore processes to confirm disaster recovery readiness.

- **Objective**:

  - Validate CNPG’s reliability, HA, and DR features in a controlled environment and ensure compatibility with the client’s application.

---

## Stage 3: Migration to Development Environment

After successful testing, the CNPG solution was deployed to the development environment to further validate its behavior in a more realistic setting. Key activities included:

- **CNPG Deployment**:

  - Replicated the CNPG setup from the test environment, including the operator, PostgreSQL cluster, and backup configurations.
  - Applied environment-specific settings, such as connection strings and resource limits.

- **CI/CD Integration**:

  - Updated the client’s GitHub Actions pipeline to include CNPG deployment steps using Helm charts.
  - Integrated CNPG manifests with ArgoCD for GitOps-based management.

- **Testing**:

  - Performed integration tests to verify application-database interactions.
  - Monitored CNPG’s observability metrics (e.g., via Prometheus and Grafana) to ensure performance and stability.
  - Conducted additional failover and recovery tests to confirm HA and DR capabilities.

- **Objective**:

  - Ensure the CNPG solution integrates seamlessly with the development environment and supports the client’s deployment workflows.

---

## Stage 4: Staging Environment Deployment and Validation

The solution was then promoted to the staging environment, which closely resembled production. This phase focused on rigorous testing and optimization:

- **Deployment**:

  - Deployed the CNPG operator and PostgreSQL cluster in the staging environment using the same Helm charts and ArgoCD manifests.
  - Configured environment-specific parameters, such as increased storage and compute resources.

- **Testing and Validation**:

  - Executed **load tests** to assess the CNPG cluster’s performance under high transaction volumes.
  - Validated **automated failover** by simulating primary node failures and verifying replica promotion.
  - Tested **disaster recovery** by restoring the database from backups and performing PITR.
  - Ran **security scans** to ensure the CNPG cluster complied with the client’s security policies.

- **Optimization**:

  - Tuned PostgreSQL parameters (e.g., `max_connections`, `work_mem`) to optimize performance.
  - Adjusted CNPG’s backup schedules to balance recovery objectives with storage costs.

- **Objective**:

  - Confirm that the CNPG solution meets reliability, HA, and DR requirements in a production-like environment.

---

## Stage 5: Production Deployment and Monitoring

The final phase involved deploying CNPG to the production environment and ensuring long-term stability:

- **Production Deployment**:

  - Executed the migration in a maintenance window to minimize downtime.
  - Deployed the CNPG operator and PostgreSQL cluster using the validated Helm charts and ArgoCD manifests.
  - Performed the final data migration from the StatefulSet database to the CNPG cluster using `pg_dump` and `pg_restore`.

- **Testing**:

  - Ran **smoke tests** to verify application connectivity and basic functionality.
  - Conducted **end-to-end tests** to ensure all application workflows operated correctly.
  - Validated failover, backup, and PITR processes in production.

- **Monitoring and Maintenance**:

  - Integrated CNPG’s metrics with the client’s monitoring stack (Prometheus and Grafana) to track database health, replication lag, and backup status.
  - Set up alerts for critical events, such as failover or backup failures.
  - Documented operational procedures for managing the CNPG cluster, including scaling, upgrades, and recovery.

- **Objective**:

  - Deliver a reliable, highly available, and disaster-resistant PostgreSQL solution in production with robust monitoring and maintenance processes.

---

## Summary

The project successfully replaced a problematic StatefulSet-based PostgreSQL database with **CloudNativePG (CNPG)**, addressing the client’s reliability, high availability, and disaster recovery needs. By migrating from RDS to StatefulSet and finally to CNPG, the client achieved a Kubernetes-native PostgreSQL solution with automated failover, replication, and robust backup/restore capabilities. The phased approach—spanning test, development, staging, and production environments—ensured a smooth transition with minimal disruption. Integration with GitOps, CI/CD pipelines, and monitoring tools further enhanced the solution’s manageability and scalability, meeting the client’s operational and business requirements.