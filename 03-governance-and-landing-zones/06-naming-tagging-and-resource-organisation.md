# Naming, Tagging and Resource Organisation

> Topic: 03 · Governance and Landing Zones · Tracks: E ● A ● C ○ · Status checked: 2026-10

## Why it matters

Most resource names cannot be changed after creation. A naming convention agreed in week one saves years of confusion; one agreed in year two requires rebuilding things. Names tell a person what something is at a glance. Tags tell systems who owns it, who pays for it and how it should be treated. Together they make an estate searchable, billable and operable.

## Key concepts

### Names vs tags

| | Name | Tag |
|---|---|---|
| Changeable | Almost never | Any time |
| Purpose | Identify the resource | Describe and classify it |
| Length and characters | Restricted, varies by type | Up to 50 tags; name 512 chars, value 256 |
| Put here | What it is, which workload, environment, region, instance | Owner, cost centre, criticality, data classification, anything that might change |

**Rule:** if a fact might change (owner, cost centre, team name, project phase), it belongs in a tag, not the name.

### A naming pattern

The Cloud Adoption Framework recommends composing names from components:

```text
<resource type>-<workload>-<environment>-<region>-<instance>

rg-hrassist-prod-swc-001
vnet-hrassist-prod-swc-001
ca-hrassist-api-prod-swc-001
```

| Component | Guidance | Examples |
|---|---|---|
| **Resource type** | Standard abbreviation, as a prefix so resources sort and filter by type | `rg`, `vnet`, `st`, `kv` |
| **Workload / application** | Short and stable | `hrassist`, `webshop` |
| **Component** (optional) | When a workload has several of one type | `api`, `web`, `ingest` |
| **Environment** | Fixed vocabulary | `dev`, `test`, `stg`, `prod` |
| **Region** | Short code | `swc`, `weu`, `eus2`, `uks` |
| **Instance** | Zero-padded number | `001` |

Organisation or business unit can be added at the front when several units share a tenant and the workload name alone is ambiguous.

### Common abbreviations

CAF publishes a full list. The ones you will use constantly:

| Resource | Abbr. | Resource | Abbr. |
|---|---|---|---|
| Management group | `mg` | Resource group | `rg` |
| Virtual network | `vnet` | Subnet | `snet` |
| Network security group | `nsg` | Route table | `rt` |
| Public IP | `pip` | Private endpoint | `pep` |
| Application Gateway | `agw` | Azure Firewall | `afw` |
| Front Door | `afd` | NAT gateway | `ng` |
| Virtual machine | `vm` | VM scale set | `vmss` |
| App Service plan | `asp` | Web app | `app` |
| Function app | `func` | Static Web App | `stapp` |
| Container Apps environment | `cae` | Container app | `ca` |
| AKS cluster | `aks` | Container registry | `cr` |
| Storage account | `st` | Key Vault | `kv` |
| SQL server | `sql` | SQL database | `sqldb` |
| Cosmos DB account | `cosmos` | PostgreSQL server | `psql` |
| Azure Managed Redis | `amr` | Service Bus namespace | `sbns` |
| Event Hubs namespace | `evhns` | Event Grid topic | `evgt` |
| API Management | `apim` | Logic app | `logic` |
| Log Analytics workspace | `log` | Application Insights | `appi` |
| User-assigned managed identity | `id` | Azure AI Search | `srch` |
| Foundry resource | `aif` | Foundry project | `proj` |
| Azure OpenAI (classic resource kind) | `oai` | Azure Machine Learning workspace | `mlw` |

Abbreviations for newer AI resource types have been revised as products were renamed. Pick one set, write it down, and be consistent; that matters more than matching Microsoft's list exactly.

### Naming rules and scopes

Each resource type has its own constraints. The awkward ones:

