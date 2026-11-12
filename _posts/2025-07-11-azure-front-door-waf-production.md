---
layout: post
title: "Azure Front Door and WAF in production: origin security, rules and safe rollout"
date: 2025-07-11 09:00:00 +0200
categories: [Networking, Azure]
tags: [Azure, Networking, Architecture, Operations, Troubleshooting]
description: "Global entry point, WAF tuning and origin protection."
---

## Reference flow

```text
Client → Azure Front Door → WAF policy → Private Link or restricted origin → Application
```

Use Front Door Standard or Premium for new deployments. Protect the origin so clients cannot bypass Front Door. Premium supports Private Link to supported origins.

## WAF rollout
Start in Detection mode, collect representative traffic, tune exclusions narrowly, then move to Prevention. Do not disable a managed rule set because of one false positive.

```bash
az network front-door waf-policy show --resource-group rg-edge --name waf-prod
```

## Origin controls

- validate host header and health-probe path;
- allow only Front Door or use Private Link;
- use managed certificates where possible;
- configure separate origin groups and health probes;
- test failover priority and weights;
- keep application redirects aware of forwarded headers.

## Workarounds

- **Backend healthy in browser but unhealthy in Front Door:** verify probe method, status code, host header, TLS name and path.
- **Redirect loop:** application may not trust `X-Forwarded-Proto`; configure proxy-aware HTTPS handling.
- **False positive WAF rule:** create a scoped exclusion for parameter, cookie or header rather than disabling protection globally.
- **Origin still public:** use Private Link on Premium or restrict network access and validate requests at origin.
- **Certificate validation fails:** ensure custom-domain DNS validation records remain in place where required.

## Official references

- [Create WAF policy for Azure Front Door](https://learn.microsoft.com/azure/web-application-firewall/afds/waf-front-door-create-portal)
- [Front Door Premium with Private Link origin](https://learn.microsoft.com/azure/frontdoor/how-to-enable-private-link-storage-static-website)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
