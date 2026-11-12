---
layout: post
title: "Azure Site Recovery for Azure VMs: production design, test failover and troubleshooting"
date: 2025-07-04 09:00:00 +0200
categories: [Reliability, Azure]
tags: [Azure, Reliability, Architecture, Operations, Troubleshooting]
description: "Replication, recovery plans and failback."
---

## What ASR does and does not do
Azure Site Recovery replicates supported VM workloads and orchestrates failover. It does not replace application-consistent backup, database-native replication or a complete business continuity plan.

## Implementation flow

```text
Assess support → Prepare target region/zone → Enable replication → Monitor health → Test failover → Document failover → Test reprotect/failback
```

Use a test network isolated from production. Test application dependency order through recovery plans.

```powershell
Get-AzRecoveryServicesAsrReplicationProtectedItem
Get-AzRecoveryServicesAsrJob | Sort-Object StartTime -Descending | Select-Object -First 20
```

## Recovery plan sequencing

```text
Group 1: Domain/DNS dependencies where required
Group 2: Databases and stateful services
Group 3: APIs and application tier
Group 4: Front ends, workers and monitoring
```

Scripts must be idempotent and credentials recoverable during disaster.

## Workarounds

- **Replication health is critical:** inspect churn rate, cache storage, agent/provider status, outbound connectivity and disk support.
- **Test failover has no connectivity:** verify target VNet, NSGs, UDRs, DNS and IP assumptions.
- **Failback takes too long:** test reprotection during exercises and size bandwidth for changed data.
- **Application consistency is missing:** coordinate VSS/application hooks or use database-native recovery.
- **Zone-to-zone limitations:** verify region, disk type and feature support before committing to design.

## Official reference

- [Zone-to-zone disaster recovery for Azure VMs](https://learn.microsoft.com/azure/site-recovery/azure-to-azure-how-to-enable-zone-to-zone-disaster-recovery)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
