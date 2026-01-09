# Automating Sync Between main and devops Branches

##  Overview
* In this, we are implementing an automated mechanism to ensure the devops-dev branch always stays updated with the latest changes from the main branch.

* This removes the need for manual pull/merge operations and ensures that infrastructure scripts, deployment pipelines, and DevOps-specific configurations remain aligned with the application code pushed by developers into main.

* The automation is designed to run through a CI workflow (GitHub Actions) and will be triggered every time a commit is pushed to the main branch. 

## What Automation Are We Building?
1. Detect when developers push new code into the main branch
2. Fetch the latest updates from both main and devops
3. Attempt to merge main → devops
4. Handle merge outcomes automatically:
    If merge succeeds → push updated devops branch
    If merge conflicts → stop the workflow and notify the DevOps team
5. Maintain continuous synchronization between branches

## Options Considered for Automation
We evaluated three different approaches for automating the synchronization between `main` and `devops-dev`:

1. Option A — Automatic Merge & Push (Auto-Sync): 
* CI system i.e., github actionsautomatically merges main into devops and pushes the updated devops branch.If conflicts occur, the workflow fails and notifies the team.

2. Option B — Create/Update a Pull Request (PR-Based Sync)
* Instead of merging automatically, CI creates or updates a PR from main → devops.Developers or DevOps engineers review the PR and merge it manually.

3. Option C — Hybrid Approach (Auto-Merge + Fallback PR)
* The workflow first tries to auto-merge.If merge fails due to conflicts, it automatically creates a PR for manual resolution.

* we opted for option B 

## PR-Based Sync (Create/Update Pull Request Automatically)

**Description:** 
* Option B automates synchronization between main → devops by creating or updating a Pull Request instead of performing automatic merges.  

**Workflow behaviour:**
* Whenever a developer pushes new code to the main branch:
1. GitHub Actions workflow automatically triggers
2. The workflow creates a new Pull Request (PR) from main → devops
3. If a PR already exists, the workflow updates the same PR instead of opening duplicates
4. Team members review the PR and merge it manually 

This approach ensures that updates to devops follow a controlled, review-based workflow, complying with teams that enforce code review, testing, and approval processes before merging.

### High-Level Architecture

```
Developer Push → main
        ↓
GitHub Actions Trigger
        ↓
Check if PR (main → devops-dev) exists
        ↓
If NO → Create PR
If YES → Update existing PR
        ↓
Manual Review & Merge
```

### Prerequisites & Points to Consider:
1. Repository Requirements
* Two branches must exist:
  * main
  * devops-dev
* `main` should be the source of truth
* Branch protection rules recommended on `devops-dev`

2. GitHub Permissions:
* The GitHub Actions workflow requires:
```
permissions:
  contents: write
  pull-requests: write
```

3. Tooling Choice 
* We’ll use GitHub CLI (gh) because:
  * Native to GitHub
  * Simple PR detection & creation
  * Widely accepted in production pipelines

### Implementation Plan (Step-by-Step) 

#### Step 1: Create Workflow File
Create this file in your repo:
```
.github/workflows/sync-main-to-devops-dev.yml
```

#### Step 2: Define Workflow Trigger
```
name: Sync main to devops-dev via PR

on:
  push:
    branches:
      - main
```
* This ensures every push to main triggers the workflow.

#### Step 3: Set Required Permissions
```
permissions:
  contents: read
  pull-requests: write
```

#### Step 4: Define Job & Runner
```
jobs:
  sync-branches:
    runs-on: ubuntu-latest
```

#### step 5: Complete Workflow
```
name: Sync main to devops-dev via PR

on:
  push:
    branches:
      - main

permissions:
  contents: read
  pull-requests: write

jobs:
  sync-branches:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Authenticate GitHub CLI
        run: echo "${{ secrets.GITHUB_TOKEN }}" | gh auth login --with-token

      - name: Check if PR already exists
        id: pr_check
        run: |
          PR_NUMBER=$(gh pr list \
            --base devops-dev \
            --head main \
            --state open \
            --json number \
            --jq '.[0].number')

          echo "PR_NUMBER=$PR_NUMBER" >> $GITHUB_ENV

      - name: Create PR if not exists
        if: env.PR_NUMBER == ''
        run: |
          gh pr create \
            --base devops-dev \
            --head main \
            --title "Sync main into devops-dev" \
            --body "Automated PR to sync latest changes from main into devops-dev."

      - name: Update existing PR
        if: env.PR_NUMBER != ''
        run: |
          echo "PR #$PR_NUMBER already exists and is updated automatically."
```

### Verification & Validation Steps

#### Functional Validation
1. Push a commit to main
2. Go to Actions tab
3. Confirm workflow execution is successful
4. Check Pull Requests
    * PR should exist from main → devops-dev
    * Title should match expected format 

#### Duplicate PR Prevention Test
1. Push another commit to main
2. Confirm:
  * No new PR created
  * Existing PR updated with latest commit




