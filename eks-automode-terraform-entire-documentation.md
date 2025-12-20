# Amazon EKS Auto Mode (Kubernetes 1.33) – Terraform documentation
## 1. Overview

EKS Auto Mode is a fully managed compute and networking model where AWS integrates Karpenter functionality directly into the EKS control plane. With Auto Mode enabled, AWS manages:

* Node provisioning and scaling

* Core networking integration

* Block storage integration

* Load balancer integration

**This setup establishes:**

* A production-ready EKS Auto Mode cluster

* Secure IAM-based authentication using EKS Access Entries

* OIDC provider configuration for future IRSA use

* A clean Terraform module structure with clear ownership boundaries 

## 2. Prerequisites

Before applying this Terraform configuration, ensure the following prerequisites are met.

AWS Prerequisites

* AWS account with permissions to create:

    1. EKS clusters

    2. IAM roles and policies

    3. VPC-related resources (already handled separately)

* AWS CLI installed and configured

* IAM user or role available for EKS admin access

**Tooling Prerequisites:** 

* Terraform ≥ 1.5

* AWS provider ≥ 5.x

* kubectl installed

* Access to the target AWS region (ap-south-2) 

## 3. Terraform Folder Structure: 
This repository follows a layered and modular structure.
```
terraform-aws/
├── main.tf
├── providers.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── eks/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
```
**Design Principles**

* VPC module is independent and reusable

* EKS module only consumes VPC outputs

* Cluster access, IAM, and OIDC are owned by the EKS module

* Kubernetes runtime resources are handled separately 

4. Provider Configuration
* the provider configuration we are following for vpc module is enough for the eks cluster setup. no need to change anything as of now

## 4. EKS Module Interface 

Variables Used by the EKS Module

### Typical variables include:

    cluster_name

    kubernetes_version

    vpc_id

    private_subnet_ids

    public_subnet_ids

    admin_principal_arns

    region 

### Calling the EKS Module
```
module "eks" {
  source = "./modules/eks"

  cluster_name          = var.cluster_name
  kubernetes_version    = "1.33"
  vpc_id                = module.vpc.vpc_id
  private_subnet_ids    = module.vpc.private_subnet_ids
  public_subnet_ids     = module.vpc.public_subnet_ids
  admin_principal_arns  = var.admin_principal_arns
}
``` 

### Outputs from the EKS Module
The EKS module exports key outputs used by other layers:

    cluster_name

    cluster_endpoint

    cluster_certificate_authority

    cluster_role_arn

    node_role_arn

    oidc_issuer_url

    oidc_provider_arn

These outputs are required later for:

    kubectl authentication

    IRSA-based workloads

    Kubernetes provider configuration

### IAM Roles for EKS Control Plane
* EKS Cluster Role

* The EKS control plane assumes an IAM role with the following AWS-managed policies:

    * AmazonEKSClusterPolicy

    * AmazonEKSComputePolicy

    * AmazonEKSNetworkingPolicy

    * AmazonEKSBlockStoragePolicy

    * AmazonEKSLoadBalancingPolicy 

* Node Role (Auto Mode)

A node IAM role is still required even though:

Nodes are provisioned dynamically

Node lifecycle is managed by AWS

This role is used by:

Auto Mode–managed EC2 instances

ECR image pulls

Basic node-to-control-plane communication

## 5. OIDC Provider and EKS Access Entries
The EKS module enables an OIDC provider using the cluster’s issuer URL.

* Why this matters:

    * Enables IAM Roles for Service Accounts (IRSA)

    * Required for future workload-level IAM access

    * Created once per cluster

Even though IRSA is not used immediately, it is enabled upfront to avoid future cluster mutation.


## 6. EKS Access Entries (Modern Authentication Model)

* This setup uses EKS Access Entries, not the legacy aws-auth ConfigMap.

* Key points:

    * Authentication is IAM-based

    * Authorization is managed via AWS-defined access policies

    * No manual RBAC YAML required

* Admins are associated with:

    * AmazonEKSClusterAdminPolicy

    * Cluster-wide access scope

