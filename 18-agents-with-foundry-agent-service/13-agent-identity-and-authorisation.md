# Agent Identity and Authorisation

## Why it matters

An agent that calls tools is an actor in your environment: it reads files, queries databases, sends messages, changes records. If it runs under a shared service principal or a developer's account, you can't tell what the agent did versus what a person did, you can't scope its permissions, and you can't switch it off cleanly. **Microsoft Entra Agent ID** gives each agent its own identity type in Entra, and Foundry provisions and uses it automatically. Combined with **user delegation (on-behalf-of)** and **endpoint authorisation**, it lets you answer the three questions every security review asks: who can call this agent, what can the agent access, and whose permissions apply?

## Key concepts

### Three separate access questions

| Question | Mechanism | File |
|---|---|---|
| **Who can call the agent?** (inbound) | Endpoint authorisation schemes: AzureRbac (Foundry Agent Consumer, agent or project scope), AgentBlueprint (Preview), Bot Service schemes | 12 |
| **Who can manage the agent?** | Foundry RBAC: Foundry User, Project Manager, Owner | [Topic 14 file 03](../14-microsoft-foundry-platform/03-foundry-access-control.md) |
| **What can the agent access?** (outbound) | Agent identity with least-privilege roles, or user delegation, or project managed identity, or stored credentials | This file |

### Entra Agent ID objects

| Object | What it is |
|---|---|
| **Agent identity blueprint** | Parent object and governing template for an agent's identities; can also be the agent's sign-in audience (AgentBlueprint scheme) |
| **Agent identity** | A special service principal created under the blueprint, representing an agent instance when it accesses resources |
| **Agent user account** | Additional identity for **autopilots** in Agent 365, letting an agent act as itself like a user (for example a mailbox) |

In the current object model, **creating an agent provisions a unique blueprint and an identity** for its default instance (`instance_identity.principal_id`). Legacy agents share the project identity.

Credentials: Foundry places a federated identity credential on the blueprint trusting a Microsoft first-party app; **no client secrets or certificates** exist for you to manage, and neither your code nor the project managed identity holds the blueprint credential.

### A new agent has no permissions

Foundry doesn't assign downstream roles to agent identities. You grant access using the target service's model:

| Target authorisation | Example |
|---|---|
| Azure RBAC | Storage Blob Data Reader on one storage account |
| Service-specific data-plane roles | Cosmos DB data-plane role assignments |
| Application permissions / app roles | An internal API's app role |
| Resource-specific policies | The service's own checks |

Prefer resource or resource-group scope over subscription scope.

### Runtime token exchange (service-managed tools)

1. Agent Service authenticates as the **blueprint**.
2. Entra issues a token for the specific **agent identity**.
3. Agent Service exchanges it for a token scoped to the downstream **audience** (for example `https://storage.azure.com`).
4. The tool call (MCP, A2A, OpenAPI) carries that token; the resource checks the agent identity's roles.

Hosted agents get tokens for their own identity through the **hosted-agent token service**; the project managed identity isn't available to hosted-agent code, and sandboxes never contain blueprint credentials.

### Application-only vs delegated access

| Model | Identity downstream | Use when |
|---|---|---|
| **Agent identity** (application-only) | The agent as an independent principal | Operations that aren't user-specific (unattended jobs, shared data) |
| **User delegation: on-behalf-of** | The signed-in user (with Foundry or the agent identity as actor) | Downstream must apply **the user's** permissions: their mail, files, records |
| **User delegation: in-conversation OAuth** | User signs in to a specific connection during the chat | Third-party SaaS with its own OAuth |
| **Project managed identity** | The Foundry project | Shared project-level access (legacy and some connections) |
| **API key / custom credential** | Whoever owns the key | Services without Entra or OAuth |
| **None** | Anonymous | Public tools only |

Microsoft's rule: don't use delegation when every user should get the same application-level authority; don't use an application identity when the downstream service must preserve user-specific authorisation.

Connection settings: `UserEntraToken` (actor chosen by incoming token audience: Foundry or agent identity), `AgentOnBehalfOfUser` (agent identity always the actor; requires the AgentBlueprint endpoint scheme), `OAuth2` (in-conversation sign-in). Foundry IQ uses on-behalf-of with source-specific authorisation ([Topic 17 file 10](../17-rag-and-knowledge/10-security-trimming-and-permissions.md)).

### Governance with Agent 365 and Entra

Agent identities appear in Entra, so you can apply **Conditional Access**, **access reviews**, lifecycle policies and owner attestation. **Microsoft Agent 365** provides a registry of all agents (Foundry, Copilot Studio, registered and discovered "shadow" agents), access control, visualisation, interoperability and security (Defender, Purview).

## How it works on Azure

Grant an agent identity read access to a storage account:

```bash
PROJECT_ENDPOINT="https://<account>.services.ai.azure.com/api/projects/<project>"
AGENT_NAME="report-agent"

AGENT_ID=$(az rest --method GET \
  --url "$PROJECT_ENDPOINT/agents/$AGENT_NAME?api-version=v1" \
  --resource "https://ai.azure.com" \
  --query "instance_identity.principal_id" -o tsv)

az role assignment create \
  --assignee-object-id "$AGENT_ID" --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Reader" \
  --scope "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<sa>"
```

