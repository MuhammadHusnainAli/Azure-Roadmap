# Bicep

## Why it matters

Bicep is Microsoft's own language for declaring Azure infrastructure. It supports every resource type and API version on the day they ship, needs no state file, and is free. If your team works only on Azure, it is the most direct route to repeatable environments. Most Microsoft reference implementations, including the landing zone and AI accelerators, are published in it.

## Key concepts

### What Bicep is

A declarative, domain-specific language that **compiles to ARM JSON**. The Bicep CLI (bundled with the Azure CLI as `az bicep`) does the compilation, and Azure Resource Manager does the deployment.

| Property | Consequence |
|---|---|
| No state file | Azure itself is the source of truth. Nothing to store, lock or corrupt |
| Day-zero support | Any resource type and API version, including previews |
| Azure only | Other providers reachable only through extensions |
| Idempotent | Deploying twice yields the same result |

Tooling: the **VS Code Bicep extension** (IntelliSense, validation, a visualiser, paste-JSON-as-Bicep), `az bicep build | lint | format | decompile | upgrade`.

### File building blocks

```bicep
targetScope = 'resourceGroup'            // default; also subscription, managementGroup, tenant

@description('Short workload name used in resource names')
@minLength(2)
@maxLength(10)
param workload string

@allowed(['dev', 'test', 'prod'])
param env string = 'dev'

param location string = resourceGroup().location

@secure()
param adminPassword string = ''

var suffix = take(uniqueString(resourceGroup().id), 5)
var tags = { application: workload, environment: env, managedBy: 'bicep' }

resource st 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: toLower('st${workload}${env}${suffix}')
  location: location
  tags: tags
  sku: { name: env == 'prod' ? 'Standard_ZRS' : 'Standard_LRS' }
  kind: 'StorageV2'
  properties: {
    allowBlobPublicAccess: false
    allowSharedKeyAccess: false
    minimumTlsVersion: 'TLS1_2'
  }
}

output storageName string = st.name
output blobEndpoint string = st.properties.primaryEndpoints.blob
```

| Element | Purpose |
|---|---|
| `targetScope` | Where the file deploys |
| `param` | Inputs, with decorators for validation and documentation |
| `var` | Named expressions |
| `resource` | A resource declaration: `symbolicName 'type@apiVersion'` |
| `module` | Deploy another Bicep file |
| `output` | Values returned to the caller |
| `type`, `func` | User-defined types and functions |
| `import`, `extension` | Reuse and provider extensions |

Decorators: `@description`, `@allowed`, `@minLength`/`@maxLength`, `@minValue`/`@maxValue`, `@secure`, `@metadata`, `@export`, `@batchSize`, `@sealed`, `@discriminator`.

### Referencing resources

**Symbolic references create dependencies automatically.** You almost never write `dependsOn`.

```bicep
resource blob 'Microsoft.Storage/storageAccounts/blobServices@2023-05-01' = {
  parent: st                       // child resource via parent
  name: 'default'
}

resource docs 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-05-01' = {
  parent: blob
  name: 'docs'
}
```

**Existing resources** are referenced, not deployed:

```bicep
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' existing = {
  name: 'kv-shared-prod'
  scope: resourceGroup('rg-shared-prod')      // can be another RG or subscription
}
output vaultUri string = kv.properties.vaultUri
```

**Extension resources** (role assignments, locks, diagnostic settings) attach with `scope`:

```bicep
resource blobReader 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(st.id, principalId, 'blob-reader')        // deterministic GUID
  scope: st
  properties: {
    principalId: principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '2a2b9908-6ea1-4ae2-8e65-a410df84e7d1')
  }
}
```

Use `guid()` with stable inputs for role assignment names so redeployment is idempotent, and always set `principalType` to avoid replication-delay failures for new identities.

### Loops and conditions

```bicep
param containers array = ['raw', 'processed', 'archive']
param deployLock bool = true

resource c 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-05-01' = [for name in containers: {
  parent: blob
  name: name
}]

resource lock 'Microsoft.Authorization/locks@2020-05-01' = if (deployLock) {
  name: 'no-delete'
  scope: st
  properties: { level: 'CanNotDelete' }
}

var endpoints = [for (name, i) in containers: '${st.properties.primaryEndpoints.blob}${name}']
output containerNames array = [for (name, i) in containers: c[i].name]
```

