# kube-prometheus-stack

* The kube-prometheus-stack is the standard, pre-configured collection of tools for end-to-end Kubernetes cluster monitoring. It combines Prometheus, Grafana, Kube-State-Metrics, and other essential components into a single deployable Helm chart, simplifying the setup process

## Key Components of the Stack: 
The kube-prometheus-stack includes several integrated tools that work together to provide comprehensive observability of your Kubernetes environment: 

* **Prometheus:** The core monitoring and alerting tool. It collects metrics from various endpoints at specified intervals and stores them in a time-series database. It uses PromQL for querying and analysis.

* **Grafana:** A visualization layer that uses Prometheus as a data source to display metrics through powerful, customizable dashboards. The stack provides many ready-to-use, pre-built dashboards for critical Kubernetes metrics. 

* **Kube-State-Metrics (KSM)**: An exporter that listens to the Kubernetes API server and generates metrics about the state of the Kubernetes objects themselves (e.g., number of running pods, deployment status, PVC (Persistent Volume Claims) status). This is distinct from resource usage metrics. 

* **Node Exporter:** An exporter that collects hardware and OS metrics (CPU usage, memory utilization, disk I/O) from the underlying nodes in the cluster.  

* **Alertmanager:** Handles alerts fired by Prometheus rules, grouping and routing notifications to various platforms like Slack, PagerDuty, or email.  

**How They Work Together**
* Exporters (Node Exporter, KSM, cAdvisor) expose metrics endpoints from the cluster nodes, Kubernetes API, and containers.
* Prometheus automatically discovers and scrapes these metrics endpoints, thanks to the configuration managed by the Prometheus Operator and its CRDs.
* Grafana connects to Prometheus to query the stored time-series data and render the pre-configured dashboards, offering a visual overview of cluster health.
* Alertmanager receives alerts from Prometheus when predefined conditions are met and sends out notifications to your teams.  

reference link: https://faun.pub/kube-prometheus-with-terraform-and-helm-24cac706fbdb


# kube-prometheus-stack Deployment on EKS (Auto Mode) using Terraform 

## Overview
* This README documents the end-to-end setup of the kube-prometheus-stack on an Amazon EKS Auto Mode cluster using Terraform and Helm, including:
    * Architecture decisions
    * Terraform module design
    * Helm value customization
    * Storage and scheduling considerations
    * IAM and EBS CSI integration
    * A complete debugging journey, covering all errors encountered, root causes, and fixes

## Prerequisites
Before deploying the kube-prometheus-stack, ensure the following prerequisites are met.
### Infrastructure
* Amazon EKS cluster (Auto Mode enabled)
* Kubernetes version: v1.33.x
* Region: ap-south-2 (or equivalent) 

### Tooling
* Terraform ≥ 1.5
* AWS Provider ≥ 5.x
* Helm Provider
* Kubernetes Provider
* kubectl (configured via kubeconfig)
* Helm CLI (for debugging) 

## Implementation Plan (Step-by-Step)
### Step 1: Terraform Provider Configuration
* Configured the following providers:
    * aws
    * kubernetes
    * helm 
* The Kubernetes and Helm providers authenticate using:
    * aws_eks_cluster
    * aws_eks_cluster_auth 
This ensures Terraform can interact with the cluster API securely.

### Step 2: Folder Structure 
A separate Terraform module was created for kube-prometheus-stack:
```
modules/
└── addons/
    └── kube-prometheus-stack/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        └── values.yaml
```

## Step 3: Namespace Creation
A dedicated namespace was created - `modules/addons/kube-prometheus-stack/main.tf`
```
resource "kubernetes_namespace_v1" "monitoring" {
  count = var.enabled ? 1 : 0
  metadata {
    name = "monitoring"
  }
}
```
## Step 4: Helm Release – kube-prometheus-stack
* The Helm release was deployed using Terraform:
    * Chart: kube-prometheus-stack
    * Repository: prometheus-community
    * Version pinned for stability
    * Values overridden via values.yaml
The Helm release itself is conditionally created using count:
```
resource "helm_release" "kube_prometheus_stack" {
  count     = var.enabled ? 1 : 0
  name      = "kube-prometheus-stack"
  namespace = kubernetes_namespace_v1.monitoring[count.index].metadata[0].name
  values    = [file("${path.module}/values.yaml")]
}
```
## Step 5: values.yaml Customization
* The default Helm values were not used, because they are unsafe for production.
* Customizations include:
    * Resource requests and limits
    * Node scheduling constraints
    * Optional persistence
    * Explicit control over behavior

* Key components customized:
    * Prometheus
    * Alertmanager
    * Grafana 

## step 6: Initial Storage Design (EBS CSI Driver) – Design, Findings & Final Decision
### Initial Design Assumption
During the initial design phase, the kube-prometheus-stack was planned as a stateful observability stack, with persistent storage enabled for the following components:
* Prometheus
    * Time-series database (TSDB) for metrics retention
* Alertmanager
    * Alert state, silences, and notification metadata
* Grafana
    * Dashboards, user preferences, and session data 

* To support this design, the following were implemented:
    * Installed the Amazon EBS CSI Driver as an EKS managed addon
    * Created a dedicated IAM role for the EBS CSI controller using IRSA
    * Attached AmazonEBSCSIDriverPolicy to the IRSA role
    * Configured StorageClasses (gp2 / gp3) for PVC provisioning 

