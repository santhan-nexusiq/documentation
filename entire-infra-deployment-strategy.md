# Deployment Strategy for Newly Automated Infrastructure Using Terraform

## 1. Strategy Overview
* The deployment strategy is designed around Terraform-native orchestration, where the entire AWS and Kubernetes infrastructure is provisioned using a single terraform apply, driven by:
    * Modular Terraform design
    * Explicit and implicit dependency resolution 

## 2. Single-Environment Deployment Model
* Currently, the infrastructure targets one environment only.
**Characteristics**
* One terraform.tfvars
* One Terraform state
* One execution flow
* No environment branching or duplication 

Terraform remains future-ready to support multi-env, but the current deployment intentionally keeps complexity minimal.

## 3. Dependency-Driven Deployment

### a. Module-Level Dependencies
* Each module declares dependencies using: `depends_on` where required (especially for Kubernetes and addons) 
Example:
```
depends_on = [module.eks]
```

### b. Provider Readiness Control
* Kubernetes, Helm, and Kubectl providers are initialized only after EKS is available
* Ensures no race conditions when applying Kubernetes manifests or Helm charts 

### c. Deterministic Apply Order
* Terraform automatically executes in the following logical order:
    1. VPC & Networking
    2. EKS Control Plane (AutoMode 1.33)
    3. NodeClass & NodePools
    4. Storage Classes
    5. Cluster Addons
    6. Observability Stack  

## 4. One-Command Deployment Model
**Primary Deployment Commands**
```
terraform inti
terraform plan -out=tfplan
terraform apply "tfplan"
```
* This single terraform apply command:
    * Provisions AWS infrastructure
    * Creates EKS cluster
    * Configures node provisioning (AutoMode)
    * Deploys Kubernetes resources
    * Installs Helm-based addons 

## 5. Validation & Verification Strategy 
* After `terraform apply`, the platform is validated using: 

### 1. Infrastructure Validation (Terraform & AWS Level) 
#### 1.1 Terraform State Validation 
a. Ensure no pending changes
```
terraform plan
```
Expected result
```
No changes. Infrastructure is up-to-date.
```
* If changes appear:
    * Indicates drift, manual changes, or provider mismatch
    * Must be investigated before moving forward

b. Validate Terraform state integrity
```
terraform state list
```
* Confirms:
    * All expected resources are tracked
    * No orphaned or missing resources 

### 2. Cluster Validation (EKS & Kubernetes Layer) 

#### 2.1 Kubernetes API Validation
```
kubectl cluster-info
```
* Confirms:
    * API server reachable
    * Core services running  

#### 2.2 Node & AutoMode Validation
```
kubectl get nodes
```
* Validate:
    * Nodes are Ready
    * Both CPU & GPU nodes appear (if expected)
    * No `NotReady` or `SchedulingDisabled`

```
kubectl describe node <node-name>
```
* Check:
    * Labels applied by NodeClass
    * Taints applied correctly
    * Capacity & allocatable resources

### 3. Add-ons Validation (Controllers & Observability)
#### 3.1 AWS Load Balancer Controller Validation
**a. Pod status**
```
kubectl get pods -n kube-system | grep load-balancer
```
* Expected:
    * Pods in Running state
    * No crash loops 

**b. Logs validation**
```
kubectl logs -n kube-system deploy/aws-load-balancer-controller
```
* Look for:
    * No IAM permission errors
    * No webhook failures
    * Successful reconciliation logs 

#### 3.2 kube-prometheus-stack Validation
**a. Pod health**
```
kubectl get pods -n monitoring
```
* Validate:
    * Prometheus
    * Alertmanager
    * Grafana
    * Node exporter

* All must be Running 

**b. Metrics availability**
```
kubectl top nodes
kubectl top pods
```
* Confirms:
    * Metrics server integrated
    * Prometheus scraping working 

**c. Grafana access**
* Port-forward or Ingress access
* Dashboards loading
* Node & pod metrics visible  

### 4. Application Validation (End-to-End)
#### 4.1 Deployment Validation
```
kubectl get deploy
kubectl get pods
```
* Validate:
    * Desired replicas = available replicas
    * No crash loops
    * No image pull errors

```
kubectl describe pod <pod-name>
```
* Check:
    * Correct node selection
    * Correct resource allocation
    * No scheduling warnings 

#### 4.2 Service & Networking Validation
```
kubectl get svc
kubectl get ingress
```
* Validate:
    * Service endpoints populated
    * Ingress / ALB created 
    * DNS / IP assigned

**Functional traffic test**
```
curl <ALB-DNS>
```
* Confirms:
    * Traffic reaches pod
    * Network policies / SGs correct 

#### 4.3 Observability Validation (App Level) 
* In Grafana:
    * CPU / memory usage visible
    * Pod-level metrics present

## 6. Change Deployment Strategy (Post-Initial Setup)
* All changes follow the same pattern:
    1. Modify Terraform code
    2. Run terraform plan
    3. Review impacted resources
    4. Run terraform apply 

## 7. Rollback Strategy 
### 7.1 Git-Based Rollback (Primary & Safest)
* Every infrastructure change is committed to Git
* A previous commit represents a known-good state
* Rollback = revert to that commit and re-apply 

```
git checkout <last-known-good-commit>
terraform plan
terraform apply
```

### 7.2 Terraform State Rollback (Emergency Recovery) 
* When Is This Needed?
* State rollback is used only when:
    * Terraform state is corrupted
    * Resources were deleted accidentally
    * Manual AWS changes caused severe drift 

**State Rollback Using S3 Versioning** 
**Rollback Flow**
1. Identify last healthy state version in S3
2. Restore that version as the current state 
3. Run:
```
terraform plan
terraform apply
```

