---
layout: post
title: "Azure migration runbook: discovery, landing zone, cutover and rollback"
date: 2025-09-19 09:00:00 +0200
categories: [Migration, Azure]
tags: [Azure, Migration, Architecture, Operations, Troubleshooting]
description: "A practical server and application migration method."
---

## Migration is an application change
Inventory servers, databases, interfaces, certificates, DNS, identities, backup, monitoring, batch jobs and operational ownership. Group resources into application waves based on dependencies.

## Stages

```text
Discover → Assess → Design target → Remediate → Replicate → Test → Cut over → Hypercare → Optimise
```

## Cutover checklist

- freeze approved changes;
- confirm replication health;
- lower DNS TTL in advance;
- stop or quiesce writes where required;
- take final backup;
- perform final replication;
- start target dependencies in order;
- update DNS/routing;
- execute technical and business validation;
- monitor during hypercare;
- retain rollback window.

## Workarounds

- **Server starts but app fails:** missing DNS, certificates, service accounts and firewall dependencies are common. Validate application flows, not ping only.
- **IP address must remain:** use DNS abstraction or planned reconfiguration; recreating on-premises addressing in Azure can damage network design.
- **Database delta takes too long:** rehearse final sync and measure change rate.
- **Rollback is vague:** define exact decision time, data reconciliation and reverse-change steps.
- **Costs rise after migration:** rightsize after stable observation and remove migration staging resources.

## Validation script example

```powershell
$tests = @(
  @{Name='API'; Host='api.contoso.com'; Port=443},
  @{Name='SQL'; Host='sql.internal.contoso.com'; Port=1433}
)
$tests | ForEach-Object {
  Test-NetConnection $_.Host -Port $_.Port |
    Select-Object @{n='Test';e={$_.ComputerName}}, RemotePort, TcpTestSucceeded
}
```

## Official reference

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
