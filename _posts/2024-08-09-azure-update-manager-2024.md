---
layout: post
title: "Azure Update Manager: a production patching strategy for Azure and hybrid servers"
date: 2024-08-09 09:00:00 +0200
last_modified_at: 2026-11-13 09:00:00 +0100
categories: [Operations, Azure]
tags: [Azure Update Manager, Azure Arc, Windows Server, Linux, Patch Management, Governance]
description: "A detailed operating model for assessing, scheduling, deploying, and verifying Windows and Linux updates across Azure VMs and Azure Arc-enabled servers."
---

> **Updated 13 November 2026.** Update orchestration behavior differs between Azure virtual machines and Arc-enabled servers. Validate current OS, region, extension, and patching-mode support before applying a schedule.

Patching is a reliability and security process, not a monthly button click. A production patch program must identify the machines in scope, know which updates are available, select an approved update source, choose a maintenance window, protect service availability during restart, record the result, and handle failure without leaving the estate in an unknown state.

Azure Update Manager provides a unified Azure control-plane experience for assessing and applying operating system updates to Azure virtual machines and Azure Arc-enabled machines. It can also coordinate updates for supported hybrid and multicloud servers connected through Azure Arc. It does not replace change management, application testing, workload-aware rollout design, or recovery planning. Microsoft's [Update Manager overview](https://learn.microsoft.com/azure/update-manager/overview) describes its current coverage and capabilities.

## 1. Define the patching control plane

Before creating schedules, establish a system of record for machine ownership and patch policy. Each machine should have:

- an application or platform owner;
- an environment and criticality classification;
- an operating system and approved update source;
- a maintenance group and change calendar;
- a patching mode and reboot policy;
- a recovery or rollback procedure;
- a way to report assessment and installation results.

A tag such as `PatchRing=Prod-Week2` can help select a cohort, but tags alone do not guarantee safe orchestration. Confirm that tags are present, consistently spelled, and maintained by the deployment pipeline. Use Azure Resource Graph or an asset inventory to find machines with missing owners, stale classifications, unsupported operating systems, or unexpected patch modes.

Separate **assessment**, **deployment**, and **compliance**:

- **Assessment** determines which updates appear applicable at a point in time.
- **Deployment** installs the selected updates and may reboot the machine.
- **Compliance** compares observed state with the organization's policy and deadline.

A machine can have a recent successful assessment but still be non-compliant because updates were not installed. It can also be compliant for one classification and intentionally behind on another. Define what “compliant” means for the workload, including update classification, severity, deadline, reboot status, and any documented exception.

## 2. Understand the service and machine boundary

Update Manager is an Azure Resource Manager experience. Azure VMs use the Azure VM Agent, and Arc-enabled servers use the Azure Connected Machine agent. When an Update Manager operation is first triggered, the service deploys the operating system-specific patch extension required to perform assessment or installation. That means extension health, outbound connectivity, operating-system package tools, and agent state are dependencies of the patch workflow.

Update Manager does not require a Log Analytics workspace and Azure Automation account for its core patch orchestration model. Existing environments may still use Log Analytics, Automation, Update Compliance workbooks, or legacy scripts for adjacent reporting and automation, but do not assume those services are the control plane for a new Update Manager deployment.

For Arc-enabled servers, validate the Arc connection and machine resource state before interpreting a patch failure. If the server is disconnected, patch assessment and deployment cannot be treated as current. Include Arc connectivity freshness in compliance reporting so a machine that stopped checking in is not shown as healthy simply because the last known status was compliant.

## 3. Choose the patch orchestration model

Update Manager supports more than one way to control when updates are installed. The right model depends on the platform, degree of control, and operational requirements.

### Customer-managed schedules

Scheduled patching lets the organization define the maintenance window and update selection. Azure maintenance configurations are used for scheduled operations. For Azure VMs, scheduled patching requires patch orchestration to be set to **Customer Managed Schedules**. Microsoft's [updates and maintenance guidance](https://learn.microsoft.com/azure/update-manager/updates-maintenance-schedules) recommends this mode for a seamless scheduled experience and warns that schedules can fail if the VM's patch orchestration mode is not set correctly.

