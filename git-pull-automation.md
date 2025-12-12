# Automating Sync Between main and devops Branches
##  Overview
In this project, we are implementing an automated mechanism to ensure the devops branch always stays updated with the latest changes from the main branch.
This removes the need for manual pull/merge operations and ensures that infrastructure scripts, deployment pipelines, and DevOps-specific configurations remain aligned with the application code pushed by developers into main.

The automation is designed to run through a CI workflow (GitHub Actions) and will be triggered every time a commit is pushed to the main branch.

## What Automation Are We Building?
1. Detect when developers push new code into the main branch
2. Fetch the latest updates from both main and devops
3. Attempt to merge main → devops
4. Handle merge outcomes automatically:
    If merge succeeds → push updated devops branch
    If merge conflicts → stop the workflow and notify the DevOps team
5. Maintain continuous synchronization between branches

## Options Considered for Automation
We evaluated three different approaches for automating the synchronization between `main` and `devops`:
1. Option A — Automatic Merge & Push (Auto-Sync): 
* CI system i.e., github actionsautomatically merges main into devops and pushes the updated devops branch.If conflicts occur, the workflow fails and notifies the team.
2. Option B — Create/Update a Pull Request (PR-Based Sync)
* Instead of merging automatically, CI creates or updates a PR from main → devops.Developers or DevOps engineers review the PR and merge it manually.
3. Option C — Hybrid Approach (Auto-Merge + Fallback PR)
* The workflow first tries to auto-merge.If merge fails due to conflicts, it automatically creates a PR for manual resolution.

### Option A — Automatic Merge & Push (Auto-Sync) 
Whenever a developer pushes new code to main:
1. GitHub Actions workflow triggers automatically

2. The workflow fetches the latest main and devops

3. It attempts to merge main → devops

4. If merge is clean:

    * The workflow pushes the updated devops branch to the repository

5. If merge conflicts occur:

    * The workflow stops

    * No push is done

    * A notification can be sent to DevOps team

**Prerequisites & Points to Consider** 
1. Branch Permissions: 
    * The devops branch must allow direct pushes from CI.
    * If branch protection requires PR reviews → Option A will not work.
2. GitHub Workflow Permissions:
Workflow requires:
```
permissions:
  contents: write
```
This lets GitHub Actions push code to the repo.
3. Avoiding Infinite Loops:
    * Workflow must trigger only on pushes to main
    * Not on pushes to devops, otherwise you get endless merging back and forth.
4. Notification Strategy:
In case of failure, the team should be informed via:
    * Microsoft Teams webhook
    * Slack webhook
    * Email

**Detailed Implementation Plan:**
Create the workflow file:
`.github/workflows/sync-main-to-devops-auto-merge.yml`

```
name: Auto Sync main → devops (Auto-Merge)

on:
  push:
    branches:
      - main

permissions:
  contents: write
  pull-requests: read

jobs:
  sync:
    runs-on: ubuntu-latest
    env:
      MAIN_BRANCH: main
      DEVOPS_BRANCH: devops

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: true

      - name: Configure Git
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

      - name: Fetch branches
        run: |
          git fetch origin ${MAIN_BRANCH}:refs/remotes/origin/${MAIN_BRANCH}
          git fetch origin ${DEVOPS_BRANCH}:refs/remotes/origin/${DEVOPS_BRANCH} || true

      # Optional: Pre-merge validation
      - name: Validate main branch (tests/lint/terraform/etc)
        run: |
          echo "Add test commands here, e.g. npm test, terraform validate, helm lint"
        continue-on-error: false

      - name: Checkout devops branch
        run: |
          if git show-ref --verify --quiet refs/heads/${DEVOPS_BRANCH}; then
            git checkout ${DEVOPS_BRANCH}
          else
            git checkout -b ${DEVOPS_BRANCH} origin/${DEVOPS_BRANCH} || git checkout -b ${DEVOPS_BRANCH}
          fi

      - name: Merge main into devops
        id: merge_step
        run: |
          if git merge --no-edit origin/${MAIN_BRANCH}; then
            echo "MERGE_STATUS=success" >> $GITHUB_OUTPUT
          else
            echo "MERGE_STATUS=conflict" >> $GITHUB_OUTPUT
            git merge --abort || true
            exit 1
          fi

      # Optional: Run tests after merge
      - name: Post-merge validation
        if: steps.merge_step.outputs.MERGE_STATUS == 'success'
        run: |
          echo "Run tests after merge here, e.g. terraform plan, npm test"
      
      - name: Push updates to devops
        if: steps.merge_step.outputs.MERGE_STATUS == 'success'
        run: |
          git push origin HEAD:${DEVOPS_BRANCH}

      - name: Notify on failure
        if: failure()
        run: |
          echo "Merge failed — notify team (Slack/email/etc)"
          # Example Slack notification using webhook:
          # curl -X POST -H 'Content-type: application/json' \
          # --data '{"text": "❌ Auto-sync failed for main → devops. Manual conflict resolution required."}' \
          # ${{ secrets.SLACK_WEBHOOK }}
```

