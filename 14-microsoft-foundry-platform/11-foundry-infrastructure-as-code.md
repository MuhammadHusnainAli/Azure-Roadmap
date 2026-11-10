# Foundry Infrastructure as Code

## Why it matters

Clicking through the portal is fine for a demo, but production AI platforms need repeatable environments (dev, test, prod), reviewed changes, policy compliance and fast disaster recovery. That means describing the Foundry resource, projects, model deployments, connections, networking and role assignments as code with **Bicep** or **Terraform**, deployed through pipelines with workload identity federation. Foundry's resource model is relatively new, so knowing the right resource types and the gaps between providers saves days of trial and error.

## Key concepts

### Resource types to model

| Resource | Type | Notes |
|---|---|---|
| Foundry resource | `Microsoft.CognitiveServices/accounts` (kind `AIServices`) | `allowProjectManagement: true` (immutable after creation), `customSubDomainName`, identity, `disableLocalAuth`, `publicNetworkAccess`, `networkAcls`, encryption |
| Project | `Microsoft.CognitiveServices/accounts/projects` | Identity (system or user-assigned), display name, description |
| Model deployment | `Microsoft.CognitiveServices/accounts/deployments` | `sku.name` (deployment type, for example `GlobalStandard`), `sku.capacity`, `model.format/name/version`, `versionUpgradeOption`, RAI policy |
| Resource connection | `Microsoft.CognitiveServices/accounts/connections` | Shared with all projects |
| Project connection | `Microsoft.CognitiveServices/accounts/projects/connections` | Project-only |
| Capability hosts (agent standard setup) | `accounts/capabilityHosts`, `accounts/projects/capabilityHosts` | Point Agent Service at your Storage, Cosmos DB and Search connections |
| Content filter / RAI policy | `Microsoft.CognitiveServices/accounts/raiPolicies` | Custom content filtering configurations referenced by deployments |
| Private endpoint, DNS zones | `Microsoft.Network/*` | See file 04 |
| Role assignments | `Microsoft.Authorization/roleAssignments` | Use role definition IDs (the Foundry roles were renamed) |

Use the latest stable API version shown in the template reference. `2025-06-01` is a common baseline for accounts, projects, deployments and connections; newer previews add features.

### Bicep vs Terraform

| | Bicep | Terraform |
|---|---|---|
| Coverage | Day-zero support for all ARM resource types and API versions | **AzAPI** provider gives full coverage (any ARM type and API version, including preview). **AzureRM** covers core management capabilities only, with some Foundry features lagging |
| Recommended for Foundry | Native choice for Azure-only teams | `azapi_resource` for Foundry resources, projects, connections and capability hosts; `azurerm` for surrounding infrastructure (VNets, Key Vault, Storage) |
| Modules | **Azure Verified Modules (AVM)** for Cognitive Services accounts and AI patterns | AVM Terraform modules and AI landing zone patterns |
| State | ARM deployment history; **deployment stacks** for lifecycle | Terraform state (secure remote backend) |

Note that `azurerm_ai_foundry` and `azurerm_ai_foundry_project` model the **hub-based** (classic) resources, not the new Foundry resource. Check which resource your module creates.

### Microsoft templates

Microsoft publishes Bicep and Terraform samples for common setups, including the basic Foundry resource with a project, the **standard agent setup** (BYO Storage, Cosmos DB, AI Search) and **network-isolated** variants, plus `azd` starter templates (file 07). Start from these rather than from scratch.

## How it works on Azure

### Bicep: resource, project, deployment, connection, role

