# Azure Verified Modules

## Why it matters

Writing a production-grade storage account is not one resource. It is the account plus private endpoints, diagnostic settings, role assignments, a lock, customer-managed keys and tags, all wired correctly. Every team used to write its own version. Azure Verified Modules (AVM) is Microsoft's single, supported library of such modules for both Bicep and Terraform. Using it means starting from a secure, consistent baseline, and it is what the current landing zone and AI accelerators are built from.

## Key concepts

### What AVM is

An initiative with one set of **specifications** that every module must meet, and a growing catalogue of modules owned and maintained by Microsoft employees.

| Property | Meaning |
|---|---|
| **One standard, two languages** | The same interface conventions in Bicep and Terraform |
| **Well-Architected aligned** | Defaults favour reliability and security, such as zone redundancy and TLS 1.2 |
| **Consistent interfaces** | Common features are exposed the same way in every module |
| **Tested** | Automated deployment tests, linting and static analysis before release |
| **Supported** | Issues are raised on GitHub; modules have named Microsoft owners. Support escalation paths exist for confirmed bugs |
| **Versioned** | Semantic versioning, published to the public registries |

AVM replaced earlier community efforts: CARML for Bicep and Terraform Verified Modules.

### Module classifications

| Class | What it deploys | Examples |
|---|---|---|
| **Resource modules** (`res`) | One primary resource and its tightly coupled children and extensions | Storage account, Key Vault, virtual network, Container App, AI Search service, Cognitive Services account |
| **Pattern modules** (`ptn`) | A multi-resource solution or architecture, composed from resource modules | Hub networking, subscription vending, platform landing zone components, AI landing zone / Foundry baseline, Container Apps landing zone accelerator |
| **Utility modules** (`utl`) | Helper functions and types, deploying nothing | Common type definitions, region and naming helpers |

Rule of thumb: build your workload from **resource** modules; adopt a **pattern** module when it matches your architecture closely and you are happy to follow its opinions.

### Where modules live

| | Bicep | Terraform |
|---|---|---|
| Registry | Bicep public registry (Microsoft Container Registry) | Terraform Registry, `Azure` namespace |
| Reference | `br/public:avm/res/<provider>/<resource>:<version>` | `Azure/avm-res-<provider>-<resource>/azurerm` |
| Pattern | `br/public:avm/ptn/<group>/<name>:<version>` | `Azure/avm-ptn-<name>/azurerm` |
| Source | `Azure/bicep-registry-modules` on GitHub | One GitHub repository per module |
| Index | `aka.ms/avm` (module indexes for both languages) | Same |

The VS Code Bicep extension autocompletes `br/public:avm/...` paths and versions.

### Standard interfaces

Every resource module that supports a capability exposes it the same way. Learn these once and they apply across the catalogue.

| Interface | Purpose |
|---|---|
| `diagnosticSettings` / `diagnostic_settings` | Send logs and metrics to Log Analytics, Storage or Event Hubs |
| `roleAssignments` / `role_assignments` | Azure RBAC on the resource |
| `lock` | `CanNotDelete` or `ReadOnly` |
| `privateEndpoints` / `private_endpoints` | Private endpoints with DNS zone group integration |
| `managedIdentities` / `managed_identities` | System- and user-assigned identities |
| `customerManagedKey` / `customer_managed_key` | Encryption with your key |
| `tags` | Tags |
| `enableTelemetry` / `enable_telemetry` | AVM usage telemetry (see below) |

### Using a module in Bicep

```bicep
param location string = resourceGroup().location
param appPrincipalId string
param logAnalyticsId string
param peSubnetId string
param blobDnsZoneId string

module storage 'br/public:avm/res/storage/storage-account:0.20.0' = {
  params: {
    name: 'sthrassistprodswc001'
    location: location
    skuName: 'Standard_ZRS'
    allowSharedKeyAccess: false
    allowBlobPublicAccess: false
    publicNetworkAccess: 'Disabled'
    blobServices: {
      containers: [ { name: 'docs' }, { name: 'processed' } ]
    }
    privateEndpoints: [
      {
        service: 'blob'
        subnetResourceId: peSubnetId
        privateDnsZoneGroup: { privateDnsZoneGroupConfigs: [ { privateDnsZoneResourceId: blobDnsZoneId } ] }
      }
    ]
    roleAssignments: [
      { principalId: appPrincipalId, principalType: 'ServicePrincipal', roleDefinitionIdOrName: 'Storage Blob Data Reader' }
    ]
    diagnosticSettings: [ { workspaceResourceId: logAnalyticsId } ]
    lock: { kind: 'CanNotDelete' }
    tags: { application: 'hr-assistant', environment: 'prod' }
  }
}

output storageId string = storage.outputs.resourceId
```

Roughly forty lines replace several hundred of hand-written, correctly ordered resources. Version numbers and parameter names here are illustrative: check the module's README for the release you pin.

