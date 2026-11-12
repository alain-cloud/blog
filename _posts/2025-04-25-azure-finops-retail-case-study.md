---
layout: post
title: "Azure FinOps in practice: how a retail platform cut its monthly cloud run rate by 31%"
date: 2025-04-25 09:00:00 +0200
categories: [FinOps, Azure]
tags: [Azure, FinOps, Cost Management, Azure Advisor, PowerShell, KQL, Bicep, Azure Policy, Automation]
description: "A detailed fictionalised case study showing how a retail company reduced Azure waste, improved cost ownership and built a sustainable FinOps operating model in 90 days."
---

> **Transparency note:** This is a fictionalised, technically realistic case study built from common Azure FinOps patterns. The company, workload, people and financial figures are invented. The code is provided as a reusable starting point and must be tested before production use.

## Executive summary

A fictional European retailer, **Northstar Retail**, operated an Azure-based commerce platform across 14 subscriptions. The environment supported e-commerce, store integration, product data, analytics and internal development platforms.

By January 2025, the monthly Azure run rate had reached **€286,000**, up 38% in nine months. The increase was not linked to equivalent business growth. Cost allocation was incomplete, development resources ran continuously, several compute services were oversized, storage lifecycle rules were inconsistent, and commitment discounts had been purchased without a sufficiently stable baseline.

A 90-day FinOps programme reduced the validated monthly run rate from **€286,000 to €197,000**, representing **€89,000 per month**, or **31.1%**. The programme deliberately separated:

- gross opportunities identified;
- changes approved;
- changes implemented;
- savings visible on the invoice;
- avoided future cost;
- one-time credits or commercial adjustments.

No production resource was resized solely from an Azure Advisor recommendation. Every production change required owner confirmation, performance evidence, a rollback plan and an agreed observation period.

## Business and technical context

Northstar Retail ran a hybrid commerce platform with the following Azure footprint:

| Area | Main Azure services | Business purpose |
|---|---|---|
| Digital commerce | App Service, Application Gateway, Azure SQL | Public e-commerce platform |
| Store integration | Service Bus, Functions, API Management | POS and inventory integration |
| Data platform | Data Factory, Databricks, ADLS Gen2 | Sales and product analytics |
| Shared platform | Firewall, Bastion, Log Analytics, Private DNS | Connectivity and security services |
| Engineering | AKS, Container Registry, DevTest Labs | Development and integration testing |
| Business continuity | Recovery Services Vault, paired-region resources | Backup and disaster recovery |

The subscription model had grown organically:

```text
Management Group: Northstar
├── Platform
│   ├── Connectivity-Prod
│   ├── Identity-Prod
│   └── Management-Prod
├── Landing-Zones
│   ├── Commerce-Prod
│   ├── Commerce-NonProd
│   ├── Data-Prod
│   ├── Data-NonProd
│   ├── Integration-Prod
│   └── Integration-NonProd
└── Sandbox
    ├── Innovation
    └── Developer-Labs
```

The design itself was not the main issue. The problem was that financial ownership and technical lifecycle controls had not matured at the same pace as consumption.

## The starting baseline

The January 2025 cost baseline was normalised to remove VAT, marketplace pass-through charges, support fees and one-time credits.

| Cost family | Monthly cost | Share |
|---|---:|---:|
| Compute, App Service and AKS | €124,000 | 43.4% |
| Data and databases | €61,000 | 21.3% |
| Storage and backup | €34,000 | 11.9% |
| Network and security | €29,000 | 10.1% |
| Monitoring and operations | €23,000 | 8.0% |
| Other services | €15,000 | 5.3% |
| **Total** | **€286,000** | **100%** |

Only 57% of cost could be reliably mapped to an application owner. The rest was assigned to generic resource groups, shared subscriptions or incomplete tags.

## What we found during discovery

The team used exports from Azure Cost Management, Azure Resource Graph, Azure Advisor, Azure Monitor and application-owner interviews. The resulting opportunity register contained 73 items.

The most material findings were:

