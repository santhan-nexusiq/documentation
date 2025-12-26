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


# GITHUB PRICING AND PLANS

* Currently, there are 4 different GitHub plans available:
    * GitHub Free
    * GitHub Pro
    * GitHub Team
    * GitHub Enterprise 

## GitHub Free:
* in the github free plan, you have two different types of plans
     1. github free for personal account
     2. github free for organizational account 

**github free for personal account:**
* With GitHub Free, your personal account includes:

    * GitHub Community Support
    * Dependabot alerts
    * Deployment protection rules for public repositories
    * Two-factor authentication enforcement
    * 500 MB GitHub Packages storage
    * 120 GitHub Codespaces core hours per month
    * 15 GB GitHub Codespaces storage per month
    * GitHub Actions features:
        * 2,000 minutes per month
        * Deployment protection rules for public repositories
    * GitHub Pages in public repositories

**github free for organizational account:**
* With GitHub Free for organizations, you can work with unlimited collaborators on unlimited public repositories with a full feature set, or unlimited private repositories with a limited feature set.

* In addition to the features available with GitHub Free for personal accounts, GitHub Free for organizations includes:
    * GitHub Community Support
    * Team access controls for managing groups
    * 2,000 GitHub Actions minutes per month  

**GitHub Team:**
* In addition to the features available with GitHub Free for organizations, GitHub Team includes:
    * GitHub Support via email
    * 3,000 GitHub Actions minutes per month
    * 2 GB GitHub Packages storage
    * The option to purchase GitHub Advanced Security products:
        * GitHub Code Security
        * GitHub Secret Protection
    * Advanced tools and insights in private repositories:
        * Required pull request reviewers
        * Multiple pull request reviewers
        * Team pull request reviewers
        * Protected branches
        * Code owners
        * Scheduled reminders
        * GitHub Pages
        * Wikis
        * Security overview
        * Repository insights graphs: Pulse, contributors, traffic, commits, code frequency, network, and forks
        * The option to enable or disable GitHub Codespaces
            * Organization owners can choose to enable or disable GitHub Codespaces for the organization's private repositories, and can pay for the usage of members and collaborators. 
GitHub bills for GitHub Team on a per-user basis. For more information, see About per-user pricing.

GitHub Actions usage is free for standard GitHub-hosted runners in public repositories, and for self-hosted runners. See Choosing the runner for a job. For private repositories, each GitHub account receives a quota of free minutes and storage for use with GitHub-hosted runners, depending on the account's plan. Any usage beyond the included amounts is billed to your account. 

**GitHub Enterprise:**
* GitHub Enterprise includes two deployment options: GitHub Enterprise Cloud, which is hosted by GitHub in the cloud, and GitHub Enterprise Server, which is self-hosted

* In addition to the features available with GitHub Team, GitHub Enterprise includes:

    * GitHub Enterprise Support
    * Additional security, compliance, and deployment controls
    * Authentication with SAML single sign-on
    * Access provisioning with SAML or SCIM
    * Deployment protection rules with GitHub Actions for private or internal repositories
    * GitHub Connect
    * Additional features such as internal repositories and repository rules.

* GitHub Enterprise Cloud specifically includes:
    * 50,000 GitHub Actions minutes per month
    * 50 GB GitHub Packages storage
    * A service level agreement for 99.9% monthly uptime
    * The option to host your company's data in a specific region, on a unique subdomain 

## How GitHub Pricing Is Calculated
* GitHub pricing has two independent parts:
1. License cost (fixed)
* This is the $4 for teams and $21 for enterprise per user per month.  
2. Usage-based cost (variable)
* This applies to:
    * GitHub Actions (minutes)
    * GitHub Packages (storage & bandwidth)
    * Codespaces (compute & storage) 

### GitHub Team – $4/user/month (How It’s Calculated)
License Cost
```
$4 × number_of_active_users × billing_period
```
**Example:**
10 developers
Monthly billing = $4 × 10 = $40/month 

