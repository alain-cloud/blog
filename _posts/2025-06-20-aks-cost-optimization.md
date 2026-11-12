---
layout: post
title: "AKS cost optimisation without breaking reliability"
date: 2025-06-20 09:00:00 +0200
categories: [FinOps, Azure]
tags: [Azure, FinOps, Architecture, Operations, Troubleshooting]
description: "Requests, autoscaling, node pools, Spot and observability."
---

## Start with workload economics
AKS cost includes nodes, disks, load balancers, public IPs, outbound traffic, monitoring and dependent services. Kubernetes requests drive scheduling, so inflated requests create hidden overprovisioning.

```bash
kubectl top nodes
kubectl top pods -A --containers
kubectl get pods -A -o json | jq '.items[] | {ns:.metadata.namespace,name:.metadata.name,requests:.spec.containers[].resources.requests}'
```

## Optimisation sequence

1. measure actual CPU and memory;
2. correct requests and limits;
3. add HPA or KEDA where relevant;
4. enable cluster autoscaler;
5. separate system, general and specialised workloads;
6. evaluate reservations or savings plans only for stable baseline compute.

```bash
az aks nodepool update --resource-group rg --cluster-name aks --name userpool   --enable-cluster-autoscaler --min-count 2 --max-count 12
```

## Spot node pools
Use Spot for interruption-tolerant jobs, CI runners and batch processing. Add taints and tolerations so critical workloads cannot land there accidentally.

```bash
az aks nodepool add --resource-group rg --cluster-name aks --name spotpool   --priority Spot --eviction-policy Delete --spot-max-price -1   --node-taints kubernetes.azure.com/scalesetpriority=spot:NoSchedule
```

## Workarounds

- **Autoscaler does not scale down:** check PDBs, local storage, safe-to-evict annotations, daemonsets and pods with restrictive affinity.
- **HPA does not react:** metrics-server data, requests and target metric configuration must be valid.
- **Too many node pools:** each pool adds operational overhead and can fragment capacity. Create a pool for a real scheduling or lifecycle boundary.
- **Savings commitment covers waste:** rightsize before purchasing commitments.
- **Log cost explodes:** tune Container Insights collection and retention, and avoid duplicate ingestion.

## Reliability guardrail
Never optimise below the minimum replicas, zone spread or surge capacity required by the service objective. Cost per transaction and unused capacity are better metrics than raw monthly cost alone.

## Official references

- [AKS node pools](https://learn.microsoft.com/azure/aks/virtual-machines-node-pools)
- [AKS node OS autoupgrade](https://learn.microsoft.com/azure/aks/auto-upgrade-node-os-image)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