| Resource | Length | Allowed | Uniqueness scope |
|---|---|---|---|
| Resource group | 1–90 | Alphanumerics, `_ - . ( )` | Subscription |
| **Storage account** | 3–24 | **Lowercase letters and numbers only** | **Global** |
| **Key Vault** | 3–24 | Alphanumerics and hyphens | **Global** |
| **Container registry** | 5–50 | Alphanumerics only | **Global** |
| Virtual machine (Windows) | 1–15 | No special characters beyond hyphen | Resource group |
| Virtual machine (Linux) | 1–64 | | Resource group |
| Web app / Function app | 2–60 | Alphanumerics and hyphens | Global (DNS) |
| Cosmos DB account | 3–44 | Lowercase, numbers, hyphens | Global |
| Azure AI Search | 2–60 | Lowercase, numbers, hyphens | Global |
| Foundry / AI services resource | 2–64 | Alphanumerics and hyphens | Resource group (custom subdomain is global) |
| API Management | 1–50 | Alphanumerics and hyphens | Global |
| Virtual network / subnet | 2–64 / 1–80 | | Resource group / VNet |

Consequences:

- **Storage accounts and container registries cannot contain hyphens.** Your pattern needs a compact variant: `sthrassistprodswc001`.
- **24 characters is short.** Keep workload names brief, or the pattern will not fit.
- **Globally unique names** collide with other customers. Add an organisation prefix or a short deterministic hash.
- **Anything with a DNS name** (web apps, storage, Foundry custom subdomains) exposes the name publicly. Do not put sensitive information in names.

In IaC, generate unique suffixes deterministically so redeployments produce the same name:

```bicep
param workload string = 'hrassist'
param env string = 'prod'
param regionCode string = 'swc'

var suffix = take(uniqueString(subscription().id, resourceGroup().id), 5)

var names = {
  storage:  toLower('st${workload}${env}${regionCode}${suffix}')        // no hyphens, ≤ 24
  keyVault: 'kv-${workload}-${env}-${regionCode}-${suffix}'             // ≤ 24
  search:   'srch-${workload}-${env}-${regionCode}-${suffix}'
  foundry:  'aif-${workload}-${env}-${regionCode}-${suffix}'
}
```

### Tooling for names

| Tool | Use |
|---|---|
| **Azure Naming Tool** | A self-hosted web app and API from the CAF team to define a convention and generate compliant names |
| **Naming modules** | Terraform and Bicep modules that produce names for every resource type from a few inputs |
| **Azure Policy** | Audit or deny names that do not match a pattern, with `like` or `match` conditions |

Example policy rule to require a resource group prefix:

```json
{
  "if": {
    "allOf": [
      { "field": "type", "equals": "Microsoft.Resources/subscriptions/resourceGroups" },
      { "field": "name", "notLike": "rg-*" }
    ]
  },
  "then": { "effect": "deny" }
}
```

Enforce names lightly. A central module that generates them is a better experience than a deny that rejects them.

### Tagging strategy

Tags fall into a few purposes. A workable core set:

| Purpose | Tag | Example values |
|---|---|---|
| **Accountability** | `owner` | A team mailbox or group, not an individual |
| | `businessUnit` | `hr`, `retail` |
| **Cost** | `costCentre` | `CC1234` |
| | `application` | `hr-assistant` |
| **Operations** | `environment` | `prod`, `dev` |
| | `criticality` | `mission-critical`, `high`, `medium`, `low` |
| | `opsTeam` | `platform-ops` |
| **Security and compliance** | `dataClassification` | `public`, `internal`, `confidential`, `restricted` |
| | `compliance` | `pci`, `gdpr`, `none` |
| **Automation** | `managedBy` | `terraform`, `bicep`, `portal` |
| | `startStopSchedule` | `weekdays-0800-1800` |
| **Lifecycle** | `expiresOn` | `2026-12-31` (sandboxes and experiments) |

For AI workloads consider also `aiUseCase` or `agentSponsor`, linking a resource to an entry in the organisation's AI inventory.

Principles:

- **Keep the mandatory set small.** Five to eight tags that are always correct beat twenty that are often empty.
- **Fix the vocabulary.** Agree casing and allowed values. `Prod`, `prod` and `production` are three different values to a cost report. Tag **names** are case-insensitive for operations; tag **values** are case-sensitive.
- **Automate.** Tags applied by hand decay. Apply them in IaC and inherit where possible.
- **No secrets or personal data.** Tags are visible to anyone with read access and appear in billing exports.

### Tag inheritance

Tags do **not** inherit from subscription or resource group to resources. Two mechanisms fill the gap:

| Mechanism | What it does | Use for |
|---|---|---|
| **Azure Policy `modify`** | Built-ins such as *Inherit a tag from the resource group* or *…from the subscription* copy the tag onto resources, and remediation fixes existing ones | When the tag must be on the resource itself (automation, Resource Graph queries) |
| **Cost Management tag inheritance** | Applies subscription and resource group tags to **usage records** in cost data, without touching resources | Cost reporting and chargeback |

