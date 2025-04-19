# DevOps HR Questions and AWS Lambda Scripts

This document provides answers to HR-like behavioral and technical questions from a DevOps perspective, drawing on experiences from the **CloudNativePG (CNPG) Database Migration Project** and the **mTLS Service Mesh Project**. It also outlines AWS Lambda scripts used in day-to-day DevOps tasks, relevant to these projects.

## HR Behavioral Questions

### 1. Can you tell me about a time you made a mistake at work?

**Answer**:

In the **CNPG Database Migration Project**, during **Stage 2 (Test Environment Deployment)**, I mistakenly configured the MinIO endpoint in the CNPG backup configuration with an incorrect URL, causing backups to fail silently. This oversight went unnoticed initially because the test environment lacked comprehensive monitoring for backup status.

**What I Did**:
- **Acknowledged the Error**: Upon discovering the issue during failover testing, I immediately informed the team, taking full responsibility for the misconfiguration.
- **Root Cause Analysis**: I traced the error to a typo in the `endpointURL` field in the CNPG cluster manifest (`deploy-cnpg.sh`). I reviewed the logs (`kubectl logs -n cnpg-system`) and validated the MinIO service accessibility.
- **Resolution**: I corrected the endpoint, redeployed the CNPG cluster, and triggered a manual backup to confirm success. I also added a validation step in the CI/CD pipeline (`migrate-and-test.sh`) to check backup status post-deployment.
- **Preventive Measures**: To avoid similar issues, I integrated CNPG backup metrics with Prometheus and Grafana in **Stage 3 (Development Environment)** and documented the backup configuration process in the project README. I also proposed peer reviews for critical manifests in the GitHub Actions workflow.

**Outcome**:
The mistake delayed testing by a few hours but led to improved monitoring and validation processes, enhancing the reliability of the CNPG solution. The team appreciated my transparency, and the incident reinforced the importance of thorough configuration checks in Kubernetes environments.

**DevOps Mindset**:
This experience highlights my commitment to ownership and continuous improvement, key DevOps principles. By automating validations and enhancing observability, I ensured such errors wouldn’t recur, aligning with the project’s goal of reliability.

---

### 2. Can you describe a time when you explained a new topic to people of a different background?

**Answer**:

In the **mTLS Service Mesh Project**, during **Stage 3 (Development Environment Deployment)**, I needed to explain the concept of mutual TLS (mTLS) and the role of the Linkerd service mesh to a team of application developers unfamiliar with service mesh technologies. They primarily focused on application logic and had limited exposure to Kubernetes networking and security.

**Approach**:
- **Simplified Explanation**: I used an analogy, comparing mTLS to a secure handshake between two people exchanging verified IDs before sharing sensitive information. I explained that Linkerd automates this handshake for all service-to-service communication, ensuring encryption and authentication.
- **Tailored Content**: I created a short presentation with diagrams showing how Linkerd, cert-manager, and Vault work together in the EKS cluster. I avoided technical jargon like “sidecar proxies” and focused on benefits like security and audit compliance.
- **Hands-On Demo**: I set up a sandbox environment (similar to **Stage 1**) and demonstrated how to inject Linkerd into their application pods using `linkerd inject`. I showed them the Linkerd dashboard to visualize encrypted traffic, making the concept tangible.
- **Documentation**: I updated the project README with a section on mTLS basics and provided links to Linkerd’s beginner-friendly documentation.

**Outcome**:
The developers gained confidence in deploying their application with Linkerd, and their feedback helped refine the Helmfiles and ArgoCD manifests for easier integration. This collaboration ensured the mTLS solution met both security and usability requirements, passing the client’s audit in **Stage 5**.

**DevOps Mindset**:
This experience reflects my ability to bridge technical and non-technical perspectives, fostering collaboration—a core DevOps value. By aligning explanations with the audience’s context, I facilitated smoother adoption of the service mesh.

---

## Technical Questions

### 3. What do you do when a process uses 100% of CPU, memory, or disk is full?

**Answer (DevOps Perspective)**:

As a DevOps engineer, my approach to handling a process consuming 100% CPU, memory, or a full disk is systematic, proactive, and rooted in observability, automation, and root cause analysis. Below is how I’d address each scenario, referencing the CNPG and mTLS projects.

#### High CPU Usage
**Scenario**: A CNPG PostgreSQL pod in the `cnpg-system` namespace is pegged at 100% CPU during **Stage 4 (Staging Environment)** load testing.

**Steps**:
1. **Identify the Process**:
   - Use `kubectl top pod -n cnpg-system` to confirm the high CPU usage.
   - Exec into the pod (`kubectl exec -n cnpg-system <pod> -- top`) to identify the PostgreSQL process or query causing the spike.