* Important Lifecycle Consideration

  * Access entries are protected with:
    ```
    lifecycle {
      prevent_destroy = true
    }
    ``` 

* Why:

    * Prevents accidental lockout during terraform destroy

    * Ensures Kubernetes resources can be cleaned up safely

## 7. Why Karpenter IRSA Is NOT Required in EKS Auto Mode

* In traditional EKS setups:

    * Karpenter is installed via Helm

    * A dedicated IRSA role is required

    * Customers manage upgrades and permissions

* In EKS Auto Mode:

    * Karpenter is fully managed by AWS

    * It runs outside customer-visible namespaces

    * IAM permissions are handled internally by AWS

    * No Helm installation is required

    * No IRSA configuration is required

## 8. NodeClass & NodePool 
* The objective of this setup is to:

    * Enable dynamic, demand-based node provisioning

* Isolate workloads using multiple NodePools 

    * CPU – General workloads

    * CPU – Heavy workloads

    * GPU – AI/ML workloads

* Avoid static EC2 node groups

* Follow AWS-recommended EKS Auto Mode architecture

### Provider Configuration

**Why kubectl_manifest Provider Is Used**
* Initially, Kubernetes resources were attempted using the Terraform Kubernetes provider (kubernetes_manifest).
* However, this provider strictly validates schemas and fails for custom resources (CRDs) like:

  * NodeClass.eks.amazonaws.com

  * NodePool.karpenter.sh

* Since both NodeClass and NodePool are CRDs, the correct provider is:

  * gavinbunney/kubectl provider

* This provider applies raw YAML directly using Kubernetes APIs and supports CRDs reliably.

**Require provider configuration** 
```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 6.24.0"
    }

    kubectl = {
      source  = "gavinbunney/kubectl"
      version = ">= 1.14.0"
    }
  }
}
```
```
provider "kubectl" {
  config_path      = pathexpand("~/.kube/config")
  load_config_file = true
}
```
* Key Architectural Insight

    * **EKS Auto Mode = NodeClass (AWS) + NodePool (Karpenter)**
    * Both are required and owned by different controllers.

### IAM Role Used in NodeClass
* The NodeClass references the EKS Auto Mode node IAM role created as part of the EKS cluster provisioning.
Example: 
```
spec:
  role: arn:aws:iam::<account-id>:role/<eks-auto-node-role>
```

**Important Validation Rule**

* Exactly one of the following must be provided:
    * `role`
    * `instanceProfile` 

* Using `roleARN` results in validation failure.

### GPU NodeClass – Block Device Mapping 
* this is one of the key-differnces between cpu-nodeclass and gpu-nodeclass Configuration

* GPU workloads require larger local storage for:

    * Model downloads

    * CUDA libraries

    * Runtime artifacts

```
blockDeviceMappings:
  - deviceName: /dev/xvda
    ebs:
      volumeSize: 100
      volumeType: gp3
``` 

### NodePool Requirements – Design & Learnings

**Why Requirements Are Used Instead of Instance Types** 
* Hard-coding instance types (e.g., c5.large) limits flexibility and increases operational overhead.

* Instead, requirements express intent:

* What kind of nodes are allowed not Which exact instance to use 

* This allows Karpenter to:

  * Choose cost-optimal instances

  * Automatically adopt newer EC2 generations

  * Scale intelligently based on pod demand 

**GPU NodePool Requirements (Dev-Aligned):** 
Mapped NodePool requirements:
```
requirements:
  - key: karpenter.k8s.aws/instance-category
    operator: In
    values: ["g"]

  - key: karpenter.k8s.aws/instance-family
    operator: In
    values: ["g5"]

  - key: karpenter.k8s.aws/instance-size
    operator: In
    values: ["xlarge", "2xlarge", "4xlarge"]

  - key: kubernetes.io/arch
    operator: In
    values: ["amd64"]
``` 

## NodePool Limits – Safety Guardrails 
* Purpose of Limits is NodePool limits define maximum infrastructure capacity, not pod limits.
Example:
```
limits:
  cpu: "80"
``` 
Means:Total CPU provisioned by this NodePool cannot exceed 80 vCPUs. 