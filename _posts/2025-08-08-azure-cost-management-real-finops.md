---
layout: post
title: "Azure Cost Management and FinOps: allocation, anomalies, commitments and realised savings"
date: 2025-08-08 09:00:00 +0200
categories: [FinOps, Azure]
tags: [Azure, FinOps, Architecture, Operations, Troubleshooting]
description: "A defensible method for cloud cost governance."
---

## Separate four numbers

```text
Identified opportunity
Approved saving
Implemented saving
Realised invoice reduction
```

Do not report Advisor recommendations as realised savings. Establish a normalised baseline and account for business growth, price changes, credits and currency.

## Cost allocation
Use subscriptions, resource groups, tags and cost allocation rules. Measure tag compliance by spend as well as resource count.

```bash
az graph query -q "Resources
| extend App=tostring(tags.Application), CostCentre=tostring(tags.CostCentre)
| where isempty(App) or isempty(CostCentre)
| project subscriptionId, resourceGroup, name, type, App, CostCentre" --first 5000
```

## Commitment order
Rightsize and remove waste before purchasing Reservations or Savings Plans. Model conservative, balanced and aggressive coverage. Monitor utilisation and coverage separately.

## Workarounds

- **Costs appear late:** Cost Management data is not real-time. Use operational metrics for immediate controls and cost data for financial reporting.
- **Tag historical data is missing:** tags generally do not retroactively rewrite past cost allocation. Maintain change dates and allocation rules.
- **Reservation utilisation is high but bill remains high:** utilisation does not show overall coverage or unmatched PAYG growth.
- **Automatic renewal preserves wrong commitment:** re-evaluate baseline before allowing renewal.
- **Shared services cannot be tagged fairly:** allocate by agreed drivers such as traffic, ingestion, users or protected resources.

## Official references

- [Well-Architected component cost optimisation](https://learn.microsoft.com/azure/well-architected/cost-optimization/optimize-component-costs)
- [Savings plan renewal](https://learn.microsoft.com/azure/cost-management-billing/savings-plan/renew-savings-plan)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
