---
layout: post
title: "Sustainable Azure engineering: measure emissions, reduce waste, preserve reliability"
date: 2024-10-18 09:00:00 +0100
last_modified_at: 2026-11-13 09:00:00 +0100
categories: [Sustainability, Azure]
tags: [Azure, Green Software, Carbon Optimization, Well-Architected, FinOps, Reliability]
description: "A practical engineering method for measuring Azure workload emissions and improving compute, storage, network, data, and operational efficiency without weakening service objectives."
---

> **Updated 13 November 2026.** Azure Carbon Optimization data is monthly and retained for a limited period. Microsoft has announced retirement of the Emissions Impact Dashboard for Azure on 31 March 2027; export historical data and validate current tooling before building reporting around it.

Sustainable cloud engineering is the practice of delivering the required business outcome with less unnecessary resource consumption over the workload lifecycle. It is not simply a cost-cutting exercise, and it is not achieved by moving a system to Azure and assuming the environmental impact has been solved. Cloud providers operate and improve shared infrastructure, but application architecture determines how much compute, storage, network, and operational work a workload continuously asks that infrastructure to perform.

The most reliable starting point is to reduce waste that is visible in both the bill and the architecture: idle resources, oversized capacity, duplicated data, chatty calls, unnecessary logs, and infrastructure that stays active without serving a business need. Then measure emissions and workload efficiency, balance improvements against service objectives, and build the decisions into normal engineering governance. Microsoft's [Azure Well-Architected sustainability guidance](https://learn.microsoft.com/azure/well-architected/sustainability/sustainability-application-design) treats sustainability as a workload design concern across the lifecycle, with explicit trade-offs against performance, reliability, security, and compliance.

## 1. Define what you are trying to improve

Before setting a target, agree on the unit of work and the boundary of the system. “Reduce cloud carbon by 20%” is difficult to act on if no one knows whether the number covers one application, one subscription, shared platform services, non-production environments, or only direct compute consumption.

Define a boundary that matches an accountable team. For an application, include the resources the team operates and describe how shared services are allocated. If a central network, identity, or observability platform serves many applications, either report it as a shared component or allocate it using a documented driver such as request volume or protected resource count. Do not pretend an allocation is a direct measurement; label it as an allocation and keep the method stable enough to compare trends.

Choose a functional unit that reflects value delivered, such as emissions per 1,000 API requests, per completed order, per processed terabyte, or per training job. Absolute emissions remain important for total impact, but intensity shows whether the system is becoming more efficient as usage grows. Track both. If total emissions rise because usage doubled while emissions per transaction fell, that is a different engineering story from emissions rising while business throughput stayed flat.

For application-level measurement, the Software Carbon Intensity (SCI) specification provides a useful conceptual model:

```text
SCI = ((E × I) + M) / R
```

Where:

- **E** is the energy consumed by the software system.
- **I** is the carbon intensity of the electricity supply over the relevant time and location.
- **M** is the allocated embodied carbon of the hardware.
- **R** is a functional unit representing useful work delivered.

The formula helps teams distinguish several kinds of improvement: reduce energy per unit of work, shift flexible work to a cleaner time or location when reliable data and architecture allow it, reduce unnecessary hardware demand, or improve the amount of useful work delivered. It is not a magic calculator. Many teams cannot directly measure every variable, so they start with documented proxies and improve measurement quality over time. Microsoft's [SCI example for Azure applications](https://learn.microsoft.com/azure/architecture/example-scenario/apps/measure-azure-app-sustainability-sci-score) describes this formula and the use of Azure Carbon Optimization data as a monthly baseline.

## 2. Use the right measurement source for the question

Azure Carbon Optimization helps teams review emissions across subscriptions, resource groups, resources, resource types, and locations. It also provides utilization-based recommendations, such as reviewing idle or underused resources, with estimated emissions and cost impact. The service publishes emissions monthly; the latest month's data becomes available after the calculation period. It is useful for baselining and validating longer-term changes, not for minute-by-minute workload scheduling.

