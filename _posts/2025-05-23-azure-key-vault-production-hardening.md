---
layout: post
title: "Azure Key Vault production hardening: RBAC, Private Link, rotation and recovery"
date: 2025-05-23 09:00:00 +0200
categories: [Security, Azure]
tags: [Azure, Security, Architecture, Operations, Troubleshooting]
description: "A secure Key Vault baseline with operational recovery."
---

## Security baseline
Use Azure RBAC permission model for new designs, private connectivity where required, managed identities for applications, purge protection, soft delete, diagnostic logs and separation between administration and secret consumption.

Never put a secret value in Bicep parameters, deployment outputs, source control, pipeline logs or command history.

## Bicep baseline

```bicep
resource kv 'Microsoft.KeyVault/vaults@2024-11-01' = {
  name: keyVaultName
  location: location
  properties: {
    tenantId: tenant().tenantId
    sku: { family: 'A', name: 'standard' }
    enableRbacAuthorization: true
    enablePurgeProtection: true
    softDeleteRetentionInDays: 90
    publicNetworkAccess: 'Disabled'
    networkAcls: { bypass: 'AzureServices', defaultAction: 'Deny' }
  }
}
```

## Application access
Assign a managed identity only the data-plane role it requires, such as Key Vault Secrets User. Keep Key Vault Administrator restricted to operational administrators and preferably eligible through PIM.

```bash
az role assignment create   --assignee-object-id <managed-identity-object-id>   --assignee-principal-type ServicePrincipal   --role 'Key Vault Secrets User'   --scope <vault-resource-id>
```

## Rotation pattern
Store credentials only when managed identity or federation cannot be used. Where secrets remain necessary:

- define owner and expiry;
- rotate before expiry;
- support two valid credentials during transition;
- update consumers before revoking the old value;
- alert on near-expiry certificates and secrets;
- test recovery from accidental deletion.

## Troubleshooting and workarounds

- **403 despite role assignment:** distinguish control-plane and data-plane roles, verify token tenant, role propagation and vault permission model.
- **Private endpoint works by IP but not name:** fix private DNS. Applications must use the vault FQDN.
- **Deployment overwrites access policies:** avoid embedding mutable access-policy arrays in a repeatedly deployed vault template. Prefer RBAC or separate child resources.
- **Secret retrieval fails immediately after role creation:** RBAC propagation is eventual. Add bounded retry with exponential backoff, not a fixed long sleep.
- **Vault deletion cannot be undone:** soft-delete recovery is possible during retention, while purge protection blocks early permanent purge.

```powershell
$delay = 5
1..6 | ForEach-Object {
  try { return Get-AzKeyVaultSecret -VaultName $vault -Name $secret -AsPlainText }
  catch { Start-Sleep -Seconds $delay; $delay *= 2 }
}
throw 'Secret access did not become available within retry window.'
```

## Official reference

- [Create Key Vault with infrastructure as code](https://learn.microsoft.com/azure/key-vault/general/vault-create-template)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
