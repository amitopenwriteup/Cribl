# Cribl IAM + Distributed Deployment Hands-On Lab

## Lab: Identity, Access Control, RBAC, and Worker-Group Security

**Platform:** Cribl Stream / Cribl Suite on-prem  
**Topology:** 1 Leader + 2 Workers, running with Docker Compose  
**Audience:** Cribl administrators, DevOps, SRE, security engineers, and MLOps/platform engineers  
**Difficulty:** Intermediate  
**Estimated time:** 2.5–4 hours

> This lab is designed around an existing distributed Docker Compose deployment. It focuses on Cribl Identity and Access Management (IAM), rather than teaching the initial Leader/Worker installation.

---

## 1. Learning Objectives

By the end of this lab you should be able to:

- Explain the difference between authentication and authorization.
- Identify the IAM components available in an on-prem Cribl deployment.
- Create named local users instead of using the built-in `admin` account for routine administration.
- Understand Cribl's Permissions model.
- Understand the legacy Roles/Policies model and when it is relevant.
- Create least-privilege access for different operational personas.
- Restrict access at the product and Worker Group level.
- Test access using different users.
- Understand permission inheritance and permission conflicts.
- Understand the role of the built-in `admin` and `system` accounts in logs.
- Create a service-account/API automation exercise.
- Understand how SSO fits into the design.
- Perform an IAM failure/recovery exercise.
- Produce evidence that can be used as an audit record.

Cribl's IAM documentation describes authentication, access control, Permissions, legacy Roles/Policies, service accounts, and SSO as major IAM capabilities. On-prem deployments can support both the Permissions model and the legacy Roles/Policies model, depending on deployment/license configuration.

---

# 2. Lab Architecture

Your existing environment:

```text
                         ┌─────────────────────────┐
                         │       Docker Host        │
                         │                         │
                         │  ┌───────────────────┐  │
                         │  │  Cribl Leader     │  │
                         │  │  IAM / Management │  │
                         │  └─────────┬─────────┘  │
                         │            │             │
                         │      Distributed        │
                         │      configuration      │
                         │            │             │
                         │    ┌───────┴───────┐     │
                         │    │               │     │
                         │ ┌──▼───────┐   ┌──▼──────┐
                         │ │ Worker 1 │   │ Worker 2│
                         │ └──────────┘   └─────────┘
                         └─────────────────────────┘
```

For this lab, assume:

```text
Leader   = cribl-leader
Worker 1 = cribl-worker-1
Worker 2 = cribl-worker-2
```

Replace these names with your actual Docker Compose service/container names.

---

# 3. Important IAM Concepts

## 3.1 Authentication

Authentication answers:

> "Who are you?"

Examples:

- Local username/password
- LDAP
- OIDC
- SAML
- Other supported authentication mechanisms

Cribl on-prem supports several authentication approaches depending on license and configuration.

## 3.2 Authorization

Authorization answers:

> "What are you allowed to do?"

Examples:

```text
User A → Read Only
User B → Editor
User C → Admin
```

Cribl's Permissions model provides fine-grained access control across deployment, product, Worker Group/Edge Fleet, and resource levels.

## 3.3 Members

A Member is an individual user who can access the deployment.

## 3.4 Teams

Teams allow common permissions to be assigned to a group of users instead of configuring each user individually.

Teams are available in Cribl.Cloud and Distributed deployments.

## 3.5 Permissions

Permissions define what a user or team can do at a particular level.

Common permissions include:

```text
Admin
Editor
Read Only
User
Collect
Maintainer
```

The exact available permission depends on the object level.

## 3.6 Roles and Policies

On-prem Enterprise deployments also support the legacy Roles/Policies RBAC model.

The conceptual structure is:

```text
User
  ↓
Role
  ↓
Policy
  ↓
Permissions
```

Cribl recommends using the newer Permissions model where appropriate, while some use cases still depend on legacy Roles.

---

# 4. Lab 0 — Verify the Docker Environment

## Objective

Confirm that the Leader and both Workers are running before changing IAM configuration.