**Verification Steps / Commands:**
After enabling the workflow, perform the following checks.
1. Validate Workflow Trigger: 
Push a test commit to `main`:
```
git checkout main
echo "test" >> testfile.txt
git add testfile.txt
git commit -m "Test sync"
git push origin main
```
Now go to:

GitHub → Repository → Actions → Auto Sync main → devops

Check:
    * Workflow triggered
    * Merge step executed
    * Push step executed

**Verify devops branch updated:** 
Run: 
```
git fetch origin
git log origin/devops --oneline -n 5
```
You should see a merge commit.

**Verify Conflict Handling:**

Create intentional conflicting commit on main:
1. Modify the same line in both main and devops
2. Push to main
3. The workflow should:
    * Fail
    * Not push anything
    * Send notification (if configured) 
    * Log merge conflict in Actions log

**Verify Notification:**
Check Slack / Email / GitHub notifications based on your configuration.

## Advantages & Disadvantages
**Advantages:**
1. No manual merging; devops branch stays always updated.
2. Changes flow instantly as soon as developers push to main.
3. Merge conflicts are rare in many cases, so automation works smoothly.
4. CI/CD config, Helm charts, Kubernetes manifests often must track application changes closely.
**Disadvantages:**
1. Requires push permissions on devops: If devops branch is protected → this approach fails.
2. Merge conflicts break automation: Workflow halts until a human resolves the conflict manually.
3. Sensitive for production pipelines

### PR-Based Sync (Create/Update Pull Request Automatically)
1. Description of Option B: 
Option B automates synchronization between main → devops by creating or updating a Pull Request instead of performing automatic merges.
**Workflow behavior:**
Whenever a developer pushes new code to the main branch:
1. GitHub Actions workflow automatically triggers
2. The workflow creates a new Pull Request (PR) from main → devops
3. If a PR already exists, the workflow updates the same PR instead of opening duplicates
4. Team members review the PR and merge it manually

This approach ensures that updates to devops follow a controlled, review-based workflow, complying with teams that enforce code review, testing, and approval processes before merging.

**Why this option exists:**
* Many organizations protect branches like devops from direct pushes
* DevOps automation often requires manual oversight before propagation
* Ensures no accidental breakage or unexpected merge happens automatically

2. Prerequisites & Points to Consider:
1. Branch Protection Policies:
If devops branch has:
    * Required reviewers
    * Required status checks
    * Push restrictions
…this option works perfectly, because the workflow does not push directly to devops.
2. GitHub Permissions:
The workflow requires:
```
permissions:
  contents: write
  pull-requests: write
```
3. Avoiding Infinite Loops
```
on:
  push:
    branches:
      - main
```
NOT on devops branch updates.
4. Conflict Handling
Conflicts will appear in the Pull Request view.
GitHub will mark the PR as:
    **This branch has conflicts that must be resolved**

3. Detailed Implementation Plan:
This workflow uses the stable and widely-used GitHub Action:
```
peter-evans/create-pull-request
```
It creates or updates a PR automatically.

Create the file:
`.github/workflows/pr-sync-main-to-devops.yml`
```
name: PR Sync main → devops

on:
  push:
    branches:
      - main

permissions:
  contents: write
  pull-requests: write

jobs:
  sync:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Create or update PR from main to devops
        uses: peter-evans/create-pull-request@v6
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          base: devops
          head: main
          branch: auto-sync/main-to-devops
          title: "Sync main → devops"
          body: |
            This automated Pull Request brings the latest changes from **main** into **devops**.
            Please review and merge when ready.
          labels: "auto-sync"
          delete-branch: true
```
4. Verification Steps:
1. 1 Test the Workflow Trigger:
Push a commit to `main`:
```
git checkout main
echo "test sync" >> sync.txt
git add sync.txt
git commit -m "Trigger PR sync"
git push origin main
```
Expected:
* Workflow runs
* A PR named “Sync main → devops” appears

5. Advantages & Disadvantages:
**Advantages:**
1. Works with protected branches: No direct push → safe for enterprise setups.
2. Human review before merge: Nothing updates devops without approval.
3. Audit-friendly: Every sync is recorded as a Pull Request.
**Disadvantages:**
1. Requires manual merge
2. Slower propagation


