---
layout: post
title: "Microsoft Entra PIM in practice: a complete implementation guide for privileged access"
date: 2025-05-02 09:00:00 +0200
categories: [Security, Azure]
tags: [Microsoft Entra, PIM, Azure, IAM, Zero Trust, Conditional Access, Microsoft Graph, PowerShell, RBAC, Security]
description: "A detailed practitioner guide to designing, deploying, automating and operating Microsoft Entra Privileged Identity Management for Entra roles, Azure resources and privileged groups."
---

> **Scope and accuracy note:** This article is a technical implementation guide, not legal or licensing advice. Microsoft product behaviour and licensing change over time. Validate the current Microsoft Product Terms, Microsoft Learn documentation and your licensing agreement before production deployment.

## Executive summary

Privileged accounts are among the highest-value targets in a cloud environment. A compromised account with permanent Global Administrator, Privileged Role Administrator, Owner or User Access Administrator rights can change identities, security controls, role assignments and production resources.

**Microsoft Entra Privileged Identity Management**, commonly called **PIM**, reduces this exposure by replacing permanent privileged access with controlled, time-bound and auditable activation.

Instead of giving an administrator a role continuously, PIM can make the administrator **eligible**. The administrator activates the role only when needed and, depending on policy, must:

- complete multifactor authentication;
- satisfy a Conditional Access authentication context;
- provide a business justification;
- provide a change or incident ticket;
- obtain approval;
- accept a short activation duration;
- generate notifications and audit records.

A strong PIM implementation is not simply “enable MFA and approval.” It is an operating model covering role design, licensing, emergency access, separation of duties, approval availability, Conditional Access, monitoring, access reviews, automation and recurring governance.

This guide covers:

1. PIM architecture and terminology;
2. licensing and scope;
3. Entra roles, Azure resource roles and PIM for Groups;
4. a risk-based policy design;
5. deployment steps;
6. Conditional Access integration;
7. approvals and emergency access;
8. Microsoft Graph and PowerShell automation;
9. monitoring, alerts and KQL;
10. testing and troubleshooting;
11. a practical 30-day rollout plan.

## 1. What PIM protects

PIM has three main providers:

| Provider | Protects | Management API |
|---|---|---|
| PIM for Microsoft Entra roles | Directory roles such as Global Administrator, Security Administrator and Exchange Administrator | Microsoft Graph |
| PIM for Azure resources | Azure RBAC roles such as Owner, Contributor and User Access Administrator | Azure Resource Manager |
| PIM for Groups | Eligible group membership or ownership for selected security or Microsoft 365 groups | Microsoft Graph |

These providers solve related but different problems.

### Microsoft Entra roles

An Entra role grants permissions in the identity and Microsoft 365 control plane. Examples include:

- Global Administrator;
- Privileged Role Administrator;
- Conditional Access Administrator;
- Security Administrator;
- Exchange Administrator;
- SharePoint Administrator;
- Intune Administrator;
- Application Administrator;
- Cloud Application Administrator;
- Authentication Administrator.

### Azure resource roles

Azure RBAC controls access to management groups, subscriptions, resource groups and resources. Examples include:

- Owner;
- Contributor;
- User Access Administrator;
- Reader;
- Network Contributor;
- Virtual Machine Contributor;
- Key Vault Administrator;
- custom Azure roles.

A user can have different privilege paths. For example, an identity administrator might be eligible for Conditional Access Administrator in Entra ID and eligible for Reader on a security subscription.

### PIM for Groups

PIM for Groups makes group membership or ownership eligible and time-bound. This is useful when an application, data platform, administrative unit or third-party system grants access through a group rather than a directly manageable PIM role.

A common pattern is:

```text
User
  ↓ activates eligible group membership
PIM-managed privileged group
  ↓ assigned to
Azure RBAC role / application role / resource permission
```

> **Design warning:** Group-mediated administration can simplify operations, but it can also hide the effective privilege path. Document every privileged group, downstream role and nested membership dependency.

## 2. Core concepts

### Eligible assignment

An eligible assignment means the principal can request activation. The privileged permissions are not continuously available.

### Active assignment

An active assignment grants the permissions immediately. It can be permanent or time-bound. Active assignments bypass the eligible activation experience because the role is already active.

### Activation

Activation creates temporary active access from an eligible assignment. The activation policy may require MFA, Conditional Access, justification, ticket information and approval.

### Assignment duration versus activation duration

These are different controls:

- **Eligible assignment duration**: how long the user remains eligible, for example 180 days.
- **Activation duration**: how long each activation lasts, for example 1 or 4 hours.

A user might be eligible for six months but activate the role for only one hour at a time.

### Scope

