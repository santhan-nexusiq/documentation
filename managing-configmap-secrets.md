# Configuration & Secrets Management – GitOps Approach
## Overview
* This project follows a GitOps-driven configuration and secrets management model using Argo CD, External Secrets Operator, and AWS SSM Parameter Store to ensure secure, scalable, and fully automated deployments in Kubernetes.

* The key principle of this design is a clear separation of concerns:
  * ConfigMaps (non-sensitive configuration) are version-controlled in Git
  * Secrets (sensitive data) are never stored in Git and instead live in AWS SSM Parameter Store
  * ArgoCD continuously reconciles the desired state from Git into the cluster
  * External Secrets Operator (ESO) securely syncs secrets from AWS into Kubernetes at runtime

* This approach eliminates manual kubectl apply steps, reduces configuration drift, and aligns with production-grade GitOps best practices.

## Detailed Production Flow (Step-by-Step)
```
┌────────────────────────────┐
│ 1. Git Commit              │
│----------------------------│
│ - ConfigMap YAML           │
│ - ExternalSecret YAML      │
│ - Application YAML         │
└─────────────┬──────────────┘
              |
              v
┌────────────────────────────┐
│ 2. GitOps Repository       │
│----------------------------│
│ Single source of truth     │
│ (No secrets stored here)   │
└─────────────┬──────────────┘
              |
              v
┌────────────────────────────┐
│ 3. ArgoCD                  │
│----------------------------│
│ - Watches Git repo         │
│ - Detects changes          │
│ - Starts reconciliation    │
└─────────────┬──────────────┘
              |
              v
┌─────────────────────────────────────────┐
│ 4. ArgoCD Sync Phase                     │
│-----------------------------------------│
│ a. Applies ConfigMap manifests           │
│ b. Applies ExternalSecret manifests      │
│ c. Pulls Helm chart from Helm repo       │
│ d. Renders Kubernetes manifests          │
└─────────────┬───────────────────────────┘
              |
              v
┌─────────────────────────────────────────┐
│ 5. Kubernetes Control Plane              │
│-----------------------------------------│
│ - ConfigMaps created immediately         │
│ - ExternalSecret CRs registered          │
└─────────────┬───────────────────────────┘
              |
              v
┌─────────────────────────────────────────┐
│ 6. External Secrets Operator (ESO)       │
│-----------------------------------------│
│ - Watches ExternalSecret resources       │
│ - Reads referenced ClusterSecretStore   │
│ - Uses IRSA for authentication           │
└─────────────┬───────────────────────────┘
              |
              v
┌─────────────────────────────────────────┐
│ 7. AWS SSM Parameter Store               │
│-----------------------------------------│
│ - Encrypted SecureString parameters     │
│ - Hierarchical path per service/env     │
│ - IAM-controlled access                 │
└─────────────┬───────────────────────────┘
              |
              v
┌─────────────────────────────────────────┐
│ 8. Kubernetes Secrets                   │
│-----------------------------------------│
│ - ESO creates native Secret objects     │
│ - Secret names match Helm references    │
└─────────────┬───────────────────────────┘
              |
              v
┌─────────────────────────────────────────┐
│ 9. Application Pods                     │
│-----------------------------------------│
│ - envFrom: configMapRef                 │
│ - envFrom: secretRef                   │
│ - App runs with final config            │
└─────────────────────────────────────────┘
```

## GitOps Repository
```
nexusiq-gitops/
├── bootstrap/
│   ├── argocd/                      # ArgoCD install (one-time)
│   └── external-secrets/            # ESO + ClusterSecretStore
│
├── apps/
│   ├── user-service/
│   │   ├── application.yaml         # ArgoCD Application (LINK)
│   │   ├── configmap.yaml           # ConfigMap (non-sensitive)
│   │   └── externalsecret.yaml      # ExternalSecret (SSM ref)
│   │
│   ├── payment-service/
│   └── order-service/
│
└── README.md
``` 

## Prerequisites
* kubectl
* helm
* aws cli
* Access to EKS + AWS IAM

## STEP 1 – Design SSM Parameter Store Structure

Recommended hierarchy
```
/dev/user-service/DB_PASSWORD
/dev/user-service/JWT_SECRET

/prod/user-service/DB_PASSWORD
/prod/user-service/JWT_SECRET
``` 
* One parameter = one secret
* Type = SecureString
* Encrypted with AWS-managed KMS (free)  

## STEP 2 – Upload Secrets to SSM
* here we have multiple secrets to upload. we can prepare a shell script or terraform modules to do automation(depends on mahesh call) 

