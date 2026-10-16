# Azure RBAC

> Topic: 02 · Identity and Access · Tracks: E ● A ● C ● · Status checked: 2026-10

## Why it matters

Azure role-based access control decides who can do what to which resource. It is how you grant a developer access to one resource group, let an application read one storage container, and allow an agent to call one model deployment. Almost every "access denied" you will debug is an RBAC question, and almost every over-privileged environment got that way by assigning Contributor because it was quicker.

## Key concepts

### The three parts of a role assignment

```text
Role assignment  =  Security principal  +  Role definition  +  Scope
                    (who)                  (what)              (where)
```

- **Security principal:** user, group, service principal, managed identity or agent identity.
- **Role definition:** a named set of permissions.
- **Scope:** management group, subscription, resource group or resource.

Assignments are **inherited downward** and are **additive**: effective permissions are the union of all assignments that apply. There is no way to subtract with a normal assignment.

### Anatomy of a role definition

```json
{
  "Name": "Storage Blob Data Reader",
  "IsCustom": false,
  "Description": "Allows for read access to Azure Storage blob containers and data",
  "Actions": [
    "Microsoft.Storage/storageAccounts/blobServices/containers/read",
    "Microsoft.Storage/storageAccounts/blobServices/generateUserDelegationKey/action"
  ],
  "NotActions": [],
  "DataActions": [
    "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read"
  ],
  "NotDataActions": [],
  "AssignableScopes": ["/"]
}
```

| Property | Meaning |
|---|---|
| `Actions` | Control-plane operations allowed |
| `NotActions` | Control-plane operations excluded from `Actions` (not a deny) |
| `DataActions` | Data-plane operations allowed |
| `NotDataActions` | Data-plane operations excluded from `DataActions` |
| `AssignableScopes` | Where the role can be assigned |

Permission strings follow `{Provider}/{resourceType}/{operation}`, where the operation is `read`, `write`, `delete`, `action` or `*`.

`NotActions` only narrows a wildcard in the same role. If another assignment grants the action, the principal still has it.

### The fundamental built-in roles

| Role | Manage resources | Assign roles | Access data |
|---|:-:|:-:|:-:|
| **Owner** | Yes | Yes | No |
| **Contributor** | Yes | No | No |
| **Reader** | View only | No | No |
| **User Access Administrator** | No | Yes | No |
| **Role Based Access Control Administrator** | No | Yes (can be constrained by conditions) | No |

None of these include `DataActions`. This is the control plane and data plane split from topic 01: being Owner of a storage account does not let you read its blobs with Entra authentication.

Prefer **Role Based Access Control Administrator** with a condition over Owner or User Access Administrator when someone only needs to delegate specific roles.

### Job-function and data roles

There are several hundred built-in roles. Learn the pattern, not the list: `{Service} {Data?} {Reader | Contributor | Owner | User}`.

| Service | Common roles |
|---|---|
| Storage | Storage Blob Data Reader / Contributor / Owner; Storage Queue Data Contributor; Storage Table Data Contributor |
| Key Vault | Key Vault Secrets User, Key Vault Secrets Officer, Key Vault Crypto User, Key Vault Administrator |
| Compute | Virtual Machine Contributor, Virtual Machine Administrator Login, Virtual Machine User Login |
| Networking | Network Contributor |
| Containers | AcrPull, AcrPush, Azure Kubernetes Service Cluster User Role, AKS RBAC Reader / Writer / Admin |
| Databases | SQL DB Contributor (control plane; data access uses database-level users); Cosmos DB uses its own data-plane RBAC |
| Monitoring | Monitoring Reader, Monitoring Contributor, Log Analytics Reader |
| Cost | Cost Management Reader, Billing Reader |
| Security | Security Reader, Security Admin |

### Roles for AI workloads

These are the ones you will assign in nearly every AI project. Names in this area have been revised more than once as Foundry evolved, so confirm the current list in the Foundry RBAC documentation.