1. **Thirty-eight non-production VMs ran 24/7**, despite an agreed support window of 07:00–20:00 on weekdays.
2. **Eleven production VMs showed sustained CPU below 10%**, but memory data was incomplete, so no immediate resize was approved.
3. **Three App Service Plans were isolated per application** even though workload characteristics allowed consolidation.
4. **AKS node pools used fixed capacity**, including six development nodes overnight and at weekends.
5. **€8,700 per month of unattached managed disks and old snapshots** had no active owner.
6. **Log Analytics ingestion had doubled** after diagnostic settings were enabled broadly without table-level retention decisions.
7. **ADLS Gen2 contained 410 TB in the hot tier**, including data older than 180 days that had not been accessed.
8. **Reservations covered volatile workloads**, while several stable database and compute baselines remained on pay-as-you-go pricing.
9. **Azure Hybrid Benefit was not enabled consistently** for eligible Windows Server and SQL Server workloads.
10. **Cost alerts were subscription-level only**, providing no useful accountability to application teams.

## Phase 1: establish trustworthy cost data

### Define the cost allocation model

The team agreed on six mandatory business tags:

```yaml
Application: commerce-web
Environment: prod
BusinessUnit: digital
CostCentre: CC-4102
BusinessOwner: digital-commerce
TechnicalOwner: platform-commerce
```

Two additional operational tags were introduced:

```yaml
Criticality: tier-1
ShutdownSchedule: weekday-0700-2000
```

Tags were not treated as the only source of truth. Subscription and resource-group inheritance was used where appropriate, and shared services were allocated using agreed drivers.

For example:

- Azure Firewall costs were allocated by processed traffic;
- ExpressRoute was allocated by connected landing-zone percentage;
- central Log Analytics cost was allocated by billable ingestion per application;
- Microsoft Defender for Cloud cost was allocated by protected resource count;
- shared platform engineering cost remained in a transparent central pool.

### Find resources with missing tags using Azure Resource Graph

The following query identifies resources missing one or more mandatory tags:

```kusto
Resources
| extend application = tostring(tags['Application'])
| extend environment = tostring(tags['Environment'])
| extend costCentre = tostring(tags['CostCentre'])
| extend businessOwner = tostring(tags['BusinessOwner'])
| extend technicalOwner = tostring(tags['TechnicalOwner'])
| where isempty(application)
    or isempty(environment)
    or isempty(costCentre)
    or isempty(businessOwner)
    or isempty(technicalOwner)
| project subscriptionId,
          resourceGroup,
          name,
          type,
          location,
          application,
          environment,
          costCentre,
          businessOwner,
          technicalOwner
| order by subscriptionId asc, resourceGroup asc
```

### Export the same control with Azure CLI

```bash
az graph query -q "Resources
| extend Application=tostring(tags.Application), Environment=tostring(tags.Environment), CostCentre=tostring(tags.CostCentre)
| where isempty(Application) or isempty(Environment) or isempty(CostCentre)
| project subscriptionId, resourceGroup, name, type, Application, Environment, CostCentre" \
--first 5000 \
--output table
```

### Trick: measure tag compliance by cost, not only resource count

A compliance score based only on resource count can be misleading. One untagged Databricks workspace may matter more than 100 inexpensive resources.

The team therefore published two KPIs:

```text
Resource tag compliance = compliant resources / total resources
Cost tag compliance     = allocated monthly cost / total allocatable monthly cost
```

The first month showed 82% resource compliance but only 57% cost compliance. This immediately changed the prioritisation.

## Phase 2: stop obvious waste safely

### Detect stopped but still allocated VMs

A VM in the `PowerState/stopped` state may still incur compute charges. The desired state for releasing compute capacity is `deallocated`.

```powershell
$subscriptions = Get-AzSubscription
$results = foreach ($subscription in $subscriptions) {
    Set-AzContext -SubscriptionId $subscription.Id | Out-Null

    Get-AzVM -Status | ForEach-Object {
        $powerState = ($_.Statuses | Where-Object Code -like 'PowerState/*').DisplayStatus

        if ($powerState -eq 'VM stopped') {
            [PSCustomObject]@{
                Subscription = $subscription.Name
                ResourceGroup = $_.ResourceGroupName
                VM = $_.Name
                Location = $_.Location
                PowerState = $powerState
                Recommendation = 'Deallocate if shutdown is intended'
            }
        }
    }
}

$results | Export-Csv -Path './stopped-not-deallocated.csv' -NoTypeInformation
```

