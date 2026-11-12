---
layout: post
title: "Azure reliability testing and chaos engineering: a safe production approach"
date: 2025-09-12 09:00:00 +0200
categories: [Reliability, Azure]
tags: [Azure, Reliability, Architecture, Operations, Troubleshooting]
description: "Fault injection with hypotheses, guardrails and evidence."
---

## Start with a hypothesis

```text
If one application instance is terminated, the service remains available and p95 latency stays below the agreed threshold because traffic is distributed across healthy replicas.
```

Define steady-state metrics, blast radius, abort condition, owner, maintenance window and recovery steps before injecting a fault.

## Test progression

```text
Local test → integration → staging → limited production → broader recurring exercise
```

Examples:

- stop one VM or AKS node;
- block a dependency temporarily;
- increase latency;
- simulate zone loss through controlled failover;
- test throttling and retry behaviour;
- remove access to a noncritical dependency.

## Workarounds

- **Test proves nothing:** define measurable steady-state and expected recovery before execution.
- **Blast radius expands:** target explicit resources and use Azure Policy/RBAC to constrain the experiment identity.
- **Monitoring fails during test:** observability is itself a dependency; add an external synthetic check.
- **Retry storm worsens outage:** use exponential backoff, jitter, circuit breakers and bounded queues.
- **Production test is rejected:** demonstrate the same experiment in staging, document controls, then propose a narrow production scope.

## Evidence template

```text
Hypothesis
Scope
Start/end time
Injected fault
Metrics before/during/after
Abort threshold
Observed recovery
RTO achieved
Unexpected behaviour
Actions and owners
```

## Official reference

- [Azure Well-Architected reliability testing strategy](https://learn.microsoft.com/azure/well-architected/reliability/testing-strategy)

## Practitioner validation checklist

- [ ] Confirm current service availability, retirement notices and API versions in the target Azure region.
- [ ] Test commands in a non-production subscription first.
- [ ] Use managed identity or workload identity federation instead of stored credentials.
- [ ] Run infrastructure `what-if` or equivalent preview before deployment.
- [ ] Document owner, rollback, monitoring and service objective.
- [ ] Capture screenshots or exported evidence for the change record.
- [ ] Review cost, security, reliability and operational impact together.

> **Accuracy statement:** This article describes real Azure capabilities and documented engineering patterns. It does not claim invented customer results. Service limits, prices, regions and product behaviour change; verify current Microsoft Learn documentation before production use.