Loops work on resources, modules, variables, properties and outputs, over arrays, ranges (`range(0, 3)`) and object items (`items(obj)`).

Useful operators: ternary `a ? b : c`, null-coalescing `a ?? b`, safe dereference `obj.?prop`, non-null assertion `x!`, spread `...obj`, and lambda functions (`map`, `filter`, `reduce`, `sort`, `toObject`).

### Modules

A module is any Bicep file called from another. Modules are the unit of reuse.

```bicep
module storage './modules/storage.bicep' = {
  name: 'storage-${env}'                  // optional in recent versions
  params: {
    workload: workload
    env: env
    location: location
  }
}

module network 'br/public:avm/res/network/virtual-network:0.5.2' = {   // from a registry
  params: { name: 'vnet-${workload}', addressPrefixes: ['10.10.0.0/16'] }
}

module rg2 './modules/monitoring.bicep' = {
  scope: resourceGroup('rg-monitor-prod')     // deploy to another scope
  params: { ... }
}

output storageId string = storage.outputs.storageId
```

Module sources:

| Source | Syntax | Use |
|---|---|---|
| Local file | `'./modules/x.bicep'` | Within a repository |
| Public registry | `'br/public:avm/res/<provider>/<type>:<version>'` | Azure Verified Modules (file 06) |
| Private registry (ACR) | `'br:myregistry.azurecr.io/bicep/modules/x:1.2.0'` | Sharing across an organisation |
| Template spec | `'ts:<sub>/<rg>/<name>:<version>'` | Sharing with RBAC in Azure (file 07) |

Publish with `az bicep publish --file x.bicep --target br:myregistry.azurecr.io/bicep/modules/x:1.2.0`. Aliases in `bicepconfig.json` shorten registry paths. Always pin a version.

Each module becomes a nested deployment. Modules can target a different scope, which is how one file creates a resource group at subscription scope and then fills it:

```bicep
targetScope = 'subscription'
resource rg 'Microsoft.Resources/resourceGroups@2024-03-01' = { name: 'rg-${workload}-${env}', location: location }
module app './app.bicep' = { scope: rg, params: { workload: workload, env: env } }
```

### Types, functions and sharing

```bicep
@export()
type environmentName = 'dev' | 'test' | 'prod'

@export()
type modelDeployment = {
  name: string
  model: string
  version: string
  @minValue(1)
  capacity: int
  sku: 'Standard' | 'GlobalStandard' | 'DataZoneStandard' | 'ProvisionedManaged'
}

@export()
func resourceName(prefix string, workload string, env string) string => '${prefix}-${workload}-${env}'
```

```bicep
import { modelDeployment, resourceName } from './shared/types.bicep'
param deployments modelDeployment[]
```

User-defined types give modules a typed, self-documenting interface and catch mistakes at compile time. Resource-derived types (`resourceInput<'Microsoft.Storage/storageAccounts@2023-05-01'>.sku`) reuse the platform's own schema.

### Parameter files

`.bicepparam` files are Bicep-syntax parameter files, one per environment:

```bicep
// prod.bicepparam
using './main.bicep'

param workload = 'hrassist'
param env = 'prod'
param deployments = [
  { name: 'chat', model: 'gpt-4.1-mini', version: '2025-04-14', capacity: 50, sku: 'DataZoneStandard' }
]
param adminPassword = az.getSecret('<sub-id>', 'rg-shared', 'kv-shared', 'vmAdminPassword')
param buildId = readEnvironmentVariable('BUILD_ID', 'local')
```

They support expressions, variables, `az.getSecret()` for Key Vault, `readEnvironmentVariable()`, and `extends` to share a base. Deploy with `--parameters prod.bicepparam`.

### Validation and preview