## STEP 3 – Install ArgoCD
**namespace**
```
kubectl create namespace argocd 
```
**Install**
```
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
after install login to the argocd and add gitops repo to it 

## STEP 4 – Install External Secrets Operator (ESO) 

```
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace
```

## STEP 5 - Create IAM Role & Policy (IRSA) 
* Create a DEDICATED IAM ROLE for ESO via terraform 
**IAM policy**
```
{
  "Effect": "Allow",
  "Action": [
    "ssm:GetParameter",
    "ssm:GetParametersByPath"
  ],
  "Resource": "arn:aws:ssm:*:*:parameter/prod/*"
}
```
Bind Role to ESO ServiceAccount (IRSA) 
```
nexusiq-gitops/bootstrap/external-secrets/serviceaccount.yaml
```
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets
  namespace: external-secrets
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/eks-external-secrets-role
```
ArgoCD syncs automatically

## STEP 6 – Create ClusterSecretStore
* It defines the connection to AWS SSM Parameter Store
```
nexusiq-gitops/bootstrap/external-secrets/clustersecretstore.yaml
```
```
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-ssm
spec:
  provider:
    aws:
      service: ParameterStore
      region: ap-south-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```
ArgoCD can manage ClusterSecretStore, but it cannot manage it until the required CRDs and controllers exist.so we have to run the command kubectl apply initially 

## STEP 7 – Create ConfigMaps per microservice 
```
nexusiq-gitops/apps/user-service/configmap.yaml
```
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
  namespace: app
data:
  LOG_LEVEL: "info"
  FEATURE_X: "true"
``` 

## STEP 8 – Create ExternalSecret
* Defines which secrets to fetch and how to create a Kubernetes Secret 
```
nexusiq-gitops/apps/user-service/externalsecret.yaml
```

```
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: user-service-secret
  namespace: app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-ssm
    kind: ClusterSecretStore
  target:
    name: user-service-secret     # must match Helm secretRef
  dataFrom:
    - find:
        path: /prod/user-service
```

No kubectl apply
ArgoCD applies this 

## STEP 9 – Create ArgoCD Application 
here we have 8 different applications for each microservice
```
nexusiq-gitops/apps/user-service/application.yaml
``` 

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: user-service
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/org/nexusiq-helm-charts
    targetRevision: main
    path: .
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
THIS FILE LINKS Helm repo ↔ GitOps repo 

## STEP 10 - Final Runtime Flow 

```
Git commit
   ↓
ArgoCD sync
   ├── ConfigMap from Git
   ├── ExternalSecret from Git
   ├── Helm chart from Helm repo
   ↓
ESO pulls secrets from SSM
   ↓
Kubernetes Secret created
   ↓
Pod starts
``` 

**Note:**
## aws ssm parameter store vs aws secrets manager

| Feature            | Secrets Manager   | SSM Parameter Store (Standard) |
| ------------------ | ----------------- | ------------------------------ |
| Cost               | ❌ Paid per secret | ✅ FREE                         |
| Pricing            | $0.40 /secret/month | Free storage
| Encryption         | ✅ Yes             | ✅ Yes                          |
| IAM control        | ✅ Yes(fine-grained) | ✅ Yes                          |
| Auto rotation      | ✅ Built-in        | ❌ Manual                       |
| Audit logs         | ✅ Yes             | ✅ Yes                          |
| Kubernetes support | ✅ Yes             | ✅ Yes                          | 

* we can store 10000 parameters per AWS account 

## Cost Comparison Table

| Secret Manager                         | Pricing Model                        | Monthly Cost (≈120 secrets) | Yearly Cost                | Notes                                |
| -------------------------------------- | ------------------------------------ | --------------------------- | -------------------------- | ------------------------------------ |
| **AWS Secrets Manager**                | $0.40 per secret / month + API calls | **$48 – $50 / month**       | **$576 – $600 / year**     | Cost increases linearly with secrets |
| **AWS SSM Parameter Store (Standard)** | Free up to 10,000 parameters         | **$0 / month**              | **$0 / year**              | No per-secret cost                   |
| **HashiCorp Vault (Open Source)**      | Software free, infra + ops cost      | **$150 – $400 / month**     | **$1,800 – $4,800 / year** | Infra + ops effort                   |
| **HashiCorp Vault (Enterprise)**       | License + infra                      | **$2,000+ / month**         | **$25,000+ / year**        | Large enterprises only               |