#### Included Usage - github teams

| Resource                       | Included                           |
| ------------------------------ | ---------------------------------- |
| GitHub Actions (private repos) | **3,000 minutes/month (org-wide)** |
| GitHub Actions (public repos)  | Unlimited                          |
| Self-hosted runners            | Unlimited                          |
| GitHub Packages storage        | **2 GB**                           |
| Codespaces                     | Pay-as-you-go                      |
 

### What If You Use Excessively? (Team)
**GitHub Actions Overuse**
* If you exceed 3,000 minutes/month on GitHub-hosted runners:
    * GitHub does NOT stop workflows immediately
    * It starts billing per extra minute 
Typical rate (Linux runners):
```
≈ $0.008 per minute
```
**Example:**
* Used 5,000 minutes
* Free: 3,000
* Billable: 2,000
* `2,000 × $0.008 ≈ $16 extra` 

### GitHub Enterprise – $21/user/month
License Cost
```
$21 × number_of_users × billing_period
```
**Example:**
* 100 developers
* Monthly billing
* $21 × 100 = $2,100/month 
#### ncluded Usage - github enterprise 

| Resource                       | Included (Enterprise Cloud) |
| ------------------------------ | --------------------------- |
| GitHub Actions (private repos) | **50,000 minutes/month**    |
| GitHub Packages storage        | **50 GB**                   |
| Public repo Actions            | Unlimited                   |
| Self-hosted runners            | Unlimited                   |
| Codespaces                     | Pay-as-you-go               |

#### What If You Use Excessively? (Enterprise)
* Same principle as teams 
* github Actions Overuse 
Typical rate (Linux runners): 
```
≈ $0.008 per minute
```


## Detailed comparison between GitHub Free (Organization) and GitHub Team 

### Pricing & Billing Model

| Plan                  | Pricing Model       | Cost                |
| --------------------- | ------------------- | ------------------- |
| **GitHub Free (Org)** | Free                | **$0**              |
| **GitHub Team**       | Per-user, per-month | **≈ $4/user/month** |

Note: 
GitHub Team is billed per active member, making it suitable for small–medium teams.

### Detailed Feature Comparison

| Feature Category                 | GitHub Free – Organization           | GitHub Team                          |
| -------------------------------- | ------------------------------------ | ------------------------------------ |
| **Target Audience**              | Small teams, interns, early startups | Growing teams, production workloads  |
| **Repositories**                 | Unlimited public & private           | Unlimited public & private           |
| **Collaborators**                | Unlimited                            | Unlimited                            |
| **Team-Based RBAC**              | ✅ Yes                               | ✅ Yes                                |
| **Repository Roles**             | Read, Triage, Write, Maintain, Admin | Read, Triage, Write, Maintain, Admin |
| **Org-level 2FA Enforcement**    | ✅ Yes                               | ✅ Yes                                |
| **Support**                      | Community support                    | **Email support**                    |
| **GitHub Actions Minutes**       | 2,000 min/month (org-wide)           | **3,000 min/month**                  |
| **Actions on Public Repos**      | Free                                 | Free                                 |
| **Self-hosted Runners**          | Free                                 | Free                                 |
| **GitHub Packages Storage**      | 500 MB                               | **2 GB**                             |
| **Codespaces Access**            | Enabled by default                   | Org can enable/disable               |
| **Codespaces Billing Control**   | ❌ No                                | ✅ Yes                                |
| **Deployment Protection Rules**  | Public repos only                    | Public repos only                    |
| **Protected Branches**           | ❌ Limited                           | ✅ Full support                       |
| **Required PR Reviewers**        | ❌ No                                | ✅ Yes                                |
| **Multiple PR Reviewers**        | ❌ No                                | ✅ Yes                                |
| **Team PR Reviewers**            | ❌ No                                | ✅ Yes                                |
| **Code Owners**                  | ❌ No                                | ✅ Yes                                |
| **Scheduled Reminders**          | ❌ No                                | ✅ Yes                                |
| **Wikis**                        | ❌ No                                | ✅ Yes                                |
| **Security Overview Dashboard**  | ❌ No                                | ✅ Yes                                |
| **Repository Insights (Graphs)** | ❌ No                                | ✅ Yes                                |
| **Advanced Security (Add-on)**   | ❌ Not available                     | ✅ Optional add-on                    |
| **Compliance Controls**          | ❌ No                                | ❌ No (Enterprise only)               |

