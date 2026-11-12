---
layout: post
title: "Azure landing zones in production: management groups, policy, RBAC and platform automation"
date: 2025-05-09 09:00:00 +0200
categories: [Architecture, Azure]
tags: [Azure, Architecture, Architecture, Operations, Troubleshooting]
description: "Landing zones, governance and a safe canary hierarchy."
---

## What a landing zone actually solves
An Azure landing zone is the governed platform foundation on which workload subscriptions are placed. It is not a single template and it is not the application environment itself. The platform normally combines management-group hierarchy, subscription organisation, identity, network connectivity, policy, logging, security and deployment automation.

A practical hierarchy is:

```text
Tenant Root Group
├── Platform
│   ├── Identity
│   ├── Connectivity
│   └── Management
├── Landing Zones
│   ├── Corp
│   └── Online
├── Sandbox
└── Decommissioned
```

Keep the hierarchy shallow. A management group should exist because it needs a distinct policy or access boundary, not because the organisation chart has another layer.

## Production design decisions

### Subscription vending
Create subscriptions through an automated request with owner, cost centre, workload name, environment, data classification, network model and business criticality. Apply a standard baseline before handing the subscription to the workload team.

### Policy strategy
Start with `audit` and `deployIfNotExists`, measure impact, remediate, then move selected controls to `deny`. Test platform changes in a canary management-group hierarchy before production.

```bash
az deployment mg what-if   --management-group-id mg-platform-canary   --location westeurope   --template-file main.bicep   --parameters environment=canary
```

### RBAC
Prefer group-based assignments, PIM eligibility for privileged roles and narrow scopes. Avoid broad permanent Owner assignments. Separate platform operators from workload operators.

## Workarounds and failure modes

- **Policy blocks deployment unexpectedly:** inspect the policy evaluation details, exemption scope and alias used. Use a time-bound exemption instead of disabling the initiative.
- **Policy remediation has no identity:** assign a managed identity to `deployIfNotExists` or `modify` assignments and grant only required remediation permissions.
- **Hierarchy changes are risky:** deploy a parallel canary hierarchy and test policy, custom roles and RBAC there first.
- **Subscription move fails:** check inherited policy, role assignments, resource-provider constraints and tenant ownership before moving.

## Baseline validation

```bash
az account management-group list --output table
az policy assignment list --disable-scope-strict-match --output table
az role assignment list --all --include-inherited --output table
```

## Operational checklist

- version platform code in Git;
- require pull-request review and `what-if` evidence;
- use deployment stacks or controlled state management;
- export policy compliance daily;
- maintain named platform owners and emergency procedures;
- test management-group changes before production.

## Official references

- [Azure landing zone testing approach](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/enterprise-scale/testing-approach)
- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