For Entra roles, scope can be tenant-wide, administrative-unit scoped or application scoped where supported.

For Azure roles, scope can be:

```text
Management Group
  └── Subscription
       └── Resource Group
            └── Resource
```

Apply permissions at the narrowest practical scope. A Network Contributor role on one resource group is preferable to Contributor on an entire subscription when the operational task is limited to that network scope.

## 3. Licensing model

PIM is available through Microsoft Entra ID P2 and Microsoft Entra ID Governance-related offers, subject to current Microsoft licensing terms.

For planning, count the people using PIM capabilities, including relevant populations such as:

- users assigned eligible Entra or Azure roles;
- users assigned eligible membership or ownership in PIM for Groups;
- users approving or rejecting PIM activation requests;
- users assigned to privileged access reviews;
- users performing those access reviews.

### Worked example

Suppose an organisation has:

- 50 distinct administrators with eligible PIM assignments;
- 5 distinct approvers who do not overlap with those administrators;
- 2 reviewers who are already among the 5 approvers.

The conservative licensing population is:

```text
50 administrators
+ 5 distinct approvers
+ 0 additional reviewers because the reviewers overlap
= 55 licensed people
```

If all five approvers are already part of the 50 administrators, the distinct population is 50, not 55.

> **Important:** Count people, not merely accounts. If one licensed employee uses a standard account and a separate privileged account, confirm the applicable Microsoft Product Terms rather than automatically buying two licences or assuming account-level licensing.

Potential licence entitlements can also come from suites such as Microsoft 365 E5 or Enterprise Mobility + Security E5. Always inventory existing entitlements before purchasing standalone add-ons.

## 4. A practical target architecture

A mature design separates daily productivity identities, privileged identities, emergency identities and workload identities.

```text
Daily user account
  └── Email, Teams, documents, standard applications

Dedicated admin account
  └── Eligible PIM roles only
      ├── MFA / phishing-resistant authentication
      ├── compliant or privileged workstation
      ├── justification and ticket
      └── short activation duration

Emergency access accounts
  └── Permanent Global Administrator
      ├── tightly monitored
      ├── excluded only from policies that would create lockout
      ├── strong independent authentication
      └── tested regularly

Workload identities
  └── Managed identities or service principals
      ├── least-privilege app permissions
      ├── no interactive sign-in
      └── separate lifecycle and credential governance
```

### Why dedicated admin accounts?

A separate privileged identity reduces exposure to phishing, browser extensions, email attachments and normal productivity activity. PIM reduces standing privilege, but PIM does not make a daily account safe by itself.

### Why emergency access accounts?

PIM, approval and Conditional Access can create lockout if poorly designed. Maintain at least two emergency access accounts with independent monitoring and operational procedures. Their use should generate an urgent security response.

## 5. Role classification

Do not give every role the same activation policy. Classify roles by potential impact.

### Tier 0: identity control plane

Examples:

- Global Administrator;
- Privileged Role Administrator;
- Conditional Access Administrator;
- Authentication Policy Administrator;
- Domain Name Administrator;
- roles able to modify trusted identity or federation controls.

Recommended policy:

| Setting | Recommendation |
|---|---|
| Assignment | Eligible, time-bound where practical |
| Activation duration | 1 hour |
| Authentication | Phishing-resistant strength or tightly controlled authentication context |
| Approval | Yes, two or more configured approvers for availability |
| Justification | Required |
| Ticket | Required for planned changes |
| Device | Compliant device or privileged access workstation |
| Notification | Security operations and identity governance |

### Tier 1: security and platform administration

Examples:

- Security Administrator;
- Intune Administrator;
- Exchange Administrator;
- SharePoint Administrator;
- Azure Owner or User Access Administrator at subscription scope;
- Key Vault Administrator on critical vaults.

Recommended activation duration: 2 hours, with MFA or authentication context, justification and approval for high-impact scopes.

### Tier 2: workload administration

Examples:

- Virtual Machine Contributor;
- Network Contributor;
- Website Contributor;
- SQL DB Contributor;
- custom application-operator roles.

Recommended activation duration: 4 hours when operationally justified. Approval may be optional for narrow, well-monitored roles.

### Reader roles

Do not assume Reader is harmless. Security Reader, Global Reader and subscription Reader can expose sensitive architecture, incidents, policy and configuration. Use eligibility where disclosure risk is material.

## 6. Baseline policy matrix

The following matrix is a practical starting point, not a universal standard.