For Arc-enabled servers, the orchestration model differs and this Azure VM prerequisite does not apply in the same way. Arc machines do not support Azure automatic guest patching or hotpatching as Azure VMs do. Build an explicit machine-type branch into policy and runbooks rather than applying one patch configuration to all resources.

### Automatic guest patching

For supported Azure VMs, automatic guest patching can install selected critical and security updates without the operator choosing each update's exact timing. It reduces manual effort but gives the customer less control over the maintenance timing and update selection. Check the supported operating system, image, and VM configuration before using it for a workload with strict change windows.

### Operating-system orchestration

Windows automatic updates are configured at the Windows level and can interact with Group Policy and Windows Server Update Services (WSUS). Update Manager does not override every operating-system policy. For Azure Arc-enabled machines, Group Policy may fully control Windows update behavior. If WSUS is the approved source, verify that the updates are approved there and that the machine can reach the configured server. Do not troubleshoot an Update Manager deployment as though it can change the configured update source; Microsoft's [Windows Update settings guidance](https://learn.microsoft.com/azure/automation/update-management/configure-wuagent) documents these differences.

### Hotpatching

Hotpatching can reduce restart requirements for supported Windows Server Azure VMs and configurations. It is not a general patch mode for every operating system or Arc-enabled server. Confirm the current support matrix, baseline update requirements, and workload compatibility. Even when a patch does not require a reboot, application validation and security verification remain necessary.

## 4. Establish assessment coverage and freshness

An assessment is a point-in-time view of applicable updates. Enable periodic assessment for machines where the organization needs a recurring compliance picture. Microsoft's [assessment options](https://learn.microsoft.com/azure/update-manager/assessment-options) describes periodic assessment that fetches update information every 24 hours. The assessment cadence is not the same as the patch-installation cadence.

For every patch ring, define:

- the assessment frequency and the maximum allowed age of the result;
- which update classifications are included;
- whether updates come from Microsoft Update, WSUS, or the configured Linux package repository;
- how missing or failed assessments are escalated;
- which machine states are excluded and why;
- how reboot-required status is represented.

For Linux, package availability is affected by the configured repository, distribution support, package manager state, and repository metadata. For Windows, policy may select WSUS or Microsoft Update and may affect which products or classifications are visible. Update Manager reports what the machine's update source exposes; it does not prove that the update source itself is complete or healthy.

Track **assessment freshness** separately from **patch compliance**. A disconnected server whose last assessment is 45 days old is unknown, even if its last recorded assessment showed no updates. A useful reporting state model includes `Compliant`, `Non-compliant`, `Assessment stale`, `Assessment failed`, `Deployment in progress`, `Deployment failed`, `Maintenance exception`, and `Out of scope`.

## 5. Design rollout rings around workload risk

A safe patch program deploys progressively. One practical ring model is:

1. **Canary:** representative non-production machines with the same OS, image, extensions, and application dependencies as production.
2. **Early production:** a small set of low-blast-radius production nodes or a redundant service instance.
3. **Main production:** remaining machines after the observation window and service checks pass.
4. **Exception cohort:** machines that need a separate window, vendor approval, or manual coordination.

The ring is only useful if machines in it are representative. A test VM on a different OS image or package repository does not validate the production patch path. For clustered services, coordinate with the application's availability model. Update one node at a time when quorum, capacity, or connection draining requires it. Confirm that service health is good before proceeding to the next node.

For virtual machine scale sets or availability sets, use the platform's supported orchestration behavior and understand its rollout constraints. Do not assume that an Update Manager schedule will preserve the application's own quorum or replication rules. A maintenance window limits deployment time; it does not understand all application dependencies.

Classify workloads by recovery and availability needs:

- **Stateless web tier:** drain traffic, patch a small batch, validate health checks, then continue.
- **Database cluster:** patch replicas or secondaries first where supported, validate replication, then coordinate the primary under an approved runbook.
- **Single-instance legacy application:** require a business outage window, verified backup, and an explicit recovery decision.
- **Batch worker pool:** pause new work, allow in-flight tasks to finish or checkpoint, patch, then resume.
- **Domain or identity infrastructure:** follow product-specific sequencing and maintain service quorum; do not treat it as an ordinary VM batch.

