# MCP Server on Azure

## Why it matters

The Model Context Protocol (MCP) has become the standard way to give agents tools. Foundry agents, Copilot Studio, GitHub Copilot, Visual Studio Code and many third-party clients can all consume MCP servers. That makes MCP servers a new kind of enterprise API: exposed to AI clients, and open to misuse through prompt injection, over-broad permissions and tool poisoning. In this capstone you **build, secure, host and govern a remote MCP server** on Azure.

**Brief:** Contoso's HR systems team wants to expose leave balances, leave requests and policy lookup to approved AI agents (a Foundry agent and Microsoft 365 Copilot via Copilot Studio). Each user may only see and change their own data, and every tool call must be auditable.

## Key concepts

### Requirements

| # | Requirement |
|---|---|
| R1 | Remote MCP server (Streamable HTTP transport) with three tools: `get_leave_balance`, `request_leave` (write), `search_leave_policy` |
| R2 | OAuth 2.1 authorisation with Microsoft Entra ID; tools act **on behalf of the signed-in user** |
| R3 | Clear tool descriptions and input schemas; server-side validation of every input |
| R4 | `request_leave` requires explicit user confirmation in the client, and server-side business rules |
| R5 | Hosted on Azure Functions or Container Apps with managed identity to back-end systems |
| R6 | Fronted by API Management for rate limits, logging and policy (stretch: register in API Center) |
| R7 | Audit log of every tool call: user, tool, arguments (redacted), result status |
| R8 | Consumed successfully from a Foundry agent and from one other MCP client |
| R9 | Security tests: prompt injection via policy text, attempt to read another user's data |

### Skills practised

Topics [19 files 04–05](../19-agent-frameworks-and-protocols/04-model-context-protocol.md), [18](../18-agents-with-foundry-agent-service/README.md), [02](../02-identity-and-access/README.md), [11](../11-integration-messaging-and-apis/README.md), [23](../23-responsible-ai-security-and-governance/README.md) and [25](../25-security/README.md).

## How it works on Azure

### Architecture

```text
MCP clients (Foundry agent, Copilot Studio, VS Code)
   └─ OAuth (Entra ID) ─▶ API Management (validate token, rate limit, log)  ─▶ MCP server
                                                                              (Functions MCP extension or
                                                                               Container Apps with an MCP SDK)
                                                                               ├─ on-behalf-of → HR API (user context)
                                                                               ├─ AI Search → leave policy index
                                                                               └─ audit events → Log Analytics
```

### Hosting options

| Option | Notes |
|---|---|
| **Azure Functions** (MCP extension or self-hosted MCP SDK servers) | Serverless, event-driven, quick start with templates |
| **Azure Container Apps** | Any language or SDK, long-lived connections, scaling rules, dynamic sessions for code tools |
| **API Management: expose REST API as MCP server** | Turn an existing REST API into MCP tools without new code; govern centrally |
| **App Service** | Familiar web hosting for existing teams |

### Milestones

| Milestone | Deliverable |
|---|---|
| M1 | Tool design: names, descriptions, schemas, read or write classification, threat model |
| M2 | Local MCP server with mocked HR API; test with MCP Inspector or VS Code |
| M3 | Deploy to Functions or Container Apps with IaC |
| M4 | Entra app registrations, OAuth flow, on-behalf-of calls; prove user isolation |
| M5 | API Management in front; rate limits and logging |
| M6 | Connect a Foundry agent (MCP tool with approval for write tools) and a second client |
| M7 | Security test report and audit dashboard |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| User vs app identity | Use the user's identity (on-behalf-of) for personal data; app identity only for shared, non-sensitive data |
| Number of tools | Few, well-described tools beat many overlapping ones; fewer tools reduce misuse and token cost |
| Write tools | Require client-side approval **and** server-side checks |
| Governance | Register servers in an inventory (API Center) with owners, data classification and approved clients |

## Hands-on

1. Complete milestones M1–M7.
2. Demonstrate that user A can't read user B's balance, even with a crafted prompt.
3. Show an audit trail for one leave request from agent prompt to HR API.
4. Write the server's README: tools, auth, data classification, owner, SLOs.
5. Compare token cost of tool descriptions before and after trimming.

## Security, cost and reliability notes

- Treat all tool inputs as untrusted. Validate types, ranges and ownership on the server.
- Watch for tool poisoning: pin approved server versions and review tool description changes.
- Don't return more data than needed; redact sensitive fields in logs.
- Rate-limit per user and per client; set timeouts on back-end calls.

## Common mistakes

| Mistake | Fix |
|---|---|
| Shared API key for all users | OAuth with user context |
| Trusting the model to check permissions | Server-side authorisation |
| Vague tool descriptions | Precise names, descriptions and schemas |
| No inventory | Register and govern MCP servers |

## Check yourself

1. Which transport do remote MCP servers typically use?
2. Why use on-behalf-of authentication for HR data?
3. Name three hosting options for MCP servers on Azure.
4. How do you protect write tools?
5. What is tool poisoning, and how do you mitigate it?

<details>
<summary>Answers</summary>

1. Streamable HTTP.
2. So the back end enforces the signed-in user's permissions and audit records show the real user.
3. Any three: Azure Functions, Container Apps, App Service, or API Management exposing a REST API as an MCP server.
4. Client-side approval, server-side business rules and authorisation, and audit logging.
5. Malicious or altered tool descriptions that steer the model. Mitigate by pinning approved versions, reviewing changes and using a governed inventory.

</details>

## References

- [Build a remote MCP server with Azure Functions](https://learn.microsoft.com/azure/azure-functions/functions-mcp-tutorial)
- [MCP on Azure Container Apps](https://learn.microsoft.com/azure/container-apps/mcp-overview)
- [MCP servers in API Management](https://learn.microsoft.com/azure/api-management/mcp-server-overview)
- [Expose a REST API as an MCP server](https://learn.microsoft.com/azure/api-management/export-rest-mcp-server)
- [Connect Foundry agents to MCP servers](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/model-context-protocol)