| Role category | Duration | MFA/Auth context | Approval | Justification | Ticket |
|---|---:|---|---|---|---|
| Global Administrator | 1 hour | Phishing-resistant | Required | Required | Required |
| Privileged Role Administrator | 1 hour | Phishing-resistant | Required | Required | Required |
| Conditional Access Administrator | 1 hour | Phishing-resistant | Required | Required | Required |
| Security Administrator | 2 hours | Strong MFA | Required | Required | Recommended |
| Subscription Owner | 2 hours | Strong MFA | Required | Required | Required |
| User Access Administrator | 1 hour | Strong MFA | Required | Required | Required |
| Contributor on production RG | 2 hours | MFA | Risk-based | Required | Recommended |
| Contributor on non-production RG | 4 hours | MFA | Optional | Required | Optional |
| Reader | 8 hours | MFA for sensitive scopes | No | Optional | No |

### Trick: minimise role stacking

Administrators often activate several broad roles because the task is unclear. Create task-based runbooks showing the minimum role needed for common activities.

Example:

| Task | Avoid | Prefer |
|---|---|---|
| Restart a VM | Contributor | Virtual Machine Contributor at resource-group scope |
| Manage network rules | Owner | Network Contributor at network resource-group scope |
| Assign Azure RBAC | Owner | User Access Administrator at the required scope |
| Manage enterprise apps | Global Administrator | Cloud Application Administrator or Application Administrator |
| Review sign-in logs | Global Administrator | Security Reader or Reports Reader |

## 7. Deployment prerequisites

Before onboarding roles:

- confirm licensing populations;
- inventory privileged assignments;
- remove stale accounts;
- establish at least two emergency access accounts;
- ensure eligible users have suitable authentication methods;
- define approver groups and backups;
- document role owners;
- establish a ServiceNow, Jira or Azure DevOps ticket convention;
- export current Entra and Azure RBAC assignments;
- enable audit-log retention and SIEM integration;
- pilot Conditional Access in report-only mode.

## 8. Inventory existing Entra role assignments

Install Microsoft Graph PowerShell:

```powershell
Install-Module Microsoft.Graph -Scope CurrentUser
```

Connect with read permissions:

```powershell
Connect-MgGraph -Scopes @(
    'RoleManagement.Read.Directory',
    'Directory.Read.All'
)
```

Export active Entra role assignments:

```powershell
$roles = Get-MgRoleManagementDirectoryRoleDefinition -All
$roleMap = @{}
$roles | ForEach-Object { $roleMap[$_.Id] = $_.DisplayName }

$assignments = Get-MgRoleManagementDirectoryRoleAssignment -All

$report = foreach ($assignment in $assignments) {
    $principal = Get-MgDirectoryObject -DirectoryObjectId $assignment.PrincipalId

    [PSCustomObject]@{
        Role              = $roleMap[$assignment.RoleDefinitionId]
        PrincipalId       = $assignment.PrincipalId
        PrincipalType     = $principal.AdditionalProperties['@odata.type']
        DirectoryScopeId  = $assignment.DirectoryScopeId
        AssignmentId      = $assignment.Id
    }
}

$report |
    Sort-Object Role, PrincipalId |
    Export-Csv './entra-active-role-assignments.csv' -NoTypeInformation -Encoding UTF8
```

> **Operational tip:** Resolve users, groups and service principals separately for readable display names. Graph SDK object shapes can vary, so test the report in a non-production tenant.

## 9. Inventory Azure RBAC assignments

```powershell
Connect-AzAccount

$results = foreach ($subscription in Get-AzSubscription) {
    Set-AzContext -SubscriptionId $subscription.Id | Out-Null

    Get-AzRoleAssignment | ForEach-Object {
        [PSCustomObject]@{
            SubscriptionName = $subscription.Name
            SubscriptionId   = $subscription.Id
            DisplayName      = $_.DisplayName
            SignInName       = $_.SignInName
            ObjectType       = $_.ObjectType
            Role             = $_.RoleDefinitionName
            Scope            = $_.Scope
            AssignmentId     = $_.RoleAssignmentId
        }
    }
}

$results |
    Sort-Object SubscriptionName, Role, DisplayName |
    Export-Csv './azure-rbac-assignments.csv' -NoTypeInformation -Encoding UTF8
```

Look specifically for:

- permanent Owner and User Access Administrator assignments;
- Contributor at management-group or subscription scope;
- assignments to broad groups;
- guest administrators;
- inactive users;
- service principals with human-oriented roles;
- role assignments at root management-group scope;
- custom roles containing `Microsoft.Authorization/roleAssignments/write`.

## 10. Configure Entra role settings

In the Microsoft Entra admin center:

```text
ID Governance
  → Privileged Identity Management
  → Microsoft Entra roles
  → Roles
  → Select role
  → Role settings
  → Edit
```

Key controls include:

### Activation maximum duration

Microsoft supports 1–24 hours. Shorter is better for critical roles, but the duration must accommodate real operational tasks.

### Require multifactor authentication