### Step 1 — Check containers

Run:

```bash
docker compose ps
```

Expected result:

```text
NAME              STATUS
cribl-leader      Up
cribl-worker-1    Up
cribl-worker-2    Up
```

Your names may differ.

### Step 2 — Check logs

```bash
docker compose logs --tail=50
```

For an individual service:

```bash
docker compose logs --tail=50 cribl-leader
docker compose logs --tail=50 cribl-worker-1
docker compose logs --tail=50 cribl-worker-2
```

### Step 3 — Verify the UI

Open the URL you normally use for the Cribl Leader UI.

Confirm:

- Leader is accessible.
- Worker Group is visible.
- Worker 1 is connected.
- Worker 2 is connected.

### Success Criteria

- All three containers are running.
- Leader UI is accessible.
- Both Workers appear healthy/connected.
- No unexplained authentication or communication errors are present.

---

# 5. Lab 1 — Inventory the IAM Configuration

## Objective

Understand what IAM configuration already exists.

On the Leader, inspect:

```text
Settings
└── Global
    └── Access Management
        ├── Authentication
        ├── Local Users
        ├── Members & Teams
        ├── Permissions
        └── Roles / Policies
```

The exact UI labels can vary by Cribl version and enabled access model.

### Tasks

Record:

| Item | Your Environment |
|---|---|
| Cribl version | |
| Deployment type | Distributed |
| Leader hostname | |
| Worker 1 hostname | |
| Worker 2 hostname | |
| Authentication type | |
| IAM model available | |
| Enterprise/RBAC available? | |
| Existing users | |
| Existing teams | |
| Existing roles | |

### Questions

1. Which authentication method is currently enabled?
2. Is the deployment using local authentication?
3. Is the Permissions model available?
4. Is the Roles/Policies model available?
5. Which account are you currently using?
6. Are there any existing non-admin accounts?

---

# 6. Lab 2 — Create Named Administrative Accounts

## Objective

Stop using the built-in `admin` account for routine human administration.

Cribl documents the built-in `admin` account as a privileged account and recommends limiting interactive use of it to initial setup and emergency access. Create named accounts for normal administration.

### Create

Create a user representing an administrator:

```text
Username: lab-admin
First name: Lab
Last name: Administrator
```

Assign the appropriate administrative permission/role for your environment.

### Verification

Log out of the original `admin` account.

Log in as:

```text
lab-admin
```

Perform a harmless read operation.

### Evidence

Capture:

- User creation screen
- Assigned permission/role
- Successful login
- Worker Group visibility

### Security Question

Why is this better than having five engineers all use:

```text
admin
```

Expected discussion:

```text
Named account
     ↓
Individual identity
     ↓
Individual authorization
     ↓
Better accountability
     ↓
Better auditing
```

---

# 7. Lab 3 — Build a Least-Privilege Team Model

## Objective

Create different operational personas.

Use these fictional teams:

```text
Cribl-Admins
Cribl-Operators
Cribl-Auditors
```

Suggested design:

| Team | Intended access |
|---|---|
| Cribl-Admins | Administrative management |
| Cribl-Operators | Day-to-day operational configuration |
| Cribl-Auditors | Read-only visibility |

### Create users

Create at least:

```text
alice-admin
bob-operator
carol-auditor
```

Do not reuse passwords between accounts.

### Assign permissions

Design permissions appropriate to your available on-prem model.

A conceptual target:

```text
alice-admin
    → Admin

bob-operator
    → Editor

carol-auditor
    → Read Only
```

Do not automatically give all three users Admin.

### Verification Matrix

Test each user.

| Action | Admin | Operator | Auditor |
|---|---:|---:|---:|
| Login | YES | YES | YES |
| View configuration | YES | YES | YES |
| Modify configuration | YES | YES | NO |
| Delete resource | YES | Depending on scope | NO |
| Manage users | YES | NO | NO |
| Change IAM | YES | NO | NO |
| Deploy configuration | YES | Depending on permission | NO |

Document what actually happens in your Cribl version.

