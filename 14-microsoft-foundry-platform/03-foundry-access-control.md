# Foundry Access Control

## Why it matters

A Foundry resource holds model deployments that cost money, agents that act on data, and connections to storage, search and databases. Who can create projects, deploy models, build agents, publish them, or just call them is controlled by **Azure RBAC** at resource, project and even agent scope. Getting this right, with **keyless** authentication and **managed identities** everywhere, is a security baseline, and it's an area where the role names recently changed.

## Key concepts

### Authentication options

| Method | How | Recommendation |
|---|---|---|
| **Microsoft Entra ID** (tokens) | Users, service principals, managed identities get tokens; RBAC decides what they can do | **Default**. Supports least privilege, Conditional Access, auditing |
| **API keys** | Two resource keys in the `api-key` header (only on the OpenAI and Foundry Tools endpoints) | Prototypes only. Keys grant **full access with no role restrictions**. Disable with `disableLocalAuth: true` in production |

Token scopes used with `DefaultAzureCredential`:

| Endpoint | Scope / resource |
|---|---|
| Project endpoint (`*.services.ai.azure.com/api/projects/...`) via Foundry SDK | Handled by the SDK (Foundry audience `https://ai.azure.com`) |
| OpenAI v1 endpoint (`*.openai.azure.com/openai/v1`) via OpenAI SDK | `https://cognitiveservices.azure.com/.default` |
| Anthropic endpoint (`*.services.ai.azure.com/anthropic`) via `AnthropicFoundry` | `https://ai.azure.com/.default` |

### Scopes

| Scope | Use |
|---|---|
| **Subscription / resource group** | Platform admins; Cognitive Services Usages Reader for quota visibility (subscription scope only) |
| **Foundry resource** | Resource owners and managers; deployments; resource-wide access |
| **Foundry project** | Developers limited to one project |
| **Agent** | Endpoint access to a single agent (currently evaluated only for agent endpoint access) |

### Built-in Foundry roles

The Foundry roles were **renamed** in 2026. Role IDs and permissions are unchanged, so use role IDs in automation.

| Current name | Previous name | Role ID | Purpose |
|---|---|---|---|
| **Foundry Agent Consumer** | — | `eed3b665-ab3a-47b6-8f48-c9382fb1dad6` | Only interact with agent endpoints (least privilege for apps and end users calling agents) |
| **Foundry User** | Azure AI User | `53ca6127-db72-4b80-b1b0-d745d6d5456d` | Build and develop in projects (data actions) plus read access; call models and agents. Least privilege for developers |
| **Foundry Project Manager** | Azure AI Project Manager | `eadc314b-1a2d-4efa-be10-5d325db5065e` | Manage projects, build, **publish agents**, assign Foundry User to others |
| **Foundry Account Owner** | Azure AI Account Owner | `e47c6f54-e4a2-4754-9501-8e0985b135e1` | Create resources and projects, manage models and deployments; can assign Foundry User, ACR and monitoring roles; **no** data actions |
| **Foundry Owner** | Azure AI Owner | `c883944f-8b7b-4483-af10-35834be79c4a` | Full access including data actions and model management (highly privileged, self-serve) |

Permissions at a glance (from Microsoft Learn):

| Role | Create projects | Create resources | Build (data actions) | Assign roles | Manage models | Publish agents | Call agents |
|---|---|---|---|---|---|---|---|
| Foundry Agent Consumer | ✘ | ✘ | ✘ | ✘ | ✘ | ✘ | ✔ |
| Foundry User | ✘ | ✘ | ✔ | ✘ | ✘ | ✘ | ✔ |
| Foundry Project Manager | ✘ | ✘ | ✔ | Foundry User only | ✘ | ✔ | ✔ |
| Foundry Account Owner | ✔ | ✔ | ✘ | Foundry User, ACR, monitoring | ✔ | ✘ | ✘ |
| Foundry Owner | ✔ | ✔ | ✔ | Foundry User, ACR, monitoring | ✔ | ✔ | ✔ |
| Azure Owner | ✔ | ✔ | ✘ | Any role | ✔ | ✔ | ✘ |
| Azure Contributor | ✔ | ✔ | ✘ | ✘ | ✔ | ✘ | ✘ |

