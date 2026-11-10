# Connections and Bring-Your-Own Resources

> Topic: 14 · Microsoft Foundry Platform · Tracks: E ● A ● C ○ · Status checked: 2026-11

## Why it matters

Foundry rarely works alone. Agents need knowledge (Azure AI Search), files (Storage), conversation state (Cosmos DB), secrets (Key Vault), telemetry (Application Insights), other models (another Foundry resource or a partner endpoint) and business tools (APIs, MCP servers). **Connections** are how a Foundry resource or project references these services securely. **Bring-your-own (BYO) resources** decide where your data actually lives. Both are central to security reviews, data residency questions and agent design.

## Key concepts

### Connections

A **connection** is a named, stored reference to an external resource: its target endpoint, category and authentication. Tools, agents, evaluations and SDKs refer to it by name or ID instead of embedding endpoints and secrets in code.

| Level | Scope | Example |
|---|---|---|
| **Resource-level connection** | Shared with all projects in the Foundry resource | Shared AI Search, Key Vault, a partner model endpoint |
| **Project-level connection** | Only that project | Team-specific Search index, a project's App Insights, an external API |

Common connection categories (check the current list in the portal or REST reference):

| Target | Typical use |
|---|---|
| **Azure AI Search** | Knowledge bases (Foundry IQ), vector stores, agent search tools |
| **Azure Blob Storage / ADLS Gen2** | Data for evaluations, batch, fine-tuning, agent files (standard setup) |
| **Azure Cosmos DB** | Agent conversation storage (standard setup) |
| **Azure Key Vault** | Customer-managed store for connection secrets (one Key Vault connection per Foundry resource) |
| **Application Insights** | Tracing and monitoring for agents and apps |
| **Another Foundry / Azure OpenAI resource** | Use models deployed elsewhere (for example in another region) |
| **API Management (AI gateway)** | Route model traffic through APIM; bring your own model behind APIM to Agent Service |
| **Custom keys / API key / OAuth** | External APIs, MCP servers, SaaS tools |
| **Fabric data agent, SharePoint, Bing grounding and other tool connections** | Specific Agent Service tools (topics 12, 18) |

### Authentication types

| Auth type | Meaning | Recommendation |
|---|---|---|
| **Microsoft Entra ID** (managed identity / user identity) | Foundry resource or project identity, or the end user's identity, gets a token for the target | **Preferred**: no secrets; grant RBAC on the target |
| **API key / custom keys** | Secret stored by Foundry (in a managed Key Vault, or your Key Vault) | Only when the target doesn't support Entra ID |
| **OAuth** | Delegated access for tools that act as the user | For user-context tools (for example some MCP servers) |

When the connection uses **managed identity**, you must grant that identity a role on the target, for example:

| Target | Role for the Foundry resource or project identity |
|---|---|
| Azure AI Search (query) | Search Index Data Reader |
| Azure AI Search (index management) | Search Index Data Contributor + Search Service Contributor |
| Storage (read) | Storage Blob Data Reader |
| Storage (write, standard agent setup) | Storage Blob Data Contributor (or the roles the setup template assigns) |
| Cosmos DB (standard agent setup) | Cosmos DB built-in data contributor (data plane RBAC) |
| Key Vault (CMK) | Key Vault Crypto User |

### Microsoft-managed vs bring-your-own

| Data | Default (basic) | Bring your own (standard) |
|---|---|---|
| Files uploaded to agents / OpenAI files | Microsoft-managed storage, logically separated | Your Azure Storage account |
| Agent conversations (threads, messages) | Microsoft-managed multitenant storage | Your **Azure Cosmos DB for NoSQL** |
| Vector stores for agent file search | Microsoft-managed | Your **Azure AI Search** |
| Connection secrets | Microsoft-managed Key Vault | Your **Azure Key Vault** (connection at resource level) |
| Encryption keys | Microsoft-managed keys | Customer-managed keys in your Key Vault (same region, soft delete and purge protection enabled) |

In the **standard agent setup**, data is isolated **per project** inside your Storage, Cosmos DB and Search resources. The agent service uses a **capability host** configuration to know which connections hold files, threads and vectors. Microsoft's templates create it for you.

### Why bring your own?

| Driver | BYO helps by |
|---|---|
| Data residency and sovereignty | Data stays in resources you control, in your chosen region |
| Network isolation | You can put Storage, Cosmos DB and Search behind private endpoints (file 04) |
| Encryption and retention policies | Your CMK, backup, retention and deletion processes |
| Audit and compliance | Your diagnostic settings, Defender and Purview coverage |
| Reuse | Existing Search indexes and data lakes become agent knowledge |

The cost is more resources to run, secure and pay for.

## How it works on Azure

### Create a project connection to AI Search (Bicep, Entra ID)

```bicep
resource project 'Microsoft.CognitiveServices/accounts/projects@2025-06-01' existing = {
  name: '${foundryName}/claims-assistant'
}

resource searchConn 'Microsoft.CognitiveServices/accounts/projects/connections@2025-06-01' = {
  parent: project
  name: 'search-kb'
  properties: {
    category: 'CognitiveSearch'
    target: 'https://srch-contoso-kb.search.windows.net'
    authType: 'AAD'                       // Microsoft Entra ID, no key stored
    metadata: {
      ApiType: 'Azure'
      ResourceId: searchServiceId
      location: searchLocation
    }
  }
}
```