---

# 8. Lab 4 — Worker Group-Level Access Control

## Objective

Demonstrate that access can be restricted below the deployment/product level.

Create or use two Worker Groups if your environment permits:

```text
WG-LAB-PROD
WG-LAB-TEST
```

If your current environment already has Worker Groups, use them instead.

### Scenario

You have two teams:

```text
Production Operators
Test Operators
```

Production Operators should manage only:

```text
WG-LAB-PROD
```

Test Operators should manage only:

```text
WG-LAB-TEST
```

### Tasks

1. Create the two Worker Groups if required.
2. Create two teams.
3. Assign the appropriate product-level access.
4. Assign Worker Group-level permissions.
5. Log in as a member of each team.
6. Confirm visibility.
7. Attempt to modify the unauthorized Worker Group.

### Expected Learning

This demonstrates the principle:

```text
Global
  ↓
Product
  ↓
Worker Group
  ↓
Resource
```

Cribl's Permissions model supports this type of fine-grained access, and permissions can be inherited from higher levels.

---

# 9. Lab 5 — Permission Inheritance

## Objective

Understand how permissions propagate down the hierarchy.

Create a test user:

```text
inheritance-test
```

Give the user a higher-level permission and inspect what is inherited below it.

Conceptually:

```text
Global
  ↓
Product
  ↓
Worker Group
  ↓
Resource
```

### Experiment

Test:

```text
Scenario A:
User → Admin at higher level

Scenario B:
User → User at higher level

Scenario C:
User → Read Only at higher level
```

Observe what permissions become available at lower levels.

### Questions

1. Does the user automatically receive lower-level access?
2. Can you assign a less-privileged permission below an inherited higher privilege?
3. What happens when direct assignment and team assignment conflict?

Cribl documents that inheritance can determine the minimum permission at lower levels and that conflicting permissions can result in the most permissive effective permission taking precedence.

---

# 10. Lab 6 — Permission Conflict Test

## Objective

Demonstrate effective permissions when a user receives access from multiple sources.

Create:

```text
Team-A → Read Only
Team-B → Editor
```

Add:

```text
bob-operator
```

to both teams.

Then inspect Bob's effective permissions.

### Expected Result

The effective permission should reflect the most permissive applicable assignment.

Record:

```text
Team A permission:
Team B permission:
Direct permission:
Effective permission:
```

### Security Discussion

Why can conflicting permissions be dangerous?

Because adding a user to another team can unintentionally increase access.

---

# 11. Lab 7 — Legacy Roles and Policies

## Objective

Understand the legacy RBAC model available on supported on-prem Enterprise deployments.

Navigate to the Roles/Policies configuration.

Inspect the default roles.

Examples documented by Cribl include:

```text
stream_user
stream_reader
stream_editor
stream_admin

owner_all
editor_all
reader_all
collect_all
```

Do not modify or delete built-in roles during the lab.

### Exercise

Clone an appropriate role:

```text
stream_reader
```

to:

```text
lab_stream_reader
```

Inspect the policies attached to it.

Document:

```text
Role:
Policy:
Object:
Actions:
Scope:
```

### Important

Use a clone for experimentation rather than modifying default roles.

---

# 12. Lab 8 — Local User Authentication

## Objective

Understand local authentication as a fallback mechanism.

Navigate to:

```text
Settings
→ Global
→ Access Management
→ Authentication
```

Confirm the current authentication configuration.

Then inspect:

```text
Local Users
```

### Tasks

1. Confirm the built-in `admin` user exists.
2. Confirm your named test users exist.
3. Disable a test user.
4. Attempt to log in using that user.
5. Re-enable the user.
6. Confirm login works again.

### Expected Learning

Authentication determines whether the identity can log in.

Authorization determines what that identity can do after login.

---

# 13. Lab 9 — Service Account / Automation Exercise

## Objective

Understand non-human identities.

Create an automation scenario:

```text
CI/CD pipeline
      ↓
Cribl API
      ↓
Configuration / operational action
```