### Option C — Hybrid Sync (Auto-Merge + PR Fallback)
1. Description of Option C
Option C combines the benefits of Option A (Auto-merge) and Option B (PR-based workflow) into a single, intelligent synchronization system.
**Workflow behavior:**
Whenever a developer pushes new changes to the main branch:
1. Workflow triggers → attempts automatic merge of main → devops
2. If merge succeeds:
    * (Optional) Run tests/lint/unit checks
    * Push merged changes to devops automatically
    * No human intervention required
3. If merge fails due to merge conflicts or errors:
    * The workflow does NOT push anything
    * The workflow automatically opens or updates a Pull Request from main → devops
    * Reviewers manually resolve conflicts and merge PR
    * Notifications (Slack/email) can be sent
**Why this hybrid approach is useful:**
* Ensures maximum automation when possible
* Ensures manual control when necessary
* Avoids dangerous automatic conflict resolution
* Reduces the number of PRs when no conflicts exist
* Fits into both fast-paced development and compliance-heavy organizations

This option is the most balanced and often the best choice for modern DevOps teams.

2. Prerequisites & Points to Consider: 
1. Branch Protection Policies
This option works if:
    * devops branch allows automatic push OR
    * Fallback PR is allowed to update devops
If devops does not allow direct pushes → merge attempt will fail → workflow will automatically create PR.
2. GitHub Permissions
Workflow requires:
```
permissions:
  contents: write
  pull-requests: write
```
3. Avoiding Infinite Loops
Workflow triggers only on:
```
on:
  push:
    branches:
      - main
```
PR merges into devops must not re-trigger infinite sync loops.

3. Detailed Implementation Plan:
Below is a fully working GitHub Actions workflow implementing the hybrid system.
Create file:
`.github/workflows/hybrid-sync-main-to-devops.yml`
```
name: Hybrid Sync main → devops (Auto-Merge + PR Fallback)

on:
  push:
    branches:
      - main

permissions:
  contents: write
  pull-requests: write

jobs:
  sync:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure Git user
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

      - name: Fetch branches
        run: |
          git fetch origin main:refs/remotes/origin/main
          git fetch origin devops:refs/remotes/origin/devops || true

      - name: Checkout devops branch
        run: |
          if git show-ref --verify --quiet refs/heads/devops; then
            git checkout devops
          else
            git checkout -b devops origin/devops || git checkout -b devops
          fi

      - name: Attempt auto-merge main → devops
        id: auto_merge
        run: |
          set +e
          git merge --no-edit origin/main
          MERGE_EXIT_CODE=$?
          
          if [ $MERGE_EXIT_CODE -eq 0 ]; then
            echo "MERGE_STATUS=success" >> $GITHUB_OUTPUT
            exit 0
          else
            echo "MERGE_STATUS=conflict" >> $GITHUB_OUTPUT
            git merge --abort || true
            exit 0
          fi

      - name: Push auto-merged devops branch
        if: steps.auto_merge.outputs.MERGE_STATUS == 'success'
        run: |
          git push origin devops

      - name: Create or Update PR on conflict
        if: steps.auto_merge.outputs.MERGE_STATUS == 'conflict'
        uses: peter-evans/create-pull-request@v6
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          base: devops
          head: main
          branch: auto-sync/main-to-devops
          title: "Sync main → devops (Conflict Detected)"
          body: |
            Automatic merge of **main → devops** failed due to merge conflicts.
            This PR was created automatically.
            Please resolve conflicts and merge manually.
          labels: auto-sync, conflict
          delete-branch: true

      - name: Notify about merge conflict
        if: steps.auto_merge.outputs.MERGE_STATUS == 'conflict'
        run: |
          echo "Conflict detected — a PR has been created for manual resolution."
          # Optional: Add Slack or Email notification
```

4. Verification Steps / Commands:
**Test Auto-Merge Path**
1. Add a harmless commit to main:\
2. Check GitHub Actions:
    * Workflow should succeed
    * devops branch should have a merge commit
3. Verify:
```
git fetch origin
git log origin/devops --oneline -n 5
```
**Test Conflict Path:**
1. Create conflict:
    * Modify same line in main and devops
```
git checkout devops
echo "devops version" > conflict.txt
git commit -am "Devops conflicting change"
git push origin devops
```
```
git checkout main
echo "main version" > conflict.txt
git commit -am "Main conflicting change"
git push origin main
```
2. Action should:
    * Detect conflict
    * Abort merge
    * Create PR 

5. Advantages & Disadvantages:
**Advantages:**
1. Best of both worlds: Automatic merge when possible; PR-based fallback only when needed.
2. Zero manual effort in normal scenarios: If main changes often but conflicts are rare → sync is fully automated.
3. Safe for protected branches: If devops push is blocked → PR fallback still works.
4. Conflict visibility: Clear PR showing conflicts and required human intervention.

**Disadvantages** 
1. Slightly more complex workflow
2. Requires both write & PR permissions
3. 