2. **Check Logs and Metrics**:
   - Review CNPG logs (`kubectl logs -n cnpg-system -l cnpg.io/cluster=test-cluster`) for query errors or anomalies.
   - Use Prometheus/Grafana (integrated in **Stage 3**) to analyze CPU metrics and correlate with query patterns.
3. **Root Cause Analysis**:
   - Run `EXPLAIN ANALYZE` on suspected queries via `psql` to identify inefficient queries (e.g., missing indexes).
   - Check CNPG configuration (`max_connections`, `work_mem`) for potential over-allocation.
4. **Immediate Mitigation**:
   - If a specific query is the culprit, terminate it using `pg_terminate_backend` to stabilize the system.
   - Scale the CNPG cluster (`kubectl edit cluster test-cluster -n cnpg-system`) to add more replicas or increase CPU limits temporarily.
5. **Long-Term Fix**:
   - Optimize queries or add indexes based on analysis.
   - Adjust PostgreSQL parameters in the CNPG manifest (`manifests/cnpg-cluster.yaml`) and redeploy via ArgoCD.
   - Update the CI/CD pipeline (`ci-cd.yaml`) to include load testing with query optimization checks.
6. **Automation**:
   - Set up alerts in Prometheus for CPU thresholds (e.g., >90% for 5 minutes).
   - Use an AWS Lambda function (see below) to scale EKS nodes or notify the team via Slack.

**Example from CNPG Project**:
During load testing, a poorly optimized query caused CPU spikes. I added an index, tuned `work_mem`, and automated CPU monitoring, reducing resource usage by 30%.

---

#### High Memory Usage
**Scenario**: The Linkerd proxy sidecar in the mTLS project’s staging environment (**Stage 4**) consumes excessive memory, impacting application pods.

**Steps**:
1. **Identify the Issue**:
   - Use `kubectl top pod -n app-namespace` to pinpoint the pod with high memory usage.
   - Check Linkerd metrics via `linkerd dashboard` or Prometheus for memory trends.
2. **Analyze Cause**:
   - Review Linkerd logs (`kubectl logs -n app-namespace <pod> -c linkerd-proxy`) for errors like connection leaks.
   - Verify Linkerd configuration (`helmfiles/linkerd.yaml`) for overly aggressive retry policies or buffering.
3. **Mitigation**:
   - Reduce memory allocation for Linkerd proxies by adjusting resource limits in Helmfiles.
   - Restart affected pods (`kubectl delete pod -n app-namespace <pod>`) to clear memory leaks.
4. **Long-Term Fix**:
   - Optimize Linkerd settings (e.g., lower `maxRetries`, adjust `bufferCapacity`) and redeploy via GitHub Actions.
   - Increase EKS node capacity using an AWS Lambda function if memory pressure persists.
5. **Automation**:
   - Configure Prometheus alerts for memory usage.
   - Implement a Lambda script to auto-scale EKS nodes based on memory metrics from CloudWatch.

**Example from mTLS Project**:
A misconfigured retry policy caused memory spikes in Linkerd. I tuned the configuration and added memory alerts, ensuring stable performance in production (**Stage 5**).

---

#### Disk Full
**Scenario**: The MinIO pod in the CNPG project’s test environment (**Stage 2**) runs out of disk space due to excessive backup storage.

**Steps**:
1. **Confirm the Issue**:
   - Check disk usage (`kubectl exec -n cnpg-system minio -- df -h`).
   - Verify PV status (`kubectl get pvc -n cnpg-system`).
2. **Immediate Action**:
   - Delete old CNPG backups (`velero backup delete <old-backup> -n cnpg-system`) to free space.
   - Pause CNPG backup schedules (`kubectl edit schedule cnpg-backup-schedule -n cnpg-system`) to prevent further writes.
3. **Root Cause Analysis**:
   - Check CNPG backup retention policy (`manifests/cnpg-cluster.yaml`) for overly long retention (e.g., 30 days).
   - Verify Velero backup schedules (`velero-backup-schedule.yaml`) for overlapping backups.
4. **Resolution**:
   - Adjust retention policies (e.g., reduce to 7 days) and redeploy via ArgoCD.
   - Increase PV size (`kubectl edit pvc -n cnpg-system`) or provision a larger storage class.
5. **Prevention**:
   - Set up CloudWatch alarms for disk usage on EKS nodes.
   - Use a Lambda function to prune old backups automatically.
   - Document disk management procedures in the README.

**Example from CNPG Project**:
A disk full issue in MinIO was resolved by shortening retention and automating cleanup, preventing disruptions in **Stage 3**.

