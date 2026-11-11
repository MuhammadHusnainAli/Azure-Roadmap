# Building and Hosting MCP Servers on Azure

## Why it matters

Once agents speak MCP, every internal capability you want them to use (order lookup, ticket creation, document search, cost reports) becomes an MCP server. Those servers need the same disciplines as any API: hosting, scaling, authentication, rate limiting, monitoring, versioning and a catalogue so teams can find and reuse them. Azure offers several paths: **Azure Functions** (MCP extension or self-hosted SDK servers), **Azure Container Apps** (standalone containers or dynamic sessions), **App Service**, **API Management** (turn existing REST APIs into MCP servers and govern any MCP server), and **Azure API Center** (an organisational MCP registry that feeds Foundry tool catalogues).

## Key concepts

### Hosting options

| Option | How | Best for | Status notes |
|---|---|---|---|
| **Azure Functions: MCP extension** | `mcp_tool_trigger` (plus resource and prompt triggers) exposes functions as tools; endpoint `https://<app>.azurewebsites.net/runtime/webhooks/mcp` | Teams already using Functions; event-driven, serverless, bindings to Storage, Cosmos DB and more | Requires `azure-functions` ≥ 1.24.0 (Python), Core Tools ≥ 4.0.7030; C# isolated worker only; no PowerShell |
| **Azure Functions: self-hosted SDK server** | Run a server built with the official MCP SDKs on Functions | Existing SDK-based servers needing serverless scale | **Preview** |
| **Azure Container Apps: standalone** | Containerised MCP server with HTTP ingress | Any language or framework, full control, scale to zero, VNet | GA platform |
| **Container Apps: dynamic sessions** | Platform-managed session pools with built-in MCP tooling | Sandboxed code execution for agents | Session pools |
| **App Service** | Host MCP servers in web apps | Existing App Service estates | |
| **API Management: REST API as MCP server** | API operations become MCP tools without code | Reusing existing governed APIs | Supported tiers: Developer, Basic, Basic v2, Standard, Standard v2, Premium, Premium v2 |
| **API Management: existing MCP server** | Front any MCP server (Functions, Logic Apps, LangChain, third-party) | Central governance of MCP traffic | |
| **Logic Apps** | Expose workflows and connectors as MCP tools | Low-code integrations | |

### Securing a remote MCP server

| Layer | Options |
|---|---|
| **Client → server (inbound)** | OAuth 2.1 with **Microsoft Entra ID** per the MCP authorisation spec. Functions and App Service **built-in authentication** implement the MCP OAuth requirements (portal one-click setup creates the Entra app registration). Function keys (`x-functions-key`) are simple but are shared secrets; avoid in production. |
| **Gateway** | API Management validates JWTs (`validate-jwt`), applies rate limits, quotas, IP filtering, caching and logging |
| **Server → backend (outbound)** | Managed identity from the server or APIM to databases and APIs |
| **Network** | Private endpoints, VNet integration; Foundry private MCP requires Container Apps with internal ingress on a dedicated subnet |
| **On-behalf-of** | When the server acts on user data, exchange the user token (OBO) rather than using app permissions |

### Governance with API Management

API Management policies apply to all tools of an MCP server:

- Rate limiting and quotas per client or subscription.
- JWT validation for Entra ID or other identity providers.
- IP filtering.
- Response caching.
- Monitoring through Azure Monitor and Application Insights, correlation IDs and trace policies.

It also supports managing MCP servers as code (REST, ARM, Bicep, CLI, Terraform). This is the "MCP gateway" pattern alongside the AI gateway for models ([Topic 11 file 07](../11-integration-messaging-and-apis/07-api-management-as-ai-gateway.md)).

### Discovery with Azure API Center

API Center keeps an inventory of **remote and local MCP servers** (yours and partner servers), with metadata, owners and lifecycle, and a portal for discovery. Registered servers can feed **Foundry tool catalogues** and **private tool catalogues**, so agent builders only pick approved servers ([Topic 11 file 08](../11-integration-messaging-and-apis/08-azure-api-center.md)).

### Design principles for servers

| Principle | Why |
|---|---|
| Task-level tools with clear names and descriptions | Better tool selection, less misuse |
| Strict input schemas and server-side validation | Model arguments are untrusted |
| Structured outputs, minimal fields | Fewer tokens, less injection surface |
| Idempotent writes with request IDs | Safe retries |
| Stateless design (aligns with MCP 2026-07-28) | Horizontal scaling; explicit handles for state |
| Deterministic `tools/list` order | Better client caching and prompt caching |
| Versioning | Additive changes; new tools rather than breaking old ones |

## How it works on Azure

Azure Functions MCP extension (Python):

```python
import json, logging
import azure.functions as func

app = func.FunctionApp()

tool_props = json.dumps([
    {"propertyName": "order_id", "propertyType": "string", "description": "Contoso order ID"}
])

@app.mcp_tool_trigger(arg_name="context", tool_name="get_order_status",
                      description="Get the delivery status of a Contoso order.",
                      tool_properties=tool_props)
def get_order_status(context) -> str:
    args = json.loads(context)["arguments"]
    order_id = args["order_id"]
    logging.info("Lookup %s", order_id)
    return json.dumps({"order_id": order_id, "status": "out for delivery"})
```