```bicep
param location string = resourceGroup().location
param name string = 'fdy-contoso-dev'
param devGroupObjectId string

resource foundry 'Microsoft.CognitiveServices/accounts@2025-06-01' = {
  name: name
  location: location
  kind: 'AIServices'
  sku: { name: 'S0' }
  identity: { type: 'SystemAssigned' }
  properties: {
    allowProjectManagement: true          // required for projects; can't change later
    customSubDomainName: name             // required for Entra ID auth
    disableLocalAuth: true                // keyless only
    publicNetworkAccess: 'Enabled'        // 'Disabled' + private endpoint in prod
  }
}

resource project 'Microsoft.CognitiveServices/accounts/projects@2025-06-01' = {
  parent: foundry
  name: 'claims-assistant'
  location: location
  identity: { type: 'SystemAssigned' }
  properties: {
    displayName: 'Claims assistant'
    description: 'Claims handling agents'
  }
}

resource chat 'Microsoft.CognitiveServices/accounts/deployments@2025-06-01' = {
  parent: foundry
  name: 'gpt-5-mini'
  sku: { name: 'GlobalStandard', capacity: 50 }      // capacity in thousands of TPM for standard types
  properties: {
    model: { format: 'OpenAI', name: 'gpt-5-mini', version: '2025-08-07' }
    versionUpgradeOption: 'OnceNewDefaultVersionAvailable'
  }
}

resource appInsightsConn 'Microsoft.CognitiveServices/accounts/connections@2025-06-01' = {
  parent: foundry
  name: 'appinsights'
  properties: {
    category: 'AppInsights'
    target: appInsights.id
    authType: 'ApiKey'
    isSharedToAll: true
    credentials: { key: appInsights.properties.ConnectionString }
    metadata: { ApiType: 'Azure', ResourceId: appInsights.id }
  }
}

// Foundry User (role ID) for the dev group on the project
resource devRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: project
  name: guid(project.id, devGroupObjectId, '53ca6127-db72-4b80-b1b0-d745d6d5456d')
  properties: {
    principalId: devGroupObjectId
    principalType: 'Group'
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '53ca6127-db72-4b80-b1b0-d745d6d5456d')
  }
}

// Foundry User for the project's managed identity on the resource
resource projectMiRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: foundry
  name: guid(foundry.id, project.id, '53ca6127-db72-4b80-b1b0-d745d6d5456d')
  properties: {
    principalId: project.identity.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '53ca6127-db72-4b80-b1b0-d745d6d5456d')
  }
}

output projectEndpoint string = 'https://${name}.services.ai.azure.com/api/projects/${project.name}'
```

Model versions and connection property shapes change; confirm the model version with `az cognitiveservices model list` and the connection schema in the template reference. Multiple deployments on one resource may need `dependsOn` chaining (deploy sequentially) to avoid conflicts.

### Terraform with AzAPI

```hcl
resource "azapi_resource" "foundry" {
  type      = "Microsoft.CognitiveServices/accounts@2025-06-01"
  name      = "fdy-contoso-dev"
  parent_id = azurerm_resource_group.rg.id
  location  = azurerm_resource_group.rg.location
  identity { type = "SystemAssigned" }
  body = {
    kind = "AIServices"
    sku  = { name = "S0" }
    properties = {
      allowProjectManagement = true
      customSubDomainName    = "fdy-contoso-dev"
      disableLocalAuth       = true
    }
  }
}

resource "azapi_resource" "project" {
  type      = "Microsoft.CognitiveServices/accounts/projects@2025-06-01"
  name      = "claims-assistant"
  parent_id = azapi_resource.foundry.id
  location  = azurerm_resource_group.rg.location
  identity { type = "SystemAssigned" }
  body = { properties = { displayName = "Claims assistant", description = "Claims handling agents" } }
}

resource "azapi_resource" "chat" {
  type      = "Microsoft.CognitiveServices/accounts/deployments@2025-06-01"
  name      = "gpt-5-mini"
  parent_id = azapi_resource.foundry.id
  body = {
    sku        = { name = "GlobalStandard", capacity = 50 }
    properties = { model = { format = "OpenAI", name = "gpt-5-mini", version = "2025-08-07" } }
  }
}
```

### Pipeline (GitHub Actions, OIDC)

```yaml
permissions: { id-token: write, contents: read }
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - run: az deployment group what-if -g rg-foundry-dev -f infra/main.bicep -p infra/dev.bicepparam
      - run: az deployment group create -g rg-foundry-dev -f infra/main.bicep -p infra/dev.bicepparam
```

The pipeline identity needs rights to create resources and role assignments. Use **Foundry Account Owner** + **Role Based Access Control Administrator** (with conditions) on the resource group, or Owner scoped narrowly through PIM, rather than subscription Owner.

### What else to codify