## 6. Build schedules and maintenance windows

A scheduled deployment includes a target scope, recurrence, start time, maintenance duration, update classifications, and reboot behavior. Maintenance configurations can be assigned to individual resources or groups, depending on the supported workflow. Use schedules that align with time zones, change freezes, business calendars, and dependency windows.

The maintenance window is a hard operational constraint. If an update deployment cannot safely finish within the window, the service may stop before installing all remaining updates. Estimate duration from observed installation and reboot times for the target OS and patch volume. Leave time for validation and recovery; do not allocate the entire window to package installation.

Avoid placing all production systems in one large schedule. Stagger regions and workload tiers so that a patch defect or package repository issue does not affect the full service at once. Ensure that redundant nodes are not patched concurrently if doing so could remove capacity or quorum. Use separate maintenance configurations where workloads have different recovery constraints.

For recurring schedules, document ownership and review dates. A schedule created for a temporary migration may continue long after the original team has changed. A monthly schedule should not remain unowned or silently patch machines that were repurposed. Inventory schedules and their assignments as part of platform governance.

### Example: a three-ring service rollout

Assume a stateless API runs on six VMs across two zones behind a load balancer. The team needs at least four healthy instances during maintenance and has a one-hour recovery objective. A safe rollout could patch one canary first, drain it from the load balancer, install updates, restart if required, and run a synthetic transaction. After the observation period, the next change batch contains no more than one VM at a time. The change pauses automatically if fewer than five instances are healthy or if error rate exceeds the service threshold. The final batch can proceed only after the application owner confirms that the canary and first production node remain healthy.

This example is intentionally workload-specific. A six-node service with stateful local data would need a different sequence. A replicated database may require role-aware ordering, replication-lag checks, and a failover test. A single VM cannot be made highly available by choosing a longer maintenance window; the architecture or business outage plan must account for that constraint.

The schedule itself should specify the target machines and update selection, but the application rollout controller or change runbook should enforce the health gates. Keep the two concerns separate: Update Manager installs guest updates, while the workload's orchestration layer determines whether the application can safely lose a node.

## 7. Control update sources and package behavior

Update classification is a policy decision, not a substitute for understanding a specific security bulletin. A team may use a shorter deadline for critical security updates and a regular cadence for other approved updates, but the selected classifications and service behavior must be verified for the operating system and package source. Some updates supersede earlier packages, require prerequisite servicing changes, or require a restart to become effective. An assessment list can change between assessment and deployment if new packages arrive or the repository state changes; record the assessment time and re-assess when the delay is material.

Patch installation behavior depends on the machine's update source:

- Windows may use Microsoft Update, WSUS, or policy-controlled settings.
- Linux distributions use their configured package manager and repositories.
- Arc-enabled machines depend on their configured guest OS update behavior and connected-machine prerequisites.

Before deployment, confirm that the source is reachable and returns the intended updates. For WSUS, confirm approval state and Group Policy. For Linux, check repository reachability, package metadata, held packages, dependency conflicts, and available disk space. A deployment that fails because an update source is unavailable is not fixed by repeatedly starting the same job.

Keep update source configuration outside the patch schedule. Changing from WSUS to Microsoft Update is a policy and machine-configuration change that should be handled through the approved Group Policy or OS management process. Microsoft's Windows Update settings article explicitly documents source selection and cautions against using Update Manager to change it.

Define how to handle updates that require manual approval, application compatibility testing, or a vendor-specific procedure. Exclude them only with an owner, reason, expiry, and compensating action. A blanket exclusion for “all drivers” or “all optional updates” should be reviewed against the organization's actual risk and operational needs.

## 8. Reboots, availability, and change safety

A patch may require a restart to complete installation or activate a kernel change. The reboot policy affects both security state and availability. If restarts are deferred indefinitely, the operating system may report updates installed while vulnerable code remains active until restart. If every machine restarts at the same time, service availability may be lost.