| Role | Purpose |
|---|---|
| **Azure AI User** | Data-plane access to a Foundry project: build and run agents, call models, use the playground |
| **Azure AI Project Manager** | Manage a project and assign Azure AI User to others |
| **Azure AI Account Owner** | Manage the Foundry resource: create projects and model deployments |
| **Cognitive Services OpenAI User** | Call Azure OpenAI inference endpoints |
| **Cognitive Services OpenAI Contributor** | Inference plus create deployments and fine-tune |
| **Cognitive Services User** | Call Foundry Tools (Speech, Language, Vision, and so on) |
| **Search Index Data Reader** | Query an Azure AI Search index |
| **Search Index Data Contributor** | Read and write index documents |
| **Search Service Contributor** | Manage the search service and index definitions |
| **Storage Blob Data Reader / Contributor** | Read or write source documents |
| **Cosmos DB Built-in Data Contributor** | Read and write Cosmos DB data (assigned through Cosmos DB's own RBAC) |

A typical RAG application's identity map:

| Identity | Needs | On |
|---|---|---|
| Application's managed identity | Cognitive Services OpenAI User or Azure AI User | Foundry resource or project |
| Application's managed identity | Search Index Data Reader | Search service |
| Search service's managed identity | Storage Blob Data Reader | Storage account (for indexers) |
| Search service's managed identity | Cognitive Services OpenAI User | Foundry resource (for vectorisation) |
| Foundry project's managed identity | Search Index Data Contributor, Storage Blob Data Contributor | Search and storage (for agent tools and file handling) |
| Developers | Azure AI User | Project (dev only) |

Drawing this table is a standard part of an AI solution design.

### Scope

```text
/                                                         root (tenant)
/providers/Microsoft.Management/managementGroups/{mg}     management group
/subscriptions/{sub}                                      subscription
/subscriptions/{sub}/resourceGroups/{rg}                  resource group
/subscriptions/{sub}/resourceGroups/{rg}/providers/...    resource
```

Assign at the **narrowest scope that works**. A data role for one application belongs on the one storage account or search service, not on the subscription.

### Deny assignments

A deny assignment blocks actions even when a role grants them, and takes precedence. You cannot create them directly. They are created by the platform: by **deployment stacks** (to protect managed resources) and **managed applications**. If you need "everyone except" behaviour, restructure scopes or use Azure Policy.

### Conditions (Azure ABAC)

An optional condition on a role assignment that narrows it using attributes.

Two uses matter:

1. **Constrained delegation.** Let a team assign roles, but only certain roles to certain principal types. For example, allow a platform pipeline to assign only *Storage Blob Data Reader* and only to managed identities:

   ```text
   (
     @Request[Microsoft.Authorization/roleAssignments:RoleDefinitionId]
       ForAnyOfAnyValues:GuidEquals {2a2b9908-6ea1-4ae2-8e65-a410df84e7d1}
     AND
     @Request[Microsoft.Authorization/roleAssignments:PrincipalType]
       ForAnyOfAnyValues:StringEqualsIgnoreCase {'ServicePrincipal'}
   )
   ```

2. **Fine-grained storage data access.** Restrict blob access by container name, blob path, blob index tags, or the principal's custom security attributes. For example, read only blobs tagged `project=alpha`.

ABAC for data access is currently limited to Storage (blobs and queues).

### Custom roles

Create one when no built-in role fits and combining two would over-grant.

- Defined in JSON, stored in the tenant, assignable within `AssignableScopes`.
- Up to 5,000 per tenant.
- Start from the closest built-in role and remove permissions.
- Avoid wildcards; they silently include permissions added later.

Example: an operator who can restart web apps and read everything, nothing else.

```json
{
  "Name": "Web App Operator",
  "Description": "Can view resources and restart web apps.",
  "Actions": [
    "*/read",
    "Microsoft.Web/sites/restart/action",
    "Microsoft.Web/sites/start/action",
    "Microsoft.Web/sites/stop/action",
    "Microsoft.Support/*"
  ],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": ["/subscriptions/00000000-0000-0000-0000-000000000000"]
}
```

### How a request is evaluated

1. The principal obtains a token for Azure Resource Manager (or the data-plane service).
2. Azure collects all role assignments and deny assignments that apply at the resource scope and above, including through group membership.
3. If a deny assignment matches, access is denied.
4. Otherwise, if any role assignment grants the action (in `Actions` or `DataActions`, not removed by `NotActions`) and its condition, if any, is satisfied, access is allowed.
5. Otherwise, denied.

### Limits

| Item | Limit |
|---|---|
| Role assignments per subscription | Several thousand (see the limits page; this figure has changed) |
| Role assignments per management group | 500 |
| Custom roles per tenant | 5,000 |

Assigning to groups instead of individuals is how you stay well inside these.

### Classic administrators

Service Administrator and Co-Administrator roles from the pre-ARM era have been retired. If you see them in an old subscription or old documentation, the replacement is Owner at subscription scope.

## How it works on Azure

### Local authentication and why to disable it

Many services support two ways in: Entra ID with RBAC, and **local authentication** using account keys, connection strings, SAS tokens or API keys. Keys bypass RBAC, cannot be scoped to an identity, do not appear meaningfully in sign-in logs, and tend to leak.

Where your clients all support Entra authentication, **disable local auth**:

| Service | Setting |
|---|---|
| Storage | `allowSharedKeyAccess: false` |
| Foundry / Azure OpenAI / Foundry Tools | `disableLocalAuth: true` |
| Azure AI Search | Disable API keys (RBAC only) |
| Cosmos DB | `disableLocalAuth: true` |
| Service Bus, Event Hubs | `disableLocalAuth: true` |
| App Configuration | `disableLocalAuth: true` |

Enforce it with Azure Policy (topic 03).

### Key Vault: use RBAC

Key Vault supports the older **access policy** model and **Azure RBAC**. Use RBAC for new vaults: it gives per-secret scope, PIM support and one consistent model.

## Design decisions and trade-offs

| Choice | Guidance |
|---|---|
| Group vs individual | Always groups for people. Direct assignment is acceptable for a managed identity that has exactly one job |
| Scope | Resource or resource group for workloads; subscription and above for platform teams and policy |
| Built-in vs custom | Built-in first. Custom roles need maintenance as services add operations |
| Owner | Reserve for break-glass and automation that must assign roles. Most humans need Contributor at most |
| Standing vs just-in-time | Privileged roles should be eligible through PIM, not permanent |
| Role assignment by pipelines | Give the pipeline RBAC Administrator with a condition, not Owner |

**Separation of duties:** the people who deploy infrastructure should not be the same identity that approves role assignments in production. Conditions and PIM approval make this practical.

## Hands-on

```bash
RG=rg-rbac-lab
LOC=swedencentral
az group create -n $RG -l $LOC
ST=strbaclab$RANDOM
az storage account create -n $ST -g $RG -l $LOC --sku Standard_LRS --allow-shared-key-access false
ST_ID=$(az storage account show -n $ST -g $RG --query id -o tsv)
ME=$(az ad signed-in-user show --query id -o tsv)

# 1. Control plane works, data plane does not
az storage container create --account-name $ST -n docs --auth-mode login      # fails: no data role

# 2. Inspect a role definition
az role definition list --name "Storage Blob Data Contributor" \
  --query "[0].permissions[0].{actions:actions, dataActions:dataActions}"

# 3. Assign a data role at resource scope
az role assignment create --assignee-object-id $ME --assignee-principal-type User \
  --role "Storage Blob Data Contributor" --scope $ST_ID

# 4. Wait a minute for propagation, then retry
az storage container create --account-name $ST -n docs --auth-mode login

# 5. See what applies and where it comes from
az role assignment list --assignee $ME --scope $ST_ID --include-inherited \
  --query "[].{Role:roleDefinitionName, Scope:scope}" -o table

# 6. Find roles that grant a specific action
az role definition list \
  --query "[?contains(to_string(permissions[0].dataActions), 'Microsoft.CognitiveServices/accounts/OpenAI/deployments/chat/completions/action')].roleName" -o tsv

# 7. Clean up
az group delete -n $RG --yes --no-wait
```

In the portal, open **Access control (IAM) → Check access** on any resource to see effective access for a principal, and **Role assignments** to see inherited ones with their source scope.

## Security, cost and reliability notes

- **Security:** review assignments at subscription scope and above every quarter. Remove orphaned assignments (shown as *Identity not found*) left by deleted principals.
- **Security:** alert on `Microsoft.Authorization/roleAssignments/write` in the activity log for sensitive scopes.
- **Security:** keys are the escape hatch from RBAC. An RBAC design is only as good as the local-auth settings behind it.
- **Reliability:** role assignments can take up to a few minutes to propagate. IaC that assigns a role and immediately uses it should retry.
- **Cost:** RBAC is free. PIM and access reviews need P2 or ID Governance licences.

## Common mistakes

- Assigning Contributor at subscription scope "to get things working".
- Expecting Owner or Contributor to grant data access.
- Assigning to the application's client ID instead of the service principal's object ID.
- Using `NotActions` as if it were a deny.
- Leaving account keys enabled, which makes the careful RBAC design irrelevant.
- Creating custom roles with `*` and never revisiting them.
- Giving a deployment pipeline Owner so that it can create two role assignments.
- Forgetting that the *search service's* identity, not just the app's, needs access to storage and the model.

## Check yourself

1. What are the three components of a role assignment?
2. Why can a subscription Owner be denied when listing blobs with `--auth-mode login`?
3. What is the difference between `NotActions` and a deny assignment?
4. Which role lets an application call a chat model in Foundry without managing the resource?
5. What two problems do role assignment conditions solve?
6. An app's managed identity needs to query one search index. What role, at what scope?
7. Why disable local authentication?

<details>
<summary>Answers</summary>

1. Security principal, role definition, scope.
2. Owner contains no `DataActions`. Blob access needs a data role such as Storage Blob Data Reader.
3. `NotActions` removes operations from the same role's wildcard and can be overridden by another assignment. A deny assignment blocks the action regardless of other grants and can only be created by the platform.
4. Cognitive Services OpenAI User, or Azure AI User on the Foundry project.
5. Constrained delegation of role assignment, and attribute-based data access to storage.
6. Search Index Data Reader, scoped to the search service (or the index).
7. Keys bypass RBAC and Conditional Access, are not tied to an identity, are hard to audit and are easily leaked.

</details>

## References

- [What is Azure role-based access control?](https://learn.microsoft.com/azure/role-based-access-control/overview)
- [Understand Azure role definitions](https://learn.microsoft.com/azure/role-based-access-control/role-definitions)
- [Azure built-in roles](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles)
- [Understand scope for Azure RBAC](https://learn.microsoft.com/azure/role-based-access-control/scope-overview)
- [What is Azure attribute-based access control?](https://learn.microsoft.com/azure/role-based-access-control/conditions-overview)
- [Delegate Azure role assignment management with conditions](https://learn.microsoft.com/azure/role-based-access-control/delegate-role-assignments-overview)
- [Azure custom roles](https://learn.microsoft.com/azure/role-based-access-control/custom-roles)
- [Role-based access control for Microsoft Foundry](https://learn.microsoft.com/azure/foundry/concepts/rbac-foundry)
- [Best practices for Azure RBAC](https://learn.microsoft.com/azure/role-based-access-control/best-practices)