| Step | Command | Catches |
|---|---|---|
| **Lint** | `az bicep lint --file main.bicep` | Style and correctness rules (unused params, hard-coded locations, secrets in outputs). Configured in `bicepconfig.json` |
| **Build** | `az bicep build --file main.bicep` | Syntax and type errors |
| **Validate** | `az deployment group validate` | Preflight against ARM: names, quotas, some policy |
| **What-if** | `az deployment group what-if` | The actual changes: create, modify, delete, no change, ignore |
| **Deploy** | `az deployment group create` | – |

What-if output marks each resource `+` create, `~` modify, `-` delete, `=` no change, `*` ignore, `!` deploy (unknown). It can show false positives ("noise") for some properties, and has limits on nested module depth; treat it as a strong signal, not a proof.

### Linter and configuration

`bicepconfig.json` at the repository root sets linter rule levels, module aliases, and experimental feature flags:

```json
{
  "analyzers": { "core": { "rules": {
    "no-hardcoded-location": { "level": "error" },
    "outputs-should-not-contain-secrets": { "level": "error" },
    "use-recent-api-versions": { "level": "warning" },
    "secure-parameter-default": { "level": "error" }
  } } },
  "moduleAliases": { "br": { "corp": { "registry": "contosomodules.azurecr.io", "modulePath": "bicep/modules" } } }
}
```

### Extensions

Bicep can manage some non-ARM resources through extensions:

| Extension | Manages |
|---|---|
| **Microsoft Graph** | Entra app registrations, service principals, groups, federated credentials |
| **Kubernetes** | Manifests in an AKS cluster |

The Graph extension lets one deployment create an app registration, its federated credential and the Azure role assignment together. Check each extension's maturity before depending on it in production.

## How it works on Azure

### A Foundry workload in Bicep

A minimal, keyless Foundry resource with a project, a model deployment and a role assignment. Verify the API version and SKU names against the current resource reference before use.

```bicep
param location string = resourceGroup().location
param workload string = 'hrassist'
param env string = 'dev'
param appPrincipalId string            // object ID of the app's managed identity

var suffix = take(uniqueString(resourceGroup().id), 5)

resource foundry 'Microsoft.CognitiveServices/accounts@2025-06-01' = {
  name: 'aif-${workload}-${env}-${suffix}'
  location: location
  kind: 'AIServices'
  sku: { name: 'S0' }
  identity: { type: 'SystemAssigned' }
  properties: {
    allowProjectManagement: true
    customSubDomainName: 'aif-${workload}-${env}-${suffix}'
    disableLocalAuth: true                 // keyless only
    publicNetworkAccess: 'Disabled'        // pair with a private endpoint
  }
}

resource project 'Microsoft.CognitiveServices/accounts/projects@2025-06-01' = {
  parent: foundry
  name: 'proj-${workload}'
  location: location
  identity: { type: 'SystemAssigned' }
  properties: { displayName: 'HR assistant' }
}

resource chat 'Microsoft.CognitiveServices/accounts/deployments@2025-06-01' = {
  parent: foundry
  name: 'chat'
  sku: { name: 'DataZoneStandard', capacity: 50 }      // capacity in thousands of tokens per minute
  properties: {
    model: { format: 'OpenAI', name: 'gpt-4.1-mini', version: '2025-04-14' }
    versionUpgradeOption: 'NoAutoUpgrade'
  }
}

resource openAiUser 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(foundry.id, appPrincipalId, 'openai-user')
  scope: foundry
  properties: {
    principalId: appPrincipalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '5e0bd9bd-7b93-4f28-af87-19fc36ad61bd')  // Cognitive Services OpenAI User
  }
}

output endpoint string = foundry.properties.endpoint
```

Multiple model deployments on one account must be created **serially**; add `@batchSize(1)` to a deployment loop or chain them with `dependsOn`, otherwise you get conflict errors.

### Repository layout

```text
infra/
├── main.bicep                 orchestration: scopes and modules only
├── main.dev.bicepparam
├── main.prod.bicepparam
├── bicepconfig.json
├── shared/types.bicep         exported types and functions
└── modules/
    ├── network.bicep
    ├── identity.bicep
    ├── ai.bicep
    ├── data.bicep
    └── app.bicep
```