The goal is to avoid using a personal administrator credential for automation.

### Exercise

Identify an automation task that could use an API credential.

Examples:

```text
Configuration validation
Deployment automation
Health checks
Inventory collection
CI/CD integration
```

Create the minimum required credential/access according to your Cribl version and available licensing.

### Security Requirements

Document:

```text
Service account name:
Purpose:
Owner:
Required permissions:
Expiration/rotation plan:
Where credential is stored:
```

Never store credentials directly in:

```text
docker-compose.yml
Git repository
README.md
shell history
screenshots
```

Use a secret-management mechanism appropriate to your environment.

---

# 14. Lab 10 — Audit and Access Logs

## Objective

Observe how administrative actions appear in logs.

Cribl documents that the built-in `admin` and internal `system` users can appear in logs and audit trails. These identities should be correlated with expected administrative or automated activity.

### Exercise

1. Log in using `lab-admin`.
2. Perform a harmless configuration read.
3. Make a harmless configuration change.
4. Save/commit if appropriate.
5. Inspect the relevant Cribl logs.
6. Search for the username.

Look for fields such as:

```text
user
method
url
status
time
requestId
src
```

### Questions

1. Can you identify which user performed an action?
2. Can you identify the request?
3. Can you distinguish human activity from `system` activity?
4. What evidence would you provide during an audit?

---

# 15. Lab 11 — Admin Account Security

## Objective

Practice protecting the built-in privileged account.

Cribl recommends limiting interactive use of the built-in `admin` account to initial setup and emergency access.

### Tasks

1. Confirm named administrator access works.
2. Verify the named administrator has sufficient privileges.
3. Document the emergency `admin` recovery procedure.
4. Do not permanently remove the only recovery path.
5. Record who is authorized to use the emergency account.

### Security Principle

```text
Normal operations
        ↓
Named accounts
        ↓
Least privilege

Emergency
        ↓
Break-glass admin
```

---

# 16. Lab 12 — SSO Design Exercise

This exercise is optional if you do not have an IdP.

Cribl supports on-prem SSO using supported OIDC/SAML configurations depending on environment and licensing.

## Objective

Design an SSO integration without actually changing your production authentication.

Draw:

```text
                 ┌──────────────┐
                 │ Identity     │
                 │ Provider     │
                 └──────┬───────┘
                        │
                   OIDC/SAML
                        │
                        ▼
                ┌───────────────┐
                │ Cribl Leader  │
                └───────┬───────┘
                        │
                Authorization
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
       Admin Team             Operator Team
             │                     │
          Admin                  Editor
```

### Important Safety Step

Before enabling SSO, maintain fallback local access.

Cribl's on-prem SSO documentation specifically recommends configuring fallback local authentication so administrators are not locked out if SSO fails.

---

# 17. Lab 13 — IAM Failure Scenario

## Objective

Simulate an access-control problem safely.

Create:

```text
iam-test
```

Give it intentionally limited access.

### Test

Attempt:

```text
Login
View Worker Group
View configuration
Edit configuration
Deploy configuration
Manage users
```

Record the results.

Then increase its permission one level at a time and repeat the tests.

### Evidence Table

| Permission Level | Login | View | Edit | Deploy | IAM |
|---|---:|---:|---:|---:|---:|
| User | | | | | |
| Read Only | | | | | |
| Editor | | | | | |
| Admin | | | | | |

Fill the table with actual results from your environment.

---

# 18. Lab 14 — Least-Privilege Design Challenge

## Scenario

Your organization has four teams:

```text
SOC
Platform Engineering
Application Team
Audit
```

Requirements:

### SOC

Needs:

- View operational status
- View monitoring
- Investigate data
- No IAM administration

### Platform Engineering

Needs:

- Configure Worker Groups
- Configure Sources
- Configure Destinations
- Deploy changes
- No organization-level user administration

### Application Team

Needs:

- Access only to its assigned Worker Group
- Modify its own resources
- No access to other teams' Worker Groups

### Audit

Needs:

- Read-only access
- No configuration changes

### Task