For each workload, define:

- who approves the maintenance window;
- whether the service can drain connections or pause work;
- how many nodes can be unavailable at once;
- what health signal must be green before the next batch;
- how long to wait for restart and service recovery;
- which update failures trigger rollback, restore, or vendor support;
- who communicates to business stakeholders.

Not every OS update can be rolled back safely. A VM snapshot is not a universal rollback plan, especially for databases or systems with external side effects. Verify backup consistency, data recovery, configuration state, and application compatibility. For high-impact systems, test the recovery path before relying on it during an emergency patch.

Do not use patch deferral as a substitute for change management. Use a time-bound exception with documented exposure, risk owner, compensating controls, and a planned remediation date. Revisit critical security updates sooner than routine updates when the threat and business context justify an expedited change.

## 9. Execute and verify a one-time deployment

A one-time update deployment is useful for an urgent security fix, a controlled pilot, or a remediation outside the recurring schedule. The operational sequence is:

1. Confirm the machine is online and its update source is healthy.
2. Run an assessment and record the applicable updates.
3. Select the intended classifications and specific update scope.
4. Confirm change approval, maintenance window, and reboot behavior.
5. Deploy to the canary or approved cohort.
6. Review per-machine deployment status and installation details.
7. Validate OS health, application health, monitoring, and business transactions.
8. Expand the deployment only after the observation criteria pass.

Update Manager retains deployment history and provides deployment status such as succeeded, failed, completed with warnings, in progress, or not started. Use the activity/deployment record to investigate which updates were selected and which failed. A top-level “completed” state can hide a warning or an individual machine failure, so review the details at the machine and update level.

Capture both control-plane and guest-level evidence. The Azure operation record shows that a deployment was requested and processed. The operating system's package or Windows Update history helps confirm installation state. Application health checks show whether the business service recovered. Use all three when closing a high-risk change.

## 10. Automate at scale without losing ownership

Azure Policy can help enable periodic assessment or assign patch configuration at scale. Use policy initiatives and remediation identities deliberately. Validate policy effects in audit mode before deploying configuration changes across a management group. Check the target scope, exclusions, managed identity permissions, and behavior for machines that are offline or in a different patching mode.

Use dynamic scope for machines that meet clear criteria, such as environment, region, operating system, and patch ring. Do not base production maintenance scope only on a wildcard tag if any team can assign that tag. Protect patch-ring tags through deployment pipelines or policy and report membership changes.

A scalable design needs reconciliation, not just deployment. A recurring controller should compare the intended machine inventory with:

- actual assignment to a schedule;
- latest assessment state and timestamp;
- deployment outcome;
- pending reboot state;
- exception or maintenance-window status;
- application owner and service health.

Keep one authoritative source for schedule ownership. Avoid overlapping schedules or independent automation that can initiate concurrent patch jobs. If an external service desk system owns change approvals, make the linkage between approved change ID and Update Manager deployment discoverable.

## 11. Monitor compliance and report honestly

A fleet dashboard should distinguish failure and uncertainty. Report compliance by operating system, environment, criticality, update classification, patch ring, and owner. Include a separate count for disconnected or stale machines. Show both the age of the latest assessment and the age of the latest successful installation.

Useful operational measures include:

- percentage of in-scope machines assessed within the freshness objective;
- percentage meeting the patch deadline by ring and severity;
- deployment success rate and failure reason;
- number of systems pending restart beyond the allowed period;
- average time from update availability to production completion;
- number and age of approved exceptions;
- machines with unsupported or misconfigured patch orchestration;
- application incidents correlated with patch deployments.

Do not label the whole estate “compliant” if a material portion has stale results. Publish the denominator and scope. If a workload is intentionally excluded, report it as excluded with an owner, not as compliant.

Use Resource Graph, Update Manager views, and supported query/reporting paths to join patch status to the asset inventory. Validate the current schema before automating reports because APIs and data models can evolve. Retain deployment results according to audit and operational requirements, and avoid sending duplicate detailed telemetry to multiple monitoring systems without a consumer.

## 12. Troubleshooting guide