**DevOps Mindset**:
These responses emphasize observability (Prometheus, CloudWatch), automation (Lambda, CI/CD), and collaboration (documenting fixes), aligning with DevOps principles of reliability and efficiency.

---

## AWS Lambda Scripts for Day-to-Day DevOps Tasks

In the context of the **CNPG Database Migration** and **mTLS Service Mesh** projects, AWS Lambda scripts are invaluable for automating routine DevOps tasks, especially in EKS-based environments. Below are example Lambda scripts tailored to these projects, leveraging AWS services like CloudWatch, S3, EKS, and SNS.

### 1. Auto-Scale EKS Nodes Based on Resource Usage

**Purpose**: Automatically scale EKS worker nodes when CPU or memory usage exceeds thresholds, addressing high CPU/memory issues in CNPG or mTLS deployments.

**Script**:

```python
import json
import boto3
import os

def lambda_handler(event, context):
    eks = boto3.client('eks')
    autoscaling = boto3.client('autoscaling')
    cloudwatch = boto3.client('cloudwatch')
    
    cluster_name = os.environ['EKS_CLUSTER_NAME']
    asg_name = os.environ['ASG_NAME']
    threshold_cpu = float(os.environ['CPU_THRESHOLD'])  # e.g., 80.0
    threshold_mem = float(os.environ['MEM_THRESHOLD'])  # e.g., 80.0
    
    # Get CPU and memory metrics from CloudWatch
    cpu_metrics = cloudwatch.get_metric_data(
        MetricDataQueries=[{
            'Id': 'cpu',
            'MetricStat': {
                'Metric': {
                    'Namespace': 'AWS/EKS',
                    'MetricName': 'node_cpu_utilization',
                    'Dimensions': [{'Name': 'ClusterName', 'Value': cluster_name}]
                },
                'Period': 300,
                'Stat': 'Average'
            }
        }],
        StartTime=datetime.utcnow() - timedelta(minutes=5),
        EndTime=datetime.utcnow()
    )
    
    mem_metrics = cloudwatch.get_metric_data(
        MetricDataQueries=[{
            'Id': 'mem',
            'MetricStat': {
                'Metric': {
                    'Namespace': 'AWS/EKS',
                    'MetricName': 'node_memory_utilization',
                    'Dimensions': [{'Name': 'ClusterName', 'Value': cluster_name}]
                },
                'Period': 300,
                'Stat': 'Average'
            }
        }],
        StartTime=datetime.utcnow() - timedelta(minutes=5),
        EndTime=datetime.utcnow()
    )
    
    cpu_usage = cpu_metrics['MetricDataResults'][0]['Values'][0] if cpu_metrics['MetricDataResults'][0]['Values'] else 0
    mem_usage = mem_metrics['MetricDataResults'][0]['Values'][0] if mem_metrics['MetricDataResults'][0]['Values'] else 0
    
    # Scale up if thresholds exceeded
    if cpu_usage > threshold_cpu or mem_usage > threshold_mem:
        response = autoscaling.update_auto_scaling_group(
            AutoScalingGroupName=asg_name,
            DesiredCapacity=min(current_capacity + 1, max_capacity)
        )
        return {
            'statusCode': 200,
            'body': json.dumps(f'Scaled up ASG {asg_name} due to CPU: {cpu_usage}%, Memory: {mem_usage}%')
        }
    
    return {
        'statusCode': 200,
        'body': json.dumps('No scaling needed')
    }
```

**Usage**:
- Trigger: CloudWatch Events (e.g., every 5 minutes).
- Environment Variables: `EKS_CLUSTER_NAME`, `ASG_NAME`, `CPU_THRESHOLD`, `MEM_THRESHOLD`.
- Permissions: `eks:DescribeCluster`, `autoscaling:UpdateAutoScalingGroup`, `cloudwatch:GetMetricData`.
- Relevance: Used in CNPG **Stage 4** to handle CPU/memory spikes during load testing and in mTLS **Stage 5** for production stability.

---

### 2. Prune Old CNPG and Velero Backups

**Purpose**: Automatically delete old backups from MinIO/S3 to prevent disk full issues, as seen in the CNPG project.

**Script**:

```python
import boto3
import datetime
import os

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    bucket = os.environ['BUCKET_NAME']
    prefix = os.environ['BACKUP_PREFIX']  # e.g., 'cnpg-backups/' or 'velero-backups/'
    retention_days = int(os.environ['RETENTION_DAYS'])  # e.g., 7
    
    # List objects in S3
    response = s3.list_objects_v2(Bucket=bucket, Prefix=prefix)
    deleted_objects = []
    
    # Check each object
    for obj in response.get('Contents', []):
        last_modified = obj['LastModified']
        age = (datetime.datetime.now(datetime.timezone.utc) - last_modified).days
        
        if age > retention_days:
            s3.delete_object(Bucket=bucket, Key=obj['Key'])
            deleted_objects.append(obj['Key'])
    
    return {
        'statusCode': 200,
        'body': json.dumps(f'Deleted {len(deleted_objects)} old backups: {deleted_objects}')
    }
```

**Usage**:
- Trigger: CloudWatch Events (e.g., daily).
- Environment Variables: `BUCKET_NAME`, `BACKUP_PREFIX`, `RETENTION_DAYS`.
- Permissions: `s3:ListBucket`, `s3:DeleteObject`.
- Relevance: Used in CNPG **Stage 2** and **Stage 3** to manage MinIO disk space and in mTLS **Stage 1** for Velero backup cleanup (though not required by the client).

---

### 3. Notify Team of CNPG Failover or Backup Failures

**Purpose**: Send Slack notifications when CNPG failover occurs or backups fail, enhancing monitoring in the CNPG project.

**Script**:

```python
import json
import boto3
import urllib.request
import os

def lambda_handler(event, context):
    sns = boto3.client('sns')
    topic_arn = os.environ['SNS_TOPIC_ARN']
    webhook_url = os.environ['SLACK_WEBHOOK_URL']
    
    # Parse CloudWatch event
    message = json.loads(event['Records'][0]['Sns']['Message'])
    alarm_name = message['AlarmName']
    new_state = message['NewStateValue']
    
    # Send Slack notification
    slack_message = {
        'text': f'Alert: {alarm_name} is in {new_state} state. Check CloudWatch for details.'
    }
    
    req = urllib.request.Request(
        webhook_url,
        data=json.dumps(slack_message).encode('utf-8'),
        headers={'Content-Type': 'application/json'}
    )
    
    with urllib.request.urlopen(req) as response:
        if response.getcode() != 200:
            raise Exception('Failed to send Slack notification')
    
    return {
        'statusCode': 200,
        'body': json.dumps('Notification sent to Slack')
    }
```

**Usage**:
- Trigger: SNS topic subscribed to CloudWatch alarms (e.g., CNPG failover or backup failure).
- Environment Variables: `SNS_TOPIC_ARN`, `SLACK_WEBHOOK_URL`.
- Permissions: `sns:Publish`.
- Relevance: Used in CNPG **Stage 5** for production monitoring and in mTLS **Stage 5** for alerting on service mesh issues.

---

### 4. Rotate Vault Secrets for mTLS

**Purpose**: Automatically rotate secrets in Vault for the mTLS project to maintain security compliance.

**Script**:

```python
import hvac
import os
import json

def lambda_handler(event, context):
    vault_url = os.environ['VAULT_URL']
    vault_token = os.environ['VAULT_TOKEN']
    secret_path = os.environ['SECRET_PATH']  # e.g., 'secret/mtls-cert'
    
    # Initialize Vault client
    client = hvac.Client(url=vault_url, token=vault_token)
    
    # Rotate secret (generate new certificate key)
    new_secret = {
        'private_key': generate_new_private_key(),  # Custom function to generate key
        'last_rotated': datetime.datetime.now().isoformat()
    }
    
    # Update secret in Vault
    client.secrets.kv.v2.create_or_update_secret(
        path=secret_path,
        secret=new_secret,
        mount_point='secret'
    )
    
    return {
        'statusCode': 200,
        'body': json.dumps(f'Rotated secret at {secret_path}')
    }

def generate_new_private_key():
    # Placeholder for key generation logic (e.g., using cryptography library)
    return 'new-private-key-placeholder'
```

**Usage**:
- Trigger: CloudWatch Events (e.g., weekly).
- Environment Variables: `VAULT_URL`, `VAULT_TOKEN`, `SECRET_PATH`.
- Permissions: IAM role with access to Vault (via AWS IAM Auth or static token).
- Relevance: Used in mTLS **Stage 5** to rotate secrets for cert-manager and ensure audit compliance.

---

## Summary

The behavioral answers demonstrate my ability to learn from mistakes, communicate effectively with diverse teams, and uphold ethical standards, drawing on real scenarios from the CNPG and mTLS projects. The technical responses showcase a DevOps approach to resource issues, emphasizing observability, automation, and collaboration. The AWS Lambda scripts provide practical automation for scaling, cleanup, notifications, and secret management, directly supporting the operational needs of both projects. These tools and practices ensure reliability, security, and efficiency in Kubernetes-based environments, aligning with the projects’ goals of high availability and audit compliance.