# Azure Resource Graph

## Why it matters

"What do we actually have?" is the first question in any assessment, audit or incident. Azure Resource Graph answers it in seconds across every subscription you can read. For a consultant it is the fastest route to a current-state picture. For an architect it measures whether governance is working. For an engineer it replaces slow loops over `az resource list`.

## Key concepts

### What it is

Resource Graph is a service that indexes resource metadata from Azure Resource Manager and lets you query it with **Kusto Query Language (KQL)**.

| Property | Detail |
|---|---|
| Scope | Any set of subscriptions or management groups you have read access to; the tenant by default in the portal |
| Freshness | Updated within seconds to minutes of a change through ARM notifications, with a regular full scan as a safety net |
| Permissions | Azure RBAC. You see only what you can read |
| Cost | Free |
| Read-only | It cannot change anything |
| What it covers | Control-plane properties (configuration). Not data-plane content, metrics or logs |

### Tables

| Table | Contains |
|---|---|
| `resources` | Most resources. The default |
| `resourcecontainers` | Management groups, subscriptions, resource groups |
| `advisorresources` | Advisor recommendations |
| `policyresources` | Policy assignments, definitions, compliance states |
| `securityresources` | Defender for Cloud assessments, alerts, secure score |
| `healthresources` | Resource health and availability status |
| `servicehealthresources` | Service Health events |
| `resourcechanges` | Change records for resource properties, kept 14 days |
| `resourcecontainerchanges` | Changes to subscriptions, resource groups, management groups |
| `authorizationresources` | Role assignments, role definitions |
| `iotsecurityresources`, `patchassessmentresources`, `kubernetesconfigurationresources`, and others | Service-specific data |
| `aksresources`, `computeresources`, `networkresources`, and others | Deeper per-service detail |

### Common columns in `resources`

`id`, `name`, `type`, `kind`, `location`, `resourceGroup`, `subscriptionId`, `tenantId`, `sku`, `tags`, `identity`, `zones`, `managedBy`, and `properties` (a dynamic bag of everything service-specific).

`type` is stored in lowercase. Compare with `=~` (case-insensitive) or write the type in lowercase.

### KQL in ten operators

| Operator | Purpose | Example |
|---|---|---|
| `where` | Filter rows | `where type =~ 'microsoft.storage/storageaccounts'` |
| `project` | Choose or compute columns | `project name, location, sku = sku.name` |
| `extend` | Add a computed column | `extend tls = tostring(properties.minimumTlsVersion)` |
| `summarize` | Aggregate | `summarize count() by location` |
| `order by` / `sort by` | Sort | `order by count_ desc` |
| `top` / `take` / `limit` | Limit rows | `top 10 by count_` |
| `join` | Combine tables | `join kind=leftouter (...) on subscriptionId` |
| `mv-expand` | Expand arrays into rows | `mv-expand ip = properties.ipConfigurations` |
| `parse` / `extract` / `split` | Pull pieces from strings | `extend vnet = tostring(split(subnetId, '/')[8])` |
| `union` | Combine result sets | `union resources, resourcecontainers` |

Resource Graph supports a **subset** of KQL. Some functions and operators from Log Analytics are unavailable, and `join` and `mv-expand` have limits per query (a few of each).

Working with `properties`:

```kusto
Resources
| where type =~ 'microsoft.storage/storageaccounts'
| extend publicBlob  = tobool(properties.allowBlobPublicAccess),
         sharedKey   = tobool(properties.allowSharedKeyAccess),
         publicNet   = tostring(properties.publicNetworkAccess),
         minTls      = tostring(properties.minimumTlsVersion)
| project name, resourceGroup, location, publicBlob, sharedKey, publicNet, minTls
```

Cast dynamic values with `tostring()`, `tobool()`, `toint()` before comparing or summarising.

### Limits

| Limit | Value |
|---|---|
| Rows per response | 1,000 (page with `$skipToken`; the CLI and portal handle paging) |
| Portal download | 55,000 rows to CSV |
| User throttle | 15 queries per 5-second window per user (tenant-level quota can be requested) |
| `resourcechanges` retention | 14 days |
| Subscriptions per query (API) | 1,000 unless you scope by management group |

Check the `x-ms-user-quota-remaining` header when automating.

## How it works on Azure

### Where to run queries

