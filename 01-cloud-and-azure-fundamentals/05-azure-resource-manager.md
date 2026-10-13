# Azure Resource Manager

> Topic: 01 · Cloud and Azure Fundamentals · Tracks: E ● A ● C ● · Status checked: 2026-10

## Why it matters

Azure Resource Manager (ARM) is the single front door for creating, changing and deleting anything in Azure. The portal, the CLI, PowerShell, Bicep, Terraform and the SDKs are all clients of the same API. Once you understand ARM, error messages make sense, infrastructure as code stops being magic, and you can reason about why "Contributor" does not let someone read a blob.

## Key concepts

### One API, many clients

```text
Portal   CLI   PowerShell   Bicep   Terraform   SDKs   REST
   \      |        |          |         |         |     /
    └─────┴────────┴──── Azure Resource Manager ──┴────┘
              authentication (Entra ID)
              authorisation (RBAC)
              policy evaluation
              locks, throttling, activity logging
                          │
        ┌─────────────────┼─────────────────┐
  Microsoft.Compute  Microsoft.Storage  Microsoft.CognitiveServices ...
              (resource providers)
```

Every request to `https://management.azure.com` goes through the same steps:

1. **Authenticate** the caller with Microsoft Entra ID.
2. **Authorise** the action with Azure RBAC at the target scope.
3. **Evaluate Azure Policy** (deny, modify, append, audit).
4. **Check locks.**
5. **Route** to the resource provider, which does the work.
6. **Record** the operation in the activity log.

Because every tool takes this path, a policy that denies public storage accounts blocks the portal, a pipeline and a script equally.

### Control plane and data plane

| | Control plane | Data plane |
|---|---|---|
| What it does | Manages resources | Uses the capability a resource provides |
| Endpoint | `management.azure.com` | Service-specific, for example `<account>.blob.core.windows.net` |
| Goes through ARM | Yes | No |
| Example: storage | Create an account, set its firewall | Read or write a blob |
| Example: Foundry | Create a resource, deploy a model | Send a chat completion request |
| Example: Key Vault | Create a vault, set network rules | Get a secret |

The planes have separate permissions. RBAC roles list control-plane permissions under `actions` and data-plane permissions under `dataActions`. That is why **Contributor** can create a storage account but cannot read its blobs through Entra ID: Contributor has no `dataActions`. You need a data role such as *Storage Blob Data Reader*.

The same applies to AI. Managing a Foundry resource and calling a model on it are different permissions (for example *Cognitive Services OpenAI User* or the Foundry user roles for inference).

Azure Policy and locks mostly govern the control plane. A `ReadOnly` lock on a storage account does not stop someone uploading blobs.

### Resource providers and types

A **resource provider** is a service that supplies resource types. Its name is a namespace:

| Namespace | Example resource types |
|---|---|
| `Microsoft.Compute` | `virtualMachines`, `disks`, `virtualMachineScaleSets` |
| `Microsoft.Network` | `virtualNetworks`, `privateEndpoints`, `applicationGateways` |
| `Microsoft.Storage` | `storageAccounts` |
| `Microsoft.Web` | `sites`, `serverfarms` |
| `Microsoft.App` | `containerApps`, `managedEnvironments` |
| `Microsoft.CognitiveServices` | `accounts` (Foundry resources, Azure OpenAI), `accounts/projects`, `accounts/deployments` |
| `Microsoft.Search` | `searchServices` |
| `Microsoft.DocumentDB` | `databaseAccounts` (Cosmos DB) |

A provider must be **registered** in a subscription before you can create its resources. Common providers are registered by default and the portal and CLI register most others automatically. In locked-down subscriptions, registration can fail for users without the `*/register/action` permission, producing a `MissingSubscriptionRegistration` error.

### API versions

Every resource type is exposed through dated API versions such as `2025-04-01` or `2025-06-01-preview`. The version fixes the shape of the resource: which properties exist and what they mean.

- New features appear in new API versions, often in a `-preview` version first.
- Old API versions are eventually retired.
- Bicep and ARM templates pin an API version per resource. Terraform's AzureRM provider pins them for you per provider release; AzAPI lets you choose.

When a new Foundry feature "isn't supported in Bicep yet", it usually means it only exists in a preview API version you have not selected.

### Declarative and idempotent

ARM's core write operation is `PUT`: "make this resource look like this". Sending the same definition twice gives the same result. This idempotency is what makes infrastructure as code work. You describe the desired end state and deploy it repeatedly without worrying about the current state.

### Deployments

A **deployment** is a named operation that submits a template (ARM JSON, or Bicep which compiles to it) to ARM. ARM works out dependencies, creates resources in parallel where it can, and records the result.

**Deployment scopes:**

| Scope | Typical use | CLI command |
|---|---|---|
| Resource group | Workload resources | `az deployment group create` |
| Subscription | Resource groups, policy and role assignments | `az deployment sub create` |
| Management group | Policy and RBAC across subscriptions | `az deployment mg create` |
| Tenant | Management groups | `az deployment tenant create` |