This can require MFA at activation. However, a user might not be prompted if the current session already contains a suitable authentication claim.

For stricter control, use a Conditional Access authentication context and an authentication strength.

### Require justification

Require meaningful business context. Avoid generic text such as “admin work.”

Good example:

```text
CHG0042187 - Update Conditional Access policy for privileged workstation pilot.
Rollback owner: Identity Operations. Maintenance window: 20:00–21:00 CET.
```

### Require ticket information

PIM records the ticket system and number, but the field does not automatically validate the ticket against ServiceNow, Jira or another ITSM platform.

### Require approval

Use approval for high-risk roles, but design for operational availability:

- configure at least two approvers;
- avoid making the requestor the approver;
- define out-of-hours coverage;
- configure escalation procedures;
- test the approval path before removing permanent access.

For Entra roles, if approval is required and no explicit approvers are configured, active Privileged Role Administrators or Global Administrators can become default approvers. Relying on this default can create lockout when all such administrators are only eligible.

## 11. Configure Azure resource role settings

Path:

```text
ID Governance
  → Privileged Identity Management
  → Azure resources
  → Select management group, subscription, resource group or resource
  → Settings
  → Select role
  → Edit
```

Azure resource role settings are configured **per role and per resource scope**. A policy configured for Contributor at subscription scope is not automatically inherited by Contributor at child resource-group scope.

This has major operational consequences. Build a policy inventory rather than assuming inheritance.

Example policy register:

```text
Scope: /subscriptions/11111111-1111-1111-1111-111111111111
Role: Owner
Activation: 1 hour
Approval: Identity Governance Approvers
MFA: Required
Ticket: Required

Scope: /subscriptions/.../resourceGroups/rg-app-prod
Role: Contributor
Activation: 2 hours
Approval: Application Operations Approvers
MFA: Required
Ticket: Required
```

## 12. Conditional Access integration

PIM can require a Conditional Access authentication context during activation. This allows controls such as:

- phishing-resistant authentication;
- compliant device;
- approved client application;
- terms of use;
- specific sign-in risk requirements;
- reauthentication every time.

### Recommended two-policy model

A subtle but important limitation is that satisfying the activation authentication context does not necessarily prevent use of the activated role from another session or device.

Use two policies:

1. **Activation policy** targeting the PIM authentication context.
2. **Usage policy** targeting the activated directory role or eligible users.

Example:

```text
Policy 1: CA-PIM-Activation-Tier0
Target: Authentication context "Tier0 PIM Activation"
Grant: Phishing-resistant MFA
Device: Compliant
Session: Sign-in frequency every time

Policy 2: CA-Privileged-Roles-Usage
Target: Selected directory roles
Grant: Phishing-resistant MFA
Device: Compliant
Location: Block unsupported countries where appropriate
```

> **Critical warning:** Never target an authentication context and the directory role in the same activation policy expecting the role condition to apply before activation. The user does not yet have the active role at the activation decision point.

### 10-minute reauthentication window

When Conditional Access uses sign-in frequency **Every time**, Microsoft documents a 10-minute window in which activation of another eligible role may not prompt again. Design expectations and audit analysis around this behaviour.

## 13. Approval design

A robust approval model answers:

- Which roles require approval?
- Who can approve during office hours?
- Who can approve out of hours?
- What evidence must an approver check?
- What is the maximum response time?
- What happens during an incident?
- How are conflicts of interest prevented?

### Suggested decision checklist for approvers

```text
[ ] Requestor is authorised for the operational team
[ ] Requested role is the least privileged role for the task
[ ] Scope is no broader than necessary
[ ] Ticket exists and is approved
[ ] Maintenance window is valid
[ ] Request duration is appropriate
[ ] High-risk production change has rollback steps
[ ] No active security incident affects the requestor
```

### Availability pattern

For a 24×7 environment:

```text
Primary approvers: Identity Operations
Secondary approvers: Security Operations
Emergency escalation: Major Incident Manager
Excluded: Requestor and direct automation identity
```

PIM recommends multiple approvers for availability. Approval does not require the approver to hold the requested role, but approvers are part of the governed and licensed PIM population under applicable terms.

## 14. Create an eligible Entra role assignment with Microsoft Graph

PIM for Entra roles uses schedule request objects. For an eligible assignment, use the eligibility schedule request endpoint.

First find the role definition:

```powershell
Connect-MgGraph -Scopes @(
    'RoleManagement.ReadWrite.Directory',
    'Directory.Read.All'
)

$role = Get-MgRoleManagementDirectoryRoleDefinition -Filter "displayName eq 'Security Administrator'"
$user = Get-MgUser -UserId 'admin.user@contoso.com'
```