Keep `main.bicep` thin. Put resources in modules with typed parameters and minimal outputs.

### In a pipeline

```yaml
# GitHub Actions (excerpt)
permissions: { id-token: write, contents: read }
steps:
  - uses: actions/checkout@v4
  - uses: azure/login@v2
    with: { client-id: ${{ vars.AZURE_CLIENT_ID }}, tenant-id: ${{ vars.AZURE_TENANT_ID }}, subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }} }
  - run: az bicep lint --file infra/main.bicep
  - run: az deployment group what-if -g $RG -f infra/main.bicep -p infra/main.prod.bicepparam
  # manual approval gate on the environment, then:
  - run: az deployment group create  -g $RG -f infra/main.bicep -p infra/main.prod.bicepparam
```

Post the what-if result on the pull request so reviewers see the real change. Topic 27 covers pipelines in depth.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Bicep or Terraform | Bicep for Azure-only teams wanting no state and day-zero features. Terraform for multi-cloud or an existing Terraform practice. See file 05 |
| Write resources or use modules | Azure Verified Modules for standard resources; your own modules for organisation-specific composition |
| Registry or template specs | Container registry for developer-style module versioning and CI; template specs when you want Azure RBAC and portal deployment |
| One big deployment or several | Split by lifecycle: network, shared data, application. Smaller deployments are faster and limit blast radius |
| API versions | Recent stable versions; preview only for a feature you need, noted in a comment |
| Removing resources | Deleting a resource from a Bicep file does **not** delete it in Azure (incremental mode). Use deployment stacks for that |
| Secrets | Avoid them. Managed identity first; `@secure()` params from Key Vault when unavoidable; never in outputs |

**The no-state trade-off.** Bicep does not know what it deployed previously, so it cannot detect a resource you removed from the file, and drift is only visible through what-if. Deployment stacks (file 07) add managed lifecycle on top.

## Hands-on

Build the structure below, then deploy it.

```bash
mkdir -p bicep-lab/modules && cd bicep-lab

cat > modules/storage.bicep <<'EOF'
param name string
param location string
param containers array = []
param tags object = {}

resource st 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: name
  location: location
  tags: tags
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: { allowBlobPublicAccess: false, allowSharedKeyAccess: false, minimumTlsVersion: 'TLS1_2' }
}
resource blob 'Microsoft.Storage/storageAccounts/blobServices@2023-05-01' = { parent: st, name: 'default' }
resource c 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-05-01' = [for n in containers: { parent: blob, name: n }]

output id string = st.id
output name string = st.name
EOF

cat > main.bicep <<'EOF'
@minLength(2) @maxLength(10)
param workload string
@allowed(['dev','prod'])
param env string = 'dev'
param location string = resourceGroup().location
param containers array = ['docs']

var suffix = take(uniqueString(resourceGroup().id), 5)
var tags = { application: workload, environment: env, managedBy: 'bicep' }

resource id 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'id-${workload}-${env}'
  location: location
  tags: tags
}

module storage './modules/storage.bicep' = {
  params: { name: toLower('st${workload}${env}${suffix}'), location: location, containers: containers, tags: tags }
}

resource st 'Microsoft.Storage/storageAccounts@2023-05-01' existing = { name: storage.outputs.name }

resource reader 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(st.id, id.id, 'blob-reader')
  scope: st
  properties: {
    principalId: id.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '2a2b9908-6ea1-4ae2-8e65-a410df84e7d1')
  }
}

output storageName string = storage.outputs.name
output identityClientId string = id.properties.clientId
EOF

cat > dev.bicepparam <<'EOF'
using './main.bicep'
param workload = 'biceplab'
param env = 'dev'
param containers = ['docs', 'archive']
EOF

RG=rg-bicep-lab
az group create -n $RG -l swedencentral -o none
az bicep lint  --file main.bicep
az deployment group what-if -g $RG -f main.bicep -p dev.bicepparam
az deployment group create  -g $RG -f main.bicep -p dev.bicepparam --query properties.outputs

# Idempotency: run what-if again and expect no changes
az deployment group what-if -g $RG -f main.bicep -p dev.bicepparam

# Remove 'archive' from dev.bicepparam, redeploy, and confirm the container still exists (no state, incremental)
# View the compiled JSON and the dependency graph
az bicep build --file main.bicep --stdout | head -40

az group delete -n $RG --yes --no-wait
```