### Schedule non-production shutdown

The team used Azure Automation for existing VMs and required a schedule tag for new non-production VMs.

A simplified PowerShell runbook:

```powershell
param(
    [Parameter(Mandatory = $false)]
    [string]$RequiredTag = 'ShutdownSchedule',

    [Parameter(Mandatory = $false)]
    [string]$ScheduleValue = 'weekday-0700-2000'
)

Connect-AzAccount -Identity

$today = (Get-Date).DayOfWeek
$isWeekend = $today -in @('Saturday', 'Sunday')

Get-AzSubscription | ForEach-Object {
    Set-AzContext -SubscriptionId $_.Id | Out-Null

    $targets = Get-AzVM -Status | Where-Object {
        $_.Tags[$RequiredTag] -eq $ScheduleValue
    }

    foreach ($vm in $targets) {
        $powerState = ($vm.Statuses | Where-Object Code -like 'PowerState/*').DisplayStatus

        if (($isWeekend -or (Get-Date).Hour -ge 20) -and $powerState -eq 'VM running') {
            Write-Output "Deallocating $($vm.Name) in $($vm.ResourceGroupName)"
            Stop-AzVM \
                -ResourceGroupName $vm.ResourceGroupName \
                -Name $vm.Name \
                -Force \
                -NoWait
        }
    }
}
```

> **Production safeguard:** The runbook identity received permission only on approved non-production resource groups. Production subscriptions were explicitly excluded from the automation scope.

### Trick: calculate realistic shutdown savings

Do not claim 65% savings simply because a VM is off for 65% of the week. Disks, backup, monitoring, networking and reservations may continue to cost money.

Use:

```text
Avoidable compute cost = PAYG compute rate × hours avoided
Net saving = avoidable compute cost - automation cost - operational exceptions
```

If the VM is covered by a reservation, shutdown may not reduce the invoice unless the reservation benefit is consumed elsewhere.

## Phase 3: rightsizing with evidence

### The three-signal rule

A production resize required three independent signals:

1. Azure Advisor or cost-engine recommendation;
2. at least 14–30 days of CPU, memory, disk and network evidence;
3. confirmation from the application owner regarding peaks, batch windows and planned growth.

CPU alone was never considered sufficient.

### Query CPU utilisation with Log Analytics

```kusto
Perf
| where TimeGenerated > ago(30d)
| where ObjectName == 'Processor'
| where CounterName == '% Processor Time'
| where InstanceName == '_Total'
| summarize
    AvgCPU = avg(CounterValue),
    P95CPU = percentile(CounterValue, 95),
    MaxCPU = max(CounterValue)
    by Computer
| where P95CPU < 25
| order by P95CPU asc
```

### Query available memory

```kusto
Perf
| where TimeGenerated > ago(30d)
| where ObjectName == 'Memory'
| where CounterName == 'Available MBytes'
| summarize
    MinAvailableMB = min(CounterValue),
    P05AvailableMB = percentile(CounterValue, 5),
    AvgAvailableMB = avg(CounterValue)
    by Computer
| order by P05AvailableMB asc
```

### Example resize decision

A `Standard_D8s_v5` VM showed:

- average CPU: 6%;
- P95 CPU: 18%;
- maximum CPU: 67% during a weekly batch;
- minimum available memory: 13.4 GB;
- no sustained disk queue;
- no planned volume growth.

The team tested `Standard_D4s_v5` in pre-production, then resized production during a maintenance window. The change was observed for seven days with an agreed rollback command:

```bash
az vm resize \
  --resource-group rg-commerce-prod-weu \
  --name vm-commerce-worker-01 \
  --size Standard_D4s_v5
```

Rollback:

```bash
az vm resize \
  --resource-group rg-commerce-prod-weu \
  --name vm-commerce-worker-01 \
  --size Standard_D8s_v5
```

### Trick: verify SKU availability before approving a resize

Not every VM size is available in every cluster. Check before the maintenance window:

```bash
az vm list-vm-resize-options \
  --resource-group rg-commerce-prod-weu \
  --name vm-commerce-worker-01 \
  --query "[].name" \
  --output table
```

