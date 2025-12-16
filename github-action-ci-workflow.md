# GITHUB ACTIONS CI WORKFLOW

## GOAL: 
1. Workflow should trigger automatically
2. Checkout the latest code
3. Build a Docker image
4. Tag the image with the commit ID
5. Push the image to AWS ECR 

## Prerequisites

### AWS ECR Repository
You need an ECR repo already created.
Example:
```
Repository name: 
Region: ap-south-2
```
ECR URI format:
```
<ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/
```

### AWS IAM User
Create an IAM user with programmatic access and attach:
Minimum permissions:
    * AmazonEC2ContainerRegistryFullAccess(or a custom ECR policy) 

You will get:
    `AWS_ACCESS_KEY_ID`
    `AWS_SECRET_ACCESS_KEY` 

## store AWS credentials in Github secrets
```
Github repo
--> settings
--> secrets and variables 
--> actions
--> new repository secret
```
add your aws secret key, access key, aws account id 

## GitHub variables 
add non-secret config as variables: 

| variable           | value                                              |
| ------------------ |:--------------------------------------------------:|
| aws_region         | ap-south-2                                         |
| ecr_repository     | backend #this should match the ecr repository name |

## folder structure 
inside your GITHUB repo: 
```
.github/
    workflows/
        githubactions-ci.yaml
Dockerfile
```
github only detects workflows inside .github/workflows/. 

## complete github actions workflow(yaml) 
first Create this file:.github/workflows/nexsaa-ci.yml 

**FULL WORKFLOW**
```
name: nexsaa-ci

on:
  push:
    branches:
      - main   # change if required

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    env:
      AWS_REGION: ${{ vars.AWS_REGION }}
      ECR_REPOSITORY: ${{ vars.ECR_REPOSITORY }}

    steps:
      # Checkout latest code
      - name: Checkout source code
        uses: actions/checkout@v4

      # Configure AWS credentials
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      # Login to Amazon ECR
      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      # Get short commit SHA (for image tag)
      - name: Get commit SHA
        id: gitsha
        run: echo "sha=${GITHUB_SHA::7}" >> $GITHUB_OUTPUT

      # Build Docker image
      - name: Build Docker image
        run: |
          docker build \
            -t ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${{ steps.gitsha.outputs.sha }} \
            .

      # Push Docker image to ECR
      - name: Push Docker image to ECR
        run: |
          docker push \
            ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${{ steps.gitsha.outputs.sha }}
```

## Verification Steps: 
1. GitHub
    * Go to Actions tab
    * Workflow should be green
2. AWS ECR
    * Open ECR repo
    * You should see image with tag like: <git commit id>
3. Audit Check
    * Image tag = Git commit