Then open `main.bicep` in VS Code and use *Open Bicep Visualizer*. Extensions: add a typed `containers` parameter with a user-defined type, and add a `CanNotDelete` lock for `prod` only.

## Security, cost and reliability notes

- **Security:** mark sensitive parameters `@secure()`; they are excluded from deployment history. Do not output secrets; the linter flags it.
- **Security:** the deploying identity needs rights to create every resource and, for role assignments, *Role Based Access Control Administrator* (ideally with a condition) or Owner. Scope it tightly.
- **Security:** bake secure defaults into modules: no public access, no local auth, TLS 1.2, diagnostics on.
- **Cost:** what-if shows resources about to be created. Review SKUs before approving.
- **Reliability:** deployments are not transactional. A failure part-way leaves earlier resources in place; fix and redeploy. Design for re-runnability.
- **Reliability:** pin module and API versions so a build is reproducible.

## Common mistakes

- Expecting a resource removed from the file to be deleted.
- Random or non-deterministic names that change each run.
- `newGuid()` for role assignment names, creating duplicates on every deployment.
- Omitting `principalType`, causing intermittent "principal not found" errors.
- Parallel model deployments on one Foundry account.
- Hard-coded locations, subscription IDs and resource IDs.
- Enormous single files with no modules.
- Unpinned registry modules.
- Ignoring what-if because "it is only a small change".
- Secrets as plain `string` parameters.

## Check yourself

1. Where does Bicep keep its state?
2. How does Bicep know the order in which to deploy resources?
3. How do you reference a resource that already exists without redeploying it?
4. Why name a role assignment with `guid(scope, principal, role)`?
5. What do lint, build, validate and what-if each check?
6. What happens when you remove a resource from a Bicep file and redeploy?
7. Name three places a module can be loaded from.
8. How do you pass a Key Vault secret as a parameter?

<details>
<summary>Answers</summary>

1. Nowhere. Azure itself is the state; there is no state file.
2. From symbolic references between resources, which create implicit dependencies.
3. Declare it with the `existing` keyword, optionally with a `scope`.
4. It produces a deterministic name so redeployment updates the same assignment instead of creating a new one.
5. Lint: style and best-practice rules. Build: syntax and types. Validate: ARM preflight checks. What-if: the actual changes the deployment would make.
6. Nothing happens to the resource; incremental deployments do not delete. Use a deployment stack to manage removal.
7. A local file, a container registry (public or private), a template spec.
8. With `az.getSecret()` in a `.bicepparam` file, feeding a `@secure()` parameter.

</details>

## References

- [What is Bicep?](https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview)
- [Bicep file structure and syntax](https://learn.microsoft.com/azure/azure-resource-manager/bicep/file)
- [Bicep modules](https://learn.microsoft.com/azure/azure-resource-manager/bicep/modules)
- [User-defined data types in Bicep](https://learn.microsoft.com/azure/azure-resource-manager/bicep/user-defined-data-types)
- [Bicep parameters files](https://learn.microsoft.com/azure/azure-resource-manager/bicep/parameter-files)
- [Bicep what-if operation](https://learn.microsoft.com/azure/azure-resource-manager/bicep/deploy-what-if)
- [Use Bicep linter](https://learn.microsoft.com/azure/azure-resource-manager/bicep/linter)
- [Best practices for Bicep](https://learn.microsoft.com/azure/azure-resource-manager/bicep/best-practices)
- [Bicep templates for Microsoft Graph](https://learn.microsoft.com/graph/templates/bicep/overview-bicep-templates-for-graph)
- [Microsoft.CognitiveServices resource reference](https://learn.microsoft.com/azure/templates/microsoft.cognitiveservices/accounts)
- [Learn Bicep (Microsoft Learn path)](https://learn.microsoft.com/training/paths/fundamentals-bicep/)