| Tool | How |
|---|---|
| **Portal** | *Resource Graph Explorer*: schema browser, charts, pin to dashboard, save queries |
| **Azure CLI** | `az graph query -q "<query>"` (extension `resource-graph`) |
| **PowerShell** | `Search-AzGraph -Query "<query>"` |
| **REST / SDKs** | `POST /providers/Microsoft.ResourceGraph/resources` |
| **Azure Monitor workbooks** | Resource Graph as a data source for dashboards |
| **Azure Copilot** | Ask in natural language; it generates the query |
| **Power BI** | Resource Graph connector |
| **Logic Apps / Functions** | For scheduled reports and alerts |

**Shared queries** are saved as Azure resources with RBAC, so a team can maintain a library.

### A query library for assessments

**Inventory**

```kusto
// Resource count by type
Resources
| summarize count() by type
| order by count_ desc

// Resources by region and subscription name
Resources
| join kind=leftouter (
    ResourceContainers
    | where type =~ 'microsoft.resources/subscriptions'
    | project subscriptionId, subscriptionName = name
  ) on subscriptionId
| summarize resources = count() by subscriptionName, location
| order by resources desc

// Subscriptions with their management group ancestry
ResourceContainers
| where type =~ 'microsoft.resources/subscriptions'
| extend mgChain = properties.managementGroupAncestorsChain
| project name, subscriptionId, state = properties.state,
          parentMg = tostring(mgChain[0].displayName)
```

**Governance**

```kusto
// Tag coverage for a mandatory tag
Resources
| extend hasOwner = isnotempty(tags['owner'])
| summarize total = count(), tagged = countif(hasOwner) by subscriptionId
| extend coveragePct = round(100.0 * tagged / total, 1)

// Naming compliance for resource groups
ResourceContainers
| where type =~ 'microsoft.resources/subscriptions/resourcegroups'
| extend compliant = name startswith 'rg-'
| summarize count() by compliant

// Non-compliant resources by policy assignment
PolicyResources
| where type =~ 'microsoft.policyinsights/policystates'
| where properties.complianceState =~ 'NonCompliant'
| summarize nonCompliant = count() by assignment = tostring(properties.policyAssignmentName)
| order by nonCompliant desc

// Policy exemptions and when they expire
PolicyResources
| where type =~ 'microsoft.authorization/policyexemptions'
| project name, category = properties.exemptionCategory,
          expires = todatetime(properties.expiresOn), scope = id
| order by expires asc
```

**Security posture**

```kusto
// Public IP addresses and what they are attached to
Resources
| where type =~ 'microsoft.network/publicipaddresses'
| project name, ip = tostring(properties.ipAddress),
          attachedTo = tostring(properties.ipConfiguration.id), resourceGroup

// PaaS resources with public network access enabled
Resources
| where type in~ ('microsoft.storage/storageaccounts','microsoft.keyvault/vaults',
                  'microsoft.cognitiveservices/accounts','microsoft.search/searchservices',
                  'microsoft.documentdb/databaseaccounts','microsoft.containerregistry/registries')
| extend publicAccess = tostring(properties.publicNetworkAccess)
| where publicAccess !~ 'Disabled'
| project type, name, resourceGroup, publicAccess

// Key Vaults still on access policies instead of RBAC
Resources
| where type =~ 'microsoft.keyvault/vaults'
| where tobool(properties.enableRbacAuthorization) != true
| project name, resourceGroup

// Owner assignments at subscription scope
AuthorizationResources
| where type =~ 'microsoft.authorization/roleassignments'
| extend roleId = tostring(split(properties.roleDefinitionId, '/')[-1]),
         scope = tostring(properties.scope)
| where roleId == '8e3af657-a8ff-443c-a75c-2fe8c4bcb635'      // Owner
| where scope matches regex '^/subscriptions/[^/]+$'
| project principalId = properties.principalId, principalType = properties.principalType, scope

// Unhealthy Defender for Cloud recommendations by severity
SecurityResources
| where type =~ 'microsoft.security/assessments'
| where properties.status.code =~ 'Unhealthy'
| summarize count() by severity = tostring(properties.metadata.severity)
```

**AI estate**

