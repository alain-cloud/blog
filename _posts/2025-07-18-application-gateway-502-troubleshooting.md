---
layout: post
title: "Azure Application Gateway 502 troubleshooting: probes, TLS, DNS and host headers"
date: 2025-07-18 09:00:00 +0200
categories: [Networking, Azure]
tags: [Azure, Networking, Architecture, Operations, Troubleshooting]
description: "A deterministic backend-health workflow."
---

## Start with backend health
A 502 normally means Application Gateway could not receive a valid response from a healthy backend. Inspect backend health before changing random settings.

```bash
az network application-gateway show-backend-health   --resource-group rg-appgw   --name appgw-prod   --output jsonc
```

## Diagnostic order

1. resolve backend FQDN from the gateway network path;
2. validate NSG and UDR;
3. validate probe host, path, protocol and accepted status code;
4. validate backend TLS certificate chain and hostname;
5. validate backend HTTP setting host override and SNI;
6. inspect application logs and timeout.

## TLS test

```bash
openssl s_client -connect backend.contoso.internal:443 -servername backend.contoso.internal -showcerts
curl -vk https://backend.contoso.internal/health
```

## Workarounds

- **Certificate name mismatch:** use the correct backend FQDN and SNI host, not the backend IP.
- **Private endpoint backend fails:** verify private DNS from the App Gateway VNet.
- **Probe receives 401:** publish a dedicated unauthenticated health endpoint that verifies essential dependencies without exposing sensitive data.
- **Probe path redirects:** use a direct healthy path and correct protocol.
- **Timeout only on large requests:** inspect request timeout, backend processing and connection draining rather than increasing every timeout blindly.

## Official reference

- [Application Gateway backend certificate troubleshooting](https://learn.microsoft.com/azure/application-gateway/create-gateway-internal-load-balancer-app-service-environment)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
