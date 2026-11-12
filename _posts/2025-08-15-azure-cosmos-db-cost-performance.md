---
layout: post
title: "Azure Cosmos DB cost and performance: RU/s, partitioning, indexing and reserved capacity"
date: 2025-08-15 09:00:00 +0200
categories: [Data, Azure]
tags: [Azure, Data, Architecture, Operations, Troubleshooting]
description: "Optimise throughput without creating hot partitions."
---

## Cost drivers
Cosmos DB cost depends on capacity mode, provisioned or autoscale throughput, consumed RU/s, regions, storage, backups, network and optional features.

Start with access patterns and partition key. A poor partition key causes hot partitions that cannot be fixed by adding unlimited global throughput.

## Diagnostics
Review throttled requests, normalised RU consumption, partition-key range distribution and query metrics.

```csharp
using FeedIterator<MyItem> iterator = container.GetItemQueryIterator<MyItem>(query);
while (iterator.HasMoreResults)
{
    FeedResponse<MyItem> page = await iterator.ReadNextAsync();
    Console.WriteLine($"RU charge: {page.RequestCharge}");
}
```

## Optimisation

- project only required fields;
- avoid cross-partition queries when possible;
- tune indexing paths;
- batch writes;
- use autoscale for variable predictable peaks;
- use serverless for suitable intermittent workloads;
- use reserved capacity only against consistent throughput.

## Workarounds

- **429 throttling:** use SDK retry behaviour, inspect hot partitions and capacity before simply increasing RU/s.
- **High RU query:** inspect query metrics and index utilisation; redesign query or index.
- **Autoscale minimum seems expensive:** consolidate containers where isolation requirements allow and size max RU/s correctly.
- **Reserved capacity waste:** it is use-it-or-lose-it for matching throughput. Buy against stable hourly usage.
- **Multi-region cost surprise:** each additional write/read region changes cost and consistency design.

## Official references

- [Plan and manage Cosmos DB costs](https://learn.microsoft.com/azure/cosmos-db/plan-manage-costs)
- [Cosmos DB reserved capacity](https://learn.microsoft.com/azure/cosmos-db/reserved-capacity)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