```kusto
// Every Foundry / AI services resource with its key governance settings
Resources
| where type =~ 'microsoft.cognitiveservices/accounts'
| project name, kind, location, sku = tostring(sku.name), resourceGroup, subscriptionId,
          publicAccess     = tostring(properties.publicNetworkAccess),
          localAuthDisabled = tobool(properties.disableLocalAuth),
          privateEndpoints = array_length(properties.privateEndpointConnections),
          identityType     = tostring(identity.type),
          customDomain     = tostring(properties.customSubDomainName)

// AI resources that still allow keys or public access
Resources
| where type in~ ('microsoft.cognitiveservices/accounts','microsoft.search/searchservices')
| extend localAuthDisabled = tobool(properties.disableLocalAuth),
         publicAccess = tostring(properties.publicNetworkAccess)
| where localAuthDisabled != true or publicAccess !~ 'Disabled'
| project type, name, resourceGroup, localAuthDisabled, publicAccess

// Where are AI resources concentrated?
Resources
| where type =~ 'microsoft.cognitiveservices/accounts'
| summarize count() by kind, location
| order by count_ desc

// Azure AI Search services: tier, replicas, partitions
Resources
| where type =~ 'microsoft.search/searchservices'
| project name, sku = tostring(sku.name),
          replicas = toint(properties.replicaCount), partitions = toint(properties.partitionCount),
          semantic = tostring(properties.semanticSearch), location

// GPU virtual machines
Resources
| where type =~ 'microsoft.compute/virtualmachines'
| extend size = tostring(properties.hardwareProfile.vmSize)
| where size startswith 'Standard_N'
| project name, size, location, resourceGroup,
          power = tostring(properties.extended.instanceView.powerState.displayStatus)
```

Model **deployments** are child resources. Depending on current indexing they may not appear in `resources`; list them with `az cognitiveservices account deployment list` per account, driven by the first query's output.

**Cost and hygiene**

```kusto
// Unattached managed disks
Resources
| where type =~ 'microsoft.compute/disks'
| where properties.diskState =~ 'Unattached'
| project name, sizeGb = toint(properties.diskSizeGB), sku = tostring(sku.name), resourceGroup

// Orphaned public IPs and NICs
Resources
| where (type =~ 'microsoft.network/publicipaddresses' and isnull(properties.ipConfiguration))
     or (type =~ 'microsoft.network/networkinterfaces' and isnull(properties.virtualMachine))
| project type, name, resourceGroup

// Empty resource groups
ResourceContainers
| where type =~ 'microsoft.resources/subscriptions/resourcegroups'
| extend rgId = tolower(id)
| join kind=leftouter (
    Resources | extend rgId = tolower(strcat('/subscriptions/', subscriptionId, '/resourceGroups/', resourceGroup))
    | summarize n = count() by rgId
  ) on rgId
| where isnull(n)
| project name, subscriptionId, location

// Advisor cost recommendations
AdvisorResources
| where type =~ 'microsoft.advisor/recommendations'
| where properties.category =~ 'Cost'
| project problem = tostring(properties.shortDescription.problem),
          resource = tostring(properties.impactedValue),
          savings = todouble(properties.extendedProperties.savingsAmount)
| order by savings desc
```

**Reliability**

```kusto
// VMs not in an availability zone
Resources
| where type =~ 'microsoft.compute/virtualmachines'
| where isnull(zones) or array_length(zones) == 0
| project name, location, resourceGroup

// Storage redundancy overview
Resources
| where type =~ 'microsoft.storage/storageaccounts'
| summarize count() by redundancy = tostring(sku.name)

// Resources currently unavailable
HealthResources
| where type =~ 'microsoft.resourcehealth/availabilitystatuses'
| where properties.availabilityState !~ 'Available'
| project resource = tostring(properties.targetResourceId), state = properties.availabilityState,
          reason = properties.reasonType
```

**Change tracking**

```kusto
// What changed in the last 24 hours, and who changed it
ResourceChanges
| extend changeTime = todatetime(properties.changeAttributes.timestamp),
         changeType = tostring(properties.changeType),
         changedBy  = tostring(properties.changeAttributes.changedBy),
         client     = tostring(properties.changeAttributes.clientType),
         target     = tostring(properties.targetResourceId)
| where changeTime > ago(1d)
| project changeTime, changeType, changedBy, client, target, changes = properties.changes
| order by changeTime desc

// Was public access recently re-enabled anywhere?
ResourceChanges
| extend changeTime = todatetime(properties.changeAttributes.timestamp)
| where changeTime > ago(7d)
| mv-expand change = properties.changes
| extend prop = tostring(bag_keys(change)[0])
| where prop has 'publicNetworkAccess'
| project changeTime, target = tostring(properties.targetResourceId),
          before = tostring(change[prop].previousValue), after = tostring(change[prop].newValue),
          by = tostring(properties.changeAttributes.changedBy)
```

Change records are a quick first stop in an incident: "what changed just before it broke?"

### Resource Graph and throttling

Resource Graph offers a way to serve ARM GET and LIST calls from its index, which helps tools that would otherwise hit resource provider throttling when reading at scale. For your own automation the simpler rule holds: one Resource Graph query instead of hundreds of GET calls.

## Design decisions and trade-offs