Create a 180-day eligible assignment:

```powershell
$params = @{
    Action            = 'adminAssign'
    PrincipalId       = $user.Id
    RoleDefinitionId  = $role.Id
    DirectoryScopeId  = '/'
    Justification     = 'Approved privileged access onboarding - CHG0042001'
    ScheduleInfo      = @{
        StartDateTime = (Get-Date).ToUniversalTime()
        Expiration    = @{
            Type     = 'afterDuration'
            Duration = 'P180D'
        }
    }
}

New-MgRoleManagementDirectoryRoleEligibilityScheduleRequest -BodyParameter $params
```

### Why time-bound eligibility?

Permanent eligibility is safer than permanent active access, but periodic expiration forces an ownership decision. A 90- or 180-day eligibility window aligns well with access reviews in many organisations.

## 15. Activate an eligible Entra role using Graph

Find the current user's eligible assignment and role definition, then create a self-activation request.

Conceptual REST example:

```http
POST https://graph.microsoft.com/v1.0/roleManagement/directory/roleAssignmentScheduleRequests
Content-Type: application/json
Authorization: Bearer {token}

{
  "action": "selfActivate",
  "principalId": "00000000-0000-0000-0000-000000000001",
  "roleDefinitionId": "00000000-0000-0000-0000-000000000002",
  "directoryScopeId": "/",
  "justification": "INC0012345 - Investigate critical identity incident",
  "scheduleInfo": {
    "startDateTime": "2025-05-02T08:00:00Z",
    "expiration": {
      "type": "afterDuration",
      "duration": "PT1H"
    }
  },
  "ticketInfo": {
    "ticketSystem": "ServiceNow",
    "ticketNumber": "INC0012345"
  }
}
```

The role policy determines whether approval, justification, ticket information or other controls are required.

For self-service update, renewal or extension operations through Graph, Microsoft requires MFA enforcement and a session in which the caller was challenged for MFA.

## 16. List PIM assignment requests

Microsoft Graph PowerShell:

```powershell
Import-Module Microsoft.Graph.Identity.Governance

Connect-MgGraph -Scopes 'RoleAssignmentSchedule.ReadWrite.Directory'

$requests = Get-MgRoleManagementDirectoryRoleAssignmentScheduleRequest -All

$requests |
    Select-Object Id,
                  Action,
                  Status,
                  PrincipalId,
                  RoleDefinitionId,
                  DirectoryScopeId,
                  Justification,
                  CreatedDateTime,
                  CompletedDateTime |
    Export-Csv './pim-role-assignment-requests.csv' -NoTypeInformation -Encoding UTF8
```

Graph supports OData filtering and expansion. Prefer server-side filters for large tenants.

Example REST query:

```http
GET https://graph.microsoft.com/v1.0/roleManagement/directory/roleAssignmentScheduleRequests?$filter=status eq 'PendingApproval'
```

## 17. Retrieve role policies and rules

PIM role settings are represented in Microsoft Graph as role management policies and rules.

```http
GET https://graph.microsoft.com/v1.0/policies/roleManagementPolicies?$filter=scopeId eq '/' and scopeType eq 'DirectoryRole'&$expand=rules
```

PowerShell:

```powershell
Connect-MgGraph -Scopes 'RoleManagementPolicy.Read.Directory'

$uri = "https://graph.microsoft.com/v1.0/policies/roleManagementPolicies?`$filter=scopeId eq '/' and scopeType eq 'DirectoryRole'&`$expand=rules"
$response = Invoke-MgGraphRequest -Method GET -Uri $uri
$response.value | ConvertTo-Json -Depth 20 | Out-File './pim-role-policies.json'
```

This is useful for configuration baselines and drift detection.

### Policy-as-code approach

Store a desired policy matrix in source control:

```yaml
roles:
  Global Administrator:
    activationDuration: PT1H
    requireApproval: true
    requireJustification: true
    requireTicket: true
    authenticationContext: c1
    approverGroup: PIM-Tier0-Approvers

  Security Administrator:
    activationDuration: PT2H
    requireApproval: true
    requireJustification: true
    requireTicket: false
    authenticationContext: c2
    approverGroup: PIM-Security-Approvers
```

A scheduled automation can compare actual policies against the baseline and report drift. Do not automatically overwrite critical policies without change control and emergency rollback.

## 18. Azure resource PIM automation

PIM for Azure resource roles uses Azure Resource Manager APIs, not the Entra directory role Graph endpoints.

An eligibility schedule request uses a path similar to:

```text
PUT {scope}/providers/Microsoft.Authorization/roleEligibilityScheduleRequests/{requestName}?api-version=2020-10-01
```

Conceptual Azure CLI REST example:

```bash
REQUEST_ID=$(uuidgen)
SCOPE="/subscriptions/11111111-1111-1111-1111-111111111111"
ROLE_DEFINITION_ID="$SCOPE/providers/Microsoft.Authorization/roleDefinitions/22222222-2222-2222-2222-222222222222"
PRINCIPAL_ID="33333333-3333-3333-3333-333333333333"

az rest \
  --method put \
  --url "https://management.azure.com${SCOPE}/providers/Microsoft.Authorization/roleEligibilityScheduleRequests/${REQUEST_ID}?api-version=2020-10-01" \
  --body "{
    \"properties\": {
      \"principalId\": \"${PRINCIPAL_ID}\",
      \"roleDefinitionId\": \"${ROLE_DEFINITION_ID}\",
      \"requestType\": \"AdminAssign\",
      \"justification\": \"Approved onboarding - CHG0042002\",
      \"scheduleInfo\": {
        \"startDateTime\": \"2025-05-02T08:00:00Z\",
        \"expiration\": {
          \"type\": \"AfterDuration\",
          \"duration\": \"P180D\"
        }
      }
    }
  }"
```

> **Safety:** Use a test subscription first. Confirm API versions and schemas in current Microsoft documentation. Azure resource PIM automation requires appropriate Azure RBAC, typically Owner or User Access Administrator at the managed scope.

## 19. PIM for Groups design

Use PIM for Groups when applications or resources consume group membership and direct PIM integration is unavailable or operationally inefficient.

Good candidates:

- privileged application administrator groups;
- production SQL support groups;
- high-value data-access groups;
- emergency operational groups;
- administrative unit management groups.

Avoid using one group to aggregate unrelated privileges.

Bad design:

```text
PIM-Super-Admins
  ├── Azure Owner
  ├── Exchange Administrator
  ├── Intune Administrator
  ├── Production database admin
  └── SaaS super administrator
```

Better design:

```text
PIM-Azure-Prod-Network-Operators
PIM-Entra-Conditional-Access-Admins
PIM-M365-Exchange-Admins
PIM-Data-Prod-SQL-Operators
```

This enables separate approval, activation duration, ownership and audit context.

## 20. Monitoring and alerting

A PIM deployment is incomplete without monitoring.

Monitor at least:

- activation of Global Administrator and Privileged Role Administrator;
- permanent active role assignments;
- role assignment outside the approved PIM workflow;
- repeated denied activation requests;
- after-hours activations;
- emergency-account sign-ins;
- policy changes;
- approver changes;
- Conditional Access changes affecting PIM;
- Azure Owner or User Access Administrator activation at broad scopes;
- activations without ticket or meaningful justification;
- unused eligible assignments;
- assignments nearing expiry.

### Send logs to Log Analytics or Microsoft Sentinel

Configure Microsoft Entra diagnostic settings to send relevant audit and sign-in logs to a Log Analytics workspace used by Microsoft Sentinel or another monitoring process.

### KQL: PIM-related audit activity

```kusto
AuditLogs
| where TimeGenerated > ago(30d)
| where Category =~ "RoleManagement"
| project TimeGenerated,
          OperationName,
          Result,
          ResultReason,
          InitiatedBy,
          TargetResources,
          CorrelationId
| order by TimeGenerated desc
```

### KQL: high-impact role management events

```kusto
AuditLogs
| where TimeGenerated > ago(30d)
| where Category =~ "RoleManagement"
| where OperationName has_any (
    "Global Administrator",
    "Privileged Role Administrator",
    "Conditional Access Administrator",
    "role assignment",
    "eligible assignment"
)
| project TimeGenerated,
          OperationName,
          Result,
          ResultReason,
          InitiatedBy,
          TargetResources,
          CorrelationId
| order by TimeGenerated desc
```

Because audit operation names and schemas can change, first inspect your tenant data:

```kusto
AuditLogs
| where TimeGenerated > ago(14d)
| where Category =~ "RoleManagement"
| summarize Count=count() by OperationName
| order by Count desc
```

### KQL: after-hours privileged operations

```kusto
AuditLogs
| where TimeGenerated > ago(30d)
| where Category =~ "RoleManagement"
| extend LocalHour = datetime_part("hour", TimeGenerated + 1h)
| where LocalHour < 7 or LocalHour >= 20
| project TimeGenerated,
          LocalHour,
          OperationName,
          Result,
          InitiatedBy,
          TargetResources
| order by TimeGenerated desc
```

> **Timezone warning:** The example uses a fixed UTC+1 offset and is not daylight-saving aware. Production analytics should use an agreed timezone strategy or enrich events appropriately.

## 21. Access reviews and lifecycle

PIM controls activation, but eligibility can still become stale. Run periodic access reviews for privileged roles and groups.

