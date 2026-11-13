---
layout: post
title: "Microsoft Defender for Cloud: engineering posture management and workload protection"
date: 2024-06-28 09:00:00 +0100
last_modified_at: 2026-11-13 09:00:00 +0100
categories: [Security, Azure]
tags: [Azure, Microsoft Defender for Cloud, CSPM, CWPP, CNAPP, Security Operations]
description: "A technical operating guide to Defender for Cloud plans, Azure and multicloud onboarding, risk-based remediation, workload protection, automation, and security operations."
---

> **Updated 13 November 2026.** Microsoft Defender for Cloud is changing its portal experience, plan model, and recommendation views. Check the current release notes and feature availability before applying this guide to a tenant.

Microsoft Defender for Cloud is often introduced as a dashboard that reports a secure score. That description misses the engineering work required to make the service useful. A cloud security program must connect asset inventory, configuration assessment, workload threat protection, ownership, remediation, exception governance, and incident response. Defender for Cloud can provide a unified view across Azure, connected hybrid systems, AWS, GCP, and supported development repositories, but each capability has its own plan, data dependency, identity scope, and operational owner.

This guide explains how to design Defender for Cloud as a security control plane. It focuses on the questions that matter in production: what to enable, what data the service evaluates, how to distinguish posture findings from threat alerts, how to prioritize and automate actions, and how to keep costs and exceptions under control. Microsoft's [Defender for Cloud overview](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-cloud-introduction) describes the current CNAPP components and plan capabilities.

## 1. Understand the three security jobs

A cloud-native application protection platform (CNAPP) brings together capabilities that are related but solve different problems. In Defender for Cloud, organize the design around three jobs.

### Cloud security posture management

Cloud security posture management (CSPM) evaluates configuration and exposure. It asks questions such as: Is a storage account publicly reachable? Does a machine have a missing security update? Is an identity or network configuration inconsistent with the selected standard? Recommendations are findings against resources and policies. They help teams reduce misconfiguration risk, but a recommendation is not proof that an attack is occurring.

### Cloud workload protection

Cloud workload protection platform (CWPP) capabilities protect specific workload types and identify active or emerging threats. Depending on the enabled plan and workload, this can include server threat protection, container runtime signals, database threat detection, or other service-specific findings. Workload protection generally requires a plan decision and may involve agents, extensions, resource integrations, or data collection. Do not infer CWPP coverage from a CSPM recommendation view.

### Development security operations

DevSecOps capabilities connect supported source repositories and pipelines to cloud security posture. They can identify infrastructure-as-code and code-level issues before deployment and help connect a deployed cloud asset to its source. This shifts some remediation earlier in the lifecycle, where a secure default can prevent the same misconfiguration from being redeployed.

These are complementary layers. CSPM helps identify risky configuration; CWPP helps protect deployed workloads; DevSecOps can prevent insecure changes from reaching the cloud. Security operations teams still need a separate incident process for triage, containment, evidence, and recovery.

## 2. Model plans before enabling them

The first design artifact should be a capability-to-resource matrix, not a list of every plan toggle. For each subscription, cloud account, cluster, or repository, record:

| Design question | Example decision |
|---|---|
| Which assets are in scope? | Production subscriptions, selected development subscriptions, AWS production accounts |
| Which posture baseline applies? | Microsoft Cloud Security Benchmark plus organization-specific controls |
| Which workload plans are required? | Server protection for production machines; container protection for AKS and connected clusters |
| What data or components are required? | Resource discovery, Azure Arc, Defender sensor, vulnerability assessment, audit events |
| Who owns findings? | Subscription platform team, workload team, security operations |
| What is the expected cost? | Estimate each resource-metered plan using the current pricing model |
| How are exceptions approved? | Named risk owner, compensating control, expiry date, review schedule |

Defender for Cloud includes **Foundational CSPM** capabilities without a paid Defender CSPM plan, and **Defender CSPM** adds advanced posture capabilities such as risk prioritization, attack path analysis, governance, and other tools. The exact capability set varies by cloud, resource type, portal, and plan. Some recommendations appear only when a related workload protection plan is enabled.

There is a timing detail administrators must account for: Microsoft's 2026 release notes state that, starting **27 October 2026**, Foundational CSPM moves to an opt-in model and is no longer enabled by default for new Azure subscriptions. It remains available at no cost and can be enabled. Existing subscriptions that already have it enabled remain enabled unless the plan is turned off. Include an explicit onboarding check for new subscriptions rather than assuming the previous default. See the [current Defender for Cloud release notes](https://learn.microsoft.com/azure/defender-for-cloud/release-notes) and [CSPM plan guidance](https://learn.microsoft.com/azure/defender-for-cloud/concept-cloud-security-posture-management).

