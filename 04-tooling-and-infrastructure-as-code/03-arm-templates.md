# ARM Templates

> Topic: 04 · Tooling and Infrastructure as Code · Tracks: E ○ A – C – · Status checked: 2026-10

## Why it matters

ARM templates are the JSON format Azure Resource Manager natively understands. You will rarely write one by hand now, because Bicep compiles to this format and is far easier to work with. You still need to **read** them: the portal's *Export template* produces JSON, older estates are full of it, deployment history stores it, and when a Bicep deployment fails the error refers to the compiled template. This file is for literacy, not authorship.

## Key concepts

### Template anatomy

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageName": { "type": "string", "minLength": 3, "maxLength": 24 },
    "location":    { "type": "string", "defaultValue": "[resourceGroup().location]" },
    "sku":         { "type": "string", "defaultValue": "Standard_LRS", "allowedValues": ["Standard_LRS", "Standard_ZRS"] }
  },
  "variables": {
    "tags": { "env": "dev", "managedBy": "arm" }
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2023-05-01",
      "name": "[parameters('storageName')]",
      "location": "[parameters('location')]",
      "sku": { "name": "[parameters('sku')]" },
      "kind": "StorageV2",
      "tags": "[variables('tags')]",
      "properties": { "allowBlobPublicAccess": false, "minimumTlsVersion": "TLS1_2" }
    }
  ],
  "outputs": {
    "storageId": { "type": "string", "value": "[resourceId('Microsoft.Storage/storageAccounts', parameters('storageName'))]" }
  }
}
```

| Section | Purpose |
|---|---|
| `$schema` | Identifies the deployment scope: `deploymentTemplate` (resource group), `subscriptionDeploymentTemplate`, `managementGroupDeploymentTemplate`, `tenantDeploymentTemplate` |
| `contentVersion` | Your own version label |
| `parameters` | Inputs: `string`, `securestring`, `int`, `bool`, `object`, `secureObject`, `array` |
| `variables` | Computed values reused in the template |
| `functions` | User-defined functions (rare) |
| `resources` | What to deploy |
| `outputs` | Values returned after deployment |

### Expressions and functions

Anything in `[...]` is an expression evaluated at deployment time.

| Function | Use |
|---|---|
| `parameters('x')`, `variables('y')` | Read inputs and variables |
| `resourceGroup()`, `subscription()`, `tenant()`, `deployment()` | Context |
| `resourceId(type, name…)` | Build a resource ID |
| `reference(resourceId)` | Read runtime properties of a resource; creates an implicit dependency |
| `listKeys(...)`, `list*()` | Call list operations, for example to fetch keys |
| `concat()`, `format()`, `uniqueString()`, `toLower()`, `substring()` | Strings |
| `if()`, `equals()`, `and()`, `or()`, `not()` | Logic |
| `union()`, `length()`, `contains()` | Collections |
| `copyIndex()` | Position in a loop |

### Dependencies, loops and conditions

```json
{
  "condition": "[parameters('deployContainer')]",
  "type": "Microsoft.Storage/storageAccounts/blobServices/containers",
  "apiVersion": "2023-05-01",
  "name": "[format('{0}/default/{1}', parameters('storageName'), 'docs')]",
  "dependsOn": [
    "[resourceId('Microsoft.Storage/storageAccounts', parameters('storageName'))]"
  ],
  "copy": { "name": "containerLoop", "count": 3 }
}
```

- **`dependsOn`** orders deployment explicitly. `reference()` and `resourceId()` to a resource in the same template also establish order.
- **`copy`** creates multiple instances; **`condition`** deploys a resource only when true.
- Resources without dependencies between them deploy **in parallel**.

### Child resources

Written either nested inside the parent's `resources` array, or at top level with a name that includes the parent (`parent/child`) and a full type (`Microsoft.Storage/storageAccounts/blobServices/containers`).

### Linked and nested templates

A resource of type `Microsoft.Resources/deployments` deploys another template, either inline (**nested**) or from a URI or template spec (**linked**). This is how large solutions were modularised before Bicep modules. Linked templates need a location ARM can reach, which is why template specs exist (file 07).

### Parameter files

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageName": { "value": "stdemoprodswc001" },
    "adminPassword": {
      "reference": {
        "keyVault": { "id": "/subscriptions/.../providers/Microsoft.KeyVault/vaults/kv-demo" },
        "secretName": "vmAdminPassword"
      }
    }
  }
}
```

A Key Vault reference keeps the secret out of the file. The vault must have *enabled for template deployment* (or the deploying identity needs the appropriate permission under RBAC).

### Symbolic names (language version 2.0)

Templates compiled from recent Bicep versions may declare `"languageVersion": "2.0"`, in which case `resources` is an **object** keyed by symbolic name instead of an array. If exported or compiled JSON looks unfamiliar, this is usually why.

### Deployment mechanics

Covered in topic 01, file 05. In short: a deployment is submitted at a scope (resource group, subscription, management group or tenant), in **incremental** (default) or **complete** mode, and can be previewed with **what-if**.

```bash
az deployment group validate  -g rg-demo --template-file azuredeploy.json --parameters @params.json
az deployment group what-if   -g rg-demo --template-file azuredeploy.json --parameters @params.json
az deployment group create    -g rg-demo --template-file azuredeploy.json --parameters @params.json
az deployment group list      -g rg-demo -o table
az deployment operation group list -g rg-demo -n <deployment-name> \
  --query "[?properties.provisioningState=='Failed'].properties.statusMessage"
```

