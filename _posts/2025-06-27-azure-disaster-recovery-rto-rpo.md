---
layout: post
title: "Azure disaster recovery: turning RTO and RPO into a tested recovery design"
date: 2025-06-27 09:00:00 +0200
categories: [Reliability, Azure]
tags: [Azure, Reliability, Architecture, Operations, Troubleshooting]
description: "HA versus DR, recovery tiers and exercises."
---

## Define objectives first
RTO is the maximum tolerated recovery time. RPO is the maximum tolerated data loss. Neither should be selected by IT alone. Business impact, dependencies and cost determine the target.

| Tier | Example RTO | Example RPO | Typical pattern |
|---|---:|---:|---|
| Tier 0 | minutes | near-zero | active-active or hot standby |
| Tier 1 | <4 hours | <1 hour | active-passive warm standby |
| Tier 2 | <24 hours | daily | redeploy and restore |

These are illustrative tiers, not guarantees.

## Dependency map
Include identity, DNS, certificates, secrets, network, images, code repositories, pipelines, data stores, monitoring and third-party services. A replicated VM is not a recoverable application if DNS, Key Vault or database recovery is missing.

## Recovery strategy

```text
Detect → Declare → Contain → Fail over → Validate → Communicate → Operate degraded → Fail back → Review
```

Use infrastructure as code to rebuild regional resources. Replicate state with service-native mechanisms. Store runbooks where they remain available during an Azure or identity incident.

## Exercise design

- tabletop every quarter for critical services;
- technical failover at least annually, more frequently for Tier 0;
- evidence RTO/RPO achieved;
- test degraded mode and failback;
- document manual steps and elapsed time.

## Workarounds

- **Failover works but clients cannot connect:** validate DNS TTL, private DNS, certificates and firewall paths before exercise.
- **Backup exists but restore exceeds RTO:** measure restore throughput and consider snapshots, warm replicas or data partitioning.
- **Secondary region has no quota:** reserve or request quota before disaster.
- **Secrets differ by region:** define replicated or independently recoverable secret strategy.
- **Runbook depends on same tenant:** retain emergency access and offline-approved procedures.

## Official references

- [Disaster recovery recommendations for Azure data platforms](https://learn.microsoft.com/azure/architecture/data-guide/disaster-recovery/dr-for-azure-data-platform-recommendations)
- [Reliability testing strategy](https://learn.microsoft.com/azure/well-architected/reliability/testing-strategy)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