**Security Capabilities (Very Important Difference)** 

| Security Feature           | Free (Org)   | Team        |
| -------------------------- | ----------   | ------------|
| Dependabot Alerts          | ✅ Yes      | ✅ Yes      |
| Security Overview          | ❌ No       | ✅ Yes      |
| Code Security (Add-on)     | ❌ No       | ✅ Yes      |
| Secret Protection (Add-on) | ❌ No       | ✅ Yes      |
| Branch Protection Rules    | ❌ Minimal  | ✅ Advanced |

note: 
* GitHub Team does not include Advanced Security by default, but allows you to buy it, which Free Org cannot.


## Deatailed Comparison between GitHub Team and GitHub Enterprise

### Pricing & Billing Model

| Plan                  | Pricing Model       | Approx Cost     |
| --------------------- | ------------------- | --------------- |
| **GitHub Team**       | Per-user, per-month | ~$4/user/month  |
| **GitHub Enterprise** | Per-user, per-month | ~$21/user/month |


### Detailed Feature Comparison

| Feature Category                                | GitHub Team                | GitHub Enterprise                    |
| ----------------------------------------------- | -------------------------- | ------------------------------------ |
| **Target Audience**                             | Growing engineering teams  | Large orgs, regulated enterprises    |
| **Repositories**                                | Unlimited public & private | Unlimited public, private & internal |
| **Internal Repositories**                       | ❌ Not available            | ✅ Available                          |
| **Team-Based RBAC**                             | ✅ Yes                      | ✅ Yes (advanced)                     |
| **Repository Rules**                            | ❌ Limited                  | ✅ Advanced rulesets                  |
| **Org-level 2FA Enforcement**                   | ✅ Yes                      | ✅ Yes                                |
| **Authentication (SSO)**                        | ❌ No                       | ✅ SAML SSO                           |
| **User Provisioning**                           | ❌ Manual                   | ✅ SCIM / SAML provisioning           |
| **Support Level**                               | Email support              | **Enterprise support**               |
| **GitHub Actions Minutes**                      | 3,000 min/month            | **50,000 min/month (Cloud)**         |
| **Deployment Protection Rules (Private Repos)** | ❌ No                       | ✅ Yes                                |
| **GitHub Packages Storage**                     | 2 GB                       | **50 GB**                            |
| **Codespaces Control**                          | Enable/disable             | Advanced org-wide policies           |
| **Audit Logs**                                  | ❌ Limited                  | ✅ Full audit logs                    |
| **Compliance Controls**                         | ❌ No                       | ✅ Yes                                |
| **GitHub Connect**                              | ❌ No                       | ✅ Yes                                |
| **SLA (Uptime Guarantee)**                      | ❌ No                       | ✅ 99.9% SLA                          |
| **Data Residency Options**                      | ❌ No                       | ✅ Yes (Cloud)                        |
| **Self-hosted Option**                          | ❌ No                       | ✅ Enterprise Server                  |


### Security & Compliance

| Security Feature     | GitHub Team     | GitHub Enterprise       |
| -------------------- | --------------- | --------------------    |
| Dependabot Alerts    | ✅ Yes          | ✅ Yes                 |
| Advanced Security    | Optional add-on | Optional / bundled      |
| Secret Scanning      | Optional        | Enterprise-ready        |
| Code Scanning        | Optional        | Enterprise-ready        |
| Security Overview    | ✅ Yes          | ✅ Yes (org-wide)      |
| Audit Logs           | ❌ No           | ✅ Yes                 |
| Compliance Standards | ❌ No           | ✅ SOC, ISO readiness  |  


