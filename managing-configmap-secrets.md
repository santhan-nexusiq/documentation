GitOps Repository --
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

