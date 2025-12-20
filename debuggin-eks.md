# Debugging & Key Fixes – EKS Auto Mode NodeClass & NodePool Setup

**Overview**

* While setting up NodeClass and NodePool for EKS Auto Mode (v1.33) using Terraform and Karpenter, multiple issues were encountered across:

    * Terraform module structure

    * Provider selection

    * Kubernetes CRD handling

    * API version mismatches

    * Schema validation

    * IAM role usage

    * YAML structure requirements

## Issue 1: Terraform Plan Did Not Show NodeClass / NodePool Resources
**Error / Symptom:**

* Running terraform plan showed only VPC and EKS resources, but no NodeClass or NodePool resources, even though the Terraform files existed.

**Root Cause:**

* NodeClass and NodePool Terraform files were placed outside the root Terraform module

* Terraform only evaluates .tf files that belong to the current module

* Files located in unrelated folders are ignored unless explicitly referenced as modules

**Fix (Step-by-Step):**

* Moved all NodeClass and NodePool Terraform files into the root Terraform directory

* Ensured all files share the same Terraform state

* Re-ran:
```
terraform init
terraform plan
```
**Key Learning:**

Terraform does not automatically discover .tf files outside the active module.
All resources that must be planned together must live in the same module or be explicitly referenced.

## Issue 2: cannot create REST client: no client config
**Error Message**
```
Error: Failed to construct REST client
cannot create REST client: no client config
```
**Root Cause:**

* Terraform was using a Kubernetes provider configuration that did not have:

    * A valid kubeconfig

    * Or valid EKS authentication context

* Terraform could not authenticate to the Kubernetes API server

**Fix (Step-by-Step):**
1. Verified cluster access manually:
```
kubectl get nodes
```

2. Confirmed kubeconfig was valid

3. Updated provider configuration to explicitly reference kubeconfig:
```
provider "kubectl" {
  config_path      = pathexpand("~/.kube/config")
  load_config_file = true
}
```

4. Re-ran:
```
terraform init -reconfigure
terraform plan
```
**Key Learning:**

Terraform Kubernetes operations require explicit, working cluster authentication, even if kubectl works locally.

## Issue 3: Using the Wrong Terraform Provider for CRDs
**Error / Symptom:**

* Terraform failed with schema validation errors such as:
```
Attribute not found in schema
Manifest configuration incompatible with resource schema
```
**Root Cause:**

* Initially used kubernetes_manifest (Kubernetes provider)

* This provider strictly validates schemas

* NodeClass and NodePool are Custom Resource Definitions (CRDs)

* Their schemas are not known to the Kubernetes provider

**Fix (Step-by-Step):**
1. Identified that NodeClass and NodePool are CRDs
2. switched from:
```
kubernetes_manifest
```
to:
```
kubectl_manifest
```
3. Used raw YAML via yaml_body

4. Re-ran Terraform plan and apply

**Key Learning:**

CRDs should always be managed using kubectl_manifest, not kubernetes_manifest, to avoid schema enforcement issues. 

## Issue 4: Invalid API Version for NodePool
**Error Message:**
```
resource [eks.amazonaws.com/v1/NodePool] isn't valid for cluster
```
and later:
```
resource [karpenter.sh/v1beta1/NodePool] isn't valid for cluster
```
**Root Cause**

* Incorrect API group and version were used for NodePool

* NodePool does not belong to eks.amazonaws.com

* The cluster did not support `v1beta1`

**Debugging Step**

Verified the CRD versions installed in the cluster:
```
kubectl get crd nodepools.karpenter.sh -o jsonpath='{.spec.versions[*].name}'
```
Output
```
v1
```

**Fix (Step-by-Step):**
1. Updated NodePool manifests to:
```
apiVersion: karpenter.sh/v1
kind: NodePool
```
2. Removed all incorrect API versions

3. Re-applied Terraform

**Key Learning**

CRD existence does not imply API version compatibility.
Always inspect .spec.versions of the CRD before writing manifests.

## Issue 5: NodeClass Validation Errors – IAM Role

**Error Message:**
```
spec: Invalid value: exactly one of role OR instanceProfile must be provided
```

**Root Cause:**

* Used roleARN instead of the expected role field

* EKS Auto Mode NodeClass requires exactly one of:

    * role

    * instanceProfile

**Fix (Step-by-Step):**

1. Replaced:
```
roleARN: <arn>
```
with:
```
role: <arn>
```
2. Ensured only one field was defined

3. Re-applied Terraform

**Key Learning:**

EKS Auto Mode NodeClass has strict validation rules.
Field names must match the CRD spec exactly.

## Issue 6: Selector Structure Errors in NodeClass

Error Message
```
spec.subnetSelectorTerms must be of type array
spec.securityGroupSelectorTerms must be of type array
```
**Root Cause:**

* Used single objects instead of arrays

* NodeClass expects selector terms as lists, not maps

**Fix (Step-by-Step):**

Replaced:
```
subnetSelector:
  key: value
```
With:
```
subnetSelectorTerms:
  - tags:
      key: value
```
And similarly for securityGroupSelectorTerms.

**Key Learning:**

Even when selecting a single subnet or security group, CRDs often require list structures.

## Issue 7: YAML Parsing Errors During Terraform Plan

**Error Message:**
```
yaml: did not find expected key
```

**Root Cause:**

* Incorrect indentation inside yaml_body

* Mixed tabs and spaces

* Misaligned blocks inside Terraform heredoc

**Fix (Step-by-Step):**

* Validated YAML indentation carefully

* Ensured consistent two-space indentation

* Removed inline comments that broke YAML structure

* Re-ran:
```
terraform plan
```

**Key Learning:**

When using yaml_body, Terraform does not validate YAML syntax — Kubernetes does.
YAML formatting must be exact.

## Issue 8: NodePools Failed Even After NodeClass Creation

**Error Message:**
```
resource [karpenter.sh/v1/NodePool] isn't valid for cluster
```

**Root Cause:**

NodeClass creation succeeded

NodePool failed due to incorrect API version

Fix required verifying CRD versions before writing manifests 

**Fix:**

* Same as Issue 4 (CRD version validation)  