Enable Cost Management tag inheritance in almost every case: it also covers usage for resources that do not support tags.

### Enforcing tags

| Policy | Effect | Use |
|---|---|---|
| *Require a tag on resource groups* | Deny | Force teams to supply `owner` and `costCentre` at the resource group |
| *Inherit a tag from the resource group if missing* | Modify | Propagate to resources |
| *Add a tag to resources* | Modify | Apply a default |
| *Require a tag and its value* | Deny | Fixed values such as `environment=prod` at a subscription |
| Custom allowed-values policy | Deny | Constrain vocabulary |

A common pattern: **require** a few tags on resource groups (deny), then **inherit** them to resources (modify). Teams tag once; everything else follows.

Not every resource type supports tags, and tags on some resources are not emitted in usage data. Check before designing a report around them.

### Resource organisation

Beyond names and tags, organise by lifecycle.

**Resource group patterns**

| Pattern | Example |
|---|---|
| One per workload per environment | `rg-hrassist-prod-swc-001` |
| Split by lifecycle within a workload | `rg-hrassist-network-prod-…`, `rg-hrassist-data-prod-…`, `rg-hrassist-app-prod-…` |
| Shared services | `rg-network-hub-swc-001`, `rg-monitor-prod-swc-001` |

Split when parts of a workload have different owners, different deployment cadence, or when stateful resources (databases, storage) should be protected from the frequent redeployment of stateless ones.

**For an AI workload**, a reasonable split is:

```text
rg-hrassist-network-prod-swc-001   VNet, NSGs, private endpoints
rg-hrassist-data-prod-swc-001      Storage, Cosmos DB, AI Search      (stateful, locked)
rg-hrassist-ai-prod-swc-001        Foundry resource and project, model deployments
rg-hrassist-app-prod-swc-001       Container Apps, registry, managed identities, Key Vault
rg-hrassist-monitor-prod-swc-001   Log Analytics, Application Insights
```

Simpler workloads can use a single resource group. Do not split for its own sake: every boundary adds cross-group references.

**Locks:** `CanNotDelete` on resource groups holding stateful data.

**Region:** create the resource group in the same region as its resources.

## How it works on Azure

### Writing the standard

A naming and tagging standard should fit on two pages:

1. The pattern, with each component and its allowed values.
2. The abbreviation table (link to CAF and list your deviations).
3. Variants for constrained resource types (storage, registry, Windows VMs).
4. How uniqueness is achieved for global names.
5. The mandatory tags, optional tags, and allowed values.
6. How it is enforced and how exceptions are requested.
7. Ten worked examples.

Publish it with the IaC modules that implement it. A standard nobody can easily follow is ignored.

### Brownfield

You cannot rename existing resources. For an inherited estate:

- Apply the standard to **new** resources from an agreed date.
- **Tag** existing resources to recover ownership and cost data; tags can be added any time.
- Rename only when a resource is rebuilt for another reason.
- Use Resource Graph to measure adoption (file 07).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Prefix vs suffix for resource type | Prefix: sorts by type and matches CAF. Either works if consistent |
| Region in the name | Include it for regional resources; it prevents confusion in multi-region designs |
| Environment in the name | Yes, even when subscriptions separate environments; names appear in logs and alerts out of context |
| Instance numbers | Useful for scale-out and blue-green; do not let them imply ordering or importance |
| Long descriptive vs short names | Short. The 24-character resources set the ceiling |
| Random vs deterministic suffix | Deterministic (`uniqueString`), so redeployments are idempotent |
| Tags on resources vs resource groups | Require at resource group, inherit to resources |
| Strict enforcement vs audit | Deny for two or three critical tags; audit the rest |

## Hands-on