## How it works on Azure

### JSON and Bicep compared

The same storage account in Bicep:

```bicep
param storageName string
param location string = resourceGroup().location
@allowed(['Standard_LRS', 'Standard_ZRS'])
param sku string = 'Standard_LRS'

resource st 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageName
  location: location
  sku: { name: sku }
  kind: 'StorageV2'
  tags: { env: 'dev', managedBy: 'bicep' }
  properties: { allowBlobPublicAccess: false, minimumTlsVersion: 'TLS1_2' }
}

output storageId string = st.id
```

Bicep is a transparent abstraction: every Bicep file compiles to an ARM template, and ARM sees no difference.

### Moving between the two

```bash
az bicep decompile --file azuredeploy.json     # JSON → Bicep (best effort; review and tidy)
az bicep build     --file main.bicep           # Bicep → JSON
az group export -n rg-demo > exported.json     # Existing resources → JSON
```

Decompiled and exported output is a starting point. Expect hard-coded names, superfluous default properties and missing parameterisation.

### When JSON still appears

| Situation | What to do |
|---|---|
| Portal *Export template* or *Download a template for automation* | Decompile to Bicep and clean up |
| Inherited estate with linked templates | Maintain as-is, or migrate module by module to Bicep |
| Marketplace offers and managed applications | These require a `mainTemplate.json`; author in Bicep and compile |
| Azure Policy `deployIfNotExists` | The embedded deployment is an ARM template |
| Deployment history and error messages | Read the compiled template to find the failing resource |
| Some third-party tools | May generate or expect JSON |

## Design decisions and trade-offs

- **New work: Bicep (or Terraform), not JSON.** There is no capability advantage to hand-written JSON.
- **Migrate opportunistically.** Convert a template when you next need to change it. A big-bang conversion of working templates carries risk for little gain.
- **Avoid complete mode.** Use deployment stacks for lifecycle management.
- **Keep secrets out of parameter files.** Use Key Vault references or pipeline secret variables.

## Hands-on

1. In the portal, open any resource group and choose **Export template**. Read the JSON and identify parameters, a resource's `type` and `apiVersion`, and any `dependsOn`.
2. Decompile it and compare:

   ```bash
   az group export -n <rg> > exported.json
   az bicep decompile --file exported.json
   ```

   Note what you would change before using the result.
3. Take the Bicep file from the Bicep lab (file 04), run `az bicep build`, and read the generated JSON. Find how a Bicep `module` appears as `Microsoft.Resources/deployments`.
4. Deliberately break a deployment (for example an invalid SKU) and find the error with `az deployment operation group list`.
5. Open **Deployments** on a resource group in the portal and inspect the template and inputs of a past deployment.

## Security, cost and reliability notes

- **Security:** deployment history stores templates, parameter values and outputs. `securestring` parameters are not stored, but anything passed as plain `string` or emitted as an output is visible to anyone who can read deployments. Never output secrets.
- **Security:** `listKeys()` in a template pulls keys into the deployment. Prefer managed identity and RBAC so keys are not needed.
- **Reliability:** resource group deployment history is capped at 800 entries and pruned automatically.
- **Reliability:** always run `what-if` before deploying to shared environments.

## Common mistakes

- Writing new solutions in JSON.
- Using exported templates unchanged.
- Returning keys or connection strings in `outputs`.
- Missing dependencies that make deployments fail intermittently.
- Complete mode against a resource group that contains anything else.
- Editing compiled JSON instead of the Bicep source.

## Check yourself

1. Name the main sections of an ARM template.
2. What does `reference()` do, and what side effect does it have?
3. How do you keep a password out of a parameter file?
4. How does Bicep relate to ARM templates?
5. Where would you look to find which resource failed in a deployment?
6. Why should you not put secrets in template outputs?

<details>
<summary>Answers</summary>

1. `$schema`, `contentVersion`, `parameters`, `variables`, `functions`, `resources`, `outputs`.
2. Returns runtime properties of a resource and creates an implicit dependency on it.
3. With a Key Vault reference in the parameter file, or by supplying the value from a pipeline secret at deployment time.
4. Bicep is a language that compiles to an ARM JSON template; ARM deploys the JSON.
5. The deployment's operations: `az deployment operation group list`, or the *Deployments* blade in the portal.
6. Outputs are stored in deployment history and readable by anyone with access to deployments.

</details>

## References

- [What are ARM templates?](https://learn.microsoft.com/azure/azure-resource-manager/templates/overview)
- [Understand the structure and syntax of ARM templates](https://learn.microsoft.com/azure/azure-resource-manager/templates/syntax)
- [ARM template functions](https://learn.microsoft.com/azure/azure-resource-manager/templates/template-functions)
- [Use Azure Key Vault to pass secure parameter values](https://learn.microsoft.com/azure/azure-resource-manager/templates/key-vault-parameter)
- [Decompile ARM template JSON to Bicep](https://learn.microsoft.com/azure/azure-resource-manager/bicep/decompile)
- [Azure resource reference (all types and API versions)](https://learn.microsoft.com/azure/templates/)