## Cost Reductions for Long-Term Subscriptions
* Yes, there are cost reductions for long-term commitments. GitHub pricing reductions fall into three practical categories
    1. Annual billing (official)
    2. Volume-based enterprise discounts
    3. Negotiated enterprise contracts 

### 1. Annual Billing (Official & Guaranteed) 
* github allows annual billing for teams and enterprise 
* GitHub does not publish a flat “X% discount”, but annual billing is cheaper than monthly. typically ~5%–10% cheaper than monthly billing

**Example – GitHub Team**
```
Monthly: $4 × 12 = $48 per user/year
Annual: ≈ $43 – $46 per user/year
savings: ~$2–$5 per user/year 
```

For 10 users:

* Monthly billing: $480 /year 
* Annual: ~$440–$460 /year
* Savings: $20–$40 /year 

**Example – GitHub Enterprise**
```
Monthly: $21 × 12 = $252 per user/year
Annual: ≈ $225 – $240 per user/year
savings: ~$12–$27 per user/year 
```

For 10 users: 

* Monthly: $2520 /year
* Annual: ~$2,250–$2,400 / year  
* Savings: $120–$270 / year  

## 2.Volume-Based Discounts (Enterprise Only)
* It means The per-user price of GitHub Enterprise decreases when an organization commits to a larger number
of users, usually under an annual or multi-year contract. 
* These apply ONLY to GitHub Enterprise, GitHub Team has no volume discounts 
* GitHub reduces the effective per-user cost based on:
    * Number of licensed users
    * Contract length(duration)
**Typical Volume Discount Ranges**
* These are industry-observed ranges, not official numbers.

| Licensed Users | Typical Discount  | Effective Price/User/Month |
| -------------- | ----------------- | -------------------------- |
| 1–25           | ❌ None           | $21                        |
| 25–50          | ~5%               | ~$20                       |
| 50–100         | ~8–12%            | ~$18.50–$19.30             |
| 100–250        | ~12–18%           | ~$17–$18.50                |
| 250–500        | ~18–25%           | ~$15.50–$17                |
| 500+           | 25%+ (negotiated) | <$15                       |

Example 1: 50-User Enterprise Org
* Without discount: 50 × $252 = $12,600/year 
* With ~10% volume discount: 50 × $227 = $11,350/year
* Savings: ~$1,250/year

**Multi-Year Commitments**

| Contract Length | Typical Effect       |
| --------------- | -------------------- |
| 1 year          | Base volume discount |
| 2 years         | +3–5% extra          |
| 3 years         | +5–10% extra         |

## 3. Negotiated Enterprise Contracts 
GitHub Negotiated Enterprise Contracts are custom commercial agreements between GitHub and a company where, 
Pricing, terms, limits, and add-ons are not list price and are tailored to the organization. 
* When Negotiation Is Possible
    * 100+ users
    * Regulated industries (banks, healthcare)
    * Multi-year contracts (2–3 years) 
**Per-User License Cost:**
| List Price     | Negotiated Range    |
| -------------- | ------------------- |
| $21/user/month | ~$14–$18/user/month |

**Contract Length (Time-Based Discounts):**
| Term    | Typical Benefit |
| ------- | --------------- |
| 1 year  | Base discount   |
| 2 years | +3–5%           |
| 3 years | +5–10%          |

Example 1: 150-User Enterprise (2-Year Contract)

| Item       | List           | Negotiated   |
| ---------- | -------------- | ------------ |
| License    | $252/user/year | ~$210        |
| Total/year | $37,800        | ~$31,500     |
| Savings    | —              | ~$6,300/year |


