---
layout: post
title: "Azure Monitor in production: metrics, logs, alerts, workbooks and cost control"
date: 2025-06-06 09:00:00 +0200
categories: [Operations, Azure]
tags: [Azure, Operations, Architecture, Operations, Troubleshooting]
description: "An actionable observability operating model."
---

## Four telemetry types

- **Metrics** for low-latency numeric time series.
- **Logs** for rich queryable records.
- **Traces** for distributed request flow.
- **Activity Log** for subscription-level control-plane events.

A dashboard is not monitoring. Monitoring requires service-level objectives, actionable alerts, ownership, escalation and tested response.

## Alert design

Prefer symptoms that affect users over raw infrastructure signals. Combine:

- availability and latency alerts;
- saturation alerts;
- dependency failures;
- certificate expiry;
- backup and replication failures;
- abnormal cost or ingestion.

```kusto
AppRequests
| where TimeGenerated > ago(15m)
| summarize Total=count(), Failed=countif(Success == false) by bin(TimeGenerated, 5m)
| extend FailureRate = 100.0 * Failed / Total
| where Total > 100 and FailureRate > 5
```

## Dynamic thresholds with a fallback
Dynamic alerts need enough history. For new workloads, start with conservative static thresholds, then introduce dynamic detection once normal behaviour is understood.

## Log cost control

```kusto
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize BillableGB=sum(Quantity)/1000 by DataType
| order by BillableGB desc
```

Control cost through diagnostic-setting scope, table plans, retention, data collection rules and source-side filtering. Do not collect everything “just in case.”

## Workarounds

- **Alert storm:** add dimensions carefully, aggregation windows, auto-mitigation and suppression rules. Fix root causes rather than silencing alerts globally.
- **No data alert creates false green:** use separate data-absence detection where missing telemetry is itself a failure.
- **KQL query is expensive:** filter early by time, resource and category; project only needed columns; summarise before joins.
- **Dashboard is slow:** reduce workbook query scope, cache where supported and separate executive views from deep diagnostics.
- **Activity Log is missing data:** Activity Log is control plane only. Configure resource logs for data-plane events.

## Official reference

- [Monitor Azure resources with Azure Monitor](https://learn.microsoft.com/azure/azure-monitor/platform/monitor-azure-resource)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