### Issues Observed During Implementation
During deployment and debugging on EKS Auto Mode, the following limitations were identified:
* PersistentVolumeClaims (PVCs) remained in Pending state
* Scheduler errors related to:
    * Topology requirements
    * Accessibility constraints
    * Node lifecycle uncertainty in Auto Mode
* Auto Mode dynamically manages nodes, which conflicts with:
    * AZ-pinned EBS volumes
    * Stateful workload guarantees required by Prometheus and Alertmanager 
This revealed a fundamental incompatibility between EBS-backed stateful workloads and EKS Auto Mode for this use case.

### Final Architectural Decision
After detailed debugging and validation:
* Persistent storage was removed from:
    * prometheus
    * grafana
    * alert manager
* The kube-prometheus-stack was redesigned as a stateless monitoring stack


# Debugging Journey – kube-prometheus-stack on EKS Auto Mode
This section documents the complete debugging journey encountered while deploying kube-prometheus-stack using Terraform and Helm on EKS Auto Mode

## Issue 1: PersistentVolumeClaims (PVCs) stuck in Pending state
### What was the issue?
* Prometheus, Alertmanager, and Grafana pods were stuck in Pending
* Corresponding PVCs remained in Pending state
* No PersistentVolumes (PVs) were getting provisioned

### Why did this happen?
* PVCs referenced a StorageClass (gp2 / gp3) that uses the EBS CSI provisioner
* The EBS CSI Driver was not installed in the cluster
* Without the CSI driver, Kubernetes cannot dynamically provision EBS volumes 

### How we diagnosed it
* Checked PVC events:
```
kubectl describe pvc -n monitoring <pvc-name>
```
* Observed errors indicating:
    * External provisioner ebs.csi.aws.com not responding
* Confirmed no EBS CSI pods were running

### Fix applied

* Installed Amazon EBS CSI Driver as an EKS managed addon using Terraform

## Issue 2: EBS CSI controller pods in CrashLoopBackOff

### What was the issue?
    * EBS CSI controller pods were created
    * Pods entered CrashLoopBackOff state
    * Addon creation kept timing out in Terraform

### Why did this happen?
* The EBS CSI controller requires AWS API access
* In EKS Auto Mode:
    * Node IAM role is not used by CSI controllers
    * CSI controllers must use IRSA
* We initially expected EKS Auto Mode to auto-manage this, which it does not

### How we diagnosed it
* Checked controller logs:
```
kubectl logs -n kube-system <ebs-csi-pod> -c ebs-plugin
```
* Observed errors:
```
failed to refresh cached credentials
no EC2 IMDS role found
DescribeAvailabilityZones failed
```
* This clearly indicated missing IAM credentials

### Fix applied
* Explicitly created:
    * IAM Role for EBS CSI Controller
    * IRSA trust relationship with EKS OIDC provider
* Attached AmazonEBSCSIDriverPolicy

### What we learned
* EKS Auto Mode does NOT auto-configure IRSA for CSI drivers
* CSI controllers always require IRSA, regardless of Auto Mode  

## Issue 3: Helm installation failures due to nodeSelector mismatch 
### What was the issue?
* Helm release failed with:
    * context deadline exceeded
* Pods stuck in Pending
* Scheduler errors:
```
node(s) didn't match Pod's node affinity/selector
```
### Why did this happen?
* kube-prometheus-stack values.yaml contained nodeSelectors:
```
nodeSelector:
  workload: nodepool-cpu-general
```
* Actual Auto Mode nodes had labels:
```
karpenter.sh/nodepool=general-purpose
```

### Fix applied
* Updated values.yaml to use:
```
nodeSelector:
  karpenter.sh/nodepool: general-purpose
```

## Issue 4: StorageClass mismatch (gp3 vs gp2) 
### What was the issue?
* Some PVCs referenced gp3
* Some referenced gp2
* Inconsistent provisioning behavior 

### Why did this happen?
* Initial design used gp3
* Cluster also had legacy gp2
* CSI + Auto Mode behavior was inconsistent across classes 

### How we diagnosed it
* Checked storage classes:
```
kubectl get storageclass
```
* Reviewed PVC definitions from Helm templates

### Fix applied
* Standardized all Helm persistence configs to gp2

## Issue 5: PVC provisioning failed with topology errors 
### What was the issue?
* PVCs kept retrying provisioning
* Events showed:
```
error generating accessibility requirements:
no topology key found for node
```

### Why did this happen?
* EBS volumes are AZ-scoped
* CSI driver needs node topology labels like:
```
topology.ebs.csi.aws.com/zone
```
* Auto Mode nodes are:
    * Dynamically created
    * Abstracted from traditional node group guarantees
* This caused topology resolution failures

### How we diagnosed it
* Checked PVC events:
```
kubectl describe pvc -n monitoring
```
* Inspected node labels:
```
kubectl get nodes --show-labels
```

### Fix applied
* Removed persistence from:
    * Prometheus
    * Alertmanager
    * Grafana
* Treated observability stack as ephemeral 

### What we learned
* Auto Mode is ideal for:
    * Stateless system components
    * Elastic workloads
* Stateful observability requires:
    * Dedicated node groups
    * Or managed services 




