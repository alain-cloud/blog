---
layout: post
title: "Azure VM security hardening: identity, network, patching, encryption and recovery"
date: 2025-08-01 09:00:00 +0200
categories: [Security, Azure]
tags: [Azure, Security, Architecture, Operations, Troubleshooting]
description: "A layered VM baseline for Windows and Linux."
---

## Harden by layers

1. identity and privileged access;
2. network exposure;
3. secure image and configuration;
4. patching and endpoint protection;
5. disk and host encryption;
6. logging, backup and recovery.

Avoid direct RDP or SSH from the internet. Use Azure Bastion, VPN/ExpressRoute, JIT access and private management paths.

## Defender and patching
Use Microsoft Defender for Servers where required, Microsoft Defender for Endpoint integration, vulnerability assessment and Azure Update Manager. Define maintenance windows and reboot expectations.

## Encryption at host
Encryption at host encrypts temporary disks and host cache at rest and sends encrypted data to Storage. Verify VM-size and disk compatibility. It cannot be combined in all ways with Azure Disk Encryption.

```powershell
Register-AzProviderFeature -FeatureName EncryptionAtHost -ProviderNamespace Microsoft.Compute
Get-AzProviderFeature -FeatureName EncryptionAtHost -ProviderNamespace Microsoft.Compute
```

## Workarounds

- **Bastion cannot connect:** validate NSGs, guest firewall, credentials, Bastion subnet and VM agent status.
- **Patch says succeeded but app fails:** include application smoke test and rollback, not only patch status.
- **Encryption feature unavailable:** confirm provider registration, region, VM SKU and disk restrictions.
- **Defender agent unhealthy:** inspect extension state, proxy and outbound endpoint access.
- **Backup is green but restore is untested:** perform isolated restore tests and validate application consistency.

## Official references

- [Azure VM security overview](https://learn.microsoft.com/azure/security/fundamentals/virtual-machines-overview)
- [Encryption at host](https://learn.microsoft.com/azure/virtual-machines/windows/disks-enable-host-based-encryption-powershell)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