```bash
RG=rg-naming-lab-swc-001; LOC=swedencentral

# 1. Create a tagged resource group
az group create -n $RG -l $LOC \
  --tags owner=platform-team@example.com costCentre=CC1234 environment=dev application=naming-lab

# 2. Check name availability for globally unique types
az storage account check-name --name stnaminglabdevswc001
az keyvault check-name --name kv-naminglab-dev-swc
az cognitiveservices account list-kinds | head -5

# 3. Deploy with generated names
cat > names.bicep <<'EOF'
param workload string = 'naminglab'
param env string = 'dev'
param regionCode string = 'swc'
param location string = resourceGroup().location
var suffix = take(uniqueString(subscription().id, resourceGroup().id), 5)

resource st 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: toLower('st${workload}${env}${regionCode}${suffix}')
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}
output storageName string = st.name
output nameLength int = length(st.name)
EOF
az deployment group create -g $RG --template-file names.bicep --query properties.outputs

# 4. Confirm tags did not inherit, then update tags without replacing existing ones
az resource list -g $RG --query "[].{Name:name, Tags:tags}" -o json
ST_ID=$(az resource list -g $RG --query "[0].id" -o tsv)
az tag update --resource-id $ST_ID --operation Merge --tags dataClassification=internal

# 5. Find untagged resources across the subscription
az graph query -q "Resources | where isnull(tags.owner) | summarize count() by type | order by count_ desc" -o table

# 6. Clean up
az group delete -n $RG --yes --no-wait
```

Then write your own two-page standard and name every resource in the reference stack from topic 01, file 09.

In the portal, open **Cost Management → Settings → Manage subscription** (or the billing scope's configuration) and locate **Tag inheritance**.

## Security, cost and reliability notes

- **Security:** names and tags are metadata readable by anyone with Reader. Public DNS names reveal workload names. Avoid project code names that are sensitive.
- **Security:** `dataClassification` tags can drive policy (for example requiring customer-managed keys or private endpoints for `confidential`).
- **Cost:** untagged or wrongly tagged resources land in "unallocated" on every cost report. Tag coverage is a FinOps key metric.
- **Reliability:** `criticality` tags let operations prioritise alerts, backup policy and recovery order.

## Common mistakes

- Putting team names, owners or project phases in resource names.
- A pattern that produces 30-character names and breaks on storage accounts.
- Forgetting that storage accounts and registries cannot have hyphens.
- Random name suffixes that change on every deployment.
- Twenty mandatory tags, mostly blank or `tbd`.
- Inconsistent values (`Prod`, `production`, `PRD`).
- Assuming tags inherit.
- An individual's name as `owner`.
- Secrets or personal data in tags.
- Trying to rename an existing estate instead of tagging it.

## Check yourself

1. What kind of information belongs in a tag and not in a name?
2. Give a compliant name for a production storage account for workload `webshop` in West Europe.
3. Which common resource types require globally unique names?
4. Do tags inherit from a resource group? How do you achieve the effect?
5. What is the difference between policy-based tag inheritance and Cost Management tag inheritance?
6. Why use a deterministic suffix in IaC?
7. When would you split a workload across several resource groups?

<details>
<summary>Answers</summary>

1. Anything that may change over the resource's life: owner, cost centre, criticality, team, project phase.
2. For example `stwebshopprodweu001` (3–24 characters, lowercase letters and numbers only, globally unique, so often with an added prefix or hash).
3. Storage accounts, Key Vaults, container registries, web and function apps, Cosmos DB accounts, AI Search services, API Management services.
4. No. Use an Azure Policy modify effect to copy tags to resources, and Cost Management tag inheritance for cost data.
5. Policy inheritance writes the tag onto the resource itself. Cost Management inheritance applies resource group and subscription tags to usage records only.
6. So redeploying produces the same name and the deployment stays idempotent.
7. When parts have different owners, lifecycles or deployment cadences, or to protect stateful resources from frequent redeployments.

</details>

## References

- [Define your naming convention](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)
- [Abbreviation recommendations for Azure resources](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations)
- [Naming rules and restrictions for Azure resources](https://learn.microsoft.com/azure/azure-resource-manager/management/resource-name-rules)
- [Define your tagging strategy](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-tagging)
- [Use tags to organize your Azure resources](https://learn.microsoft.com/azure/azure-resource-manager/management/tag-resources)
- [Assign policy definitions for tag compliance](https://learn.microsoft.com/azure/azure-resource-manager/management/tag-policies)
- [Group and allocate costs using tag inheritance](https://learn.microsoft.com/azure/cost-management-billing/costs/enable-tag-inheritance)
- [Azure Naming Tool](https://github.com/mspnp/AzureNamingTool)