### Using a module in Terraform

```hcl
module "storage" {
  source  = "Azure/avm-res-storage-storageaccount/azurerm"
  version = "~> 0.6"

  name                          = "sthrassistprodswc001"
  resource_group_name           = azurerm_resource_group.this.name
  location                      = var.location
  account_replication_type      = "ZRS"
  shared_access_key_enabled     = false
  public_network_access_enabled = false

  containers = {
    docs      = { name = "docs" }
    processed = { name = "processed" }
  }

  private_endpoints = {
    blob = {
      subnet_resource_id            = var.pe_subnet_id
      subresource_name              = "blob"
      private_dns_zone_resource_ids = [var.blob_dns_zone_id]
    }
  }

  role_assignments = {
    app_reader = {
      role_definition_id_or_name = "Storage Blob Data Reader"
      principal_id               = var.app_principal_id
    }
  }

  diagnostic_settings = { to_law = { workspace_resource_id = var.log_analytics_id } }
  lock                = { kind = "CanNotDelete" }
  tags                = local.tags
}
```

Terraform AVM modules take maps keyed by a name you choose, so items are stable under `for_each`.

### Versioning

- Modules follow **semantic versioning**. Many are still `0.x`, where **minor versions can contain breaking changes**.
- **Pin an exact version** in Bicep; use a pessimistic constraint (`~> 0.6.0`) and a committed lock file in Terraform.
- Read the changelog before upgrading. Automate upgrade pull requests with Dependabot or Renovate so each bump is reviewed and planned.
- Provider major versions matter: check which `azurerm` and `azapi` versions a Terraform module supports before moving to a new provider major.

### Telemetry

By default each module deploys a small, empty deployment (Bicep) or sends a provider-level signal (Terraform) so Microsoft can measure module usage. It contains no customer data. Set `enableTelemetry: false` / `enable_telemetry = false` to turn it off, which some organisations do as policy.

### Pattern modules worth knowing

| Pattern | Use |
|---|---|
| **Platform landing zone modules** | The current implementation of Azure landing zones for both languages (management groups and policy, connectivity, management resources) |
| **Subscription vending** (`lz/sub-vending`) | Create and configure workload subscriptions |
| **Hub networking / Virtual WAN** | Connectivity foundations |
| **AI landing zone / Foundry patterns** | A network-isolated Foundry baseline with dependencies |
| **Container Apps, AKS, App Service landing zone accelerators** | Workload landing zones for application platforms |
| **Private DNS zones for Private Link** | The standard set of zones and links |

Catalogue contents change monthly. Browse the index for what exists now.

## How it works on Azure

### Composition model

```text
Your workload repository
└── main.bicep / main.tf                    orchestration, your naming, your wiring
    ├── your organisation's wrapper modules  (optional: enforce company defaults)
    │    └── AVM resource modules            (storage, key vault, search, container app, …)
    └── AVM pattern module                   (optional: e.g. a hub network)
```

**Wrapper modules.** Many organisations publish a thin internal module around an AVM module that fixes non-negotiable settings (no public access, mandatory diagnostics workspace, required tags) and exposes only what teams may choose. Teams consume the wrapper from a private registry. This gives central control with little code to maintain.

### AVM and policy

AVM gives secure defaults; it does not enforce them, since a caller can override most parameters. Azure Policy remains the control. The two fit together: modules make the compliant path the easy one, and policy catches whatever does not use them.

### When no module exists

