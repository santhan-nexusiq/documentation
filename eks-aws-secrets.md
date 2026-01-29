```
Terraform
 └─ Creates 10 AWS Secrets (containers only)

AWS Secrets Manager
 └─ prod/<microservice>  (JSON with many keys)

External Secrets Operator
 └─ Creates 10 Kubernetes Secrets

Helm (single chart, loop-based)
 └─ Each microservice consumes its own Secret via envFrom

Argo CD
 └─ Syncs Helm + ExternalSecret manifests
```

## 1. TERRAFORM – Create secrets for 10 microservices 

### 1.1 Terraform variables
```
variable "microservices" {
  description = "List of microservices"
  type        = set(string)
  default = [
    "payment-service",
    "user-service",
    "auth-service",
    "course-service",
    "notification-service",
    "exam-service",
    "report-service",
    "media-service",
    "search-service",
    "gateway-service"
  ]
}
```

### 1.2 Create AWS Secrets Manager secrets
```
resource "aws_secretsmanager_secret" "microservices" {
  for_each = var.microservices

  name        = "prod/${each.value}"
  description = "Secrets for ${each.value}"
}
```
✅ Creates 10 secrets
✅ No values
✅ No keys
✅ No Git exposure
✅ No Terraform state conflicts

### 1.3 What exists after Terraform apply
In AWS Secrets Manager:
```
prod/payment-service
prod/user-service
prod/auth-service
prod/course-service
...
```
At this point → secrets are empty

### 1.4 Inject secret values manually
Example: nexusiq/nexeditor
```
{
  "DB_HOST": "rds-endpoint",
  "DB_USER": "payment_user",
  "DB_PASSWORD": "*****",
  "JWT_SECRET": "*****",
  "API_KEY": "*****"
}
```
Repeat per microservice, each with its own keys.

### 1.5 Create IAM Role for Each Microservice (IRSA)
**IAM Policy**
```
{
  "Effect": "Allow",
  "Action": ["secretsmanager:GetSecretValue"],
  "Resource": "arn:aws:secretsmanager:*:*:secret:prod/payment-service*"
}
```


## 2. EXTERNAL SECRETS – One per microservice

### 2.1 Install External Secrets Operator
* Tool: External Secrets Operator

* Installed via Helm by platform team.

* What it does:

    * Authenticates using IRSA
    * Polls AWS Secrets Manager 
    * Creates Kubernetes Secrets dynamically

* These YAMLs live in a separate GitOps repo
* Helm chart does NOT contain these 

### 2.2 ClusterSecretStore 
```
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secret-store
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
```

### 2.3 ExternalSecret pattern (Each per microservide) 
```
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-secrets
  namespace: payment
spec:
  refreshInterval: 5m
  secretStoreRef:
    name: aws-secret-store
    kind: ClusterSecretStore
  target:
    name: payment-secrets
  dataFrom:
    - extract:
        key: nexusiq/nexeditor
```

* Repeat for remaining microservices also  

## 3. HELM CHART

### 3.1 values.yaml
```
services:
  payment:
    enabled: true
    image:
      repository: payment
      tag: v1
    secrets:
      secretName: payment-secrets
```

### 3.2 deployment.yaml
```
envFrom:
  - secretRef:
      name: {{ $svc.secrets.secretName }}
```
**What this does**

* Kubernetes reads all keys from the Secret

* Injects them as environment variables

* Helm does not care how many keys exist 