# EKS CAPABILITIES
* EKS Capabilities are Kubernetes-native features for declarative continuous deployment, AWS resource management, and Kubernetes resource authoring and orchestration, all fully managed by AWS

* With EKS Capabilities, you can focus more on building and scaling your workloads, offloading the operational burden of these foundational platform services to AWS

* These capabilities run within EKS rather than in your clusters, eliminating the need to install, maintain, and scale critical platform components on your worker nodes.

* To get started, you can create one or more EKS Capabilities on a new or existing EKS cluster. To do this, you can use the AWS CLI, the AWS Management Console, EKS APIs, eksctl, or your preferred infrastructure-as-code tools

**Available Capabilities:**
1. AWS Controllers for Kubernetes (ACK)
2. Argo CD
3. kro (Kube Resource Orchestrator) 

## ARGO CD
* Argo CD implements GitOps-based continuous deployment for your applications, using Git repositories as the source of truth for your workloads and system state.

* Argo CD automatically syncs application resources to your clusters from your Git repositories, detecting and remediating drift to ensure your deployed applications match your desired state

* Argo CD continuously monitors both the source and the live state in the cluster, automatically synchronizing any changes to ensure the cluster state matches the desired state.

* Argo CD continuously monitors both the source and the live state in the cluster, automatically synchronizing any changes to ensure the cluster state matches the desired state.

* As part of EKS Managed Capabilities, Argo CD is fully managed by AWS, eliminating the need to install, configure, and maintain Argo CD infrastructure. AWS handles scaling, patching, and operational management, allowing your teams to focus on application delivery rather than tool maintenance.

* The Argo CD UI provides visualization and monitoring capabilities, allowing you to view the deployment status, health, and history of your applications

* The UI integrates with AWS Identity Center (formerly AWS SSO) for seamless authentication and authorization, enabling you to control access using your existing identity management infrastructure.

## Integration with AWS Identity Center
* EKS Managed Capabilities provides direct integration between Argo CD and AWS Identity Center, enabling seamless authentication and authorization for your users
* When you enable the Argo CD capability, you can configure AWS Identity Center integration to map Identity Center groups and users to Argo CD RBAC roles, allowing you to control who can access and manage applications in Argo CD.

## Create an Argo CD capability
### Prerequisites
Before creating an Argo CD capability, ensure you have:
* An existing Amazon EKS cluster running a supported Kubernetes version
* AWS Identity Center configured - Required for Argo CD authentication
* An IAM Capability Role with permissions for Argo CD
* An IAM Capability Role with permissions for Argo CD
* kubectl configured to communicate with your cluster

### Choose your tool
* You can create an Argo CD capability using the AWS Management Console, AWS CLI, or eksctl or any other infrastructure-as-a-code tool of your wish

### What happens when you create an Argo CD capability
When you create an Argo CD capability:

1. EKS creates the Argo CD capability service and configures it to monitor and manage resources in your cluster

2. Custom Resource Definitions (CRDs) are installed in your cluster

3. The capability assumes the IAM Capability Role you provide

4. Argo CD begins watching for its custom resources

5. The capability status changes from CREATING to ACTIVE

6. The Argo CD UI becomes accessible through its endpoint

Once active, you can create Argo CD Applications in your cluster to deploy from Git repositories. 

