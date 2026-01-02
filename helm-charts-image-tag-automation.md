# Image Tag Automation in Helm Charts (Git SHA–Based)

## overview
* This section documents the standardized approach for automating Docker image tag updates during Kubernetes deployments using Helm, without modifying Helm chart files (values.yaml) manually or dynamically. 

## Problem Statement
* Each new image push requires updating the image tag in values.yaml before deployment, which:
    * Introduces manual steps
    * Causes configuration drift
    * Pollutes Git history
    * Breaks automation principles

* The solution uses:
    * GitHub Actions for CI/CD
    * Git SHA as the Docker image tag for traceability
    * Helm --set flag to inject the image tag dynamically at deployment time
    * Push trigger to the main branch only for production-grade deployments  

## Trigger Strategy
* Selected Trigger: `push` to `main` branch
```
on:
  push:
    branches:
      - main
```

## Helm Chart Configuration
`values.yaml`
```
image:
  repository: <aws_account_id>.dkr.ecr.ap-south-1.amazonaws.com/my-node-app
  tag: "latest"
```
The tag value acts as a default fallback only and is overridden during deployment.

## GitHub Actions Workflow (CI/CD)
* High-Level Flow
```
Push to main
   ↓
Build Docker image
   ↓
Tag image with Git SHA
   ↓
Push image to Amazon ECR
   ↓
Deploy using Helm with --set image.tag
```

## Workflow Implementation
```
name: Build & Deploy (Git SHA Based)

on:
  push:
    branches:
      - main

env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: my-node-app
  HELM_RELEASE: my-node-app
  HELM_CHART_PATH: ./helm-chart

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout source code
      uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Login to Amazon ECR
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build, Tag & Push Docker Image
      run: |
        IMAGE_TAG=${GITHUB_SHA::7}

        docker build -t $ECR_REPOSITORY:$IMAGE_TAG .
        docker tag $ECR_REPOSITORY:$IMAGE_TAG \
          ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPOSITORY:$IMAGE_TAG
        docker push \
          ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPOSITORY:$IMAGE_TAG

    - name: Deploy to EKS using Helm
      run: |
        helm upgrade --install $HELM_RELEASE $HELM_CHART_PATH \
          --set image.tag=${GITHUB_SHA::7}
```

## Deployment Behavior
* Each deployment uses a unique image version
* The deployed Kubernetes workload references the exact commit that triggered the pipeline
* No Helm chart files are changed during CI/CD execution  