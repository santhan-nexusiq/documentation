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