## Phase 4: optimise AKS without damaging resilience

The development AKS cluster used a fixed six-node pool. The team enabled cluster autoscaling and separated system and user workloads.

```bash
az aks nodepool update \
  --resource-group rg-aks-dev-weu \
  --cluster-name aks-northstar-dev \
  --name userpool \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 6
```

Pod requests were then reviewed because autoscaling cannot work properly when requests are absent or inflated.

Example deployment resources:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi
```

A Horizontal Pod Autoscaler was added for the API layer:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: inventory-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: inventory-api
  minReplicas: 2
  maxReplicas: 12
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 65
```

### Trick: optimise requests before node count

The team found pods requesting four times their observed P95 memory. Reducing inflated requests allowed the scheduler to pack workloads more efficiently and avoided adding nodes.

The sequence was:

```text
Measure pod usage → correct requests → validate limits → enable HPA → tune cluster autoscaler
```

Not:

```text
Reduce nodes first → create resource pressure → investigate incidents later
```

## Phase 5: storage and snapshot hygiene

### Find unattached managed disks with Azure Resource Graph

```kusto
Resources
| where type =~ 'microsoft.compute/disks'
| extend diskState = tostring(properties.diskState)
| where diskState == 'Unattached'
| project subscriptionId,
          resourceGroup,
          name,
          location,
          sku = tostring(sku.name),
          sizeGB = toint(properties.diskSizeGB),
          timeCreated = todatetime(properties.timeCreated),
          tags
| order by sizeGB desc
```

### Find old snapshots

```kusto
Resources
| where type =~ 'microsoft.compute/snapshots'
| extend created = todatetime(properties.timeCreated)
| extend ageDays = datetime_diff('day', now(), created)
| where ageDays > 90
| project subscriptionId, resourceGroup, name, location, created, ageDays,
          sizeGB = toint(properties.diskSizeGB), tags
| order by ageDays desc
```

### Safe deletion workflow

No script deleted disks automatically. The workflow was:

```text
Discover → identify owner → confirm dependency → record approval → apply 14-day quarantine tag → delete → monitor
```

Example quarantine tags:

```bash
az disk update \
  --resource-group rg-legacy-app-prod \
  --name disk-old-data-02 \
  --set tags.DeletionCandidate=true tags.DeleteAfter=2025-03-15 tags.ApprovedBy=app-owner
```

### Apply a lifecycle policy to a storage account

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "tier-analytics-data",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterLastAccessTimeGreaterThan": 30
            },
            "tierToArchive": {
              "daysAfterLastAccessTimeGreaterThan": 180
            },
            "delete": {
              "daysAfterModificationGreaterThan": 2555
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["analytics/raw/"]
        }
      }
    }
  ]
}
```

> **Risk:** Archive retrieval has latency and retrieval charges. Lifecycle rules must reflect recovery time, legal retention and access patterns, not only storage price.

## Phase 6: control Log Analytics cost

The team analysed ingestion by table:

```kusto
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize BillableGB = sum(Quantity) / 1000 by DataType
| order by BillableGB desc
```

A daily trend exposed a sudden increase:

```kusto
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize BillableGB = sum(Quantity) / 1000 by bin(TimeGenerated, 1d), DataType
| render timechart
```

The main cause was verbose application logs and duplicate diagnostic routing. The remediation included:

- removing duplicated diagnostic settings;
- filtering unnecessary application events at source;
- selecting resource-specific tables;
- setting table retention according to security and operational need;
- creating a daily ingestion budget alert.

### Trick: use ingestion cost per service transaction

Absolute log cost can rise with business activity. The more useful KPI was:

```text
Log observability unit cost = monthly monitoring cost / completed customer orders
```

This distinguished healthy business growth from uncontrolled telemetry growth.

## Phase 7: purchase commitments only after optimisation

The team waited until rightsizing and shutdown changes were implemented before purchasing additional commitments.

The commitment decision tree was:

```text
Is workload stable for at least 30–60 days?
├── No → remain PAYG and improve elasticity
└── Yes
    ├── Is the exact VM family/region predictable? → consider Reservation
    ├── Is compute usage stable but flexible across eligible services? → consider Savings Plan
    └── Are Windows/SQL licences eligible? → validate Azure Hybrid Benefit