**Deployment modes** (resource group scope):

| Mode | Behaviour |
|---|---|
| **Incremental** (default) | Adds and updates resources in the template. Leaves everything else alone |
| **Complete** | Makes the resource group match the template exactly. **Deletes** resources not in the template |

Complete mode is dangerous and is being superseded by **deployment stacks**, which manage a set of resources as a unit with explicit behaviour for resources that are removed from the template (covered in topic 04).

**What-if** previews the changes a deployment would make before you run it. Use it in every pipeline.

Limits to know: 800 deployments are kept in history per resource group (older entries are pruned automatically), up to 800 resources per deployment, and a 4 MB maximum request size.

### Locks

| Lock | Effect |
|---|---|
| `CanNotDelete` | Read and modify allowed. Delete blocked |
| `ReadOnly` | Read allowed. Modify and delete blocked |

Locks are inherited by child scopes and apply to everyone, including Owners, until removed. Only Owner and User Access Administrator (or a custom role with `Microsoft.Authorization/locks/*`) can manage them. Up to 20 locks per scope.

`ReadOnly` has side effects because some "read-like" operations are POST requests. A read-only lock on a storage account blocks listing its keys; on an App Service it blocks scaling.

### Tags

Tags are name and value pairs on resources, resource groups and subscriptions.

- Up to 50 per resource, resource group or subscription.
- Name up to 512 characters, value up to 256 (some resource types are stricter).
- **Not inherited.** Use Azure Policy to copy them down or enforce them.
- Not every resource type supports tags.
- Tags are stored as plain text and appear in cost exports. Never put secrets in them.

Typical tags: `env`, `owner`, `costCentre`, `application`, `dataClassification`, `criticality`.

### Throttling

ARM limits request rates using a **token bucket** applied per region.

| Scope | Operation | Bucket size | Refill per second |
|---|---|---|---|
| Subscription | Reads | 250 | 25 |
| Subscription | Writes | 200 | 10 |
| Subscription | Deletes | 200 | 10 |
| Tenant | Reads | 250 | 25 |
| Tenant | Writes | 200 | 10 |
| Tenant | Deletes | 200 | 10 |

Subscription limits apply per subscription, per service principal and per operation type, with a global subscription limit of 15 times the per-principal value. Resource providers add their own, often tighter, limits on top.

When throttled you get **HTTP 429** with a `Retry-After` header. Honour it. Response headers such as `x-ms-ratelimit-remaining-subscription-reads` show how much budget is left. The Azure SDKs and CLI retry automatically.

### Activity log

Every control-plane write is recorded in the **activity log**: who, what, when, from where, and whether it succeeded. It is kept for 90 days by default; send it to a Log Analytics workspace for longer retention. It is the first place to look when something changed and nobody knows why. It does not record data-plane operations; those need resource diagnostic logs.

## How it works on Azure

Anatomy of a real call. Creating a resource group from the CLI:

```bash
az group create -n rg-demo -l swedencentral --debug
```

sends:

```http
PUT https://management.azure.com/subscriptions/{sub}/resourcegroups/rg-demo?api-version=2022-09-01
Authorization: Bearer <token from Entra ID>
Content-Type: application/json

{ "location": "swedencentral" }
```

ARM validates the token, checks that you hold `Microsoft.Resources/subscriptions/resourceGroups/write` at that scope, evaluates policy, and returns `201 Created` or `200 OK` if it already existed with the same definition.

Long-running operations return `201` or `202` with a URL to poll. Tools hide the polling; `--no-wait` in the CLI skips it.

## Design decisions and trade-offs

- **Portal for learning, code for anything that matters.** Portal changes are not repeatable or reviewable. Use the portal to explore, then export or rewrite as Bicep or Terraform.
- **Scope deployments as low as possible.** Resource group scope for workloads; reserve subscription and management group deployments for the platform team.
- **Avoid complete mode.** Use deployment stacks when you need "delete what I removed" behaviour.
- **Locks are a backstop, not access control.** They stop accidents by privileged users. They break automation that legitimately needs to delete, so pair them with a documented removal process.
- **Enforce tags with policy at creation.** Retro-tagging thousands of resources is miserable.
- **Design automation for throttling.** Batch reads through Azure Resource Graph instead of looping over ARM GET calls.

## Hands-on

