---
layout: post
title: "Azure Private Endpoint and DNS: the production troubleshooting guide"
date: 2025-05-16 09:00:00 +0200
categories: [Networking, Azure]
tags: [Azure, Networking, Architecture, Operations, Troubleshooting]
description: "Private Link, private DNS, hybrid resolution and NXDOMAIN troubleshooting."
---

## The real problem is usually DNS
A private endpoint is a network interface with a private IP mapped to a supported Azure service subresource. Creating the endpoint is only half the job. Clients must resolve the service FQDN to the private IP from every relevant network path.

```text
Client → DNS resolver → private DNS zone → private endpoint IP → Azure service
```

## Required design

1. Create the private endpoint in a dedicated subnet where appropriate.
2. Create or reuse the service-specific private DNS zone.
3. Link the zone to every VNet that needs resolution.
4. For on-premises clients, forward the relevant zone to Azure DNS Private Resolver or a DNS forwarder in Azure.
5. Disable public network access only after private resolution and connectivity tests pass.

Example zones include `privatelink.blob.core.windows.net`, `privatelink.vaultcore.azure.net` and service-specific database zones.

## Diagnostic commands

```powershell
Resolve-DnsName mystorage.blob.core.windows.net
Test-NetConnection mystorage.blob.core.windows.net -Port 443
```

```bash
nslookup mystorage.blob.core.windows.net
az network private-endpoint-connection list   --id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<name>
```

Expected resolution often follows a CNAME chain to a `privatelink` name and finally a private RFC1918 address.

## Common failures and workarounds

### Public IP is returned
The VNet is not linked to the private DNS zone, the client uses an external resolver, or a custom DNS server is not forwarding the zone to Azure.

### NXDOMAIN from on-premises
Conditional forwarding is missing. Configure Azure DNS Private Resolver inbound endpoints and an on-premises conditional forwarder for the private zone.

### Works in one VNet only
Private DNS links are per VNet. Link the central zone to each spoke or use an intentional DNS architecture with resolver rulesets.

### Endpoint is approved but connection fails
Validate NSGs, UDRs, firewall DNS proxy behaviour, service subresource, endpoint approval and whether the application uses the correct FQDN rather than a hard-coded IP.

### Split-brain DNS breaks public clients
Do not create an empty private zone without the required record. Empty zones can cause NXDOMAIN for linked VNets.

## Bicep example

```bicep
resource zone 'Microsoft.Network/privateDnsZones@2024-06-01' = {
  name: 'privatelink.blob.core.windows.net'
  location: 'global'
}

resource link 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2024-06-01' = {
  parent: zone
  name: 'link-spoke-prod'
  location: 'global'
  properties: {
    registrationEnabled: false
    virtualNetwork: { id: vnetId }
  }
}
```

## Official reference

- [Azure network foundation services](https://learn.microsoft.com/azure/networking/foundations/network-foundations-overview)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