Foundry roles were renamed; Microsoft advises using **role definition IDs** in automation during the rollout (for example Foundry User `53ca6127-db72-4b80-b1b0-d745d6d5456d`). Role assignments can take a few minutes to propagate.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Agent identity vs OBO** | Agent identity for agent-owned tasks; OBO whenever results must reflect the user's own access (SharePoint, mail, CRM records). |
| **Per-agent vs shared identity** | Per-agent (default in the current model) for least privilege and audit; avoid reusing one identity across agents. |
| **Scope of roles** | Smallest resource scope; separate read and write roles; time-bound elevation for rare actions. |
| **Inbound access** | Agent-scope Foundry Agent Consumer for apps that call one agent; avoid giving callers project-wide Foundry User. |
| **Keys** | Avoid; use Entra or OAuth. If a key is unavoidable, store it in a project connection backed by Key Vault. |
| **Autopilots** | Only for agents that must act as a standalone worker (mailbox, Teams presence); apply strong governance. |
| **Conditional Access** | Apply risk-based policies to agent identities where supported. |

## Hands-on

1. Create a prompt agent `report-agent` with an MCP or OpenAPI tool that reads from Blob Storage using **agent identity** authentication.
2. Call it: expect an authorisation failure (new identities have no permissions).
3. Assign Storage Blob Data Reader to the agent identity at the storage account scope (script above); wait a few minutes; retry and succeed.
4. Check the storage account's sign-in and diagnostic logs: the caller is the agent's service principal, not you.
5. Create a second agent with the SharePoint tool (Preview), which uses the **user's** identity. Sign in as two users with different SharePoint permissions and compare answers.
6. In Entra, find the agent identity and blueprint; review what an administrator can see and govern.

## Security, cost and reliability notes

- **Excessive permissions** are the main identity risk; review agent role assignments regularly (access reviews).
- **Confused deputy**: an agent with broad application permissions can be tricked (prompt injection) into accessing data for users who shouldn't see it. Prefer OBO for user data.
- **Delegation chains**: when agents call agents, track whose authority flows downstream; avoid escalating from user context to application context silently.
- **Legacy agents** share the project identity; recreate them to get per-agent identities before production.
- **Audit**: Entra and resource logs show agent identities as distinct principals; feed them into Sentinel or your SIEM.
- **Propagation**: role changes aren't instant; build retries and clear error messages.

## Common mistakes

| Mistake | Fix |
|---|---|
| Assuming a new agent can access project resources | Grant roles explicitly |
| Assigning roles to the project managed identity for agent-identity tools | Assign to the agent's `instance_identity.principal_id` |
| Application identity for user mailbox or file access | Use OBO |
| Subscription-wide Contributor for convenience | Resource-scoped, least-privilege roles |
| API keys in tool connections for Azure services | Entra-based connections |
| Hard-coding renamed role names in scripts | Use role definition IDs |

## Check yourself

1. What are the three access questions for an agent, and which mechanism answers each?
2. What is an agent identity blueprint, and how is it related to an agent identity?
3. What permissions does a newly created agent identity have?
4. Describe the runtime token exchange for a service-managed tool call.
5. When should you use on-behalf-of rather than agent identity?
6. What do `UserEntraToken`, `AgentOnBehalfOfUser` and `OAuth2` mean in tool connections?
7. Why can't hosted-agent code use the project managed identity?
8. What does Microsoft Agent 365 add to agent governance?

<details>
<summary>Answers</summary>

1. Who can call it (endpoint authorisation schemes), who can manage it (Foundry RBAC), what it can access (agent identity, delegation, managed identity or credentials).
2. The blueprint is the parent object and governing template; agent identities are service principals created under it, and the blueprint can impersonate them.
3. None; you must grant access through each target service's authorisation model.
4. Agent Service authenticates as the blueprint, gets an agent identity token from Entra, exchanges it for a token scoped to the downstream audience, and calls the tool; the resource checks the agent identity's roles.
5. When the downstream service must apply the signed-in user's own permissions (their data).
6. `UserEntraToken`: OBO with actor chosen by incoming token audience (Foundry or agent identity); `AgentOnBehalfOfUser`: OBO with the agent identity always the actor (needs AgentBlueprint scheme); `OAuth2`: separate in-conversation OAuth sign-in.
7. Hosted agents get tokens for their own agent identity through the token service; the project managed identity isn't exposed to sandboxed code, which also contains no blueprint credentials.
8. A registry of all agents, Entra-based access control (RBAC, ABAC, Conditional Access), visualisation, interoperability with Microsoft 365, and security via Defender and Purview.

</details>

## References

- [Agent identity concepts in Foundry](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-identity)
- [User delegation for tools](https://learn.microsoft.com/azure/foundry/agents/concepts/user-delegation)
- [Identity and access overview](https://learn.microsoft.com/azure/foundry/agents/concepts/identity-access-overview)
- [Agent endpoint authorisation](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-endpoint-authorization)
- [Tool identity and authentication](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/tool-authentication)
- [Microsoft Agent 365 integration](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-365-integration)
- [Microsoft Entra Agent ID](https://learn.microsoft.com/entra/agent-id/identity-platform/what-is-agent-id)