Recommended frequencies:

| Population | Review frequency |
|---|---|
| Global Administrator / Privileged Role Administrator | Monthly |
| Tier 0 privileged roles | Monthly or quarterly |
| Subscription Owner / User Access Administrator | Quarterly |
| Production Contributor roles | Quarterly |
| Non-production privileged roles | Every 6 months |
| Guest privileged access | Monthly or quarterly |

Review decisions should consider:

- recent activation history;
- current employment and team membership;
- role-owner confirmation;
- unused eligibility;
- scope appropriateness;
- conflicts of interest;
- incident history;
- licence population.

## 22. Emergency access without weakening PIM

Emergency accounts should not be normal operational shortcuts.

Controls:

- at least two cloud-only accounts;
- permanent Global Administrator assignment;
- no mailbox or productivity use;
- long, independently protected credentials or approved phishing-resistant methods;
- exclusion only from policies that could prevent emergency access;
- sign-in alerts routed to security operations;
- credentials stored through a controlled break-glass process;
- quarterly sign-in tests;
- immediate incident process after any use.

### Test procedure

```text
1. Open approved emergency-access test change.
2. Confirm monitoring team is ready.
3. Sign in with one emergency account.
4. Confirm access to the Entra admin center.
5. Confirm expected alert generation.
6. Sign out and revoke sessions if required.
7. Rotate credentials according to procedure.
8. Close test with evidence.
```

Do not test both emergency accounts simultaneously. Preserve one fallback path.

## 23. Common failure modes

### Failure 1: all approvers are unavailable

**Symptom:** Critical work waits indefinitely.

**Fix:** Configure multiple approvers, out-of-hours coverage and an emergency process.

### Failure 2: approval is required but no safe approval path exists

This can lock out the tenant when all Privileged Role Administrators and Global Administrators are eligible, none are active and no explicit approvers are configured.

**Fix:** Configure emergency access and explicit approvers before removing permanent active access.

### Failure 3: MFA does not prompt on every activation

**Cause:** The session already contains an acceptable MFA claim.

**Fix:** Use Conditional Access authentication context, an authentication strength and sign-in frequency set to Every time when operationally justified.

### Failure 4: activated role works from another device

**Cause:** Activation policy controls the activation event, not necessarily every later use of the activated role.

**Fix:** Add a second Conditional Access policy targeting activated directory roles or eligible users.

### Failure 5: Azure child scopes have different PIM settings

**Cause:** Azure resource PIM settings are per role and per resource, with no automatic child-scope inheritance.

**Fix:** Maintain a policy inventory and drift report across management groups, subscriptions and resource groups.

### Failure 6: ticket fields create false confidence

**Cause:** PIM records ticket information but does not inherently validate the ticket in ServiceNow or Jira.

**Fix:** Use approver validation, automation or a custom extension where available and properly licensed.

### Failure 7: users activate broad roles for narrow tasks

**Fix:** build task-to-role runbooks and custom roles with constrained actions.

### Failure 8: legacy API automation breaks

Older `/beta/privilegedAccess` APIs are deprecated. Do not build new automation using retired or deprecated endpoints. Use Microsoft Graph schedule request APIs for Entra roles and Groups, and Azure Resource Manager APIs for Azure resources.

## 24. Testing plan

Before production rollout, test these scenarios:

| Test | Expected result |
|---|---|
| Eligible user activates low-risk role | Activation succeeds within configured duration |
| User omits justification | Request blocked where justification is required |
| User enters missing ticket | Request blocked where ticket is required |
| User requests Tier 0 role | Approval request generated |
| Approver denies request | Role remains inactive and audit entry exists |
| User uses noncompliant device | Activation blocked by Conditional Access |
| User activates second role within 10 minutes | Behaviour matches documented reauthentication window |
| Activation expires | Permissions removed after configured duration |
| User manually deactivates | Temporary role removed before expiry |
| Emergency account signs in | Access works and urgent alert fires |
| Azure child resource group | Correct policy verified explicitly |
| Policy is changed | Monitoring detects configuration drift |

## 25. Operational KPIs

Track outcomes, not only configuration:

```text
Standing privilege ratio
= permanently active privileged assignments / all privileged assignments

Eligible activation rate
= successful activations / eligible users

Approval SLA
= median time from request to decision

Privileged access utilisation
= users activating role in review period / users eligible for role

Unused eligibility rate
= eligible assignments with no activation in 90 days / all eligible assignments

High-risk activation count
= Tier 0 activations per month

PIM policy coverage
= privileged roles with approved baseline policy / privileged roles in scope
```

Suggested targets:

- standing privilege ratio below 5%, excluding justified emergency or service scenarios;
- 100% of Tier 0 roles covered by an approved PIM policy;
- 100% of privileged activations logged to the SIEM;
- zero single-approver dependencies for critical roles;
- quarterly review completion above 95%;
- unused eligibility remediated within 30 days of review.

## 26. 30-day deployment plan

### Days 1–5: discover

- export Entra and Azure privileged assignments;
- identify role owners;
- classify Tier 0, Tier 1 and Tier 2 roles;
- identify licence populations;
- validate emergency access;
- select a pilot group.

### Days 6–10: design

- approve role policy matrix;
- define activation durations;
- define authentication contexts;
- select approvers and backups;
- define ticket convention;
- design notifications and Sentinel alerts.

### Days 11–15: pilot

- onboard Security Reader, selected workload Contributor and one noncritical administrator role;
- test MFA, approval, expiry and audit;
- test user and approver experience;
- measure approval times;
- remediate lockout risks.

### Days 16–20: protect high-impact roles

- onboard Security Administrator;
- onboard subscription Owner and User Access Administrator;
- deploy Tier 0 authentication context;
- test Conditional Access activation and usage policies;
- test emergency access.

### Days 21–25: automate and monitor

- export PIM schedule requests;
- create baseline drift report;
- connect audit logs to Sentinel;
- create after-hours and Tier 0 alerts;
- document operational runbooks.

### Days 26–30: transition to operations

- remove unjustified permanent active assignments;
- launch access reviews;
- publish KPI dashboard;
- train administrators and approvers;
- obtain security and service-owner sign-off;
- schedule the first monthly governance review.

## 27. Production checklist

### Identity and licensing

- [ ] PIM users, approvers and reviewers are correctly licensed
- [ ] Daily accounts and admin accounts are separated where required
- [ ] Guest privileged users are explicitly reviewed
- [ ] Workload identities are governed separately

### Emergency access

- [ ] At least two emergency access accounts exist
- [ ] Emergency accounts are monitored
- [ ] Quarterly tests are scheduled
- [ ] Credentials and procedures are independently recoverable

### Policy

- [ ] Tier 0 activations are short and protected
- [ ] At least two approvers cover critical roles
- [ ] Justification is required
- [ ] Ticket information is required where appropriate
- [ ] Assignment expiration is configured
- [ ] Permanent active assignments are exceptions

### Conditional Access

- [ ] Authentication context exists
- [ ] Policy is enabled, not only report-only
- [ ] Phishing-resistant authentication is considered for critical roles
- [ ] Activation and role usage are both protected
- [ ] Exclusions are documented and minimal

### Monitoring

- [ ] Audit logs are retained
- [ ] SIEM receives PIM events
- [ ] Tier 0 activation alerts are active
- [ ] Emergency-account alerts are active
- [ ] Policy changes are monitored
- [ ] Access reviews are scheduled

## Final takeaways

A strong PIM implementation is not measured by the number of eligible assignments. It is measured by the reduction of standing privilege, the quality of activation controls, the resilience of the approval process and the ability to detect and investigate every privileged action.

The most important design principles are:

1. **Use eligible access by default and active access by exception.**
2. **Apply the narrowest role and scope that supports the task.**
3. **Protect activation and subsequent role use with Conditional Access.**
4. **Keep Tier 0 activation short, strongly authenticated and approved.**
5. **Design explicit approver redundancy and emergency access before removing standing roles.**
6. **Treat Azure resource policies as per-role, per-scope configurations, not inherited settings.**
7. **Automate inventory, reporting and drift detection, but retain change control for critical policy changes.**
8. **Review eligibility regularly and remove access that is not used or justified.**
9. **Use current Microsoft Graph and Azure Resource Manager APIs, not deprecated PIM endpoints.**
10. **Measure standing privilege, approval time, unused eligibility and policy coverage as operational KPIs.**

PIM is one of the highest-value identity security controls in a Microsoft cloud environment. Implemented carefully, it converts privilege from a permanent account property into a short-lived, justified and observable business process.

## Official references

- [Microsoft Entra licensing](https://learn.microsoft.com/en-us/entra/fundamentals/licensing)
- [Microsoft Entra PIM documentation](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/)
- [Configure Microsoft Entra role settings in PIM](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-how-to-change-default-settings)
- [Configure Azure resource role settings in PIM](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-resource-roles-configure-role-settings)
- [Microsoft Graph role assignment schedule requests](https://learn.microsoft.com/en-us/graph/api/rbacapplication-post-roleassignmentschedulerequests?view=graph-rest-1.0)
- [Microsoft Graph role assignment request listing](https://learn.microsoft.com/en-us/graph/api/rbacapplication-list-roleassignmentschedulerequests?view=graph-rest-1.0)