- For a resource type without a module, write the resource directly in Bicep or with `azurerm`/`azapi`, following the same interface conventions so it can be swapped later.
- For a property a module does not expose yet, check for a newer version, then consider a small patch resource (`azapi_update_resource`, or an additional Bicep resource referencing the module's output) before forking.
- Raise an issue or contribute. Fast-moving AI resource types are the likeliest place to hit gaps.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| AVM or hand-written | AVM for standard resources. Hand-write when the module adds complexity without value for a trivial resource, or does not support what you need |
| Resource or pattern modules | Resource modules for flexibility. Pattern modules when their architecture is yours |
| Direct use or wrappers | Wrappers once several teams consume the same modules and you want company defaults |
| Fork or contribute | Avoid forks; they cut you off from updates. Contribute upstream or wrap |
| Version strategy | Exact pins, automated upgrade pull requests, upgrade regularly so jumps stay small |
| Telemetry | Off if organisational policy requires |

**Costs of using AVM.** Modules expose many parameters, so there is a learning curve. Each module is a nested deployment (Bicep), which adds some deployment time and makes what-if output deeper. `0.x` versions can break on upgrade. Compiled templates are larger. For most teams the consistency and built-in good practice outweigh these.

## Hands-on

1. Open the AVM module index (`aka.ms/avm`). Find the storage account, Key Vault, Azure AI Search and Cognitive Services account modules for your language. Note the latest version of each and whether it is `0.x`.
2. Read the storage account module's README: parameters, the standard interfaces, and the usage examples (*defaults*, *max*, *WAF-aligned*).
3. Deploy with a module (Bicep shown; use the Terraform example above if that is your language):

   ```bash
   RG=rg-avm-lab; az group create -n $RG -l swedencentral -o none
   LAW=$(az monitor log-analytics workspace create -g $RG -n log-avm-lab --query id -o tsv)
   ME=$(az ad signed-in-user show --query id -o tsv)

   cat > main.bicep <<'EOF'
   param location string = resourceGroup().location
   param logAnalyticsId string
   param userObjectId string

   module storage 'br/public:avm/res/storage/storage-account:0.20.0' = {
     params: {
       name: 'stavmlab${take(uniqueString(resourceGroup().id), 8)}'
       location: location
       skuName: 'Standard_LRS'
       allowSharedKeyAccess: false
       allowBlobPublicAccess: false
       blobServices: { containers: [ { name: 'docs' } ] }
       roleAssignments: [
         { principalId: userObjectId, principalType: 'User', roleDefinitionIdOrName: 'Storage Blob Data Contributor' }
       ]
       diagnosticSettings: [ { workspaceResourceId: logAnalyticsId } ]
       lock: { kind: 'CanNotDelete' }
     }
   }
   output name string = storage.outputs.name
   EOF

   az deployment group what-if -g $RG -f main.bicep -p logAnalyticsId=$LAW userObjectId=$ME
   az deployment group create  -g $RG -f main.bicep -p logAnalyticsId=$LAW userObjectId=$ME
   ```

   If the version does not exist, use the latest shown by IntelliSense or the index, and adjust parameter names to that version's README.
4. Inspect the result: the lock, the diagnostic setting, the role assignment, and the nested deployments under *Deployments*.
5. Count how many resources were created from how many lines.
6. Remove the lock (`az lock delete`) and delete the resource group.

Exercise: write a ten-line wrapper module that calls the storage module with public access and shared keys fixed to off and exposes only `name`, `containers` and `roleAssignments`.

## Security, cost and reliability notes

- **Security:** a module is third-party code running with your deployment identity's permissions. Pin versions, review upgrades, and consider mirroring modules into a private registry for supply-chain control and for environments without public internet access.
- **Security:** "WAF-aligned" defaults are a good baseline, not a guarantee. Check that public access and local authentication are actually disabled in your call.
- **Cost:** defaults may select zone-redundant or premium SKUs. Review before deploying to non-production.
- **Reliability:** a `0.x` minor upgrade can rename or restructure parameters. Run what-if or plan and read the changelog on every bump.

## Common mistakes

- Referencing a module without a version, or with a floating one.
- Assuming a module enforces security when the caller has overridden the defaults.
- Forking a module for a small change and never merging upstream updates.
- Using a pattern module whose architecture differs from yours and fighting its assumptions.
- Upgrading several minor versions at once without reading the changelog.
- Copying parameter names from an old blog post instead of the pinned version's README.
- Rebuilding private endpoint and diagnostic plumbing by hand next to a module that already offers it.

## Check yourself

1. What are the three AVM module classifications?
2. Name four standard interfaces offered by resource modules.
3. How do you reference an AVM resource module in Bicep and in Terraform?
4. Why is exact version pinning especially important for `0.x` modules?
5. What is a wrapper module and why would an organisation write one?
6. Does using AVM remove the need for Azure Policy? Explain.
7. What should you do when a module does not expose a property you need?

<details>
<summary>Answers</summary>

1. Resource, pattern and utility modules.
2. Any four of: diagnostic settings, role assignments, lock, private endpoints, managed identities, customer-managed key, tags.
3. Bicep: `br/public:avm/res/<provider>/<resource>:<version>`. Terraform: `source = "Azure/avm-res-<provider>-<resource>/azurerm"` with a `version`.
4. Under semantic versioning, `0.x` minor releases may include breaking changes.
5. A thin internal module around an AVM module that fixes organisational defaults and exposes a reduced interface, giving central control with little maintenance.
6. No. Modules provide secure defaults that callers can override or bypass; policy is what enforces requirements.
7. Check for a newer version, then patch with a small additional resource or `azapi_update_resource`, and raise or contribute the change upstream, avoiding a fork.

</details>

## References

- [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/)
- [AVM module indexes](https://azure.github.io/Azure-Verified-Modules/indexes/)
- [AVM specifications](https://azure.github.io/Azure-Verified-Modules/specs/module-specs/)
- [Bicep registry modules repository](https://github.com/Azure/bicep-registry-modules)
- [Azure Verified Modules on the Terraform Registry](https://registry.terraform.io/namespaces/Azure)
- [Azure landing zones accelerators built on AVM](https://azure.github.io/Azure-Landing-Zones/)
- [Bicep modules: public module registry](https://learn.microsoft.com/azure/azure-resource-manager/bicep/modules#public-module-registry)
