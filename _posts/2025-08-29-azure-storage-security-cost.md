---
layout: post
title: "Azure Storage in production: security, lifecycle, redundancy and cost traps"
date: 2025-08-29 09:00:00 +0200
categories: [Storage, Azure]
tags: [Azure, Storage, Architecture, Operations, Troubleshooting]
description: "A durable and cost-aware storage baseline."
---

## Design dimensions
Select account type, performance tier, namespace, redundancy and access tier from workload requirements. Security and recovery choices must be made before cost optimisation.

Baseline controls:

- disable shared-key access where supported by applications;
- use Entra ID and managed identity;
- require secure transfer and minimum TLS;
- use private endpoints or controlled network access;
- enable soft delete, versioning and change feed where justified;
- define lifecycle policies;
- protect critical data with immutability where required.

## Lifecycle example

```json
{
  "rules": [{
    "enabled": true,
    "name": "tier-and-expire",
    "type": "Lifecycle",
    "definition": {
      "actions": { "baseBlob": {
        "tierToCool": { "daysAfterLastAccessTimeGreaterThan": 30 },
        "tierToArchive": { "daysAfterLastAccessTimeGreaterThan": 180 },
        "delete": { "daysAfterModificationGreaterThan": 2555 }
      }},
      "filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["archive/"] }
    }
  }]
}
```

## Workarounds

- **Archive saves storage but restore is slow:** align tier with RTO and retrieval cost, and keep active recovery data online.
- **Private endpoint blocks trusted service:** validate each dependency; “trusted services” bypass is not a universal allow-list.
- **Lifecycle deletes needed versions:** test rules on a limited prefix and account for version/snapshot actions separately.
- **GRS assumed to be backup:** replication protects infrastructure availability, not accidental deletion or logical corruption by itself.
- **Egress surprise:** model data movement, region pairing and client location.

## Validation commands

```bash
az storage account show --name <name> --resource-group <rg>   --query '{httpsOnly:enableHttpsTrafficOnly,tls:minimumTlsVersion,public:publicNetworkAccess,sharedKey:allowSharedKeyAccess}'
```

## Official reference

- [Azure network foundation and private connectivity](https://learn.microsoft.com/azure/networking/foundations/network-foundations-overview)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
