---
layout: post
title: "Azure FinOps under pressure: from cost shock to sustainable governance"
date: 2026-11-12 09:00:00 +0100
categories: [finops, azure]
tags: [Azure, FinOps, Cost Management, Azure Advisor, governance]
description: "A practical six-week approach to regain Azure cost visibility, capture quick wins and establish a sustainable FinOps operating model."
---

An unexpected Azure bill rarely has a single cause. It is usually the combined result of weak ownership, limited cost visibility, idle resources, oversized services and purchasing decisions that are disconnected from real usage.

This article describes an anonymised intervention in a large Azure environment. The objective was not simply to cut the bill. It was to reduce waste without disrupting production and to establish a model that would prevent the same situation from returning.

> **Publication note:** The figures below must be validated and approved before publication if they originate from a customer engagement.

## The starting point

The customer had three immediate concerns:

- Azure consumption had moved significantly above budget.
- Finance could see the total invoice but not a reliable breakdown by application, team, environment or business unit.
- Several workloads were running continuously, despite uncertain utilisation.

The initial review identified five recurring cost drivers:

1. Idle or underutilised virtual machines, databases and storage resources.
2. Inconsistent tags and unclear cost ownership.
3. Services sized for theoretical peaks rather than observed demand.
4. Non-production resources running outside business hours.
5. Limited use of Azure Reservations, Savings Plans and Azure Hybrid Benefit.

## A three-stage FinOps response

We structured the intervention around **Inform, Optimise and Operate**.

### 1. Inform: establish a reliable cost baseline

Before changing resources, we needed evidence. The first workstream focused on visibility and accountability:

- Use **Azure Cost Management** to analyse spend by subscription, resource group, service, region and resource.
- Create budgets and alerts for material deviations from forecast.
- Introduce a minimum tagging model such as `Application`, `Environment`, `BusinessOwner`, `TechnicalOwner` and `CostCentre`.
- Publish a Power BI dashboard for engineering, finance and business stakeholders.
- Decide where showback is sufficient and where chargeback is required.

A useful baseline should answer four questions: **What are we spending? Who owns it? What business service does it support? Is the cost justified by usage and value?**

### 2. Optimise: capture savings without creating operational risk

The optimisation backlog was prioritised by value, confidence, effort and production risk.

#### Quick wins

- Remove unattached disks, obsolete snapshots and unused public IP addresses after owner validation.
- Schedule shutdown and startup for eligible development and test workloads.
- Review low-utilisation VMs and databases using Azure Monitor metrics and Azure Advisor recommendations.
- Reassess storage redundancy and access tiers against recovery and retention requirements.

#### Commitment-based savings

Reservations and Savings Plans can deliver significant savings, but they should follow workload analysis rather than precede it. We validated:

- stable baseline consumption;
- expected workload lifetime;
- region and service eligibility;
- existing licence entitlements for Azure Hybrid Benefit;
- the operational impact of one-year versus three-year commitments.

#### Elasticity

Where technically appropriate, we aligned capacity with demand through:

- virtual machine scale sets;
- Azure App Service autoscale;
- AKS cluster and workload autoscaling;
- database scaling options;
- scheduled scaling for predictable business cycles.

### 3. Operate: make cost ownership part of cloud operations

Savings disappear when FinOps is treated as a one-off exercise. We therefore introduced:

- a monthly FinOps review with finance, engineering, operations and business owners;
- a tracked optimisation backlog with owner, expected value, deadline and status;
- Azure Policy controls for mandatory tags and approved deployment patterns;
- cost checks in architecture reviews and delivery pipelines;
- quarterly reviews of commitments, forecasts and realised savings.

## Six-week outcome

The intervention reported the following results over six weeks:

- **35% reduction in Azure compute costs** through rightsizing and scaling changes;
- **20% reduction in storage costs** by removing redundant disks and unused snapshots;
- **more than USD 100,000 in savings** through reservations and Azure Hybrid Benefit;
- improved cost visibility through shared dashboards and KPIs;
- a repeatable governance cadence, with no reported production impact.

These figures are context-specific. A credible FinOps case should distinguish **identified savings**, **implemented savings**, **realised invoice reduction** and **avoided future cost**.

## Practical lessons

- **Start with visibility, not discounts.** A commitment applied to waste is still waste.
- **Protect production.** Every optimisation needs an owner, evidence, a rollback path and a maintenance window when relevant.
- **Measure outcomes consistently.** Agree the baseline and savings methodology before reporting benefits.
- **Connect cost to business value.** The cheapest architecture is not automatically the best architecture.
- **Make ownership explicit.** Finance cannot optimise cloud costs alone, and engineering should not make financial commitments in isolation.

## A reusable 30-day action plan

| Period | Focus | Deliverable |
|---|---|---|
| Days 1–5 | Baseline and ownership | Cost map, top cost drivers, data-quality gaps |
| Days 6–10 | Quick wins | Validated removal, shutdown and rightsizing backlog |
| Days 11–20 | Implementation | Approved low-risk changes with tracked savings |
| Days 21–25 | Commitments | Reservation, Savings Plan and Hybrid Benefit assessment |
| Days 26–30 | Governance | Dashboard, review cadence, owners and policy backlog |

## Final thought

FinOps is not a cost-cutting project. It is an operating discipline that helps teams make better technical and financial decisions together. The strongest outcome is not a lower invoice for one month, but a cloud environment where cost, value and accountability remain visible over time.