```bash
# Grant the project's managed identity query access on the search service
PROJ_MI=$(az cognitiveservices account project show -g rg-foundry -n fdy-contoso-prod \
  --project-name claims-assistant --query identity.principalId -o tsv)
az role assignment create --assignee-object-id $PROJ_MI --assignee-principal-type ServicePrincipal \
  --role "Search Index Data Reader" \
  --scope $(az search service show -g rg-search -n srch-contoso-kb --query id -o tsv)

# Inspect connections with the CLI
az cognitiveservices account project connection list -g rg-foundry -n fdy-contoso-prod --project-name claims-assistant -o table
```

Connections can also be created with `az cognitiveservices account project connection create --file <json-or-yaml>`, the REST API, the SDK or the portal (**Manage → Connected resources**). Resource-level connections use `Microsoft.CognitiveServices/accounts/connections`. Check the REST reference for the category and auth-type values of each connection type.

### Use a connection from code

```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

project = AIProjectClient(endpoint="https://<res>.services.ai.azure.com/api/projects/<proj>",
                          credential=DefaultAzureCredential())

for c in project.connections.list():
    print(c.name, c.type, c.target)

search = project.connections.get("search-kb")      # use search.id when configuring an agent tool
```

### Bring your own Key Vault

Create one **Azure Key Vault connection** at the Foundry resource level, and all API-key-based connection secrets for the resource and its projects are stored there instead of the Microsoft-managed vault. Grant the Foundry resource's managed identity the required Key Vault role (for example **Key Vault Secrets Officer**, per the setup guide) and plan the switch early. Moving secrets later means recreating connections.

### Bring your own model endpoint via the AI gateway

Agent Service can use models behind your **API Management** AI gateway ("bring your own model"), for example models hosted in another region, by a partner, or self-hosted. That gives you central token limits, logging and routing for agent model traffic (topic 11 file 07).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Resource vs project connections | Resource-level for shared, platform-owned services (central Search, Key Vault, App Insights); project-level for team-owned resources |
| Basic vs standard agent setup | Standard (BYO Storage, Cosmos DB, Search) for production with sensitive data, private networking or residency needs; basic for prototypes |
| Keys vs Entra ID in connections | Entra ID wherever the target supports it; keys only for external APIs that require them, stored in your Key Vault |
| One shared Search vs per-project Search | Shared for cost; per-project or per-domain for security boundaries and independent scaling |
| Managed vs own Key Vault | Own Key Vault when you need control over secret rotation, access policies and auditing |

## Hands-on

1. Create a project connection to an Azure AI Search service using Entra ID and grant the project identity **Search Index Data Reader**.
2. List the connections from Python and print their targets.
3. Create an agent that uses the Search connection as a knowledge tool and confirm it answers from your index.
4. Set up a Key Vault connection at the resource level, then create an API-key connection to an external API and confirm the secret appears in your vault.
5. Deploy Microsoft's standard agent setup template and inspect the Storage, Cosmos DB and Search resources it uses, and the per-project data separation.

## Security, cost and reliability notes

- **Security:** connections are powerful. Anyone with Foundry User on a project can use its connections through agents and tools, so treat project membership as access to every connected resource.
- **Security:** prefer project-level connections for sensitive resources, so other projects can't use them.
- **Security:** grant connected-resource roles to the narrowest identity (project identity rather than resource identity) and the narrowest scope (index or container, where supported).
- **Cost:** BYO resources (Cosmos DB, dedicated Search tiers, Storage) are billed separately. Size them for agent workloads.
- **Reliability:** connected resources need their own backup, zone redundancy and disaster recovery. Foundry doesn't fail them over.

## Common mistakes

- Creating a connection with managed identity but forgetting the RBAC assignment on the target.
- Storing API keys in agent instructions or code instead of connections.
- Assuming project isolation applies to resource-level connections.
- Switching to a customer Key Vault after many key-based connections exist.
- Using basic agent setup for regulated production data.

## Check yourself

1. What is a connection in Foundry?
2. What's the difference between resource-level and project-level connections?
3. Which three resources does the standard agent setup bring your own of?
4. What must you do after creating an Entra ID-based connection to AI Search?
5. Where are API-key connection secrets stored by default, and how do you change that?
6. Why might you route agent model calls through APIM?

<details>
<summary>Answers</summary>

1. A named, stored reference to an external resource (target, category, authentication) that tools, agents and SDKs use without embedding endpoints or secrets.
2. Resource-level connections are shared by all projects in the Foundry resource; project-level connections are usable only within that project.
3. Azure Storage (files), Azure Cosmos DB for NoSQL (conversations) and Azure AI Search (vector stores).
4. Grant the relevant identity (usually the project managed identity) a data role on the Search service, such as Search Index Data Reader.
5. In a Microsoft-managed Key Vault; create an Azure Key Vault connection at the Foundry resource level to store them in your own vault.
6. To apply central governance (token limits, logging, routing, failover) and to bring models hosted elsewhere into Agent Service.

</details>

## References

- [Add a new connection to your project](https://learn.microsoft.com/azure/foundry/how-to/connections-add)
- [Bring-your-own resources with Agent Service](https://learn.microsoft.com/azure/foundry/agents/how-to/use-your-own-resources)
- [Set up an Azure Key Vault connection](https://learn.microsoft.com/azure/foundry/how-to/set-up-key-vault-connection)
- [Customer-managed keys for Foundry](https://learn.microsoft.com/azure/foundry/concepts/customer-managed-keys)
- [Bring your own model to Agent Service via AI gateway](https://learn.microsoft.com/azure/foundry/agents/how-to/ai-gateway)
- [az cognitiveservices account project connection](https://learn.microsoft.com/cli/azure/cognitiveservices/account/project/connection)
