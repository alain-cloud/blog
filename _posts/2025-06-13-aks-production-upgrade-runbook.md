---
layout: post
title: "AKS production upgrades: a runbook for zero-surprise Kubernetes maintenance"
date: 2025-06-13 09:00:00 +0200
categories: [AKS, Azure]
tags: [Azure, AKS, Architecture, Operations, Troubleshooting]
description: "Version planning, surge, PDBs, blue-green and rollback boundaries."
---

## Before upgrading

```bash
az aks get-upgrades --resource-group rg-aks-prod --name aks-prod --output table
kubectl get pdb -A
kubectl get nodes -o wide
kubectl get pods -A --field-selector=status.phase!=Running
```

Inventory deprecated APIs, admission controllers, CSI drivers, ingress controllers, service meshes and self-managed add-ons. Validate the target version in development and staging.

## Rolling upgrade controls
Use max surge to create temporary capacity. Ensure subscription quota and subnet IP space can accommodate surge nodes.

```bash
az aks nodepool update   --resource-group rg-aks-prod   --cluster-name aks-prod   --name userpool   --max-surge 33%
```

A Pod Disruption Budget can protect availability, but an impossible PDB can block node drain.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: api-pdb }
spec:
  minAvailable: 2
  selector:
    matchLabels: { app: api }
```

Make sure replicas exceed `minAvailable` during maintenance.

## Blue-green strategy
Use two clusters for very low downtime or high-risk version jumps. Deploy to green, validate workloads, shift traffic through Azure Front Door, Traffic Manager or controlled DNS, then retain blue for an agreed rollback window.

A Kubernetes control-plane downgrade is generally not your rollback plan. Rollback means preserving the previous cluster or redeploying from infrastructure and application code.

## Workarounds

- **Drain blocked:** inspect PDBs, standalone pods, local storage and finalizers. Fix the workload instead of force-deleting blindly.
- **Nodes cannot surge:** check regional VM capacity, quota and subnet IP exhaustion. Temporarily select a supported alternate SKU if architecture permits.
- **Deprecated API breaks deployment:** scan manifests before upgrade and update API versions.
- **Ingress change causes outage:** test certificates, health probes, host headers and external-dns behaviour on green.
- **Stateful workload risk:** validate volume attachment, topology constraints, backup and restore separately.

## Future support consideration
Keep clusters on supported versions and configure both Kubernetes and node OS upgrade strategies. Platform-initiated upgrades after support grace periods can cause disruption.

## Official references

- [AKS production upgrade strategies](https://learn.microsoft.com/azure/aks/aks-production-upgrade-strategies)
- [How AKS upgrades work](https://learn.microsoft.com/azure/aks/upgrade-conceptual)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