| Symptom | Likely cause | First investigation |
|---|---|---|
| Assessment fails immediately | Agent, extension, permissions, or provider registration | VM/Arc agent health, extension status, resource provider, Activity Log |
| No updates appear but the server is behind | Stale assessment, wrong source, WSUS policy, or repository issue | Assessment timestamp, source configuration, package metadata, update history |
| Scheduled deployment never starts | Invalid scope, schedule association, patch mode, or time-zone mismatch | Maintenance configuration, machine assignment, Customer Managed Schedules setting |
| Deployment fails on Windows | WSUS approval, Group Policy, pending reboot, disk space, servicing stack | Windows Update logs, policy, free space, update-specific error |
| Deployment fails on Linux | Repository unreachable, package lock, dependency conflict, disk pressure | Package manager logs, repository health, held packages, filesystem capacity |
| Updates install but app is unhealthy | Restart sequencing or application dependency | Service logs, health probes, cluster quorum, recent deployment correlation |
| Arc machine is absent or stale | Connected Machine agent disconnected or scope mismatch | Arc status, connectivity, identity and subscription registration |
| Many machines fail in one ring | Shared image, repository, policy, or schedule defect | Stop later rings, compare common dependencies, fix cohort-level cause |

Use the failure details and guest logs before retrying. Repeated retries can create overlapping operations or obscure the original cause. If many machines in a cohort fail with the same update or source error, pause the ring, preserve the evidence, and assess the shared dependency before resuming.

## 13. A phased patch program

**Phase 1 — Inventory:** establish owners, OS support, update source, criticality, maintenance constraints, Arc state, and existing patch tooling. Identify overlap with legacy Update Management solutions.

**Phase 2 — Assessment:** enable periodic assessment in a pilot, validate freshness and update-source behavior, and build the compliance state model. Do not schedule broad deployments yet.

**Phase 3 — Pilot deployment:** choose representative canary machines, define the reboot and rollback approach, execute one scheduled cycle, and verify guest and application state.

**Phase 4 — Ring expansion:** add cohorts gradually, stagger critical systems, and monitor the first production cycles. Preserve a stop condition if failure or application impact exceeds the agreed threshold.

**Phase 5 — Governance:** assign schedules through policy or automation, protect scope tags, report stale systems and exceptions, and review schedule ownership quarterly.

**Phase 6 — Improvement:** correlate patch data with incidents, reduce deployment failures at the source, shorten exposure for high-risk updates, and test recovery procedures.

## Production checklist

- [ ] Azure VMs and Arc-enabled servers are handled according to their distinct support and orchestration behavior.
- [ ] Each machine has an owner, patch ring, approved update source, and maintenance policy.
- [ ] Periodic assessment is enabled where required, and stale results are reported as unknown.
- [ ] Azure VMs scheduled by Update Manager use the required Customer Managed Schedules orchestration mode.
- [ ] WSUS, Group Policy, and Linux repository configuration are verified before deployment.
- [ ] Maintenance windows include installation, reboot, validation, and recovery time.
- [ ] Redundant workloads are patched in a sequence that preserves quorum and service capacity.
- [ ] Deployment results are verified at machine, guest OS, and application levels.
- [ ] Automation has a protected scope, least-privilege identity, audit trail, and stop condition.
- [ ] Exceptions have an owner, risk statement, compensating control, and expiry date.
- [ ] The organization can identify unsupported, disconnected, stale, and misconfigured machines.

## References

- [Azure Update Manager overview](https://learn.microsoft.com/azure/update-manager/overview)
- [How Update Manager works](https://learn.microsoft.com/azure/update-manager/workflow-update-manager)
- [Assessment options](https://learn.microsoft.com/azure/update-manager/assessment-options)
- [Updates and maintenance schedules](https://learn.microsoft.com/azure/update-manager/updates-maintenance-schedules)
- [Schedule recurring updates](https://learn.microsoft.com/azure/update-center/scheduled-patching)
- [Configure Windows Update settings](https://learn.microsoft.com/azure/automation/update-management/configure-wuagent)
- [Check update compliance](https://learn.microsoft.com/azure/update-manager/view-updates)
