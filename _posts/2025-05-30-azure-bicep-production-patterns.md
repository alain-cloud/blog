---
layout: post
title: "Bicep for production Azure platforms: modules, what-if, validation and safe deployments"
date: 2025-05-30 09:00:00 +0200
categories: [Automation, Azure]
tags: [Azure, Automation, Architecture, Operations, Troubleshooting]
description: "From reusable modules to safe deployment pipelines."
---

## Repository structure

```text
infra/
├── main.bicep
├── main.bicepparam
├── modules/
│   ├── network.bicep
│   ├── monitoring.bicep
│   └── key-vault.bicep
└── tests/
```

Use modules with explicit inputs and outputs. Pin module versions when using a private registry. Avoid dynamic resource names where stable names are operationally important.

## Safe delivery workflow

```text
Lint → Build → Validate → What-if → Security checks → Approval → Deploy → Smoke test
```

```bash
az bicep build --file infra/main.bicep
az deployment sub validate --location westeurope --template-file infra/main.bicep --parameters infra/prod.bicepparam
az deployment sub what-if --location westeurope --template-file infra/main.bicep --parameters infra/prod.bicepparam
```

## Useful patterns

```bicep
@allowed(['dev','test','prod'])
param environment string

param tags object

var commonTags = union(tags, {
  Environment: environment
  ManagedBy: 'Bicep'
})
```

Use `existing` resources when the deployment consumes but does not own them.

```bicep
resource hubVnet 'Microsoft.Network/virtualNetworks@2024-05-01' existing = {
  scope: resourceGroup(hubSubscriptionId, hubResourceGroup)
  name: hubVnetName
}
```

## Workarounds

- **What-if shows noise:** eliminate runtime-generated values, use stable names and avoid reading properties that are normalised by the provider.
- **Complete mode can delete resources:** prefer incremental deployments and deployment stacks for managed lifecycle boundaries.
- **Secrets leak through parameters:** pass Key Vault references or use workload federated identity; never output secrets.
- **Cross-subscription module fails:** ensure deployment identity has rights on every target scope and providers are registered.
- **Policy denies a deployment:** run `validate` and `what-if` against the real target scope so policy evaluation occurs before production deployment.

## VS Code
The Bicep extension provides linting, type checking, deployment pane, validation and what-if. Use `.bicepparam` files for environment-specific values, with secrets externalised.

## Official reference

- [Deploy Bicep with Visual Studio Code](https://learn.microsoft.com/azure/azure-resource-manager/bicep/deploy-vscode)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