Microsoft documents a 12-month retention period for Azure Carbon Optimization data. Export the data regularly if the organization needs a longer reporting history. Microsoft has also announced that the Emissions Impact Dashboard for Azure, a Power BI solution, will be retired on **31 March 2027**, with Azure Carbon Optimization as the recommended tool for Azure emissions tracking. The transition matters for anyone who has embedded the older dashboard into a sustainability reporting process. Review the [Carbon Optimization overview](https://learn.microsoft.com/azure/carbon-optimization/overview), [emissions data guide](https://learn.microsoft.com/azure/carbon-optimization/view-emissions), and [CSV export guide](https://learn.microsoft.com/azure/carbon-optimization/export-data) for current data availability and export behavior.

Use several measures together:

| Measure | What it helps answer | Important limitation |
|---|---|---|
| Monthly Azure emissions | Which subscriptions, resource groups, resource types, or locations contribute most? | Aggregated and delayed; not a live signal |
| Cost and utilization | Which resources appear idle or oversized? | A low-utilization resource may still meet a resilience or business requirement |
| Application throughput | How much useful work did the system deliver? | Requires a stable definition of successful work |
| CPU, memory, and queue metrics | Is capacity being used efficiently? | Utilization alone does not reveal energy or business value |
| Software Carbon Intensity proxy | Is environmental impact per functional unit improving? | Proxy quality and boundary assumptions must be documented |
| Reliability and user experience | Did efficiency changes preserve service quality? | Must be interpreted with workload-specific objectives |

A carbon report should show its assumptions. Document what resource types are included, what shared-resource allocation method is used, which months are compared, and which workload volume is used for normalization. If the measurement methodology changes, mark the break in the time series. Otherwise, a reporting improvement can look like an emissions improvement even when the system did not change.

Do not confuse illustrative equivalents with a precise business metric. Carbon calculators sometimes present emissions as everyday equivalents to make a large number understandable. These can help communicate scale, but engineering prioritization should use the original emissions data, the workload boundary, and the assumptions behind the calculation.

## 3. Establish a baseline and identify high-value work

A practical baseline includes at least one representative business cycle. Seasonal commerce, end-of-month processing, annual reporting, and batch workloads can have very different resource profiles. One quiet week is not enough evidence to resize a service that supports a quarterly peak.

Start with a workload inventory:

1. Map subscriptions and resource groups to applications and owners.
2. Identify shared services and agree how they are represented.
3. Record the primary business output and operational objectives.
4. Collect cost, emissions, utilization, throughput, and service-quality data over an appropriate period.
5. Mark planned architecture changes, incidents, and exceptional load events.
6. Rank opportunities by expected emissions reduction, cost impact, engineering effort, and operational risk.

Prioritize changes that remove waste without reducing useful service. Shutting down an abandoned development environment is often safer than reducing production redundancy. Removing an unused public IP or orphaned disk may be straightforward after ownership validation. Resizing a production database requires performance evidence and an agreed rollback plan. The Azure Carbon Optimization recommendation is a lead for investigation, not an authorization to change a workload.

Keep a decision log for each significant change:

```text
Workload:
Resource or service:
Baseline period:
Observed inefficiency:
Proposed change:
Expected cost/emissions effect:
Reliability and security impact:
Owner and approval:
Canary / rollout method:
Rollback condition:
Measurement period and result:
```

This record makes it possible to distinguish verified outcomes from estimates. Report potential savings, approved opportunities, implemented changes, measured reductions, and avoided future consumption separately. An implementation estimate is not a measured result, and an avoided-cost projection should not be presented as a reduction already visible in emissions data.

## 4. Improve compute efficiency without buying fragility

Compute is often the most visible source of waste, but utilization must be interpreted in context. Low average CPU may indicate an oversized VM, or it may be the deliberate cost of a low-latency peak, a recovery environment, or a licensed appliance that must remain available. Review CPU, memory, disk, network, concurrency, and application latency across both normal and peak periods.

### Virtual machines

For virtual machines, combine Azure Advisor recommendations with Azure Monitor metrics, workload traces, and owner confirmation. Check whether memory is the limiting resource; CPU-only observations can lead to an unsafe resize. Identify burst patterns and scheduled maintenance. If the VM is used only during business hours, verify startup dependencies, data consistency, patch windows, and the effect of shutdown on service-level objectives before scheduling it.

For non-production systems, agree a schedule with the developers and test teams. Use resource tags and automation with a clear opt-out path for environments that support overnight testing, long-running migrations, or release work. Measure schedule compliance and exceptions so that automated shutdown does not become a source of emergency workarounds.

### Containers and Kubernetes

For AKS, inspect requests and limits, node-pool utilization, pending pods, autoscaler behavior, and workload distribution. Overly generous pod requests can force additional nodes even when the actual workload uses little capacity. Requests that are too low create contention and reliability problems. Use observed percentiles and representative load tests to establish values, then monitor throttling, restarts, and latency after rollout.

Right-size system and user node pools separately. Use autoscaling to match demand where workload startup time and disruption budgets permit it. Scale-to-zero can be appropriate for non-production or event-driven workloads, but not every service can tolerate cold starts or delayed capacity. Spot VMs can improve utilization for interrupt-tolerant batch jobs; they are a poor fit for work that cannot checkpoint or retry safely.

### App platforms and serverless

Managed platforms can improve utilization by sharing infrastructure and reducing the need to manage idle servers, but serverless does not guarantee low emissions. An inefficient function that wakes frequently, processes the same payload repeatedly, or performs excessive retries can consume more resources than a well-sized long-running service. Profile the application and choose the hosting model that meets the workload's latency, throughput, availability, and scaling requirements with the least waste.

Test efficiency under realistic load. A configuration that minimizes resources at idle may trigger aggressive scale-out, repeated cold starts, or high retry amplification during a burst. Track resource consumption per unit of successful business work, not just per instance.

## 5. Reduce application and data work

The most effective optimization may be to avoid computation entirely. Review the request path and data lifecycle for duplicated work:

- Cache stable results when consistency requirements permit.
- Batch small operations when batching does not violate latency objectives.
- Avoid polling when an event-driven design can safely notify consumers.
- Use idempotency and bounded retries to prevent failure loops.
- Filter data early so downstream stages process only relevant records.
- Reuse a validated model or dataset instead of retraining or reprocessing without a business reason.
- Remove obsolete data after retention and legal requirements are satisfied.

For data platforms, measure bytes read and written, query duration, job retries, data scanned per output, and storage by access pattern. Partitioning and file sizing can reduce repeated scans. Compression and deduplication can reduce storage and transfer, but may add CPU overhead; test the full pipeline rather than assuming each compression choice is beneficial. Keep hot storage for data that needs frequent low-latency access and evaluate lifecycle policies for older data. Align retention with legal, audit, and recovery requirements before deletion or archival changes.

Telemetry itself has a footprint. Broad diagnostic settings can create large volumes of low-value logs. Define what each data collection rule collects, which tables need long retention, and which fields should be transformed or dropped. Keep the telemetry necessary for security, incident response, and service objectives; reduce duplicate, noisy, or unused data. Microsoft's [Operational Excellence recommendations for sustainable workloads](https://learn.microsoft.com/azure/well-architected/sustainability/sustainability-operational-excellence-recommendations) call out data lifecycle management and deliberate telemetry collection as part of sustainability governance.

## 6. Design network paths deliberately

Network consumption grows through extra hops, repeated API calls, oversized payloads, cross-region traffic, and unnecessary replication. Reduce chatty interactions where a coarser request or a local cache is safe. Compress payloads when network savings exceed added CPU cost. Use a content delivery network for suitable static or cacheable content, and keep dependent resources in the same region where residency, availability, and latency needs allow.

Cross-region architecture is sometimes essential for recovery, latency, or data residency. It also has an ongoing footprint: replicated data, duplicated compute, health probes, and synchronization traffic continue outside an incident. Make the recovery objective explicit and size standby capacity to the required recovery model. A warm or pilot-light environment may meet the requirement with less always-on capacity than a full active-active deployment, but the team must prove recovery time and data loss objectives through tests.

Avoid equating fewer regions with better sustainability. A region closer to users can reduce network distance and improve response time. A lower-carbon region may increase latency or data-transfer requirements. Evaluate carbon intensity, user location, data residency, service availability, and failure-domain requirements together. For flexible batch jobs, scheduling or regional placement might be practical; for interactive applications, customer experience and resilience often constrain those options.

## 7. Storage and lifecycle are architectural controls

Storage waste is often caused by unclear ownership and retention rather than a lack of cheaper tiers. Inventory snapshots, backups, unattached disks, duplicate datasets, old exports, and diagnostic archives. For each data class, define the purpose, owner, retention, recovery requirement, access frequency, and deletion or archival rule.

Use lifecycle management to move eligible data to cooler tiers or delete it after the approved retention period. Test retrieval time and early deletion or transaction costs for the relevant storage class. Ensure backups and immutable data meet security and recovery policy. A lower-cost tier may not reduce the total footprint if applications repeatedly rehydrate the same data or if copies are retained elsewhere.

Reduce data duplication upstream where possible. Repeated full exports and full database copies consume storage and network resources. Incremental processing, change data capture, and retention-aware snapshots can reduce repeated work, provided they preserve consistency and recovery evidence. For analytics, retain derived outputs only when their reuse or audit value justifies it.

## 8. Balance sustainability with reliability and security

Sustainability improvements must preserve the system's business requirements. Reliability patterns have a footprint, but so do outages and inefficient recovery behavior. Excessive active-active redundancy can keep duplicated capacity and replication running continuously. Under-provisioned recovery capacity can lengthen an outage and trigger repeated retries, data repair, and emergency provisioning. Define recovery time objective (RTO), recovery point objective (RPO), and availability targets before changing redundancy.

Use fault isolation, circuit breakers, throttling, queue-based buffering, graceful degradation, and idempotent retry policies to prevent failures from amplifying work. These patterns can reduce wasted compute during a dependency incident and improve recovery. Test them with realistic failure scenarios and verify the workload still meets its objective. The [Well-Architected reliability recommendations](https://learn.microsoft.com/azure/well-architected/sustainability/sustainability-reliability-recommendations) discuss the sustainability impact of intentional resiliency and stable demand patterns.

Security controls also protect sustainability by preventing malicious or accidental resource consumption. Publicly exposed services can be abused to generate compute load; compromised credentials can create mining workloads or exfiltration traffic. Apply identity least privilege, network controls, policy guardrails, and anomaly detection. Avoid adding redundant security logging without a consumer, but do not drop evidence required for threat detection or investigation. Tune collection based on threat model, legal duties, and operational utility.

## 9. Make region and timing decisions with honest data

Carbon intensity varies by location and time. The SCI model makes this visible through the `I` factor, but the data required for runtime decisions must be sufficiently fresh and aligned to the workload. Azure Carbon Optimization provides monthly aggregated data; it is suitable for a baseline and retrospective analysis, not for choosing a region hour by hour.

For a flexible batch job, a carbon-aware scheduler may defer execution within an agreed service window if it has trustworthy near-real-time carbon intensity data, a defined fallback, and a reliable way to preserve job state. Do not move regulated data to an unapproved region or delay a critical business process because a carbon signal is lower. A carbon-aware policy should specify:

- the eligible regions and data residency constraints;
- the maximum scheduling delay;
- the allowed execution window;
- the source and freshness of carbon-intensity data;
- the fallback when data is unavailable;
- cost and performance guardrails;
- an audit record of the scheduling decision.

Start with a non-critical batch workload and compare outcomes with a fixed schedule. Include additional transfer, storage, and orchestration costs. If carbon savings cannot be measured credibly, prefer a simpler utilization improvement that has clear operational evidence.

## 10. Build sustainability into engineering governance

Sustainability becomes repeatable when it is part of existing design and operating processes. Add a few targeted questions to architecture reviews:

- What unit of useful work will we measure?
- Which resources remain active when demand is low?
- What data is collected and retained, and who uses it?
- Can demand be smoothed or work avoided?
- What reliability or security control constrains the proposed reduction?
- How will we verify the outcome and roll back safely?

At the platform level, use policy to prevent known waste without blocking legitimate architecture. Examples include required owner and environment tags, approved SKU ranges, lifecycle settings for eligible storage, and schedules for non-production resources. Use audit mode before deny or deploy effects when the estate's current configuration is not understood. Make exceptions visible and time-bound.

Measure sustainability alongside service objectives and FinOps outcomes. A balanced scorecard might track:

- total monthly emissions and emissions per functional unit;
- cost per functional unit;
- compute utilization and idle resource age;
- data retained beyond the approved period;
- telemetry volume by workload and purpose;
- availability, latency, RTO, and RPO;
- percentage of optimization opportunities verified after implementation.

Do not set targets that reward teams for reducing a proxy while harming the product. For example, lowering the number of VMs is not a success if latency worsens beyond the objective or recovery no longer works. Record design trade-offs and review them when usage or business needs change. Microsoft's [sustainability operational excellence guidance](https://learn.microsoft.com/azure/well-architected/sustainability/sustainability-operational-excellence-recommendations) recommends workload-level targets, consistent measurement, and governance that reinforces sustainable choices.

## 11. Measure software efficiency with a functional unit

A simple SCI-style trend can make application changes comparable. Suppose an online service processes 10 million successful requests per month. A team estimates its workload's monthly emissions using the available Azure data and divides by successful requests. The resulting value is not an exact physical measurement of each request, but it can serve as a consistent proxy if the resource boundary, allocation, and denominator remain stable.

A measurement record should include:

```text
Workload and owner:
Included subscriptions/resources:
Shared-resource allocation method:
Measurement period:
Emissions source and calculation version:
Functional unit and source:
Reliability/performance measures:
Known gaps and assumptions:
```

If the deployment is split into more subscriptions or an emissions API changes its resource categories, do not silently compare the new numbers with the old series. Recompute the baseline when possible or annotate the methodological change. Use a sample of large contributors to check whether allocation changes make sense.

Where direct energy or carbon measurements are unavailable, use proxies carefully. Resource hours, CPU-seconds, storage volume, data transfer, and cost can help identify inefficient trends, but none is identical to energy or emissions. Cost can change because of pricing, discounts, or licensing without changing resource consumption. A proxy becomes useful when it is transparent and consistently related to the workload's actual behavior.

## 12. A 90-day engineering program

**Weeks 1–2: define scope.** Select a workload, owner, functional unit, and service objectives. Map compute, storage, network, telemetry, and shared resources. Confirm access to emissions data and export requirements.

**Weeks 3–4: baseline and prioritize.** Collect representative operating data, including a peak period if relevant. Review Azure Carbon Optimization and Azure Advisor recommendations. Identify a small set of changes with clear evidence and low operational risk.

**Weeks 5–8: implement controlled changes.** Start with low-risk waste removal, non-production scheduling, or telemetry cleanup. Use a canary for production changes. Capture before-and-after cost, emissions, throughput, and reliability data.

**Weeks 9–10: validate.** Wait for the monthly emissions data where required. Compare the expected and observed outcome. Investigate differences rather than rounding them into a success claim. Confirm that the workload still meets its service objectives.

**Weeks 11–12: institutionalize.** Update architecture standards, policy, service templates, dashboards, and owner workflows. Export emissions history regularly. Publish verified results and unresolved trade-offs.

Use a change acceptance rule: no optimization is complete until it has an owner, a tested implementation, a measured outcome, and an operational rollback or recovery story. For estimates that cannot be measured at resource level, state the limitation and report the proxy as a proxy.

## 13. Common failure modes

| Failure mode | Why it fails | Better practice |
|---|---|---|
| Treating cost savings as emissions measurements | Price changes and discounts can alter cost independently of usage | Report financial and emissions measures separately |
| Resizing from one quiet period | It ignores peak, seasonal, and recovery demand | Use representative windows and load tests |
| Moving all workloads to a “green” region | Residency, latency, availability, and transfer costs can make the design worse | Evaluate constraints and total workload behavior together |
| Disabling telemetry indiscriminately | It can weaken security and incident response | Remove unused or duplicate signals with data-owner approval |
| Using monthly emissions as a real-time scheduler input | The data is not granular enough for hour-by-hour choices | Use it for baselines; obtain an appropriate live data source for runtime decisions |
| Claiming estimated savings as realized | The estimate may not materialize or may be offset elsewhere | Separate identified, implemented, and verified outcomes |
| Reducing redundancy without RTO/RPO evidence | The business may lose recovery capability | Design to explicit objectives and test recovery |
| Setting a target with no functional unit | Teams cannot compare efficiency as usage changes | Define and version a useful-work denominator |

## Production checklist

- [ ] Workload boundary, ownership, shared-service allocation, and functional unit are documented.
- [ ] Azure emissions data is exported regularly if history beyond the service retention period is needed.
- [ ] The retirement of the Emissions Impact Dashboard is reflected in reporting plans.
- [ ] Emissions, cost, utilization, throughput, and reliability are reported as distinct measures.
- [ ] Compute changes use representative telemetry and include a rollback condition.
- [ ] Data lifecycle, backup, and telemetry retention meet security and legal requirements.
- [ ] Network and region changes account for latency, data transfer, and residency.
- [ ] Recovery architecture is sized against tested RTO and RPO rather than assumed peak symmetry.
- [ ] Efficiency changes are measured after implementation and their assumptions are recorded.
- [ ] Sustainability is part of architecture, FinOps, platform policy, and workload review cadences.

## References

- [Azure Well-Architected sustainability guidance](https://learn.microsoft.com/azure/well-architected/sustainability/sustainability-application-design)
- [Azure Carbon Optimization overview](https://learn.microsoft.com/azure/carbon-optimization/overview)
- [View and analyze Azure emissions](https://learn.microsoft.com/azure/carbon-optimization/view-emissions)
- [Export Azure Carbon Optimization data](https://learn.microsoft.com/azure/carbon-optimization/export-data)
- [Reduce carbon emissions in Azure](https://learn.microsoft.com/azure/carbon-optimization/reduce-emissions)
- [Measure Azure app sustainability with the SCI score](https://learn.microsoft.com/azure/architecture/example-scenario/apps/measure-azure-app-sustainability-sci-score)
- [Sustainability reliability recommendations](https://learn.microsoft.com/azure/well-architected/sustainability/sustainability-reliability-recommendations)
- [Sustainability operational excellence recommendations](https://learn.microsoft.com/azure/well-architected/sustainability/sustainability-operational-excellence-recommendations)
- [Sustainability performance efficiency recommendations](https://learn.microsoft.com/azure/well-architected/sustainability/sustainability-performance-efficiency-recommendations)
- [Sustainable AI design for Azure](https://learn.microsoft.com/azure/well-architected/sustainability/sustainable-ai-design)