| Item | How |
|---|---|
| Agents and agent versions | Agent definitions via SDK or `azd ai agent` in the app pipeline (not ARM) |
| Evaluations | Evaluation datasets and thresholds in the repo, run by CI |
| Guardrail policies and content filters | `raiPolicies` resources and Control Plane policies |
| Azure Policy assignments | In the landing zone IaC (disable local auth, private networking, allowed models) |
| Monitoring | Diagnostic settings, Application Insights, alerts |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Bicep vs Terraform | Bicep for Azure-only teams and day-zero features; Terraform (AzAPI for Foundry) for multicloud standardisation |
| AzAPI vs AzureRM for Foundry | AzAPI for the Foundry resource, projects, connections and capability hosts; AzureRM where resources are mature |
| Where model deployments live | Platform IaC for shared deployments; app IaC for app-dedicated deployments. Avoid two pipelines fighting over the same deployment |
| Immutable settings | Set `allowProjectManagement`, custom subdomain and network design right the first time |
| Secrets | No keys in templates; use Entra ID connections; Key Vault references if keys are unavoidable |
| Drift | Run what-if/plan in CI; restrict portal write access in production |

## Hands-on

1. Deploy the Bicep sample to a dev resource group with `what-if` first; confirm the project endpoint output works with the Python SDK.
2. Re-deploy with a changed `versionUpgradeOption` and observe the idempotent update.
3. Convert the sample to Terraform with AzAPI and deploy to a second resource group.
4. Deploy Microsoft's standard agent setup (network-isolated) template and diff its resources against your sample.
5. Add a GitHub Actions workflow with OIDC that runs `what-if` on pull requests and deploys on merge.

## Security, cost and reliability notes

- **Security:** pipelines that create role assignments are highly privileged. Constrain them with RBAC conditions and environment approvals.
- **Security:** set `disableLocalAuth: true` and network restrictions in templates, and enforce them with Azure Policy.
- **Cost:** deployments with large capacity (or PTU) are created by code, so review capacity changes in pull requests.
- **Reliability:** deleting a Foundry resource soft-deletes it. Plan for purge in teardown pipelines, and for re-creation with the same name in disaster recovery.
- **Reliability:** keep a second-region version of the template ready (topic 28).

## Common mistakes

- Using `azurerm_ai_foundry` expecting the new Foundry resource (it creates a hub).
- Forgetting `customSubDomainName`, so Entra ID authentication fails.
- Missing role assignments for the project managed identity, so agents can't reach models or tools.
- Role names instead of role IDs during the rename rollout.
- Parallel deployment creation on the same resource causing conflicts.

## Check yourself

1. Which ARM resource types represent a Foundry resource, project and deployment?
2. Which Terraform provider gives full Foundry coverage, and why?
3. Which property must be true to create projects, and can it be changed later?
4. Why use role definition IDs instead of names?
5. What does `azurerm_ai_foundry` create?
6. What should a CI pipeline run before deploying Foundry IaC?

<details>
<summary>Answers</summary>

1. `Microsoft.CognitiveServices/accounts` (kind `AIServices`), `Microsoft.CognitiveServices/accounts/projects` and `Microsoft.CognitiveServices/accounts/deployments`.
2. AzAPI, because it can use any ARM resource type and API version (including preview), while AzureRM covers only core capabilities.
3. `allowProjectManagement`; it can't be changed after creation.
4. The Foundry roles were renamed; IDs are stable, so automation doesn't break.
5. A hub-based (classic) Foundry hub, not the new Foundry resource.
6. Validation and a what-if (Bicep) or plan (Terraform), plus policy and security checks, before applying.

</details>

## References

- [Create a Foundry resource with Bicep](https://learn.microsoft.com/azure/foundry/how-to/create-resource-template)
- [Create a Foundry resource with Terraform](https://learn.microsoft.com/azure/foundry/how-to/create-resource-terraform)
- [Microsoft.CognitiveServices/accounts template reference](https://learn.microsoft.com/azure/templates/microsoft.cognitiveservices/accounts)
- [Microsoft.CognitiveServices/accounts/projects template reference](https://learn.microsoft.com/azure/templates/microsoft.cognitiveservices/accounts/projects)
- [Microsoft.CognitiveServices/accounts/deployments template reference](https://learn.microsoft.com/azure/templates/microsoft.cognitiveservices/accounts/deployments)
- [Standard agent setup with your own resources](https://learn.microsoft.com/azure/foundry/agents/how-to/use-your-own-resources)
- [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/)
