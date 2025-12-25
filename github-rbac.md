* A permission is the ability to perform a specific action. For example, the ability to delete an issue is a permission.

* A role is a set of permissions you can assign to individuals or teams. 

you cannot customize permissions for a collaborator on a repository owned by a personal account. 
In 2025, personal repositories strictly follow a two-tier model: you are either the Owner or a Collaborator 

**How to get Customized Permissions**
* If you need granular control (e.g., giving one person "Read-only" access and another "Admin" access), you must use an Organization account.  

**Roles in an organization**
* Repository-level roles give organization members, outside collaborators and teams of people varying levels of access to repositories

* Team-level roles are roles that give permissions to manage a team. You can give any individual member of a team the team maintainer role, which gives the member a number of administrative permissions over a team 

* Organization-level roles are sets of permissions that can be assigned to individuals or teams to manage an organization and the organization's repositories, teams, and settings ex: owner, member, billing manager, security manager etc

**Repository roles for an organization**
* You can give organization members, outside collaborators, and teams of people different levels of access to repositories owned by an organization by assigning them to roles. Choose the role that best fits each person or team's function in your project without giving people more access to the project than they need.

From least access to most access, the roles for an organization repository are:

* Read: Recommended for non-code contributors who want to view or discuss your project
* Triage: Recommended for contributors who need to proactively manage issues, discussions, and pull requests without write access
* Write: Recommended for contributors who actively push to your project
* Maintain: Recommended for project managers who need to manage the repository without access to sensitive or destructive actions
* Admin: Recommended for people who need full access to the project, including sensitive and destructive actions like managing security or deleting a repositor 


If your organization uses GitHub Enterprise Cloud, you can create custom repository roles. For more information, see Managing custom repository roles for an organization in the GitHub Enterprise Cloud documentation.

**About organization teams**
* Teams are groups of organization members that reflect your company or group's structure with cascading access permissions and mentions. 

* You can use teams to manage access for people in an organization, and for sending notifications. Organization owners and team maintainers can give teams admin, read, or write access to organization repositories 

**Managing team access to an organization repository**
* You can give a team access to a repository, remove a team's access to a repository, or change a team's permission level for a repository. 

* People with admin access to a repository can manage team access to the repository

* You can give a team access to a repository or change a team's level of access to a repository in your repository settings 


https://docs.github.com/en/get-started/learning-about-github/access-permissions-on-github


# GitHub RBAC Design & Implementation

## 1. Prerequisites 
### Organizational Prerequisites 
Before RBAC, ensure the following decisions are finalized: 
* Decide to use a GitHub Organization
* Identify code ownership model

    * Who owns repos?
    * Who approves PRs?

* Identify environments

    * Dev
    * Staging
    * Production

* Identify team structure

    * Developers
    * DevOps
    * Interns
    * Security / Leads

RBAC fails if org structure is unclear.

### GitHub-Level Prerequisites

| Requirement                 | Status                     |
| --------------------------- | -------------------------- |
| GitHub Organization created | Required                   |
| Personal repos migrated     | Recommended                |
| Branch naming strategy      | Required                   |
| CI strategy decided         | Optional (but recommended) |

### Security Prerequisites (Strongly Recommended)

* Enable 2FA for organization
* Restrict who can create repositories
* Decide who can:
    * Delete repos
    * Manage secrets
    * Approve deployments

## RBAC Design 
### Step 1 – Identify Roles in Your Company

| Role Category  | Examples                 |
| -------------- | ------------------------ |
| Platform/Admin | CTO, Tech Lead           |
| DevOps         | Cloud / DevOps Engineers |
| Developers     | Backend, Frontend        |
| Interns        | Trainees                 |


### Step 2 – Design GitHub Teams
Create teams that reflect responsibility

| Team Name         | Members                    | Purpose                               |
| ----------------- | -------------------------- | ------------------------------------- |
| `app-team`        | 5 app devs (incl. interns) | Owns app repos                        |
| `devops-team`     | DevOps lead + members      | Owns infra & helm                     |
| `interns`         | All interns (app + DevOps) | Safety net (optional but recommended) |
| `platform-admins` | Tech lead / CTO            | Admin access                          |

A user can be in multiple teams
Example:
    DevOps intern → app-team + interns 

* App interns belong to both app-team and interns.
* Access is restricted by repository permissions, not team membership alone.
### Step 3 – Repository Classification
Classify the repositories logically
classify which repo belongs to which repo type 

| Repo Type            | Examples          | Ownership        |
| -------------------- | ----------------- | ---------------- |
| Application repos    | backend, frontend | Application team |
| Infrastructure repos | terraform         | DevOps team      |
| Platform repos       | helm-charts       | DevOps team      | 