```

The model used three scenarios:

| Scenario | Coverage target | Rationale |
|---|---:|---|
| Conservative | 60% | Protect against demand and architecture changes |
| Balanced | 75% | Cover stable baseline and retain flexibility |
| Aggressive | 90% | Maximum discount but higher utilisation risk |

Northstar selected the balanced scenario. The team tracked both:

```text
Commitment utilisation = used commitment / purchased commitment
Commitment coverage    = eligible usage covered / total eligible usage
```

High utilisation alone was not proof of a good purchase if coverage remained too low. Similarly, high coverage with low utilisation could indicate overcommitment.

## Phase 8: deploy guardrails with Azure Policy

The following policy denies selected expensive VM SKUs in non-production subscriptions. In practice, the allowed list must be managed carefully and exceptions documented.

```json
{
  "properties": {
    "displayName": "Restrict VM SKUs in non-production",
    "policyType": "Custom",
    "mode": "Indexed",
    "parameters": {
      "allowedSkus": {
        "type": "Array",
        "metadata": {
          "displayName": "Allowed VM SKUs"
        }
      }
    },
    "policyRule": {
      "if": {
        "allOf": [
          {
            "field": "type",
            "equals": "Microsoft.Compute/virtualMachines"
          },
          {
            "field": "Microsoft.Compute/virtualMachines/sku.name",
            "notIn": "[parameters('allowedSkus')]"
          }
        ]
      },
      "then": {
        "effect": "deny"
      }
    }
  }
}
```

Example assignment with Azure CLI:

```bash
az policy definition create \
  --name restrict-nonprod-vm-skus \
  --display-name "Restrict VM SKUs in non-production" \
  --rules policy-rule.json \
  --params policy-parameters.json \
  --mode Indexed

az policy assignment create \
  --name restrict-nonprod-vm-skus \
  --policy restrict-nonprod-vm-skus \
  --scope /subscriptions/00000000-0000-0000-0000-000000000000 \
  --params '{"allowedSkus":{"value":["Standard_B2s","Standard_B4ms","Standard_D2s_v5","Standard_D4s_v5"]}}'
```

### Trick: audit before deny

The team first deployed cost policies with `audit` or `modify` effects for two sprints. A deny policy was introduced only after measuring impact and establishing an exception path.

## Phase 9: make cost visible in infrastructure as code

A Bicep example added mandatory tags and a cost-conscious default SKU:

```bicep
@description('Deployment environment')
@allowed([
  'dev'
  'test'
  'prod'
])
param environment string

@description('Application name')
param applicationName string

@description('Cost centre')
param costCentre string

@description('App Service Plan SKU')
param skuName string = environment == 'prod' ? 'P1v3' : 'B2'

var commonTags = {
  Application: applicationName
  Environment: environment
  CostCentre: costCentre
  ManagedBy: 'Bicep'
}

resource plan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'asp-${applicationName}-${environment}-weu'
  location: resourceGroup().location
  tags: commonTags
  sku: {
    name: skuName
    capacity: environment == 'prod' ? 2 : 1
  }
  properties: {
    reserved: true
  }
}
```

### Add a cost-control check in GitHub Actions

```yaml
name: Validate infrastructure

on:
  pull_request:
    paths:
      - 'infra/**'

jobs:
  validate:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Build Bicep
        run: az bicep build --file infra/main.bicep

      - name: What-if deployment
        run: |
          az deployment sub what-if \
            --location westeurope \
            --template-file infra/main.bicep \
            --parameters infra/dev.bicepparam

      - name: Reject unapproved premium SKU
        shell: bash
        run: |
          if grep -R "P3v3\|E16ds_v5\|M64ms" infra/; then
            echo "High-cost SKU detected. Architecture approval required."
            exit 1
          fi
