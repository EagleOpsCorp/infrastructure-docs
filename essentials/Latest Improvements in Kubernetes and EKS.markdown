# Latest Improvements in Kubernetes and EKS

As of April 2025, Kubernetes and Amazon Elastic Kubernetes Service (EKS) have introduced significant enhancements that improve container orchestration, scalability, and ease of use. Below is a comprehensive overview of the latest improvements in Kubernetes version 1.32 and the corresponding updates in EKS, which supports this version.

## Kubernetes Version 1.32: Key Enhancements

Kubernetes 1.32, released in late 2024, includes a range of new features and improvements aimed at enhancing resource management, platform compatibility, and operational efficiency. The following table summarizes the major updates, as detailed in the official changelog ([Kubernetes 1.32 Changelog](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.32.md)).

| **Feature** | **Description** | **Impact** |
|-------------|-----------------|------------|
| **Windows Support for Node Memory Manager** | Adds memory management capabilities for Windows nodes ([PR #128560](https://github.com/kubernetes/kubernetes/pull/128560)). | Enables better resource allocation for Windows-based workloads. |
| **New Metrics for DRA** | Introduces metrics for Dynamic Resource Allocation (DRA) Node operations and GRPC calls latency ([PR #127146](https://github.com/kubernetes/kubernetes/pull/127146)). | Improves monitoring and performance tuning for resource-intensive applications. |
| **Kubelet Memory Manager to GA** | Graduates the kubelet memory manager to General Availability ([PR #128517](https://github.com/kubernetes/kubernetes/pull/128517)). | Ensures stable and reliable memory management for containers. |
| **SchedulerQueueingHints to Beta** | Promotes SchedulerQueueingHints to beta, enabled by default ([PR #128472](https://github.com/kubernetes/kubernetes/pull/128472)). | Enhances scheduling efficiency, reducing pod placement delays. |
| **CPU Affinity on Windows via CRI** | Adds support for CPU affinity on Windows through the Container Runtime Interface ([PR #124285](https://github.com/kubernetes/kubernetes/pull/124285)). | Allows precise CPU resource allocation for Windows containers. |
| **API Changes** | Introduces the `/resize` subresource for pod resource resizing ([PR #128266](https://github.com/kubernetes/kubernetes/pull/128266)). | Simplifies dynamic resource adjustments for running pods. |

### Additional Kubernetes Improvements
- **API Enhancements**: The Dynamic Resource Allocation (DRA) API now supports up to 256 pods per ResourceClaim, up from 32, though downgrading to 1.32.0 is not supported if exceeding 32 entries ([PR #129544](https://github.com/kubernetes/kubernetes/pull/129544)). A new `/flagz` endpoint for kube-apiserver ([PR #127581](https://github.com/kubernetes/kubernetes/pull/127581)) and a `singleProcessOOMKill` flag for cgroups v2 ([PR #126096](https://github.com/kubernetes/kubernetes/pull/126096)) improve operational control.
- **Performance and Scalability**: The `SchedulerAsyncPreemption` feature gate introduces alpha support for asynchronous pod preemption, boosting scheduler performance ([PR #128170](https://github.com/kubernetes/kubernetes/pull/128170)). The PVC Protection Controller now batch-processes PVCs by namespace, enhancing scalability ([PR #125372](https://github.com/kubernetes/kubernetes/pull/125372)).
- **Operational Features**: A new `/statusz` endpoint for kube-apiserver ([PR #125577](https://github.com/kubernetes/kubernetes/pull/125577)) and a health check for device plugin gRPC registration ([PR #128432](https://github.com/kubernetes/kubernetes/pull/128432)) improve system reliability. The `OrderedNamespaceDeletion` feature gate ensures pods are deleted before other resources, enhancing workload security ([PR #130508](https://github.com/kubernetes/kubernetes/pull/130508)).
- **Tooling Updates**: Kubernetes is built with Go 1.23.3 ([PR #128852](https://github.com/kubernetes/kubernetes/pull/128852)), and tools like coreDNS (v1.11.3, [PR #126449](https://github.com/kubernetes/kubernetes/pull/126449)), cni-plugins (v1.6.0, [PR #128091](https://github.com/kubernetes/kubernetes/pull/128091)), and etcd client (v3.5.16, [PR #127279](https://github.com/kubernetes/kubernetes/pull/127279)) have been upgraded.

These updates collectively make Kubernetes 1.32 a robust platform for managing diverse workloads, with improved support for Windows, better resource allocation, and enhanced operational tools.

## Amazon EKS: Support for Kubernetes 1.32 and New Features

Amazon EKS supports Kubernetes 1.32 as of January 23, 2025, with standard support until March 23, 2026, and extended support until March 23, 2027 ([AWS EKS Kubernetes Versions](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)). Beyond supporting the latest Kubernetes version, EKS has introduced several features to simplify cluster management and enhance integration with AWS services. The following table outlines the key EKS-specific enhancements, as reported in recent AWS blog posts.

| **Feature** | **Description** | **Impact** |
|-------------|-----------------|------------|
| **EKS Auto Mode** | Streamlines Kubernetes operations by automating cluster infrastructure management ([EKS Auto Mode Workshop](https://aws.amazon.com/blogs/containers/introducing-the-amazon-eks-auto-mode-workshop/)). | Reduces operational overhead, enabling focus on application development. |
| **EKS Pod Identity** | Simplifies IAM permissions for EKS add-ons, launched at re:Invent 2023 ([EKS Pod Identity](https://aws.amazon.com/blogs/containers/simplifying-iam-permissions-for-amazon-eks-addons-with-eks-pod-identity/)). | Enhances security and simplifies permission management. |
| **EKS Hybrid Nodes** | Enables running GenAI inference workloads across cloud and on-premises environments ([EKS Hybrid Nodes](https://aws.amazon.com/blogs/containers/run-genai-inference-across-environments-with-amazon-eks-hybrid-nodes/)). | Supports hybrid cloud strategies for AI workloads. |
| **Community Add-ons Catalog** | Provides add-ons to support operational capabilities for Kubernetes applications ([EKS Add-ons Catalog](https://aws.amazon.com/blogs/containers/announcing-amazon-eks-community-add-ons-catalog/)). | Expands functionality with community-supported tools. |
| **EBS Volume Modification** | Uses VolumeAttributesClass API to modify EBS volumes without downtime ([EBS Volume Modification](https://aws.amazon.com/blogs/containers/modify-amazon-ebs-volumes-on-kubernetes-with-volume-attributes-classes/)). | Improves storage flexibility and application availability. |

### Detailed EKS Enhancements
- **EKS Auto Mode**: Launched at re:Invent 2024, EKS Auto Mode eliminates the need to manually manage cluster infrastructure for production-grade Kubernetes applications. It supports deployment with a single command, dynamic resource scaling, cost optimization, and streamlined cluster upgrades. Users can enable it via AWS CLI, Terraform, eksctl, or the AWS console ([EKS Auto Mode Documentation](https://docs.aws.amazon.com/eks/latest/userguide/automode.html)).
- **EKS Pod Identity**: This feature simplifies the process of assigning IAM roles to EKS add-ons, reducing the complexity of permission management. It is particularly useful for securing add-ons without extensive IAM configuration.
- **EKS Hybrid Nodes**: Designed for GenAI inference workloads, this feature allows seamless operation across AWS cloud and on-premises environments, catering to organizations with hybrid infrastructure needs.
- **Community Add-ons Catalog**: The catalog provides a curated set of add-ons to enhance Kubernetes applications, such as monitoring, logging, and security tools, making it easier to extend EKS functionality.
- **EBS Volume Modification**: Leveraging the VolumeAttributesClass API, EKS users can now modify Amazon EBS volumes (e.g., size or performance characteristics) without interrupting running applications, improving storage management efficiency.

### Important Notes for EKS Users
- **Kubernetes 1.32 on EKS**: The availability of Kubernetes 1.32 on EKS includes specific changes, such as the removal of the `flowcontrol.apiserver.k8s.io/v1beta3` API version and the deprecation of certain ServiceAccount annotations. Users must update configurations before upgrading ([Kubernetes 1.32 Release Notes](https://kubernetes.io/blog/2024/12/11/kubernetes-v1-32-release/)).
- **Amazon Linux Transition**: Kubernetes 1.32 is the last version to support Amazon Linux 2 (AL2) AMIs on EKS. Starting with 1.33, only Amazon Linux 2023 (AL2023) and Bottlerocket AMIs will be available.
- **Security Enhancements**: Anonymous authentication in Kubernetes 1.32 on EKS is restricted to specific endpoints (`/healthz`, `/livez`, `/readyz`), with other endpoints returning `401 Unauthorized` for unauthenticated users.

## Community Insights
Recent discussions on X highlight ongoing interest in EKS optimization. For example, a post from April 15, 2025, by @TheNJDevOpsGuy emphasizes workload isolation, multi-AZ configurations, cluster security, and performance optimization as key considerations for EKS deployments ([X Post](https://x.com/TheNJDevOpsGuy/status/1912152103809655289)). These insights align with the new EKS features, which aim to simplify and secure cluster management.

## Conclusion
The latest improvements in Kubernetes 1.32 and EKS demonstrate a commitment to enhancing container orchestration. Kubernetes 1.32 offers better resource management, Windows support, and operational tools, while EKS builds on these with features like Auto Mode and Hybrid Nodes. These updates make it easier for developers and enterprises to manage complex, scalable, and hybrid workloads. For detailed guidance, users can refer to the official Kubernetes changelog, AWS EKS documentation, and AWS blog posts.

## Key Citations
- [Kubernetes 1.32 Official Changelog](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.32.md)
- [AWS EKS Supported Kubernetes Versions](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)
- [Amazon EKS Auto Mode Workshop Announcement](https://aws.amazon.com/blogs/containers/introducing-the-amazon-eks-auto-mode-workshop/)
- [EKS Pod Identity for Simplified IAM Permissions](https://aws.amazon.com/blogs/containers/simplifying-iam-permissions-for-amazon-eks-addons-with-eks-pod-identity/)
- [EKS Hybrid Nodes for GenAI Inference Workloads](https://aws.amazon.com/blogs/containers/run-genai-inference-across-environments-with-amazon-eks-hybrid-nodes/)
- [Amazon EKS Community Add-ons Catalog Announcement](https://aws.amazon.com/blogs/containers/announcing-amazon-eks-community-add-ons-catalog/)
- [Modifying EBS Volumes on Kubernetes with VolumeAttributesClass](https://aws.amazon.com/blogs/containers/modify-amazon-ebs-volumes-on-kubernetes-with-volume-attributes-classes/)
- [EKS Auto Mode Official Documentation](https://docs.aws.amazon.com/eks/latest/userguide/automode.html)
- [Kubernetes 1.32 Release Announcement Blog](https://kubernetes.io/blog/2024/12/11/kubernetes-v1-32-release/)
- [X Post on EKS Optimization Strategies](https://x.com/TheNJDevOpsGuy/status/1912152103809655289)