| Need | Use |
|---|---|
| Current configuration across subscriptions | Resource Graph |
| Who did what, with full request detail | Activity log |
| Property changes over the last two weeks | `resourcechanges` |
| Longer history and alerting | Export to Log Analytics; Azure Monitor alerts support Resource Graph queries through workspace functions |
| Metrics and telemetry | Azure Monitor |
| Data-plane content | The service's own API |
| Cost | Cost Management (join with Resource Graph results on resource ID in a report) |

**Dashboards:** build governance workbooks on Resource Graph queries (tag coverage, public exposure, policy compliance, AI resource posture) and review them in a regular governance meeting. Numbers that are looked at get better.

## Hands-on

```bash
az extension add --name resource-graph 2>/dev/null

# Count by type
az graph query -q "Resources | summarize count() by type | order by count_ desc" --first 20 -o table

# Scope to a management group
az graph query -q "Resources | summarize count() by location" --management-groups <mg-id> -o table

# Paging beyond 1,000 rows
az graph query -q "Resources | project id, name, type" --first 1000 --skip 1000 -o table

# Export to JSON for a report
az graph query -q "Resources | where type =~ 'microsoft.cognitiveservices/accounts' | project name, kind, location, properties.publicNetworkAccess" -o json > ai-inventory.json
```

Exercises:

1. Run five queries from each section above against your environment.
2. Write a query that lists every resource group with its `owner` tag and subscription name.
3. Write a query that reports, per subscription, the percentage of storage accounts with `allowSharedKeyAccess` disabled.
4. Find everything you changed in the last seven days.
5. Save your best three as **shared queries** and pin one chart to a dashboard.
6. Ask Azure Copilot for "virtual machines without backup" and compare its query with one you would write.

## Security, cost and reliability notes

- **Security:** results respect RBAC, so two people can get different answers from the same query. State the scope and identity used in any assessment report.
- **Security:** Resource Graph shows configuration, not secrets, but an inventory of public endpoints and weak settings is sensitive. Handle exports accordingly.
- **Cost:** free. Use it liberally.
- **Reliability:** the index can lag briefly. For a just-created resource, query ARM directly.

## Common mistakes

- Comparing `type` with case-sensitive `==` and mixed-case strings.
- Forgetting to cast dynamic `properties` values, giving empty or wrong aggregations.
- Reading only the first 1,000 rows and reporting it as the total.
- Expecting metrics, logs or blob contents.
- Running broad queries in a tight loop and hitting the throttle.
- Assuming you can see the whole tenant when your access covers only some subscriptions.
- Looking for changes older than 14 days.

## Check yourself

1. What does Resource Graph index, and what does it not?
2. Which table holds subscriptions and resource groups?
3. Why use `=~` when filtering on `type`?
4. How many rows does a single response return, and how do you get more?
5. Write a query counting storage accounts by SKU.
6. How would you find AI resources that still permit key authentication?
7. Where would you look to see what changed on a resource yesterday, and how far back can you go?

<details>
<summary>Answers</summary>

1. Control-plane metadata and configuration of resources, plus related data such as policy states, Advisor and security assessments, health and recent changes. Not metrics, logs or data-plane content.
2. `resourcecontainers`.
3. Types are stored in lowercase; `=~` is case-insensitive.
4. 1,000. Page with a skip token (`--skip` / `--skip-token`).
5. `Resources | where type =~ 'microsoft.storage/storageaccounts' | summarize count() by tostring(sku.name)`
6. Query `microsoft.cognitiveservices/accounts` and `microsoft.search/searchservices` where `properties.disableLocalAuth` is not true.
7. The `resourcechanges` table, which keeps 14 days.

</details>

## References

- [What is Azure Resource Graph?](https://learn.microsoft.com/azure/governance/resource-graph/overview)
- [Understanding the Azure Resource Graph query language](https://learn.microsoft.com/azure/governance/resource-graph/concepts/query-language)
- [Supported tables and resource types](https://learn.microsoft.com/azure/governance/resource-graph/reference/supported-tables-resources)
- [Starter query samples](https://learn.microsoft.com/azure/governance/resource-graph/samples/starter)
- [Advanced query samples](https://learn.microsoft.com/azure/governance/resource-graph/samples/advanced)
- [Get resource changes](https://learn.microsoft.com/azure/governance/resource-graph/changes/get-resource-changes)
- [Guidance for throttled requests](https://learn.microsoft.com/azure/governance/resource-graph/concepts/guidance-for-throttled-requests)
- [Working with large data sets](https://learn.microsoft.com/azure/governance/resource-graph/concepts/work-with-data)
