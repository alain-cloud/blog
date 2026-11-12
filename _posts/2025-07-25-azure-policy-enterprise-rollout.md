---
layout: post
title: "Azure Policy at enterprise scale: design, remediation, exemptions and rollout"
date: 2025-07-25 09:00:00 +0200
categories: [Governance, Azure]
tags: [Azure, Governance, Architecture, Operations, Troubleshooting]
description: "Guardrails without blocking delivery."
---

## Policy lifecycle

```text
Define requirement → Select alias/effect → Test → Audit → Remediate → Deny → Monitor → Review
```

Organise definitions into initiatives aligned with outcomes such as logging, networking, security baseline and allowed regions. Assign at the highest correct scope, but avoid massive initiatives containing unrelated controls.

## Effects

- `audit` reports noncompliance;
- `deny` blocks a request;
- `modify` changes supported properties;
- `deployIfNotExists` deploys a related resource;
- `auditIfNotExists` checks for related configuration.

## Remediation

```bash
az policy remediation create   --name remediate-diagnostics   --policy-assignment <assignment-id>   --resource-group rg-target
```

Managed identity permissions must match the remediation action.

## Workarounds

- **Existing resource remains noncompliant:** `deployIfNotExists` evaluates but does not automatically remediate historical resources without a remediation task.
- **Policy is not evaluated immediately:** trigger a compliance scan and allow evaluation time.
- **Deny breaks emergency change:** create a documented, narrow, time-bound exemption at correct scope.
- **Tag inheritance overwrites application tags:** use `modify` rules carefully and define source-of-truth precedence.
- **Wrong alias:** inspect available aliases for the resource provider and API version.

## Governance
Keep exemptions with owner, reason, expiry and compensating control. Review expired and permanent exemptions monthly.

## Official reference

- [Azure Policy support and remediation patterns](https://learn.microsoft.com/azure/postgresql/security/security-azure-policy)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