Do not enable every paid plan at the management-group root without first checking scope and projected coverage. Broad assignment can protect resources quickly, but it can also activate billing on resources that were outside the intended program. Conversely, a plan left disabled at a parent scope can create quiet coverage gaps in new subscriptions. Use a documented landing-zone policy and a recurring coverage report to make the result explicit.

## 3. Separate assessment, recommendation, and remediation

A robust operating model has four stages:

1. **Assessment:** Defender for Cloud evaluates resource state against built-in or custom security policy.
2. **Finding:** A recommendation identifies affected resources and explains the security concern.
3. **Decision:** A team assesses business context, exploitability, exposure, criticality, and remediation risk.
4. **Action:** The owner fixes the issue, applies a time-bound exemption, or escalates a risk acceptance.

Do not treat every recommendation as a production change ticket with the same priority. A finding that exposes a sensitive data store to the internet is materially different from a low-impact hardening recommendation on an isolated development resource. Risk context can include internet exposure, sensitive data, lateral movement, attack paths, and resource criticality. The recommendation should be reviewed with the owning team and the service's architecture, not blindly fixed by a broad automation rule.

Defender for Cloud now has distinct score experiences. The classic secure score in the Azure portal and the newer risk-based Cloud Secure Score in the Defender portal use different models and values. Microsoft documentation says that the risk-based model factors resource risk and criticality to prioritize remediation; the classic score remains a separate calculation. Therefore, do not compare a chart from one portal with a score from the other as if it were the same KPI. Use the score as a navigation and trend signal, not as a substitute for an organization-specific risk measure. See [Cloud Secure Score](https://learn.microsoft.com/azure/defender-for-cloud/secure-score-access-and-track).

The recommendation risk level and secure score also answer different questions. Risk prioritization helps determine what to work on first. A score may be calculated from a different defined subset of recommendations and controls. A team can reduce material risk without maximizing a score, and a score can change when scope, assets, or included recommendations change. Record scope changes alongside score trends so a broader assessment is not mistaken for a security regression.

### Build a remediation service-level agreement

Define response targets by risk and asset criticality, for example:

- **Critical and internet exposed:** immediate security review and containment plan; same-day owner assignment.
- **High risk on a production asset:** remediation or documented mitigation in the agreed expedited window.
- **Medium risk:** planned backlog with a due date and service-owner confirmation.
- **Low risk or accepted design trade-off:** periodic review, evidence, and a named exception owner.

The exact targets are organization-specific. The important properties are a named owner, a due date, an escalation route, and a record of what happens when the target is missed. Use governance rules where available to assign owners and due dates consistently, but avoid using automated assignment to imply that a team has accepted responsibility without validating the ownership mapping.

## 4. Design Azure policy and subscription onboarding

At Azure scale, Defender for Cloud inherits much of its scope and configuration from Azure subscriptions and management groups. Define a control plane for:

- enabling the posture capabilities required for the subscription class;
- assigning the approved security standard;
- enabling workload protection plans by resource or environment scope;
- exporting security events to the central operations destination;
- deploying required diagnostic or integration components;
- assigning least-privilege roles to platform and security teams.

Use Azure Policy initiatives and management-group hierarchy to establish repeatable defaults, with carefully scoped exceptions. A useful pattern is to separate policy that **audits** a setting from policy that **deploys or modifies** it. Audit first where the existing estate is unknown. Remediation tasks and deploy-if-not-exists effects require managed identity permissions and can create or change resources; review the effect, role assignments, and blast radius before enabling them at scale.

Create a new-subscription validation that confirms the intended Defender plans are enabled, posture standards are assigned, connectors are healthy, and export routes are present. Because defaults can change, test this control against the current service behavior rather than relying on assumptions from a prior subscription. Keep plan configuration in a central inventory or policy assignment record that identifies the business owner and cost owner.

Use resource inventory to establish coverage. A useful weekly report shows in-scope resources by type, protection plan status, assessment freshness, connector state, owner, and region. The report should distinguish “not applicable” from “not onboarded” and “onboarded but unhealthy.” A percentage that combines all three states hides important gaps.

## 5. Extend posture and workload protection to multicloud

Defender for Cloud can connect AWS accounts and GCP projects for posture management and selected workload protection. CSPM discovery for connected clouds is largely agentless after the connector and required cloud permissions are configured. Workload protection has different dependencies. For example, machine and container protection may need Azure Arc, agent or extension deployment, a Kubernetes sensor, cloud audit-log integration, or other prerequisites depending on the plan.

Plan the connector as a trust relationship. Use the supported federated authentication pattern rather than creating a long-lived cloud access key where possible. Review the requested AWS IAM role or GCP service-account permissions with the cloud security owner. Start with a non-production account, validate inventory completeness and assessment results, then extend through infrastructure as code or a repeatable onboarding runbook.

Do not assume that “AWS connected” means all AWS workload protection is active. Separate the status of:

- account/project connector health;
- CSPM asset discovery and policy assessment;
- server protection and endpoint integration;
- container discovery and runtime protection;
- database protection;
- cloud audit-log ingestion;
- alert routing and SIEM integration.

Microsoft notes that the Log Analytics agent retired in August 2024; Defender for Servers capabilities do not depend on that retired agent. Multicloud architecture should use the current documented dependencies for each plan rather than preserving legacy agent assumptions. See [multicloud CSPM and CWPP dependencies](https://learn.microsoft.com/azure/defender-for-cloud/plan-multicloud-security-determine-multicloud-dependencies).

Use a shared control taxonomy across Azure, AWS, and GCP, but do not force identical technical remediations where the providers differ. For each finding, map to an organization control, the provider-specific resource, evidence, owner, and corrective action. Keep a documented mapping when one organization control is satisfied by different provider-native mechanisms.

## 6. Connect workload protection with the operations model

Security alerts represent potential malicious activity or a workload threat. Recommendations represent security posture issues. A threat alert should follow the incident workflow: validate the alert, scope affected resources and identities, preserve evidence, contain safely, investigate the initial access path, recover, and record lessons learned. A posture recommendation should usually follow an engineering or governance workflow: assign an owner, plan a change, validate impact, implement, and verify the assessment clears.

These paths can converge when an alert identifies a resource with a known posture weakness. The SOC may need to provide incident context to the workload team, while the workload owner helps explain normal behavior and change history. Define when the SOC can isolate a VM, disable an identity, or modify network access, and when approval is required. Automation should not perform a destructive containment action unless the organization has agreed on the trigger, scope, evidence, and rollback procedure.

For containers, align the Defender sensor and Kubernetes controls with the cluster's deployment model. Confirm supported Kubernetes versions and network requirements. Decide who owns image vulnerabilities, cluster configuration findings, runtime alerts, and namespace-level remediation. A central platform team may own the cluster baseline while application teams own vulnerable dependencies in their images. The responsibility split must be visible in the ticketing workflow.

## 7. Make recommendation automation safe

Defender for Cloud supports workflow automation for recommendations, alerts, and regulatory compliance changes. A workflow can trigger a Logic App or downstream system to create a ticket, enrich an event, or notify an owner. Continuous export can send selected security data to Log Analytics, Event Hubs, or a SIEM/SOAR platform.

A safe automation pipeline starts with **routing and enrichment**, not auto-remediation:

1. Receive the finding or alert.
2. Normalize subscription, cloud, resource ID, recommendation ID, severity, risk, and environment.
3. Resolve an owner from a maintained service catalogue.
4. Suppress duplicates using a stable finding key and current state.
5. Create or update a work item with due date, evidence, and remediation guidance.
6. Reconcile when the finding closes, reopens, or receives an approved exemption.

Only automate remediation after the change has been tested, the target scope is narrow, and the action is reversible or has a recovery plan. Some fixes can restart a service, alter public reachability, change identity permissions, or disrupt a business integration. Use a staged rollout and an approval gate for production resources. Keep an audit trail that identifies the triggering finding, automation version, identity, action, and result.

Continuous export is event-oriented. It sends data when configured events or state changes occur; it is not automatically a complete snapshot of every current finding. Microsoft notes, for example, that enabling export does not replay every pre-existing alert and that recommendation records are emitted when compliance state changes. If a downstream system needs a complete current-state view, pair event export with a periodic inventory/query process rather than assuming that the event stream is a full database. See [continuous export behavior](https://learn.microsoft.com/azure/defender-for-cloud/benefits-of-continuous-export) and the [continuous export FAQ](https://learn.microsoft.com/azure/defender-for-cloud/faq-general).

An example KQL query for an exported recommendation inventory can begin with:

```kusto
SecurityRecommendation
| summarize LatestRecord = arg_max(TimeGenerated, *)
    by RecommendationId, ResourceId
| project TimeGenerated, RecommendationId, RecommendationName,
          RecommendationSeverity, ResourceId, ResourceName,
          Status, AssessmentKey
| order by RecommendationSeverity asc, ResourceName asc
```

Export schemas and field names can differ by data source version and destination. Inspect the table schema and use the “Open query” or equivalent query generated for the recommendation rather than assuming every field is present. This example shows the intended deduplication pattern, not a universal production query.

## 8. Govern exceptions without hiding risk

No assessment system can represent every business context. A recommendation may not apply to a particular resource, or remediation may be temporarily unsafe. Use an explicit exemption or risk-acceptance process rather than silently disabling a rule or ignoring a dashboard.

A good exception record includes:

- affected recommendation and resource scope;
- reason the recommendation is not currently applicable or cannot be fixed;
- named business risk owner and technical owner;
- compensating controls and evidence;
- expiry or review date;
- change that would invalidate the exception;
- approval and audit reference.

Prefer the narrowest resource scope and a time-limited exception. Re-evaluate when architecture, data classification, network exposure, or workload criticality changes. Microsoft is transitioning away from disable rules toward exemptions in recommendation workflows; validate current behavior and migrate old suppression processes. See [Exempt resources from recommendations](https://learn.microsoft.com/azure/defender-for-cloud/exempt-resource).

An exemption should not automatically clear the operational risk from a separate risk register. Decide whether the exception affects the score or reporting model, and disclose that behavior to auditors and service owners. If an exemption is used to handle a genuine risk acceptance, preserve the approval record in the organization's system of record as well as in the cloud tool.

## 9. Integrate security data with SIEM and service management

Use continuous export when security teams need to correlate Defender alerts or recommendations with identity, endpoint, network, or application telemetry. A Log Analytics workspace can support KQL investigations and Azure Monitor alerting; Event Hubs can provide a decoupled stream to downstream platforms. Select the destination based on the consumer and retention requirement, not simply because it is already available.

Define the data flow and ownership:

- Which subscription or management-group scope exports data?
- Which event types and severities are included?
- Which workspace or Event Hub receives the records?
- Who owns retention, privacy, and access control at the destination?
- How are duplicates, delayed events, and schema changes handled?
- How does the SOC correlate recommendation events with security alerts?

If using Log Analytics, account for ingestion and retention costs in that workspace. The Defender continuous-export feature itself may have no separate fee, but the destination's storage and processing can incur charges. Review current pricing and data volume before exporting every recommendation from every cloud.

Test both initial and ongoing state behavior. Generate a known test alert or recommendation change in a safe environment, verify the record reaches the destination, validate field mapping, and confirm that a ticket or incident is created once. Then close or remediate the test finding and verify that the downstream state updates. A one-time successful event does not prove the workflow handles closure or duplicate events.

## 10. Cost, coverage, and effectiveness measurement

Defender for Cloud billing is plan- and resource-dependent. Model cost by resource type, enabled plan, cloud connector, and expected protected inventory. Do not use subscription count as the sole cost driver. For exact rates and current billable-resource rules, use the official [Defender for Cloud pricing page](https://azure.microsoft.com/pricing/details/defender-for-cloud/) and calculator.

Track three separate measures:

1. **Coverage:** in-scope assets that are inventoried, assessed, and protected by the required plan.
2. **Risk reduction:** critical/high-risk issues remediated, mitigated, or covered by an approved exception, with recurrence tracked.
3. **Operational quality:** time to assign, time to remediate, false-positive rate, exception age, automation success, and alert-to-incident conversion.

A secure score is not a financial KPI and should not be gamed. If teams are rewarded only for a score, they may prioritize low-effort score gains over the risks that matter most to the business. Use the score to identify controls and trends, and pair it with contextual risk, critical asset coverage, and incident outcomes.

Review cost and value monthly during rollout and quarterly after the service stabilizes. Look for plans enabled on out-of-scope resources, redundant endpoint products, stale connectors, resources with no owner, and exports that no longer have a consumer. Disable a plan only after validating the protection and operational impact; the goal is intentional coverage, not merely a smaller bill.

## 11. A phased implementation runbook

### Phase A: discovery and design

Inventory Azure subscriptions, management groups, AWS accounts, GCP projects, connected machines, Kubernetes clusters, repositories, and current security tools. Classify workloads by criticality and data sensitivity. Map each asset class to CSPM, CWPP, and DevSecOps requirements. Identify identity, network, regional, and data retention constraints.

### Phase B: baseline and pilot

Enable the intended no-cost posture capabilities on a representative scope, confirm the Microsoft Cloud Security Benchmark and assessment freshness, and test one workload plan in non-production. Validate connector permissions, agent or extension health, recommendation ownership, alert routing, and destination export. Record baseline findings and costs before expanding.

### Phase C: remediation workflow

Create owner mappings and severity targets. Route findings to a ticketing system with resource identity, evidence, and remediation instructions. Build an exception process with expiry. Start with manual or approval-gated remediation. Measure duplicates, assignment failures, and findings that cannot map to an owner.

### Phase D: scale through policy and automation

Use management-group policy and infrastructure as code to apply the approved configuration to new scopes. Roll out workload plans in batches and reconcile actual coverage with the resource inventory. Automate low-risk, well-tested fixes only after an owner and rollback path exist. Monitor connector health and export completeness.

### Phase E: continuous improvement

Review release notes and plan changes, assess new recommendations, retire obsolete automation, review exceptions, and revisit cost assumptions. Re-test important remediation paths after platform or service changes. Include Defender for Cloud onboarding and coverage checks in the landing-zone and application lifecycle processes.

## 12. Troubleshooting patterns

| Symptom | Likely cause | Investigation |
|---|---|---|
| A new subscription has no posture findings | Foundational CSPM not enabled or assessment not complete | Check plan state, assigned standards, scope, and assessment freshness |
| Score changes sharply after onboarding a cloud | Assessment scope expanded | Compare asset inventory, standards, recommendation set, and score model |
| Recommendations appear but no team acts | Missing owner map or ticket workflow | Check resource ownership tags, service catalogue, and workflow logs |
| AWS/GCP posture exists but machines lack protection | CSPM connected; CWPP dependencies not complete | Review plan, cloud permissions, Arc/agent/extension requirements |
| Alert is visible in Defender but absent in SIEM | Export filters, destination configuration, or event-driven semantics | Validate scope, severity filters, destination, and test a new state change |
| Recommendation was “fixed” but returns | Control drift or repeated deployment pattern | Identify source template, policy, pipeline, or service default and fix upstream |
| Exemption hides a recurring risk | Scope too broad or no expiry | Narrow resource scope, add owner and compensating control, set review date |
| Automated fix disrupts a workload | Remediation ran without impact gate | Disable that action, review audit trail, restore service, add staged validation |

## 13. Production readiness checklist

- [ ] Every subscription, cloud account, cluster, and repository has a documented in-scope status and owner.
- [ ] Foundational CSPM is explicitly enabled where required, including on new subscriptions created after the opt-in change.
- [ ] Paid Defender plans are mapped to resource types, business need, owner, and cost estimate.
- [ ] The selected portal and score model are stated in reporting definitions.
- [ ] CWPP prerequisites are verified separately from CSPM connector health.
- [ ] Recommendation assignment, due dates, escalations, and exceptions have a named process.
- [ ] Workflow automation starts with tested routing and enrichment; remediation is gated and auditable.
- [ ] Continuous export behavior is understood as change-oriented, with a separate current-state inventory where needed.
- [ ] SIEM, ticketing, and Log Analytics destinations have access, retention, and cost owners.
- [ ] A representative alert, recommendation, closure, and exception have been tested end to end.
- [ ] Release notes and plan changes are part of the recurring security platform review.

## Conclusion

Defender for Cloud is most effective when treated as an engineering feedback system rather than a score screen. The platform can unify posture and workload signals across a broad estate, but useful outcomes depend on a clear scope, deliberate plan selection, correct dependencies, risk-aware ownership, safe automation, and disciplined exception handling. Begin with a measurable pilot, make the data and responsibility boundaries visible, then scale the controls that demonstrably reduce risk.

## References

- [Microsoft Defender for Cloud overview](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-cloud-introduction)
- [Cloud Security Posture Management plans](https://learn.microsoft.com/azure/defender-for-cloud/concept-cloud-security-posture-management)
- [Defender for Cloud release notes](https://learn.microsoft.com/azure/defender-for-cloud/release-notes)
- [Cloud Secure Score](https://learn.microsoft.com/azure/defender-for-cloud/secure-score-access-and-track)
- [Review security recommendations](https://learn.microsoft.com/azure/defender-for-cloud/review-security-recommendations)
- [Multicloud CSPM and CWPP dependencies](https://learn.microsoft.com/azure/defender-for-cloud/plan-multicloud-security-determine-multicloud-dependencies)
- [Continuous export](https://learn.microsoft.com/azure/defender-for-cloud/benefits-of-continuous-export)
- [Continuous export FAQ](https://learn.microsoft.com/azure/defender-for-cloud/faq-general)
- [Exempt resources from recommendations](https://learn.microsoft.com/azure/defender-for-cloud/exempt-resource)
- [Defender for Cloud pricing](https://azure.microsoft.com/pricing/details/defender-for-cloud/)
