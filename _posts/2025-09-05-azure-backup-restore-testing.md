---
layout: post
title: "Azure Backup: designing recovery, immutability and restore testing"
date: 2025-09-05 09:00:00 +0200
categories: [Reliability, Azure]
tags: [Azure, Reliability, Architecture, Operations, Troubleshooting]
description: "Backups that can actually meet recovery objectives."
---

## Backup is a recovery product
Success is not “backup job green.” Success is restoring the right data, securely, within RTO and RPO.

Define:

- protected assets and owners;
- backup frequency and retention;
- application consistency;
- vault redundancy;
- immutability and soft delete;
- encryption and key dependencies;
- restore network and identity requirements;
- recurring restore tests.

## Restore test

```text
Select recovery point → restore to isolated target → boot/mount → application validation → malware/security checks → measure time → destroy test target → record evidence
```

## Workarounds

- **Restore requires unavailable Key Vault:** include key recovery and permissions in the runbook.
- **Cross-region restore is assumed:** verify vault redundancy, workload support and regional availability before relying on it.
- **Application-consistent snapshot times out:** inspect extension state, VSS writers or Linux pre/post scripts.
- **Retention cost grows unexpectedly:** model daily, weekly, monthly and yearly points and churn.
- **Ransomware actor deletes backup:** use soft delete, immutability, MUA where applicable, PIM and security alerts.

## Database caution
Use database-native backup when it provides the required consistency and granularity. VM-level backup alone may not satisfy database recovery objectives.

## Official reference

- [Business continuity and backup method considerations](https://learn.microsoft.com/azure/well-architected/oracle-iaas/optimize-business-continuity-disaster-recovery)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