```

> **Important:** A simple string check is only a basic guardrail. Mature implementations should parse deployment plans, integrate an IaC cost-estimation tool, and use an approval workflow for exceptions.

## Governance model

The programme introduced a monthly FinOps review with four roles:

| Role | Accountability |
|---|---|
| Application owner | Confirms business need and accepts service trade-offs |
| Engineering owner | Implements and validates technical changes |
| Finance partner | Validates baseline and realised financial impact |
| FinOps lead | Maintains backlog, KPIs, methodology and governance cadence |

The optimisation backlog contained:

```text
ID
Subscription
Application
Resource
Recommendation
Technical evidence
Risk
One-time effort
Estimated monthly saving
Approved monthly saving
Realised monthly saving
Owner
Target date
Status
Rollback plan
```

## Financial results after 90 days

| Action | Validated monthly saving |
|---|---:|
| Non-production scheduling | €18,400 |
| VM and database rightsizing | €16,800 |
| AKS requests and autoscaling | €11,600 |
| Storage, snapshots and lifecycle | €9,700 |
| Log Analytics optimisation | €7,900 |
| App Service consolidation | €6,300 |
| Reservations and Savings Plan changes | €12,400 |
| Azure Hybrid Benefit | €5,900 |
| **Total** | **€89,000** |

The normalised monthly run rate moved from **€286,000 to €197,000**.

```text
Monthly reduction = €89,000
Reduction rate    = €89,000 / €286,000 = 31.1%
Annualised value  = €89,000 × 12 = €1,068,000
```

The annualised figure was presented as a run-rate projection, not as guaranteed cash savings. It assumed that workloads and prices remained broadly stable.

## What did not work

Several early ideas were rejected or corrected:

### 1. Resizing from CPU alone

The first report proposed resizing VMs with average CPU below 10%. This was rejected because memory, batch peaks, disk throughput and business forecasts were missing.

### 2. Buying reservations before rightsizing

The team initially considered purchasing commitments against the January estate. Doing so would have locked discounts onto oversized resources.

### 3. Deleting every unattached disk

Some disks were intentional recovery assets or linked to change windows. Discovery was automated, but deletion remained approval-driven.

### 4. Using tags as the only allocation mechanism

Shared services required allocation drivers. Tags alone could not fairly distribute Firewall, ExpressRoute or central monitoring costs.

### 5. Reporting theoretical savings as realised savings

Azure Advisor recommendations were recorded as opportunities, not benefits. Savings became realised only after the invoice baseline reflected the change.

## Reusable FinOps checklist

### Visibility

- [ ] Cost Management exports enabled at the correct scope
- [ ] Cost allocation rules documented
- [ ] Resource and cost tag-compliance KPIs published
- [ ] Daily and monthly anomaly alerts configured
- [ ] Shared-cost allocation drivers agreed

### Optimisation

- [ ] Non-production schedules validated
- [ ] Rightsizing uses CPU, memory, disk, network and business context
- [ ] Unattached disks and snapshots have an approval workflow
- [ ] Storage lifecycle policies reflect retrieval and compliance needs
- [ ] AKS requests, limits, HPA and cluster autoscaling reviewed together
- [ ] Log Analytics ingestion analysed by table

### Commitments

- [ ] Rightsizing completed before commitment purchase
- [ ] Stable baseline measured for at least 30–60 days
- [ ] Reservation and Savings Plan scenarios compared
- [ ] Azure Hybrid Benefit eligibility validated
- [ ] Coverage and utilisation reviewed monthly

### Governance

- [ ] Monthly FinOps review established
- [ ] Backlog includes owner, risk, target date and realised saving
- [ ] Policies introduced in audit mode before deny
- [ ] Exceptions are time-bound and documented
- [ ] Savings methodology agreed with finance

## Final takeaways

This case demonstrates that Azure cost optimisation is not a single technical exercise. The largest benefits came from combining financial data, platform telemetry, architecture decisions, automation and clear ownership.

The most reusable lessons were:

1. **Build a trustworthy baseline before promising savings.**
2. **Optimise demand before purchasing discounts.**
3. **Treat production rightsizing as an engineering change, not a spreadsheet action.**
4. **Automate discovery, but require approval for destructive actions.**
5. **Measure realised savings separately from recommendations and avoided cost.**
6. **Make cost visible during design and deployment, not only after invoicing.**
7. **Use policies and automation to prevent waste from returning.**

FinOps becomes sustainable when cost is treated as an architectural signal and a shared operational responsibility, rather than a finance report delivered after the month has closed.