Points that catch people out:

- **Owner and Contributor don't grant data actions.** An Azure Owner can't build agents or call models with Entra ID until they also have Foundry User.
- Don't use **Cognitive Services** roles for Foundry project work (except **Cognitive Services Usages Reader** for quota visibility). They're for direct AI services access. (For a standalone Azure OpenAI resource, Cognitive Services OpenAI User/Contributor still apply.)
- Don't use **Azure AI Developer** for Foundry projects; it's scoped to Azure ML workspaces and hubs.
- Fine-tuning needs data and control plane permissions: Foundry Owner, or Foundry User + Foundry Account Owner.

### Minimum assignments to get started

1. **Foundry User** on the Foundry resource for each developer.
2. **Foundry User** on the Foundry resource for each **project's managed identity**.
3. **Cognitive Services Usages Reader** at subscription scope for anyone who needs to see quota.

When a project is created in the portal by someone who can assign roles, the first two are added automatically. This doesn't happen through the SDK or CLI.

### Managed identities

| Identity | Used for |
|---|---|
| **Foundry resource identity** (system or user-assigned) | Resource-level access: CMK in Key Vault, reading Blob storage, AI Search for indexing |
| **Project managed identity** | Agents and tools in the project accessing connected resources (Search, Storage, Cosmos DB, APIs) |
| **Agent identity (Microsoft Entra Agent ID)** | Agents can have their own Entra identities for least-privilege access and governance (topic 02 file 10 and topic 18) |
| **Your app's managed identity** | Calling Foundry from App Service, Container Apps, Functions or AKS (workload identity) with Foundry User or Foundry Agent Consumer |

## How it works on Azure

### Assign roles with role IDs

```bash
FOUNDRY_ID=$(az cognitiveservices account show -g rg-foundry -n fdy-contoso-prod --query id -o tsv)
PROJECT_ID="$FOUNDRY_ID/projects/claims-assistant"

# Developer group: Foundry User on one project, plus Reader on the resource (enterprise pattern)
az role assignment create --assignee-object-id <dev-group-object-id> --assignee-principal-type Group \
  --role 53ca6127-db72-4b80-b1b0-d745d6d5456d --scope "$PROJECT_ID"
az role assignment create --assignee-object-id <dev-group-object-id> --assignee-principal-type Group \
  --role Reader --scope "$FOUNDRY_ID"

# Web app's managed identity only needs to call one agent
az role assignment create --assignee-object-id <app-mi-principal-id> --assignee-principal-type ServicePrincipal \
  --role eed3b665-ab3a-47b6-8f48-c9382fb1dad6 --scope "$PROJECT_ID/agents/claims-agent"

# Disable key-based auth
az resource update --ids "$FOUNDRY_ID" --set properties.disableLocalAuth=true
```

### Keyless calls from code

```python
# Foundry SDK (project endpoint)
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
project = AIProjectClient(endpoint="https://<res>.services.ai.azure.com/api/projects/<proj>", credential=DefaultAzureCredential())

# OpenAI SDK (v1 endpoint)
from azure.identity import get_bearer_token_provider
from openai import OpenAI
tp = get_bearer_token_provider(DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default")
client = OpenAI(base_url="https://<res>.openai.azure.com/openai/v1/", api_key=tp)
```

In Azure, `DefaultAzureCredential` picks up the managed identity. Locally it uses your `az login` or VS Code sign-in. In production, prefer `ManagedIdentityCredential` (or set `AZURE_TOKEN_CREDENTIALS=prod`) to avoid credential-chain surprises.

### Enterprise RBAC mapping (Microsoft's sample)

| Persona | Role and scope |
|---|---|
| IT admin | Owner on subscription |
| AI platform managers | Foundry Account Owner on the Foundry resource |
| Team leads | Foundry Project Manager on the Foundry resource |
| Developers | Foundry User on their project + Reader on the resource |
| Apps and end users calling agents | Foundry Agent Consumer on the project or on a single agent |