Design:

```text
Users
Teams
Permissions
Worker Groups
```

Produce a diagram:

```text
Global
│
├── Platform Team
│     └── Admin/Editor as appropriate
│
├── SOC Team
│     └── Read/operational permissions
│
├── Application Team
│     └── Restricted Worker Group
│
└── Audit Team
      └── Read Only
```

---

# 19. Lab 15 — Docker-Level Security Review

Because your Cribl environment runs in Docker Compose, review security outside the Cribl UI as well.

## Check

```bash
docker compose ps
docker compose config
docker inspect <leader-container>
docker inspect <worker-container>
```

Review:

- Published ports
- Environment variables
- Mounted volumes
- Secrets
- Container privileges
- Network configuration
- Persistent configuration directories

### Questions

1. Are credentials stored in environment variables?
2. Are Cribl configuration directories persisted?
3. Is the Leader UI exposed outside the lab network?
4. Are Worker ports unnecessarily exposed?
5. Can a compromised Worker directly reach management services?
6. Are container logs retained?

---

# 20. Lab 16 — Backup and Recovery

## Objective

Verify that IAM-related configuration can be recovered.

Before changing major IAM configuration:

```text
Backup
   ↓
Change
   ↓
Test
   ↓
Recover if required
```

Document where your Docker volumes/configuration are stored.

Example:

```bash
docker volume ls
```

Inspect your Compose file for persistent volumes.

Do not delete volumes as part of this lab unless you have a verified backup.

---

# 21. Final Capstone Lab

## Scenario

You are the administrator of this deployment:

```text
Cribl Leader
     │
     ├── Worker Group: Production
     │       ├── Worker 1
     │       └── Worker 2
     │
     └── Worker Group: Development
```

Create the following access model:

```text
                    Cribl IAM
                       │
       ┌───────────────┼────────────────┐
       │               │                │
    Platform          SOC             Audit
       │               │                │
     Admin           Reader          Read Only
       │
       ├──────── Production
       │
       └──────── Development
```

### Required identities

```text
platform-admin
soc-operator
dev-operator
auditor
```

### Required outcomes

#### platform-admin

Can:

- Manage configuration
- Manage Worker Groups
- Perform deployments
- Manage IAM

#### soc-operator

Can:

- Monitor
- Investigate
- View configuration

Cannot:

- Manage IAM
- Change production configuration

#### dev-operator

Can:

- Modify Development Worker Group

Cannot:

- Modify Production Worker Group
- Manage IAM

#### auditor

Can:

- View relevant configuration and access information

Cannot:

- Modify anything

---

# 22. Capstone Validation

Run the following test matrix:

| Test | platform-admin | soc-operator | dev-operator | auditor |
|---|---:|---:|---:|---:|
| Login | PASS/FAIL | PASS/FAIL | PASS/FAIL | PASS/FAIL |
| View Production | | | | |
| View Development | | | | |
| Edit Production | | | | |
| Edit Development | | | | |
| Deploy | | | | |
| Create User | | | | |
| Modify Permissions | | | | |
| View Monitoring | | | | |

The important part is not to assume the result. **Test each permission in the actual deployment.**

---

# 23. Troubleshooting

## User cannot log in

Check:

```text
Authentication configuration
User enabled/disabled state
Username/password
Local authentication fallback
SSO configuration
```

For local authentication, inspect the Local Users configuration on the Leader.

## User can log in but cannot see a Worker Group

Check:

```text
Global permission
Product permission
Worker Group permission
Team membership
Direct permission
Permission inheritance
```

## User has more access than expected

Check:

```text
Direct assignment
Team membership
Inherited permission
Multiple team memberships
Legacy Roles
```

Cribl documents that conflicting Permissions can result in the most permissive effective permission taking precedence.

## SSO users are locked out

Use the configured local fallback account if enabled.

Do not disable your only working administrative access before validating SSO.

## Worker problems after IAM changes

Check:

```bash
docker compose ps
docker compose logs --tail=100 cribl-leader
docker compose logs --tail=100 cribl-worker-1
docker compose logs --tail=100 cribl-worker-2
```