### Step 4 – Repository-Level RBAC Design

#### Application Repositories (Owned by App Team) 
| Team              | Permission | Why                       |
| ----------------- | ---------- | ------------------------- |
| `app-team`        | Write      | They develop & test       |
| `devops-team`     | Read       | Visibility only           |
| `interns`         | Read       | Prevent accidental damage |
| `platform-admins` | Admin      | Governance                |

* App team owns their code
* DevOps cannot change app logic

#### Infrastructure Repositories (Terraform) 
| Team              | Permission | Why               |
| ----------------- | ---------- | ----------------- |
| `devops-team`     | Maintain   | Infra ownership   |
| `platform-admins` | Admin      | Emergency control |
| `app-team`        | Read       | Infra visibility  |
| `interns`         | Read       | Safe learning     |

* App team should never have write on Terraform
* Even DevOps interns should not have Admin 

#### Helm Charts Repository
| Team              | Permission       | Why                   |
| ----------------- | ---------------- | --------------------- |
| `devops-team`     | Maintain         | Chart ownership       |
| `platform-admins` | Admin            | Control               |
| `app-team`        | Read             | Understand deployment |
| `interns`         | Read             | Learning              |

### Branch Protection Rules
RBAC fails without branch protection. 

* Mandatory Rules for `main / release`
    * Require Pull Request
    * Require 1 approval
    * Disable direct pushes
    * Disable force pushes
    * (Later) Require CI checks

## RBAC Implementation 

### Step 1 – Create GitHub Organization
1. Go to GitHub, click on your profile and then click on organizations
2. Click new Organization. 
3. Choose plan (Free is fine initially)
4. Enter:
    * Organization name (company name)
    * Company email
5. Organization → Settings
5. Enable:
    * 2FA requirement
    * Disable “Members can create repositories” 

### Step 2 – Create Teams 
For each team:
* Go to Organization → Teams 
* Create team (e.g., devops)
* Add members to the respective teams  
* Assign team maintainers (senior members) 

**Teams represent functions, not people** 

### Step 3 – Assign Team Access to Repositories
For each repository: 
* Go to Repo → Settings → Collaborators & teams
* Add a team
* Assign permission:
    * Read / Write / Maintain / Admin
* Remove direct user access

**Always prefer teams over individuals**

### Step 4 – Configure Branch Protection
RBAC without branch protection is dangerous.

* Repo → Settings → Branches → Add rule
* Recommended Rules for main / release
    * Require pull request
    * Require at least 1–2 reviews
    * Require CI checks to pass
    * Disable force push
    * Restrict who can push

### Step 5 - GitHub Actions & RBAC (DevOps Angle)
* when we add CI/CD:
    * Only devops or platform-admins manage:
        * Secrets
        * Environments
* Use environment protection rules
    * Manual approval for production
* Developers deploy to dev
* DevOps approves prod

### Step 6 – Audit & Review
RBAC is not one-time.

| Activity               | Frequency |
| ---------------------- | --------- |
| Access review          | Quarterly |
| Remove inactive users  | Monthly   |
| Team permission review | Quarterly |

**Note:1**
**One-Glance Comparison Table** 

| Action / Capability   | Read | Triage | Write | Maintain | Admin |
| --------------------- | ---- | ------ | ----- | -------- | ----- |
| View code             | ✅    | ✅      | ✅     | ✅        | ✅     |
| Comment               | ✅    | ✅      | ✅     | ✅        | ✅     |
| Create issues         | ❌    | ✅      | ✅     | ✅        | ✅     |
| Label / assign issues | ❌    | ✅      | ✅     | ✅        | ✅     |
| Push code             | ❌    | ❌      | ✅     | ✅        | ✅     |
| Create branches       | ❌    | ❌      | ✅     | ✅        | ✅     |
| Merge PRs             | ❌    | ❌      | ✅     | ✅        | ✅     |
| Manage workflows      | ❌    | ❌      | ❌     | ✅        | ✅     |
| Manage secrets        | ❌    | ❌      | ❌     | ❌        | ✅     |
| Change repo settings  | ❌    | ❌      | ❌     | ❌        | ✅     |
| Delete repo           | ❌    | ❌      | ❌     | ❌        | ✅     |

**Note:2**

with our current setup of personal account, free organisation, team/pro organisation **Custom roles are NOT supported** 

* Custom repository roles are available ONLY if:
    * You are using GitHub Enterprise Cloud
    * You are an Organization Owner
    * You explicitly create custom roles in Org Settings  

**Note:3**
1. if you want a role that can push code but not merge PRs. you can provision **Branch protection** which controls merging, direct pushes, approvals
2. if you want Interns should open PRs but not push. you can give them **READ** role 