```bash
# 1. See the raw API call behind a CLI command
az group list --debug 2>&1 | grep -E "Request URL|x-ms-ratelimit"

# 2. Call ARM directly
az rest --method get \
  --url "https://management.azure.com/subscriptions/$(az account show --query id -o tsv)/resourcegroups?api-version=2022-09-01" \
  --query "value[].name"

# 3. Explore resource providers
az provider list --query "[?registrationState=='Registered'].namespace" -o tsv | sort | head -30
az provider show -n Microsoft.CognitiveServices \
  --query "resourceTypes[?resourceType=='accounts'].apiVersions[0:5]" -o json

# 4. Deploy with Bicep and preview with what-if
az group create -n rg-arm-lab -l swedencentral

cat > main.bicep <<'EOF'
param location string = resourceGroup().location
param suffix string = uniqueString(resourceGroup().id)

resource st 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: 'starmlab${suffix}'
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: {
    allowBlobPublicAccess: false
    minimumTlsVersion: 'TLS1_2'
  }
  tags: { env: 'lab' }
}

output storageId string = st.id
EOF

az deployment group what-if -g rg-arm-lab --template-file main.bicep
az deployment group create  -g rg-arm-lab --template-file main.bicep

# 5. Run it again: idempotent, no changes
az deployment group what-if -g rg-arm-lab --template-file main.bicep

# 6. Control plane vs data plane: you created the account, but can you list blobs with Entra auth?
ST=$(az storage account list -g rg-arm-lab --query "[0].name" -o tsv)
az storage container list --account-name $ST --auth-mode login   # likely fails: no data role

# 7. Who did what?
az monitor activity-log list -g rg-arm-lab --offset 1h \
  --query "[].{Time:eventTimestamp, Op:operationName.localizedValue, Status:status.value, By:caller}" -o table

# 8. Clean up
az group delete -n rg-arm-lab --yes --no-wait
```

Step 6 is the important one. If it fails with an authorisation error while you are Owner of the subscription, you have just seen the control plane and data plane split first-hand. Assign yourself *Storage Blob Data Reader* on the account and it will work.

## Security, cost and reliability notes

- **Security:** prefer data-plane RBAC with Entra ID over account keys and connection strings. Many services let you disable key authentication entirely; do so.
- **Security:** export the activity log to a workspace and alert on sensitive operations such as role assignments, lock deletions and policy exemptions.
- **Cost:** ARM itself is free. Deployments are not rate-billed, but what they create is.
- **Reliability:** ARM is a globally distributed service, yet resource group metadata is regional. Keep groups co-located with their resources.

## Common mistakes

- Assuming Owner or Contributor grants access to data.
- Using complete mode against a shared resource group and deleting someone else's resources.
- Hard-coding an old API version and wondering why a new property is rejected.
- Putting a `ReadOnly` lock on a resource and breaking key listing, scaling or backups.
- Ignoring `Retry-After` and hammering a throttled API.
- Looking in the activity log for data access events.
- Making portal changes to resources managed by IaC, creating drift that the next deployment silently reverts.

## Check yourself

1. What steps does ARM perform on every request?
2. Explain control plane vs data plane with a Foundry example.
3. Why can a subscription Owner be denied when reading a blob with Entra authentication?
4. What is a resource provider, and what does registering one do?
5. What is the difference between incremental and complete deployment modes?
6. What does a `CanNotDelete` lock allow and block? Who can remove it?
7. What status code signals throttling, and what should a client do?
8. Why is idempotency essential to infrastructure as code?

<details>
<summary>Answers</summary>

1. Authenticate with Entra ID, authorise with RBAC, evaluate policy, check locks, route to the resource provider, log to the activity log.
2. Control plane: creating a Foundry resource or a model deployment through `management.azure.com`. Data plane: sending a chat completion to the deployment's inference endpoint. Different endpoints and different permissions.
3. Owner has control-plane `actions` but no `dataActions`. Reading blobs needs a data role.
4. A service that supplies resource types under a namespace. Registration enables that provider's types in a subscription.
5. Incremental adds and updates only what is in the template. Complete also deletes resources in the resource group that are not in the template.
6. Allows read and modify, blocks delete. Owner, User Access Administrator, or a custom role with lock permissions.
7. 429. Wait for the `Retry-After` interval and retry with backoff.
8. Because you can re-apply the same desired state safely any number of times and converge on the same result.

</details>

## References

- [What is Azure Resource Manager?](https://learn.microsoft.com/azure/azure-resource-manager/management/overview)
- [Azure control plane and data plane](https://learn.microsoft.com/azure/azure-resource-manager/management/control-plane-and-data-plane)
- [Azure resource providers and types](https://learn.microsoft.com/azure/azure-resource-manager/management/resource-providers-and-types)
- [Understand how Azure Resource Manager throttles requests](https://learn.microsoft.com/azure/azure-resource-manager/management/request-limits-and-throttling)
- [Azure Resource Manager deployment modes](https://learn.microsoft.com/azure/azure-resource-manager/templates/deployment-modes)
- [Bicep what-if operation](https://learn.microsoft.com/azure/azure-resource-manager/bicep/deploy-what-if)
- [Use tags to organize your Azure resources](https://learn.microsoft.com/azure/azure-resource-manager/management/tag-resources)
- [Azure activity log](https://learn.microsoft.com/azure/azure-monitor/platform/activity-log)