Use your actual service names.

---

# 24. Evidence Checklist

Collect the following for the completed lab:

- [ ] Docker Compose topology screenshot
- [ ] Leader health screenshot
- [ ] Worker 1 connected screenshot
- [ ] Worker 2 connected screenshot
- [ ] Authentication configuration
- [ ] Named administrator
- [ ] Operator account
- [ ] Auditor account
- [ ] Teams
- [ ] Permission assignments
- [ ] Worker Group permissions
- [ ] Permission conflict test
- [ ] Audit/log evidence
- [ ] Service-account design
- [ ] SSO architecture diagram
- [ ] Final access-control matrix
- [ ] Docker security review

---

# 25. Expected Skills After Completion

After completing this lab, you should be comfortable explaining:

```text
Authentication
      │
      ▼
Who are you?
      │
      ▼
Member / Local User
      │
      ▼
Team / Role
      │
      ▼
Permission / Policy
      │
      ▼
Product
      │
      ▼
Worker Group
      │
      ▼
Resource
```

You should also understand the difference between:

```text
Authentication
    ≠
Authorization
```

and:

```text
User
    ≠
Team
    ≠
Role
    ≠
Permission
```

---

# 26. Recommended Lab Order

If you want to run this as a structured training session:

```text
Day 1 / Session 1
├── Lab 0  - Environment verification
├── Lab 1  - IAM inventory
├── Lab 2  - Named accounts
└── Lab 3  - Teams and least privilege

Day 1 / Session 2
├── Lab 4  - Worker Group access
├── Lab 5  - Permission inheritance
├── Lab 6  - Permission conflicts
└── Lab 7  - Roles and Policies

Day 2
├── Lab 8  - Local authentication
├── Lab 9  - Service accounts
├── Lab 10 - Audit logs
├── Lab 11 - Admin security
└── Lab 12 - SSO design

Day 3
├── Lab 13 - IAM failure scenario
├── Lab 14 - Least privilege challenge
├── Lab 15 - Docker security
├── Lab 16 - Backup/recovery
└── Capstone - Complete IAM implementation
```

---

# 27. Reference Documentation

Use the Cribl documentation for the exact UI labels and version-specific behavior:

- Cribl Identity and Access Management: https://docs.cribl.io/iam/
- Permissions: https://docs.cribl.io/iam/permissions/
- Permissions Model: https://docs.cribl.io/iam/permissions-model/
- Members and Teams: https://docs.cribl.io/iam/members-teams/
- Local Users: https://docs.cribl.io/iam/local-users/
- Roles and Policies: https://docs.cribl.io/iam/roles-policies/
- Service Accounts: https://docs.cribl.io/iam/service-accounts
- Authentication: https://docs.cribl.io/iam/authentication/
- On-Prem SSO: https://docs.cribl.io/iam/sso-on-prem/

> **Version note:** Cribl IAM capabilities and UI paths can vary by Cribl version, deployment type, and license tier. This lab intentionally asks you to verify the actual behavior in your environment rather than assuming every permission is available. In particular, the legacy Roles/Policies RBAC model is tied to supported on-prem Distributed/Enterprise deployments, while the Permissions model is the current fine-grained access-control model.

---

# 28. Lab Completion Report

Fill this out after completing the lab.

## Environment

```text
Cribl version:
Docker version:
Docker Compose version:
Leader:
Worker 1:
Worker 2:
```

## Authentication

```text
Authentication method:
Fallback authentication:
SSO configured:
```

## Authorization

```text
IAM model:
Permissions enabled:
Legacy Roles enabled:
```

## Users

```text
Admin:
Operator:
Auditor:
```

## Worker Groups

```text
Production:
Development:
```

## Findings

```text
1.

2.

3.

```

## Security Improvements Identified

```text
1.

2.

3.

```

## Final Assessment

```text
[ ] PASS
[ ] PASS WITH FINDINGS
[ ] FAIL
```

### Notes

```text

```
