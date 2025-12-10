# Docker Image Build & Push Workflow (Using Git Commit SHA Tagging)

* This project uses a commit-based image tagging strategy to ensure full traceability and reproducibility of container images.
* Every Docker image built for this application is tagged with the exact Git commit SHA used to generate it.

## How Commit-Based Tagging Works
When building an image, we extract the current Git commit SHA:
```
git rev-parse --verify HEAD
```
This SHA is then used in two ways:

Passed into Docker as a build argument (`COMMIT_SHA`)
→ Makes the SHA available inside the Dockerfile.

Used as the Docker image tag in ECR
→ Example: `123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:<commit-sha>`

## Dockerfile Metadata (OCI Labels)
Dockerfile changes to be done (OCI Labels):
```
ARG COMMIT_SHA=""
 
LABEL org.opencontainers.image.revision=$COMMIT_SHA \
      org.opencontainers.image.source=$REPO_URL \
      org.opencontainers.image.version=$REPO_BRANCH
```
These labels:
* Store the full commit SHA inside the image configuration
* Store the Git repository URL
* Store the branch used to build the image    
This metadata is kept outside the container filesystem, inside the image config JSON, making it immutable unless the image is rebuilt.

## Required Dockerfile Arguments
The Dockerfile should define:
```
ARG COMMIT_SHA=""
ARG REPO_URL=""
ARG REPO_BRANCH="main"
```

## Building & Pushing the Image to AWS ECR
Before building, authenticate to ECR:
```
aws ecr get-login-password --region <AWS_REGION> \
  | docker login --username AWS --password-stdin <AWS_ACCOUNT>.dkr.ecr.<AWS_REGION>.amazonaws.com
```
Then build and push the image using the commit SHA as the tag:
```
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --build-arg COMMIT_SHA="$(git rev-parse --verify HEAD)" \
  --build-arg REPO_URL="https://github.com/your/repo" \
  --label org.opencontainers.image.revision="$(git rev-parse --verify HEAD)" \
  -t <AWS_ACCOUNT>.dkr.ecr.<AWS_REGION>.amazonaws.com/<ECR_REPO>:$(git rev-parse --verify HEAD) \
  --push \
  .
```
Replace placeholders:
* `<AWS_ACCOUNT>` → Your AWS account ID
* `<AWS_REGION>` → e.g., us-east-1
* `<ECR_REPO>` → Name of ECR repository

## Benefits of This Strategy

1. Traceability: Every deployment can be linked to a specific commit.
2. Reproducibility: The same commit always produces the same image.
3. Auditing & Compliance: Metadata stored inside image config ensures permanent history.
4. Consistent CI/CD: No ambiguity about which version is deployed.
5. Debugging Made Easy: Quickly determine which commit a running pod/image belongs to.