# EKS Auto Mode with Karpenter – NodeClass & NodePool via Terraform

This repository documents the complete implementation of EKS Auto Mode with Karpenter-based autoscaling, using Terraform and Kubernetes CRDs (NodeClass and NodePool).

The goal is to:

    Migrate a SaaS EdTech platform from standalone EC2 to EKS

    Isolate CPU workloads and GPU-heavy AI/ML workloads

    Enable scale-from-zero, cost-efficient autoscaling

    Manage everything declaratively using Terraform

This README captures:

    Architecture decisions

    Folder structure

    Provider choices

    CRD API versions

    Common pitfalls & fixes

    Final working manifests

    Verification & operational behavior  

## High-Level Architecture
```
Terraform
│
├── AWS Infra (VPC, EKS, IAM)
│
└── Kubernetes CRDs (applied via kubectl_manifest)(terraform) 
    ├── NodeClass  (AWS EKS Auto Mode)
    └── NodePool   (Karpenter)
         └── EC2 Nodes (created on demand)
```

## Key Design Principles

Scale from zero (no idle EC2 cost)

Workload isolation

    CPU general

    CPU heavy

    GPU workloads

GPU nodes only created when explicitly requested

Single EKS cluster, multiple NodePools

## Core Concepts (Very Important)
NodeClass (AWS-owned)

API Group: eks.amazonaws.com/v1

Controller: EKS Auto Mode

Purpose: How nodes are created

Defines:

IAM role

Subnets

Security Groups

Root volume

Tags

NodePool (Karpenter-owned)

API Group: karpenter.sh/v1

Controller: Karpenter

Purpose: When & why nodes are created

Defines:

Instance types

Labels

Taints

Limits

Capacity type

⚠️ NodeClass and NodePool belong to DIFFERENT API groups
This is intentional and required in EKS Auto Mode. 

## Repository Structure
```
nexusig-tf-vpc-infra/
├── main.tf
├── providers.tf    
├── terraform.tfvars
├── variables.tf 
├── bootstrap.tf
├── nodeclass-cpu.tf
├── nodeclass-gpu.tf
├── nodepool-cpu-general.tf
├── nodepool-cpu-heavy.tf
├── nodepool-gpu.tf
└── modules/
    ├── vpc/
    └── eks/
```

## Terraform Providers 
Required Providers
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
kubectl Provider Configuration
```
provider "kubectl" {
  config_path      = pathexpand("~/.kube/config")
  load_config_file = true
}
```

## EKS Cluster Setup (Summary)

EKS created via Terraform

Auto Mode enabled

OIDC provider configured

Access entries managed via Terraform

No separate IRSA or Karpenter Helm install required

Karpenter is pre-installed by AWS 

NodeClass Definitions: 
* you can refer the .tf files for node class and node pools in the working repository 

## Critical Rule -- while creating nodeclass 

Exactly one of the following must be provided:

role

instanceProfile

❌ roleARN is invalid in Auto Mode.

## NodePool Definitions (Karpenter) 
Important: API Version Discovery
CRDs were verified using:  -- run this only if you create nodeclass 
```
kubectl get crd nodepools.karpenter.sh -o jsonpath='{.spec.versions[*].name}'
``` 

Result:
```
v1
```

Therefore:
```
apiVersion: karpenter.sh/v1
``` 

## Validation Commands
```
kubectl get nodeclass
kubectl get nodepool
kubectl get nodes
```

Expected behavior:

Custom NodeClasses show READY: False initially

Custom NodePools show READY: False

No EC2 instances running

This is expected.