### Custom roles

Build custom roles from `Microsoft.CognitiveServices/*` actions and data actions (for example only `Microsoft.CognitiveServices/accounts/AIServices/agents/*`) when built-in roles are too broad. The "disable preview features" guide lists data actions you can deny.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Keys vs Entra ID | Entra ID only in production; disable local auth and enforce it with Azure Policy |
| Project scope vs resource scope for developers | Project scope (Foundry User) for least privilege; resource scope only for platform teams |
| Who deploys models | Platform team (Foundry Account Owner) in production; developers can deploy in dev resources |
| Agent consumers | Foundry Agent Consumer at agent scope for apps and service principals that only invoke agents |
| Elevated roles | Use Privileged Identity Management (PIM) for Foundry Owner, Account Owner and Azure Owner |
| Naming in scripts | Use role IDs during the rename rollout |

## Hands-on

1. Assign yourself only **Contributor** on a Foundry resource and try to run an agent or chat via Entra ID. Note the 401/403 error. Add **Foundry User** and retry.
2. Disable local auth and confirm key-based calls fail while Entra calls succeed.
3. Create a user-assigned managed identity for a Container App, assign **Foundry Agent Consumer** at agent scope, and call the agent from the app.
4. Assign **Cognitive Services Usages Reader** at subscription scope and verify the quota page loads.
5. Write an Azure Policy assignment that audits Foundry resources with `disableLocalAuth` not set to true (built-in policies exist for Cognitive Services local auth).

## Security, cost and reliability notes

- **Security:** keys bypass RBAC entirely. A leaked key gives full access, so disable them.
- **Security:** project managed identities often need data access to Search and Storage. Grant narrow data roles (for example Search Index Data Reader, Storage Blob Data Reader) on specific resources.
- **Security:** apply Conditional Access to developers, use PIM for privileged roles, and review assignments regularly.
- **Cost:** restrict who can create deployments (Account Owner or Contributor) to prevent surprise PTU or high-TPM deployments.
- **Reliability:** role assignments can take a few minutes to propagate. Build retries into automation that creates and immediately uses identities.

## Common mistakes

- Assuming Azure Owner or Contributor can call models with Entra ID.
- Assigning legacy Cognitive Services or Azure AI Developer roles for Foundry project work.
- Forgetting the project managed identity's role, so agents fail to use tools or knowledge.
- Using role names in scripts during the rename and getting "role not found".
- Leaving API keys enabled in production.

## Check yourself

1. Which role should a developer building agents in one project get?
2. Why can't an Azure Owner call a model with Entra ID by default?
3. What is the least-privilege role for an app that only calls an agent?
4. What were the Foundry roles previously called?
5. Which role is needed to publish agents?
6. What scope does the Cognitive Services Usages Reader role need?
7. Why disable local auth?

<details>
<summary>Answers</summary>

1. Foundry User at the project scope (plus Reader on the resource for visibility).
2. Owner has control-plane permissions but no data actions; calling models and building agents are data actions granted by roles such as Foundry User.
3. Foundry Agent Consumer, at project or agent scope.
4. Azure AI User, Azure AI Owner, Azure AI Account Owner and Azure AI Project Manager.
5. Foundry Project Manager (minimum) or Foundry Owner.
6. Subscription scope.
7. API keys grant full access without role restrictions, can't be scoped or protected by Conditional Access, and are easily leaked.

</details>

## References

- [Role-based access control for Microsoft Foundry](https://learn.microsoft.com/azure/foundry/concepts/rbac-foundry)
- [Authentication and authorization in Foundry](https://learn.microsoft.com/azure/foundry/concepts/authentication-authorization-foundry)
- [AI + machine learning built-in roles](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles/ai-machine-learning)
- [Disable local authentication for Foundry Tools](https://learn.microsoft.com/azure/ai-services/disable-local-auth)
- [Disable preview features (data actions for custom roles)](https://learn.microsoft.com/azure/foundry/how-to/disable-preview-features)
- [Microsoft Entra Agent ID](https://learn.microsoft.com/entra/agent-id/identity-platform/what-is-agent-id)