`host.json` uses extension bundle `[4.0.0, 5.0.0)`. Deploy to a **Flex Consumption** function app, then enable built-in authentication (Entra) so MCP clients authenticate with OAuth.

Expose an existing REST API as an MCP server in API Management: open the API in the portal → **MCP servers → Create MCP server → Expose an API as an MCP server**, choose operations to expose as tools, then add a `validate-jwt` inbound policy and rate limits.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Functions vs Container Apps** | Functions for event-driven tools with bindings and minimal code; Container Apps for SDK-built servers, any language, long-running or stateful needs, and private Foundry MCP. |
| **Build a server vs APIM REST-to-MCP** | If a governed REST API exists, expose it through APIM first; build custom servers when tools need orchestration or logic beyond one API call. |
| **Direct vs via APIM** | Put shared or external-facing MCP servers behind APIM for auth, quotas and observability. |
| **Keys vs Entra** | Entra OAuth (built-in auth) for production; function keys only for development. |
| **Registry** | Register every production server in API Center; connect it to Foundry private tool catalogues. |
| **Granularity and ownership** | One server per domain owned by the domain team; avoid a monolithic "everything" server. |

## Hands-on

1. Create a Functions project with the MCP extension (VS Code, Python), add `get_order_status` above, run locally with Core Tools and test with the MCP Inspector.
2. Deploy to a Flex Consumption function app. In the portal, enable **built-in authentication** for MCP (one-click Entra setup).
3. Connect from VS Code (MCP server URL `https://<app>.azurewebsites.net/runtime/webhooks/mcp`), signing in with Entra.
4. Put the server behind **API Management** as an existing MCP server; add `validate-jwt` and `rate-limit-by-key` policies; confirm limits apply.
5. Register the server in **Azure API Center** with owner and lifecycle metadata.
6. Add it to a Foundry **toolbox** (Entra connection) and use it from a prompt agent ([Topic 18 file 07](../18-agents-with-foundry-agent-service/07-mcp-tools-toolboxes-and-skills.md)).

## Security, cost and reliability notes

- **Authentication**: production MCP servers should require Entra-issued, audience-bound tokens; don't rely on function keys.
- **Authorisation**: check the caller's roles or scopes per tool; separate read and write tools.
- **Least privilege outbound**: managed identity with narrow roles to backends.
- **Input validation and output sanitisation** on every tool.
- **Monitoring**: log tool, caller, latency and outcome; alert on error spikes and unusual call volumes.
- **Cost**: Functions Flex Consumption and Container Apps scale to zero; APIM tier and request volume drive gateway cost.
- **Transport deprecations**: the older SSE transport is deprecated; use Streamable HTTP.

## Common mistakes

| Mistake | Fix |
|---|---|
| Anonymous or key-only remote servers | Entra OAuth via built-in auth or APIM |
| Wrapping every API operation as a tool | Expose only task-relevant operations |
| No central inventory | Register in API Center |
| Server-side session state | Stateless design with explicit handles |
| Using the deprecated SSE endpoint | Streamable HTTP `/mcp` |
| Shared admin identity to backends | Managed identity with least privilege |

## Check yourself

1. What are the two ways to host MCP servers on Azure Functions, and their status?
2. What is the default MCP endpoint path for the Functions MCP extension?
3. What does Functions built-in authentication provide for MCP servers?
4. Name two ways API Management can expose MCP servers.
5. Which APIM policies help govern MCP servers?
6. What does Azure API Center add for MCP?
7. When would you choose Container Apps over Functions for an MCP server?
8. Why should production servers avoid function keys?

<details>
<summary>Answers</summary>

1. The MCP extension (tool/resource/prompt triggers) and self-hosting SDK-built servers (Preview).
2. `/runtime/webhooks/mcp`.
3. App Service authentication that implements the MCP authorisation OAuth requirements with Microsoft Entra ID (one-click setup in the portal).
4. Exposing a managed REST API as an MCP server, and fronting an existing MCP server.
5. Rate limiting and quotas, JWT validation, IP filtering, caching (plus tracing and monitoring).
6. An inventory/registry of remote and local MCP servers with discovery, which can feed Foundry tool catalogues.
7. For SDK-built servers in any language, long-running or stateful workloads, custom dependencies, or private Foundry MCP with internal ingress.
8. Keys are shared secrets that don't identify callers or support per-user authorisation; Entra OAuth does.

</details>

## References

- [Host an MCP server on Azure Functions](https://learn.microsoft.com/azure/azure-functions/functions-mcp-tutorial)
- [MCP bindings for Azure Functions](https://learn.microsoft.com/azure/azure-functions/functions-bindings-mcp)
- [MCP tool trigger](https://learn.microsoft.com/azure/azure-functions/functions-bindings-mcp-tool-trigger)
- [Host MCP servers on Azure Container Apps](https://learn.microsoft.com/azure/container-apps/mcp-overview)
- [MCP servers in Azure API Management](https://learn.microsoft.com/azure/api-management/mcp-server-overview)
- [Expose a REST API as an MCP server](https://learn.microsoft.com/azure/api-management/export-rest-mcp-server)
- [Register and discover MCP servers in API Center](https://learn.microsoft.com/azure/api-center/register-discover-mcp-server)
